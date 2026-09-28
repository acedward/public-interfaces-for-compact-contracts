# MIP-xxxx: Public Interfaces for Midnight Contracts

## Summary

This repository contains a Draft MIP for publishing and verifying Midnight contract interface artifacts, plus a Compact-specific reference implementation that can also evaluate operations locally. A contract emits an event identifying a bundle commitment and retrieval location. A consumer can then check the committed files, compare published verifier keys with named installed operations, and reproduce the generated artifacts.

The [MIP methodology](MIP-SPEC-DRAFT.md) defines three cumulative levels:

1. Level 1 matches published files with the on-chain event commitment.
2. Level 2 matches every published verifier key with its same-named installed
   verifier key at the identified state.
3. Level 3 reproduces keys, generated interface code, operation instructions, and supporting artifacts from source using a trusted build toolchain and disclosed inputs.

Named operations are contract entry points backed by zero-knowledge circuits.

The [published Stagenet example walkthrough](#verify-the-published-stagenet-example) provides copyable commands for the repository's Level 2 and Level 3 checks.

The MIP names the `[v1]` commitment profile and the artifact-verification workflow; it does not define this repository's full wire schema, toolchain, or command line. The Compact prototype's `L1`, `L2`, and `L3` output describes its concrete checks, not universal format conformance. Invocation and local execution are prototype features outside the MIP.

An interface can be open or partial-source:

- An open interface imports and therefore publishes the contract module.
- A partial-source interface publishes only the selected named operations and
  enough ledger layout to reproduce their generated artifacts.

Neither form authenticates original ledger field names. Those names are source labels and can change while keys remain equal. A circuit without a corresponding installed verifier key—including a pure circuit published only as a helper—cannot be independently authenticated at Level 2. Its contribution to a named keyed operation is covered by that operation's Level 3 reproduction.

The [full-contract versus private-interface comparison](#full-contract-versus-private-interface) shows the repository's concrete Compact 0.34.0 example of different source text producing the same six named verifier keys.

Repository map:

- `compact/`: publisher module and interface template.
- `compact-examples/`: OpenZeppelin-based examples.
- `src/`: bundle assembler, verifier, retrieval, and execution prototype.
- `scripts/` and `test/`: builds and checks.
- `deploy-tools/`: historical Stagenet example tooling; not library code.

The Compact reference implementation targets Midnight 2.x / Ledger v9 with Compact compiler 0.34.0, language 0.26.0, and runtime 0.19.0. Node 20 or later is required.

```sh
npm ci
npm run check
```

License: Apache-2.0. Vendored OpenZeppelin sources and the adapted interface are
MIT; see [NOTICE](NOTICE).

## How to

### Contract owners: how to use the reference format

Use `compact/OffChainInterface.compact` and the bundle assembler
`src/deployer.mjs` (`coc-deploy-check`). These steps document this repository's
format; the MIP is authoritative for the methodology and guarantee labels.
The invocation examples and safety cautions later in this README describe
prototype behavior outside the MIP's artifact-verification endpoint.

1. Import the publisher module and expose its event circuit:

   ```compact
   import "./OffChainInterface" prefix OffChainInterface_;

   export circuit publishBundle(payload: Bytes<256>): [] {
     return OffChainInterface_publishBundle(payload);
   }
   ```

2. Start from `compact/Interface.template.compact`. Export each intended read
   under its installed operation name. Do not publish a witness-dependent,
   effectful, or context-dependent operation as a public read.

   An open interface can forward operations from the full module:

   ```compact
   import "./FungibleTokenReadable" prefix Token_;

   export circuit name(): Opaque<"string"> { return Token_name(); }
   export circuit totalSupply(): Uint<128> { return Token_totalSupply(); }
   ```

   A partial-source interface can declare the same ledger positions/types and
   include only the selected logic:

   ```compact
   ledger field1: Boolean;
   ledger field2: Map<Either<Bytes<32>, ContractAddress>, Uint<128>>;
   // Remaining fields stay in their compiled positions and types.

   export circuit totalSupply(): Uint<128> {
     assert(field1, "token not initialized");
     return field4;
   }
   ```

   Field labels in that source are not authenticated original names. Compiler
   transformations are version-sensitive; matching rebuilt artifacts and keys,
   rather than a list of supposedly safe edits, is the acceptance test.

3. Build the full contract and interface with the pinned compiler:

   ```sh
   compact compile MyContract.compact out/full
   compact compile MyContract.Interface.compact out/interface
   ```

4. Assemble the bundle:

   ```sh
   node src/deployer.mjs --interface-src MyContract.Interface.compact \
     --interface out/interface --full out/full \
     --url https://you.example/mycontract/ --out bundle/
   ```

   The prototype compares the interface keys with the local full build, writes
   the bundle, and prints the index URI, commitment, and 256-byte payload. This
   does not compare against live installed state and therefore does not itself
   establish Level 2.

5. Host the exact bundle bytes. Keep every published version immutable.

6. Submit `publishBundle(payload)` and retain the transaction. Treat a
   submission as pending until its event is applied and observed. A later
   recognized event supersedes an older one even if the later bundle is invalid
   or unavailable.

7. Decide publication authorization in the contract. The supplied module has
   none: any successful caller can publish a newer event. Event provenance does
   not itself mean owner endorsement.

   Authorization is application policy outside the MIP. For example, a contract
   can require knowledge of a separately committed publisher secret:

   ```compact
   import "./OffChainInterface" prefix OffChainInterface_;

   witness publisherSecret(): Bytes<32>;
   export ledger publisher: Bytes<32>;

   export circuit publishBundle(payload: Bytes<256>): [] {
     assert(
       persistentHash<Vector<2, Bytes<32>>>(
         [pad(32, "coc:publisher:"), publisherSecret()]
       ) == publisher,
       "only the publisher can publish a bundle"
     );
     return OffChainInterface_publishBundle(payload);
   }
   ```

   This example requires a witness for publication, not for a public read. Its
   authority, secret lifecycle, and deployment are the contract owner's
   responsibility.

For a local simulation after the examples have been built:

```sh
node src/deployer.mjs --example fungible --url https://example.invalid/fungible/
node scripts/simulate-deploy.mjs fungible
node src/verify.mjs --bundle bundle/fungible \
  --event-payload "$(cat sim/fungible/event-payload.hex)" \
  --state sim/fungible/state.hex --level 3
```

Offline payload and state files can exercise verification, but they do not
establish a network, emitting address, canonical order, or state provenance.

### Full contract versus private interface

The checked-in [full contract](compact-examples/fungible/Full.compact) imports the [readable wrapper](compact-examples/openzeppelin/FungibleTokenReadable.compact), which imports the complete [vendored token module](compact-examples/openzeppelin/vendor/token/FungibleToken.compact). The [private interface](compact-examples/fungible-private/Interface.compact) instead declares the same seven ledger slots in the same order and with the same types under `hidden1` through `hidden7`, then includes only six named reads. The full contract also exports write circuits such as `transfer` and `approve`; the private interface does not publish that write logic.

These are labeled excerpts from the linked files, not standalone Compact programs. The full module keeps its original field names and calls a helper before reading `_totalSupply`:

**Full token module excerpt:**

```compact
  export ledger _isInitialized: Boolean;
  export ledger _balances: Map<Either<Bytes<32>, ContractAddress>, Uint<128>>;
  export ledger _allowances: Map<Either<Bytes<32>, ContractAddress>,
                                 Map<Either<Bytes<32>, ContractAddress>, Uint<128>>>;
  export ledger _totalSupply: Uint<128>;

  export sealed ledger _name: Opaque<"string">;
  export sealed ledger _symbol: Opaque<"string">;
  export sealed ledger _decimals: Uint<8>;

  // ...

  circuit assertInitialized(): [] {
    assert(_isInitialized, "FungibleToken: contract not initialized");
  }

  // ...

  export circuit totalSupply(): Uint<128> {
    assertInitialized();
    return _totalSupply;
  }
```

The private interface renames those slots and inlines the same initialization assertion in the selected operation:

**Private interface excerpt:**

```compact
ledger hidden1: Boolean;
ledger hidden2: Map<Either<Bytes<32>, ContractAddress>, Uint<128>>;
ledger hidden3: Map<Either<Bytes<32>, ContractAddress>, Map<Either<Bytes<32>, ContractAddress>, Uint<128>>>;
ledger hidden4: Uint<128>;
sealed ledger hidden5: Opaque<"string">;
sealed ledger hidden6: Opaque<"string">;
sealed ledger hidden7: Uint<8>;

// ...

export circuit totalSupply(): Uint<128> {
  assert(hidden1, "FungibleToken: contract not initialized");
  return hidden4;
}
```

These different sources reproduced the same six named keys under Compact 0.34.0: retained validation used the repository's [key comparison script](scripts/check-keys.mjs) to compare the `name`, `symbol`, `decimals`, `totalSupply`, `balanceOf`, and `allowance` verifier-key files byte for byte between the private and full builds. This controlled result does not identify unique original source text or authenticate the original ledger labels, and it does not claim that arbitrary source changes preserve keys.

Each published bundle must still pass Level 3 against its own source and generated artifacts. Changing source or generated artifacts requires rebuilding the generated artifacts and publishing a new bundle commitment. The full build and private-interface bundle are not claimed to share JavaScript, metadata, or commitments merely because these six keys match.

### Consumers: how to read and verify

Obtain the verifier independently of the bundle. The prototype command is:

```sh
node src/verify.mjs \
  --indexer https://<indexer>/api/v4/graphql \
  --address <contract-address> --level 3
```

Its current stages are:

- Level 1 parses the current index format, compares its hash with the selected
  event, recomputes the commitment, checks listed bytes, and compares selected
  package/compiler metadata.
- Level 2 compares shipped keys with named installed operations and checks the
  generated `expectedVk` table when present. It does not authenticate arbitrary
  wrapper behavior.
- Level 3 rebuilds and compares keys, `index.js`, and `contract-info.json`.
  Only Level 3 ties the code to reproduced source and artifacts under the
  selected compiler; it still does not prove unique original source, original
  ledger names, or that generated JavaScript is deployed on chain.

Use `--bundle <dir>` for a local bundle or `--bundle-url <url>` for another
retrieval source. `--event-payload <hex> --state <hex-or-file>` supplies offline
inputs. Such inputs are caller assertions unless separately bound to an
authenticated network/address/event/state record.

Execution is outside the MIP. The prototype can invoke an operation with
`--circuit <name> --args ...`; when using that separate feature:

- at Level 2 the publisher's wrapper remains trusted;
- the prototype has no complete effect/context eligibility monitor;
- it supplies dummy context for values such as the contract address; and
- the child process inherits the caller's host privileges.

Do not execute an untrusted bundle outside a separate security sandbox. Do not
interpret a returned value as a proof, transaction simulation, future-state
prediction, or evidence that a transaction would be accepted.

The prototype CLI currently accepts these argument spellings:

- `Bytes<N>`: exactly `2N` hexadecimal digits, optionally prefixed by `0x`;
- `Uint` and `Field`: exact decimal integers in range;
- `Boolean`: `true`/`false`, `1`/`0`, or `yes`/`no`;
- `Either`: `key:<hex>` or `addr:<hex>`;
- `Maybe`: `none` or `some:<value>`.

These are CLI spellings for this implementation. Other implementations can use
different representations while still reporting the typed arguments they used.

### Verify the published Stagenet example

The repository retains this historical Stagenet publication. The bundle commitment identifies the committed file paths and contents; it is not a SHA-256 hash of `index.json`.

| Item | Value |
|---|---|
| Contract | `5d3233163cd730afb8a31b3e61e77fbd5949fa05d35920bd2b5cea32febaa0f6` |
| Hosted bundle index | [https://compact-off-chain-circuits.pages.dev/public-interface/erc20-private/index.json](https://compact-off-chain-circuits.pages.dev/public-interface/erc20-private/index.json) |
| Expected bundle commitment | `4814bf93c6c0a6c81c7839f9be72c80365c2a4179d58171e7acd40906be30891` |
| Publication transaction | `79fa53ab3601a373b778d3c0f6d457784457c5540276b254c85d55a7bc55b3de` at block 608267 |
| Stagenet indexer | `https://indexer.stagenet.shielded.tools/api/v4/graphql` |

Start from an independent checkout with Node 20 or later and install the locked dependencies:

```sh
git clone https://github.com/acedward/public-interfaces-for-compact-contracts.git
cd public-interfaces-for-compact-contracts
npm ci
```

Level 2 includes Level 1. This command obtains the newest publication event and installed keys from the named contract, fetches the explicit hosted URL, checks that its index and 13 committed files give the event commitment, and compares the six published verifier keys with the same-named installed keys:

```sh
npm run verify -- \
  --bundle-url https://compact-off-chain-circuits.pages.dev/public-interface/erc20-private/index.json \
  --indexer https://indexer.stagenet.shielded.tools/api/v4/graphql \
  --address 5d3233163cd730afb8a31b3e61e77fbd5949fa05d35920bd2b5cea32febaa0f6 \
  --level 2
```

The URL override remains subject to the selected on-chain event: Level 1 requires the fetched index and recomputed file commitment to equal that event's commitment. A successful run reports commitment `4814bf93c6c0a6c81c7839f9be72c80365c2a4179d58171e7acd40906be30891`, `L1 OK` for the commitment and all 13 files, `L2 OK` for `allowance`, `balanceOf`, `decimals`, `name`, `symbol`, and `totalSupply`, then `verified up to level 2`. Compare the printed commitment with the expected value in the table; a different value identifies a replacement publication rather than this recorded example, even if the checks for that newer publication succeed.

Level 3 additionally requires the trusted Compact 0.34.0 compiler launcher on `PATH`. Check the compiler binary rather than only the toolchain manager:

```sh
compact compile --version
```

Then rerun the same artifact-only verification at Level 3:

```sh
npm run verify -- \
  --bundle-url https://compact-off-chain-circuits.pages.dev/public-interface/erc20-private/index.json \
  --indexer https://indexer.stagenet.shielded.tools/api/v4/graphql \
  --address 5d3233163cd730afb8a31b3e61e77fbd5949fa05d35920bd2b5cea32febaa0f6 \
  --level 3
```

A successful Level 3 run repeats Levels 1 and 2, reports `L3 OK` for all six verifier keys, `contract/index.js`, and `compiler/contract-info.json`, then `verified up to level 3`. Neither command supplies `--circuit`, so the verifier does not invoke an operation. No wallet, mnemonic, proof service, signing, transaction, or redeployment is required.

The historical record also reports `name() = "Off-Chain Reads Private Token"`, `symbol() = "OCRP"`, `decimals() = 18`, and `totalSupply() = 1000000000000000000000000`. That supply is assigned to the keyless demo holder `13f03a2916c2bbb04b050ffb5061187386c73af8ba57bf70c7ddf1fa8c2a005a`. The deployment inserted unpublished write operations in blocks 608232 to 608238 before publishing the read bundle. These are retained deployment facts; the verification commands above do not execute those operations.

This is a reference observation, not an activation or conformance claim. Public services can be incomplete or queried at different snapshots; record their network, event, state, and observation limitations.

## Spec

The [MIP](MIP-SPEC-DRAFT.md) defines publication and artifact verification through reproduced build-toolchain output for named operations. It stops before code invocation. This README is an operator guide for the Compact reference implementation's formats, tools, commands, and optional execution features; those details are not universal methodology requirements.
