# Notes: Fundraiser × Codama

## The checkpoint 06 question

**Why do I have to pass `fundraiser` and `vault`, but not `contributorAccount` or `contributorAta`?**

Codama can only derive a PDA when every seed is something it already has: a constant, an
argument, or the address of another account in the same instruction. `contributorAccount`
(`"contributor"` + `fundraiser` + `contributor`) and `contributorAta` (`contributor` + token
program + `mintToRaise`) are seeded only by accounts in the instruction, so the builder derives
them. In `contribute`, `fundraiser` is seeded by `fundraiser.maker` and `vault` by
`fundraiser.mint_to_raise`. Those are fields stored inside the fundraiser account's data,
which is invisible until the account is fetched, so Codama leaves them for me to pass.

### What `ContributeAsyncInput` requires

| Account | In the type | Seeds in the IDL (`contribute`) | Why |
| --- | --- | --- | --- |
| `contributor` | required | none (signer) | A signer is never derived; it has to be handed a `TransactionSigner`. |
| `mintToRaise` | required | none | A plain account, not a PDA, so there is nothing to derive it from. |
| `fundraiser` | required | `"fundraiser"`, `fundraiser.maker` | Seed read from account data. |
| `contributorAccount` | optional | `"contributor"`, `fundraiser`, `contributor` | All seeds are accounts in the instruction. |
| `contributorAta` | optional | `contributor`, token program, `mint_to_raise` | All seeds are accounts in the instruction. |
| `vault` | required | `fundraiser`, token program, `fundraiser.mint_to_raise` | Seed read from account data. |
| `tokenProgram` | optional | fixed address | Default value in the IDL. |
| `systemProgram` | optional | fixed address | Default value in the IDL. |

Checked with the compiler: dropping any one of `contributor`, `mintToRaise`, `fundraiser` or
`vault` from the TODO 3 call fails with "Property … is missing"; with those four (plus `amount`)
it compiles. The four the type forces are the four the program cannot work out alone.

### Why the same accounts are optional in `initialize`

In `initialize` the fundraiser does not exist yet, so its seeds cannot come from its own data:
`fundraiser` is seeded by `"fundraiser"` + `maker`, and `vault` by `fundraiser` + token program
+ `mint_to_raise`. Every one of those is an account in the same instruction, so
`InitializeAsyncInput` marks both `fundraiser?` and `vault?` optional and the builder derives
them. It is the same PDA either way. What changes is how the Rust constraint names the seed
(`maker.key()` against `fundraiser.maker`), and Codama resolves only what the IDL can prove.

The reverse also holds. Codama can seed from an account I pass: `contributorAccount` is derived
from `fundraiser`, which is a required input. So if `contribute` wrote the vault constraint as
`associated_token::mint = mint_to_raise` instead of `fundraiser.mint_to_raise` (the `has_one`
already makes them equal), every vault seed would be an instruction account and `vault` would
become optional too. `fundraiser` would stay required either way, because nothing in the
instruction carries `maker`. Checked by regenerating from a scratch copy of the IDL with only
that seed changed: the new `ContributeAsyncInput` has `vault?` and still has `fundraiser`. A visitor in `codama.json` could also supply a custom default for
either account; this assignment leaves `before` empty.

## Versions

| Tool | Version |
| --- | --- |
| anchor-cli | 1.1.2 |
| solana-cli | 3.1.10 (Agave) |
| node | v22.22.2 |
| yarn | 1.22.22 |
| codama CLI (`npx codama --version`) | 1.6.3 |
| @codama/renderers-js | 2.5.0 |
| @codama/nodes-from-anchor | 1.5.6 |
| @solana/kit | 8.3.0 |
| @coral-xyz/anchor (TS client) | 0.32.1 |

Tests run with `anchor test --validator legacy` (`solana-test-validator`) for the whole
assignment. Result: **15 passing, 0 failing**: `codama` 4 (TODO 1-3 and the bonus),
`fundraiser` 7, `time-window` 3, `time-window-bankrun` 1.

One snag worth knowing: the `test` script in `Anchor.toml` passes `tests/**/*.ts` unquoted, and
the shell `anchor test` uses has no `**`, so it expands to `tests/*/*.ts`, which only matches
`tests/helpers/kit-adapter.ts`. The run then reports "0 passing". Changing the pattern locally
to `tests/*.ts` runs all four suites. That edit is not committed, like the synced program ID.
