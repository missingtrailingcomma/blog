---
title: 'Beancount usage review - 2026 Mid Review'
description: 'Lorem ipsum dolor sit amet'
pubDate: 'Jul 08 2022'
---

I started using [Beancount](https://github.com/beancount/beancount/) on Dec 29, 2025, and have been back-porting all my accounts and transactions, including historical ones. As of today, I have 388 total accounts, 269 open accounts, and 15,512 transactions.

I have about 10 years of financial history, having opened and closed many accounts over that time. This post covers my repo setup for handling many accounts and transactions, and my update routine.

## Principles

Use checks (balance, close) to enforce conformity.

## Repo Setup

I have the following main folders and files:

```
.
├── accounts
│   ├── Assets.beancount
│   ├── Equity.beancount
│   ├── Expenses.beancount
│   ├── Income.beancount
│   ├── Liabilities.beancount
│   ├── balances.beancount
│   ├── commodities.beancount
│   ├── main.beancount
│   ├── not-be-used.beancount
│   └── ...
├── data
│   ├── Assets:Brokerage:Foo0
│   ├── Assets:Liquid:Foo1
│   ├── Liabilities:CreditCard:Foo2
│   ├── Liabilities:Mortgage:Foo3
│   └── ...
├── journals
│   ├── Assets:Brokerage:Foo0.beancount
│   ├── Assets:Liquid:Foo1.beancount
│   ├── Liabilities:CreditCard:Foo2.beancount
│   ├── Liabilities:Mortgage:Foo3.beancount
│   ├── ...
│   └── main.beancount
├── prices
│   ├── live.beancount
│   ├── ...
│   └── main.beancount
├── tools
├── main.beancount
└── AGENTS.md
```

Details:

### `accounts/`

`accounts/` directory contains:

* Account declarations, organized by prefix, e.g. `Assets:Brokerage.beancount`, `Liabilities.beancount`.

* Helper statements:

    * `balances.beancount` contains balance statements.

    * `not-be-used.beancount` contains close statements for accounts that are not yet closed, e.g. `3000-01-01 close Liabilities:Mortgage:CA:BMO:180k:Revolving-087`.

### `data/`

`data/` directory contains accounts' raw data (CSV statements, PDF statements, transaction histories, activity reports, etc.), organized by account name. Plain text is preferred when it has data parity with the original.

### `journals/`

`journals/` directory contains main Beancount transactions, organized by account name.

### `prices/`

`prices/` directory contains `price` statements.

### `tools/`

`tools/` directory contains scripts to generate statements, manage the repo, and analyze data.

### `main.beancount`

`main.beancount` is the entry point, containing references to entry points of sub folders, configs, and plugins. Plugins in use:

```
plugin "beancount.plugins.check_commodity"
plugin "beancount.plugins.check_drained"
plugin "beancount.plugins.close_tree"
```

### `AGENTS.md`

[`AGENTS.md`](https://agents.md) helps AI agents understand the repo.

## Workflows

### Monthly Update

On the 1st of each month, I manually follow markdown instructions:

1. Log in to the financial institution.
1. Download statements for the past month and save to the corresponding account directory in `data/`.
1. Run import scripts: `uv run python -m tools.import extract data/Assets:Brokerage:Foo0/ > journals/Assets:Brokerage:Foo0.beancount`.
1. Commit and continue to the next account.

A few things break the standard flow:

* Running import on all files in a data sub-directory sometimes triggers too many manual edits — e.g. historical transactions without easily-parsable data files, or incorrectly categorized/uncategorized posting accounts for past transactions. In these cases: `uv run python -m tools.import extract data/Assets:Brokerage:Foo0/Bar.csv >> journals/Assets:Brokerage:Foo0.beancount`.

* Some accounts, especially credit cards, don't offer statement downloads for a full calendar month — only by statement period. The latest transactions end up either in a partial statement overlapping next month, or unavailable for download. Options:

    * Download and parse only the latest statement. Data may be stale for a while. Stale data causes unmatched clearing account balances, requiring separate workarounds.

    * Download and parse both the latest statement and the partial statement. Data is not stale, but the partial statement needs to be replaced next month.

    * Download the latest statement for bookkeeping, and manually scrape transaction history into `data/`.

* Some accounts have more automated pipelines, e.g. Questrade's API for retrieving data. These use a custom script instead of a beangulp-based importer.

### Note

* Plaid makes automating monthly updates easier, but is expensive given the number of open accounts.

* AI agents are hard to use for automating monthly updates due to short-lived sessions.
