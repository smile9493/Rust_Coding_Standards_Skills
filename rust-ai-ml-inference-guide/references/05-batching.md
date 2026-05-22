# 05 — 批处理与调度 (Batching & Scheduling)

## Status
Verified — Static batching vs continuous batching comparison, vLLM-style dynamic batching with prefill/decode splitting, padding waste elimination via PagedAttention concept, and throughput optimization strategies.

## Prerequisites
- `candle-core >= 0.7`, `candle-nn >= 0.7`
- `tokio >= 1.35` for async scheduling
- Understanding of transformer prefill vs decode phase characteristics
- CUDA Graph support for decode kernel fusion (optional)

---

## 1. Static Batching vs Continuous Batching

```rust
use std::collections::VecDeque;
use std::time::Instant;

#[derive(Debug, Clone)]
pub enum BatchingStrategy {
    Static { max_batch_size: usize },
    Continuous {
        max_batch_size: usize,
        max_waiting_sequences: usize,
    },
}

#[derive(Debug)]
pub struct BatchMetrics {
    pub strategy: BatchingStrategy,
    pub batch_size: usize,
    pub total_tokens: usize,
    pub padding_tokens: usize,
    pub padding_waste_pct: f32,
    pub elapsed_ms: u64,
    pub tokens_per_second: f32,
}

pub struct BatchScheduler {
    strategy: BatchingStrategy,
    pending_requests: VecDeque<PendingRequest>,
}

#[derive(Debug, Clone)]
pub struct PendingRequest {
    pub id: u64,
    pub token_ids: Vec<u32>,
    pub max_new_tokens: usize,
    pub arrived_at: Instant,
}

impl BatchScheduler {
    pub fn new(strategy: BatchingStrategy) -> Self {
        Self {
            strategy,
            pending_requests: VecDeque::new(),
        }
    }

    pub fn enqueue(&mut self, request: PendingRequest) {
        self.pending_requests.push_back(request);
    }

    pub fn schedule_static(&mut self, max_batch_size: usize) -> Vec<PendingRequest> {
        let count = self.pending_requests.len().min(max_batch_size);
        self.pending_requests.drain(..count).collect()
    }

    pub fn compute_padding_waste(batch: &[PendingRequest]) -> f32 {
        if batch.is_empty() {
            return 0.0;
        }

        let max_len = batch
            .iter()
            .map(|r| r.token_ids.len())
            .max()
            .unwrap_or(0);

        let total_padding: usize = batch
            .iter()
            .map(|r| max_len - r.token_ids.len())
            .sum();

        let total_capacity = max_len * batch.len();
        (total_padding as f32 / total_capacity as f32) * 100.0
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_padding_waste_calculation() {
        let batch = vec![
            PendingRequest {
                id: 1,
                token_ids: vec![0; 500],
                max_new_tokens: 256,
                arrived_at: Instant::now(),
            },
            PendingRequest {
                id: 2,
                token_ids: vec![0; 100],
                max_new_tokens: 256,
                arrived_at: Instant::now(),
            },
            PendingRequest {
                id: 3,
                token_ids: vec![0; 50],
                max_new_tokens: 256,
                arrived_at: Instant::now(),
            },
            PendingRequest {
                id: 4,
                token_ids: vec![0; 200],
                max_new_tokens: 256,
                arrived_at: Instant::now(),
            },
        ];

        let waste = BatchScheduler::compute_padding_waste(&batch);
        println!("Padding waste: {:.1}%", waste);
        assert!(waste > 40.0);
    }
}
```

## 2. Prefill/Decode 分离调度

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum SequenceState {
    Waiting,
    Prefilling,
    Decoding,
    Finished,
}

#[derive(Debug)]
pub struct ManagedSequence {
    pub id: u64,
    pub state: SequenceState,
    pub prompt_tokens: Vec<u32>,
    pub generated_tokens: Vec<u32>,
    pub max_new_tokens: usize,
    pub kv_cache_slot: Option<usize>,
    pub kv_cache_len: usize,
}

pub struct PrefillDecodeScheduler {
    pub sequences: HashMap<u64, ManagedSequence>,
    pub max_prefill_tokens: usize,
    pub max_decode_sequences: usize,
    pub kv_cache: Vec<bool>,
}

impl PrefillDecodeScheduler {
    pub fn new(
        max_prefill_tokens: usize,
        max_decode_sequences: usize,
        max_kv_slots: usize,
    ) -> Self {
        Self {
            sequences: HashMap::new(),
            max_prefill_tokens,
            max_decode_sequences,
            kv_cache: vec![false; max_kv_slots],
        }
    }

    pub fn add_sequence(&mut self, id: u64, tokens: Vec<u32>, max_new_tokens: usize) {
        self.sequences.insert(
            id,
            ManagedSequence {
                id,
                state: SequenceState::Waiting,
                prompt_tokens: tokens,
                generated_tokens: Vec::new(),
                max_new_tokens,
                kv_cache_slot: None,
                kv_cache_len: 0,
            },
        );
    }

    pub fn schedule_step(&mut self) -> (Vec<u64>, Vec<u64>) {
        let mut prefill_ids = Vec::new();
        let mut decode_ids = Vec::new();

        let active_decoding = self.sequences
            .values()
            .filter(|s| s.state == SequenceState::Decoding)
            .count();

        let decode_slots = self.max_decode_sequences - active_decoding;

        let mut prefill_token_count = 0_usize;
        let mut waiting: Vec<u64> = self.sequences
            .iter()
            .filter(|(_, s)| s.state == SequenceState::Waiting)
            .map(|(id, _)| *id)
            .collect();
        waiting.sort_by_key(|id| self.sequences[id].prompt_tokens.len());

        for id in waiting {
            let seq = &self.sequences[&id];
            if seq.prompt_tokens.len() > self.max_prefill_tokens {
                continue;
            }
            if prefill_token_count + seq.prompt_tokens.len() > self.max_prefill_tokens {
                continue;
            }
            if decode_slots == 0 && !prefill_ids.is_empty() {
                break;
            }

            if let Some(slot) = self.allocate_kv_slot(seq.prompt_tokens.len()) {
                prefill_token_count += seq.prompt_tokens.len();
                prefill_ids.push(id);

                let seq = self.sequences.get_mut(&id).unwrap();
                seq.state = SequenceState::Prefilling;
                seq.kv_cache_slot = Some(slot);
                seq.kv_cache_len = seq.prompt_tokens.len();
            }
        }

        let decoding: Vec<u64> = self.sequences
            .iter()
            .filter(|(_, s)| s.state == SequenceState::Decoding)
            .map(|(id, _)| *id)
            .collect();

        for id in decoding {
            decode_ids.push(id);
        }

        (prefill_ids, decode_ids)
    }

    fn allocate_kv_slot(&mut self, needed: usize) -> Option<usize> {
        let mut start = 0_usize;
        while start < self.kv_cache.len() {
            let mut end = start;
            while end < self.kv_cache.len() && !self.kv_cache[end] {
                end += 1;
            }
            if end - start >= needed {
                for i in start..start + needed {
                    self.kv_cache[i] = true;
                }
                return Some(start);
            }
            start = end + 1;
        }
        None
    }

    pub fn finish_sequence(&mut self, id: u64) {
        if let Some(seq) = self.sequences.get_mut(&id) {
            seq.state = SequenceState::Finished;
            if let Some(slot) = seq.kv_cache_slot.take() {
                for i in slot..slot + seq.kv_cache_len {
                    self.kv_cache[i] = false;
                }
            }
        }
    }
}
```

## 3. 连续批处理 (Continuous Batching) 核心逻辑

```rust
use std::collections::BTreeMap;

#[derive(Debug)]
pub struct ContinuousBatchScheduler {
    pub waiting_queue: BTreeMap<u64, ManagedSequence>,
    pub running_sequences: Vec<ManagedSequence>,
    pub max_running: usize,
}

impl ContinuousBatchScheduler {
    pub fn new(max_running: usize) -> Self {
        Self {
            waiting_queue: BTreeMap::new(),
            running_sequences: Vec::with_capacity(max_running),
            max_running,
        }
    }

    pub fn try_add_sequence(&mut self) {
        while self.running_sequences.len() < self.max_running {
            let next_id = {
                let first = self.waiting_queue.keys().next().copied();
                match first {
                    Some(id) => id,
                    None => break,
                }
            };

            if let Some(seq) = self.waiting_queue.remove(&next_id) {
                self.running_sequences.push(seq);
            }
        }
    }

    pub fn evict_finished(&mut self) -> Vec<ManagedSequence> {
        let (finished, running): (Vec<_>, Vec<_>) = self
            .running_sequences
            .drain(..)
            .partition(|s| s.state == SequenceState::Finished);

        self.running_sequences = running;
        self.try_add_sequence();
        finished
    }

    pub fn running_count(&self) -> usize {
        self.running_sequences.len()
    }

    pub fn joint_decode_step(
        &mut self,
        next_tokens: Vec<u32>,
    ) {
        for (seq, token) in self.running_sequences.iter_mut().zip(next_tokens) {
            seq.generated_tokens.push(token);
            if seq.generated_tokens.len() >= seq.max_new_tokens {
                seq.state = SequenceState::Finished;
            }
        }
    }
}
```

## Red Lines

1. **生产环境必须使用 Continuous Batching，禁止 Static Batching** — Static batching 会阻塞整个 batch 等待最慢的序列完成。continuous batching 允许完成的序列立即退出并接入新序列。
2. **Padding waste 必须 < 15%** — 在 prefill 阶段，batch 内序列长度差异导致的 padding 浪费必须通过长度分组 (bucketing) 控制在 15% 以内。
3. **Prefill 和 Decode 必须分离调度** — prefill 是 compute-bound（矩阵乘法密集），decode 是 memory-bound（逐 token 生成）。混合调度会导致 GPU 利用率剧烈波动。
4. **KV-cache 必须显式管理 slot** — 不能依赖全量预分配超长 context，必须实现 slot-based 分配以支持不同序列长度，避免显存浪费。
5. **调度延迟 (scheduling latency) < 50ms** — 从请求进入等待队列到开始 prefill 的延迟不得超过 50ms，超过时需告警并考虑水平扩容。

## References
- [vLLM — PagedAttention and continuous batching](https://arxiv.org/abs/2309.06180)
- [Sarathi-Serve — Prefill/Decode disaggregation](https://arxiv.org/abs/2403.02310)
- [candle — batched inference examples](https://github.com/huggingface/candle)
- [llama.cpp — continuous batching server](https://github.com/ggerganov/llama.cpp/tree/master/examples/server)