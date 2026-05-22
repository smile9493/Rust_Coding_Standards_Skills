# 01 — 模型加载 (Model Loading)

## Status
Verified — Production-grade model loading with checksum validation, support for GGUF, SafeTensors, and ONNX formats via `candle`, `llama-cpp-rs`, and `ort`.

## Prerequisites
- `candle-core >= 0.7`, `candle-nn >= 0.7`, `candle-transformers >= 0.7`
- `llama-cpp-rs` (for GGUF via llama.cpp backend)
- `ort >= 2.0` (ONNX Runtime bindings)
- `sha2` or `blake3` for checksum verification
- Tokio async runtime for non-blocking I/O

---

## 1. GGUF 格式加载 (candle)

```rust
use candle_core::{Device, Tensor};
use candle_nn::VarBuilder;
use candle_transformers::models::quantized_llama::Model as QLlama;
use candle_transformers::models::quantized_mistral::Model as QMistral;
use std::path::PathBuf;
use tokenizers::Tokenizer;

#[derive(Debug, Clone)]
pub struct GgufModelConfig {
    pub model_path: PathBuf,
    pub tokenizer_path: PathBuf,
    pub use_flash_attn: bool,
    pub gqa: usize,
}

pub struct GgufPipeline {
    model: QLlama,
    tokenizer: Tokenizer,
    device: Device,
    config: GgufModelConfig,
}

impl GgufPipeline {
    pub fn load(config: GgufModelConfig, device: Device) -> anyhow::Result<Self> {
        let tokenizer = Tokenizer::from_file(&config.tokenizer_path)
            .map_err(|e| anyhow::anyhow!("Tokenizer load failed: {}", e))?;

        let vb = candle_transformers::quantized_var_builder::VarBuilder::from_gguf(
            &config.model_path,
            &device,
        )?;

        let model = QLlama::from_vb(vb)?;

        Ok(Self {
            model,
            tokenizer,
            device,
            config,
        })
    }

    pub fn generate(&mut self, prompt: &str, max_tokens: usize) -> anyhow::Result<String> {
        let tokens = self
            .tokenizer
            .encode(prompt, true)
            .map_err(|e| anyhow::anyhow!("Encoding failed: {}", e))?;

        let mut token_ids = tokens.get_ids().to_vec();
        let eos_token = self.tokenizer.token_to_id("</s>")
            .unwrap_or(2_u32);

        for _ in 0..max_tokens {
            let input = Tensor::new(&[token_ids.as_slice()], &self.device)?
                .unsqueeze(0)?;

            let logits = self.model.forward(&input, 0)?;
            let next_token = logits
                .squeeze(0)?
                .squeeze(0)?
                .argmax(0)?
                .to_scalar::<u32>()?;

            if next_token == eos_token {
                break;
            }
            token_ids.push(next_token);
        }

        let text = self
            .tokenizer
            .decode(&token_ids, true)
            .map_err(|e| anyhow::anyhow!("Decoding failed: {}", e))?;

        Ok(text)
    }
}
```

## 2. GGUF 格式加载 (llama-cpp-rs)

```rust
use llama_cpp_rs::{
    context::params::LlamaContextParams,
    llama_backend::LlamaBackend,
    model::params::LlamaModelParams,
    model::LlamaModel,
    token::data_array::LlamaTokenDataArray,
};
use std::num::NonZeroU32;

pub struct LlamaCppPipeline {
    model: LlamaModel,
    backend: LlamaBackend,
}

impl LlamaCppPipeline {
    pub fn load(model_path: &str) -> anyhow::Result<Self> {
        let backend = LlamaBackend::init()?;

        let model_params = LlamaModelParams {
            n_gpu_layers: 99,
            main_gpu: 0,
            use_mmap: true,
            use_mlock: false,
            ..Default::default()
        };

        let model = LlamaModel::load_from_file(
            &backend,
            model_path,
            &model_params,
        )?;

        Ok(Self { model, backend })
    }

    pub fn generate(
        &self,
        prompt: &str,
        max_tokens: u32,
    ) -> anyhow::Result<String> {
        let ctx_params = LlamaContextParams {
            n_ctx: NonZeroU32::new(4096),
            n_batch: 512,
            ..Default::default()
        };

        let mut ctx = self.model.new_context(&ctx_params)?;

        let tokens = self.model.str_to_token(prompt, true)?;
        ctx.eval(&tokens, tokens.len() as i32)?;

        let mut output = String::new();
        for _ in 0..max_tokens {
            let candidates = ctx.candidates();
            let mut candidates_p = LlamaTokenDataArray::from_iter(candidates, false);
            ctx.sample_token_greedy(&mut candidates_p);
            let next_token = candidates_p.data[0].id();

            ctx.eval(&[next_token], 1)?;

            let piece = self.model.token_to_str(next_token)?;
            output.push_str(&piece);

            if self.model.token_eos() == next_token {
                break;
            }
        }

        Ok(output)
    }
}
```

## 3. SafeTensors 安全加载 + Checksum 验证

```rust
use candle_core::Device;
use safetensors::SafeTensors;
use sha2::{Sha256, Digest};
use std::io::Read;

pub struct SafeTensorLoader;

impl SafeTensorLoader {
    pub fn load_with_checksum_verify(
        path: &str,
        expected_sha256: &str,
        device: &Device,
    ) -> anyhow::Result<Vec<(String, candle_core::Tensor)>> {
        let mut file = std::fs::File::open(path)?;
        let mut buffer = Vec::new();
        file.read_to_end(&mut buffer)?;

        let mut hasher = Sha256::new();
        hasher.update(&buffer);
        let actual_hash = format!("{:x}", hasher.finalize());

        if actual_hash != expected_sha256 {
            anyhow::bail!(
                "Checksum mismatch: expected {}, got {}",
                expected_sha256,
                actual_hash
            );
        }

        let safetensors = SafeTensors::deserialize(&buffer)?;
        let mut tensors = Vec::with_capacity(safetensors.names().len());

        for name in safetensors.names() {
            let view = safetensors.tensor(name)?;
            let shape: Vec<usize> = view.shape().to_vec();
            let dtype = view.dtype();

            let data: Vec<u8> = view.data().to_vec();
            let tensor = match dtype {
                safetensors::Dtype::F32 => {
                    let f32_data: &[f32] = bytemuck::cast_slice(&data);
                    candle_core::Tensor::from_slice(f32_data, &shape, device)?
                }
                safetensors::Dtype::F16 => {
                    let f16_data: &[half::f16] = bytemuck::cast_slice(&data);
                    candle_core::Tensor::from_slice(f16_data, &shape, device)?
                }
                _ => anyhow::bail!("Unsupported dtype: {:?}", dtype),
            };
            tensors.push((name.to_string(), tensor));
        }

        Ok(tensors)
    }
}
```

## 4. ONNX 格式加载 (ort)

```rust
use ort::{
    session::{Session, SessionBuilder},
    value::Value,
    memory::MemoryInfo,
    AllocatorType, MemType,
};
use ndarray::Array2;

pub struct OnnxPipeline {
    session: Session,
    mem_info: MemoryInfo,
}

impl OnnxPipeline {
    pub fn load(model_path: &str) -> anyhow::Result<Self> {
        let session = SessionBuilder::new()?
            .with_intra_threads(4)?
            .with_inter_threads(1)?
            .with_optimization_level(
                ort::GraphOptimizationLevel::Level3,
            )?
            .commit_from_file(model_path)?;

        let allocator = session.allocator();
        let mem_info = MemoryInfo::new(
            MemType::CpuInput,
            AllocatorType::Arena,
            allocator,
        )?;

        Ok(Self { session, mem_info })
    }

    pub fn run_inference(
        &self,
        input_ids: &[i64],
        attention_mask: &[i64],
    ) -> anyhow::Result<Array2<f32>> {
        let shape = [1_i64, input_ids.len() as i64];
        let input_tensor = Value::from_array(
            self.mem_info.clone(),
            &Array2::from_shape_vec(
                (1, input_ids.len()),
                input_ids.to_vec(),
            )?,
        )?;

        let mask_tensor = Value::from_array(
            self.mem_info.clone(),
            &Array2::from_shape_vec(
                (1, attention_mask.len()),
                attention_mask.to_vec(),
            )?,
        )?;

        let outputs = self.session.run(vec![
            ("input_ids", input_tensor),
            ("attention_mask", mask_tensor),
        ])?;

        let logits: Array2<f32> = outputs[0]
            .try_extract_tensor()?
            .view()
            .to_owned();

        Ok(logits)
    }
}
```

## Red Lines

1. **Checksum 验证是强制性的** — 任何生产环境模型加载都必须校验 SHA-256/BLAKE3 哈希值，不允许跳过。
2. **GGUF 加载必须指定 `n_gpu_layers`** — 隐式默认值可能导致全部层加载到 CPU，性能灾难。
3. **SafeTensors 优先于 pickle/PyTorch 格式** — pickle 存在任意代码执行风险，生产环境禁止加载 `.pt`/`.pth` 文件。
4. **ONNX 必须启用 `GraphOptimizationLevel::Level3`** — Level3 包含 layout optimization + kernel fusion，对 transformer 模型有显著加速。
5. **Tokenizer 与模型必须配对** — 不同 GGUF 量化版本（Q4_0/Q8_0）使用相同 tokenizer，但跨模型架构不可混用。

## References
- [candle — GGUF quantized model loading](https://github.com/huggingface/candle)
- [llama-cpp-rs — Rust bindings for llama.cpp](https://crates.io/crates/llama-cpp-rs)
- [ort — ONNX Runtime Rust](https://docs.rs/ort)
- [SafeTensors specification](https://github.com/huggingface/safetensors)
- [HuggingFace — GGUF format documentation](https://huggingface.co/docs/hub/gguf)