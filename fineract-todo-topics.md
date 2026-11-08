# Fineract Learning Roadmap

One linear path. Do steps in order. Tick a box only when you can explain it with no notes.
Per step: concept first, then Swagger call, then DB rows, then code.
Items marked `(verify)` are unconfirmed guesses. Fineract has no cards, VCN or tokenization.
Ask in chat "trace step N" to get a code-verified answer with `file:line`.

## Part A: Money basics (no code)

- [x] 1. Double-entry: every transaction has debits = credits (5 account types: asset, liability, equity, income, expense). Ledger rows are never updated, only reversed.
  - Bank view: a loan is an ASSET (customer owes bank), a savings balance is a LIABILITY (bank owes customer). Opposite of wallet instinct.
  - Practice: disburse 1000 into savings = Dr Loan Portfolio 1000 / Cr Customer Savings 1000. Withdraw 400 cash = Dr Customer Savings 400 / Cr Cash 400.
  - Verified: Debit/Credit have no fixed increase/decrease meaning — each account type sits on a fixed side of `Assets + Expenses = Liabilities + Equity + Income`; a posting to an account's own side increases it, to the opposite side decreases it. Golden rule per transaction_id: SUM(debit amounts) = SUM(credit amounts).
  - Verified: two separate layers, not one. Sub-ledger (`m_savings_account_transaction`, `running_balance_derived` — the customer's real balance) vs GL control accounts (`acc_gl_account`/`acc_gl_journal_entry` — pooled across all customers, e.g. "Savings Control", "Loan Portfolio", "Fund Source", no balance column, derived via trial balance). `JournalEntry.java:40-107` has no direct FK to the client; chain is `acc_gl_journal_entry.savings_transaction_id -> m_savings_account_transaction.savings_account_id -> m_savings_account.client_id -> m_client`.
- [ ] 2. Money = decimal, never float. Define currency decimals, rounding mode, and when to round.
- [ ] 3. Idempotency: a retry must not apply a repayment twice. Read `fineract-doc/src/docs/en/chapters/architecture/idempotency.adoc`, `principles.adoc`.

## Part B: Run it and meet the actors

- [ ] 4. Run server (JDK 25, see CLAUDE.md), open Swagger, use `integration-tests/` as example flows.
- [ ] 5. Office (branch tree), Staff, Users, Roles, Permissions. Module: `fineract-branch`.
- [ ] 6. Client: create, activate, close. Client vs Group vs Center. Code: `ClientsApiResource` (`fineract-provider/.../portfolio/client/api/`). Check `m_client` rows.

## Part C: Savings (the "bank account")

- [ ] 7. Savings product (template) vs savings account (instance). Lifecycle: submitted, approved, active, closed/dormant.
- [ ] 8. Deposit, withdrawal, reversal. Code: `SavingsAccountsApiResource`, `SavingsAccount`, `SavingsAccountTransaction` (`fineract-savings`).
- [ ] 9. Interest: calculate (daily accrual) vs compound (added to base) vs post (real transaction). Compare `totalInterestEarned` vs `totalInterestPosted` in `SavingsAccountSummary`. Doc: `features/savings-interest-posting.adoc`.
- [ ] 10. Overdraft (`allowOverdraft`, `overdraftAmount`), minimum balance (`enforceMinRequiredBalance`), lock-in, dormancy, savings charges.
- [ ] 11. Transfers and standing instructions (`portfolio/transfer`, `portfolio/account`).

## Part D: Balances (the "two balances" question)

Fields confirmed in code: `SavingsAccountSummary.accountBalance`, `runningBalanceOnInterestPostingTillDate`, `runningBalanceOnPivotDate`; `SavingsAccountTransaction.runningBalance`, `cumulativeBalance`.

- [ ] 12. Why store both an account balance and a per-transaction running balance. Account balance = O(1) "balance now". Running balance = statements, point-in-time balance, interest base. Cost: denormalized, and a back-dated transaction forces recompute of later rows.
- [ ] 13. Pivot date: what it checkpoints and why. Trace `SavingsAccountSummary.updateRunningBalanceAndPivotDate`.
- [ ] 14. Back-dated transactions: what gets recomputed.
- [ ] 15. Available vs ledger balance, holds (`onHold`) `(verify field names)`.
- [ ] 16. Concurrency: two withdrawals at once. Find the locking or version check in the savings write path.

## Part E: Loans

- [ ] 17. Loan product: principal, rate, term, frequency, amortization (equal installments vs equal principal), interest method (flat vs declining).
  - Practice: 1000, 3 months, 12% annual. Flat = 30 total. Declining with equal principal = 10.00 + 6.67 + 3.33 = 20.00.
- [ ] 18. Lifecycle: submitted, approved, disbursed, active, closed (paid / written-off / rescheduled). Code: `LoansApiResource`, `Loan` (`fineract-loan`).
- [ ] 19. Disbursement: single vs tranche (paid out in stages, interest only on disbursed part), to savings vs cash.
- [ ] 20. Schedule (expected) vs transactions (actual). `LoanRepaymentScheduleInstallment`, `LoanTransaction`, `LoanSummary`, package `loanschedule/`. Which is the source of truth after a back-dated repayment?
- [ ] 21. Collateral and guarantors.

## Part F: Repayment and loan problems

- [ ] 22. Repayment allocation order (principal, interest, fees, penalties). Doc: `architecture/advanced-payment-allocation.adoc`.
- [ ] 23. Prepayment, overpayment, credit balance refund.
- [ ] 24. Loan charges, penalties, taxes on charges (`fineract-charge`, `fineract-tax`). Docs: `features/loan-charges.adoc`, `taxes-on-loan-charges.adoc`.
- [ ] 25. Arrears, delinquency, grace, pause delinquency. Doc: `features/pause-delinquency.adoc`.
- [ ] 26. Waive vs write-off vs charge-off vs recovery.
- [ ] 27. Reschedule, re-age, re-amortize, downpayment. Docs: `features/re-ageing.adoc`, `re-amortization.adoc`, `downpayment.adoc`.

## Part G: Interest engines and accrual

- [ ] 28. Cumulative vs progressive schedule model: behavior on early/late repayment. Code: `fineract-progressive-loan*`.
- [ ] 29. Accrual vs cash accounting. Cash = income booked when paid. Accrual = income booked as earned, with a receivable.
- [ ] 30. Interest rate change and backdated modification. Docs: `features/interest-rate-change-progressive-loans.adoc`, `backdated-interest-modification.adoc`.

## Part H: General ledger

- [ ] 31. Chart of accounts: `GLAccount` (`fineract-core`), header vs detail accounts.
- [ ] 32. Product accounting rule: none / cash / accrual periodic / accrual upfront. Product account mappings (`accounting/productaccountmapping`).
- [ ] 33. `JournalEntry` (`fineract-accounting`): debit/credit rows, transaction id, reversal.
- [ ] 34. For each action, write the entries by hand first, then verify in `AccountingProcessorForLoan` and `AccountingProcessorHelper`: disburse, accrue 1, repay 1 interest, write off 500, undo repayment.
- [ ] 35. Accruals, provisioning, financial activity mappings (`accounting/accrual`, `accounting/provisioning`). Doc: `features/journal-entry-aggregation.adoc`.

## Part I: Platform internals

- [ ] 36. Request path: `CommandWrapperBuilder` -> `CommandSource` -> handler -> write service. Why the command is stored BEFORE the handler runs (audit, maker-checker, idempotency). Draw the sequence diagram for "repay loan".
- [ ] 37. Maker-checker: held commands, approval, stale data.
- [ ] 38. Multi-tenancy: tenant header, `TenantAwareBasicAuthenticationFilter`, `RoutingDataSource`, tenant-store DB.
- [ ] 39. Security: Basic, OAuth2, roles, permission codes, 2FA.
- [ ] 40. Business date vs wall clock (day closes after midnight, catch-up, controllable in tests). Doc: `architecture/business-date.adoc`.
- [ ] 41. Persistence and locking: Liquibase, JPA vs JDBC reads, `m_loan_account_locks` soft/hard lock, HTTP 409. Docs: `persistence.adoc`, `loan-locking.adoc`.

## Part J: Batch and events

- [ ] 42. Scheduler jobs vs Close of Business. Code: `fineract-cob`, `COBBusinessStep`, `fineract-provider/.../infrastructure/jobs/`. Doc: `architecture/batch-jobs.adoc`.
- [ ] 43. What loan COB does per loan, partitioning, crash recovery, advancing the business date.
- [ ] 44. Business events and the reliable event framework: avoids the dual-write problem (DB commit and Kafka publish cannot be atomic), outbox-style. Avro schemas in `fineract-avro-schemas`. Doc: `architecture/reliable-event-framework.adoc`.
- [ ] 45. Hooks (older mechanism), notifications, external services.

## Part K: Advanced (pick as needed)

- [ ] 46. Shares (`shareaccounts`, `shareproducts`), fixed/recurring deposits `(verify location)`.
- [ ] 47. Investors and asset externalization (`fineract-investor`). Doc: `features/asset-externalization.adoc`.
- [ ] 48. Loan origination details, buy-down fee, capitalized income (`fineract-loan-origination`).
- [ ] 49. Working-capital loans (`fineract-working-capital-loan`, `features/working-capital-*.adoc`).
- [ ] 50. Reports, documents, rates, custom modules (`custom/`), SDK generation.

## Part L: Capstone

- [ ] 51. Explain to a colleague why core banking is not "just a wallet ledger": products, schedules, accrual, GL, business date, COB, maker-checker.
- [ ] 52. List 5 Fineract design choices to copy and 5 to avoid in your own system.
