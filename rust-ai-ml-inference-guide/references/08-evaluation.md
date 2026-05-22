# 08 — Evaluation & Benchmarking

**Status**: Mandatory | **Prerequisites**: 02-quantization.md, 05-batching.md | **Parent**: rust-architecture-guide P0 Safety

## Overview

Every model serving pipeline must be evaluated for both quality (output correctness) and performance (throughput/latency). Quantization is a trade-off — measure it before deploying.

## Quality Metrics

### Perplexity: Language Model Quality

Perplexity measures how "surprised" a model is by a text. Lower = better. Commonly evaluated on WikiText-2, PTB, or C4 datasets.

```rust
use candle_core::{Device, Tensor};
use candle_nn::VarBuilder;

fn compute_perplexity<M: LanguageModel>(
    model: &M,
    dataset: &[Vec<u32>],
    device: &Device,
) -> f64 {
    let mut total_nll: f64 = 0.0;
    let mut total_tokens: u64 = 0;

    for sequence in dataset {
        let input = Tensor::new(&sequence[..sequence.len() - 1], device).unwrap();
        let target = Tensor::new(&sequence[1..], device).unwrap();

        let logits = model.forward(&input).unwrap();
        let nll = candle_nn::loss::cross_entropy(&logits, &target).unwrap();
        let nll_scalar: f32 = nll.to_scalar().unwrap();

        total_nll += nll_scalar as f64 * (sequence.len() - 1) as f64;
        total_tokens += (sequence.len() - 1) as u64;
    }

    (total_nll / total_tokens as f64).exp()
}
```

**Acceptance Criteria**:
- FP16 baseline: perplexity P_base
- Q8_0 quantized: P <= P_base * 1.02 (2% degradation)
- Q4_K_M quantized: P <= P_base * 1.05 (5% degradation)
- Q4_0 quantized: P <= P_base * 1.08 (8% degradation, acceptable for low-resource)

### BLEU/ROUGE: Generation Quality

For summarization/translation tasks:
```rust
fn evaluate_generation(model: &dyn Generator, test_pairs: &[(String, String)]) -> f64 {
    let mut total_bleu: f64 = 0.0;
    for (input, reference) in test_pairs {
        let generated = model.generate(input);
        let bleu_score = compute_sentence_bleu(reference, &generated);
        total_bleu += bleu_score;
    }
    total_bleu / test_pairs.len() as f64
}
```

## Performance Metrics

### Throughput: Tokens per Second

Measure at multiple batch sizes. GPU and CPU separately.

```rust
use std::time::Instant;

#[derive(Debug, Clone)]
struct ThroughputResult {
    batch_size: usize,
    prompt_tokens: usize,
    generated_tokens: usize,
    elapsed_secs: f64,
    tokens_per_second: f64,
}

fn benchmark_throughput(
    model: &dyn Generator,
    prompt: &str,
    max_new_tokens: usize,
    batch_sizes: &[usize],
) -> Vec<ThroughputResult> {
    let mut results = Vec::new();

    for &batch_size in batch_sizes {
        let prompts: Vec<String> = vec![prompt.to_string(); batch_size];

        let tokenizer = model.tokenizer();
        let prompt_tokens: usize = prompts.iter()
            .map(|p| tokenizer.encode(p, false).unwrap().len())
            .sum();

        let start = Instant::now();
        let outputs = model.generate_batch(&prompts, max_new_tokens);
        let elapsed = start.elapsed().as_secs_f64();

        let generated_tokens: usize = outputs.iter()
            .map(|o| o.len())
            .sum();

        results.push(ThroughputResult {
            batch_size,
            prompt_tokens,
            generated_tokens,
            elapsed_secs: elapsed,
            tokens_per_second: generated_tokens as f64 / elapsed,
        });
    }

    results
}
```

### TTFT: Time to First Token

Critical for UX. Users perceive responsiveness through TTFT.

```rust
fn measure_ttft(model: &dyn Generator, prompt: &str) -> std::time::Duration {
    let tokenizer = model.tokenizer();
    let tokens = tokenizer.encode(prompt, false).unwrap();

    // Prefill phase
    let start = Instant::now();
    model.prefill(&tokens);

    // First decode step
    let first_token = model.decode_step();
    let ttft = start.elapsed();

    println!("TTFT: {:?} for {} prompt tokens", ttft, tokens.len());
    ttft
}
```

**Acceptance Criteria**:
- TTFT < 100ms for prompts < 512 tokens (GPU)
- TTFT < 500ms for prompts < 512 tokens (CPU)
- TTFT < 2s for prompts < 4096 tokens (GPU)

### Inter-Token Latency

Time between consecutive generated tokens:
```rust
fn measure_inter_token_latency(model: &dyn Generator, prompt: &str, n: usize) -> Vec<f64> {
    let mut latencies = Vec::with_capacity(n);
    model.prefill_from_str(prompt);

    for _ in 0..n {
        let start = Instant::now();
        model.decode_step();
        latencies.push(start.elapsed().as_secs_f64() * 1000.0);
    }

    latencies
}
```
Target: median < 20ms per token on GPU.

## Benchmarking Matrix

Run this matrix before every deployment:

| Dimension | Variations | Metric |
|-----------|-----------|--------|
| Precision | FP16, Q8_0, Q4_K_M, Q4_0 | Perplexity |
| Batch Size | 1, 4, 8, 16, 32 | Tokens/sec |
| Context Length | 512, 1024, 2048, 4096 | TTFT |
| Device | GPU (CUDA), GPU (Metal), CPU | All metrics |

## Regression Testing

Automate in CI:

```rust
#[cfg(test)]
mod bench_tests {
    use super::*;

    #[test]
    fn test_perplexity_within_tolerance() {
        let fp16_perplexity = load_benchmark("fp16", "wikitext2");
        let q4_perplexity = load_benchmark("q4_k_m", "wikitext2");

        let degradation = (q4_perplexity - fp16_perplexity) / fp16_perplexity;
        assert!(
            degradation < 0.05,
            "Q4_K_M perplexity degradation {:.2}% exceeds 5% threshold",
            degradation * 100.0
        );
    }

    #[test]
    fn test_ttft_within_sla() {
        let ttft = measure_ttft_for_prompt("Explain quantum computing", 256);
        assert!(
            ttft < 500.0,
            "TTFT {}ms exceeds 500ms SLA for 256-token prompt",
            ttft
        );
    }

    #[test]
    fn test_throughput_minimum() {
        let tps = benchmark_throughput_at_batch(8, 128);
        assert!(tps > 50.0, "Throughput {:.1} tok/s below 50 minimum", tps);
    }
}
```

## Tooling

| Tool | Purpose |
|------|---------|
| `perplexity_bench` | Standard perplexity evaluation across datasets |
| `llama-perplexity` (llama.cpp) | GGUF-specific perplexity evaluation |
| `nvtop` / `nvidia-smi` | GPU VRAM and utilization monitoring |
| `cargo bench` (criterion) | Micro-benchmarks for individual ops |
| `tokio-console` | Async task latency visualization |

## Red Lines

| Category | Prohibited | Mandatory |
|----------|------------|-----------|
| Quantization | Deploy without perplexity benchmark | < 5% degradation for Q4_K_M vs FP16 |
| TTFT | No SLA for first token latency | < 100ms GPU, < 500ms CPU for 512 tokens |
| Throughput | Single-batch measurement only | Benchmark at 1, 4, 8, 16 batch sizes |
| Regression | No automated quality gate in CI | Perplexity + throughput tests as CI gates |
| Context | Measure only at default context | Test at 512, 1024, 2048, 4096 lengths |
| Hardware | Assume uniform performance | Test on target deployment hardware |
| Degradation | Accept > 8% perplexity increase | Reject or use higher-precision quantization |
| Monitoring | Deploy without runtime metrics | Track TTFT and tps in production dashboards |

## References
- [llama.cpp perplexity evaluation](https://github.com/ggerganov/llama.cpp/tree/master/examples/perplexity)
- [candle evaluation examples](https://github.com/huggingface/candle/tree/main/candle-examples)
- [Criterion.rs benchmarks](https://bheisler.github.io/criterion.rs/book/)
- [GGUF quantization type reference](https://github.com/ggerganov/llama.cpp/pull/1684)
- [vLLM performance benchmarks](https://docs.vllm.ai/en/latest/performance/)