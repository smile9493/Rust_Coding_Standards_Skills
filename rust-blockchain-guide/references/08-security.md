# 08 — Security & Auditing

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | 所有前序文档, `trident`, `cargo-fuzz` |
| 适用链 | Solana, Substrate |
| 关键工具 | `trident`, `cargo-fuzz`, `solana-verify` |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解所有前序 01-07 文档的核心概念
- 理解 Solana 事务模型和 Anchor 安全约束
- 理解 Rust 安全编程（unsafe, ownership, borrowing）

---

## 1. Checks-Effects-Interactions（反重入）

这是区块链安全最重要的模式。永远在 CPI 之前完成所有状态变更：

```rust
pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
    // 1. CHECKS — 验证所有前提条件
    require!(!ctx.accounts.vault.is_frozen, MyError::VaultFrozen);
    require!(amount > 0, MyError::ZeroAmount);
    require!(
        ctx.accounts.vault.balances.contains_key(&ctx.accounts.user.key()),
        MyError::NoDeposit,
    );

    let user_balance = ctx.accounts.vault.balances[&ctx.accounts.user.key()];
    require!(user_balance >= amount, MyError::InsufficientBalance);

    // 2. EFFECTS — 先更新状态
    ctx.accounts.vault.balances.insert(
        ctx.accounts.user.key(),
        user_balance.checked_sub(amount).ok_or(MyError::Overflow)?,
    );
    ctx.accounts.vault.total_deposited = ctx.accounts.vault.total_deposited
        .checked_sub(amount)
        .ok_or(MyError::Overflow)?;

    // 3. INTERACTIONS — 最后执行 CPI
    let bump = ctx.accounts.vault.bump;
    let vault_key = ctx.accounts.vault.key();
    let seeds: &[&[u8]] = &[
        b"vault",
        &vault_key.to_bytes(),
        &[bump],
    ];
    let signer: &[&[&[u8]]] = &[seeds];

    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.vault_token_account.to_account_info(),
            to: ctx.accounts.user_token_account.to_account_info(),
            authority: ctx.accounts.vault.to_account_info(),
        },
        signer,
    );

    token::transfer(cpi_ctx, amount)?;

    Ok(())
}
```

### 重入攻击的防御模式

```rust
#[account]
pub struct ReentrancyGuard {
    pub locked: bool,
}

impl ReentrancyGuard {
    pub fn lock(&mut self) -> Result<()> {
        require!(!self.locked, MyError::ReentrancyDetected);
        self.locked = true;
        Ok(())
    }

    pub fn unlock(&mut self) {
        self.locked = false;
    }
}

pub fn guarded_operation(ctx: Context<GuardedOperation>) -> Result<()> {
    ctx.accounts.guard.lock()?;

    // ... 执行潜在有风险的 CPI ...

    ctx.accounts.guard.unlock();
    Ok(())
}
```

---

## 2. Checked Math（溢出保护）

```rust
pub fn safe_arithmetic_example(a: u64, b: u64) -> Result<u64> {
    // ✓ 正确 — checked_* 返回 Option
    let sum = a.checked_add(b).ok_or(MyError::Overflow)?;
    let diff = a.checked_sub(b).ok_or(MyError::Underflow)?;
    let prod = a.checked_mul(b).ok_or(MyError::Overflow)?;
    let quot = a.checked_div(b).ok_or(MyError::DivisionByZero)?;

    Ok(sum)
}

pub fn safe_token_math(
    supply: u64,
    mint_amount: u64,
    burn_amount: u64,
) -> Result<(u64, u64)> {
    let new_supply_after_mint = supply
        .checked_add(mint_amount)
        .ok_or(MyError::Overflow)?;

    let new_supply_after_burn = new_supply_after_mint
        .checked_sub(burn_amount)
        .ok_or(MyError::Underflow)?;

    Ok((new_supply_after_mint, new_supply_after_burn))
}

pub fn safe_percentage(amount: u64, bps: u16) -> Result<u64> {
    amount
        .checked_mul(bps as u64)?
        .checked_div(10_000)
        .ok_or(MyError::Overflow.into())
}
```

### 常见溢出场景

```rust
pub fn dangerous_pattern_fixed() -> Result<()> {
    let fee_bps: u16 = 500; // 5%

    // ✗ 危险 — 可能溢出，且先减后除导致精度丢失
    // let fee = amount * fee_bps / 10000;

    // ✓ 安全 — checked 运算 + 正确的先乘后除顺序
    let fee = amount
        .checked_mul(fee_bps as u64)?
        .and_then(|v| v.checked_div(10_000))
        .ok_or(MyError::Overflow)?;

    // ✓ 安全 — 使用 u128 中转防止乘法溢出
    let safe_fee: u64 = {
        let wide = (amount as u128)
            .checked_mul(fee_bps as u128)?
            .checked_div(10_000)?;
        wide.try_into().map_err(|_| MyError::Overflow)?
    };

    Ok(())
}
```

---

## 3. Access Control（访问控制）

```rust
#[derive(Accounts)]
pub struct AdminOnly<'info> {
    #[account(
        seeds = [b"admin-config"],
        bump = config.bump,
        constraint = config.admin == admin.key() @ MyError::Unauthorized,
    )]
    pub config: Account<'info, AdminConfig>,

    pub admin: Signer<'info>,
}

#[derive(Accounts)]
pub struct MultiSigControl<'info> {
    #[account(
        mut,
        seeds = [b"proposal", &proposal.proposal_id.to_le_bytes()],
        bump = proposal.bump,
    )]
    pub proposal: Account<'info, Proposal>,

    #[account(
        constraint = proposal.approvers.contains(&signer.key())
            @ MyError::NotAnApprover,
    )]
    pub signer: Signer<'info>,
}

#[derive(Accounts)]
pub struct RoleBasedAccess<'info> {
    #[account(
        constraint = role_registry.has_role(
            &signer.key(),
            Role::Operator,
        ) @ MyError::NotOperator,
    )]
    pub role_registry: Account<'info, RoleRegistry>,

    pub signer: Signer<'info>,
}
```

### 所有者验证

```rust
pub fn owner_only_action(ctx: Context<OwnerAction>) -> Result<()> {
    // Anchor 在 #[derive(Accounts)] 中验证 has_one
    // 手动验证备用方案: 
    require!(
        ctx.accounts.resource.owner == ctx.accounts.caller.key(),
        MyError::Unauthorized,
    );
    Ok(())
}

pub fn validate_signer_authority(
    signer: &Signer,
    expected_authority: &Pubkey,
) -> Result<()> {
    require!(
        signer.key() == *expected_authority,
        MyError::Unauthorized,
    );
    Ok(())
}
```

---

## 4. Trident Fuzzing

### 安装和初始化

```bash
cargo install trident-cli
trident init
```

### Fuzz 测试定义

```rust
use trident_fuzz::fuzzing::*;

#[derive(Accounts)]
pub struct FuzzWithdraw<'info> {
    #[account(mut)]
    pub vault: Account<'info, Vault>,
    #[account(mut)]
    pub vault_token: Account<'info, TokenAccount>,
    #[account(mut)]
    pub user_token: Account<'info, TokenAccount>,
    pub user: Signer<'info>,
    pub token_program: Program<'info, Token>,
}

#[derive(Arbitrary)]
pub struct WithdrawInput {
    pub amount: u64,
}

fn assert_invariants(
    pre_vault: &Vault,
    post_vault: &Vault,
    pre_balance: u64,
    post_balance: u64,
    amount: u64,
) {
    // 不变量 1: total_deposited 不会无故增加
    assert!(post_vault.total_deposited <= pre_vault.total_deposited);

    // 不变量 2: 余额变化 = amount
    assert_eq!(pre_balance - post_balance, amount);

    // 不变量 3: vault 不会被重入锁定
    assert!(!post_vault.is_frozen);
}
```

### Fuzz 测试执行

```rust
#[trident_test]
async fn test_fuzz_withdraw() {
    let mut runner = runner_builder!()
        .add_fuzz(FuzzWithdraw)
        .build();

    runner.run_fuzz(|ctx| async move {
        let pre_vault = ctx.accounts.vault.clone();
        let pre_balance = ctx.accounts.vault_token.amount;

        let result = program
            .withdraw(ctx, WithdrawInput { amount: /* random */ })
            .await;

        if result.is_ok() {
            let post_vault = ctx.accounts.vault.clone();
            let post_balance = ctx.accounts.vault_token.amount;

            assert_invariants(
                &pre_vault,
                &post_vault,
                pre_balance,
                post_balance,
                ctx.input.amount,
            );
        }
    }).await;
}
```

---

## 5. 安全不变量

```rust
pub fn validate_program_invariants(
    vault: &Vault,
    vault_token: &TokenAccount,
) -> Result<()> {
    // 不变量 1: 账面余额 = 实际余额
    let book_balance = vault.current_book_balance();
    let actual_balance = vault_token.amount;
    require!(
        book_balance == actual_balance,
        MyError::InvariantViolation,
    );

    // 不变量 2: 无负数用户余额
    for (_, balance) in vault.user_balances.iter() {
        require!(
            *balance <= vault.total_deposited,
            MyError::InvariantViolation,
        );
    }

    // 不变量 3: LP token supply = 总质押量
    require!(
        vault.lp_supply == vault.total_staked,
        MyError::InvariantViolation,
    );

    Ok(())
}
```

---

## 6. 审计检查清单

| # | 检查项 | 严重程度 |
|---|--------|----------|
| 1 | 所有写入操作前执行 Checks-Effects-Interactions | CRITICAL |
| 2 | 所有算术使用 checked_* | CRITICAL |
| 3 | 所有 account 验证通过 #[derive(Accounts)] 或手动 owner 检查 | CRITICAL |
| 4 | PDA seeds 包含唯一标识符 | CRITICAL |
| 5 | CPI 调用后验证返回值 | HIGH |
| 6 | 无硬编码地址 | HIGH |
| 7 | 无 `unwrap()` 或 `expect()` | HIGH |
| 8 | `close` 账户不再被访问 | CRITICAL |
| 9 | 无签名者地址可被预测 | MEDIUM |
| 10 | 所有 pub fn 有访问控制 | CRITICAL |

---

## 7. 预先审计脚本

```rust
#[cfg(test)]
mod security_tests {
    use super::*;

    #[test]
    fn test_no_reentrancy() {
        // 验证: CPI 在状态变更之后
    }

    #[test]
    fn test_no_unauthorized_withdraw() {
        // 验证: 非所有者无法提取
    }

    #[test]
    fn test_overflow_protection() {
        // 验证: 大数值操作返回错误而非 panic
    }

    #[test]
    fn test_pda_seed_uniqueness() {
        // 验证: 不同 seeds 产生不同 PDA
    }

    #[test]
    fn test_no_silent_cpi_failure() {
        // 验证: CPI 失败被正确传播
    }

    #[test]
    fn test_close_account_cleanup() {
        // 验证: 关闭的账户不再有 lamports 或数据
    }
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| 必须先状态变更再 CPI | CRITICAL | 违反 → 重入攻击。 |
| 所有算术必须 checked_* | CRITICAL | 溢出 → 资产损失。 |
| 所有账户必须验证 owner | CRITICAL | 不验证 → 伪造账户攻击。 |
| 程序上线前必须三方审计 | CRITICAL | 无论代码量多少。 |
| 禁止在 release 关闭 overflow-checks | CRITICAL | Solana 默认开启，永远不要关闭。 |
| 必须运行 trident fuzzing | HIGH | 覆盖随机输入的边缘情况。 |

---

## References

- [Solana Security Best Practices](https://docs.solana.com/developing/programming-model/security)
- [Trident Framework](https://github.com/Ackee-Blockchain/trident)
- [Neodyme Solana Security Workshop](https://workshop.neodyme.io/)
- [SECBIT Solana Security Audit Checklist](https://github.com/sec-bit/awesome-solana-security)
- [anchor-lang security](https://www.anchor-lang.com/docs/security)