# 01 — Account Model & State Partitioning

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | `rust-architecture-guide` P0 Safety, `solana-program`, `parity-scale-codec` |
| 适用链 | Solana, Substrate/Polkadot, NEAR |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解 Rust ownership/borrowing 机制
- 理解无 `std` 环境编程（见 `rust-architecture-guide`）
- 理解 Merkle Trie / 状态数据库概念

---

## 核心概念

区块链状态是一个 **账户数据库**（account database），不是传统的关系型数据库。程序本身不持有状态——所有状态都存储在账户中。每个账户是一个带有唯一地址（pubkey）的数据容器。

```
┌──────────────────────────────────────┐
│           Validator State            │
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │Account A│  │Account B│  │Account C│ │
│  │owner: P1│  │owner: P2│  │owner: P3│ │
│  │data: ...│  │data: ...│  │data: ...│ │
│  └────────┘  └────────┘  └────────┘ │
│  Program P1 (stateless, executable)  │
└──────────────────────────────────────┘
```

---

## Solana AccountInfo

Solana 中账户由 `AccountInfo` 表示，核心字段：

```rust
use solana_program::account_info::AccountInfo;
use solana_program::pubkey::Pubkey;

pub struct AccountInfo<'a> {
    pub key: &'a Pubkey,
    pub lamports: Rc<RefCell<&'a mut u64>>,
    pub data: Rc<RefCell<&'a mut [u8]>>,
    pub owner: &'a Pubkey,
    pub rent_epoch: Epoch,
    pub is_signer: bool,
    pub is_writable: bool,
    pub executable: bool,
}
```

### 账户数据反序列化

使用 `borsh` 序列化方案：

```rust
use borsh::{BorshDeserialize, BorshSerialize};

#[derive(BorshSerialize, BorshDeserialize, Debug, Clone, PartialEq)]
pub struct UserProfile {
    pub authority: Pubkey,
    pub reputation_score: u64,
    pub achievements: u32,
    pub is_verified: bool,
}

impl UserProfile {
    pub const LEN: usize = 32 + 8 + 4 + 1;

    pub fn deserialize(data: &mut &[u8]) -> Result<Self, ProgramError> {
        if data.len() < Self::LEN {
            return Err(ProgramError::InvalidAccountData);
        }
        Self::deserialize_reader(&mut data.as_ref())
            .map_err(|_| ProgramError::InvalidAccountData)
    }
}
```

### 读取账户数据的标准模式

```rust
use solana_program::msg;

fn process_read_account(account: &AccountInfo) -> ProgramResult {
    if account.owner != &system_program::id() {
        return Err(ProgramError::IllegalOwner);
    }

    let mut data = account.data.borrow_mut();
    let profile = UserProfile::deserialize(&mut *data)?;

    msg!("Reputation: {}", profile.reputation_score);
    Ok(())
}
```

### 写入账户数据的标准模式

```rust
fn process_update_account(
    account: &AccountInfo,
    new_score: u64,
) -> ProgramResult {
    let mut data = account.data.try_borrow_mut_data()?;

    let mut profile = UserProfile::deserialize(&mut data.as_ref())?;
    profile.reputation_score = new_score;

    profile.serialize(&mut *data)?;
    Ok(())
}
```

---

## Substrate FRAME Storage

Substrate 使用 `StorageMap` / `StorageValue`，底层基于 Merkle-Patricia Trie：

```rust
use frame_support::{pallet_prelude::*, storage::bounded_vec::BoundedVec};

#[frame_support::pallet]
pub mod pallet {
    use super::*;

    #[pallet::storage]
    #[pallet::getter(fn user_profile)]
    pub type UserProfile<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::AccountId,
        UserProfileData<T>,
    >;

    #[pallet::storage]
    #[pallet::getter(fn total_users)]
    pub type TotalUsers<T: Config> = StorageValue<_, u64>;
}

#[derive(Clone, Encode, Decode, PartialEq, RuntimeDebug, TypeInfo, MaxEncodedLen)]
#[scale_info(skip_type_params(T))]
pub struct UserProfileData<T: Config> {
    pub reputation_score: u64,
    pub achievements: u32,
    pub is_verified: bool,
    pub nickname: BoundedVec<u8, T::MaxNicknameLen>,
}
```

### Substrate 读写操作

```rust
impl<T: Config> Pallet<T> {
    pub fn create_profile(
        origin: OriginFor<T>,
        nickname: BoundedVec<u8, T::MaxNicknameLen>,
    ) -> DispatchResult {
        let who = ensure_signed(origin)?;

        ensure!(
            !UserProfile::<T>::contains_key(&who),
            Error::<T>::ProfileAlreadyExists
        );

        let profile = UserProfileData {
            reputation_score: 0,
            achievements: 0,
            is_verified: false,
            nickname,
        };

        UserProfile::<T>::insert(&who, profile);
        TotalUsers::<T>::mutate(|count| *count = count.saturating_add(1));

        Self::deposit_event(Event::ProfileCreated { who });
        Ok(())
    }

    pub fn update_reputation(
        origin: OriginFor<T>,
        target: T::AccountId,
        new_score: u64,
    ) -> DispatchResult {
        let _who = ensure_signed(origin)?;

        UserProfile::<T>::try_mutate(&target, |maybe_profile| -> DispatchResult {
            let profile = maybe_profile.as_mut()
                .ok_or(Error::<T>::ProfileNotFound)?;
            profile.reputation_score = new_score;
            Ok(())
        })?;

        Ok(())
    }
}
```

---

## 序列化方案对比

| 特性 | Borsh (Solana) | Parity SCALE (Substrate) | Bincode (NEAR) |
|------|---------------|--------------------------|----------------|
| Schema | 无 schema | 无 schema | 无 schema |
| 端序 | LE | LE | LE |
| Compact encoding | 不支持 | 支持 (compact int) | 不支持 |
| MaxEncodedLen | 手动计算 | 自动 derive | 手动计算 |
| 变长类型 | 前置长度 | 前置 compact 长度 | 前置长度 |

---

## 跨链账户序列化示例

### Borsh 变体（Solana）

```rust
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub enum GovernanceProposal {
    TransferToken {
        amount: u64,
        recipient: Pubkey,
    },
    UpdateParam {
        param_index: u8,
        new_value: [u8; 32],
    },
    EmergencyPause,
}

// 手动实现 LEN 以通过 BPF 编译器检查
impl GovernanceProposal {
    pub const MAX_LEN: usize = 1 + std::cmp::max(
        8 + 32,    // TransferToken
        1 + 32,    // UpdateParam
    );             // EmergencyPause = 1
}
```

### SCALE 变体（Substrate）

```rust
#[derive(Encode, Decode, Clone, PartialEq, RuntimeDebug, TypeInfo, MaxEncodedLen)]
pub enum GovernanceProposal {
    TransferToken {
        amount: u64,
        recipient: T::AccountId,
    },
    UpdateParam {
        param_index: u8,
        new_value: [u8; 32],
    },
    EmergencyPause,
}

impl GovernanceProposal {
    fn dispatch<T: Config>(self) -> DispatchResult {
        match self {
            Self::TransferToken { amount, recipient } => {
                // transfer logic
                Ok(())
            }
            Self::UpdateParam { param_index, new_value } => {
                // update logic
                Ok(())
            }
            Self::EmergencyPause => {
                // pause logic
                Ok(())
            }
        }
    }
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| 程序不得在内部持有可变状态 | CRITICAL | 所有可变状态必须在账户中。违反则事务不可验证。 |
| 账户数据反序列化前必须校验 owner | CRITICAL | 否则可被任意数据注入攻击。 |
| 账户空间计算必须精确 | HIGH | 空间不足导致写入失败；空间过大浪费 rent。 |
| Substrate StorageMap key 必须抗碰撞 | HIGH | 使用 `Blake2_128Concat` 或 `Twox64Concat`。 |
| 序列化方案全链一致 | CRITICAL | Solana 用 borsh, Substrate 用 SCALE, 不可混用。 |

---

## References

- [solana-program - AccountInfo](https://docs.rs/solana-program/latest/solana_program/account_info/struct.AccountInfo.html)
- [borsh-rs](https://docs.rs/borsh/latest/borsh/)
- [parity-scale-codec](https://docs.rs/parity-scale-codec/latest/parity_scale_codec/)
- [Substrate FRAME Storage](https://docs.substrate.io/build/runtime-storage/)
- [rust-architecture-guide - no_std Architecture](../rust-architecture-guide/references/01-no-std-architecture.md)