# 05 — Cross-Program Invocation (CPI)

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | [04-anchor-framework.md](04-anchor-framework.md), `anchor-spl`, `solana-program` |
| 适用链 | Solana |
| 关键 API | `invoke()`, `invoke_signed()`, `anchor_spl` |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解 Anchor #[derive(Accounts)], PDA（见 04-anchor-framework.md, 02-pda.md）
- 理解 SPL Token 标准（见 07-token-economics.md）
- 理解 Solana 事务模型

---

## CPI 的两种模式

```
┌─────────────┐    invoke()      ┌─────────────┐
│ Program A   │ ───────────────► │ Program B   │
│ (caller)    │                  │ (callee)    │
│             │ ◄─────────────── │             │
│             │    result        │             │
└─────────────┘                  └─────────────┘

┌─────────────┐  invoke_signed() ┌─────────────┐
│ Program A   │ ───────────────► │ SPL Token   │
│ (PDA signer)│   seeds=[...]    │ (transfer)  │
└─────────────┘                  └─────────────┘
```

---

## 原生 Solana CPI: invoke() 和 invoke_signed()

### invoke() — 由用户签名的 CPI

```rust
use solana_program::{
    instruction::{AccountMeta, Instruction},
    program::invoke,
    pubkey::Pubkey,
};

pub fn perform_cpi_transfer(
    program_id: &Pubkey,
    source: &AccountInfo,
    destination: &AccountInfo,
    authority: &AccountInfo,
    amount: u64,
) -> ProgramResult {
    let ix = spl_token::instruction::transfer(
        &spl_token::id(),
        source.key,
        destination.key,
        authority.key,
        &[],
        amount,
    )?;

    invoke(
        &ix,
        &[
            source.clone(),
            destination.clone(),
            authority.clone(),
        ],
    )
}
```

### invoke_signed() — PDA 作为签名者的 CPI

```rust
use solana_program::program::invoke_signed;

pub fn vault_token_transfer(
    vault: &AccountInfo,
    vault_token_account: &AccountInfo,
    destination: &AccountInfo,
    token_program: &AccountInfo,
    amount: u64,
    vault_authority_bump: u8,
    vault_authority_seed: &Pubkey,
) -> ProgramResult {
    let seeds: &[&[u8]] = &[
        b"vault-authority",
        vault_authority_seed.as_ref(),
        &[vault_authority_bump],
    ];
    let signer_seeds: &[&[&[u8]]] = &[seeds];

    let ix = spl_token::instruction::transfer(
        &spl_token::id(),
        vault_token_account.key,
        destination.key,
        vault.key,
        &[],
        amount,
    )?;

    invoke_signed(
        &ix,
        &[
            vault_token_account.clone(),
            destination.clone(),
            vault.clone(),
            token_program.clone(),
        ],
        signer_seeds,
    )
}
```

---

## Anchor CPI with anchor_spl

### SPL Token Transfer

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

#[derive(Accounts)]
pub struct VaultWithdraw<'info> {
    #[account(
        mut,
        seeds = [b"vault", vault.owner.as_ref()],
        bump = vault.bump,
    )]
    pub vault: Account<'info, Vault>,

    #[account(mut)]
    pub vault_token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub destination_token_account: Account<'info, TokenAccount>,

    pub owner: Signer<'info>,

    pub token_program: Program<'info, Token>,
}

pub fn vault_withdraw(ctx: Context<VaultWithdraw>, amount: u64) -> Result<()> {
    require!(
        ctx.accounts.owner.key() == ctx.accounts.vault.owner,
        MyError::Unauthorized,
    );

    let bump = ctx.accounts.vault.bump;
    let owner_key = ctx.accounts.vault.owner;
    let seeds: &[&[u8]] = &[
        b"vault",
        owner_key.as_ref(),
        &[bump],
    ];
    let signer: &[&[&[u8]]] = &[seeds];

    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.vault_token_account.to_account_info(),
            to: ctx.accounts.destination_token_account.to_account_info(),
            authority: ctx.accounts.vault.to_account_info(),
        },
        signer,
    );

    token::transfer(cpi_ctx, amount)?;
    Ok(())
}
```

### SPL Token Mint

```rust
use anchor_spl::token::{self, Mint, MintTo, TokenAccount};

#[derive(Accounts)]
pub struct MintTokens<'info> {
    #[account(
        mut,
        seeds = [b"mint-authority"],
        bump,
    )]
    pub mint_authority: AccountInfo<'info>,

    #[account(mut)]
    pub mint: Account<'info, Mint>,

    #[account(mut)]
    pub destination: Account<'info, TokenAccount>,

    pub authority: Signer<'info>,

    pub token_program: Program<'info, Token>,
}

pub fn mint_tokens(ctx: Context<MintTokens>, amount: u64) -> Result<()> {
    let bump = ctx.bumps.mint_authority;
    let seeds: &[&[u8]] = &[b"mint-authority", &[bump]];
    let signer: &[&[&[u8]]] = &[seeds];

    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        MintTo {
            mint: ctx.accounts.mint.to_account_info(),
            to: ctx.accounts.destination.to_account_info(),
            authority: ctx.accounts.mint_authority.to_account_info(),
        },
        signer,
    );

    token::mint_to(cpi_ctx, amount)?;
    Ok(())
}
```

### SPL Token Burn

```rust
use anchor_spl::token::{self, Burn};

#[derive(Accounts)]
pub struct BurnTokens<'info> {
    #[account(mut)]
    pub token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub mint: Account<'info, Mint>,

    pub authority: Signer<'info>,

    pub token_program: Program<'info, Token>,
}

pub fn burn_tokens(ctx: Context<BurnTokens>, amount: u64) -> Result<()> {
    require!(
        ctx.accounts.token_account.amount >= amount,
        MyError::InsufficientFunds,
    );

    let cpi_ctx = CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        Burn {
            mint: ctx.accounts.mint.to_account_info(),
            from: ctx.accounts.token_account.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    );

    token::burn(cpi_ctx, amount)?;
    Ok(())
}
```

---

## 多步 CPI 编排

```rust
pub fn swap_and_stake(ctx: Context<SwapAndStake>, amount_in: u64) -> Result<()> {
    // Step 1: CPI to AMM to swap token A → token B
    let swap_ix = amm_cpi::swap(
        &ctx.accounts.amm_program,
        &ctx.accounts.user_token_a,
        &ctx.accounts.pool_token_a,
        &ctx.accounts.pool_token_b,
        &ctx.accounts.user_token_b,
        amount_in,
    );

    invoke(
        &swap_ix,
        &[
            ctx.accounts.user_token_a.to_account_info(),
            ctx.accounts.pool_token_a.to_account_info(),
            ctx.accounts.pool_token_b.to_account_info(),
            ctx.accounts.user_token_b.to_account_info(),
            ctx.accounts.amm_program.to_account_info(),
        ],
    )?;

    // Step 2: 读取 CPI 返回的金额（通过账户余额变化推断）
    ctx.accounts.user_token_b.reload()?;
    let received_amount = ctx.accounts.user_token_b.amount;

    // Step 3: CPI to staking program
    let stake_ix = stake_cpi::stake(
        &ctx.accounts.stake_program.key(),
        &ctx.accounts.user_token_b.key(),
        &ctx.accounts.stake_vault.key(),
        &ctx.accounts.user.key(),
        received_amount,
    );

    invoke(
        &stake_ix,
        &[
            ctx.accounts.user_token_b.to_account_info(),
            ctx.accounts.stake_vault.to_account_info(),
            ctx.accounts.user.to_account_info(),
            ctx.accounts.stake_program.to_account_info(),
        ],
    )?;

    Ok(())
}
```

---

## 程序所有权检查

在进行 CPI 之前，必须验证目标账户的所有权：

```rust
pub fn safe_cpi(ctx: Context<SafeCpi>, amount: u64) -> Result<()> {
    msg!("CPI safety checks starting");

    // 1. 验证目标 token 账户所有权
    require!(
        ctx.accounts.destination.owner == spl_token::id(),
        MyError::InvalidTokenAccount,
    );

    // 2. 验证 token 程序 ID
    require!(
        ctx.accounts.token_program.key() == spl_token::id(),
        MyError::InvalidTokenProgram,
    );

    // 3. 执行 CPI
    let cpi_ctx = CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.source.to_account_info(),
            to: ctx.accounts.destination.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    );

    token::transfer(cpi_ctx, amount)?;

    // 4. 验证结果
    ctx.accounts.destination.reload()?;
    msg!("CPI completed. Destination balance updated.");

    Ok(())
}
```

---

## 返回值验证模式

```rust
pub fn verify_cpi_result(
    source_balance_before: u64,
    dest_balance_before: u64,
    source: &AccountInfo,
    destination: &AccountInfo,
    expected_amount: u64,
) -> Result<()> {
    let source_after: u64 = {
        let data = source.try_borrow_data()?;
        let mut slice = &data[..];
        spl_token::state::Account::unpack(&mut slice)?.amount
    };

    let dest_after: u64 = {
        let data = destination.try_borrow_data()?;
        let mut slice = &data[..];
        spl_token::state::Account::unpack(&mut slice)?.amount
    };

    require!(
        source_balance_before.checked_sub(source_after) == Some(expected_amount),
        MyError::CpiSourceBalanceMismatch,
    );

    require!(
        dest_after.checked_sub(dest_balance_before) == Some(expected_amount),
        MyError::CpiDestBalanceMismatch,
    );

    Ok(())
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| 必须先验证所有权再 CPI | CRITICAL | 防止注入伪造程序账户。 |
| invoke_signed seeds 必须与创建 PDA 时一致 | CRITICAL | 不匹配则 CPI 签名失败。 |
| 对不可信程序 CPI 放在状态修改之前 | CRITICAL | 防止重入攻击（Checks-Effects-Interactions）。 |
| CPI 返回值必须验证 | HIGH | 静默失败 = 资产不一致。 |
| CPI 深度不超过 4 层 | MEDIUM | 超过则事务失败。 |
| 禁止在回调/嵌套 CPI 中修改调用方状态 | CRITICAL | 重入向量。 |

---

## References

- [solana-program - invoke](https://docs.rs/solana-program/latest/solana_program/program/fn.invoke.html)
- [solana-program - invoke_signed](https://docs.rs/solana-program/latest/solana_program/program/fn.invoke_signed.html)
- [anchor-spl](https://docs.rs/anchor-spl/latest/anchor_spl/)
- [anchor-lang - CpiContext](https://docs.rs/anchor-lang/latest/anchor_lang/context/struct.CpiContext.html)