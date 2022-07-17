---
title: 在基于Substrate的区块链上写智能合约
date: 2022-07-17 18:34:17
---

这几天按耐不住就是要折腾区块链，所以又用 Substrate 起了条链。使用 Substrate 主要是被它的设计吸引了，通过 Runtime 和 Node 分离，Runtime 功能又通过 Pallet 实现，极大程度的降低了耦合度，使得可以通过它开发出任意区块链。

<!--more-->

Parity团队也为其设计了一个合约 Pallet ，跑 WASM 的合约，同时又做了一个叫做 ink! 的智能合约 eDSL。那么写一个在上面跑的合约，主要就是用 ink! 咯。

核心思想方面， ink! 与 solidity 真的相差不大。如果有 sol 基础的话，编写 ink! 将会很轻松。

> 注意：本文假设您已有一定 Rust 和 Solidity 基础，并且对区块链和智能合约有一定了解

## 1. 底层逻辑

智能合约，即为区块链上运行的代码，通过交易触发。智能合约运行在图灵完备的虚拟机里，通过 gas 机制防止 DoS 攻击。以太坊采用 EVM 来执行合约代码，Merkle Patricia Tree 来存储状态。Solidity 之类高级合约语言经过编译生成 EVM 代码，并通过创建合约的交易部署上链。合约一旦被创建无法修改，只能自毁。

### 计算

在虚拟机中，执行的指令只能读取规定的内存以及状态数据，并且报错后状态数据将被还原。
每行指令所消耗的 gas 相加计算出总 gas。

### 存储

大部分区块链采用的状态存储基于 KV 数据库，Substrate 也不例外。

![摘自官方doc](https://doc.kmf.com/ke-feedback/2022/07/17/18/50/36/20220717185034.png)

基本上就是想办法把各种数据结构塞到这样子的数据库内。

substrate 合约采用的方式为将顺序的数据顺序存储到 kv 数据库内，就是这样

``` rust
#[ink(storage)]  
pub struct Spread {  
	a: i32,  
	b: [u8; 32],  
}
```

![](https://doc.kmf.com/ke-feedback/2022/07/17/18/52/43/20220717185243.png)

这个叫做 Spreading，结构体需要实现 SpreadLayout。


## 2. 创建项目

假设你已经配置好了 Rust

需要首先安装些依赖项

```bash
cargo install cargo-dylint dylint-link

cargo install cargo-contract --force --locked
```

接下来可以到任意文件夹执行

```bash
cargo contract new <Project>
```

创建新的项目，项目结构很简单

```
Project
└─ lib.rs <-- Contract Source Code  
└─ Cargo.toml <-- Rust Dependencies and ink! Configuration
```

`lib.rs` 就是你要写的合约代码

## 3. 代码结构

新创建出来的会是官方最简单的示例

```rust
use ink_lang as ink;

#[ink::contract]
mod flipper {
    /// The storage of the flipper contract.
    #[ink(storage)]
    pub struct Flipper {
        /// The single `bool` value.
        value: bool,
    }

    impl Flipper {
        /// Instantiates a new Flipper contract and initializes
        /// `value` to `init_value`.
        #[ink(constructor)]
        pub fn new(init_value: bool) -> Self {
            Self {
                value: init_value,
            }
        }

        /// Flips `value` from `true` to `false` or vice versa.
        #[ink(message)]
        pub fn flip(&mut self) {
            self.value = !self.value;
        }

        /// Returns the current state of `value`.
        #[ink(message)]
        pub fn get(&self) -> bool {
            self.value
        }
    }

    /// Simply execute `cargo test` in order to test your contract
    /// using the below unit tests.
    #[cfg(test)]
    mod tests {
        use super::*;
        use ink_lang as ink;

        #[ink::test]
        fn it_works() {
            let mut flipper = Flipper::new(false);
            assert_eq!(flipper.get(), false);
            flipper.flip();
            assert_eq!(flipper.get(), true);
        }
    }
}
```

除去必备依赖，有主要几个特征：

1. `#[ink::contract]` 和 `mod flipper` 是每个合约必不可少的定义，因为编译时将整个mod作为合约打包
2. 宏 `#[ink(storage)]` 则是其存储数据结构的定义，下面接这个合约的 `Struct`
3. `impl Flipper` 内定义合约的函数
4. `#[ink(constructor)]` 下定义的是合约的初始化函数，与Solidity合约的初始化如出一辙，返回Self。一个合约可以有多个构造方法，可以在部署时候指定调用哪个。
5. `#[ink(message)]` 则是具体合约执行的代码，传入的第一项总是`Self`。通过Self可以读取合约自身的状态。`&mut self` 同样指示了函数为更改区块链状态的函数，而 `&self` 只是简单的get函数。与sol一样，这里的函数可以是private。
6. `#[cfg(test)]` 部分可以自己写单元测试。我个人更喜欢直接上链测试。
7. 这里没有讲到宏 `#[ink(event)]` 。这个和以太坊合约的event是一样的，合约可以发出event，并且客户端可以选择监听。
8. 如果合约函数需要处理错误，则可以返回 `Error`

这个例子还是比较简单的。

如果需要对函数进行错误处理，则可以这样：

```rust
mod contract {
	use scale::{
        Decode,
        Encode,
        Output,
    };

	#[derive(Encode, Decode, Debug, PartialEq, Eq, Copy, Clone)]
    #[cfg_attr(feature = "std", derive(scale_info::TypeInfo))] // 一定要有的派生
    pub enum Error { // 所有错误类型都写到这个enum里面
	    Error1,
	    Error2,
    }

	impl contract {
		#[ink(message)]
		pub fn raise_error(&self, num: u8) -> Result<(), Error> { //函数返回值可以是Result和Option
			if num == 1 {
				Err(Error::Error1)
			} else {
				Err(Error::Error2)
			}
		}
	}
}

```


## 4. 复杂数据

和Solidity一样，必不可少的数据结构有一个叫做 Mapping，通过  `use ink_storage::Mapping;` 导入它。

使用也很简单，定义它存储的key和value的类型，key和value都得满足 `scale::EncodeLike<PackedLayout>` 这个 trait。
```rust
pub struct contract {
    balances: Mapping<AccountId, Balance>,
}
```

如果想要把自定义的结构体放进去的话，需要加上一些derive

```rust
use ink_storage::{
    traits::{
        PackedLayout,
        SpreadAllocate,
        SpreadLayout,
    },
};

#[derive(scale::Encode, scale::Decode, SpreadLayout, PackedLayout)]
#[cfg_attr(
	feature = "std",
    derive(
        Debug,
        PartialEq,
        Eq,
        scale_info::TypeInfo,
        ink_storage::traits::StorageLayout
    )
)]

pub struct Info {
    pub uri: String,
    pub creator: AccountId,
}
```

还有两个可以任意改长度的数据结构 `Vec` 和 `String` ，他们都在 `ink_prelude` 这个 crate 里面，自行在 Cargo.toml 里导入就可以。
```
use ink_prelude::string::String;
use ink_prelude::vec::Vec;
```


## 5. 编译

当你写完了，可以直接运行

```bash
cargo +nightly contract build
```

生成一个wasm，一个json，加上一个两个的结合体contract

通过 https://polkadot.js.org/apps/ 或者 https://contracts-ui.substrate.io/ 都可以传到链上并且调用。

## 6. 结束

这是我踩过的坑，如果你想要了解更多 ink!，可以去官方的doc站点 https://ink.substrate.io/

> substrate yyds
	rust yyds