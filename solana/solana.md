# Terminology

## account

A record in the Solana ledger that either holds data or is an executable program.

Like an account at a traditional bank, a Solana account may hold funds called [lamports](https://docs.solana.com/terminology#lamport). Like a file in Linux, it is addressable by a key, often referred to as a [public key](https://docs.solana.com/terminology#public-key-pubkey) or pubkey.

The key may be one of:

- an ed25519 public key
- a program-derived account address (32byte value forced off the ed25519 curve)
- a hash of an ed25519 public key with a 32 character string

## account owner

The address of the program that owns the account. Only the owning program is capable of modifying the account.

## program derived account (PDA)

An account whose owner is a program and thus is not controlled by a private key like other accounts.

## program id

The public key of the [account](https://docs.solana.com/terminology#account) containing a [program](https://docs.solana.com/terminology#program).

## signature

A 64-byte ed25519 signature of R (32-bytes) and S (32-bytes). With the requirement that R is a packed Edwards point not of small order and S is a scalar in the range of 0 <= S < L. This requirement ensures no signature malleability. Each transaction must have at least one signature for [fee account](https://docs.solana.com/terminology#fee-account). Thus, the first signature in transaction can be treated as [transacton id](https://docs.solana.com/terminology#transaction-id)

## sysvar

A system [account](https://docs.solana.com/terminology#account). [Sysvars](https://docs.solana.com/developing/runtime-facilities/sysvars) provide cluster state information such as current tick height, rewards [points](https://docs.solana.com/terminology#point) values, etc. Programs can access Sysvars via a Sysvar account (pubkey) or by querying via a syscall.

# History

From Anatoly's previous experience designing distributed systems at Qualcomm, Mesosphere and Dropbox, he knew that a reliable clock makes network synchronization very simple. When synchronization is simple the resulting network can be blazing fast, bound only by network bandwidth.

# Wallet

The *public key* (commonly shortened to *pubkey*) is known as the wallet's *receiving address* or simply its *address*. 

# Developing

An [app](https://docs.solana.com/terminology#app) interacts with a Solana cluster by sending it [transactions](https://docs.solana.com/developing/programming-model/transactions) with one or more [instructions](https://docs.solana.com/developing/programming-model/transactions#instructions). The Solana [runtime](https://docs.solana.com/developing/programming-model/runtime) passes those instructions to [programs](https://docs.solana.com/terminology#program) deployed by app developers beforehand. An instruction might, for example, tell a program to transfer [lamports](https://docs.solana.com/terminology#lamport) from one [account](https://docs.solana.com/developing/programming-model/accounts) to another or create an interactive contract that governs how lamports are transferred. Instructions are executed sequentially and atomically for each transaction. If any instruction is invalid, all account changes in the transaction are discarded.

# Accounts

Unlike a file, the account includes metadata for the lifetime of the file. That lifetime is expressed by a number of fractional native tokens called *lamports*. Accounts are held in validator memory and pay ["rent"](https://docs.solana.com/developing/programming-model/accounts#rent) to stay there. Each validator periodically scans all accounts and collects rent. Any account that drops to zero lamports is purged. Accounts can also be marked [rent-exempt](https://docs.solana.com/developing/programming-model/accounts#rent-exemption) if they contain a sufficient number of lamports.

To check an account's validity, the program should either check the account's address against a known value or check that the account is indeed owned correctly (usually owned by the program itself).

One example is when programs use a sysvar account. Unless the program checks the account's address or owner, it's impossible to be sure whether it's a real and valid sysvar account merely by successful deserialization of the account's data.

Accordingly, the Solana SDK [checks the sysvar account's validity during deserialization](https://github.com/solana-labs/solana/blob/a95675a7ce1651f7b59443eb146b356bc4b3f374/sdk/program/src/sysvar/mod.rs#L65). A alternative and safer way to read a sysvar is via the sysvar's [`get()` function](https://github.com/solana-labs/solana/blob/64bfc14a75671e4ec3fe969ded01a599645080eb/sdk/program/src/sysvar/mod.rs#L73) which doesn't require these checks.

If the program always modifies the account in question, the address/owner check isn't required because modifying an unowned account will be rejected by the runtime, and the containing transaction will be thrown out.

## Rent

### Calculation of rent

Note: The rent rate can change in the future.

As of writing, the fixed rent fee is 19.055441478439427 lamports per byte-epoch on the testnet and mainnet-beta clusters. An [epoch](https://docs.solana.com/terminology#epoch) is targeted to be 2 days (For devnet, the rent fee is 0.3608183131797095 lamports per byte-epoch with its 54m36s-long epoch).

This value is calculated to target 0.01 SOL per mebibyte-day (exactly matching to 3.56 SOL per mebibyte-year):

Rent fee: 19.055441478439427 = 10_000_000 (0.01 SOL) * 365(approx. day in a year) / (1024 * 1024)(1 MiB) / (365.25/2)(epochs in 1 year)

And rent calculation is done with the `f64` precision and the final result is truncated to `u64` in lamports.

The rent calculation includes account metadata (address, owner, lamports, etc) in the size of an account. Therefore the smallest an account can be for rent calculations is 128 bytes.

For example, an account is created with the initial transfer of 10,000 lamports and no additional data. Rent is immediately debited from it on creation, resulting in a balance of 7,561 lamports:

Rent: 2,439 = 19.055441478439427 (rent rate) * 128 bytes (minimum account size) * 1 (epoch)

Account Balance: 7,561 = 10,000 (transfered lamports) - 2,439 (this account's rent fee for an epoch)

The account balance will be reduced to 5,122 lamports at the next epoch even if there is no activity:

Account Balance: 5,122 = 7,561 (current balance) - 2,439 (this account's rent fee for an epoch)

### Rent exemption

Alternatively, an account can be made entirely exempt from rent collection by depositing at least 2 years worth of rent. This is checked every time an account's balance is reduced, and rent is immediately debited once the balance goes below the minimum amount.

Program executable accounts are required by the runtime to be rent-exempt to avoid being purged.

Note: Use the [`getMinimumBalanceForRentExemption` RPC endpoint](https://docs.solana.com/developing/clients/jsonrpc-api#getminimumbalanceforrentexemption) to calculate the minimum balance for a particular account size. The following calculation is illustrative only.

# Runtime

### Policy

After a program has processed an instruction the runtime verifies that the program only performed operations it was permitted to, and that the results adhere to the runtime policy.

The policy is as follows:

- Only the owner of the account may change owner.
  - And only if the account is writable.
  - And only if the account is not executable
  - And only if the data is zero-initialized or empty.
- An account not assigned to the program cannot have its balance decrease.
- The balance of read-only and executable accounts may not change.
- Only the system program can change the size of the data and only if the system program owns the account.
- Only the owner may change account data.
  - And if the account is writable.
  - And if the account is not executable.
- Executable is one-way (false->true) and only the account owner may set it.
- No one modification to the rent_epoch associated with this account.

# Calling Between Programs

For example, a client could create a transaction that modifies two accounts, each owned by separate on-chain programs:

```
let message = Message::new(vec![

    token_instruction::pay(&alice_pubkey),

    acme_instruction::launch_missiles(&bob_pubkey),

]);

client.send_and_confirm_message(&[&alice_keypair, &bob_keypair], &message);
```

A client may instead allow the `acme` program to conveniently invoke `token` instructions on the client's behalf:

```
let message = Message::new(vec![

    acme_instruction::pay_and_launch_missiles(&alice_pubkey, &bob_pubkey),

]);

client.send_and_confirm_message(&[&alice_keypair, &bob_keypair], &message);
```

Given two on-chain programs `token` and `acme`, each implementing instructions `pay()` and `launch_missiles()` respectively, acme can be implemented with a call to a function defined in the `token` module by issuing a cross-program invocation:

```
mod acme {

    use token_instruction;



    fn launch_missiles(accounts: &[AccountInfo]) -> Result<()> {

        ...

    }



    fn pay_and_launch_missiles(accounts: &[AccountInfo]) -> Result<()> {

        let alice_pubkey = accounts[1].key;

        let instruction = token_instruction::pay(&alice_pubkey);

        invoke(&instruction, accounts)?;



        launch_missiles(accounts)?;

    }
```

`invoke()` is built into Solana's runtime and is responsible for routing the given instruction to the `token` program via the instruction's `program_id` field.

Note that `invoke` requires the caller to pass all the accounts required by the instruction being invoked. This means that both the executable account (the ones that matches the instruction's program id) and the accounts passed to the instruction processor.

### Program signed accounts

Programs can issue instructions that contain signed accounts that were not signed in the original transaction by using [Program derived addresses](https://docs.solana.com/developing/programming-model/calling-between-programs#program-derived-addresses).

To sign an account with program derived addresses, a program may `invoke_signed()`.

```
        invoke_signed(

            &instruction,

            accounts,

            &[&["First addresses seed"],

              &["Second addresses first seed", "Second addresses second seed"]],

        )?;
```

### Call Depth

Cross-program invocations allow programs to invoke other programs directly but the depth is constrained currently to 4.

### Reentrancy

Reentrancy is currently limited to direct self recursion capped at a fixed depth. This restriction prevents situations where a program might invoke another from an intermediary state without the knowledge that it might later be called back into. Direct recursion gives the program full control of its state at the point that it gets called back.

## Program Derived Addresses

1. Allow programs to control specific addresses, called program addresses, in such a way that no external user can generate valid transactions with signatures for those addresses.
2. Allow programs to programmatically sign for program addresses that are present in instructions invoked via [Cross-Program Invocations](https://docs.solana.com/developing/programming-model/calling-between-programs#cross-program-invocations).

Given the two conditions, users can securely transfer or assign the authority of on-chain assets to program addresses and the program can then assign that authority elsewhere at its discretion.

### Private keys for program addresses

A Program address does not lie on the ed25519 curve and therefore has no valid private key associated with it, and thus generating a signature for it is impossible. While it has no private key of its own, it can be used by a program to issue an instruction that includes the Program address as a signer.

### Hash-based generated program addresses

Program addresses are deterministically derived from a collection of seeds and a program id using a 256-bit pre-image resistant hash function. Program address must not lie on the ed25519 curve to ensure there is no associated private key. During generation an error will be returned if the address is found to lie on the curve. There is about a 50/50 chance of this happening for a given collection of seeds and program id. If this occurs a different set of seeds or a seed bump (additional 8 bit seed) can be used to find a valid program address off the curve.

Programs can deterministically derive any number of addresses by using seeds. These seeds can symbolically identify how the addresses are used.

### Using program addresses

Clients can use the `create_program_address` function to generate a destination address. In this example, we assume that `create_program_address(&[&["escrow"]], &escrow_program_id)` generates a valid program address that is off the curve.

```rust
// deterministically derive the escrow key

let escrow_pubkey = create_program_address(&[&["escrow"]], &escrow_program_id);



// construct a transfer message using that key

let message = Message::new(vec![

    token_instruction::transfer(&alice_pubkey, &escrow_pubkey, 1),

]);



// process the message which transfer one 1 token to the escrow

client.send_and_confirm_message(&[&alice_keypair], &message);
```

Programs can use the same function to generate the same address. In the function below the program issues a `token_instruction::transfer` from a program address as if it had the private key to sign the transaction.

```rust
fn transfer_one_token_from_escrow(

    program_id: &Pubkey,

    accounts: &[AccountInfo],

) -> ProgramResult {

    // User supplies the destination

    let alice_pubkey = keyed_accounts[1].unsigned_key();



    // Deterministically derive the escrow pubkey.

    let escrow_pubkey = create_program_address(&[&["escrow"]], program_id);



    // Create the transfer instruction

    let instruction = token_instruction::transfer(&escrow_pubkey, &alice_pubkey, 1);



    // The runtime deterministically derives the key from the currently

    // executing program ID and the supplied keywords.

    // If the derived address matches a key marked as signed in the instruction

    // then that key is accepted as signed.

    invoke_signed(&instruction, accounts, &[&["escrow"]])

}
```

Note that the address generated using `create_program_address` is not guaranteed to be a valid program address off the curve. For example, let's assume that the seed `"escrow2"` does not generate a valid program address.

To generate a valid program address using `"escrow2` as a seed, use `find_program_address`, iterating through possible bump seeds until a valid combination is found. The preceding example becomes:

```rust
// find the escrow key and valid bump seed

let (escrow_pubkey2, escrow_bump_seed) = find_program_address(&[&["escrow2"]], &escrow_program_id);



// construct a transfer message using that key

let message = Message::new(vec![

    token_instruction::transfer(&alice_pubkey, &escrow_pubkey2, 1),

]);



// process the message which transfer one 1 token to the escrow

client.send_and_confirm_message(&[&alice_keypair], &message);
```

Within the program, this becomes:

```rust
fn transfer_one_token_from_escrow2(

    program_id: &Pubkey,

    accounts: &[AccountInfo],

) -> ProgramResult {

    // User supplies the destination

    let alice_pubkey = keyed_accounts[1].unsigned_key();



    // Iteratively derive the escrow pubkey

    let (escrow_pubkey2, bump_seed) = find_program_address(&[&["escrow2"]], program_id);



    // Create the transfer instruction

    let instruction = token_instruction::transfer(&escrow_pubkey2, &alice_pubkey, 1);



    // Include the generated bump seed to the list of all seeds

    invoke_signed(&instruction, accounts, &[&["escrow2", &[bump_seed]]])

}
```

Since `find_program_address` requires iterating over a number of calls to `create_program_address`, it may use more [compute budget](https://docs.solana.com/developing/programming-model/runtime#compute-budget) when used on-chain. To reduce the compute cost, use `find_program_address` off-chain and pass the resulting bump seed to the program.

# Native Programs

## System Program

Create new accounts, allocate account data, assign accounts to owning programs, transfer lamports from System Program owned accounts and pay transacation fees.

- Program id: `11111111111111111111111111111111`
- Instructions: [SystemInstruction](https://docs.rs/solana-sdk/1.8.1/solana_sdk/system_instruction/enum.SystemInstruction.html)

## BPF Loader

Deploys, upgrades, and executes programs on the chain.

- Program id: `BPFLoaderUpgradeab1e11111111111111111111111`
- Instructions: [LoaderInstruction](https://docs.rs/solana-sdk/1.8.1/solana_sdk/loader_upgradeable_instruction/enum.UpgradeableLoaderInstruction.html)

The BPF Upgradeable Loader marks itself as "owner" of the executable and program-data accounts it creates to store your program. When a user invokes an instruction via a program id, the Solana runtime will load both your the program and its owner, the BPF Upgradeable Loader. The runtime then passes your program to the BPF Upgradeable Loader to process the instruction.

## Ed25519 Program

Verify ed25519 signature program. This program takes an ed25519 signature, public key, and message. Multiple signatures can be verified. If any of the signatures fail to verify, an error is returned.

- Program id: `Ed25519SigVerify111111111111111111111111111`
- Instructions: [new_ed25519_instruction](https://github.com/solana-labs/solana/blob/master/sdk/src/ed25519_instruction.rs#L45)

## Secp256k1 Program

Verify secp256k1 public key recovery operations (ecrecover).

- Program id: `KeccakSecp256k11111111111111111111111111111`
- Instructions: [new_secp256k1_instruction](https://github.com/solana-labs/solana/blob/1a658c7f31e1e0d2d39d9efbc0e929350e2c2bcb/sdk/src/secp256k1_instruction.rs#L31)

# Sysvar Cluster Data

Solana exposes a variety of cluster state data to programs via [`sysvar`](https://docs.solana.com/terminology#sysvar) accounts. These accounts are populated at known addresses published along with the account layouts in the [`solana-program` crate](https://docs.rs/solana-program/1.8.1/solana_program/sysvar/index.html), and outlined below.

There are two ways for a program to access a sysvar.

The first is to query the sysvar at runtime via the sysvar's `get()` function:

```
let clock = Clock::get()
```

The following sysvars support `get`:

- Clock - Address: `SysvarC1ock11111111111111111111111111111111`
- EpochSchedule - Address: SysvarEpochSchedu1e111111111111111111111111
- Fees - Address: SysvarFees111111111111111111111111111111111
- Rent - Address: SysvarRent111111111111111111111111111111111

The second is to pass the sysvar to the program as an account by including its address as one of the accounts in the `Instruction` and then deserializing the data during execution. Access to sysvars accounts is always *readonly*.

```
let clock_sysvar_info = next_account_info(account_info_iter)?;

let clock = Clock::from_account_info(&clock_sysvar_info)?;
```

The first method is more efficient and does not require that the sysvar account be passed to the program, or specified in the `Instruction` the program is processing.

# On-chain Programs

BPF uses stack frames instead of a variable stack pointer. Each stack frame is 4KB in size.

Programs are constrained to run quickly, and to facilitate this, the program's call stack is limited to a max depth of 64 frames.

## Data Types

The loader's entrypoint macros call the program defined instruction processor function with the following parameters:

```
program_id: &Pubkey,

accounts: &[AccountInfo],

instruction_data: &[u8]
```

The program id is the public key of the currently executing program.

The accounts is an ordered slice of the accounts referenced by the instruction and represented as an [AccountInfo](https://github.com/solana-labs/solana/blob/7ddf10e602d2ed87a9e3737aa8c32f1db9f909d8/sdk/program/src/account_info.rs#L10) structures. An account's place in the array signifies its meaning, for example, when transferring lamports an instruction may define the first account as the source and the second as the destination.

The members of the `AccountInfo` structure are read-only except for `lamports` and `data`. Both may be modified by the program in accordance with the [runtime enforcement policy](https://docs.solana.com/developing/programming-model/accounts#policy). Both of these members are protected by the Rust `RefCell` construct, so they must be borrowed to read or write to them. The reason for this is they both point back to the original input byte array, but there may be multiple entries in the accounts slice that point to the same account. Using `RefCell` ensures that the program does not accidentally perform overlapping read/writes to the same underlying data via multiple `AccountInfo` structures. If a program implements their own deserialization function care should be taken to handle duplicate accounts appropriately.

## Heap

Rust programs implement the heap directly by defining a custom [`global_allocator`](https://github.com/solana-labs/solana/blob/8330123861a719cd7a79af0544617896e7f00ce3/sdk/program/src/entrypoint.rs#L50)

Programs may implement their own `global_allocator` based on its specific needs. Refer to the [custom heap example](https://docs.solana.com/developing/on-chain-programs/developing-rust#examples) for more information.

## Restrictions

On-chain Rust programs support most of Rust's libstd, libcore, and liballoc, as well as many 3rd party crates.

There are some limitations since these programs run in a resource-constrained, single-threaded environment, and must be deterministic:

- No access to
  - `rand`
  - `std::fs`
  - `std::net`
  - `std::os`
  - `std::future`
  - `std::process`
  - `std::sync`
  - `std::task`
  - `std::thread`
  - `std::time`
- Limited access to:
  - `std::hash`
  - `std::os`
- Bincode is extremely computationally expensive in both cycles and call depth and should be avoided
- String formatting should be avoided since it is also computationally expensive.
- No support for `println!`, `print!`, the Solana [logging helpers](https://docs.solana.com/developing/on-chain-programs/developing-rust#logging) should be used instead.
- The runtime enforces a limit on the number of instructions a program can execute during the processing of one instruction. See [computation budget](https://docs.solana.com/developing/programming-model/runtime#compute-budget) for more information.

## Depending on Rand

Programs are constrained to run deterministically, so random numbers are not available. Sometimes a program may depend on a crate that depends itself on `rand` even if the program does not use any of the random number functionality. If a program depends on `rand`, the compilation will fail because there is no `get-random` support for Solana. 

To work around this dependency issue, add the following dependency to the program's `Cargo.toml`:

```
getrandom = { version = "0.1.14", features = ["dummy"] }
```

or if the dependency is on getrandom v0.2 add:

```
getrandom = { version = "0.2.2", features = ["custom"] }
```

## Panicking

Rust's `panic!`, `assert!`, and internal panic results are printed to the [program logs](https://docs.solana.com/developing/on-chain-programs/debugging#logging) by default.

## Compute Budget

Use the system call [`sol_log_compute_units()`](https://github.com/solana-labs/solana/blob/d3a3a7548c857f26ec2cb10e270da72d373020ec/sdk/program/src/log.rs#L102) to log a message containing the remaining number of compute units the program may consume before execution is halted

## Logging

When running a local cluster the logs are written to stdout as long as they are enabled via the `RUST_LOG` log mask. From the perspective of program development it is helpful to focus on just the runtime and program logs and not the rest of the cluster logs. To focus in on program specific information the following log mask is recommended:

```
export RUST_LOG=solana_runtime::system_instruction_processor=trace,solana_runtime::message_processor=info,solana_bpf_loader=debug,solana_rbpf=debug
```

Log messages coming directly from the program (not the runtime) will be displayed in the form:

```
Program log: <user defined message>
```

## Instruction Tracing

During execution the runtime BPF interpreter can be configured to log a trace message for each BPF instruction executed. This can be very helpful for things like pin-pointing the runtime context leading up to a memory access violation.

The trace logs together with the [ELF dump](https://docs.solana.com/developing/on-chain-programs/debugging#elf-dump) can provide a lot of insight (though the traces produce a lot of information).

To turn on BPF interpreter trace messages in a local cluster configure the `solana_rbpf` level in `RUST_LOG` to `trace`. For example:

```
export RUST_LOG=solana_rbpf=trace
```