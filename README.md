# Pre-Req Vault

A simple Anchor-based Solana Vault program that manages user SOL and performs a Cross-Program Invocation (CPI) into the Registration Program during withdrawal.

The main objective is to extend the `withdraw` instruction so that, after withdrawing SOL, it registers the user's GitHub handle through the Registration Program.

---

## Architecture
```text
                         USER WALLET
                        User / Signer
                       User Public Key
                              |
             +----------------+----------------+
             |                                 |
             v                                 v
      +--------------+                 +-------------------+
      | VAULT PROGRAM|                 | REGISTRATION      |
      |              |                 | PROGRAM           |
      | initialize() |                 |                   |
      | deposit()    |                 | initialize(github)|
      | withdraw()   |                 |                   |
      | close()      |                 +---------+---------+
      +------+-------+                           |
             |                                   |
             | owns                              | owns
             v                                   v
      +---------------+                 +----------------------+
      | VaultState PDA|                 | Application Account  |
      |               |                 | PDA                  |
      | ["state",     |                 | ["prereqs", user]    |
      |  user]         |                 |                      |
      |               |                 | GitHub username      |
      | Vault state   |                 | Registration data    |
      +-------+-------+                 +----------------------+
              |
              | derives
              v
      +---------------+
      |   Vault PDA   |
      |               |
      | ["vault",     |
      |  vault_state] |
      |               |
      |   HOLDS SOL   |
      +---------------+
```


### Withdraw Flow : 
```text

      USER
       |
       | withdraw(amount)
       v
 +-------------+
 |    VAULT    |
 |   PROGRAM   |
 +------+------+ 
        |
        +-----------------------> USER
        |                         SOL transfer
        |
        | CPI
        v
 +---------------------+
 | REGISTRATION        |
 | PROGRAM             |
 +----------+----------+
            |
            | initialize (github: sidarth16)
            v
 +----------------------+
 | APPLICATION ACCOUNT |
 | PDA                  |
 +----------------------+
```

## Demo Video

A short walkthrough of the Vault architecture, program flow, and CPI-based registration.

https://www.youtube.com/watch?v=T4MkOpZ08zw

---

## Program Overview

The system consists of two programs:

### Vault Program

The Vault Program is the Anchor program modified for this challenge.

It exposes four instructions:

- `initialize`
- `deposit`
- `withdraw`
- `close`

### Registration Program

The Registration Program is the external program invoked by the Vault during `withdraw`.

Its `initialize` instruction is used to register the user's GitHub username.

---

## Account Model

### VaultState PDA

Derived using:

```text
["state", user]
```

The `VaultState` PDA represents the user's Vault state and stores the required bump information.

### Vault PDA

Derived using:

```text
["vault", vault_state]
```

The Vault PDA is the account that holds the user's SOL.

### Application Account PDA

Derived using:

```text
["prereqs", user]
```

This PDA belongs to the Registration Program and stores the user's registration information, including the GitHub username.

---

## Program Flow

### Initialize

Creates and initializes the user's Vault state and establishes the Vault PDA relationship.

```text
User
 |
 | initialize()
 v
Vault Program
 |
 +--> VaultState PDA
 |
 +--> Vault PDA
```

### Deposit

Transfers SOL from the user's wallet into the Vault PDA.

```text
User
 |
 | deposit(amount)
 v
Vault Program
 |
 | SOL transfer
 v
Vault PDA
```

### Withdraw

The `withdraw` instruction performs two operations:

1. Transfers the requested SOL from the Vault PDA back to the user.
2. Performs a CPI into the Registration Program.

```text
User
 |
 | withdraw(amount)
 v
Vault Program
 |
 +---- SOL transfer ----> User
 |
 +---- CPI -------------> Registration Program
                              |
                              | initialize(github)
                              v
                       Application Account PDA
```

The client only calls the Vault Program. The Registration Program is invoked internally through CPI.

### Close

Closes the Vault state and returns the remaining lamports to the user.

```text
User
 |
 | close()
 v
Vault Program
 |
 +---- remaining lamports ----> User
 |
 +---- close VaultState
```

---

## CPI Flow

The main addition for this challenge is the CPI from `withdraw` into the Registration Program.

```text
                 withdraw()
User ----------------------------> Vault Program
                                      |
                                      | SOL transfer
                                      v
                                    User

                                      |
                                      | CPI
                                      v
                              Registration Program
                                      |
                                      | initialize(github)
                                      v
                              Application Account PDA
```

The CPI implementation is located in:

`programs/pre-req-vault/src/instructions/withdraw.rs`

The Registration Program interface is provided through:

`idls/registration.json`

---

## On-Chain Verification

The modified Vault Program was deployed to Solana Devnet and the complete flow was verified on-chain.

**Vault Program ID:** `GUkfN2WWpev6PWc7ibsqwdKAB2RvkP7GrFSYLr9Tiv52`

[View Program on Solana Explorer](https://explorer.solana.com/address/GUkfN2WWpev6PWc7ibsqwdKAB2RvkP7GrFSYLr9Tiv52?cluster=devnet)


### Verified Transactions

- **Initialize the vault** — [View Transaction](https://explorer.solana.com/tx/EoVVvQgpmAwALMyXQifjeovEQqdHhe99WBEJ7NdPKacmxzWNDXxoXtoBFCeMpm13KjRpsPiQSzKEMn15YDBmK4M?cluster=devnet)

- **Deposit 1 SOL in to the vault** — [View Transaction](https://explorer.solana.com/tx/9qwjZZ7QRnpgeEVCtQFmtCstrvP1CTXMMAYGx6J992AXxEDUXHmy2dkrm2vMuMUsHDFHze2zkkd6RC9X5LtfGE2?cluster=devnet)

- **Withdraw 0.5 SOL from the vault + Registration CPI** — [View Transaction](https://explorer.solana.com/tx/3fAzEiegXk5Efre1gSBpLEFBeCtzhqf7QeKwZ5w8xSBvWaFs6p2KBWvdionwbyv7NzwjAKTrnLLUTNK6K48rbqDE?cluster=devnet)

- **Close the Vault** — [View Transaction](https://explorer.solana.com/tx/5T3Gw4k8RbYKngE5TpHdeMrsdEos5grHTGQgbA7wsFm3UdypVFCfTLWbaN2tLQC8cM6g5RXj6Y7BXBdPnTktyv4G?cluster=devnet)

The withdraw transaction contains the Registration Program's inner `initialize` instruction, confirming that registration was performed through CPI as part of the withdrawal transaction.

---

## Implementation

The CPI logic was added to:

`programs/pre-req-vault/src/instructions/withdraw.rs`

The Registration Program interface is available at:

`idls/registration.json`

---

## Running Locally

Build the program:

```bash
anchor build
```

Run the tests against the deployed program:

```bash
anchor test --skip-deploy
```