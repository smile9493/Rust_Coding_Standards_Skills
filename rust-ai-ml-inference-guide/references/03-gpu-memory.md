# 03 — GPU 显存管理 (GPU Memory)

## Status
Verified — KV-cache pre-allocation with OOM backpressure, layer offloading via `n_gpu_layers`, VRAM monitoring with CUDA/nvml, and memory pool design for inference serving.

## Prerequisites
- `candle-core >= 0.7` with CUDA/Metal backend
- `nvml-wrapper` or `sysinfo` for VRAM monitoring
- `llama-cpp-rs` for layer offloading demonstration
- CUDA >= 12.0 or ROCm for GPU acceleration

---

## 1. KV-Cache 预分配与容量管理

```rust
use candle_core::{Device, Tensor, DType};

#[derive(Debug, Clone)]
pub struct KVCacheConfig {
    pub max_seq_len: usize,
    pub n_layers: usize,
    pub n_heads: usize,
    pub n_kv_heads: usize,
    pub head_dim: usize,
    pub dtype_size: usize,
}

pub struct KVCache {
    pub k_caches: Vec<Tensor>,
    pub v_caches: Vec<Tensor>,
    pub current_len: usize,
    pub config: KVCacheConfig,
}

impl KVCache {
    pub fn new(config: KVCacheConfig, device: &Device) -> anyhow::Result<Self> {
        let cache_shape = (
            1,
            config.max_seq_len,
            config.n_kv_heads,
            config.head_dim,
        );

        let mut k_caches = Vec::with_capacity(config.n_layers);
        let mut v_caches = Vec::with_capacity(config.n_layers);

        for _ in 0..config.n_layers {
            k_caches.push(Tensor::zeros(cache_shape, DType::F16, device)?);
            v_caches.push(Tensor::zeros(cache_shape, DType::F16, device)?);
        }

        Ok(Self {
            k_caches,
            v_caches,
            current_len: 0,
            config,
        })
    }

    pub fn estimated_vram_bytes(&self) -> usize {
        let per_layer_bytes = self.config.max_seq_len
            * self.config.n_kv_heads
            * self.config.head_dim
            * self.config.dtype_size
            * 2;

        per_layer_bytes * self.config.n_layers
    }

    pub fn append(
        &mut self,
        layer_idx: usize,
        k_new: &Tensor,
        v_new: &Tensor,
    ) -> anyhow::Result<()> {
        let seq_len = k_new.dim(1)?;

        if self.current_len + seq_len > self.config.max_seq_len {
            anyhow::bail!(
                "KV-cache overflow: current={}, new={}, max={}",
                self.current_len,
                seq_len,
                self.config.max_seq_len
            );
        }

        let k_slice = self.k_caches[layer_idx]
            .narrow(1, self.current_len, seq_len)?;
        k_slice.copy_from(k_new)?;

        let v_slice = self.v_caches[layer_idx]
            .narrow(1, self.current_len, seq_len)?;
        v_slice.copy_from(v_new)?;

        if layer_idx == self.config.n_layers - 1 {
            self.current_len += seq_len;
        }

        Ok(())
    }

    pub fn get_context(&self, layer_idx: usize) -> anyhow::Result<(Tensor, Tensor)> {
        let k = self.k_caches[layer_idx]
            .narrow(1, 0, self.current_len)?;
        let v = self.v_caches[layer_idx]
            .narrow(1, 0, self.current_len)?;
        Ok((k, v))
    }

    pub fn reset(&mut self) {
        self.current_len = 0;
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_kv_cache_vram_estimation() {
        let config = KVCacheConfig {
            max_seq_len: 4096,
            n_layers: 32,
            n_heads: 32,
            n_kv_heads: 8,
            head_dim: 128,
            dtype_size: 2,
        };

        let vram_bytes = config.max_seq_len
            * config.n_kv_heads
            * config.head_dim
            * config.dtype_size
            * 2
            * config.n_layers;

        println!(
            "KV-cache VRAM: {:.2} GB",
            vram_bytes as f64 / 1e9
        );
        assert!(vram_bytes > 0);
    }
}
```

## 2. Layer Offloading 策略 (n_gpu_layers)

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LayerOffloadConfig {
    pub total_layers: usize,
    pub n_gpu_layers: i32,
    pub available_vram_gb: f32,
    pub layer_size_gb: f32,
}

impl LayerOffloadConfig {
    pub fn optimize_for_vram(
        total_layers: usize,
        available_vram_gb: f32,
        layer_size_gb: f32,
    ) -> Self {
        let kv_cache_overhead = available_vram_gb * 0.15;
        let usable_vram = available_vram_gb - kv_cache_overhead - 1.0;

        let max_gpu_layers = (usable_vram / layer_size_gb) as usize;
        let n_gpu_layers = max_gpu_layers.min(total_layers) as i32;

        Self {
            total_layers,
            n_gpu_layers,
            available_vram_gb,
            layer_size_gb,
        }
    }

    pub fn layer_distribution_report(&self) -> String {
        let gpu_layers = self.n_gpu_layers.max(0) as usize;
        let cpu_layers = self.total_layers.saturating_sub(gpu_layers);

        let gpu_vram = gpu_layers as f32 * self.layer_size_gb;
        let cpu_ram = cpu_layers as f32 * self.layer_size_gb;

        format!(
            "Layer Distribution:\n\
             GPU Layers: {} ({:.1} GB)\n\
             CPU Layers: {} ({:.1} GB)\n\
             Available VRAM: {:.1} GB\n\
             VRAM Utilization: {:.1}%",
            gpu_layers,
            gpu_vram,
            cpu_layers,
            cpu_ram,
            self.available_vram_gb,
            gpu_vram / self.available_vram_gb * 100.0
        )
    }
}
```

## 3. VRAM 监控与 OOM Backpressure

```rust
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::Semaphore;

pub struct VramMonitor {
    total_vram_bytes: AtomicU64,
    used_vram_bytes: AtomicU64,
    peak_vram_bytes: AtomicU64,
    oom_flag: AtomicBool,
    semaphore: Semaphore,
    last_check: parking_lot::Mutex<Instant>,
}

impl VramMonitor {
    pub fn new(total_vram_gb: f32, max_concurrent_requests: usize) -> Self {
        let total_bytes = (total_vram_gb * 1e9) as u64;

        Self {
            total_vram_bytes: AtomicU64::new(total_bytes),
            used_vram_bytes: AtomicU64::new(0),
            peak_vram_bytes: AtomicU64::new(0),
            oom_flag: AtomicBool::new(false),
            semaphore: Semaphore::new(max_concurrent_requests),
            last_check: parking_lot::Mutex::new(Instant::now()),
        }
    }

    pub fn try_reserve(&self, requested_bytes: u64) -> Result<VramGuard<'_>, VramError> {
        let current = self.used_vram_bytes.load(Ordering::Acquire);
        let total = self.total_vram_bytes.load(Ordering::Acquire);

        if current + requested_bytes > total {
            self.oom_flag.store(true, Ordering::Release);
            return Err(VramError::InsufficientMemory {
                requested: requested_bytes,
                available: total.saturating_sub(current),
            });
        }

        self.used_vram_bytes
            .fetch_add(requested_bytes, Ordering::AcqRel);

        let new_used = self.used_vram_bytes.load(Ordering::Acquire);
        let peak = self.peak_vram_bytes.load(Ordering::Acquire);
        if new_used > peak {
            self.peak_vram_bytes.store(new_used, Ordering::Release);
        }

        Ok(VramGuard {
            monitor: self,
            allocated: requested_bytes,
        })
    }

    pub fn usage_percentage(&self) -> f32 {
        let used = self.used_vram_bytes.load(Ordering::Acquire) as f32;
        let total = self.total_vram_bytes.load(Ordering::Acquire) as f32;
        (used / total) * 100.0
    }

    pub fn watermarks(&self) -> VramWatermarks {
        VramWatermarks {
            total_gb: self.total_vram_bytes.load(Ordering::Acquire) as f32 / 1e9,
            used_gb: self.used_vram_bytes.load(Ordering::Acquire) as f32 / 1e9,
            peak_gb: self.peak_vram_bytes.load(Ordering::Acquire) as f32 / 1e9,
            is_oom: self.oom_flag.load(Ordering::Acquire),
        }
    }
}

#[derive(Debug)]
pub struct VramWatermarks {
    pub total_gb: f32,
    pub used_gb: f32,
    pub peak_gb: f32,
    pub is_oom: bool,
}

#[derive(Debug)]
pub enum VramError {
    InsufficientMemory { requested: u64, available: u64 },
}

impl std::fmt::Display for VramError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            VramError::InsufficientMemory { requested, available } => {
                write!(
                    f,
                    "OOM: requested {} bytes, only {} available",
                    requested, available
                )
            }
        }
    }
}

impl std::error::Error for VramError {}

pub struct VramGuard<'a> {
    monitor: &'a VramMonitor,
    allocated: u64,
}

impl Drop for VramGuard<'_> {
    fn drop(&mut self) {
        self.monitor
            .used_vram_bytes
            .fetch_sub(self.allocated, Ordering::AcqRel);
    }
}
```

## 4. Memory Pool 设计

```rust
use std::collections::VecDeque;
use std::sync::Mutex;

pub struct TensorMemoryPool {
    pool: Mutex<VecDeque<Tensor>>,
    max_pooled: usize,
    device: Device,
    dtype: DType,
    shape: Vec<usize>,
}

impl TensorMemoryPool {
    pub fn new(
        prealloc_count: usize,
        max_pooled: usize,
        shape: Vec<usize>,
        dtype: DType,
        device: Device,
    ) -> anyhow::Result<Self> {
        let mut pool = VecDeque::with_capacity(max_pooled);
        for _ in 0..prealloc_count {
            pool.push_back(Tensor::zeros(shape.clone(), dtype, &device)?);
        }

        Ok(Self {
            pool: Mutex::new(pool),
            max_pooled,
            device,
            dtype,
            shape,
        })
    }

    pub fn acquire(&self) -> anyhow::Result<Tensor> {
        let mut pool = self.pool.lock().unwrap();
        if let Some(tensor) = pool.pop_front() {
            Ok(tensor)
        } else {
            Tensor::zeros(self.shape.clone(), self.dtype, &self.device)
        }
    }

    pub fn release(&self, tensor: Tensor) {
        let mut pool = self.pool.lock().unwrap();
        if pool.len() < self.max_pooled {
            pool.push_back(tensor);
        }
    }
}
```

## Red Lines

1. **KV-cache 必须在加载时预分配，不能动态扩容** — 动态扩容触发 GPU 显存碎片和 OOM 风险；必须用 `max_seq_len` 提前确定总容量。
2. **必须实现 OOM backpressure 机制** — 显存使用超过 90% 时必须拒绝新请求（返回 HTTP 503），不能等待 OOM crash。
3. **`n_gpu_layers` 必须显式配置** — 不能依赖库的默认值，必须根据实际可用显存和模型层数计算最佳值。
4. **peak VRAM 必须持续监控** — 预分配 KV-cache + 模型权重后，剩余显存留给激活值和临时张量；peak 超过阈值立即告警。
5. **禁止在 hot loop 中分配 GPU 张量** — 使用内存池 (TensorMemoryPool) 复用频繁分配的张量，避免 CUDA malloc 开销。

## References
- [llama.cpp — GPU offloading layers](https://github.com/ggerganov/llama.cpp)
- [vLLM — PagedAttention memory management](https://arxiv.org/abs/2309.06180)
- [nvml-wrapper — NVIDIA Management Library Rust bindings](https://crates.io/crates/nvml-wrapper)
- [candle — CUDA device management](https://github.com/huggingface/candle)