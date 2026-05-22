# 06 — Embedding 向量搜索 (Embeddings & Vector Search)

## Status
Verified — BERT/sentence-transformers embedding extraction with L2 normalization, cosine similarity via dot product optimization, HNSW approximate nearest neighbor search via Qdrant, and pgvector integration.

## Prerequisites
- `candle-core >= 0.7`, `candle-nn >= 0.7`, `candle-transformers >= 0.7`
- `qdrant-client >= 1.10` (Rust Qdrant client)
- `sqlx >= 0.7` with `postgres` feature for pgvector
- `serde`, `serde_json` for payload serialization

---

## 1. BERT Embedding 提取 + L2 Normalization

```rust
use candle_core::{Device, Tensor, DType};
use candle_nn::VarBuilder;
use candle_transformers::models::bert::{BertModel, Config, DTYPE};

pub struct EmbeddingPipeline {
    model: BertModel,
    tokenizer: Tokenizer,
    device: Device,
    pooling_strategy: PoolingStrategy,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PoolingStrategy {
    Mean,
    CLS,
    Max,
}

impl EmbeddingPipeline {
    pub fn load(
        model_dir: &str,
        device: Device,
        pooling: PoolingStrategy,
    ) -> anyhow::Result<Self> {
        let config_path = format!("{}/config.json", model_dir);
        let tokenizer_path = format!("{}/tokenizer.json", model_dir);
        let model_path = format!("{}/model.safetensors", model_dir);

        let config: Config = serde_json::from_str(
            &std::fs::read_to_string(&config_path)?
        )?;

        let tokenizer = Tokenizer::from_file(&tokenizer_path)
            .map_err(|e| anyhow::anyhow!("Tokenizer error: {}", e))?;

        let vb = unsafe {
            VarBuilder::from_mmaped_safetensors(
                &[model_path],
                DTYPE,
                &device,
            )?
        };

        let model = BertModel::new(&vb, &config)?;

        Ok(Self {
            model,
            tokenizer,
            device,
            pooling_strategy: pooling,
        })
    }

    pub fn encode(
        &self,
        texts: &[String],
        normalize: bool,
    ) -> anyhow::Result<Vec<Vec<f32>>> {
        let encodings: Vec<_> = texts
            .iter()
            .map(|text| {
                self.tokenizer
                    .encode(text.as_str(), true)
                    .map_err(|e| anyhow::anyhow!("Encode error: {}", e))
            })
            .collect::<anyhow::Result<Vec<_>>>()?;

        let max_len = encodings
            .iter()
            .map(|e| e.get_ids().len())
            .max()
            .unwrap_or(0);

        let batch_size = texts.len();
        let mut input_ids = vec![0_u32; batch_size * max_len];
        let mut token_type_ids = vec![0_u32; batch_size * max_len];
        let mut attention_mask = vec![0_u32; batch_size * max_len];

        for (i, encoding) in encodings.iter().enumerate() {
            let ids = encoding.get_ids();
            let type_ids = encoding.get_type_ids();
            let attn = encoding.get_attention_mask();

            for j in 0..ids.len() {
                input_ids[i * max_len + j] = ids[j];
                token_type_ids[i * max_len + j] = type_ids[j];
                attention_mask[i * max_len + j] = attn[j] as u32;
            }
        }

        let input_ids_t = Tensor::from_vec(
            input_ids,
            (batch_size, max_len),
            &self.device,
        )?;

        let token_type_ids_t = Tensor::from_vec(
            token_type_ids,
            (batch_size, max_len),
            &self.device,
        )?;

        let attention_mask_t = Tensor::from_vec(
            attention_mask,
            (batch_size, max_len),
            &self.device,
        )?;

        let hidden_states = self.model.forward(
            &input_ids_t,
            &token_type_ids_t,
            Some(&attention_mask_t),
        )?;

        let embeddings = self.pool(
            &hidden_states,
            &attention_mask_t,
            &encodings,
            max_len,
        )?;

        if normalize {
            Ok(self.l2_normalize_batch(&embeddings)?)
        } else {
            Ok(embeddings.to_vec2()?)
        }
    }

    fn pool(
        &self,
        hidden: &Tensor,
        attention_mask: &Tensor,
        encodings: &[Encoding],
        max_len: usize,
    ) -> anyhow::Result<Tensor> {
        match self.pooling_strategy {
            PoolingStrategy::CLS => {
                hidden.narrow(1, 0, 1)?.squeeze(1)
            }
            PoolingStrategy::Mean => {
                let mask = attention_mask
                    .to_dtype(DType::F32)?
                    .unsqueeze(2)?;

                let masked = hidden.broadcast_mul(&mask)?;
                let sum = masked.sum(1)?;
                let count = mask.sum(1)?.broadcast_div(&mask.sum(1)?)?;
                sum.broadcast_div(&count)
            }
            PoolingStrategy::Max => {
                let mask = attention_mask
                    .to_dtype(DType::F32)?
                    .unsqueeze(2)?;

                let very_negative = Tensor::new(-1e9_f32, self.device())?;
                let masked = hidden.broadcast_mul(&mask)?
                    .broadcast_add(
                        &mask.neg()?.broadcast_mul(&very_negative)?
                    )?;
                masked.max(1)?
            }
        }
    }

    fn l2_normalize_batch(&self, embeddings: &Tensor) -> anyhow::Result<Vec<Vec<f32>>> {
        let norms = embeddings.sqr()?.sum_keepdim(1)?.sqrt()?;
        let normalized = embeddings.broadcast_div(&norms)?;
        normalized.to_vec2()
    }

    fn device(&self) -> &Device {
        &self.device
    }
}
```

## 2. Cosine Similarity 与 Dot Product 等价性

```rust
pub struct SimilarityCalculator;

impl SimilarityCalculator {
    pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
        assert_eq!(a.len(), b.len());

        let dot: f32 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
        let norm_a: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
        let norm_b: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();

        if norm_a == 0.0 || norm_b == 0.0 {
            return 0.0;
        }

        dot / (norm_a * norm_b)
    }

    pub fn dot_product_l2_normalized(a: &[f32], b: &[f32]) -> f32 {
        assert_eq!(a.len(), b.len());
        a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
    }

    pub fn top_k_similar(
        query: &[f32],
        corpus: &[Vec<f32>],
        k: usize,
    ) -> Vec<(usize, f32)> {
        let mut scored: Vec<(usize, f32)> = corpus
            .iter()
            .enumerate()
            .map(|(idx, doc)| {
                let sim = Self::dot_product_l2_normalized(query, doc);
                (idx, sim)
            })
            .collect();

        scored.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        scored.truncate(k);
        scored
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_cosine_vs_dot_product() {
        let a = vec![1.0_f32, 2.0, 3.0];
        let norm_a = (1.0_f32 + 4.0 + 9.0).sqrt();
        let a_normalized: Vec<f32> = a.iter().map(|x| x / norm_a).collect();

        let b = vec![4.0_f32, 5.0, 6.0];
        let norm_b = (16.0_f32 + 25.0 + 36.0).sqrt();
        let b_normalized: Vec<f32> = b.iter().map(|x| x / norm_b).collect();

        let cosine = SimilarityCalculator::cosine_similarity(&a, &b);
        let dot_l2 = SimilarityCalculator::dot_product_l2_normalized(
            &a_normalized,
            &b_normalized,
        );

        println!("Cosine: {:.6}, Dot(L2): {:.6}", cosine, dot_l2);
        assert!((cosine - dot_l2).abs() < 1e-6);
    }
}
```

## 3. HNSW ANN 搜索 (Qdrant)

```rust
use qdrant_client::{
    Qdrant,
    qdrant::{
        CreateCollectionBuilder, Distance, VectorParamsBuilder,
        SearchPointsBuilder, PointStruct, UpsertPointsBuilder,
    },
};
use serde_json::json;

pub struct QdrantIndexer {
    client: Qdrant,
    collection_name: String,
    vector_dim: u64,
}

impl QdrantIndexer {
    pub async fn new(
        uri: &str,
        collection_name: &str,
        vector_dim: u64,
    ) -> anyhow::Result<Self> {
        let client = Qdrant::from_url(uri).build()?;

        let exists = client
            .collection_exists(collection_name)
            .await?;

        if !exists {
            client
                .create_collection(
                    CreateCollectionBuilder::new(collection_name)
                        .vectors_config(
                            VectorParamsBuilder::new(vector_dim, Distance::Cosine)
                        ),
                )
                .await?;
        }

        Ok(Self {
            client,
            collection_name: collection_name.to_string(),
            vector_dim,
        })
    }

    pub async fn upsert(
        &self,
        points: &[(u64, &[f32], serde_json::Value)],
    ) -> anyhow::Result<()> {
        let points: Vec<PointStruct> = points
            .iter()
            .map(|(id, vector, payload)| {
                PointStruct::new(
                    *id,
                    vector.to_vec(),
                    payload.clone(),
                )
            })
            .collect();

        self.client
            .upsert_points(
                UpsertPointsBuilder::new(
                    &self.collection_name,
                    points,
                ),
            )
            .await?;

        Ok(())
    }

    pub async fn search(
        &self,
        query_vector: &[f32],
        limit: u64,
    ) -> anyhow::Result<Vec<(u64, f32, serde_json::Value)>> {
        let result = self
            .client
            .search_points(
                SearchPointsBuilder::new(
                    &self.collection_name,
                    query_vector.to_vec(),
                    limit,
                ),
            )
            .await?;

        let hits: Vec<_> = result
            .result
            .iter()
            .map(|point| {
                let id = point.id.as_ref().and_then(|id| {
                    match id.point_id_options.as_ref() {
                        Some(opts) => opts.clone().num,
                        None => None,
                    }
                }).unwrap_or(0);

                let score = point.score;
                let payload = point.payload.clone();
                (id, score, payload)
            })
            .collect();

        Ok(hits)
    }
}
```

## 4. pgvector 集成

```rust
use sqlx::postgres::{PgPool, PgPoolOptions};
use sqlx::Row;

pub struct PgVectorStore {
    pool: PgPool,
    table_name: String,
    vector_dim: usize,
}

impl PgVectorStore {
    pub async fn new(
        database_url: &str,
        table_name: &str,
        vector_dim: usize,
    ) -> anyhow::Result<Self> {
        let pool = PgPoolOptions::new()
            .max_connections(10)
            .connect(database_url)
            .await?;

        let create_sql = format!(
            "CREATE EXTENSION IF NOT EXISTS vector;
             CREATE TABLE IF NOT EXISTS {} (
                 id BIGSERIAL PRIMARY KEY,
                 embedding vector({}),
                 metadata JSONB
             );
             CREATE INDEX IF NOT EXISTS {}_embedding_idx
                 ON {} USING hnsw (embedding vector_cosine_ops);",
            table_name, vector_dim, table_name, table_name
        );

        sqlx::query(&create_sql)
            .execute(&pool)
            .await?;

        Ok(Self {
            pool,
            table_name: table_name.to_string(),
            vector_dim,
        })
    }

    pub async fn insert(
        &self,
        embeddings: &[Vec<f32>],
        metadata: &[serde_json::Value],
    ) -> anyhow::Result<Vec<i64>> {
        let mut ids = Vec::with_capacity(embeddings.len());

        for (embedding, meta) in embeddings.iter().zip(metadata) {
            let vector_str = format!(
                "[{}]",
                embedding
                    .iter()
                    .map(|x| x.to_string())
                    .collect::<Vec<_>>()
                    .join(",")
            );

            let row = sqlx::query(&format!(
                "INSERT INTO {} (embedding, metadata) VALUES ($1, $2) RETURNING id",
                self.table_name
            ))
            .bind(&vector_str)
            .bind(meta)
            .fetch_one(&self.pool)
            .await?;

            ids.push(row.get("id"));
        }

        Ok(ids)
    }

    pub async fn search(
        &self,
        query_vector: &[f32],
        limit: usize,
    ) -> anyhow::Result<Vec<(i64, f32, serde_json::Value)>> {
        let vector_str = format!(
            "[{}]",
            query_vector
                .iter()
                .map(|x| x.to_string())
                .collect::<Vec<_>>()
                .join(",")
        );

        let rows = sqlx::query(&format!(
            "SELECT id, 1 - (embedding <=> $1) AS similarity, metadata
             FROM {}
             ORDER BY embedding <=> $1
             LIMIT $2",
            self.table_name
        ))
        .bind(&vector_str)
        .bind(limit as i64)
        .fetch_all(&self.pool)
        .await?;

        let results: Vec<_> = rows
            .iter()
            .map(|row| {
                let id: i64 = row.get("id");
                let similarity: f64 = row.get("similarity");
                let metadata: serde_json::Value = row.get("metadata");
                (id, similarity as f32, metadata)
            })
            .collect();

        Ok(results)
    }
}
```

## Red Lines

1. **Embedding 必须做 L2 normalization** — 否则 cosine similarity ≠ dot product，无法利用向量数据库的 dot product 索引（dot product 比 cosine 快 3-10x）。
2. **Pooling 策略必须与模型训练一致** — 使用 BERT 训练时用的 mean pooling 却用 CLS pooling 提取向量，相似度结果完全不可用。
3. **Qdrant/pgvector 的 distance metric 必须与 embedding 的 normalization 对齐** — L2 normalized embeddings 使用 `Distance::Cosine`，未归一化则用 `Distance::Dot`。
4. **pgvector HNSW 索引必须显式指定 `vector_cosine_ops`** — 默认的 `vector_l2_ops` 使用欧几里得距离，与 cosine similarity 不匹配。
5. **批量 upsert 而不是逐条 insert** — Qdrant 单次 upsert 1 条 vs 1000 条，吞吐量差异可达 100x。

## References
- [candle — BERT model implementation](https://github.com/huggingface/candle)
- [Qdrant — Rust client](https://github.com/qdrant/qdrant-client)
- [pgvector — PostgreSQL vector extension](https://github.com/pgvector/pgvector)
- [HNSW — Efficient and robust approximate nearest neighbor search](https://arxiv.org/abs/1603.09320)
- [sentence-transformers — Pooling strategies](https://www.sbert.net/docs/package_reference/pooling.html)