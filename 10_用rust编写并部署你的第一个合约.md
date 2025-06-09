# 用Rust编写并部署你的第一个合约

到目前为止，我们主要讨论的是Solidity合约。但区块链世界正在快速发展，Rust作为一种更安全、更高效的编程语言，正在成为智能合约开发的新选择。

如果你已经熟悉了Solidity，学习Rust合约开发会让你获得更多的技术栈选择。如果你是编程新手，Rust的严格性也会让你养成更好的编程习惯。

## 为什么选择Rust？

### Rust的优势
- **内存安全**：编译期防止内存泄漏和悬空指针
- **性能优异**：接近C++的性能，但更安全
- **并发安全**：编译期防止数据竞争
- **表达能力强**：丰富的类型系统和模式匹配
- **生态丰富**：成熟的包管理器和工具链

### 在区块链中的应用
- **Solana**：使用Rust作为主要开发语言
- **NEAR Protocol**：支持Rust合约
- **Polkadot**：Substrate框架使用Rust
- **Cosmos**：CosmWasm支持Rust

## 选择目标平台

本章我们将学习两个主要平台：

### 1. Solana（推荐新手）
- 高性能区块链
- 交易成本极低
- 开发工具成熟
- 社区活跃

### 2. NEAR Protocol
- 对开发者友好
- 支持渐进式去中心化
- Gas费用合理
- 易于学习

我们先从Solana开始。

## 环境搭建

### 安装Rust
```bash
# 安装Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# 验证安装
rustc --version
cargo --version
```

### 安装Solana CLI
```bash
# 安装Solana工具链
sh -c "$(curl -sSfL https://release.solana.com/v1.16.0/install)"

# 更新PATH
export PATH="/home/用户名/.local/share/solana/install/active_release/bin:$PATH"

# 验证安装
solana --version
```

### 配置开发环境
```bash
# 设置为开发网络
solana config set --url https://api.devnet.solana.com

# 创建新的密钥对
solana-keygen new --outfile ~/.config/solana/id.json

# 获取一些测试SOL
solana airdrop 2
```

## 第一个Rust合约

### 创建项目
```bash
# 创建新的Rust项目
cargo new --lib hello-solana
cd hello-solana
```

### 配置Cargo.toml
```toml
[package]
name = "hello-solana"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[dependencies]
solana-program = "~1.16"
borsh = "0.10.3"
borsh-derive = "0.10.3"

[features]
no-entrypoint = []
```

### 编写合约代码

**src/lib.rs**
```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};

// 定义合约状态
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
    pub counter: u32,
}

// 声明并导出程序的入口点
entrypoint!(process_instruction);

// 程序入口点，这里处理所有的指令
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");

    // 获取账户迭代器
    let accounts_iter = &mut accounts.iter();

    // 获取调用合约的账户
    let account = next_account_info(accounts_iter)?;

    // 检查账户所有者是否是我们的程序
    if account.owner != program_id {
        msg!("Greeted account does not have the correct program id");
        return Err(ProgramError::IncorrectProgramId);
    }

    // 增加并存储问候次数
    let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
    greeting_account.counter += 1;
    greeting_account.serialize(&mut &mut account.data.borrow_mut()[..])?;

    msg!("Greeted {} time(s)!", greeting_account.counter);

    Ok(())
}
```

### 编译合约
```bash
# 编译为BPF字节码
cargo build-bpf
```

编译成功后，会在`target/deploy/`目录下生成：
- `hello_solana.so`：合约字节码
- `hello_solana-keypair.json`：合约地址密钥

## 部署合约

### 部署到开发网
```bash
# 部署合约
solana program deploy target/deploy/hello_solana.so

# 输出示例：
# Program Id: 7pUAybVqcX3X2wNa2RK4UFp7fL5c3FBDZuJYQMvPumrG
```

记住这个Program Id，后面调用合约时会用到。

### 验证部署
```bash
# 查看程序信息
solana program show 7pUAybVqcX3X2wNa2RK4UFp7fL5c3FBDZuJYQMvPumrG
```

## 与合约交互

现在我们需要写一个客户端程序来调用合约。

### 创建客户端项目
```bash
# 创建新目录
mkdir hello-client
cd hello-client
cargo init
```

### 客户端代码

**Cargo.toml**
```toml
[package]
name = "hello-client"
version = "0.1.0"
edition = "2021"

[dependencies]
solana-client = "~1.16"
solana-sdk = "~1.16"
solana-program = "~1.16"
borsh = "0.10.3"
borsh-derive = "0.10.3"
```

**src/main.rs**
```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_client::rpc_client::RpcClient;
use solana_program::{
    instruction::{AccountMeta, Instruction},
    pubkey::Pubkey,
    system_instruction,
};
use solana_sdk::{
    commitment_config::CommitmentConfig,
    signature::{Keypair, Signer},
    transaction::Transaction,
};
use std::str::FromStr;

// 复制合约中的数据结构
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
    pub counter: u32,
}

fn main() {
    // 连接到Solana开发网
    let rpc_url = "https://api.devnet.solana.com".to_string();
    let client = RpcClient::new_with_commitment(rpc_url, CommitmentConfig::confirmed());

    // 加载付费者密钥（用于支付交易费用）
    let payer = Keypair::from_bytes(&[
        // 这里放入你的密钥字节，实际使用时从文件读取
        // 示例密钥（不要在生产环境使用）
        174, 47, 154, 16, 202, 193, 206, 113, 199, 190, 53, 133, 169, 175, 31, 56,
        222, 53, 138, 189, 224, 216, 117, 173, 10, 149, 53, 45, 73, 251, 237, 246,
        15, 185, 186, 82, 177, 240, 148, 69, 241, 227, 167, 80, 141, 89, 240, 121,
        121, 35, 172, 247, 68, 251, 226, 218, 48, 63, 176, 109, 168, 89, 238, 135,
    ]).unwrap();

    // 程序ID（替换为你的实际程序ID）
    let program_id = Pubkey::from_str("7pUAybVqcX3X2wNa2RK4UFp7fL5c3FBDZuJYQMvPumrG").unwrap();

    // 创建用于存储计数的账户
    let greeting_account = Keypair::new();
    
    // 计算租金
    let account_size = std::mem::size_of::<GreetingAccount>();
    let rent = client
        .get_minimum_balance_for_rent_exemption(account_size)
        .unwrap();

    // 创建账户指令
    let create_account_instruction = system_instruction::create_account(
        &payer.pubkey(),
        &greeting_account.pubkey(),
        rent,
        account_size as u64,
        &program_id,
    );

    // 调用程序指令
    let greeting_instruction = Instruction::new_with_bytes(
        program_id,
        &[], // 这个例子不需要指令数据
        vec![AccountMeta::new(greeting_account.pubkey(), false)],
    );

    // 创建交易
    let mut transaction = Transaction::new_with_payer(
        &[create_account_instruction, greeting_instruction],
        Some(&payer.pubkey()),
    );

    // 获取最新的区块哈希
    let recent_blockhash = client.get_latest_blockhash().unwrap();
    transaction.sign(&[&payer, &greeting_account], recent_blockhash);

    // 发送交易
    match client.send_and_confirm_transaction(&transaction) {
        Ok(signature) => {
            println!("Success! Transaction signature: {}", signature);
            
            // 读取账户数据
            let account_data = client.get_account_data(&greeting_account.pubkey()).unwrap();
            let greeting_account_data = GreetingAccount::try_from_slice(&account_data).unwrap();
            println!("Current counter: {}", greeting_account_data.counter);
        }
        Err(err) => {
            println!("Error: {}", err);
        }
    }
}
```

### 运行客户端
```bash
# 运行客户端
cargo run
```

## 更复杂的合约示例

让我们创建一个更有趣的合约：一个简单的计数器，支持多种操作。

### 高级合约代码

**src/lib.rs**
```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};

// 定义指令枚举
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub enum CounterInstruction {
    /// 初始化计数器
    Initialize,
    /// 增加计数器
    Increment,
    /// 减少计数器
    Decrement,
    /// 重置计数器
    Reset,
}

// 定义计数器账户结构
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct CounterAccount {
    pub count: u64,
    pub authority: Pubkey,
}

entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let instruction = CounterInstruction::try_from_slice(instruction_data)?;
    
    match instruction {
        CounterInstruction::Initialize => initialize(program_id, accounts),
        CounterInstruction::Increment => increment(program_id, accounts),
        CounterInstruction::Decrement => decrement(program_id, accounts),
        CounterInstruction::Reset => reset(program_id, accounts),
    }
}

fn initialize(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let counter_account = next_account_info(accounts_iter)?;
    let authority_account = next_account_info(accounts_iter)?;

    // 检查账户所有权
    if counter_account.owner != program_id {
        return Err(ProgramError::IncorrectProgramId);
    }

    // 检查权限
    if !authority_account.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    // 初始化计数器
    let mut counter_data = CounterAccount::try_from_slice(&counter_account.data.borrow())?;
    counter_data.count = 0;
    counter_data.authority = *authority_account.key;
    counter_data.serialize(&mut &mut counter_account.data.borrow_mut()[..])?;

    msg!("Counter initialized with authority: {}", authority_account.key);
    Ok(())
}

fn increment(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let counter_account = next_account_info(accounts_iter)?;
    let authority_account = next_account_info(accounts_iter)?;

    // 检查账户所有权
    if counter_account.owner != program_id {
        return Err(ProgramError::IncorrectProgramId);
    }

    let mut counter_data = CounterAccount::try_from_slice(&counter_account.data.borrow())?;

    // 检查权限
    if counter_data.authority != *authority_account.key {
        return Err(ProgramError::InvalidAccountData);
    }

    if !authority_account.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    // 增加计数
    counter_data.count = counter_data.count.checked_add(1)
        .ok_or(ProgramError::ArithmeticOverflow)?;
    
    counter_data.serialize(&mut &mut counter_account.data.borrow_mut()[..])?;

    msg!("Counter incremented to: {}", counter_data.count);
    Ok(())
}

fn decrement(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let counter_account = next_account_info(accounts_iter)?;
    let authority_account = next_account_info(accounts_iter)?;

    if counter_account.owner != program_id {
        return Err(ProgramError::IncorrectProgramId);
    }

    let mut counter_data = CounterAccount::try_from_slice(&counter_account.data.borrow())?;

    if counter_data.authority != *authority_account.key {
        return Err(ProgramError::InvalidAccountData);
    }

    if !authority_account.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    // 减少计数（防止下溢）
    counter_data.count = counter_data.count.checked_sub(1)
        .ok_or(ProgramError::ArithmeticOverflow)?;
    
    counter_data.serialize(&mut &mut counter_account.data.borrow_mut()[..])?;

    msg!("Counter decremented to: {}", counter_data.count);
    Ok(())
}

fn reset(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let counter_account = next_account_info(accounts_iter)?;
    let authority_account = next_account_info(accounts_iter)?;

    if counter_account.owner != program_id {
        return Err(ProgramError::IncorrectProgramId);
    }

    let mut counter_data = CounterAccount::try_from_slice(&counter_account.data.borrow())?;

    if counter_data.authority != *authority_account.key {
        return Err(ProgramError::InvalidAccountData);
    }

    if !authority_account.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    // 重置计数
    counter_data.count = 0;
    counter_data.serialize(&mut &mut counter_account.data.borrow_mut()[..])?;

    msg!("Counter reset to: {}", counter_data.count);
    Ok(())
}
```

## 测试和调试

### 编写单元测试

**src/lib.rs**（添加到文件末尾）
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use solana_program::{clock::Epoch, pubkey::Pubkey};
    use solana_program_test::*;
    use solana_sdk::{signature::Signer, transaction::Transaction};

    #[tokio::test]
    async fn test_counter_initialization() {
        let program_id = Pubkey::new_unique();
        let mut program_test = ProgramTest::new(
            "hello_solana",
            program_id,
            processor!(process_instruction),
        );

        let (mut banks_client, payer, recent_blockhash) = program_test.start().await;

        // 测试代码...
    }
}
```

### 运行测试
```bash
# 运行测试
cargo test

# 运行测试并显示输出
cargo test -- --nocapture
```

## Rust vs Solidity对比

| 特性 | Rust (Solana) | Solidity (Ethereum) |
|------|---------------|---------------------|
| 内存安全 | 编译期保证 | 运行时检查 |
| 性能 | 非常高 | 中等 |
| 开发复杂度 | 中等偏高 | 中等 |
| 生态成熟度 | 发展中 | 非常成熟 |
| 交易费用 | 极低 | 较高 |
| 学习曲线 | 陡峭 | 中等 |

## 最佳实践

### 1. 错误处理
```rust
// 使用自定义错误类型
#[derive(Debug, Error)]
pub enum CounterError {
    #[error("Arithmetic overflow")]
    ArithmeticOverflow,
    #[error("Unauthorized")]
    Unauthorized,
}

impl From<CounterError> for ProgramError {
    fn from(e: CounterError) -> Self {
        ProgramError::Custom(e as u32)
    }
}
```

### 2. 账户验证
```rust
fn validate_account(account: &AccountInfo, expected_owner: &Pubkey) -> ProgramResult {
    if account.owner != expected_owner {
        return Err(ProgramError::IncorrectProgramId);
    }
    Ok(())
}
```

### 3. 数据序列化
```rust
// 使用更安全的序列化方法
fn serialize_data<T: BorshSerialize>(data: &T, account: &AccountInfo) -> ProgramResult {
    let serialized = data.try_to_vec()?;
    if serialized.len() > account.data.borrow().len() {
        return Err(ProgramError::AccountDataTooSmall);
    }
    account.data.borrow_mut()[..serialized.len()].copy_from_slice(&serialized);
    Ok(())
}
```

## 部署到主网

### 准备主网部署
```bash
# 切换到主网
solana config set --url https://api.mainnet-beta.solana.com

# 创建主网密钥
solana-keygen new --outfile ~/.config/solana/mainnet.json

# 设置为默认密钥
solana config set --keypair ~/.config/solana/mainnet.json

# 向账户转入SOL（从交易所或其他钱包）
# 检查余额
solana balance
```

### 部署合约
```bash
# 部署到主网（需要SOL支付费用）
solana program deploy target/deploy/hello_solana.so --keypair ~/.config/solana/mainnet.json
```

## 总结

Rust智能合约开发提供了：
- **更高的安全性**：编译期内存安全检查
- **更好的性能**：接近原生代码的执行速度
- **更严格的类型系统**：减少运行时错误
- **现代化的工具链**：cargo、测试框架等

虽然学习曲线比较陡峭，但Rust在区块链领域的应用前景广阔。随着Solana、NEAR等平台的发展，掌握Rust合约开发将成为区块链开发者的重要技能。

下一章我们将学习：如何在链下与合约进行交互，这是构建完整DApp的关键技术。
