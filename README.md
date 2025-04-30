# Solana-on-chain-programs-native-vs-anchor
Comparison of core technical concepts for Anchor and native Solana on-chain programs.

[![art-2-1-01.png](https://i.postimg.cc/5NjZvS49/art-2-1-01.png)](https://postimg.cc/JGV6CJHS)

# Introduction

This post explores the core development aspects of Solana on-chain development, comparing approaches using native Rust (without a framework) and the Anchor framework.
We will delve into key development aspects such as account data management, serialization, and deserialization, instruction data matching with program instructions, and general security checks for the Solana on-chain program. 

To illustrate these concepts, we’ll provide example programs for a counter implemented in native Rust and the Anchor framework.
Later in the post, we will discuss how the Anchor framework simplifies Solana's on-chain program development.

# What you will learn

in this post, you will learn 

- How Solana’s on-chain program works with native rust
- How Anchor simplifies Solana’s on-chain program development
- Basic technical concept comparison with Anchor framework

# Basic Hello World Program In Native Rust

In this native Solana program, there is just one main entry point, where we call the program's instructions. This instruction data needs to be serialized and deserialized by Borsh. The entry point macro includes the program ID, a list of accounts used in the instruction, and the instruction data.
We use the solana_program crate, the base library for writing on-chain programs in Rust, to develop native on-chain programs.
In the native program, we need to pass accounts and instruction data to process an instruction explicitly.

```rust
use solana_program::{
    account_info::AccountInfo,
    entrypoint,
    entrypoint::ProgramResult,
    pubkey::Pubkey,
    msg
};
 
entrypoint!(process_instruction);
 
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8]
) -> ProgramResult{
    msg!("Hello, world!");
 
    Ok(())
}
```

In the above code:

- We used the Solana Program crate to import:
    - `account_info::AccountInfo` struct for account information, such as account balance and account data.
    - `entrypoint` for processing program instructions.
    - `ProgramResult`, is a type representing the result that a Solana program returns.
    - `pubkey::Pubkey`, a type of Ed25519 public key.
    - The `msg` macro prints messages to the program log.
- The `entrypoint!(process_instruction);` macro is used with the `process_instruction` instruction handler.
- The `process_instruction` function is used to write the business logic for program instructions. In this program, it takes the program ID, a list of accounts used in the instruction, and the serialized instruction data as arguments and it simply prints "Hello, world!" to the program log.

# Basic Hello World Program In Anchor Rust

Anchor writes various boilerplate code for you, such as the serialization and deserialization of accounts and instruction data.
Anchor handles certain security checks for you by providing Rust macros and attributes.
The Anchor framework provides various types of modules, traits, and macros to reduce the boilerplate needed to write Solana programs.

Besides the native Solana program, an Anchor program generally has three main parts:

- The program ID, which is stored using the `declare_id` macro.
- The `#[program]` module, where we write the program’s business logic or instruction handlers.
- The account validation struct, where we create and verify all accounts used by the instruction handlers.

```rust
use anchor_lang::prelude::*;

declare_id!("448KahuLdmiQpbrdGziRuQ6Y3mYG3uKWCWzcV3U9UWz6");

#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(_ctx: Context<Initialize>) -> ProgramResult {
    msg!("hello world");
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

In an Anchor Solana program, all programs and components are included via `anchor_lang::prelude::*`.
This re-exports the solana_program crate.
The `declare_id!()` macro stores the on-chain program address.
In the `#[program]` module, we define our first instruction named `initialize`, which takes `Context<Initialize>` as an argument and simply prints "hello world" to the program log.

- Context provides non-argument inputs to the program.

`Context` is a generic type where `T` defines the list of accounts an instruction requires. When you use `Context`, you specify the concrete type of `T` as a struct that adopts the `Accounts` trait (e.g. `Context<Initialize>`). Through this context argument, the instruction can then access:

- The accounts passed into the instruction (`ctx.accounts`)
- The program ID (`ctx.program_id`) of the executing program
- The remaining accounts (`ctx.remaining_accounts`). The `remaining_accounts` is a vector that contains all accounts that were passed into the instruction but are not declared in the `Accounts` struct.
- The bumps for any PDA accounts in the `Accounts` struct (`ctx.bumps`)

The `#[derive(Accounts)]` attribute serializes and deserializes account data.

This program simply prints the "hello world" message and does not require accounts to store any data.

# Basic Counter program in native rust

### Program features:-

- Initialize the counter
- Increment the counter by one

To write the counter program open https://beta.solpg.io/ and create a new project name it counter and choose native rust as a framework.

Create a new file named `state.rs` under the src directory and add this code

### Define state for counter Account

```rust
/// state.rs

use borsh::{BorshDeserialize, BorshSerialize};

#[derive(BorshSerialize, BorshDeserialize, Debug, Clone)]
pub struct Counter {
    pub count: u64,
}
```

As Solana programs are stateless and only store program logic code, we need to store the counter value in a separate account and pass it explicitly to the counter Solana program.
Instruction data is passed to the program as a byte array, so we need to convert that array into an instance of the instruction struct type.
Here, we are using the Borsh Rust crate to implement traits on the Counter struct using the derive attribute.
These traits deserialize struct data into a byte array and serialize a byte array into a Rust struct type.

### Counter program entrypoint

In the native Solana program, we need to manually

- Send incoming instructions to the correct instruction handlers.
- Deserialize data from incoming transactions.
- Check the accounts provided with incoming instructions; for example, verify that certain accounts are of a particular type or are distinct from other accounts.

Next, in `lib.rs` file, add this code

```rust
/// lib.rs

use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    declare_id,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};

mod state;
use state::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[cfg(not(feature = "no-entrypoint"))]
use solana_program::entrypoint;

#[cfg(not(feature = "no-entrypoint"))]
entrypoint!(process_instruction);

pub fn process_instruction(
    _program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let (instruction_discriminant, instruction_data_inner) = instruction_data.split_at(1);
    match instruction_discriminant[0] {
        0 => {
            msg!("Instruction: Increment");
            process_increment_counter(accounts, instruction_data_inner)?;
        }
        _ => {
            msg!("Error: unknown instruction")
        }
    }
    Ok(())
}

pub fn process_increment_counter(
    accounts: &[AccountInfo],
    _instruction_data: &[u8], 
) -> Result<(), ProgramError> {
    let account_info_iter = &mut accounts.iter();

    let counter_account = next_account_info(account_info_iter)?;
    assert!(
        counter_account.is_writable,
        "Counter account must be writable"
    );

    let mut counter = Counter::try_from_slice(&counter_account.try_borrow_mut_data()?)?;
    counter.count += 1;
    counter.serialize(&mut *counter_account.data.borrow_mut())?;

    msg!("Counter state incremented to {:?}", counter.count);
    Ok(())
}
```

let’s break it down:-

In the following code block:-
We import the `borsh` crate to serialize and deserialize account data.
Next, we import some required modules and structs from the Solana program crate:

- `account_info::{next_account_info, AccountInfo}`: This import is used to iterate over accounts and fetch account information.
- `declare_id`: This is a convenience macro to declare a static public key and functions to interact with it.
- `entrypoint::ProgramResult`: This type within the `entrypoint` module returns either a `Result` or a `ProgramError`.
- The `msg` macro prints a message to the log.
- `program_error::ProgramError` defines the reason for program failure.
- `Pubkey` is the Solana account public key.

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    declare_id,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};

```

Next, we import the state module 

```rust
mod state;
use state::*;
```

Next, the declare id macro is used to declare a static public key and functions to interact with it.

```rust
declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");
```

Next, Define an entrypoint and make it conditional on the `no-entrypoint` feature.

```rust
#[cfg(not(feature = "no-entrypoint"))]
use solana_program::entrypoint;
```

Next, call entrypoint by providing the argument process_instruction and make it conditional on the `no-entrypoint` feature.

```rust
#[cfg(not(feature = "no-entrypoint"))]
entrypoint!(process_instruction);
```

Next, the `fn process_instruction` function takes three arguments as reference:

- `_program_id: &Pubkey`
- `accounts: &[AccountInfo]`
- `instruction_data: &[u8]`

and returns a `ProgramResult`.
In `process_instruction`, we split the instruction data and destructure it into a tuple.
The first element of the tuple takes the first byte from the instruction data and determines which instruction to execute; in our case, it's `process_increment_counter`.
Next, we match the `instruction_discriminant` to determine which data structure is expected by the function and return the instruction or an error.

It's standard practice to structure a program to expect the first byte (or another fixed number of bytes) to be an identifier for which instruction the program should run. This could be an integer or a string identifier. In this example, we'll use the first byte and map integers to instructions, with `0` corresponding to `process_increment_counter`.

```rust
pub fn process_instruction(
    _program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let (instruction_discriminant, instruction_data_inner) = instruction_data.split_at(1);
    match instruction_discriminant[0] {
        0 => {
            msg!("Instruction: Increment");
            process_increment_counter(accounts, instruction_data_inner)?;
        }
        _ => {
            msg!("Error: unknown instruction")
        }
    }
    Ok(())
}
```

Next, the `process_increment_counter` function:

- Takes `accounts` and `instruction_data` as arguments.
- Returns a `Result` and a `ProgramError`.

In the function body:

- It first creates a mutable iterator instance over the accounts.
- Then, it fetches the counter account using `next_account_info` and asserts whether this account is writable (mutable) or not.
- Next, we deserialize the `Counter` struct account data using the `try_from_slice` method and increment the counter by one.
- We then serialize the updated data.
- Finally, we print the program log.

```rust
pub fn process_increment_counter(
    accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> Result<(), ProgramError> {
    let account_info_iter = &mut accounts.iter();

    let counter_account = next_account_info(account_info_iter)?;
    assert!(
        counter_account.is_writable,
        "Counter account must be writable"
    );

    let mut counter = Counter::try_from_slice(&counter_account.try_borrow_mut_data()?)?;
    counter.count += 1;
    counter.serialize(&mut *counter_account.data.borrow_mut())?;

    msg!("Counter state incremented to {:?}", counter.count);
    Ok(())
}
```

This is how native Solana programs work. We need to handle common tasks such as serialization and deserialization of account data, instruction data matching, converting raw data bytes to specified structs, and much more.

# Basic Counter program in Anchor Rust

The Anchor framework uses macros and traits to generate boilerplate Rust code for you.

Anchor framework handles common grunt work, such as reading data from incoming instructions and checking that the correct accounts are provided. It also offers useful crates, macros, and attributes to ensure a better developer experience.

Anchor provides 

- [IDL](https://en.wikipedia.org/wiki/Interface_description_language) specification
- TypeScript package for generating clients from IDL

The `Context` type exposes instruction metadata and accounts for instruction logic, so manual account iteration and deserialization are no longer necessary.

In this counter program, which is the same as the native version, we do not need to serialize and deserialize account data manually, nor do we need to match instruction handler data. We need to just create separate Contexts for instruction handlers.

```rust 

/// lib.rs
#![allow(clippy::result_large_err)]

use anchor_lang::prelude::*;

declare_id!("BmDHboaj1kBUoinJKKSRqKfMeRKJqQqEbUj1VgzeQe4A");

#[program]
pub mod counter_anchor {
    use super::*;

    pub fn initialize_counter(_ctx: Context<InitializeCounter>) -> Result<()> {
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        ctx.accounts.counter.count = ctx.accounts.counter.count.checked_add(1).unwrap();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeCounter<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        space = 8 + Counter::INIT_SPACE,
        payer = payer
    )]
    pub counter: Account<'info, Counter>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
}

#[account]
#[derive(InitSpace)]
pub struct Counter {
    count: u64,
}
```

Using the above code, we explain how Anchor uses types, attributes, and macros to simplify Solana’s on-chain development. 

This import, `use anchor_lang::prelude::*;`, contains all the commonly used components of the crate. All programs should include this line, which also re-exports the `solana-program` crate.

### Declare ID part

This line sets the program ID for the program using a macro to declare the program’s on-chain address.

```rust
declare_id!("BmDHboaj1kBUoinJKKSRqKfMeRKJqQqEbUj1VgzeQe4A");
```

### Instruction handler part

Using the `#[program]` attribute, we can define the module containing all instruction handlers(aka RPC endpoints), which define all entries into a Solana program.

Each public function in the module with the `#[program]` attribute will be treated as a separate instruction.

Each instruction handler requires a parameter of type `Context` and can optionally include additional function parameters representing instruction data. Anchor will automatically handle instruction data deserialization so that you can work with instruction data as Rust types.

This is our first instruction handler that initializes the counter account and returns ok(())

```rust
   pub fn initialize_counter(_ctx: Context<InitializeCounter>) -> Result<()> {
        Ok(())
    }
```

This takes the `Context` type of `<InitializeCounter>` as an argument and returns the result.

The `Context` type exposes instruction metadata and accounts for instruction logic. 

Under the hood, Context structure looks like this:-

```rust
pub struct Context<'a, 'b, 'c, 'info, T> {
    /// Currently executing program id.
    pub program_id: &'a Pubkey,
    /// Deserialized accounts.
    pub accounts: &'b mut T,
    /// Remaining accounts given but not deserialized or validated.
    /// Be very careful when using this directly.
    pub remaining_accounts: &'c [UncheckedAccount<'info>],
    /// Bump seeds found during constraint validation. This is provided as a
    /// convenience so that handlers don't have to recalculate bump seeds or
    /// pass them in as arguments.
    pub bumps: BTreeMap<String, u8>,
}
```

This is our second instruction handler that takes the Context type of `<Increment>` as an argument and increment counter count by one and returns ok(()) for successful operation.

```rust
pub fn increment(ctx: Context<Increment>) -> Result<()> {
        ctx.accounts.counter.count = ctx.accounts.counter.count.checked_add(1).unwrap();
        Ok(())
    }
```

As discussed in `Context` structure, we can access accounts using dot annotation like this:-

```rust
ctx.accounts.counter.count
```

### Account creation and validation part

### Counter state:-

First, let's use the `#[account]` attribute to define a new `Counter` account type. The `Counter` struct defines one `count` field of type `u64`. This means that any new accounts initialized as a `Counter` type will have a matching data structure. The `#[account]` attribute also automatically sets the discriminator for a new account and sets the owner of the account as the `programId` from the `declare_id!` macro.

In anchor, you can initialize an account using the `#[account]` attribute

Here, InitSpace implements a [`Space`](https://docs.rs/anchor-lang/latest/anchor_lang/trait.Space.html) trait on the Counter struct and space is used to define the space of an account for initialization.

```rust
#[account]
#[derive(InitSpace)]
pub struct Counter {
    count: u64,
}
```

### InitializeCounter struct

This is the phase where we check account validation, perform security checks, and verify the types of accounts used in the instruction

```rust
#[derive(Accounts)]
pub struct InitializeCounter<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        space = 8 + Counter::INIT_SPACE,
        payer = payer
    )]
    pub counter: Account<'info, Counter>,
    pub system_program: Program<'info, System>,
}

```

This is the `InitializeCounter` struct for the `initialize_counter` instruction handler.

On this struct, we use the `#[derive(Accounts)]` attribute to implement an [`Accounts`](https://docs.rs/anchor-lang/latest/anchor_lang/trait.Accounts.html) deserializer on the given struct, so we do not need to deserialize account data as in a native Solana program manually.

In this struct, the first account is `payer`. The attribute `#[account(mut)]` declares that this is an account and that it is mutable.

Additionally, Anchor provides some types of accounts for security checks and validations. For example, this `payer` account has the `signer` type, which means it will sign transactions. In our case, it is also the payer for account creation.

In a native Solana program, we need to check if this account is a signer or not like this

```rust
if !initializer.is_signer {
msg!("Missing required signature");
return Err(ProgramError::MissingRequiredSignature)
}
```

Next, we have the `counter` account, and it has constraints like this:

`#[account(init, space = 8 + Counter::INIT_SPACE, payer = payer)]`

- `init` is used to create a new account.
- `space` specifies the amount of space to allocate.
- `payer` indicates who will pay for the account creation.

This is an `Account` type that will store data of type `Counter`

Lastly, we have the `system_program` account, and this is a type of System Program used for creating new accounts.

### Increment count struct

```rust
#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
}
```

This is the `Increment` struct for the `increment` instruction handler.

It contains a mutable `counter` account for updating its count.

# Conclusion

In this post, we delve into the native version of a Solana program and the Anchor version, highlighting the differences in what Anchor does behind the scenes. In a native program, tasks like account data serialization, deserialization, and instruction data matching must be done manually. We also elaborate on how Anchor simplifies these processes using the provided macros and traits.

In future lessons, we will use Anchor for Solana development because it is more beginner-friendly for new developers.


