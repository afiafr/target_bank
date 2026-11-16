# Processing fees

Every successful withdrawal, deposit, bank transfer, and guardian transfer incurs a **3% processing fee**. Failed attempts, previews, balance inquiries, staff cash handling, and account administration incur no fee. Historical transactions are not charged retroactively.

The fee uses principal in the source account currency before existing ATM/exchange fees. Integer arithmetic rounds up to the smallest supported unit (one cent, or a whole yen/won). Foreign cash principal is the converted cash amount.

- Withdraw $100: debit $103, plus any existing ATM/exchange fees.
- Deposit $100: credit $97, less any existing exchange fee.
- Transfer $100: debit the sender $103; credit the recipient $100.
- Guardian transfers: the account money leaves pays the fee.

Withdrawals and transfers require sufficient funds including fees. Deposits must leave a positive net credit. Daily limits count principal, excluding processing fees. Receipts show processing fees and net totals. Each fee has a separate `processing_fee` audit entry labeled **Processing fee**, linked to its original transaction. Balance reconciliation subtracts fee entries; cash reconciliation does not treat fees as physical bills removed.

## Programmer provisioning

Generate a dedicated key pair on the programmer's machine before money transactions. Put the private key outside the bank delivery folder:

```sh
cargo run --features programmer-tools --bin bank-processing-fees -- provision /path/to/atm.db /programmer-only/atm.processing-fees.private.pem
```

This creates `/path/to/atm.processing-fees.public.pem` and the private key without overwriting existing files. Deliver **only the public key**, beside the matching bank database. Restart the ATM to install it. Existing banks can be provisioned too; the installed recipient is pinned in the sealed state. Without a provisioned public key, money transactions roll back; setup and read-only actions remain available.

Keep the private key backed up securely off the bank machine. Neither the ATM nor `atm.key` can recover it. This key is independent of `.programmer-pin.secret`, which also stays with the programmer. Normal GUI builds exclude the programmer tool and private key.

## Separate encrypted database

The ATM writes `atm.processing-fees.db` next to `atm.db`. Each record contains a fee audit reference and an RSA-OAEP-SHA256 encrypted payload. Only the private key holder can decrypt the payload. The versioned JSON array contains: version, original transaction reference, UTC timestamp, source account id, transaction kind, currency, principal in hundredths, and fee in hundredths. Keep currencies distinct when reporting totals.

Read a copy offline on the programmer machine:

```sh
cargo run --features programmer-tools --bin bank-processing-fees -- read /backup/atm.processing-fees.db /programmer-only/atm.processing-fees.private.pem
```

The bank can access the ciphertext file as application owner and see its fee charges in the bank audit. Encryption protects the separate records; it does not hide fees from the bank or customers. No teller/customer/GUI endpoint reads the private ledger.

Both databases commit in one SQLite transaction using DELETE rollback journals and FULL synchronization. A failure rolls back balances, cash, audit entries, and private records together. The bank seal includes a hash of the separate database schema, public key, and encrypted rows; missing or changed data stops operations.

Back up and restore **both databases and the matching `atm.anchor` together** from a consistent snapshot while the ATM is stopped. Keep `atm.key` and the programmer private key in their respective custody. The offline tool decrypts records; the running ATM supplies sealed-state integrity and rollback checks.
