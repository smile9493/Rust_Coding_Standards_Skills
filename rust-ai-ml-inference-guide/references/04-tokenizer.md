# 04 — 分词器 (Tokenizer)

## Status
Verified — HuggingFace tokenizers integration with BPE/WordPiece/Unigram support, special tokens handling (BOS/EOS/PAD), batched encoding with attention masks, and chat template application.

## Prerequisites
- `tokenizers >= 0.19` (HuggingFace Rust bindings)
- `candle-core >= 0.7`, `candle-nn >= 0.7`
- `serde`, `serde_json` for tokenizer config deserialization
- Tokio runtime for async batched encoding

---

## 1. Tokenizer 基础加载与 BPE/WordPiece/Unigram 分类

```rust
use tokenizers::Tokenizer;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum TokenizerType {
    BPE,
    WordPiece,
    Unigram,
    SentencePiece,
}

pub struct TokenizerInspector {
    tokenizer: Tokenizer,
    tokenizer_type: TokenizerType,
    vocab_size: usize,
}

impl TokenizerInspector {
    pub fn load(file_path: &str) -> anyhow::Result<Self> {
        let tokenizer = Tokenizer::from_file(file_path)
            .map_err(|e| anyhow::anyhow!("Failed to load tokenizer: {}", e))?;

        let vocab_size = tokenizer.get_vocab_size(true);

        let tokenizer_type = {
            let json_str = std::fs::read_to_string(file_path)?;
            let config: serde_json::Value = serde_json::from_str(&json_str)?;

            if let Some(model) = config.get("model") {
                match model.get("type").and_then(|v| v.as_str()) {
                    Some("BPE") => TokenizerType::BPE,
                    Some("WordPiece") => TokenizerType::WordPiece,
                    Some("Unigram") => TokenizerType::Unigram,
                    _ => TokenizerType::SentencePiece,
                }
            } else {
                TokenizerType::SentencePiece
            }
        };

        Ok(Self {
            tokenizer,
            tokenizer_type,
            vocab_size,
        })
    }

    pub fn tokenizer_type(&self) -> &TokenizerType {
        &self.tokenizer_type
    }

    pub fn vocab_size(&self) -> usize {
        self.vocab_size
    }
}
```

## 2. Encode/Decode 与 Special Tokens

```rust
use tokenizers::{Encoding, EncodeInput};

#[derive(Debug, Clone)]
pub struct SpecialTokens {
    pub bos_id: u32,
    pub eos_id: u32,
    pub pad_id: u32,
    pub unk_id: u32,
    pub bos_token: String,
    pub eos_token: String,
}

impl SpecialTokens {
    pub fn detect(tokenizer: &Tokenizer, model_name: &str) -> Self {
        let bos_token = match model_name {
            "llama" | "mistral" => "<s>".to_string(),
            "qwen" => "<|im_start|>".to_string(),
            "gemma" => "<bos>".to_string(),
            _ => "<s>".to_string(),
        };

        let eos_token = match model_name {
            "llama" | "mistral" => "</s>".to_string(),
            "qwen" => "<|im_end|>".to_string(),
            "gemma" => "<eos>".to_string(),
            _ => "</s>".to_string(),
        };

        let bos_id = tokenizer.token_to_id(&bos_token).unwrap_or(1);
        let eos_id = tokenizer.token_to_id(&eos_token).unwrap_or(2);
        let pad_id = tokenizer.token_to_id("<pad>").unwrap_or(0);
        let unk_id = tokenizer.token_to_id("<unk>").unwrap_or(0);

        SpecialTokens {
            bos_id,
            eos_id,
            pad_id,
            unk_id,
            bos_token,
            eos_token,
        }
    }

    pub fn wrap_prompt(&self, prompt: &str, add_bos: bool, add_eos: bool) -> String {
        let mut wrapped = String::new();

        if add_bos {
            wrapped.push_str(&self.bos_token);
        }

        wrapped.push_str(prompt);

        if add_eos {
            wrapped.push_str(&self.eos_token);
        }

        wrapped
    }
}
```

## 3. 批处理编码 + Attention Mask 生成

```rust
use candle_core::{Device, Tensor, DType};

pub struct BatchEncoder {
    tokenizer: Tokenizer,
    special_tokens: SpecialTokens,
    max_seq_len: usize,
}

impl BatchEncoder {
    pub fn new(
        tokenizer: Tokenizer,
        special_tokens: SpecialTokens,
        max_seq_len: usize,
    ) -> Self {
        Self {
            tokenizer,
            special_tokens,
            max_seq_len,
        }
    }

    pub fn encode_batch(
        &self,
        texts: &[String],
        device: &Device,
    ) -> anyhow::Result<(Tensor, Tensor)> {
        let batch_size = texts.len();

        let encodings: Vec<Encoding> = texts
            .iter()
            .map(|text| {
                self.tokenizer
                    .encode(text.as_str(), true)
                    .map_err(|e| anyhow::anyhow!("Encode error: {}", e))
            })
            .collect::<anyhow::Result<Vec<_>>>()?;

        let mut input_ids = vec![vec![self.special_tokens.pad_id as i64; self.max_seq_len]; batch_size];
        let mut attention_mask = vec![vec![0_i64; self.max_seq_len]; batch_size];

        for (batch_idx, encoding) in encodings.iter().enumerate() {
            let ids = encoding.get_ids();
            let seq_len = ids.len().min(self.max_seq_len);

            for t in 0..seq_len {
                input_ids[batch_idx][t] = ids[t] as i64;
                attention_mask[batch_idx][t] = 1;
            }
        }

        let input_ids_tensor = Tensor::new(
            input_ids.as_slice(),
            device,
        )?.reshape((batch_size, self.max_seq_len))?;

        let attention_mask_tensor = Tensor::new(
            attention_mask.as_slice(),
            device,
        )?.reshape((batch_size, self.max_seq_len))?;

        Ok((input_ids_tensor, attention_mask_tensor))
    }

    pub fn decode_batch(
        &self,
        token_ids: &[Vec<u32>],
        skip_special_tokens: bool,
    ) -> anyhow::Result<Vec<String>> {
        let results: Vec<String> = token_ids
            .iter()
            .map(|ids| {
                self.tokenizer
                    .decode(ids, skip_special_tokens)
                    .map_err(|e| anyhow::anyhow!("Decode error: {}", e))
            })
            .collect::<anyhow::Result<Vec<_>>>()?;

        Ok(results)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_padding_and_attention_mask() {
        let texts = vec![
            "hello world".to_string(),
            "short".to_string(),
            "this is a longer sequence for testing".to_string(),
        ];

        let max_seq_len = 16;
        let pad_id = 0_i64;

        let mut input_ids = vec![vec![pad_id; max_seq_len]; 3];
        let mut attention_mask = vec![vec![0_i64; max_seq_len]; 3];

        for (i, text) in texts.iter().enumerate() {
            let ids: Vec<i64> = text
                .split_whitespace()
                .enumerate()
                .map(|(j, _)| (j + 1) as i64)
                .collect();

            for (j, &id) in ids.iter().enumerate() {
                if j < max_seq_len {
                    input_ids[i][j] = id;
                    attention_mask[i][j] = 1;
                }
            }
        }

        assert_eq!(attention_mask[0][0], 1);
        assert_eq!(attention_mask[0][2], 0);
        assert_eq!(attention_mask[1][1], 1);
        assert_eq!(attention_mask[2][4], 1);
    }
}
```

## 4. Chat Template 应用

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChatMessage {
    pub role: String,
    pub content: String,
}

pub struct ChatTemplate {
    pub template: String,
    pub special_tokens: SpecialTokens,
}

impl ChatTemplate {
    pub fn llama3_instruct() -> Self {
        Self {
            template: "{{ bos_token }}\
                       <|start_header_id|>system<|end_header_id|>\n\n\
                       {{ system }}\
                       <|eot_id|>\
                       {% for message in messages %}\
                       <|start_header_id|>{{ message['role'] }}<|end_header_id|>\n\n\
                       {{ message['content'] }}<|eot_id|>\
                       {% endfor %}\
                       <|start_header_id|>assistant<|end_header_id|>\n\n"
                .to_string(),
            special_tokens: SpecialTokens {
                bos_id: 128000,
                eos_id: 128001,
                pad_id: 128004,
                unk_id: 0,
                bos_token: "<|begin_of_text|>".to_string(),
                eos_token: "<|end_of_text|>".to_string(),
            },
        }
    }

    pub fn apply(
        &self,
        messages: &[ChatMessage],
        system_prompt: Option<&str>,
    ) -> String {
        let system = system_prompt.unwrap_or("You are a helpful assistant.");

        let mut prompt = String::new();
        prompt.push_str(&self.special_tokens.bos_token);
        prompt.push_str(&format!(
            "<|start_header_id|>system<|end_header_id|>\n\n{}<|eot_id|>",
            system
        ));

        for msg in messages {
            prompt.push_str(&format!(
                "<|start_header_id|>{}<|end_header_id|>\n\n{}<|eot_id|>",
                msg.role, msg.content
            ));
        }

        prompt.push_str("<|start_header_id|>assistant<|end_header_id|>\n\n");
        prompt
    }
}
```

## Red Lines

1. **Tokenizer 必须与模型架构严格配对** — LLaMA tokenizer 不可用于 Mistral 模型，即使两者都使用 BPE。tokenizer.json 文件必须来自同一模型发布版本。
2. **BOS token 默认必须添加** — 未经 BOS token 前缀的序列会导致模型在首 token 位置的行为完全异常，perplexity 飙升 10-100 倍。
3. **pad_token_id 不同于 eos_token_id** — 使用 eos 作为 pad 会导致 attention mask 失效，模型会错误地 attend 到 padding 位置。必须设置独立的 PAD token。
4. **Batched encoding 中的 attention_mask 不可省略** — 没有 attention mask 时，左侧 padding 会导致模型把 padding token 当作有效输入。
5. **decode 时必须 `skip_special_tokens=true`** — 否则输出中会包含 `<s>`、`</s>`、`<|eot_id|>` 等控制 token，破坏用户体验。

## References
- [tokenizers — HuggingFace Rust API](https://docs.rs/tokenizers)
- [HuggingFace — Tokenizer summary](https://huggingface.co/docs/transformers/tokenizer_summary)
- [LLaMA 3 — Chat template specification](https://llama.meta.com/docs/model-cards-and-prompt-formats/meta-llama-3/)
- [Mistral — Tokenizer documentation](https://docs.mistral.ai/guides/tokenization/)