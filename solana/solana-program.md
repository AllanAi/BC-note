# Hello World

## Program

```rust
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
    /// number of greetings
    pub counter: u32,
}

// Declare and export the program's entrypoint
entrypoint!(process_instruction);

// Program entrypoint's implementation
pub fn process_instruction(
    program_id: &Pubkey, // Public key of the account the hello world program was loaded into
    accounts: &[AccountInfo], // The account to say hello to
    _instruction_data: &[u8], // Ignored, all helloworld instructions are hellos
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");

    // Iterating accounts is safer then indexing
    let accounts_iter = &mut accounts.iter();

    // Get the account to say hello to
    let account = next_account_info(accounts_iter)?;

    // The account must be owned by the program in order to modify its data
    if account.owner != program_id {
        msg!("Greeted account does not have the correct program id");
        return Err(ProgramError::IncorrectProgramId);
    }

    // Increment and store the number of times the account has been greeted
    let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
    greeting_account.counter += 1;
    greeting_account.serialize(&mut &mut account.data.borrow_mut()[..])?;

    msg!("Greeted {} time(s)!", greeting_account.counter);

    Ok(())
}
```

## Client

```tsx
export async function checkProgram(): Promise<void> {
  // Read program id from keypair file
  try {
    const programKeypair = await createKeypairFromFile(PROGRAM_KEYPAIR_PATH);
    programId = programKeypair.publicKey;
  } catch (err) {
    ...
  }
...
  // Derive the address (public key) of a greeting account from the program so that it's easy to find later.
  const GREETING_SEED = 'hello';
  greetedPubkey = await PublicKey.createWithSeed(
    payer.publicKey,
    GREETING_SEED,
    programId,
  );

  // Check if the greeting account has already been created
  const greetedAccount = await connection.getAccountInfo(greetedPubkey);
  if (greetedAccount === null) {
    console.log(
      'Creating account',
      greetedPubkey.toBase58(),
      'to say hello to',
    );
    const lamports = await connection.getMinimumBalanceForRentExemption(
      GREETING_SIZE,
    );

    const transaction = new Transaction().add(
      SystemProgram.createAccountWithSeed({
        fromPubkey: payer.publicKey,
        basePubkey: payer.publicKey,
        seed: GREETING_SEED,
        newAccountPubkey: greetedPubkey,
        lamports,
        space: GREETING_SIZE,
        programId,
      }),
    );
    await sendAndConfirmTransaction(connection, transaction, [payer]);
  }
}

/**
 * Say hello
 */
export async function sayHello(): Promise<void> {
  console.log('Saying hello to', greetedPubkey.toBase58());
  const instruction = new TransactionInstruction({
    keys: [{pubkey: greetedPubkey, isSigner: false, isWritable: true}],
    programId,
    data: Buffer.alloc(0), // All instructions are hellos
  });
  await sendAndConfirmTransaction(
    connection,
    new Transaction().add(instruction),
    [payer],
  );
}

/**
 * Report the number of times the greeted account has been said hello to
 */
export async function reportGreetings(): Promise<void> {
  const accountInfo = await connection.getAccountInfo(greetedPubkey);
  if (accountInfo === null) {
    throw 'Error: cannot find the greeted account';
  }
  const greeting = borsh.deserialize(
    GreetingSchema,
    GreetingAccount,
    accountInfo.data,
  );
  console.log(
    greetedPubkey.toBase58(),
    'has been greeted',
    greeting.counter,
    'time(s)',
  );
}
```

# Token

## Program

### state.rs

#### Mint data

```rust
/// Mint data. // token 数据
#[repr(C)]
#[derive(Clone, Copy, Debug, Default, PartialEq)]
pub struct Mint {
    /// Optional authority used to mint new tokens. The mint authority may only be provided during
    /// mint creation. If no mint authority is present then the mint has a fixed supply and no
    /// further tokens may be minted.
    pub mint_authority: COption<Pubkey>, // // 外部用户
    /// Total supply of tokens.
    pub supply: u64,
    /// Number of base 10 digits to the right of the decimal place.
    pub decimals: u8,
    /// Is `true` if this structure has been initialized
    pub is_initialized: bool,
    /// Optional authority to freeze token accounts.
    pub freeze_authority: COption<Pubkey>, // // 外部用户
}
```

#### Account data

```rust
/// Account data.
#[repr(C)]
#[derive(Clone, Copy, Debug, Default, PartialEq)]
pub struct Account {
    /// The mint associated with this account // 上面 mint 的地址
    pub mint: Pubkey,
    /// The owner of this account. // 外部用户
    pub owner: Pubkey,
    /// The amount of tokens this account holds.
    pub amount: u64,
    /// If `delegate` is `Some` then `delegated_amount` represents
    /// the amount authorized by the delegate
    pub delegate: COption<Pubkey>,
    /// The account's state
    pub state: AccountState,
    /// If is_some, this is a native token, and the value logs the rent-exempt reserve. An Account
    /// is required to be rent-exempt, so the value is used by the Processor to ensure that wrapped
    /// SOL accounts do not drop below this threshold.
    pub is_native: COption<u64>,
    /// The amount delegated
    pub delegated_amount: u64,
    /// Optional authority to close the account.
    pub close_authority: COption<Pubkey>,
}
/// Account state.
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, TryFromPrimitive)]
pub enum AccountState {
    /// Account is not yet initialized
    Uninitialized,
    /// Account is initialized; the account owner and/or delegate may perform permitted operations
    /// on this account
    Initialized,
    /// Account has been frozen by the mint freeze authority. Neither the account owner nor
    /// the delegate are able to perform operations on this account.
    Frozen,
}
```

### instruction.rs

```rust
    /// Initializes a new mint and optionally deposits all the newly minted
    /// tokens in an account.
    ///
    /// The `InitializeMint` instruction requires no signers and MUST be
    /// included within the same Transaction as the system program's
    /// `CreateAccount` instruction that creates the account being initialized.
    /// Otherwise another party can acquire ownership of the uninitialized
    /// account.
    ///
    /// Accounts expected by this instruction:
    ///
    ///   0. `[writable]` The mint to initialize.
    ///   1. `[]` Rent sysvar
    ///
    InitializeMint {
        /// Number of base 10 digits to the right of the decimal place.
        decimals: u8,
        /// The authority/multisignature to mint tokens.
        mint_authority: Pubkey,
        /// The freeze authority/multisignature of the mint.
        freeze_authority: COption<Pubkey>,
    },
    /// Initializes a new account to hold tokens.  If this account is associated
    /// with the native mint then the token balance of the initialized account
    /// will be equal to the amount of SOL in the account. If this account is
    /// associated with another mint, that mint must be initialized before this
    /// command can succeed.
    ///
    /// The `InitializeAccount` instruction requires no signers and MUST be
    /// included within the same Transaction as the system program's
    /// `CreateAccount` instruction that creates the account being initialized.
    /// Otherwise another party can acquire ownership of the uninitialized
    /// account.
    ///
    /// Accounts expected by this instruction:
    ///
    ///   0. `[writable]`  The account to initialize.
    ///   1. `[]` The mint this account will be associated with.
    ///   2. `[]` The new account's owner/multisignature.
    ///   3. `[]` Rent sysvar
    InitializeAccount,
    /// Transfers tokens from one account to another either directly or via a
    /// delegate.  If this account is associated with the native mint then equal
    /// amounts of SOL and Tokens will be transferred to the destination
    /// account.
    ///
    /// Accounts expected by this instruction:
    ///
    ///   * Single owner/delegate
    ///   0. `[writable]` The source account.
    ///   1. `[writable]` The destination account.
    ///   2. `[signer]` The source account's owner/delegate.
    ///
    ///   * Multisignature owner/delegate
    ///   0. `[writable]` The source account.
    ///   1. `[writable]` The destination account.
    ///   2. `[]` The source account's multisignature owner/delegate.
    ///   3. ..3+M `[signer]` M signer accounts.
    Transfer {
        /// The amount of tokens to transfer.
        amount: u64,
    },
    /// Approves a delegate.  A delegate is given the authority over tokens on
    /// behalf of the source account's owner.
    ///
    /// Accounts expected by this instruction:
    ///
    ///   * Single owner
    ///   0. `[writable]` The source account.
    ///   1. `[]` The delegate.
    ///   2. `[signer]` The source account owner.
    ///
    ///   * Multisignature owner
    ///   0. `[writable]` The source account.
    ///   1. `[]` The delegate.
    ///   2. `[]` The source account's multisignature owner.
    ///   3. ..3+M `[signer]` M signer accounts
    Approve {
        /// The amount of tokens the delegate is approved for.
        amount: u64,
    },
    /// Mints new tokens to an account.  The native mint does not support
    /// minting.
    ///
    /// Accounts expected by this instruction:
    ///
    ///   * Single authority
    ///   0. `[writable]` The mint.
    ///   1. `[writable]` The account to mint tokens to.
    ///   2. `[signer]` The mint's minting authority.
    ///
    ///   * Multisignature authority
    ///   0. `[writable]` The mint.
    ///   1. `[writable]` The account to mint tokens to.
    ///   2. `[]` The mint's multisignature mint-tokens authority.
    ///   3. ..3+M `[signer]` M signer accounts.
    MintTo {
        /// The amount of new tokens to mint.
        amount: u64,
    },
    /// Burns tokens by removing them from an account.  `Burn` does not support
    /// accounts associated with the native mint, use `CloseAccount` instead.
    ///
    /// Accounts expected by this instruction:
    ///
    ///   * Single owner/delegate
    ///   0. `[writable]` The account to burn from.
    ///   1. `[writable]` The token mint.
    ///   2. `[signer]` The account's owner/delegate.
    ///
    ///   * Multisignature owner/delegate
    ///   0. `[writable]` The account to burn from.
    ///   1. `[writable]` The token mint.
    ///   2. `[]` The account's multisignature owner/delegate.
    ///   3. ..3+M `[signer]` M signer accounts.
    Burn {
        /// The amount of tokens to burn.
        amount: u64,
    },
    /// Given a wrapped / native token account (a token account containing SOL)
    /// updates its amount field based on the account's underlying `lamports`.
    /// This is useful if a non-wrapped SOL account uses `system_instruction::transfer`
    /// to move lamports to a wrapped token account, and needs to have its token
    /// `amount` field updated.
    ///
    /// Accounts expected by this instruction:
    ///
    ///   0. `[writable]`  The native token account to sync with its underlying lamports.
    SyncNative,

```

## Client

### createMint

创建一个存储 token 数据的账户

```rust
  /**
   * Create and initialize a token.
   *
   * @param connection The connection to use
   * @param payer Fee payer for transaction
   * @param mintAuthority Account or multisig that will control minting
   * @param freezeAuthority Optional account or multisig that can freeze token accounts
   * @param decimals Location of the decimal place
   * @param programId Optional token programId, uses the system programId by default
   * @return Token object for the newly minted token
   */
  static async createMint(
    connection: Connection,
    payer: Signer,
    mintAuthority: PublicKey,
    freezeAuthority: PublicKey | null,
    decimals: number,
    programId: PublicKey,
  ): Promise<Token> {
    const mintAccount = Keypair.generate();
    const token = new Token(
      connection,
      mintAccount.publicKey,
      programId,
      payer,
    );

    // Allocate memory for the account
    const balanceNeeded = await Token.getMinBalanceRentForExemptMint(
      connection,
    );

    const transaction = new Transaction();
    transaction.add(
      SystemProgram.createAccount({
        fromPubkey: payer.publicKey,
        newAccountPubkey: mintAccount.publicKey,
        lamports: balanceNeeded,
        space: MintLayout.span,
        programId,
      }),
    );

    transaction.add(
      Token.createInitMintInstruction(
        programId,
        mintAccount.publicKey,
        decimals,
        mintAuthority,
        freezeAuthority,
      ),
    );

    // Send the two instructions
    await sendAndConfirmTransaction(
      'createAccount and InitializeMint',
      connection,
      transaction,
      payer,
      mintAccount,
    );

    return token;
  }
```

### createAccount

创建一个存储某个token对应某一个用户数据的账户

```rust
  /**
   * Create and initialize a new account.
   *
   * This account may then be used as a `transfer()` or `approve()` destination
   *
   * @param owner User account that will own the new account
   * @return Public key of the new empty account
   */
  async createAccount(owner: PublicKey): Promise<PublicKey> {
    // Allocate memory for the account
    const balanceNeeded = await Token.getMinBalanceRentForExemptAccount(
      this.connection,
    );

    const newAccount = Keypair.generate();
    const transaction = new Transaction();
    transaction.add(
      SystemProgram.createAccount({
        fromPubkey: this.payer.publicKey,
        newAccountPubkey: newAccount.publicKey,
        lamports: balanceNeeded,
        space: AccountLayout.span,
        programId: this.programId,
      }),
    );

    const mintPublicKey = this.publicKey;
    transaction.add(
      Token.createInitAccountInstruction(
        this.programId,
        mintPublicKey,
        newAccount.publicKey,
        owner,
      ),
    );

    // Send the two instructions
    await sendAndConfirmTransaction(
      'createAccount and InitializeAccount',
      this.connection,
      transaction,
      this.payer,
      newAccount,
    );

    return newAccount.publicKey;
  }

  /**
   * Create and initialize the associated account.
   *
   * This account may then be used as a `transfer()` or `approve()` destination
   *
   * @param owner User account that will own the new account
   * @return Public key of the new associated account
   */
  async createAssociatedTokenAccount(owner: PublicKey): Promise<PublicKey> {
    const associatedAddress = await Token.getAssociatedTokenAddress(
      this.associatedProgramId,
      this.programId,
      this.publicKey,
      owner,
    );

    return this.createAssociatedTokenAccountInternal(owner, associatedAddress);
  }

  async createAssociatedTokenAccountInternal(
    owner: PublicKey,
    associatedAddress: PublicKey,
  ): Promise<PublicKey> {
    await sendAndConfirmTransaction(
      'CreateAssociatedTokenAccount',
      this.connection,
      new Transaction().add(
        Token.createAssociatedTokenAccountInstruction(
          this.associatedProgramId,
          this.programId,
          this.publicKey,
          associatedAddress,
          owner,
          this.payer.publicKey,
        ),
      ),
      this.payer,
    );

    return associatedAddress;
  }

  /**
   * Retrieve the associated account or create one if not found.
   *
   * This account may then be used as a `transfer()` or `approve()` destination
   *
   * @param owner User account that will own the new account
   * @return The new associated account
   */
  async getOrCreateAssociatedAccountInfo(
    owner: PublicKey,
  ): Promise<AccountInfo> {
    const associatedAddress = await Token.getAssociatedTokenAddress(
      this.associatedProgramId,
      this.programId,
      this.publicKey,
      owner,
    );

    // This is the optimum logic, considering TX fee, client-side computation,
    // RPC roundtrips and guaranteed idempotent.
    // Sadly we can't do this atomically;
    try {
      return await this.getAccountInfo(associatedAddress);
    } catch (err) {
      // INVALID_ACCOUNT_OWNER can be possible if the associatedAddress has
      // already been received some lamports (= became system accounts).
      // Assuming program derived addressing is safe, this is the only case
      // for the INVALID_ACCOUNT_OWNER in this code-path
      if (
        err.message === FAILED_TO_FIND_ACCOUNT ||
        err.message === INVALID_ACCOUNT_OWNER
      ) {
        // as this isn't atomic, it's possible others can create associated
        // accounts meanwhile
        try {
          await this.createAssociatedTokenAccountInternal(
            owner,
            associatedAddress,
          );
        } catch (err) {
          // ignore all errors; for now there is no API compatible way to
          // selectively ignore the expected instruction error if the
          // associated account is existing already.
        }

        // Now this should always succeed
        return await this.getAccountInfo(associatedAddress);
      } else {
        throw err;
      }
    }
  }

  /**
   * Create and initialize a new account on the special native token mint.
   *
   * In order to be wrapped, the account must have a balance of native tokens
   * when it is initialized with the token program.
   *
   * This function sends lamports to the new account before initializing it.
   *
   * @param connection A solana web3 connection
   * @param programId The token program ID
   * @param owner The owner of the new token account
   * @param payer The source of the lamports to initialize, and payer of the initialization fees.
   * @param amount The amount of lamports to wrap
   * @return {Promise<PublicKey>} The new token account
   */
  static async createWrappedNativeAccount(
    connection: Connection,
    programId: PublicKey,
    owner: PublicKey,
    payer: Signer,
    amount: number,
  ): Promise<PublicKey> {
    // Allocate memory for the account
    const balanceNeeded = await Token.getMinBalanceRentForExemptAccount(
      connection,
    );

    // Create a new account
    const newAccount = Keypair.generate();
    const transaction = new Transaction();
    transaction.add(
      SystemProgram.createAccount({
        fromPubkey: payer.publicKey,
        newAccountPubkey: newAccount.publicKey,
        lamports: balanceNeeded,
        space: AccountLayout.span,
        programId,
      }),
    );

    // Send lamports to it (these will be wrapped into native tokens by the token program)
    transaction.add(
      SystemProgram.transfer({
        fromPubkey: payer.publicKey,
        toPubkey: newAccount.publicKey,
        lamports: amount,
      }),
    );

    // Assign the new account to the native token mint.
    // the account will be initialized with a balance equal to the native token balance.
    // (i.e. amount)
    transaction.add(
      Token.createInitAccountInstruction(
        programId,
        NATIVE_MINT,
        newAccount.publicKey,
        owner,
      ),
    );

    // Send the three instructions
    await sendAndConfirmTransaction(
      'createAccount, transfer, and initializeAccount',
      connection,
      transaction,
      payer,
      newAccount,
    );

    return newAccount.publicKey;
  }
  /**
   * Get the address for the associated token account
   *
   * @param associatedProgramId SPL Associated Token program account
   * @param programId SPL Token program account
   * @param mint Token mint account
   * @param owner Owner of the new account
   * @return Public key of the associated token account
   */
  static async getAssociatedTokenAddress(
    associatedProgramId: PublicKey,
    programId: PublicKey,
    mint: PublicKey,
    owner: PublicKey,
    allowOwnerOffCurve: boolean = false,
  ): Promise<PublicKey> {
    if (!allowOwnerOffCurve && !PublicKey.isOnCurve(owner.toBuffer())) {
      throw new Error(`Owner cannot sign: ${owner.toString()}`);
    }
    return (
      await PublicKey.findProgramAddress(
        [owner.toBuffer(), programId.toBuffer(), mint.toBuffer()],
        associatedProgramId,
      )
    )[0];
  }
```

### transfer

```rust
  /**
   * Transfer tokens to another account
   *
   * @param source Source account
   * @param destination Destination account
   * @param owner Owner of the source account
   * @param multiSigners Signing accounts if `owner` is a multiSig
   * @param amount Number of tokens to transfer
   */
  async transfer(
    source: PublicKey,
    destination: PublicKey,
    owner: any,
    multiSigners: Array<Signer>,
    amount: number | u64,
  ): Promise<TransactionSignature> {
    let ownerPublicKey;
    let signers;
    if (isAccount(owner)) {
      ownerPublicKey = owner.publicKey;
      signers = [owner];
    } else {
      ownerPublicKey = owner;
      signers = multiSigners;
    }
    return await sendAndConfirmTransaction(
      'Transfer',
      this.connection,
      new Transaction().add(
        Token.createTransferInstruction(
          this.programId,
          source,
          destination,
          ownerPublicKey,
          multiSigners,
          amount,
        ),
      ),
      this.payer,
      ...signers,
    );
  }
```

### approve

```rust
  /**
   * Grant a third-party permission to transfer up the specified number of tokens from an account
   *
   * @param account Public key of the account
   * @param delegate Account authorized to perform a transfer tokens from the source account
   * @param owner Owner of the source account
   * @param multiSigners Signing accounts if `owner` is a multiSig
   * @param amount Maximum number of tokens the delegate may transfer
   */
  async approve(
    account: PublicKey,
    delegate: PublicKey,
    owner: any,
    multiSigners: Array<Signer>,
    amount: number | u64,
  ): Promise<void> {
    let ownerPublicKey;
    let signers;
    if (isAccount(owner)) {
      ownerPublicKey = owner.publicKey;
      signers = [owner];
    } else {
      ownerPublicKey = owner;
      signers = multiSigners;
    }
    await sendAndConfirmTransaction(
      'Approve',
      this.connection,
      new Transaction().add(
        Token.createApproveInstruction(
          this.programId,
          account,
          delegate,
          ownerPublicKey,
          multiSigners,
          amount,
        ),
      ),
      this.payer,
      ...signers,
    );
  }
```

# TokenSwap

## Program

### state.rs

```rust
/// Program states.
#[repr(C)]
#[derive(Debug, Default, PartialEq)]
pub struct SwapV1 {
    /// Initialized state.
    pub is_initialized: bool,
    /// Bump seed used in program address.
    /// The program address is created deterministically with the bump seed,
    /// swap program id, and swap account pubkey.  This program address has
    /// authority over the swap's token A account, token B account, and pool
    /// token mint.
    pub bump_seed: u8,

    /// Program ID of the tokens being exchanged.
    pub token_program_id: Pubkey,

    /// Token A
    pub token_a: Pubkey,
    /// Token B
    pub token_b: Pubkey,

    /// Pool tokens are issued when A or B tokens are deposited.
    /// Pool tokens can be withdrawn back to the original A or B token.
    pub pool_mint: Pubkey,

    /// Mint information for token A
    pub token_a_mint: Pubkey,
    /// Mint information for token B
    pub token_b_mint: Pubkey,

    /// Pool token account to receive trading and / or withdrawal fees
    pub pool_fee_account: Pubkey,

    /// All fee information
    pub fees: Fees,

    /// Swap curve parameters, to be unpacked and used by the SwapCurve, which
    /// calculates swaps, deposits, and withdrawals
    pub swap_curve: SwapCurve,
}
```

### instructions.rs

```rust
    ///   Initializes a new swap
    ///
    ///   0. `[writable, signer]` New Token-swap to create.
    ///   1. `[]` swap authority derived from `create_program_address(&[Token-swap account])`
    ///   2. `[]` token_a Account. Must be non zero, owned by swap authority.
    ///   3. `[]` token_b Account. Must be non zero, owned by swap authority.
    ///   4. `[writable]` Pool Token Mint. Must be empty, owned by swap authority.
    ///   5. `[]` Pool Token Account to deposit trading and withdraw fees.
    ///   Must be empty, not owned by swap authority
    ///   6. `[writable]` Pool Token Account to deposit the initial pool token
    ///   supply.  Must be empty, not owned by swap authority.
    ///   7. '[]` Token program id
    Initialize(Initialize),

    ///   Swap the tokens in the pool.
    ///
    ///   0. `[]` Token-swap
    ///   1. `[]` swap authority
    ///   2. `[]` user transfer authority
    ///   3. `[writable]` token_(A|B) SOURCE Account, amount is transferable by user transfer authority,
    ///   4. `[writable]` token_(A|B) Base Account to swap INTO.  Must be the SOURCE token.
    ///   5. `[writable]` token_(A|B) Base Account to swap FROM.  Must be the DESTINATION token.
    ///   6. `[writable]` token_(A|B) DESTINATION Account assigned to USER as the owner.
    ///   7. `[writable]` Pool token mint, to generate trading fees
    ///   8. `[writable]` Fee account, to receive trading fees
    ///   9. '[]` Token program id
    ///   10 `[optional, writable]` Host fee account to receive additional trading fees
    Swap(Swap),

    ///   Deposit both types of tokens into the pool.  The output is a "pool"
    ///   token representing ownership in the pool. Inputs are converted to
    ///   the current ratio.
    ///
    ///   0. `[]` Token-swap
    ///   1. `[]` swap authority
    ///   2. `[]` user transfer authority
    ///   3. `[writable]` token_a user transfer authority can transfer amount,
    ///   4. `[writable]` token_b user transfer authority can transfer amount,
    ///   5. `[writable]` token_a Base Account to deposit into.
    ///   6. `[writable]` token_b Base Account to deposit into.
    ///   7. `[writable]` Pool MINT account, swap authority is the owner.
    ///   8. `[writable]` Pool Account to deposit the generated tokens, user is the owner.
    ///   9. '[]` Token program id
    DepositAllTokenTypes(DepositAllTokenTypes),

    ///   Withdraw both types of tokens from the pool at the current ratio, given
    ///   pool tokens.  The pool tokens are burned in exchange for an equivalent
    ///   amount of token A and B.
    ///
    ///   0. `[]` Token-swap
    ///   1. `[]` swap authority
    ///   2. `[]` user transfer authority
    ///   3. `[writable]` Pool mint account, swap authority is the owner
    ///   4. `[writable]` SOURCE Pool account, amount is transferable by user transfer authority.
    ///   5. `[writable]` token_a Swap Account to withdraw FROM.
    ///   6. `[writable]` token_b Swap Account to withdraw FROM.
    ///   7. `[writable]` token_a user Account to credit.
    ///   8. `[writable]` token_b user Account to credit.
    ///   9. `[writable]` Fee account, to receive withdrawal fees
    ///   10 '[]` Token program id
    WithdrawAllTokenTypes(WithdrawAllTokenTypes),
```

### processor.rs

#### process_initialize

```rust
        let (swap_authority, bump_seed) =
            Pubkey::find_program_address(&[&swap_info.key.to_bytes()], program_id);
        if *authority_info.key != swap_authority {
            return Err(SwapError::InvalidProgramAddress.into());
        }
```

#### process_swap

```rust
        if *authority_info.key
            != Self::authority_id(program_id, swap_info.key, token_swap.bump_seed())?
        {
            return Err(SwapError::InvalidProgramAddress.into());
        }
        

    /// Calculates the authority id by generating a program address.
    pub fn authority_id(
        program_id: &Pubkey,
        my_info: &Pubkey,
        bump_seed: u8,
    ) -> Result<Pubkey, SwapError> {
        Pubkey::create_program_address(&[&my_info.to_bytes()[..32], &[bump_seed]], program_id)
            .or(Err(SwapError::InvalidProgramAddress))
    }
```



```rust
    /// Issue a spl_token `Transfer` instruction.
    pub fn token_transfer<'a>(
        swap: &Pubkey,
        token_program: AccountInfo<'a>,
        source: AccountInfo<'a>,
        destination: AccountInfo<'a>,
        authority: AccountInfo<'a>,
        bump_seed: u8,
        amount: u64,
    ) -> Result<(), ProgramError> {
        let swap_bytes = swap.to_bytes();
        let authority_signature_seeds = [&swap_bytes[..32], &[bump_seed]];
        let signers = &[&authority_signature_seeds[..]];
        let ix = spl_token::instruction::transfer(
            token_program.key,
            source.key,
            destination.key,
            authority.key,
            &[],
            amount,
        )?;
        invoke_signed(
            &ix,
            &[source, destination, authority, token_program],
            signers,
        )
    }
```



#### process_deposit_all_token_types

#### process_withdraw_all_token_types

## Client

### createTokenSwap

```rust
export async function createTokenSwap(
  curveType: number,
  curveParameters?: Numberu64,
): Promise<void> {
  const connection = await getConnection();
  const payer = await newAccountWithLamports(connection, 1000000000); // 创建外部账户并获取空投
  owner = await newAccountWithLamports(connection, 1000000000); // 创建外部账户并获取空投
  const tokenSwapAccount = new Account(); // swap 存储账户

  [authority, bumpSeed] = await PublicKey.findProgramAddress( // pda
    [tokenSwapAccount.publicKey.toBuffer()],
    TOKEN_SWAP_PROGRAM_ID,
  );

  console.log('creating pool mint');
  tokenPool = await Token.createMint( // 创建 pool token；authority 拥有 mint 权限
    connection,
    payer,
    authority,
    null,
    2,
    TOKEN_PROGRAM_ID,
  );

  console.log('creating pool account');
  tokenAccountPool = await tokenPool.createAccount(owner.publicKey); // 接收流动性的账户，由owner持有；因为初始化时会给由pda控制的tokenA、tokenB账户mint钱
  const ownerKey = SWAP_PROGRAM_OWNER_FEE_ADDRESS || owner.publicKey.toString();
  feeAccount = await tokenPool.createAccount(new PublicKey(ownerKey)); // 手续费账户

  console.log('creating token A');
  mintA = await Token.createMint( // 创建 tokenA；owner 拥有 mint 权限
    connection,
    payer,
    owner.publicKey,
    null,
    2,
    TOKEN_PROGRAM_ID,
  );

  console.log('creating token A account');
  tokenAccountA = await mintA.createAccount(authority); // 创建由 authority 持有的 tokenA 账户
  console.log('minting token A to swap');
  await mintA.mintTo(tokenAccountA, owner, [], currentSwapTokenA); // 给该账户打钱，对应的在合约中会给 tokenAccountPool mint 流动性

  console.log('creating token B');
  mintB = await Token.createMint(// 创建 tokenB；owner 拥有 mint 权限
    connection,
    payer,
    owner.publicKey,
    null,
    2,
    TOKEN_PROGRAM_ID,
  );

  console.log('creating token B account');
  tokenAccountB = await mintB.createAccount(authority); // 创建由 authority 持有的 tokenA 账户
  console.log('minting token B to swap');
  await mintB.mintTo(tokenAccountB, owner, [], currentSwapTokenB); // 给该账户打钱，对应的在合约中会给 tokenAccountPool mint 流动性

  console.log('creating token swap');
  const swapPayer = await newAccountWithLamports(connection, 10000000000);
  tokenSwap = await TokenSwap.createTokenSwap( // 创建 tokenSwapAccount 账户，并初始化 swap
    connection,
    swapPayer,
    tokenSwapAccount,
    authority,
    tokenAccountA,
    tokenAccountB,
    tokenPool.publicKey,
    mintA.publicKey,
    mintB.publicKey,
    feeAccount,
    tokenAccountPool,
    TOKEN_SWAP_PROGRAM_ID,
    TOKEN_PROGRAM_ID,
    TRADING_FEE_NUMERATOR,
    TRADING_FEE_DENOMINATOR,
    OWNER_TRADING_FEE_NUMERATOR,
    OWNER_TRADING_FEE_DENOMINATOR,
    OWNER_WITHDRAW_FEE_NUMERATOR,
    OWNER_WITHDRAW_FEE_DENOMINATOR,
    HOST_FEE_NUMERATOR,
    HOST_FEE_DENOMINATOR,
    curveType,
    curveParameters,
  );
}
```

### depositAllTokenTypes

```rust
export async function depositAllTokenTypes(): Promise<void> {
  const poolMintInfo = await tokenPool.getMintInfo();
  const supply = poolMintInfo.supply.toNumber(); // 流动性总供应量
  const swapTokenA = await mintA.getAccountInfo(tokenAccountA);
  const tokenA = Math.floor(
    (swapTokenA.amount.toNumber() * POOL_TOKEN_AMOUNT) / supply,
  );
  const swapTokenB = await mintB.getAccountInfo(tokenAccountB);
  const tokenB = Math.floor(
    (swapTokenB.amount.toNumber() * POOL_TOKEN_AMOUNT) / supply,
  );

  const userTransferAuthority = new Account();
  console.log('Creating depositor token a account');
  const userAccountA = await mintA.createAccount(owner.publicKey);
  await mintA.mintTo(userAccountA, owner, [], tokenA);
  await mintA.approve(
    userAccountA,
    userTransferAuthority.publicKey,
    owner,
    [],
    tokenA,
  );
  console.log('Creating depositor token b account');
  const userAccountB = await mintB.createAccount(owner.publicKey);
  await mintB.mintTo(userAccountB, owner, [], tokenB);
  await mintB.approve(
    userAccountB,
    userTransferAuthority.publicKey,
    owner,
    [],
    tokenB,
  );
  console.log('Creating depositor pool token account');
  const newAccountPool = await tokenPool.createAccount(owner.publicKey);

  console.log('Depositing into swap');
  await tokenSwap.depositAllTokenTypes(
    userAccountA,
    userAccountB,
    newAccountPool,
    userTransferAuthority,
    POOL_TOKEN_AMOUNT,
    tokenA,
    tokenB,
  );
}
```

### createAccountAndSwapAtomic

```rust
export async function createAccountAndSwapAtomic(): Promise<void> {
  console.log('Creating swap token a account');
  let userAccountA = await mintA.createAccount(owner.publicKey);
  await mintA.mintTo(userAccountA, owner, [], SWAP_AMOUNT_IN);

  // @ts-ignore
  const balanceNeeded = await Token.getMinBalanceRentForExemptAccount(
    connection,
  );
  const newAccount = new Account();
  const transaction = new Transaction();
  transaction.add(
    SystemProgram.createAccount({
      fromPubkey: owner.publicKey,
      newAccountPubkey: newAccount.publicKey,
      lamports: balanceNeeded,
      space: AccountLayout.span,
      programId: mintB.programId,
    }),
  );

  transaction.add(
    Token.createInitAccountInstruction(
      mintB.programId,
      mintB.publicKey,
      newAccount.publicKey,
      owner.publicKey,
    ),
  );

  const userTransferAuthority = new Account();
  transaction.add(
    Token.createApproveInstruction(
      mintA.programId,
      userAccountA,
      userTransferAuthority.publicKey,
      owner.publicKey,
      [owner],
      SWAP_AMOUNT_IN,
    ),
  );

  transaction.add(
    TokenSwap.swapInstruction(
      tokenSwap.tokenSwap,
      tokenSwap.authority,
      userTransferAuthority.publicKey,
      userAccountA,
      tokenSwap.tokenAccountA,
      tokenSwap.tokenAccountB,
      newAccount.publicKey,
      tokenSwap.poolToken,
      tokenSwap.feeAccount,
      null,
      tokenSwap.swapProgramId,
      tokenSwap.tokenProgramId,
      SWAP_AMOUNT_IN,
      0,
    ),
  );

  // Send the instructions
  console.log('sending big instruction');
  await sendAndConfirmTransaction(
    'create account, approve transfer, swap',
    connection,
    transaction,
    owner,
    newAccount,
    userTransferAuthority,
  );

  let info;
  info = await mintA.getAccountInfo(tokenAccountA);
  currentSwapTokenA = info.amount.toNumber();
  info = await mintB.getAccountInfo(tokenAccountB);
  currentSwapTokenB = info.amount.toNumber();
}
```

### swap

```rust
export async function swap(): Promise<void> {
  console.log('Creating swap token a account');
  let userAccountA = await mintA.createAccount(owner.publicKey);
  await mintA.mintTo(userAccountA, owner, [], SWAP_AMOUNT_IN);
  const userTransferAuthority = new Account();
  await mintA.approve(
    userAccountA,
    userTransferAuthority.publicKey,
    owner,
    [],
    SWAP_AMOUNT_IN,
  );
  console.log('Creating swap token b account');
  let userAccountB = await mintB.createAccount(owner.publicKey);
  let poolAccount = SWAP_PROGRAM_OWNER_FEE_ADDRESS
    ? await tokenPool.createAccount(owner.publicKey)
    : null;

  console.log('Swapping');
  await tokenSwap.swap(
    userAccountA,
    tokenAccountA,
    tokenAccountB,
    userAccountB,
    poolAccount,
    userTransferAuthority,
    SWAP_AMOUNT_IN,
    SWAP_AMOUNT_OUT,
  );
}
```

# Anchor

## basic-0

### Program

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
mod basic_0 {
    use super::*;
    // Context<Initialize>: a simple container for the currently executing program_id generic over Accounts--here, the Initialize struct.
    pub fn initialize(_ctx: Context<Initialize>) -> ProgramResult {
        Ok(())
    }
}

#[derive(Accounts)] // The Accounts derive macro marks a struct containing all the accounts that must be specified for a given instruction. 
pub struct Initialize {}
```

- anchor build

- anchor deploy

### Client

```js
const anchor = require('@project-serum/anchor');

// Configure the local cluster.
anchor.setProvider(anchor.Provider.local());

async function main() {
  // #region main
  // Read the generated IDL.
  const idl = JSON.parse(require('fs').readFileSync('./target/idl/basic_0.json', 'utf8'));

  // Address of the deployed program.
  const programId = new anchor.web3.PublicKey('<YOUR-PROGRAM-ID>');

  // Generate the program client from IDL.
  const program = new anchor.Program(idl, programId);

  // Execute the RPC.
  await program.rpc.initialize();
  // #endregion main
}

console.log('Running client.');
main().then(() => console.log('Success'));
```

- solana config get keypair
- ANCHOR_WALLET=<YOUR-KEYPAIR-PATH> node client.js

### Test

```js
const anchor = require("@project-serum/anchor");

describe("basic-0", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.Provider.local());

  it("Uses the workspace to invoke the initialize instruction", async () => {
    // #region code
    // Read the deployed program from the workspace.
    const program = anchor.workspace.Basic0;

    // Execute the RPC.
    await program.rpc.initialize();
    // #endregion code
  });
});
```

- anchor test

## basic-1 Arguments and Accounts

### Program

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
mod basic_1 {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, data: u64) -> ProgramResult {
        let my_account = &mut ctx.accounts.my_account;
        my_account.data = data;
        Ok(())
    }

    pub fn update(ctx: Context<Update>, data: u64) -> ProgramResult {
        let my_account = &mut ctx.accounts.my_account;
        my_account.data = data;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    // init attribute will create a new account owned by the current program, zero initialized. When using init, one must also provide payer, which will fund the account creation, space, which defines how large the account should be, and the system_program, which is required by the runtime for creating the account.
    // All accounts created with Anchor are laid out as follows: 8-byte-discriminator || borsh serialized data. 
    #[account(init, payer = user, space = 8 + 8)]
    // Using this type within an Accounts struct will ensure the account is owned by the address defined by declare_id! where the inner account was defined.
    pub my_account: Account<'info, MyAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Update<'info> {
    #[account(mut)]
    pub my_account: Account<'info, MyAccount>,
}

#[account] // attribute macro implementing AccountSerialize (opens new window)and AccountDeserialize (opens new window), automatically prepending a unique 8 byte discriminator to the account array. 
pub struct MyAccount {
    pub data: u64,
}
```

### Test

```js
const assert = require("assert");
const anchor = require("@project-serum/anchor");
const { SystemProgram } = anchor.web3;

describe("basic-1", () => {
  // Use a local provider.
  const provider = anchor.Provider.local();

  // Configure the client to use the local cluster.
  anchor.setProvider(provider);

  it("Creates and initializes an account in a single atomic transaction (simplified)", async () => {
    // #region code-simplified
    // The program to execute.
    const program = anchor.workspace.Basic1;

    // The Account to create.
    const myAccount = anchor.web3.Keypair.generate();

    // Create the new account and initialize it with the program.
    // #region code-simplified
    await program.rpc.initialize(new anchor.BN(1234), {
      accounts: {
        myAccount: myAccount.publicKey,
        user: provider.wallet.publicKey,
        systemProgram: SystemProgram.programId,
      },
      signers: [myAccount], // Because myAccount is being created, the Solana runtime requries it to sign the transaction. 不需要其他在Program中标记为Signer的账户
    });
    // #endregion code-simplified

    // Fetch the newly created account from the cluster.
    const account = await program.account.myAccount.fetch(myAccount.publicKey);

    // Check it's state was initialized.
    assert.ok(account.data.eq(new anchor.BN(1234)));

    // Store the account for the next test.
    _myAccount = myAccount;
  });

  it("Updates a previously created account", async () => {
    const myAccount = _myAccount;

    // #region update-test

    // The program to execute.
    const program = anchor.workspace.Basic1;

    // Invoke the update rpc.
    await program.rpc.update(new anchor.BN(4321), {
      accounts: {
        myAccount: myAccount.publicKey,
      },
    });

    // Fetch the newly updated account.
    const account = await program.account.myAccount.fetch(myAccount.publicKey);

    // Check it's state was mutated.
    assert.ok(account.data.eq(new anchor.BN(4321)));

    // #endregion update-test
  });
});
```

## basic-2 Access Control

This tutorial covers how to specify constraints and access control on accounts, a problem somewhat unique to the parallel nature of Solana.

### Program

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
mod basic_2 {
    use super::*;

    pub fn create(ctx: Context<Create>, authority: Pubkey) -> ProgramResult {
        let counter = &mut ctx.accounts.counter;
        counter.authority = authority;
        counter.count = 0;
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> ProgramResult {
        let counter = &mut ctx.accounts.counter;
        counter.count += 1;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Create<'info> {
    #[account(init, payer = user, space = 8 + 40)]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    // has_one: enforces the constraint that Increment.counter.authority == Increment.authority.key
    #[account(mut, has_one = authority)]
    pub counter: Account<'info, Counter>,
    // Signer enforces the constraint that the authority account signed the transaction. However, anchor doesn't fetch the data on that account. More: https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html
    pub authority: Signer<'info>,
}

#[account]
pub struct Counter {
    pub authority: Pubkey,
    pub count: u64,
}
```

### Test

```js
const assert = require('assert');
const anchor = require('@project-serum/anchor');
const { SystemProgram } = anchor.web3;

describe('basic-2', () => {
  const provider = anchor.Provider.local()

  // Configure the client to use the local cluster.
  anchor.setProvider(provider)

  // Counter for the tests.
  const counter = anchor.web3.Keypair.generate()

  // Program for the tests.
  const program = anchor.workspace.Basic2

  it('Creates a counter', async () => {
    await program.rpc.create(provider.wallet.publicKey, {
      accounts: {
        counter: counter.publicKey,
        user: provider.wallet.publicKey,
        systemProgram: SystemProgram.programId,
      },
      signers: [counter],
    })

    let counterAccount = await program.account.counter.fetch(counter.publicKey)

    assert.ok(counterAccount.authority.equals(provider.wallet.publicKey))
    assert.ok(counterAccount.count.toNumber() === 0)
  })

  it('Updates a counter', async () => {
    await program.rpc.increment({
      accounts: {
        counter: counter.publicKey,
        authority: provider.wallet.publicKey,
      },
    })

    const counterAccount = await program.account.counter.fetch(counter.publicKey)

    assert.ok(counterAccount.authority.equals(provider.wallet.publicKey))
    assert.ok(counterAccount.count.toNumber() == 1)
  })
})
```

## basic-3 CPI

### Program

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod puppet {
    use super::*;
    pub fn initialize(_ctx: Context<Initialize>) -> ProgramResult {
        Ok(())
    }

    pub fn set_data(ctx: Context<SetData>, data: u64) -> ProgramResult {
        let puppet = &mut ctx.accounts.puppet;
        puppet.data = data;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub puppet: Account<'info, Data>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct SetData<'info> {
    #[account(mut)]
    pub puppet: Account<'info, Data>,
}

#[account]
pub struct Data {
    pub data: u64,
}
```

```rust
// #region core
use anchor_lang::prelude::*;
use puppet::cpi::accounts::SetData;
use puppet::program::Puppet;
use puppet::{self, Data};

declare_id!("HmbTLCmaGvZhKnn1Zfa1JVnp7vkMV4DYVxPLWBVoN65L");

#[program]
mod puppet_master {
    use super::*;
    pub fn pull_strings(ctx: Context<PullStrings>, data: u64) -> ProgramResult {
        let cpi_program = ctx.accounts.puppet_program.to_account_info();
        let cpi_accounts = SetData {
            puppet: ctx.accounts.puppet.to_account_info(),
        };
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        // CpiContext::new_with_signer(cpi_accounts, cpi_program, signer_seeds)
        puppet::cpi::set_data(cpi_ctx, data)
    }
}

#[derive(Accounts)]
pub struct PullStrings<'info> {
    #[account(mut)]
    pub puppet: Account<'info, Data>,
    pub puppet_program: Program<'info, Puppet>,
}
// #endregion core
```

Cargo.toml

```toml
puppet = { path = "../puppet", features = ["cpi"] }
```

### Test

```js
const assert = require("assert");
const anchor = require("@project-serum/anchor");
const { SystemProgram } = anchor.web3;

describe("basic-3", () => {
  const provider = anchor.Provider.local();

  // Configure the client to use the local cluster.
  anchor.setProvider(provider);

  it("Performs CPI from puppet master to puppet", async () => {
    const puppetMaster = anchor.workspace.PuppetMaster;
    const puppet = anchor.workspace.Puppet;

    // Initialize a new puppet account.
    const newPuppetAccount = anchor.web3.Keypair.generate();
    const tx = await puppet.rpc.initialize({
      accounts: {
        puppet: newPuppetAccount.publicKey,
        user: provider.wallet.publicKey,
        systemProgram: SystemProgram.programId,
      },
      signers: [newPuppetAccount],
    });

    // Invoke the puppet master to perform a CPI to the puppet.
    await puppetMaster.rpc.pullStrings(new anchor.BN(111), {
       accounts: {
          puppet: newPuppetAccount.publicKey,
          puppetProgram: puppet.programId,
       },
    });

    // Check the state updated.
    puppetAccount = await puppet.account.data.fetch(newPuppetAccount.publicKey);
    assert.ok(puppetAccount.data.eq(new anchor.BN(111)));
  });
});
```

Solana currently has no way to return values from CPI, alas. However, you can approximate this by having the callee write return values to an account and the caller read that account to retrieve the return value. In future work, Anchor should do this transparently.

## errors

### Program

```rust
use anchor_lang::prelude::*;

#[program]
mod errors {
    use super::*;
    pub fn hello(_ctx: Context<Hello>) -> Result<()> {
        Err(ErrorCode::Hello.into())
    }
}

#[derive(Accounts)]
pub struct Hello {}

#[error]
pub enum ErrorCode {
    #[msg("This is an error message clients will automatically display")]
    Hello,
}
```

### Test

```js
try {
  const tx = await program.rpc.hello();
  assert.ok(false);
} catch (err) {
  const errMsg = "This is an error message clients will automatically display";
  assert.equal(err.toString(), errMsg);
}
```



## basic-4 state

### Program

```rust
// #region code
use anchor_lang::prelude::*;

declare_id!("CwrqeMj2U8tFr1Rhkgwc84tpAsqbt9pTt2a4taoTADPr");

#[program]
pub mod basic_4 {
    use super::*;

    #[state]
    pub struct Counter {
        pub authority: Pubkey,
        pub count: u64,
    }

    impl Counter {
        pub fn new(ctx: Context<Auth>) -> Result<Self> {
            Ok(Self {
                authority: *ctx.accounts.authority.key,
                count: 0,
            })
        }

        pub fn increment(&mut self, ctx: Context<Auth>) -> Result<()> {
            if &self.authority != ctx.accounts.authority.key {
                return Err(ErrorCode::Unauthorized.into());
            }
            self.count += 1;
            Ok(())
        }
    }
}

#[derive(Accounts)]
pub struct Auth<'info> {
    authority: Signer<'info>,
}
// #endregion code

#[error]
pub enum ErrorCode {
    #[msg("You are not authorized to perform this action.")]
    Unauthorized,
}
```

### Test

```js
const assert = require("assert");
const anchor = require("@project-serum/anchor");

describe("basic-4", () => {
  const provider = anchor.Provider.local();

  // Configure the client to use the local cluster.
  anchor.setProvider(provider);

  const program = anchor.workspace.Basic4;

  it("Is runs the constructor", async () => {
    // #region ctor
    // Initialize the program's state struct.
    await program.state.rpc.new({
      accounts: {
        authority: provider.wallet.publicKey,
      },
    });
    // #endregion ctor

    // Fetch the state struct from the network.
    // #region accessor
    const state = await program.state.fetch();
    // #endregion accessor

    assert.ok(state.count.eq(new anchor.BN(0)));
  });

  it("Executes a method on the program", async () => {
    // #region instruction
    await program.state.rpc.increment({
      accounts: {
        authority: provider.wallet.publicKey,
      },
    });
    // #endregion instruction
    const state = await program.state.fetch();
    assert.ok(state.count.eq(new anchor.BN(1)));
  });
});
```



# merkle-distributor

## new_distributor

```rust
    /// Creates a new [MerkleDistributor].
    /// After creating this [MerkleDistributor], the account should be seeded with tokens via its ATA.
    pub fn new_distributor(
        ctx: Context<NewDistributor>,
        bump: u8,
        root: [u8; 32],
        max_total_claim: u64,
        max_num_nodes: u64,
    ) -> ProgramResult {
        let distributor = &mut ctx.accounts.distributor;

        distributor.base = ctx.accounts.base.key();
        distributor.bump = bump;

        distributor.root = root;
        distributor.mint = ctx.accounts.mint.key();

        distributor.max_total_claim = max_total_claim;
        distributor.max_num_nodes = max_num_nodes;
        distributor.total_amount_claimed = 0;
        distributor.num_nodes_claimed = 0;

        Ok(())
    }
/// Accounts for [merkle_distributor::new_distributor].
#[derive(Accounts)]
#[instruction(bump: u8)]
pub struct NewDistributor<'info> {
    /// Base key of the distributor.
    pub base: Signer<'info>,

    /// [MerkleDistributor].
    #[account(
        init,
        seeds = [
            b"MerkleDistributor".as_ref(),
            base.key().to_bytes().as_ref()
        ],
        bump = bump,
        payer = payer
    )]
    pub distributor: Account<'info, MerkleDistributor>,

    /// The mint to distribute.
    pub mint: Account<'info, Mint>,

    /// Payer to create the distributor.
    pub payer: Signer<'info>,

    /// The [System] program.
    pub system_program: Program<'info, System>,
}
/// State for the account which distributes tokens.
#[account]
#[derive(Default)]
pub struct MerkleDistributor {
    /// Base key used to generate the PDA.
    pub base: Pubkey,
    /// Bump seed.
    pub bump: u8,

    /// The 256-bit merkle root.
    pub root: [u8; 32],

    /// [Mint] of the token to be distributed.
    pub mint: Pubkey,
    /// Maximum number of tokens that can ever be claimed from this [MerkleDistributor].
    pub max_total_claim: u64,
    /// Maximum number of nodes that can ever be claimed from this [MerkleDistributor].
    pub max_num_nodes: u64,
    /// Total amount of tokens that have been claimed.
    pub total_amount_claimed: u64,
    /// Number of nodes that have been claimed.
    pub num_nodes_claimed: u64,
}
```

```typescript
tokenMint = createMint(...) //创建token
const baseKey = Keypair.generate();

export const findDistributorKey = async ( // 查找pda
  base: PublicKey
): Promise<[PublicKey, number]> => {
  return await PublicKey.findProgramAddress(
    [utils.bytes.utf8.encode("MerkleDistributor"), base.toBytes()],
    PROGRAM_ID
  );
};

    const ixs: TransactionInstruction[] = [];
    ixs.push(
      sdk.program.instruction.newDistributor( // 创建pda账户，并初始化
        bump,
        toBytes32Array(root),
        args.maxTotalClaim,
        args.maxNumNodes,
        {
          accounts: {
            base: baseKey.publicKey,
            distributor,
            mint: tokenMint,
            payer: provider.wallet.publicKey,
            systemProgram: SystemProgram.programId,
          },
        }
      )
    );
    
    const { distributorATA, instruction } = await getOrCreateATA({ //创建pda账户的token关联账户
      provider,
      mint: tokenMint,
      owner: distributor,
    });

  // Seed merkle distributor with tokens
  const ix = SPLToken.createMintToInstruction( // 给关联账户打钱
    TOKEN_PROGRAM_ID,
    mint,
    pendingDistributor.distributorATA,
    provider.wallet.publicKey,
    [],
    maxTotalClaim
  );
```

## claim

```rust
    /// Claims tokens from the [MerkleDistributor].
    pub fn claim(
        ctx: Context<Claim>,
        _bump: u8,
        index: u64,
        amount: u64,
        proof: Vec<[u8; 32]>,
    ) -> ProgramResult {
        let claim_status = &mut ctx.accounts.claim_status;
        assert_owner!(claim_status.to_account_info(), ID);
        require!(
            // This check is redudant, we should not be able to initialize a claim status account at the same key.
            !claim_status.is_claimed && claim_status.claimed_at == 0,
            DropAlreadyClaimed
        );

        let claimant_account = &ctx.accounts.claimant;
        let distributor = &ctx.accounts.distributor;
        require!(claimant_account.is_signer, Unauthorized);

        // Verify the merkle proof.
        let node = solana_program::keccak::hashv(&[
            &index.to_le_bytes(),
            &claimant_account.key().to_bytes(),
            &amount.to_le_bytes(),
        ]);
        require!(
            merkle_proof::verify(proof, distributor.root, node.0),
            InvalidProof
        );

        // Mark it claimed and send the tokens.
        claim_status.amount = amount;
        claim_status.is_claimed = true;
        let clock = Clock::get()?;
        claim_status.claimed_at = clock.unix_timestamp;
        claim_status.claimant = claimant_account.key();

        let seeds = [
            b"MerkleDistributor".as_ref(),
            &distributor.base.to_bytes(),
            &[ctx.accounts.distributor.bump],
        ];

        assert_ata!(
            ctx.accounts.from,
            ctx.accounts.distributor,
            distributor.mint
        );
        require!(
            ctx.accounts.to.owner == claimant_account.key(),
            OwnerMismatch
        );
        token::transfer(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                token::Transfer {
                    from: ctx.accounts.from.to_account_info(),
                    to: ctx.accounts.to.to_account_info(),
                    authority: ctx.accounts.distributor.to_account_info(),
                },
            )
            .with_signer(&[&seeds[..]]),
            amount,
        )?;

        let distributor = &mut ctx.accounts.distributor;
        distributor.total_amount_claimed =
            unwrap_int!(distributor.total_amount_claimed.checked_add(amount));
        require!(
            distributor.total_amount_claimed <= distributor.max_total_claim,
            ExceededMaxClaim
        );
        distributor.num_nodes_claimed = unwrap_int!(distributor.num_nodes_claimed.checked_add(1));
        require!(
            distributor.num_nodes_claimed <= distributor.max_num_nodes,
            ExceededMaxNumNodes
        );

        emit!(ClaimedEvent {
            index,
            claimant: claimant_account.key(),
            amount
        });
        Ok(())
    }

/// [merkle_distributor::claim] accounts.
#[derive(Accounts)]
#[instruction(_bump: u8, index: u64)]
pub struct Claim<'info> {
    /// The [MerkleDistributor].
    #[account(mut)]
    pub distributor: Account<'info, MerkleDistributor>,

    /// Status of the claim.
    #[account(
        init,
        seeds = [
            b"ClaimStatus".as_ref(),
            index.to_le_bytes().as_ref(),
            distributor.key().to_bytes().as_ref()
        ],
        bump = _bump,
        payer = payer
    )]
    pub claim_status: Account<'info, ClaimStatus>,

    /// Distributor ATA containing the tokens to distribute.
    #[account(mut)]
    pub from: Account<'info, TokenAccount>,

    /// Account to send the claimed tokens to.
    #[account(mut)]
    pub to: Account<'info, TokenAccount>,

    /// Who is claiming the tokens.
    pub claimant: Signer<'info>,

    /// Payer of the claim.
    pub payer: Signer<'info>,

    /// The [System] program.
    pub system_program: Program<'info, System>,

    /// SPL [Token] program.
    pub token_program: Program<'info, Token>,
}
#[account]
#[derive(Default)]
pub struct ClaimStatus {
    /// If true, the tokens have been claimed.
    pub is_claimed: bool,
    /// Authority that claimed the tokens.
    pub claimant: Pubkey,
    /// When the tokens were claimed.
    pub claimed_at: i64,
    /// Amount of tokens claimed.
    pub amount: u64,
}

/// Emitted when tokens are claimed.
#[event]
pub struct ClaimedEvent {
    /// Index of the claim.
    pub index: u64,
    /// User that claimed.
    pub claimant: Pubkey,
    /// Amount of tokens to distribute.
    pub amount: u64,
}

#[error]
pub enum ErrorCode {
    #[msg("Invalid Merkle proof.")]
    InvalidProof,
    #[msg("Drop already claimed.")]
    DropAlreadyClaimed,
    #[msg("Exceeded maximum claim amount.")]
    ExceededMaxClaim,
    #[msg("Exceeded maximum number of claimed nodes.")]
    ExceededMaxNumNodes,
    #[msg("Account is not authorized to execute this instruction")]
    Unauthorized,
    #[msg("Token account owner did not match intended owner")]
    OwnerMismatch,
}
```

```js
  public async claimIX(
    args: ClaimArgs,
    payer: PublicKey
  ): Promise<TransactionInstruction> {
    const { amount, claimant, index, proof } = args;
    const [claimStatus, bump] = await findClaimStatusKey(index, this.key);

    return this.program.instruction.claim(
      bump,
      index,
      amount,
      proof.map((p) => toBytes32Array(p)),
      {
        accounts: {
          distributor: this.key,
          claimStatus,
          from: this.distributorATA,
          to: await getATAAddress({ mint: this.data.mint, owner: claimant }),
          claimant,
          payer,
          systemProgram: SystemProgram.programId,
          tokenProgram: TOKEN_PROGRAM_ID,
        },
      }
    );
  }
  
export const findClaimStatusKey = async (
  index: u64,
  distributor: PublicKey
): Promise<[PublicKey, number]> => {
  return await PublicKey.findProgramAddress(
    [
      utils.bytes.utf8.encode("ClaimStatus"),
      index.toArrayLike(Buffer, "le", 8),
      distributor.toBytes(),
    ],
    PROGRAM_ID
  );
};
```

