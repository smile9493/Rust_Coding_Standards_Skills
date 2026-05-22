# 02 — Program Derived Addresses (PDA)

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | [01-account-model.md](01-account-model.md), `solana-program` |
| 适用链 | Solana |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解 Solana 账户模型（见 01-account-model.md）
- 理解 ed25519 密钥对和签名机制
- 理解 `Pubkey` 和 `system_program`

---

## 核心概念

PDA（Program Derived Address）是一个从 seeds + program_id 确定性推导出的地址，**没有对应的私钥**。PDA 位于 ed25519 曲线之外，因此任何人都无法为其生成有效签名。

```
seeds = [b"escrow", user_pubkey, escrow_id]
         │
         ▼
find_program_address(seeds, program_id)
         │
         ▼
(pda, bump)  ← bump 确保地址在曲线外
```

---

## 地址推导

### find_program_address

```rust
use solana_program::pubkey::Pubkey;

fn derive_pda(program_id: &Pubkey) -> (Pubkey, u8) {
    let seeds: &[&[u8]] = &[
        b"escrow",
        b"v1",
    ];

    let (pda, bump) = Pubkey::find_program_address(seeds, program_id);
    (pda, bump)
}

fn derive_pda_with_user(user: &Pubkey, program_id: &Pubkey) -> (Pubkey, u8) {
    let seeds: &[&[u8]] = &[
        b"vault",
        user.as_ref(),
    ];

    Pubkey::find_program_address(seeds, program_id)
}
```

### 内部机制

`find_program_address` 的核心逻辑：

```rust
pub fn find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> (Pubkey, u8) {
    for bump in (0..=255).rev() {
        let mut hasher = Sha256::default();
        for seed in seeds {
            hasher.update(seed);
        }
        hasher.update(&[bump]);
        hasher.update(program_id.as_ref());

        let hash = Pubkey::new_from_array(hasher.finalize().into());
        if !hash.is_on_curve() {
            return (hash, bump);
        }
    }
    panic!("Unable to find a valid PDA");
}
```

---

## Anchor PDA 声明

使用 Anchor 框架时的 PDA 声明方式：

```rust
use anchor_lang::prelude::*;

#[derive(Accounts)]
#[instruction(escrow_id: u64)]
pub struct CreateEscrow<'info> {
    #[account(
        init,
        payer = maker,
        space = Escrow::LEN,
        seeds = [
            b"escrow",
            maker.key().as_ref(),
            &escrow_id.to_le_bytes(),
        ],
        bump
    )]
    pub escrow: Account<'info, Escrow>,

    #[account(mut)]
    pub maker: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[account]
pub struct Escrow {
    pub maker: Pubkey,
    pub escrow_id: u64,
    pub amount: u64,
    pub taker: Option<Pubkey>,
    pub bump: u8,
}

impl Escrow {
    pub const LEN: usize = 8 + 32 + 8 + 8 + 33 + 1;
}
```

---

## 核心模式

### 模式 1: Escrow（托管账户）

```rust
#[derive(Accounts)]
#[instruction(escrow_id: u64)]
pub struct EscrowContext<'info> {
    #[account(
        init,
        payer = maker,
        space = EscrowVault::LEN,
        seeds = [
            b"escrow",
            maker.key().as_ref(),
            &escrow_id.to_le_bytes(),
        ],
        bump
    )]
    pub vault: Account<'info, EscrowVault>,

    #[account(mut)]
    pub maker: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[account]
pub struct EscrowVault {
    pub maker: Pubkey,
    pub taker: Pubkey,
    pub amount: u64,
    pub bump: u8,
    pub state: EscrowState,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq, Eq)]
pub enum EscrowState {
    Pending,
    Funded,
    Released,
    Refunded,
}
```

### 模式 2: Vault（保险库）

```rust
#[derive(Accounts)]
pub struct VaultContext<'info> {
    #[account(
        seeds = [b"vault", authority.key().as_ref()],
        bump = vault.bump,
    )]
    pub vault: Account<'info, Vault>,

    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    pub authority: Pubkey,
    pub bump: u8,
    pub total_deposits: u64,
    pub user_count: u32,
}

impl Vault {
    pub const LEN: usize = 8 + 32 + 1 + 8 + 4;

    pub fn deposit(&mut self, amount: u64) {
        self.total_deposits = self.total_deposits
            .checked_add(amount)
            .expect("overflow");
        self.user_count = self.user_count.saturating_add(1);
    }
}
```

### 模式 3: Authority Delegation（权限委托）

通过 PDA 签名来授权 CPI 调用。PDA 可以"签名"——只需程序使用 `invoke_signed` 并提供正确的 seeds：

```rust
pub fn delegate_token_transfer(ctx: Context<DelegateTransfer>, amount: u64) -> Result<()> {
    let vault_bump = ctx.accounts.vault.bump;
    let vault_key = ctx.accounts.vault.key();
    let seeds: &[&[u8]] = &[
        b"vault",
        ctx.accounts.authority.key().as_ref(),
        &[vault_bump],
    ];
    let signer_seeds: &[&[&[u8]]] = &[seeds];

    let transfer_ix = spl_token::instruction::transfer(
        ctx.accounts.token_program.key,
        &ctx.accounts.vault_token_account.key(),
        &ctx.accounts.destination.key(),
        &vault_key,
        &[],
        amount,
    )?;

    invoke_signed(
        &transfer_ix,
        &[
            ctx.accounts.vault_token_account.to_account_info(),
            ctx.accounts.destination.to_account_info(),
            ctx.accounts.vault.to_account_info(),
            ctx.accounts.token_program.to_account_info(),
        ],
        signer_seeds,
    )?;

    Ok(())
}

#[derive(Accounts)]
pub struct DelegateTransfer<'info> {
    #[account(
        seeds = [b"vault", authority.key().as_ref()],
        bump,
    )]
    pub vault: Account<'info, Vault>,

    #[account(mut)]
    pub vault_token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub destination: Account<'info, TokenAccount>,

    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
}
```

### 模式 4: Unique Seed 防碰撞

```rust
#[derive(Accounts)]
#[instruction(challenge_id: [u8; 32])]
pub struct CreateChallenge<'info> {
    #[account(
        init,
        payer = creator,
        space = Challenge::LEN,
        seeds = [
            b"challenge",
            creator.key().as_ref(),
            &challenge_id,
        ],
        bump
    )]
    pub challenge: Account<'info, Challenge>,

    #[account(mut)]
    pub creator: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct ClaimReward<'info> {
    #[account(
        seeds = [
            b"reward",
            player.key().as_ref(),
            &challenge.pool_id.to_le_bytes(),
        ],
        bump,
    )]
    pub reward_vault: Account<'info, RewardVault>,

    pub player: Signer<'info>,
    pub challenge: Account<'info, Challenge>,
}
```

---

## PDA 签名内部机制

`invoke_signed` 在 runtime 中验证 PDA 签名。程序提供 seeds，runtime 重新计算 PDA 地址并验证其与调用者匹配：

```rust
pub fn invoke_signed(
    instruction: &Instruction,
    account_infos: &[AccountInfo],
    signers_seeds: &[&[&[u8]]],
) -> ProgramResult {
    for signer_seeds in signers_seeds {
        let (derived_key, _bump) =
            Pubkey::find_program_address(signer_seeds, &account_infos[0].owner);

        // Runtime 验证 derived_key == account.key
        // 如果匹配，视为有效签名
    }

    solana_program::program::invoke(instruction, account_infos)
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| Seeds 必须包含唯一标识符 | CRITICAL | 缺少唯一性 → PDA 碰撞 → 资产被盗。 |
| 必须存储并验证 bump | HIGH | Anchor 自动处理；原生 Rust 必须手动存储。 |
| PDA 不得暴露给用户直接签名 | CRITICAL | PDA 无对应私钥，企图签名将 panic。 |
| Seeds 不得包含用户可控的可变数据 | CRITICAL | 攻击者可操纵 seeds 改变 PDA 推导结果。 |
| invoke_signed 的 seeds 必须严格匹配创建时的 seeds | CRITICAL | 不匹配导致 CPI 签名失败。 |

---

## References

- [Solana PDA Docs](https://docs.solana.com/developing/programming-model/calling-between-programs#program-derived-addresses)
- [anchor-lang - PDA](https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html)
- [solana-program - find_program_address](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.find_program_address)