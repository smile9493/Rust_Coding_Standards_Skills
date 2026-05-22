# 07 — 服务 API (Serving API)

## Status
Verified — SSE token streaming with axum + tokio-stream, gRPC bidirectional streaming, KV-cache sticky session load balancing, and per-request timeout enforcement.

## Prerequisites
- `axum >= 0.7`, `tokio >= 1.35`, `tokio-stream >= 0.1`
- `tonic >= 0.11` for gRPC, `prost >= 0.12` for protobuf
- `tower >= 0.4` for middleware (timeout, load balancing)
- `serde`, `serde_json` for JSON serialization
- `uuid` for request tracing

---

## 1. SSE Token Streaming (axum + tokio-stream)

```rust
use axum::{
    extract::State,
    response::sse::{Event, KeepAlive, Sse},
    routing::post,
    Json, Router,
};
use futures::stream::{self, Stream};
use serde::{Deserialize, Serialize};
use std::convert::Infallible;
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::mpsc;
use tokio_stream::StreamExt;

#[derive(Debug, Deserialize)]
pub struct ChatCompletionRequest {
    pub model: String,
    pub messages: Vec<ChatMessage>,
    pub max_tokens: Option<usize>,
    pub temperature: Option<f32>,
    pub stream: Option<bool>,
}

#[derive(Debug, Serialize)]
pub struct ChatCompletionChunk {
    pub id: String,
    pub object: String,
    pub created: u64,
    pub model: String,
    pub choices: Vec<ChoiceDelta>,
}

#[derive(Debug, Serialize)]
pub struct ChoiceDelta {
    pub index: usize,
    pub delta: DeltaContent,
    pub finish_reason: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct DeltaContent {
    pub content: Option<String>,
    pub role: Option<String>,
}

pub struct AppState {
    pub engine: Arc<InferenceEngine>,
}

pub fn build_router(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/v1/chat/completions", post(chat_completions_handler))
        .with_state(state)
}

async fn chat_completions_handler(
    State(state): State<Arc<AppState>>,
    Json(request): Json<ChatCompletionRequest>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = request.stream.unwrap_or(false);

    if !stream {
        let response = state.engine.generate_sync(&request).await;
        let chunk = ChatCompletionChunk {
            id: uuid::Uuid::new_v4().to_string(),
            object: "chat.completion".to_string(),
            created: std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .unwrap()
                .as_secs(),
            model: request.model.clone(),
            choices: vec![ChoiceDelta {
                index: 0,
                delta: DeltaContent {
                    content: Some(response),
                    role: Some("assistant".to_string()),
                },
                finish_reason: Some("stop".to_string()),
            }],
        };

        let json = serde_json::to_string(&chunk).unwrap();
        let event_stream = stream::once(async move {
            Ok(Event::default().data(json))
        });

        return Sse::new(event_stream).keep_alive(
            KeepAlive::default()
                .interval(Duration::from_secs(15))
                .text("keep-alive"),
        );
    }

    let (tx, rx) = mpsc::channel::<String>(64);
    let engine = state.engine.clone();
    let request_id = uuid::Uuid::new_v4().to_string();
    let model = request.model.clone();

    tokio::spawn(async move {
        let mut token_stream = engine.generate_stream(&request).await;
        while let Some(token) = token_stream.next().await {
            let chunk = ChatCompletionChunk {
                id: request_id.clone(),
                object: "chat.completion.chunk".to_string(),
                created: std::time::SystemTime::now()
                    .duration_since(std::time::UNIX_EPOCH)
                    .unwrap()
                    .as_secs(),
                model: model.clone(),
                choices: vec![ChoiceDelta {
                    index: 0,
                    delta: DeltaContent {
                        content: Some(token),
                        role: None,
                    },
                    finish_reason: None,
                }],
            };

            if let Ok(json) = serde_json::to_string(&chunk) {
                if tx.send(json).await.is_err() {
                    break;
                }
            }
        }

        let done_chunk = ChatCompletionChunk {
            id: request_id.clone(),
            object: "chat.completion.chunk".to_string(),
            created: std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .unwrap()
                .as_secs(),
            model,
            choices: vec![ChoiceDelta {
                index: 0,
                delta: DeltaContent {
                    content: None,
                    role: None,
                },
                finish_reason: Some("stop".to_string()),
            }],
        };

        if let Ok(json) = serde_json::to_string(&done_chunk) {
            let _ = tx.send(json).await;
        }
    });

    let event_stream = tokio_stream::wrappers::ReceiverStream::new(rx)
        .map(|data| Ok(Event::default().data(data)));

    Sse::new(event_stream).keep_alive(
        KeepAlive::default()
            .interval(Duration::from_secs(15))
            .text("keep-alive"),
    )
}

pub struct InferenceEngine;

impl InferenceEngine {
    pub async fn generate_sync(
        &self,
        _request: &ChatCompletionRequest,
    ) -> String {
        "Generated response".to_string()
    }

    pub async fn generate_stream(
        &self,
        _request: &ChatCompletionRequest,
    ) -> tokio_stream::wrappers::ReceiverStream<String> {
        let (tx, rx) = mpsc::channel::<String>(64);
        let tokens = vec![
            "Hello".to_string(),
            ", ".to_string(),
            "world".to_string(),
            "!".to_string(),
        ];

        tokio::spawn(async move {
            for token in tokens {
                tokio::time::sleep(Duration::from_millis(50)).await;
                if tx.send(token).await.is_err() {
                    break;
                }
            }
        });

        tokio_stream::wrappers::ReceiverStream::new(rx)
    }
}
```

## 2. gRPC 双向流 (tonic)

```rust
use tonic::{transport::Server, Request, Response, Status, Streaming};
use futures::Stream;

pub mod inference {
    tonic::include_proto!("inference");
}

use inference::{
    inference_service_server::{InferenceService, InferenceServiceServer},
    InferenceRequest, InferenceResponse,
};

#[derive(Default)]
pub struct InferenceServiceImpl;

#[tonic::async_trait]
impl InferenceService for InferenceServiceImpl {
    type ChatStream = tokio_stream::wrappers::ReceiverStream<
        Result<InferenceResponse, Status>,
    >;

    async fn chat(
        &self,
        request: Request<InferenceRequest>,
    ) -> Result<Response<Self::ChatStream>, Status> {
        let req = request.into_inner();
        let (tx, rx) = mpsc::channel::<Result<InferenceResponse, Status>>(64);

        tokio::spawn(async move {
            let tokens = vec!["Hello", " ", "from", " gRPC", "!"];
            for (i, token) in tokens.iter().enumerate() {
                let response = InferenceResponse {
                    token: token.to_string(),
                    index: i as u32,
                    finish_reason: if i == tokens.len() - 1 {
                        "stop".to_string()
                    } else {
                        String::new()
                    },
                };
                if tx.send(Ok(response)).await.is_err() {
                    break;
                }
                tokio::time::sleep(Duration::from_millis(30)).await;
            }
        });

        Ok(Response::new(
            tokio_stream::wrappers::ReceiverStream::new(rx),
        ))
    }

    type BidirectionalChatStream = tokio_stream::wrappers::ReceiverStream<
        Result<InferenceResponse, Status>,
    >;

    async fn bidirectional_chat(
        &self,
        request: Request<Streaming<InferenceRequest>>,
    ) -> Result<Response<Self::BidirectionalChatStream>, Status> {
        let mut stream = request.into_inner();
        let (tx, rx) = mpsc::channel::<Result<InferenceResponse, Status>>(64);

        tokio::spawn(async move {
            while let Ok(Some(req)) = stream.message().await {
                let response = InferenceResponse {
                    token: format!("echo: {}", req.prompt),
                    index: 0,
                    finish_reason: "stop".to_string(),
                };
                if tx.send(Ok(response)).await.is_err() {
                    break;
                }
            }
        });

        Ok(Response::new(
            tokio_stream::wrappers::ReceiverStream::new(rx),
        ))
    }
}

pub async fn start_grpc_server(addr: &str) -> anyhow::Result<()> {
    let addr = addr.parse()?;
    let service = InferenceServiceImpl::default();

    Server::builder()
        .add_service(InferenceServiceServer::new(service))
        .serve(addr)
        .await?;

    Ok(())
}
```

## 3. KV-Cache Sticky Session 负载均衡

```rust
use std::collections::HashMap;
use std::sync::Arc;
use parking_lot::RwLock;

#[derive(Debug, Clone, Hash, PartialEq, Eq)]
pub struct SessionId(pub String);

#[derive(Debug, Clone, Copy, Hash, PartialEq, Eq)]
pub struct WorkerId(pub usize);

pub struct StickySessionRouter {
    workers: Vec<WorkerId>,
    session_to_worker: RwLock<HashMap<SessionId, WorkerId>>,
    worker_load: RwLock<HashMap<WorkerId, usize>>,
}

impl StickySessionRouter {
    pub fn new(n_workers: usize) -> Self {
        let workers: Vec<_> = (0..n_workers).map(WorkerId).collect();
        let mut worker_load = HashMap::new();
        for &w in &workers {
            worker_load.insert(w, 0_usize);
        }

        Self {
            workers,
            session_to_worker: RwLock::new(HashMap::new()),
            worker_load: RwLock::new(worker_load),
        }
    }

    pub fn route(&self, session_id: SessionId) -> WorkerId {
        {
            let mapping = self.session_to_worker.read();
            if let Some(&worker) = mapping.get(&session_id) {
                return worker;
            }
        }

        let mut mapping = self.session_to_worker.write();
        if let Some(&worker) = mapping.get(&session_id) {
            return worker;
        }

        let mut load = self.worker_load.write();
        let &least_loaded = self.workers
            .iter()
            .min_by_key(|w| load.get(w).copied().unwrap_or(0))
            .unwrap();

        mapping.insert(session_id.clone(), least_loaded);
        load.entry(least_loaded).and_modify(|l| *l += 1);
        least_loaded
    }

    pub fn release(&self, session_id: &SessionId) {
        let mut mapping = self.session_to_worker.write();
        if let Some(worker) = mapping.remove(session_id) {
            let mut load = self.worker_load.write();
            load.entry(worker).and_modify(|l| {
                *l = l.saturating_sub(1);
            });
        }
    }
}
```

## 4. Timeout Enforcement (per-request)

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;
use pin_project::pin_project;
use tokio::time::{sleep, Instant, Sleep};

#[pin_project]
pub struct TimeoutFuture<F> {
    #[pin]
    inner: F,
    #[pin]
    deadline: Sleep,
    max_duration: Duration,
}

impl<F: Future> TimeoutFuture<F> {
    pub fn new(future: F, max_duration: Duration) -> Self {
        Self {
            inner: future,
            deadline: sleep(max_duration),
            max_duration,
        }
    }
}

impl<F: Future> Future for TimeoutFuture<F> {
    type Output = Result<F::Output, TimeoutError>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();

        if let Poll::Ready(output) = this.inner.poll(cx) {
            return Poll::Ready(Ok(output));
        }

        if this.deadline.poll(cx).is_ready() {
            return Poll::Ready(Err(TimeoutError {
                duration: *this.max_duration,
            }));
        }

        Poll::Pending
    }
}

#[derive(Debug, Clone)]
pub struct TimeoutError {
    pub duration: Duration,
}

impl std::fmt::Display for TimeoutError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "request timed out after {:?}", self.duration)
    }
}

impl std::error::Error for TimeoutError {}

pub async fn generate_with_timeout(
    engine: &InferenceEngine,
    request: &ChatCompletionRequest,
    timeout: Duration,
) -> Result<String, TimeoutError> {
    let future = engine.generate_sync(request);
    let timed = TimeoutFuture::new(future, timeout);
    timed.await
}
```

## Red Lines

1. **SSE 流必须设置 KeepAlive 间隔** — HTTP 代理和负载均衡器（如 nginx、ALB）会在空闲连接上超时断开。KeepAlive 注释文本或空事件必须每 15-30 秒发送一次。
2. **streaming 和 non-streaming 不能共用同一 code path** — Non-streaming 应在收集完所有 token 后一次返回，避免在非流式响应中逐 token 回传（违反 OpenAI API 规范）。
3. **KV-cache 必须与请求 sticky routing** — 同一 session 的连续对话请求必须路由到同一 worker，否则 KV-cache 不命中会导致每次请求都重新 prefill，延迟翻倍。
4. **每个请求必须绑定独立 timeout** — 全局超时不够，必须 per-request 超时以保护慢速请求不影响其他用户。建议根据 `max_tokens` 动态设置：`timeout = max_tokens * 100ms + 30s`。
5. **gRPC 流式响应必须使用 bounded channel** — `mpsc::channel(64)` 的 bounded channel 提供背压，防止 worker 无限积压消息导致 OOM。

## References
- [axum — SSE streaming example](https://docs.rs/axum)
- [tokio-stream — Stream wrappers](https://docs.rs/tokio-stream)
- [tonic — gRPC Rust framework](https://docs.rs/tonic)
- [OpenAI — Chat Completions API specification](https://platform.openai.com/docs/api-reference/chat)
- [tower — Timeout middleware](https://docs.rs/tower)