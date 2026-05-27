# TON Smart Contract Audit Checklist

> Do not use this as a linear tick-box list. Use it as an attack map: first build the message model and value-flow, then verify invariants, then review language-specific and low-level details.

This checklist is not a replacement for a full audit, formal verification, or project-specific threat modeling. For every high-impact item, write either an exploit scenario, a concrete test, or a clear “not applicable” reason.

---

## 0. Audit scope, versions, and artifacts

- The contract language and compiler version are fixed: Tolk / FunC / Tact.
- TVM/version assumptions, toolchain version are fixed.
- Commit hash, build commands, compiler flags, stdlib imports, generated wrappers, ABI, and source maps are fixed.
- For Tolk: version changelog changes are reviewed, especially `lazy`, `createMessage`, `BounceMode`, `address`, `array<T>`, `string`, `unknown`, ABI, wrappers, and source maps.
- For FunC: manual serialization, `impure`, modifying/non-modifying methods, storage packing, `set_data`, and `set_code` are reviewed.
- For Tact: generated wrappers, traits, receivers, fallback behavior, and bounced-message behavior are fixed.
- A list of external contracts, minter/wallet code cells, trusted contracts, oracle/indexer assumptions, and off-chain services is prepared.
- A list of roles is prepared: admin, owner, governance, operator, wallet user, relayer, indexer, validator, attacker.
- Tool availability and language support are checked per project; no static or symbolic tool is treated as complete proof of safety.

---

## 1. Architecture and threat model

- All message-flow diagrams are drawn: internal, external, bounced, deployment, upgrade.
- A complete call/message graph is built: all `recv_internal`, `recv_external`, `onInternalMessage`, `onExternalMessage`, bounce/fallback/get handlers, deployment, upgrade, and child-contract paths.
- For every handler, the following are documented: authorization, parsed fields, state writes, outbound sends, gas/value assumptions, possible bounce, and possible action-phase failure.
- For every operation, value-flow is described: who pays gas, who receives TON/Jetton/NFT, where excesses/refunds are returned.
- For every operation, state transitions are described: pending → sent → success/bounce/finalized/cancelled.
- Invariants are defined: total supply, balances, reserves, ownership, fee accounting, LP supply, debt, pending withdrawals, locked funds, pending claims.
- Invariants are verified under partial execution and interleaving of several message chains.
- The contract does not assume synchronous getter calls to another contract. In TON, on-chain interaction between contracts happens through messages.
- Every step of a multi-message flow revalidates state and does not trust a check made in a previous message.
- Downstream failure does not lead to irreversible loss of funds, incorrect accounting, or a permanently stuck state.
- Protocol assumptions are separated from implementation assumptions: validators, oracles, indexers, relayers, bridges, and wallets are not silently trusted.

---

## 2. Entry points and message handlers

- All entry points are found: `recv_internal`, `recv_external`, `onInternalMessage`, `onExternalMessage`, `onBouncedMessage`, empty receiver, fallback, get methods.
- For each entry point, it is clear who can call it, which fields are parsed, and which checks must happen before state is changed.
- Entry-point inventory includes hidden paths through fallback, empty messages, bounced messages, deployment messages, upgrade messages, and plugin/callback-like flows.
- Empty body is handled intentionally: top-up/deploy/cashback or reject.
- Unknown opcode is handled explicitly: reject or intentional ignore; silent accept is forbidden without a clear reason.
- Unknown opcode handling does not change value-flow in a dangerous way and does not allow griefing through accepted junk messages.
- Bounced messages are not parsed as ordinary internal messages.
- For FunC: the bounced flag in `int_msg_info` is checked before parsing op/body.
- For Tolk: the incoming message union is matched exhaustively; the `else` branch does not silently accept unknown opcodes.
- For Tact: fallback receivers and bounced receivers cover the expected message types.
- Get methods are checked for unsafe assumptions if their output is consumed by off-chain services, UIs, indexers, or relayers.

---

## 3. Authorization, identity, and replay protection

- Every state-changing handler has an explicit caller/identity policy: privileged paths restrict `sender` / `in.senderAddress`, while permissionless paths still validate all sender-derived assumptions.
- Admin/governance functions have strict authorization, preferably multisig/timelock for upgrade/drain operations.
- Blast radius is reviewed: whether a single admin key can withdraw all funds, change code, replace wallet code, change fee receiver, pause withdrawals, or alter accounting.
- Privileged operations cannot be reached through fallback, bounced, empty, deployment, plugin, callback, or upgrade paths.
- For Jetton/NFT: sender is not just “some wallet”; it is the computed expected wallet/collection/minter/item address.
- Forwarded addresses, payload fields, and notification bodies are not trusted as identity unless the sender contract is verified.
- For workchain-sensitive logic: workchain is validated explicitly.
- External messages: signature, seqno, valid_until, subwallet_id, and chain separation are checked before `accept_message()` / `acceptExternalMessage()`.
- Signed data includes recipient, amount, op, query_id/seqno, valid_until, contract address, workchain, and protocol-specific intent. There is no partial signing.
- Seqno/nonce is updated so replay is impossible even under action-phase failure.
- Replay between contracts, cross-wallet replay, cross-workchain replay, signature reuse, and front-running are checked.
- Any `null` / zero / default address state is reviewed so it cannot accidentally become a privileged identity.

---

## 4. External messages and gas-drain

- There is no unconditional `accept_message()` / `acceptExternalMessage()`.
- Cheap checks are performed before accept: format, op, seqno, valid_until, signature, subwallet, and basic bounds.
- Heavy parsing, dictionary traversal, signature-independent hashing, and dynamic allocations do not happen before cheap rejection where avoidable.
- After accept, there are no unbounded loops, heavy parsing, attacker-controlled batch sends, or unbounded dictionary operations.
- For wallet-like contracts, the order is verified: parse → time check → seqno → signature → accept → update seqno → commit → actions.
- For Tolk, the use of `acceptExternalMessage()` and, where needed, `commitContractDataAndActions()` is reviewed.
- `commitContractDataAndActions()` placement does not create replay, committed-bad-state, or action-failure bugs.
- External messages do not rely on randomness.
- Repeated external messages, old seqno, expired `valid_until`, wrong subwallet, and wrong recipient are tested.

---

## 5. Asynchrony, race conditions, and partial execution

- For each multi-message flow, the following scenarios are considered: delayed message, message lost because of action failure, bounce, duplicate, retry, timeout, cancellation, and parallel flow.
- There is no logic like “balance was checked in step 1, therefore it is still the same in step 3.”
- The carry-value pattern is used: value is passed in the message, not requested synchronously from another contract.
- State changes before outbound send are either reversible through bounce/recovery or do not break invariants on failure.
- Partial execution is checked with send modes in mind: with ignored/suppressed action errors, especially `+2` / `SendIgnoreErrors`, compute-phase state and some actions may be committed while one or more outbound sends are skipped.
- Pending states have timeout/cancel/retry/recovery paths.
- Idempotency: a repeated or late message does not credit reward, withdrawal, mint, claim, or refund twice.
- Query IDs are unique within the relevant in-flight/pending-operation scope, or the protocol explicitly proves that reuse is harmless.
- Query IDs are checked together with sender, expected operation, pending state, and amount; they are not used as the only trust anchor.
- Parallel deposits/withdrawals/liquidations/swaps/mints/burns/claims do not overwrite shared variables incorrectly.
- Multi-step flows are tested in different interleavings, including success → bounce, bounce → retry, duplicate response, and late response after cancellation.

---

## 6. Bounce handling

- All bounceable outgoing messages have a corresponding bounce handler, or it is explicitly proven that ignoring bounce is safe.
- The bounce handler restores state: balances, supply, locked amount, pending status, debt, LP accounting, claim status, or protocol-specific reserved value.
- Jetton mint/transfer/burn: failed `internal_transfer`, transfer, mint, or burn notification restores totalSupply/balance/pending accounting where applicable.
- If the bounce body is truncated, critical fields are located in the first bits; for Tact, the 224 useful bits limit is considered; for Tolk, the correct `BounceMode` is selected.
- For Tolk: `BounceMode.NoBounce`, `Only256BitsOfBody`, `RichBounce`, and `RichBounceOnlyRootCell` are selected intentionally.
- Rich/new bounce-body schema compatibility is reviewed where relevant: root-cell-only vs full-body bounce, reserved/unknown fields, original body, phase, exit code, gas used, and parser behavior for future schema changes.
- Different bounce modes are not mixed without explicit parsing logic.
- Bounced messages are not used as a source of trusted authorization data without context verification.
- The bounce handler does not trust body fields without checking sender, query_id, expected pending state, expected operation, and expected amount/value.

---

## 7. Message sending, send modes, and action phase

- All send sites are listed separately: destination, value, bounce flag, send mode, body, stateInit, deploy behavior, and follow-up assumptions.
- All reserve actions (`RAWRESERVE`, `nativeReserve`, or language equivalents) are listed like send sites: amount, reserve mode, ordering relative to sends, bounce-on-fail behavior, and failure impact.
- There are no magic numbers for flags/modes; named constants are used.
- `mode=64`, `mode=128`, `+1`, `+2`, `+16`, `+32`, and their combinations are checked.
- For outgoing external/log messages, only modes permitted for external messages are used; `+16` is avoided unless explicitly justified, because there is no ordinary sender to receive a bounce.
- `+2` and `+16` interaction is reviewed: `+16` matters only for failures not suppressed by `+2`; `+2` ignores many, but not all, send/action errors.
- `+2` does not mask a critical failure after which state remains inconsistent; non-ignored cases such as malformed messages, invalid StateInit libraries, invalid external-message modes, and invalid mode combinations are tested where relevant.
- `SendRemainingValue` / carry remaining value does not break later logic and does not make subsequent sends meaningless.
- `mode=128 + 32` / account destruction is available only to authorized flows and only when there are no pending obligations.
- Carry-all-balance and account-destruction paths are reviewed as drain paths, not as ordinary sends.
- Action-phase failure is reviewed separately: under ignored/suppressed action errors, compute state may be applied while some outbound messages are skipped; otherwise the transaction may roll back depending on the failure and mode.
- The number of actions does not exceed limits; batch loops are bounded.
- No message send depends on a user-controlled unbounded loop, list, map, or dictionary traversal.
- State after failed action is either still safe or recoverable through a later message/bounce/retry path.
- External outgoing logs/events are not used as a source of on-chain truth.

---

## 8. Gas, fees, storage reserve, freezing/deletion

- Gas is measured for each handler on the worst-case path.
- `msg_value` covers compute fee, forward fee, action fee, storage reserve, deployment fee for child contracts, and excess return.
- Fee accounting includes forward fee, action fee, storage reserve, deploy fee, excesses, refunds, and protocol-specific service fees.
- Storage fees are accounted for; the contract maintains a positive reserve.
- Reservation logic is tested: exact reserve, reserve-all-except, reserve-at-most, insufficient balance, action-list limit, and interaction with later sends.
- Freeze/delete thresholds and storage debt scenarios are checked.
- There is no unbounded storage growth: maps/dicts/lists/history arrays/pending operations have a cap, pruning, pagination, rent mechanism, or sharding.
- Attacker-controlled storage griefing is checked: fake pending operations, dust positions, spam deposits, metadata growth, and history growth.
- Loops have a hard bound; dictionary traversal is limited.
- Loop griefing is checked: attacker-controlled number of sends, iterations, refs, or parsed items cannot lead to denial of service.
- There is no message sending from a user-controlled unbounded loop.
- Excess gas is returned correctly, usually through `excesses` op `0xd53276db` or a protocol-specific equivalent.
- Excesses/refunds return to the correct address and do not break accounting, replay protection, or pending-state logic.
- Returning excesses does not create reentrancy-like / race issues in the async model.
- Fee constants are not hardcoded without tests; recalibration is needed when toolchain/network fees change.

---

## 9. Storage, serialization, cells, and limits

- Storage schema is documented and compatible with current on-chain data.
- Every storage layout upgrade has a migration path and tests on the old state.
- For FunC: the order of `load_*` / `store_*` matches; `end_parse()` is used where needed.
- For Tolk/Tact: auto-serialization is not bypassed with raw cells without a reason.
- Cell size/depth limits are considered: 1023 bits / 4 refs per cell, message/c4/c5 depth limits.
- After parsing, `slice` is fully consumed except for explicitly allowed payload refs.
- Raw cell/slice parsing checks size, remaining bits, refs, type tags, opcodes, and `end_parse()` / assert-end behavior.
- Extra payload tolerance is explicitly justified; disabled `assertEndAfterReading` / equivalent is treated as a red flag until proven safe.
- Incorrect type handling is ruled out: uint written → uint read; address/internal/any address are used consistently.
- Return values from functions that return a success flag are not ignored.
- There is no storage collision, namespace collision, name shadowing, or confusing identifiers.
- Storage schema, wallet code cells, minter code cells, and trusted code cells cannot be silently replaced with incompatible versions.

---

## 10. Arithmetic, types, and rounding

- Signed integers are used only when truly needed.
- All sums/balances/amounts are validated: `amount > 0`, `amount <= balance`, `amount <= max`.
- There is no negative amount spoofing.
- There is no division before multiplication where it creates precision loss.
- Rounding direction is selected and documented: who receives dust.
- Fee/slippage math is checked for min/max, zero liquidity, dust, overflow/underflow, precision loss.
- Sized integer serialization is checked: values do not exceed width.
- For Tolk: arithmetic on sized fields and writing back into `uint32/uint64/coins` is checked explicitly.
- For Tact: `Int as uintX/intX` is checked against the expected range.
- Unsafe casts from untrusted data have explicit range/type proofs.
- Custom error codes do not use 0/1 and do not conflict with reserved ranges.

---

## 11. Randomness, time, validators, commit-reveal

- Randomness is not used for high-value decisions without commit-reveal or an equivalent scheme.
- The contract does not use randomness in external receivers.
- Validator influence is considered: seed choice, message inclusion, ordering.
- For low-value randomness, the language-specific initialization/randomize function is used correctly.
- Time-based logic (`now`, `valid_until`, deadlines) handles clock/ordering edge cases.
- Expiration/cancel flows do not allow funds to be locked forever.
- Commit-reveal schemes include domain separation, deadline, reveal validation, anti-replay, and safe refund/cancel paths.

---

## 12. Upgrades, code/data replacement, and governance

- Every `set_code`, `set_data`, `contract.setCodePostponed`, and `contract.setData` has strict authorization.
- Tact direct `setData()` usage is audited for implicit end-of-receiver state save; manual state replacement branches intentionally terminate and are covered by tests.
- New code hash/code cell is validated, preferably through allowlist/timelock/multisig.
- Upgrade does not break storage layout; migration tests with the old data cell exist.
- Upgrade corruption is checked: new version does not break storage layout, code hash assumptions, wallet code, minter code, trusted code cells, ABI, wrappers, or indexer assumptions.
- Timing is considered: code update is applied after the current execution/action phase; data replacement may be immediate depending on the primitive.
- Governance cannot bypass user rights to withdrawal/claim.
- Emergency pause does not let the owner steal funds or permanently block users.
- Upgrade functions are not accessible through bounced/fallback/empty messages.
- Third-party code / plugin / arbitrary code execution is absent, or external code cells and continuations are strictly validated and tested.
- Admin key compromise scenarios are documented together with blast radius and recovery path.

---

## 13. Tolk-specific checklist

### 13.1 Language and version

- Tolk version and changelog are checked against the used features.
- ABI/source maps/wrappers are used for tests and review, but do not replace manual checking of TVM effects.
- Low-level assembler/Fift/raw cells are used only with justification and tests.
- Generated output visibility is checked: tests rebuild generated wrappers/output files before executing narrow test suites.

### 13.2 Entry points and unions

- `onInternalMessage` uses modern `InMessage`, not legacy manual parsing, unless there is a reason.
- `incomingMessages` union is complete and opcodes are unique.
- `lazy Union.fromSlice(in.body)` does not hide acceptance of an unknown opcode through a permissive `else`.
- Unknown or incomplete payloads do not pass into state-changing logic through a lazy union or permissive fallback branch.
- Empty message is intentionally ignored/cashbacked.
- Bounced messages are handled in `onBouncedMessage` and are not mixed with internal messages.

### 13.3 Lazy loading

- Security-critical lazy fields are actually read before the check.
- There is no “lazy validation bypass”: a field exists in a struct but is not accessed, so validation/deserialization never happens.
- Delayed validation does not allow state write / accept / send before the critical field has been read and validated.
- Partial update through lazy does not break invariants and does not overwrite fields incorrectly.
- `assertEndAfterReading` / equivalent is not disabled without a reason.

### 13.4 Types and casts

- `address` is used for internal addresses; `any_address` is used only when external/none addresses are truly allowed.
- External/none addresses do not pass where an internal address is expected.
- There are no unsafe `as` casts from untrusted `slice/cell/builder`.
- Nullable values are not force-unwrapped (`!`) without a prior null check.
- `null` is not used as a valid privileged state without a clear invariant.
- `unknown`, `builder`, `slice`, and `cell` are not used to bypass type safety or hide untrusted raw payload.
- `Cell<T>` / typed cells match the expected TL-B layout.

### 13.5 Message construction

- `createMessage()` is used instead of raw building unless there is a proxy/raw reason.
- Compiler-managed serialization is used where raw body layout is not required.
- `body: obj.toCell()` is not passed when the compiler should be allowed to choose inline/ref; `body: obj` is passed instead.
- `UnsafeBodyNoRef` is used only after a size proof.
- `sendRawMessage` is allowed only for validated raw/proxy messages.
- `AutoDeployAddress` / StateInit: code+data match the expected address, shard targeting is justified.
- StateInit, deploy address, wallet code, owner address, workchain, and salt/domain separation are checked.

### 13.6 BounceMode

- For stateful sends, a bounce mode with enough recovery data is selected.
- `RichBounce` is used where full original body / exitCode / gasUsed is needed.
- `Only256BitsOfBody` / root-cell-only behavior is sufficient for the parser and recovery logic.
- `NoBounce` is used only when loss of the downstream message is safe.
- Bounce parser compatibility is tested for selected bounce mode and malformed/truncated body.

### 13.7 Commit and state

- `commitContractDataAndActions()` is not called before validation is complete.
- Its placement is checked for replay protection and action failure.
- Committed state cannot become permanently wrong if an outbound action later fails.
- Global variables are not used as persistent state.

---

## 14. FunC-specific checklist

- All state-changing / send / random / `set_code` / `set_data` functions have `impure`.
- There is no confusion between modifying (`~`) and non-modifying (`.`) methods.
- Storage is manually parsed/packed in the same order everywhere.
- `end_parse()` is used for payload/storage where applicable.
- Redeclaration/shadowing does not mask storage variables or return values.
- `accept_message()` is placed only after cheap validation.
- `send_raw_message` modes are documented and tested.
- `set_code`/`set_data` are gated and tested; data replacement is reviewed for immediate-vs-postponed effects and migration safety.
- Third-party code execution is absent or code is validated; the `COMMIT` + out-of-gas scenario is analyzed.
- Manual raw-cell parsing has tests for malformed slices, missing refs, extra refs, oversized payloads, and trailing garbage.

---

## 15. Tact-specific checklist

- For Tact projects: although TON Docs mark Tact as deprecated in favor of Tolk, existing Tact contracts must still be audited against the current Tact compiler, generated wrappers, and dedicated Tact documentation.
- Receiver coverage: `receive`, `external`, `bounced`, fallback.
- Bounced payload limit is considered; important fields are first or a fallback Slice parser is used.
- Traits do not allow bypassing auth/pausable/stoppable invariants.
- Optional values are used intentionally; there are no unsafe nullable patterns.
- Native/asm functions are audited manually.
- Direct `setData()` usage is treated as dangerous: every branch using manual state replacement prevents Tact's implicit final state save from overwriting or corrupting the intended data.
- `myBalance()` / context balance assumptions are not used after sends as the actual live balance.
- After sends, stale assumptions about contract balance are not used for later accounting decisions.
- Exit codes, default codes, and generated wrappers are checked.
- Gas best practices do not conflict with security best practices.

---

## 16. Jetton / NFT / DEX / bridge-specific checks

### Jetton

- Jetton wallet authenticity is checked by recomputing the address from master/wallet code/owner.
- Sender of transfer notification is checked.
- Fake Jetton deposit is checked: payload may look correct, but sender must be the expected Jetton wallet for master+owner.
- Untrusted notification is checked: `transfer_notification` / callback must come from the expected wallet/contract.
- Minter mint flow handles bounce and corrects totalSupply.
- Wallet transfer handles bounce and restores balance.
- Burn flow handles notification/bounce/failure and keeps totalSupply and wallet balance consistent.
- Forward payload/ref constraints are checked.
- TEP-74 message formats/opcodes/query_id are followed.

### NFT

- Ownership transfer authorization is checked.
- Collection/item address derivation is checked.
- Fake NFT transfer is checked: item address must be derived from collection/index/code and not accepted only from payload claims.
- Mint indexing cannot be race-attacked or skipped.
- Metadata mutability/admin controls are checked.
- Random traits use a safe randomness model.

### DEX/vault/lending/bridge

- False deposit/top-up attack is checked: inbound transfer + outbound refund/bounce correlation.
- Slippage/min_out/deadline are applied on every path.
- Fee collection happens on every path.
- LP supply math is symmetric for add/remove liquidity.
- Oracle/indexer assumptions are documented; stale price is handled.
- Bridge messages include domain separation, source chain, destination, nonce, amount, token id.
- Pending claims cannot be double-spent through parallel traces.
- DEX/vault/lending race scenarios are tested: slippage, deadline, min_out, fee, liquidity, reserves, debt, and liquidation state are preserved under parallel traces.
- Refund/excess logic cannot be used to fake a deposit, cancel accounting, or bypass slippage/deadline checks.

---

## 17. Testing, fuzzing, and static analysis

- Unit tests cover all handlers, including empty/unknown/bounced/fallback.
- Integration tests cover full multi-contract traces.
- There are explicit tests for action-phase failure, bounce, insufficient funds, invalid destination, oversized message, malformed cell, wrong sender, and wrong workchain.
- Race-condition simulations: two flows interleaved in different orders.
- Replay tests: same external message twice, old seqno, expired valid_until, wrong subwallet, wrong contract address, wrong op, and wrong workchain.
- Gas snapshots exist for each handler; worst-case path is tested.
- Storage migration tests run with the previous deployed data layout.
- Fuzzing/mutation tests cover message bodies, amounts, opcodes, slices, refs, map sizes, lazy fields, and send modes.
- Tools are considered: Misti, TON Symbolic Analyzer, tolk-less, Universalmutator, custom linters.
- Tool output is reviewed manually; coverage is checked, but is not treated as proof of security.
- For each high-impact scenario, tests include success path, failure path, bounce path, insufficient funds path, replay path, and duplicate/late message path where relevant.
- The audit report marks which items were checked by code review, which by tests, which by tooling, and which only by manual reasoning.

---

## 18. High-signal red flags

Immediate manual review is needed if any of the following are found:

- `accept_message()` / `acceptExternalMessage()` is called before signature/seqno/valid_until checks.
- `mode=128`, `mode=128+32`, or carry-all-balance is used outside strict authorization.
- A bounce handler is missing for a stateful outgoing message.
- State is updated before outbound send and there is no recovery on bounce/action failure.
- Unknown opcode is silently accepted.
- A state-changing handler has no explicit caller/identity policy.
- Sender identity is taken from payload/forwarded address instead of the actual verified sender contract.
- Jetton deposit trusts sender without recomputing the expected wallet address.
- NFT transfer trusts payload without deriving the expected item address.
- `set_code` / `set_data` / `contract.setCodePostponed` are used without multisig/timelock/allowlist.
- Raw `sendRawMessage` / raw cell parsing is used without tests.
- Unsafe `as` cast or nullable force unwrap is applied to untrusted data.
- `unknown`, `slice`, `cell`, or `builder` is used to bypass type safety without strict parsing.
- Unbounded loop/storage growth.
- Randomness is used for a high-value outcome without commit-reveal.
- Ignored/suppressed action-phase failure can leave state inconsistent, or the code incorrectly assumes that action failure always rolls back state.
- `assertEndAfterReading` is disabled / extra payload is accepted without a reason.
- A privileged operation can be reached through fallback, bounce, empty message, upgrade, or plugin-like path.
- Tests use generated wrappers/output files without rebuilding them after source changes.

---

## 19. Evidence to collect in the audit report

- Message-flow diagrams.
- Value-flow diagrams.
- Complete call/message graph.
- Storage layout and migration notes.
- List of trusted roles and admin powers.
- Send-site table: dest/value/mode/bounce/body/stateInit.
- Entry-point table: caller/auth/state changes/outgoing messages/gas/bounce/action-failure assumptions.
- Gas table for each handler.
- Test matrix and missing tests.
- Exploit scenarios or “not applicable” rationale for high-impact checklist areas.
- Known accepted risks and rationale.
- Classification of evidence: checked by code review, checked by tests, checked by tooling, or checked only by manual reasoning.

---

## 20. Precision notes for reviewers

- `+2` / `SendIgnoreErrors` ignores many send/action errors but does not make every malformed or invalid action safe. The non-ignored cases are part of the test matrix.
- `+16` / bounce-on-action-failure is meaningful only for failures that are not suppressed by `+2`; do not describe `+2 +16` as two independent protections.
- Action-phase failure is not a single outcome: distinguish rollback, skipped action, ignored error, bounce, and successful later actions.
- `query_id` is a correlation key, not an authorization primitive. Its uniqueness requirement is scoped to the relevant pending/in-flight operation unless a standard or protocol requires more.
- Permissionless state-changing handlers are allowed, but only when their caller/identity policy is explicit and all sender-derived assumptions are verified.
- Tact `setData()` is exceptional and dangerous; audit it together with Tact's implicit receiver-end state save.
- Reserve actions are first-class action-phase effects and must be reviewed like sends.

## Sources to keep next to this checklist

### Official TON and language documentation

- TON Security Best Practices: https://docs.ton.org/blockchain-basics/contract-dev/techniques/security
- TON Core Concepts: https://docs.ton.org/blockchain-basics/core-concepts
- TON Internal Messages: https://docs.ton.org/blockchain-basics/primitives/messages/internal
- TON Sending modes: https://docs.ton.org/foundations/messages/modes
- Tolk message handling and BounceMode: https://docs.ton.org/tolk/features/message-handling
- Tact context/state, `setData`, and `nativeReserve`: https://docs.tact-lang.org/ref/core-contextstate/
- Tact bounced messages: https://docs.tact-lang.org/book/bounced/
- Tolk changelog: https://docs.ton.org/blockchain-basics/tolk/changelog
- Tolk overview: https://docs.ton.org/blockchain-basics/tolk/overview
- Tact documentation: https://docs.tact-lang.org/

### Standards

- TEP-74 Jettons Standard: https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md
- TEP-89 Jetton Wallet Discovery: https://github.com/ton-blockchain/TEPs/blob/master/text/0089-jetton-wallet-discovery.md
- TEP-62 NFT Standard: https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md
- TEP-64 Token Data Standard: https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md
- TEP-66 NFT Royalty Standard: https://github.com/ton-blockchain/TEPs/blob/master/text/0066-nft-royalty-standard.md

### Community / research

- Polaristow awesome-ton-security: https://github.com/Polaristow/awesome-ton-security
- Trail of Bits “Not So Smart Contracts — TON”: https://github.com/crytic/building-secure-contracts/tree/master/not-so-smart-contracts/ton
- Zellic TON Security Primer: https://www.zellic.io/blog/ton-security-primer/
- Positive Security blog on TON audits: https://blog.positive.com/security-audit-of-smart-contracts-in-ton-key-mistakes-and-tips
- arXiv “From Paradigm Shift to Audit Rift”: https://arxiv.org/abs/2509.10823
- Sanbir TON Auditor Skills: https://github.com/sanbir/ton-auditor-skills
