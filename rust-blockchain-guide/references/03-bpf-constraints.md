# 03 — BPF Bytecode Constraints & no_std Hybrid

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | `rust-architecture-guide` no_std, `solana-program` |
| 适用链 | Solana (eBPF/SBF runtime) |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解 no_std 编程（见 rust-architecture-guide/01-no-std-architecture.md）
- 理解 CPU 指令集约束
- 理解 `cargo build-sbf` 工具链

---

## 硬约束摘要

| 约束 | 值 | 影响 |
|------|------|------|
| 最大程序大小 | 200 KB | 必须在编译时优化，控制依赖数量 |
| 每事务 CU 预算 | 200,000 CU | 默认 200K，可设置最高 1.4M |
| 最大 BPF 栈帧 | 4096 bytes | 大型栈分配需使用堆 |
| 最大 CPI 深度 | 4 层 | 影响可组合性 |
| 指令最大账户数 | 64 | 影响复杂操作 |
| 无原生浮点 | ❌ | 需使用 fixed-point u64 |
| 无动态分发 | ❌ | 全静态分发 |
| 无递归 | ❌ (总体) | BPF 不支持无界递归 |

---

## 无浮点：Fixed-Point Arithmetic

BPF 没有 f32/f64 指令。使用 u64 定点数：

```rust
pub const DECIMALS: u8 = 9;
pub const PRECISION: u64 = 1_000_000_000;

pub fn mul_fixed(a: u64, b: u64) -> Option<u64> {
    a.checked_mul(b)?
        .checked_div(PRECISION)
}

pub fn div_fixed(a: u64, b: u64) -> Option<u64> {
    a.checked_mul(PRECISION)?
        .checked_div(b)
}

pub fn add_fixed(a: u64, b: u64) -> Option<u64> {
    a.checked_add(b)
}

pub fn sub_fixed(a: u64, b: u64) -> Option<u64> {
    a.checked_sub(b)
}

pub fn sqrt_fixed(x: u64) -> Option<u64> {
    if x == 0 {
        return Some(0);
    }
    let mut z = x.checked_add(PRECISION)? / 2;
    let mut y = x;
    for _ in 0..64 {
        if z == 0 {
            break;
        }
        y = y.checked_add(z)?;
        z = (y.checked_sub(x.checked_div(y)?))? / 2;
        y = z;
    }
    Some(y)
}

pub fn pow_fixed(base: u64, exp: u32) -> Option<u64> {
    let mut result = PRECISION;
    let mut base_val = base;
    let mut exp_val = exp;

    while exp_val > 0 {
        if exp_val & 1 == 1 {
            result = mul_fixed(result, base_val)?;
        }
        exp_val >>= 1;
        if exp_val > 0 {
            base_val = mul_fixed(base_val, base_val)?;
        }
    }
    Some(result)
}
```

---

## 无动态分发：静态分发

BPF 不支持 `Box<dyn Trait>`, `Rc`, `Arc`, `dyn Trait`：

```rust
use solana_program::pubkey::Pubkey;

enum TransferMethod {
    Direct,
    ViaDelegate(Pubkey),
    MultiSig { signers: Vec<Pubkey>, threshold: u8 },
}

impl TransferMethod {
    fn execute(&self, amount: u64) -> Result<(), ProgramError> {
        match self {
            Self::Direct => {
                // 直接转账逻辑
                Ok(())
            }
            Self::ViaDelegate(delegate) => {
                // 委托转账逻辑
                let _ = delegate;
                Ok(())
            }
            Self::MultiSig { signers, threshold } => {
                require!(
                    signers.len() >= *threshold as usize,
                    ProgramError::NotEnoughSigners
                );
                Ok(())
            }
        }
    }
}

fn process_with_unit_variant(
    variant: u8,
    user: &Pubkey,
    counter: u64,
) -> Result<(), ProgramError> {
    match variant {
        0 => {
            msg!("Creating account for {}", user);
            Ok(())
        }
        1 => {
            msg!("Transferring {} tokens from {}", counter, user);
            Ok(())
        }
        2 => {
            msg!("Deleting account of {}", user);
            Ok(())
        }
        _ => Err(ProgramError::InvalidInstructionData),
    }
}
```

### 避免的写法

```rust
// ✗ BPF 不支持 — Box<dyn Trait>
// fn process(handler: Box<dyn Fn(u64) -> u64>, value: u64) -> u64 { ... }

// ✗ BPF 不支持 — Arc
// use std::sync::Arc;
// let shared = Arc::new(MyData { ... });

// ✗ BPF 不支持 — Rc
// use std::rc::Rc;
// let rc = Rc::new(42);

// ✗ BPF 不支持 — 直接向量动态扩容（应在编译期定界）
// let mut vec = Vec::new();
// for i in 0..unknown_limit { vec.push(i); }  // 动态 push 可能超栈

// ✓ 正确 — 预分配或定界
let mut buf = [0u8; 256];
// 在固定缓冲区中操作
```

---

## no_std + solana-program 混合模式

Solana 程序必须声明 `#![no_std]`，但 `solana-program` 提供了 `std` 替代品：

```rust
#![no_std]

use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};

entrypoint!(process_instruction);

fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    msg!("Program invoked at {}", program_id);

    let accounts_iter = &mut accounts.iter();
    let payer = next_account_info(accounts_iter)?;
    let target = next_account_info(accounts_iter)?;

    if !payer.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    if target.owner != program_id {
        return Err(ProgramError::IllegalOwner);
    }

    let amount = instruction_data
        .get(..8)
        .and_then(|slice| slice.try_into().ok())
        .map(u64::from_le_bytes)
        .ok_or(ProgramError::InvalidInstructionData)?;

    msg!("Processing amount: {}", amount);

    Ok(())
}
```

### solana-program 提供的 std 替代

```rust
use solana_program::msg;
use solana_program::program_error::ProgramError;

fn using_solana_alternatives() {
    msg!("Log message — replaces println!");

    let serialized = {
        let mut buf = [0u8; 128];
        let len = min(128, data_len);
        buf[..len].copy_from_slice(&source[..len]);
        buf
    };

    // solana-program 提供:
    // - msg!()      → println!() 替代
    // - ProgramResult → Result 类型
    // - ProgramError  → 标准错误
    // - entrypoint!() → main() 替代
    // - sol_memcpy/sol_memmove → 低层内存操作
}
```

---

## Compute Budget Profiling

### 设置 CU 预算

```rust
use solana_program::compute_budget::ComputeBudgetInstruction;

fn set_compute_budget_instruction(units: u32) -> Instruction {
    ComputeBudgetInstruction::set_compute_unit_limit(units)
}

fn set_compute_unit_price(micro_lamports: u64) -> Instruction {
    ComputeBudgetInstruction::set_compute_unit_price(micro_lamports)
}

fn build_transaction_with_budget(
    instructions: Vec<Instruction>,
    payer: &Pubkey,
    budget_units: u32,
    priority_fee_lamports: u64,
) -> (Vec<Instruction>, u32) {
    let budget_ix = ComputeBudgetInstruction::set_compute_unit_limit(budget_units);
    let price_ix = ComputeBudgetInstruction::set_compute_unit_price(priority_fee_lamports);

    let mut all_instructions = vec![budget_ix, price_ix];
    all_instructions.extend(instructions);

    (all_instructions, budget_units)
}
```

### CU 预算估算

```rust
fn estimate_cu_for_instruction(
    account_count: usize,
    has_cpi: bool,
    data_size: usize,
) -> u32 {
    let base_cost: u32 = 50_000;
    let per_account_cost: u32 = account_count as u32 * 5_000;
    let data_cost: u32 = data_size as u32 * 10;
    let cpi_cost: u32 = if has_cpi { 50_000 } else { 0 };

    base_cost
        .saturating_add(per_account_cost)
        .saturating_add(data_cost)
        .saturating_add(cpi_cost)
}

#[cfg(test)]
mod cu_tests {
    use super::*;

    #[test]
    fn test_budget_estimation() {
        let budget = estimate_cu_for_instruction(4, true, 200);
        assert!(budget > 0);
        assert!(budget <= 200_000);
    }
}
```

### 使用 solana_compute_budget 进行日志分析

```bash
cargo build-sbf

solana-test-validator \
  --log \
  | grep -E "consumed [0-9]+ of [0-9]+ compute units"
```

---

## 程序大小优化

```rust
// ✓ 使用 strip 和 LTO 优化
// Cargo.toml:
// [profile.release]
// lto = true
// codegen-units = 1
// opt-level = "s"
// strip = true

#[cfg(not(feature = "no-entrypoint"))]
use solana_program::entrypoint;

mod error;
mod instruction;
mod processor;
mod state;

// 避免引入不必要的 crate
// ✗ use serde::{Serialize, Deserialize};
// ✓ use borsh::{BorshSerialize, BorshDeserialize};  // BPF 兼容

fn check_program_size() {
    // 部署前验证:
    // $ ls -la target/deploy/my_program.so
    // 应 < 200 KB (204800 bytes)
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| 禁止使用 f32/f64 | CRITICAL | BPF 无浮点指令，编译失败。使用 fixed-point u64。 |
| 禁止使用 Box<dyn Trait>/Rc/Arc | CRITICAL | BPF 不支持动态分发。使用 enum 静态分发。 |
| 禁止超过 200K CU 预算 | CRITICAL | 默认 200K，exceed 则事务失败。手动设置更高预算。 |
| 禁止递归 | HIGH | BPF 不支持无界递归调用。使用迭代替代。 |
| 禁止使用 std | CRITICAL | 仅使用 solana-program + core + alloc。 |
| 禁止超过 4KB 栈帧 | HIGH | 大数组应分配在堆上或使用零拷贝账户。 |
| 程序大小不超过 200KB | HIGH | 启用 LTO + strip + opt-level="s"。 |

---

## References

- [Solana BPF Constraints](https://docs.solana.com/developing/on-chain-programs/overview#berkeley-packet-filter-bpf)
- [solana-program](https://docs.rs/solana-program/latest/solana_program/)
- [Solana Compute Budget](https://docs.solana.com/developing/programming-model/transactions#compute-budget)
- [Solana no_std entrypoint](https://docs.solana.com/developing/on-chain-programs/developing-rust#project-layout)
- [Fixed-Point Arithmetic](https://en.wikipedia.org/wiki/Fixed-point_arithmetic)