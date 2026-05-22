# 04 — Anchor Framework & Program Structure

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | [01-account-model.md](01-account-model.md), [03-bpf-constraints.md](03-bpf-constraints.md) |
| 适用链 | Solana |
| 关键 crate | `anchor-lang`, `anchor-spl` |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解 Solana 账户模型和 PDA
- 理解 BPF 约束
- 完成 Anchor [官方教程](https://www.anchor-lang.com/docs)

---

## 程序骨架

Anchor 程序的标准结构：

```
my_program/
├── programs/
│   └── my_program/
│       ├── src/
│       │   ├── lib.rs          # entry point
│       │   ├── instructions/   # 指令模块
│       │   │   ├── mod.rs
│       │   │   ├── create.rs
│       │   │   └── update.rs
│       │   ├── state/          # 账户数据结构
│       │   │   ├── mod.rs
│       │   │   └── config.rs
│       │   └── errors.rs       # 自定义错误
│       └── Cargo.toml
└── Cargo.toml
```

---

## #[program] 入口模块

```rust
use anchor_lang::prelude::*;

declare_id!("MyProgram1111111111111111111111111111111111");

#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, bump: u8) -> Result<()> {
        ctx.accounts.config.authority = ctx.accounts.authority.key();
        ctx.accounts.config.bump = bump;
        ctx.accounts.config.is_paused = false;
        Ok(())
    }

    pub fn create_profile(ctx: Context<CreateProfile>, name: String, bio: String) -> Result<()> {
        require!(name.len() <= 50, MyError::NameTooLong);
        require!(bio.len() <= 200, MyError::BioTooLong);

        ctx.accounts.profile.authority = ctx.accounts.authority.key();
        ctx.accounts.profile.name = name;
        ctx.accounts.profile.bio = bio;
        ctx.accounts.profile.reputation = 0;
        ctx.accounts.profile.bump = ctx.bumps.profile;

        Ok(())
    }

    pub fn update_reputation(ctx: Context<UpdateReputation>, new_score: u64) -> Result<()> {
        ctx.accounts.profile.reputation = new_score;
        Ok(())
    }

    pub fn pause_protocol(ctx: Context<AdminControl>) -> Result<()> {
        ctx.accounts.config.is_paused = true;
        Ok(())
    }
}
```

---

## #[derive(Accounts)] 账户验证

这是 Anchor 的核心——一个指令的所有账户验证都集中在一个 struct 中：

```rust
#[derive(Accounts)]
#[instruction(name: String)]
pub struct CreateProfile<'info> {
    #[account(
        init,
        payer = authority,
        space = Profile::LEN,
        seeds = [
            b"profile",
            authority.key().as_ref(),
            name.as_bytes(),
        ],
        bump,
    )]
    pub profile: Account<'info, Profile>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct UpdateReputation<'info> {
    #[account(
        mut,
        seeds = [
            b"profile",
            authority.key().as_ref(),
            profile.name.as_bytes(),
        ],
        bump = profile.bump,
        has_one = authority @ MyError::Unauthorized,
    )]
    pub profile: Account<'info, Profile>,

    pub authority: Signer<'info>,
}

#[derive(Accounts)]
pub struct AdminControl<'info> {
    #[account(
        mut,
        seeds = [b"config"],
        bump = config.bump,
        constraint = config.authority == authority.key() @ MyError::Unauthorized,
    )]
    pub config: Account<'info, GlobalConfig>,

    pub authority: Signer<'info>,
}
```

### Anchor 约束速查

| 约束 | 语义 | 示例 |
|------|------|------|
| `#[account(mut)]` | 账户可写 | `pub token: Account<'info, TokenAccount>` |
| `#[account(signer)]` | 账户必须签名 | `pub payer: Signer<'info>` |
| `#[account(init)]` | 创建新账户 | `payer`, `space`, `seeds`, `bump` 必填 |
| `#[account(close = target)]` | 关闭账户，lamports 退还到 target | `close = authority` |
| `has_one = field` | 验证账户某字段 == 另一个账户的 key | `has_one = authority` |
| `constraint = expr` | 自定义布尔约束 | `constraint = config.authority == authority.key()` |
| `seeds = [...]` | PDA 验证 | 配合 `bump` |
| `token::mint = ...` | SPL Token mint 验证 | `token::mint = config.usdc_mint` |
| `token::authority = ...` | SPL Token authority 验证 | `token::authority = vault` |

---

## #[account] 数据结构

```rust
use anchor_lang::prelude::*;

#[account]
#[derive(InitSpace)]
pub struct GlobalConfig {
    pub authority: Pubkey,
    pub bump: u8,
    pub is_paused: bool,
    pub fee_basis_points: u16,
}

#[account]
#[derive(InitSpace)]
pub struct Profile {
    pub authority: Pubkey,
    pub bump: u8,
    pub reputation: u64,
    #[max_len(50)]
    pub name: String,
    #[max_len(200)]
    pub bio: String,
}

#[account]
pub struct Leaderboard {
    pub entries: Vec<LeaderboardEntry>,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, InitSpace)]
pub struct LeaderboardEntry {
    pub player: Pubkey,
    pub score: u64,
}

impl Leaderboard {
    pub const MAX_ENTRIES: usize = 100;

    pub fn add_entry(&mut self, entry: LeaderboardEntry) -> Result<()> {
        require!(
            self.entries.len() < Self::MAX_ENTRIES,
            MyError::LeaderboardFull
        );
        self.entries.push(entry);
        self.entries.sort_by(|a, b| b.score.cmp(&a.score));
        self.entries.truncate(Self::MAX_ENTRIES);
        Ok(())
    }
}
```

### 账户鉴别器（Discriminator）

Anchor 自动为每个 `#[account]` struct 生成 8 字节 SHA256 鉴别器，防止账户类型混淆：

```rust
fn discriminator() -> [u8; 8] {
    let mut hasher = Sha256::default();
    hasher.update(b"account:GlobalConfig");
    let hash = hasher.finalize();
    let mut disc = [0u8; 8];
    disc.copy_from_slice(&hash[..8]);
    disc
}
```

---

## 自定义错误

```rust
use anchor_lang::prelude::*;

#[error_code]
pub enum MyError {
    #[msg("Name must not exceed 50 characters")]
    NameTooLong,

    #[msg("Bio must not exceed 200 characters")]
    BioTooLong,

    #[msg("Caller is not authorized")]
    Unauthorized,

    #[msg("Protocol is currently paused")]
    ProtocolPaused,

    #[msg("Leaderboard is full")]
    LeaderboardFull,

    #[msg("Insufficient funds")]
    InsufficientFunds,

    #[msg("Arithmetic overflow detected")]
    Overflow,
}
```

---

## 在指令中使用 require!

```rust
pub fn transfer_conditional(ctx: Context<ConditionalTransfer>, amount: u64) -> Result<()> {
    require!(!ctx.accounts.config.is_paused, MyError::ProtocolPaused);
    require!(amount > 0, MyError::InsufficientFunds);

    let source_balance = ctx.accounts.source.amount;
    require!(source_balance >= amount, MyError::InsufficientFunds);

    let new_source_balance = source_balance
        .checked_sub(amount)
        .ok_or(MyError::Overflow)?;

    let new_dest_balance = ctx.accounts.destination.amount
        .checked_add(amount)
        .ok_or(MyError::Overflow)?;

    ctx.accounts.source.amount = new_source_balance;
    ctx.accounts.destination.amount = new_dest_balance;

    Ok(())
}
```

---

## 零拷贝账户（Zero-Copy）

对于大型账户（>10KB），使用零拷贝避免栈复制：

```rust
use anchor_lang::prelude::*;

#[account(zero_copy)]
#[derive(InitSpace)]
pub struct LargeOrderBook {
    pub head: u64,
    pub bids: [Order; 1024],
    pub asks: [Order; 1024],
}

#[zero_copy]
#[derive(AnchorSerialize, AnchorDeserialize, PartialEq, Eq)]
pub struct Order {
    pub price: u64,
    pub quantity: u64,
    pub owner: Pubkey,
    pub order_id: u64,
    pub timestamp: i64,
}

#[derive(Accounts)]
pub struct InitializeOrderBook<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + LargeOrderBook::INIT_SPACE,
        seeds = [b"orderbook"],
        bump
    )]
    pub order_book: AccountLoader<'info, LargeOrderBook>,

    #[account(mut)]
    pub payer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

pub fn place_bid(ctx: Context<PlaceBid>, price: u64, quantity: u64) -> Result<()> {
    let mut order_book = ctx.accounts.order_book.load_mut()?;
    // 直接对 mmap 数据操作，无栈复制
    let idx = (order_book.head % 1024) as usize;
    order_book.bids[idx] = Order {
        price,
        quantity,
        owner: ctx.accounts.user.key(),
        order_id: order_book.head,
        timestamp: Clock::get()?.unix_timestamp,
    };
    order_book.head = order_book.head.wrapping_add(1);
    Ok(())
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| 所有指令必须使用 #[derive(Accounts)] | CRITICAL | 禁止手动验证账户，极易遗漏安全检查。 |
| 不要跳过 `bump` 种子验证 | CRITICAL | PDA 创建不安全。Anchor 的 `ctx.bumps` 必须使用。 |
| `has_one` + `constraint` 覆盖所有授权检查 | CRITICAL | 单一缺失 = 授权绕过。 |
| Anchor 版本锁定 | HIGH | 升级 anchor-lang 需要重新部署程序 ID。 |
| `#[account(close)]` 后不要再写该账户 | CRITICAL | 数据已释放，二次写入 = UB。 |

---

## References

- [Anchor Book](https://book.anchor-lang.com/)
- [anchor-lang crate](https://docs.rs/anchor-lang/latest/anchor_lang/)
- [Anchor Constraints](https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html#constraints)
- [anchor-spl crate](https://docs.rs/anchor-spl/latest/anchor_spl/)