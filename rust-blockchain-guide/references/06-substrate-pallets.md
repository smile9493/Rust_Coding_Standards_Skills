# 06 — Substrate FRAME Pallets

## Status

| 属性 | 值 |
|------|-----|
| 依赖 | [01-account-model.md](01-account-model.md), `frame-support`, `frame-system` |
| 适用链 | Polkadot, Kusama, Substrate 独立链 |
| 关键 crate | `frame-support`, `frame-system`, `pallet-*` |
| Rust Edition | 2024 |

---

## Prerequisites

- 理解 Rust trait 和泛型约束
- 理解 Substrate 运行时和账户模型
- 理解 WASM 编译目标（no_std）

---

## FRAME Pallet 骨架

FRAME（Framework for Runtime Aggregation of Modularized Entities）是 Substrate 的标准运行时框架：

```
┌─────────────────────────────────────────────────────┐
│                    Runtime                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │ Balances │  │  Staking │  │  Custom Pallet   │ │
│  │  Pallet  │  │  Pallet  │  │  (your code)     │ │
│  └──────────┘  └──────────┘  └──────────────────┘ │
│         construct_runtime! macro                    │
└─────────────────────────────────────────────────────┘
```

---

## #[frame::pallet] 基本结构

```rust
#![cfg_attr(not(feature = "std"), no_std)]

pub use pallet::*;

#[frame_support::pallet]
pub mod pallet {
    use frame_support::pallet_prelude::*;
    use frame_system::pallet_prelude::*;
    use sp_runtime::traits::{CheckedAdd, CheckedSub};

    #[pallet::pallet]
    pub struct Pallet<T>(_);

    #[pallet::config]
    pub trait Config: frame_system::Config {
        type RuntimeEvent: From<Event<Self>>
            + IsType<<Self as frame_system::Config>::RuntimeEvent>;

        type MaxNameLength: Get<u32>;

        type WeightInfo: WeightInfo;
    }
}
```

---

## #[pallet::storage] 存储定义

```rust
use frame_support::{
    pallet_prelude::*,
    storage::bounded_vec::BoundedVec,
    Blake2_128Concat,
};

#[pallet::storage]
#[pallet::getter(fn profiles)]
pub type Profiles<T: Config> = StorageMap<
    _,
    Blake2_128Concat,    // key hasher
    T::AccountId,        // key type
    Profile<T>,          // value type
>;

#[pallet::storage]
#[pallet::getter(fn counter)]
pub type Counter<T: Config> = StorageValue<_, u64, ValueQuery>;

#[pallet::storage]
#[pallet::getter(fn registered_accounts)]
pub type RegisteredAccounts<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::AccountId,
    RegistrationInfo,
>;

#[pallet::storage]
#[pallet::getter(fn account_count)]
pub type AccountCount<T: Config> =
    CountedStorageMap<_, Blake2_128Concat, T::AccountId, ()>;
```

### 存储数据结构

```rust
#[derive(Clone, Encode, Decode, PartialEq, RuntimeDebug, TypeInfo, MaxEncodedLen)]
#[scale_info(skip_type_params(T))]
pub struct Profile<T: Config> {
    pub nickname: BoundedVec<u8, T::MaxNameLength>,
    pub reputation: u64,
    pub level: u32,
    pub registered_at: BlockNumberFor<T>,
}

#[derive(Clone, Encode, Decode, PartialEq, RuntimeDebug, TypeInfo, MaxEncodedLen)]
pub struct RegistrationInfo {
    pub timestamp: u64,
    pub referrer: Option<AccountId>,
}
```

---

## #[pallet::call] 可调用函数

```rust
#[pallet::call]
impl<T: Config> Pallet<T> {
    #[pallet::call_index(0)]
    #[pallet::weight(T::WeightInfo::register())]
    pub fn register(
        origin: OriginFor<T>,
        nickname: BoundedVec<u8, T::MaxNameLength>,
    ) -> DispatchResultWithPostInfo {
        let who = ensure_signed(origin)?;

        ensure!(
            !Profiles::<T>::contains_key(&who),
            Error::<T>::AlreadyRegistered,
        );

        let profile = Profile {
            nickname,
            reputation: 0,
            level: 1,
            registered_at: frame_system::Pallet::<T>::block_number(),
        };

        Profiles::<T>::insert(&who, profile);
        AccountCount::<T>::insert(&who, ());

        Self::deposit_event(Event::Registered {
            who,
            block_number: frame_system::Pallet::<T>::block_number(),
        });

        Ok(().into())
    }

    #[pallet::call_index(1)]
    #[pallet::weight(T::WeightInfo::update_reputation())]
    pub fn update_reputation(
        origin: OriginFor<T>,
        target: T::AccountId,
        delta: i64,
    ) -> DispatchResultWithPostInfo {
        let who = ensure_signed(origin)?;

        ensure!(
            who != target,
            Error::<T>::CannotUpdateSelf,
        );

        Profiles::<T>::try_mutate(&target, |maybe_profile| -> DispatchResult {
            let profile = maybe_profile.as_mut()
                .ok_or(Error::<T>::NotRegistered)?;

            if delta >= 0 {
                profile.reputation = profile.reputation
                    .checked_add(delta as u64)
                    .ok_or(Error::<T>::ArithmeticOverflow)?;
            } else {
                profile.reputation = profile.reputation
                    .checked_sub(delta.unsigned_abs())
                    .ok_or(Error::<T>::ArithmeticUnderflow)?;
            }

            Ok(())
        })?;

        Self::deposit_event(Event::ReputationUpdated {
            updater: who,
            target,
            new_reputation: Profiles::<T>::get(&target)
                .map(|p| p.reputation)
                .unwrap_or(0),
        });

        Ok(().into())
    }

    #[pallet::call_index(2)]
    #[pallet::weight(T::WeightInfo::deregister())]
    pub fn deregister(origin: OriginFor<T>) -> DispatchResultWithPostInfo {
        let who = ensure_signed(origin)?;

        ensure!(
            Profiles::<T>::contains_key(&who),
            Error::<T>::NotRegistered,
        );

        Profiles::<T>::remove(&who);
        AccountCount::<T>::remove(&who);

        Self::deposit_event(Event::Deregistered { who });

        Ok(().into())
    }
}
```

---

## #[pallet::event] 事件定义

```rust
#[pallet::event]
#[pallet::generate_deposit(pub(super) fn deposit_event)]
pub enum Event<T: Config> {
    Registered {
        who: T::AccountId,
        block_number: BlockNumberFor<T>,
    },
    ReputationUpdated {
        updater: T::AccountId,
        target: T::AccountId,
        new_reputation: u64,
    },
    Deregistered {
        who: T::AccountId,
    },
    LevelUp {
        who: T::AccountId,
        old_level: u32,
        new_level: u32,
    },
}
```

---

## #[pallet::error] 错误定义

```rust
#[pallet::error]
pub enum Error<T> {
    AlreadyRegistered,
    NotRegistered,
    CannotUpdateSelf,
    ArithmeticOverflow,
    ArithmeticUnderflow,
    NicknameTooLong,
    NicknameTooShort,
}
```

---

## Weights 基准测试

### weight 定义文件

```rust
pub trait WeightInfo {
    fn register() -> Weight;
    fn update_reputation() -> Weight;
    fn deregister() -> Weight;
}

pub struct SubstrateWeight<T>(PhantomData<T>);

impl<T: frame_system::Config> WeightInfo for SubstrateWeight<T> {
    fn register() -> Weight {
        Weight::from_parts(15_000_000, 1500)
            .saturating_add(T::DbWeight::get().reads(2))
            .saturating_add(T::DbWeight::get().writes(2))
    }

    fn update_reputation() -> Weight {
        Weight::from_parts(10_000_000, 1000)
            .saturating_add(T::DbWeight::get().reads(1))
            .saturating_add(T::DbWeight::get().writes(1))
    }

    fn deregister() -> Weight {
        Weight::from_parts(8_000_000, 800)
            .saturating_add(T::DbWeight::get().reads(1))
            .saturating_add(T::DbWeight::get().writes(2))
    }
}

impl WeightInfo for () {
    fn register() -> Weight {
        Weight::from_parts(15_000_000, 1500)
            .saturating_add(RocksDbWeight::get().reads(2))
            .saturating_add(RocksDbWeight::get().writes(2))
    }

    fn update_reputation() -> Weight {
        Weight::from_parts(10_000_000, 1000)
            .saturating_add(RocksDbWeight::get().reads(1))
            .saturating_add(RocksDbWeight::get().writes(1))
    }

    fn deregister() -> Weight {
        Weight::from_parts(8_000_000, 800)
            .saturating_add(RocksDbWeight::get().reads(1))
            .saturating_add(RocksDbWeight::get().writes(2))
    }
}
```

### benchmark 命令行

```bash
./target/release/node-template benchmark pallet \
  --chain dev \
  --execution wasm \
  --wasm-execution compiled \
  --pallet pallet_profile \
  --extrinsic "*" \
  --steps 50 \
  --repeat 20 \
  --output pallets/profile/src/weights.rs
```

### 使用从 Runtime 获取的数据进行动态 weight

```rust
#[pallet::call_index(3)]
#[pallet::weight(
    Weight::from_parts(10_000, 0)
        .saturating_add(T::DbWeight::get().reads(1_u64))
        .saturating_add(T::DbWeight::get().writes(1_u64))
)]
pub fn level_up(origin: OriginFor<T>) -> DispatchResultWithPostInfo {
    let who = ensure_signed(origin)?;

    Profiles::<T>::try_mutate(&who, |maybe_profile| -> DispatchResult {
        let profile = maybe_profile.as_mut()
            .ok_or(Error::<T>::NotRegistered)?;

        let old_level = profile.level;
        profile.level = profile.level
            .checked_add(1)
            .ok_or(Error::<T>::ArithmeticOverflow)?;

        Self::deposit_event(Event::LevelUp {
            who: who.clone(),
            old_level,
            new_level: profile.level,
        });

        Ok(())
    })?;

    Ok(().into())
}
```

---

## construct_runtime! 组装 Runtime

```rust
use frame_support::construct_runtime;

construct_runtime! {
    pub enum Runtime {
        System: frame_system,
        Timestamp: pallet_timestamp,
        Balances: pallet_balances,
        Profile: pallet_profile,
        Treasury: pallet_treasury,
    }
}
```

### Runtime 配置

```rust
impl pallet_profile::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type MaxNameLength = ConstU32<64>;
    type WeightInfo = pallet_profile::weights::SubstrateWeight<Runtime>;
}

impl pallet_balances::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type Balance = u128;
    type DustRemoval = ();
    type ExistentialDeposit = ConstU128<500>;
    type AccountStore = System;
    type WeightInfo = pallet_balances::weights::SubstrateWeight<Runtime>;
    type MaxLocks = ConstU32<50>;
    type MaxReserves = ConstU32<50>;
    type ReserveIdentifier = [u8; 8];
    type RuntimeHoldReason = ();
    type RuntimeFreezeReason = ();
    type FreezeIdentifier = ();
    type MaxFreezes = ConstU32<0>;
}
```

---

## Pallet 测试

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::mock::*;
    use frame_support::{assert_noop, assert_ok};

    #[test]
    fn register_account_works() {
        new_test_ext().execute_with(|| {
            System::set_block_number(1);

            assert_ok!(Profile::register(
                RuntimeOrigin::signed(1),
                BoundedVec::try_from(b"Alice".to_vec()).unwrap(),
            ));

            let profile = Profiles::<Test>::get(1).unwrap();
            assert_eq!(profile.reputation, 0);
            assert_eq!(profile.level, 1);
            assert_eq!(AccountCount::<Test>::count(), 1);
        });
    }

    #[test]
    fn duplicate_register_fails() {
        new_test_ext().execute_with(|| {
            assert_ok!(Profile::register(
                RuntimeOrigin::signed(1),
                BoundedVec::try_from(b"Alice".to_vec()).unwrap(),
            ));

            assert_noop!(
                Profile::register(
                    RuntimeOrigin::signed(1),
                    BoundedVec::try_from(b"Bob".to_vec()).unwrap(),
                ),
                Error::<Test>::AlreadyRegistered,
            );
        });
    }
}
```

---

## Red Lines

| 规则 | 严重程度 | 说明 |
|------|----------|------|
| 所有 extrinsic 必须有 benchmark 派生的 weight | CRITICAL | 无界 weight → 区块溢出。 |
| 所有状态变更必须 deposit_event | HIGH | 否则索引器和前端无法感知变化。 |
| 存储 key 必须抗碰撞（Blake2_128Concat） | HIGH | Twox64Concat 可被碰撞。 |
| 不要绕过 try_mutate | CRITICAL | 直接 mutate 可能导致重入不一致。 |
| construct_runtime! 中 pallet 顺序影响交易执行顺序 | HIGH | 越早声明越早执行。 |

---

## References

- [Substrate FRAME Docs](https://docs.substrate.io/reference/frame-pallets/)
- [frame-support crate](https://docs.rs/frame-support/latest/frame_support/)
- [frame-system crate](https://docs.rs/frame-system/latest/frame_system/)
- [Substrate Weight Benchmarking](https://docs.substrate.io/test/benchmark/)
- [construct_runtime!](https://docs.rs/frame-support/latest/frame_support/macro.construct_runtime.html)