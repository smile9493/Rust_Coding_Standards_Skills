# 07 — Token Economics & SPL Standards

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | [01-account-model.md](01-account-model.md), [02-pda.md](02-pda.md), [04-anchor-framework.md](04-anchor-framework.md) |
| 适用链 | Solana |
| 关键 crate | `spl-token`, `spl-token-2022`, `anchor-spl`, `spl-associated-token-account` |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解 Solana PDA 和账户模型
- 理解 Anchor CPI 机制
- 理解 `checked_add` / `checked_sub` 算术

---

## SPL Token 标准

SPL Token 是 Solana 的同质化代币标准。架构如下：

```
┌──────────┐  1:N   ┌───────────────┐
│  Mint    │───────►│ Token Account │──► owner: Pubkey
│ decimals │        │ amount: u64   │    mint: Pubkey
│ supply   │        │ delegate      │
│ authority│        │ state         │
└──────────┘        └───────────────┘
```

---

## SPL Token 账户结构

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::TokenAccount;

#[derive(Accounts)]
pub struct TokenAccountsCheck<'info> {
    #[account(
        constraint = usdc_account.mint == USDC_MINT @ MyError::InvalidMint,
        constraint = usdc_account.amount >= MIN_BALANCE @ MyError::InsufficientBalance,
    )]
    pub usdc_account: Account<'info, TokenAccount>,

    #[account(
        token::mint = USDC_MINT,
        token::authority = owner,
    )]
    pub ata_usdc: Account<'info, TokenAccount>,

    pub owner: Signer<'info>,
}
```

---

## Associated Token Account (ATA)

ATA 是一个确定性的 PDA——每个 (钱包, Mint) 对应唯一的 Token 账户：

```rust
use anchor_spl::associated_token::AssociatedToken;

#[derive(Accounts)]
pub struct InitializeAta<'info> {
    #[account(
        init,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = payer,
    )]
    pub ata: Account<'info, TokenAccount>,

    #[account(mut)]
    pub payer: Signer<'info>,

    pub mint: Account<'info, Mint>,

    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
}
```

### ATA 的 PDA 推导

```rust
pub fn get_ata_address(wallet: &Pubkey, mint: &Pubkey) -> (Pubkey, u8) {
    Pubkey::find_program_address(
        &[
            wallet.as_ref(),
            &spl_token::id().to_bytes(),
            mint.as_ref(),
        ],
        &spl_associated_token_account::id(),
    )
}
```

---

## Token 转账（含安全检查）

```rust
use anchor_spl::token::{self, Transfer, TokenAccount, Token};

#[derive(Accounts)]
pub struct SafeTransfer<'info> {
    #[account(mut)]
    pub source: Account<'info, TokenAccount>,

    #[account(mut)]
    pub destination: Account<'info, TokenAccount>,

    pub authority: Signer<'info>,

    pub token_program: Program<'info, Token>,
}

pub fn safe_transfer(ctx: Context<SafeTransfer>, amount: u64) -> Result<()> {
    // 1. 验证金额
    require!(amount > 0, MyError::ZeroAmount);

    // 2. 验证来源余额
    require!(
        ctx.accounts.source.amount >= amount,
        MyError::InsufficientBalance,
    );

    // 3. 验证 mint 一致性
    require!(
        ctx.accounts.source.mint == ctx.accounts.destination.mint,
        MyError::MintMismatch,
    );

    // 4. 验证地址不相同
    require!(
        ctx.accounts.source.key() != ctx.accounts.destination.key(),
        MyError::SelfTransfer,
    );

    // 5. 验证目的账户不是冻结的
    require!(
        ctx.accounts.destination.state == spl_token::state::AccountState::Initialized as u8,
        MyError::DestinationFrozen,
    );

    // 6. 验证不会溢出
    let new_dest_balance = ctx.accounts.destination.amount
        .checked_add(amount)
        .ok_or(MyError::Overflow)?;

    // 7. 执行转账
    let cpi_ctx = CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.source.to_account_info(),
            to: ctx.accounts.destination.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    );

    token::transfer(cpi_ctx, amount)?;

    // 8. 验证结果
    ctx.accounts.source.reload()?;
    ctx.accounts.destination.reload()?;
    require!(
        ctx.accounts.source.amount
            == ctx.accounts.source.amount.checked_sub(amount).unwrap_or(0),
        MyError::TransferVerificationFailed,
    );

    Ok(())
}
```

---

## Decimal Scaling（定点小数）

```rust
pub const TOKEN_DECIMALS: u8 = 6;
pub const ONE_TOKEN: u64 = 1_000_000;

pub fn human_to_raw(amount: f64) -> u64 {
    (amount * 10f64.powi(TOKEN_DECIMALS as i32)) as u64
}

pub fn raw_to_human(raw: u64) -> f64 {
    raw as f64 / 10f64.powi(TOKEN_DECIMALS as i32)
}

pub fn calc_fee(amount: u64, fee_bps: u16) -> Option<u64> {
    let fee = amount
        .checked_mul(fee_bps as u64)?
        .checked_div(10_000)?;
    Some(fee)
}

pub fn calc_amount_out(
    reserve_in: u64,
    reserve_out: u64,
    amount_in: u64,
    fee_bps: u16,
) -> Option<u64> {
    let amount_in_with_fee = amount_in
        .checked_mul(10_000u64.checked_sub(fee_bps as u64)?)?;

    let numerator = amount_in_with_fee
        .checked_mul(reserve_out)?;

    let denominator = reserve_in
        .checked_mul(10_000)?
        .checked_add(amount_in_with_fee)?;

    numerator.checked_div(denominator)
}
```

---

## SPL Token-2022 扩展

Token-2022 向后兼容 SPL Token，但新增扩展功能：

```rust
use spl_token_2022::{
    extension::{
        transfer_fee::TransferFeeConfig,
        confidential_transfer::ConfidentialTransferMint,
        metadata_pointer::MetadataPointer,
        interest_bearing_mint::InterestBearingConfig,
    },
    state::Mint,
};

pub fn read_transfer_fee(mint_data: &[u8]) -> Option<(u16, u64)> {
    let mint = Mint::unpack(mint_data).ok()?;
    let transfer_fee = TransferFeeConfig::unpack(mint_data).ok()?;

    let fee_bps: u16 = u16::from(transfer_fee.get_epoch_fee(0).transfer_fee_basis_points);
    let max_fee: u64 = u64::from(transfer_fee.get_epoch_fee(0).maximum_fee);

    Some((fee_bps, max_fee))
}

pub fn is_confidential_mint(mint_data: &[u8]) -> bool {
    ConfidentialTransferMint::unpack(mint_data).is_ok()
}

pub fn read_interest_rate(mint_data: &[u8]) -> Option<i16> {
    let config = InterestBearingConfig::unpack(mint_data).ok()?;
    Some(config.get_initial_rate())
}
```

---

## Token Vault 模式

```rust
#[account]
pub struct TokenVault {
    pub bump: u8,
    pub authority: Pubkey,
    pub total_deposited: u64,
    pub total_withdrawn: u64,
    pub depositor_count: u32,
    pub is_frozen: bool,
}

impl TokenVault {
    pub const LEN: usize = 8 + 1 + 32 + 8 + 8 + 4 + 1;

    pub fn record_deposit(&mut self, amount: u64) -> Result<()> {
        self.total_deposited = self.total_deposited
            .checked_add(amount)
            .ok_or(MyError::Overflow)?;
        self.depositor_count = self.depositor_count
            .checked_add(1)
            .ok_or(MyError::Overflow)?;
        Ok(())
    }

    pub fn record_withdrawal(&mut self, amount: u64) -> Result<()> {
        self.total_withdrawn = self.total_withdrawn
            .checked_add(amount)
            .ok_or(MyError::Overflow)?;
        Ok(())
    }

    pub fn current_balance(&self) -> u64 {
        self.total_deposited
            .checked_sub(self.total_withdrawn)
            .unwrap_or(0)
    }
}

#[derive(Accounts)]
pub struct DepositToVault<'info> {
    #[account(
        mut,
        seeds = [b"token-vault", vault.authority.as_ref()],
        bump = vault.bump,
    )]
    pub vault: Account<'info, TokenVault>,

    #[account(mut)]
    pub user_token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub vault_token_account: Account<'info, TokenAccount>,

    pub user: Signer<'info>,

    pub token_program: Program<'info, Token>,
}

pub fn deposit(ctx: Context<DepositToVault>, amount: u64) -> Result<()> {
    require!(!ctx.accounts.vault.is_frozen, MyError::VaultFrozen);

    let cpi_ctx = CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.user_token_account.to_account_info(),
            to: ctx.accounts.vault_token_account.to_account_info(),
            authority: ctx.accounts.user.to_account_info(),
        },
    );

    token::transfer(cpi_ctx, amount)?;
    ctx.accounts.vault.record_deposit(amount)?;

    emit!(DepositEvent {
        user: ctx.accounts.user.key(),
        amount,
        vault: ctx.accounts.vault.key(),
    });

    Ok(())
}
```

---

## 跨 Token 程序验证

```rust
pub fn verify_token_account(
    account: &AccountInfo,
    expected_owner: &Pubkey,
    expected_mint: &Pubkey,
) -> Result<()> {
    require!(
        account.owner == &spl_token::id(),
        MyError::NotTokenAccount,
    );

    let token_account = spl_token::state::Account::unpack(
        &account.try_borrow_data()?
    )?;

    require!(
        token_account.owner == *expected_owner,
        MyError::WrongTokenOwner,
    );

    require!(
        token_account.mint == *expected_mint,
        MyError::WrongMint,
    );

    Ok(())
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| 所有 Token 运算必须使用 checked_* | CRITICAL | 溢出导致资产损失。 |
| 转账前必须验证 mint 一致 | CRITICAL | 跨 mint 污染。 |
| ATA 地址必须不能硬编码 | HIGH | 不同环境的程序 ID 不同。 |
| Token-2022 转账必须计算 transfer fee | CRITICAL | 忽略手续费 → 金额不匹配。 |
| Vault 模式必须跟踪 deposited/withdrawn | HIGH | 无法对账 → 审计逃逸。 |

---

## References

- [SPL Token Program](https://spl.solana.com/token)
- [SPL Token-2022](https://spl.solana.com/token-2022)
- [anchor-spl token module](https://docs.rs/anchor-spl/latest/anchor_spl/token/index.html)
- [SPL Associated Token Account](https://spl.solana.com/associated-token-account)
- [spl-token crate](https://docs.rs/spl-token/latest/spl_token/)