# Ethereum Beacon API: Ambiguities and Underspecified Behavior

A systematic audit of the [beacon-APIs](https://github.com/ethereum/beacon-apis) OpenAPI specification, identifying areas where the spec is ambiguous, underspecified, or inconsistent. Organized by category with file references.

---

## Table of Contents

1. [Identifier Parameters (StateId, BlockId)](#1-identifier-parameters-stateid-blockid)
2. [Optional Parameter Omission Behavior](#2-optional-parameter-omission-behavior)
3. [Empty Array Semantics](#3-empty-array-semantics)
4. [Error Responses and Status Codes](#4-error-responses-and-status-codes)
5. [Version Enum Inconsistencies](#5-version-enum-inconsistencies)
6. [Fork-Dependent Behavior](#6-fork-dependent-behavior)
7. [Content Negotiation (JSON vs SSZ)](#7-content-negotiation-json-vs-ssz)
8. [Response Envelope Inconsistencies](#8-response-envelope-inconsistencies)
9. [Validator Duty Endpoints](#9-validator-duty-endpoints)
10. [Pool Endpoints](#10-pool-endpoints)
11. [Block Production and Execution Payload](#11-block-production-and-execution-payload)
12. [Event Stream](#12-event-stream)
13. [Rewards Endpoints](#13-rewards-endpoints)
14. [Node and Config Endpoints](#14-node-and-config-endpoints)
15. [Type-Level Issues](#15-type-level-issues)
16. [Schema Keyword Inconsistencies](#16-schema-keyword-inconsistencies)

---

## 1. Identifier Parameters (StateId, BlockId)

### 1.1 `BlockId` as slot number for a skipped slot -- behavior undefined

**File:** `params/index.yaml:11-20`

The `BlockId` parameter accepts `<slot>` but no block exists at a skipped slot. The spec does not define whether:
- A 404 should be returned
- The node should return the block at the most recent non-empty slot
- Behavior differs per endpoint

This affects every endpoint taking `block_id`: `getBlockHeader`, `getBlockRoot`, `getBlockV2`, `getBlindedBlock`, `getBlockAttestationsV2`, `getBlobSidecars`, `getBlobs`, and all reward endpoints.

### 1.2 Slot with multiple blocks (reorg/fork)

**File:** `params/index.yaml:11-20`

When `block_id` is a slot number during a reorg, the spec does not define whether the canonical block is returned or how ties are broken. For `getBlockHeaders` (plural), the `slot` parameter could return multiple blocks. For `getBlockHeader` (singular), only one can be returned but the selection criteria is unspecified.

### 1.3 `"head"` during an active reorg

**File:** `params/index.yaml:11-20`

The "head" value is "canonical head in node's view." The spec does not address TOCTOU issues -- the head may change between request receipt and response generation. No atomicity guarantees are recommended.

### 1.4 `"finalized"` resolves to which block?

It is not stated whether `block_id = "finalized"` returns the block at the finalized **checkpoint slot** or the most recent block on the finalized chain. The `finalized` response field will always be `true` in this case, which is redundant but not acknowledged.

### 1.5 `StateId` supports `"justified"` but `BlockId` does not

**File:** `params/index.yaml:10,19-20`

`StateId` lists `"justified"` as a valid value. `BlockId` does not. No explanation is given for this asymmetry.

### 1.6 `StateId` for a skipped slot -- behavior undefined

**File:** `params/index.yaml:1-10`

Like `BlockId`, using a slot number for a skipped slot as a `state_id` is not specified. It is unclear whether the state at the end of the skipped slot (which does exist) should be returned or if this is treated as an error.

### 1.7 No schema-level validation

**File:** `params/index.yaml:6-7`

Both `StateId` and `BlockId` are typed as bare `type: string` with no pattern constraint. Any string passes schema validation, including empty strings, negative numbers, or arbitrary text.

### 1.8 Unknown root: 400 vs 404 ambiguity

When a hex-encoded root is syntactically valid but does not match any known state/block, the spec does not clarify whether the response should be 400 (invalid input) or 404 (not found).

---

## 2. Optional Parameter Omission Behavior

### 2.1 `getBlockHeaders` -- both parameters omitted

**File:** `apis/beacon/blocks/headers.yaml:4-17`

The description says "By default it will fetch current head slot blocks." Both `slot` and `parent_root` are optional. The spec does not define:
- What "current head slot blocks" means when the head slot was skipped
- Whether non-canonical blocks at the head slot are included
- The maximum number of results (no pagination)

### 2.2 `getBlockHeaders` -- both parameters supplied simultaneously

**File:** `apis/beacon/blocks/headers.yaml:8-17`

Both `slot` and `parent_root` can be supplied. The spec does not say whether they are an AND filter, an OR filter, or whether supplying both is an error.

### 2.3 `getPoolAttestationsV2` -- both filters omitted

**File:** `apis/beacon/pool/attestations.v2.yaml:5-15`

When both `slot` and `committee_index` are omitted, the endpoint returns "attestations known by the node." There is no cap, pagination, or ordering guarantee.

### 2.4 `getHealth` -- `syncing_status` conflicts

**File:** `apis/node/health.yaml:8-16`

The `syncing_status` parameter allows any integer 100-599 to customize the syncing response code. There is no specification for what happens if the client supplies a code that conflicts with other defined responses (e.g., `syncing_status=200` or `syncing_status=503`). The response body for the custom status code is also undefined.

### 2.5 `getLightClientUpdatesByRange` -- `count=0`

**File:** `apis/beacon/light_client/updates.yaml:11-21`

Both `start_period` and `count` are `Uint64` with no minimum constraint. A `count=0` creates a contradictory empty range against the stated requirement to return at least the earliest known result.

### 2.6 Committee epoch range unspecified

**File:** `apis/beacon/states/committee.yaml:11-16`

The `epoch` parameter for committees says "If not present then the committees for the epoch of the state will be obtained." But valid epoch ranges relative to the state are not specified. Can a client request committees for an epoch far in the past or future relative to the state?

### 2.7 `getPoolPayloadAttestations` -- slot omitted behavior not explicit

**File:** `apis/beacon/pool/payload_attestations.yaml:5-10`

When the `slot` parameter is omitted, the endpoint presumably returns all payload attestations in the pool. This is not explicitly stated.

### 2.8 `getPeers` -- empty/absent filter array semantics

**File:** `apis/node/peers.yaml:8-23`

The `state` and `direction` query parameters are optional arrays. The spec does not clarify:
- Whether an empty array `?state=` is equivalent to "all" or "none"
- What happens if an invalid enum value is mixed with valid ones (e.g., `?state=connected&state=bogus`)
- No 400 response is defined for this endpoint, so there is no specified behavior for malformed filter values

### 2.9 `getDebugDataColumnSidecars` -- empty `indices` array

**File:** `apis/debug/data_column_sidecars.yaml:17-25`

The `indices` parameter is optional. Omitting it returns all columns the node custodies. If `indices=` (empty array) is sent, does that mean "return nothing" or "return all"?

---

## 3. Empty Array Semantics

### 3.1 POST endpoints with empty arrays -- no defined behavior

**Files:**
- `apis/beacon/pool/attestations.v2.yaml` (POST)
- `apis/beacon/pool/sync_committees.yaml` (POST)
- `apis/beacon/pool/bls_to_execution_changes.yaml` (POST)
- `apis/beacon/pool/payload_attestations.yaml` (POST)
- `apis/validator/beacon_committee_subscriptions.yaml`
- `apis/validator/sync_committee_subscriptions.yaml`
- `apis/validator/aggregate_and_proofs.v2.yaml`
- `apis/validator/prepare_beacon_proposer.yaml`
- `apis/validator/register_validator.yaml`

These endpoints accept arrays in request bodies but do not specify what happens if an empty array `[]` is submitted. Should the server return 200 (vacuous success) or 400?

### 3.2 Inconsistent `minItems` constraints

Some POST endpoints require `minItems: 1` while many others are silent:

**Has `minItems: 1`:**
- `apis/validator/duties/attester.yaml:39`
- `apis/validator/duties/sync.yaml:24`
- `apis/validator/duties/ptc.yaml:38`
- `apis/validator/liveness.yaml:30`

**No `minItems` (ambiguous):**
- All subscription, selection, aggregate, proposer prep, and registration endpoints

### 3.3 GET endpoints returning empty `data` arrays

**Files:** All pool GET endpoints, `getBlockHeaders`

None specify whether an empty pool/result returns 200 with `data: []` or 404. The convention of 200 with empty array is implied but never stated.

### 3.4 Empty result metadata ambiguity

**File:** `apis/beacon/blocks/headers.yaml:19-44`

When `data` is an empty array, the required `execution_optimistic` and `finalized` booleans have no meaningful value. The spec does not define what they should be set to when there is no data to characterize.

### 3.5 `required: true` body with no required properties

**File:** `apis/beacon/states/validators.yaml:107-113`

The POST variant of `getStateValidators` has `required: true` on the request body but `required: []` on the schema object. This means a client must send a body but the body can be `{}` with no properties, creating a confusing contract.

### 3.6 POST body `required` inconsistency across state endpoints

**Files:** `apis/beacon/states/validators.yaml:108`, `apis/beacon/states/validator_balances.yaml:97`, `apis/beacon/states/validator_identities.yaml:23`

The POST variant of `getStateValidators` has `required: true` on the request body, while the POST variants of `getStateValidatorBalances` and `getStateValidatorIdentities` have `required: false`. These three endpoints all accept validator ID lists but disagree on whether the body must be present.

### 3.7 `getPeers` -- zero peers matching filters

**File:** `apis/node/peers.yaml:24-46`

If no peers match the filter, the spec does not state whether `data` should be `[]` with `meta.count: 0` or a 404. No 404 is defined, implying an empty array is correct, but this is only implicit.

### 3.8 `getForkSchedule` -- no minimum forks guarantee

**File:** `apis/config/fork_schedule.yaml:16-20`

The `data` array of Fork objects has no `minItems: 1`. While there should always be at least one fork (phase0 genesis), this is not schema-enforced.

### 3.9 `getDebugChainHeadsV2` -- empty chain heads

**File:** `apis/debug/heads.v2.yaml:16-28`

The `data` array of chain heads has no `minItems`. During initialization, a node may have no chain head. No error response other than 500 is defined for this case.

### 3.10 `minItems` inconsistency between debug endpoints

**File:** `apis/debug/fork_choice.yaml:25`

`getDebugForkChoice` specifies `minItems: 1` on `fork_choice_nodes`, but `getDebugChainHeadsV2` has no `minItems` on its `data` array. Both represent aspects of fork choice state and should be consistent.

### 3.11 Block attestations with zero attestations -- version metadata

**File:** `apis/beacon/blocks/attestations.v2.yaml:28-41`

The genesis block and potentially other blocks may contain zero attestations. The response will have `data: []` but the `version` field is still required. It is not stated whether `version` should reflect the fork of the block.

---

## 4. Error Responses and Status Codes

### 4.1 `getBlockHeaders` 400 error references non-existent `block_id` parameter (bug)

**File:** `apis/beacon/blocks/headers.yaml:45-53`

The 400 error says "The block ID supplied could not be parsed" with example `"Invalid block ID: current"`. This endpoint has no `block_id` parameter -- it has `slot` and `parent_root`. This is a copy-paste error.

### 4.2 Missing 404 on `getBlockHeaders`

**File:** `apis/beacon/blocks/headers.yaml`

Unlike every other block GET endpoint, `getBlockHeaders` does not define a 404. If a `slot` is specified for a future slot, the expected response is ambiguous.

### 4.3 `publishBlockV2` / `publishBlindedBlockV2` -- 202 response has no body

**Files:** `apis/beacon/blocks/blocks.v2.yaml:67-68`, `apis/beacon/blocks/blinded_blocks.v2.yaml:65-66`

The 202 response ("broadcast but failed integration") has no body schema. Callers cannot learn *why* integration failed. The 400 response does include an `ErrorMessage`, creating an asymmetry.

### 4.4 `Eth-Consensus-Version` header mismatch with body -- no error specified

**Files:** All POST endpoints requiring `Eth-Consensus-Version`

No endpoint specifies what happens when the header value does not match the body content. This is critical for SSZ payloads where the header is the only way to determine the type.

### 4.5 `InternalError` and `CurrentlySyncing` schema examples show wrong codes

**File:** `types/http.yaml:29-30,77-78`

`InternalError` has `code` property `example: 404` (should be 500). `CurrentlySyncing` has `example: 404` (should be 503). These misleading schema-level examples could cause implementation errors.

### 4.6 Missing 503 (CurrentlySyncing) across most endpoints

Despite a `CurrentlySyncing` response type existing in `types/http.yaml`, it is absent from:
- All state query endpoints
- All block GET endpoints
- All pool endpoints (except block publish)
- All rewards endpoints
- All node/config/debug endpoints
- All light client endpoints
- `execution_payload_bid`

Only validator duty endpoints and block publish consistently include 503.

### 4.7 Overloaded 503 semantics (syncing vs. optimistic)

**Files:** `apis/validator/attestation_data.yaml`, `apis/validator/aggregate_attestation.v2.yaml`, `apis/validator/sync_committee_contribution.yaml`

Multiple endpoints use 503 for both "node is syncing" AND "response references optimistic execution payload." Clients cannot distinguish these fundamentally different conditions. The `aggregate_attestation.v2.yaml` describes 503 behavior in its description text but does not list it in the `responses` section.

### 4.8 Missing 415 on POST endpoints

POST endpoints accepting only JSON do not consistently specify 415 (Unsupported Media Type). Meanwhile, POST endpoints accepting SSZ do specify it, creating an inconsistency about what happens when wrong content types are sent to JSON-only endpoints.

### 4.9 Inconsistent 404 descriptions across rewards endpoints

- `apis/beacon/rewards/attestations.yaml:79`: "Epoch not known or required data not available"
- `apis/beacon/rewards/blocks.yaml:44`: "Block or required state not found"
- `apis/beacon/rewards/sync_committee.yaml:53`: "Block not found"

The sync committee 404 does not mention "required state" unavailability, unlike block rewards.

### 4.10 `getHealth` -- no 500 response

**File:** `apis/node/health.yaml`

This is the only endpoint that does not define a 500 (Internal Error) response.

### 4.11 `getPeers` -- no 400 response

**File:** `apis/node/peers.yaml`

Despite accepting query parameters that could be malformed, no 400 is defined. `getPeer` (single) does define 400.

### 4.12 Single-validator endpoint -- dual 404 ambiguity

**File:** `apis/beacon/states/validator.yaml:44-58`

The 404 response has two named examples: `StateNotFound` and `ValidatorNotFound`. Both use the same 404 code, so clients cannot programmatically distinguish "the state does not exist" from "the validator does not exist within this state." This differs from the plural `validators.yaml`, where unknown validators produce empty data (no error) and 404 means only "state not found."

### 4.13 Pre-Electra/Fulu state queries -- "should return 400" is normatively imprecise

**Files:** `apis/beacon/states/pending_consolidations.yaml:5`, `apis/beacon/states/pending_deposits.yaml:5`, `apis/beacon/states/pending_partial_withdrawals.yaml:5`, `apis/beacon/states/proposer_lookahead.yaml:5`

These descriptions say "Should return 400 if the state retrieved is prior to Electra" (or Fulu). However:
- "Should" is not "must" (RFC 2119 semantics). Is this mandatory or recommended?
- The 400 `InvalidRequest` description says "Invalid request syntax," which does not accurately describe a valid request against an inapplicable fork
- No specific error message example is given for this case

### 4.14 RANDAO epoch out of range -- 400 vs 404 ambiguity

**File:** `apis/beacon/states/randao.yaml:46-54`

The 400 example says "Epoch is out of range for the `randao_mixes` of the state." Whether an out-of-range epoch is an invalid request (400) or a not-found condition (404) is debatable. A future epoch with no RANDAO yet could be argued either way.

### 4.15 Sync committee state endpoint -- period boundary undefined

**File:** `apis/beacon/states/sync_committees.yaml:35-43`

The 400 example says "Epoch is outside the sync committee period of the state." The spec does not define what "outside the sync committee period" precisely means. Sync committees rotate every 256 epochs; it is unclear whether adjacent periods can be queried.

### 4.16 Two parallel error schema definition styles

Older state endpoints use inline error schemas with `$ref` to `ErrorMessage` (e.g., `validators.yaml:62-63`, `committee.yaml:51-52`). Newer endpoints use `$ref` to shared response objects (`InvalidRequest`, `NotFound` in `types/http.yaml`). The shared responses define structurally identical but separate, non-`$ref`-linked schemas that could diverge from `ErrorMessage`.

### 4.17 `publishBlockV2` / `publishBlindedBlockV2` -- 200 response also has no body

**Files:** `apis/beacon/blocks/blocks.v2.yaml:66`, `apis/beacon/blocks/blinded_blocks.v2.yaml:64`

The 200 (success) response also has no body schema, only a description. While consistent with other POST endpoints, it is never explicitly documented as "empty body."

### 4.18 Missing 406 on `getPoolPayloadAttestations` GET (supports SSZ)

**File:** `apis/beacon/pool/payload_attestations.yaml`

This GET endpoint supports both JSON and SSZ responses but does not list a 406 response. Other endpoints supporting SSZ (`getBlobSidecars`, `getBlockV2`, `getBlindedBlock`, `getBlobs`) all list 406.

### 4.19 Missing 404 on attester and proposer duty endpoints

**Files:** `apis/validator/duties/attester.yaml`, `apis/validator/duties/proposer.yaml`, `apis/validator/duties/proposer.v2.yaml`

These endpoints do not define a 404 response. In contrast, `aggregate_attestation.v2.yaml`, `sync_committee_contribution.yaml`, and `execution_payload_bid.yaml` do. It is unclear what response should be given if an epoch is valid but too far in the future for the node to compute.

### 4.20 `getNetworkIdentity` -- minimal error coverage

**File:** `apis/node/identity.yaml`

Only 200 and 500 are defined. There is no mention of what happens if the node is still initializing and cannot determine its identity (e.g., no ENR generated yet). Compare with `getHealth` which has a 503 for "not initialized."

### 4.21 `publishExecutionPayloadBid` -- missing 406 and 503

**File:** `apis/beacon/execution_payload/bid.yaml`

The endpoint accepts `application/octet-stream` but has no 406 (Not Acceptable) response. It also has no 503 for when the node is syncing, and no specification for what happens if the `Eth-Consensus-Version` header does not match the content.

---

## 5. Version Enum Inconsistencies

### 5.1 `gloas` missing from most endpoint version enums

| Endpoint | Version Enum | `gloas` present? |
|---|---|---|
| `getBlockV2` | `[phase0..gloas]` | Yes |
| `getBlindedBlock` | `[phase0..fulu]` | **No** |
| `getBlockAttestationsV2` | `[phase0..fulu]` | **No** |
| `getPoolAttestationsV2` | `[phase0..fulu]` | **No** |
| `getPoolAttesterSlashingsV2` | `[phase0..fulu]` | **No** |
| `getStateV2` | `[phase0..fulu]` | **No** |
| `produceBlockV3` | `[phase0..fulu]` | **No** |
| `ConsensusVersion` (global) | `[phase0..gloas]` | Yes |

If `gloas` blocks exist, these endpoints will return a `version` value not in their own declared enum.

### 5.2 `getBlindedBlock` -- version enum includes `fulu` but no Fulu data schema

**File:** `apis/beacon/blocks/blinded_block.yaml:29,36-42`

The version enum includes `fulu` but the `data` field's `anyOf` only lists schemas through `Electra.SignedBlindedBeaconBlock`. No `Fulu.SignedBlindedBeaconBlock` exists.

### 5.3 `getBlobSidecars` -- missing `fulu` from version enum

**File:** `apis/beacon/blob_sidecars/blob_sidecars.yaml:42`

Lists only `[deneb, electra]`. This may be intentional (deprecated, replaced by data columns in fulu) but is not documented.

### 5.4 Light client endpoints use full `ConsensusVersion` including `phase0`

Light client data only exists from Altair onward, but referencing the full `ConsensusVersion` enum means `phase0` is listed as valid despite never being applicable.

### 5.5 Narrow version enums on debug and execution payload endpoints

**Files:** `apis/debug/data_column_sidecars.yaml:41`, `apis/beacon/execution_payload/envelope_get.yaml:29`

`getDebugDataColumnSidecars` has version enum `[fulu]` only. `getSignedExecutionPayloadEnvelope` has enum `[gloas]` only. These per-endpoint narrowed enums duplicate and potentially diverge from the canonical `ConsensusVersion`. Neither the table in 5.1 nor these endpoints document why a restricted enum was chosen.

---

## 6. Fork-Dependent Behavior

### 6.1 `getBlobSidecars` / `getBlobs` for pre-Deneb blocks

**Files:** `apis/beacon/blob_sidecars/blob_sidecars.yaml`, `apis/beacon/blobs/blobs.yaml`

Blobs did not exist before Deneb. If `block_id` refers to a pre-Deneb block, the spec does not say whether to return 200 with `data: []`, 404, or 400.

### 6.2 `getBlobSidecars` / `getBlobs` with pruned data

Beacon nodes may prune blob data after the data availability window. The response for a valid `block_id` whose blobs have been pruned is unspecified.

### 6.3 Light client endpoints on a phase0-only chain

**Files:** `apis/beacon/light_client/*.yaml`

No specification for what these endpoints return before the Altair fork. Should they return 404? 501?

### 6.4 `publishBlindedBlockV2` for pre-Bellatrix

**File:** `apis/beacon/blocks/blinded_blocks.v2.yaml:14-16`

States "Before Bellatrix, this endpoint will accept a SignedBeaconBlock" but does not specify what happens if a pre-Bellatrix block is submitted to a post-Bellatrix node.

### 6.5 `payload_attestation_data` for pre-Gloas slots

**File:** `apis/validator/payload_attestation_data.yaml`

The version enum is `[gloas]`. No behavior is defined for pre-Gloas slots.

### 6.6 `committee_index` deprecation for Electra is vague

**File:** `apis/validator/attestation_data.yaml:25-34`

The parameter uses "MAY always be set to 0" for Electra and "MAY be omitted" for Gloas. Whether the node ignores non-zero values for Electra+ is unclear.

### 6.7 `getBlobSidecars` deprecated without successor guidance

**File:** `apis/beacon/blob_sidecars/blob_sidecars.yaml:4`

Marked `deprecated: true` but no replacement endpoint is indicated. Presumably `getBlobs` replaces it, but this is not stated.

### 6.8 `publishBlockV2` mentions Gloas but `publishBlindedBlockV2` does not

**File:** `apis/beacon/blocks/blocks.v2.yaml:14-17`

`publishBlockV2` documents Gloas behavior (blobs broadcast via `ExecutionPayloadEnvelope`). `publishBlindedBlockV2` has no Gloas mention and it is unclear whether it is applicable to Gloas at all.

### 6.9 Finality checkpoints -- pre-finality behavior uses "should" not "must"

**File:** `apis/beacon/states/finality_checkpoints.yaml:4-6`

The description says "In case finality is not yet achieved, checkpoint should return epoch 0 and ZERO_HASH as root." The word "should" is not "must." It is also unclear whether this applies to all three checkpoints (`previous_justified`, `current_justified`, `finalized`) or only `finalized`.

### 6.10 Proposer duties v1 deprecated without migration timeline

**File:** `apis/validator/duties/proposer.yaml:6`

Marked `deprecated: true` but no guidance on when v1 will be removed or how clients should migrate to v2. This is the only duty endpoint carrying a deprecation marker.

---

## 7. Content Negotiation (JSON vs SSZ)

### 7.1 SSZ responses lack type mapping documentation

**Files:** `apis/beacon/blocks/block.v2.yaml:44-46`, `apis/beacon/blocks/blinded_block.yaml:43-45`

SSZ response descriptions say "SSZ serialized block bytes" without specifying which container type is serialized per fork. For Deneb+, the JSON payload is `SignedBlockContents` (with blobs), not just `SignedBeaconBlock`. Clients cannot determine the correct SSZ type to deserialize without out-of-band knowledge.

### 7.2 SSZ request type unspecified per fork

**File:** `apis/beacon/blocks/blocks.v2.yaml:61-63`

The SSZ request body says "SSZ serialized block bytes" but the JSON schema shows different types per fork (`SignedBlockContents` vs `SignedBeaconBlock`). The mapping from `Eth-Consensus-Version` header to SSZ container type is not documented.

### 7.3 SSZ responses: `data` field only vs envelope -- never stated

All endpoints supporting SSZ are ambiguous about whether the SSZ response is the `data` field contents only or the entire response envelope. The convention that SSZ covers only `data` is never explicitly stated.

### 7.4 SSZ header name inconsistency in block production

**File:** `apis/validator/block.v3.yaml:116-118`

The SSZ description references `Eth-Blinded-Payload` but the JSON response uses `Eth-Execution-Payload-Blinded`. These are different header names for the same concept.

### 7.5 Inconsistent SSZ support across pool endpoints

| Endpoint | SSZ Support |
|---|---|
| `submitPoolAttestationsV2` | Yes |
| `submitPayloadAttestationMessages` | Yes |
| `submitPoolAttesterSlashingsV2` | No |
| `submitPoolSyncCommitteeSignatures` | No |
| `submitPoolProposerSlashings` | No |
| `submitPoolVoluntaryExit` | No |

The rationale for which endpoints support SSZ is not given.

### 7.6 `getPoolPayloadAttestations` is the only pool GET with SSZ support

**File:** `apis/beacon/pool/payload_attestations.yaml`

No other pool GET endpoint supports SSZ responses. The reason is undocumented.

### 7.7 `getBlobs` SSZ type vs JSON wrapper

**File:** `apis/beacon/blobs/blobs.yaml:45-47`

The SSZ description says `List[Blob, MAX_BLOB_COMMITMENTS_PER_BLOCK]` but the JSON response wraps this in an envelope with `execution_optimistic`, `finalized`, and `data`. The SSZ format is ambiguous about whether these metadata fields are included.

---

## 8. Response Envelope Inconsistencies

### 8.1 `execution_optimistic` and `finalized` -- present vs absent

Present on all block GET endpoints and state query endpoints. Absent on all pool GET endpoints. The absence on pool endpoints is likely intentional but never documented.

### 8.2 `Eth-Consensus-Version` header -- present vs absent

| Endpoint | Header in Response |
|---|---|
| `getBlockV2` | Yes |
| `getBlindedBlock` | Yes |
| `getBlockAttestationsV2` | Yes |
| `getBlobSidecars` | Yes |
| `getBlobs` | **No** |
| `getBlockHeaders` | **No** |
| `getBlockHeader` | **No** |
| `getBlockRoot` | **No** |

`getBlobs` has no version header or body field despite blob structures potentially varying across forks.

### 8.3 `getPoolAttestationsV2` version header during fork transition

**File:** `apis/beacon/pool/attestations.v2.yaml`

When the pool contains attestations from multiple forks (e.g., during a fork transition), the required single `Eth-Consensus-Version` header cannot accurately represent the content.

### 8.4 `getDebugForkChoice` violates the `data` wrapper contract

**File:** `apis/debug/fork_choice.yaml:14-29`

The main spec (line 21) states: "All JSON responses return the requested data under a `data` key." This endpoint puts `justified_checkpoint`, `finalized_checkpoint`, `fork_choice_nodes`, and `extra_data` directly at the top level with no `data` wrapper.

### 8.5 Duplicate version info (header + body)

Many endpoints include `version` both as a response header (`Eth-Consensus-Version`) and in the JSON body. It is not specified whether these must always match or which takes precedence.

### 8.6 `getBlockHeaders` -- `finalized` is a single boolean for multiple results

**File:** `apis/beacon/blocks/headers.yaml:31-32`

If some headers in the array are finalized and others are not, a single top-level boolean cannot accurately represent the state.

### 8.7 `canonical` field semantics undefined

**Files:** `apis/beacon/blocks/headers.yaml:42`, `apis/beacon/blocks/header.yaml:32`

The `canonical` boolean is never operationally defined. Is it relative to the current fork-choice head? Is it stable?

### 8.8 Sync committee duties response lacks `dependent_root`

**File:** `apis/validator/duties/sync.yaml:34`

Attester duties and PTC duties include `dependent_root`. Sync committee duties do not, despite also being affected by reorgs. Clients have no reorg-detection mechanism for sync committee duties.

### 8.9 Several validator responses lack `execution_optimistic`

Missing from: `aggregate_attestation.v2.yaml`, `sync_committee_contribution.yaml`, `liveness.yaml` -- despite these returning state-dependent data.

### 8.10 `getDebugChainHeadsV2` -- per-item `execution_optimistic`

**File:** `apis/debug/heads.v2.yaml`

Unlike all other endpoints that use a top-level `execution_optimistic` boolean, `getDebugChainHeadsV2` includes `execution_optimistic` per-item in the data array. This structurally different pattern is not documented as intentional.

### 8.11 `getPeers` `meta` wrapper vs `getPeerCount` flat structure

**Files:** `apis/node/peers.yaml`, `apis/node/peer_count.yaml`

`getPeers` returns `{data: [...], meta: {count: N}}` while `getPeerCount` returns `{data: {disconnected: "N", ...}}`. These overlapping endpoints provide the same information (peer counts) with different structures and different types (`meta.count` is `type: number` vs Uint64 strings in `getPeerCount`).

---

## 9. Validator Duty Endpoints

### 9.1 Epoch range constraints use inconsistent language

| Endpoint | Constraint |
|---|---|
| Attester duties | "Should only be allowed 1 epoch ahead" (soft) |
| PTC duties | "Should only be allowed 1 epoch ahead" (soft) |
| Proposer duties | No constraint stated |
| Sync duties | Formula-based constraint, no plain English |

"Should" is ambiguous in RFC terms. It is not specified whether exceeding the range produces 400.

### 9.2 Non-existent or inactive validator indices -- behavior undefined

**Files:** `apis/validator/duties/attester.yaml`, `apis/validator/duties/sync.yaml`, `apis/validator/duties/ptc.yaml`, `apis/validator/liveness.yaml`

The spec does not define whether:
- A non-existent validator index returns 400
- An inactive/exited validator is silently omitted from the response
- The response array must have the same length as the request (1:1 mapping)

Only `prepare_beacon_proposer.yaml` (line 18-19) explicitly states that unknown validators are accepted.

### 9.3 Inconsistent `uniqueItems` constraints

`apis/validator/duties/ptc.yaml:39` specifies `uniqueItems: true`. Attester duties, sync duties, and liveness do not, despite identical interface patterns. Duplicate handling is unspecified.

### 9.4 Response array ordering undefined

No duty endpoint specifies the order of the `data` array. Should results be ordered by slot, validator index, committee, or is ordering arbitrary?

### 9.5 Proposer duties completeness undefined

How many entries should proposer duty responses contain? One per slot in the epoch (32 for mainnet)? This is not documented.

### 9.6 Liveness epoch range is soft

**File:** `apis/validator/liveness.yaml:9`

"A beacon node SHOULD support the current and previous epoch, however it MAY support earlier epoch." No response code is specified for unsupported epochs.

### 9.7 Proposer duties use GET while all other duties use POST

**Files:** `apis/validator/duties/proposer.yaml`, `apis/validator/duties/proposer.v2.yaml` (GET) vs `apis/validator/duties/attester.yaml`, `apis/validator/duties/sync.yaml`, `apis/validator/duties/ptc.yaml` (POST)

Proposer duties are the only duty endpoint using GET, returning all proposers for an epoch with no filter. This means there is no mechanism to ask "will my specific validators propose this epoch?" without downloading the full set.

### 9.8 No documentation of optimal validator calling order

The typical validator flow is: get duties -> subscribe to subnets -> produce attestation data -> aggregate -> publish. The spec does not document timing constraints:
- How early before a slot should subnet subscriptions be made?
- How early can attestation data be produced relative to the slot?
- Is there a deadline after which aggregate publication will be rejected?

### 9.9 Sync committee subscription lifetime edge case

**File:** `types/altair/sync_committee.yaml:37`

The `until_epoch` field in `SyncCommitteeSubscription` is "The final epoch (exclusive value)." What happens if a subscription with `until_epoch` in the past is submitted? Is it a no-op or an error?

### 9.10 Prepare beacon proposer persistence is informal

**File:** `apis/validator/prepare_beacon_proposer.yaml:7-11`

Fee recipient info "will persist through the epoch in which the call is submitted and for a further two epochs after that, or until the beacon node restarts." The "until restart" clause makes the persistence guarantee non-deterministic and implementation-dependent.

---

## 10. Pool Endpoints

### 10.1 Single-item vs batch error semantics inconsistency

| Endpoint | Accepts | Error Type |
|---|---|---|
| `submitPoolAttestationsV2` | Array | `IndexedErrorMessage` |
| `submitPoolBLSToExecutionChange` | Array | `IndexedErrorMessage` |
| `submitPoolProposerSlashings` | Single | `ErrorMessage` |
| `submitPoolVoluntaryExit` | Single | `ErrorMessage` |
| `submitPoolAttesterSlashingsV2` | Single | `ErrorMessage` |

The distinction between single-item and batch semantics is never made explicit. `submitPoolAttesterSlashingsV2` accepts a single item, inconsistent with its GET variant returning an array.

### 10.2 Pool submissions while syncing

No pool submission endpoint (except block publish) includes 503. Whether a syncing node should accept or reject pool submissions is unspecified.

### 10.3 `indices` / `versioned_hashes` with non-matching values

**Files:** `apis/beacon/blob_sidecars/blob_sidecars.yaml:18-26`, `apis/beacon/blobs/blobs.yaml:20-28`

When blob `indices` exceed the number of blobs in a block or `versioned_hashes` don't match any blob, the behavior is not defined (silent omission? 400?).

---

## 11. Block Production and Execution Payload

### 11.1 `builder_boost_factor` default unspecified

**File:** `apis/validator/block.v3.yaml:39-70`

The parameter is optional with no default value. When omitted, should the node use 100 (profit maximization)? A node-configured default? This is critical for validators expecting predictable behavior.

### 11.2 `graffiti` omission behavior

**File:** `apis/validator/block.v3.yaml:33-36`

No specification of what graffiti the node uses when the parameter is omitted. Zero bytes? Node-configured default?

### 11.3 `skip_randao_verification` with non-infinity `randao_reveal`

**File:** `params/index.yaml:30-41`

States the randao_reveal "must be set to the point at infinity" when skip flag is set, but does not specify the error if it is not.

### 11.4 Block production for past/future slots

**File:** `apis/validator/block.v3.yaml`

No specification for what happens if the requested slot is in the past or far in the future.

### 11.5 `produceBlockV3` not forward-compatible after Gloas

**File:** `apis/validator/block.v3.yaml:17`

The description says this endpoint is not forwards-compatible after Gloas, but no replacement is documented and no error behavior is specified for Gloas slot requests.

### 11.6 `execution_payload_bid` slot constraint is informal

**File:** `apis/validator/execution_payload_bid.yaml:12`

The `slot` parameter description says "Must be current slot or next slot." This constraint exists only in prose, not at the schema level (no `minimum`/`maximum` since the valid range is dynamic).

### 11.7 `builder_index` is unvalidated and undiscoverable

**File:** `apis/validator/execution_payload_bid.yaml:18`

The `builder_index` parameter ("Index of the builder") has no validation and no documentation about what constitutes a valid builder index or how to discover valid values. This concept is not referenced elsewhere in the spec.

---

## 12. Event Stream

### 12.1 Event data schemas defined only as examples

**File:** `apis/eventstream/index.yaml:54-178`

SSE event data structures are specified only via JSON examples, not formal schemas. Clients must reverse-engineer the structure. The `payload_attributes` event references `PayloadAttributesV<N>` from execution-apis with `snake_case` adaptation but no actual schema.

### 12.2 Connection lifecycle unspecified

**File:** `apis/eventstream/index.yaml`

No specification for:
- How the server terminates the stream
- Whether retry hints are sent
- Error events within an active stream (after 200 has been sent, HTTP status codes cannot be used)

### 12.3 Empty `topics` array

The `topics` parameter is `required: true` with `uniqueItems: true`, but behavior for an empty array is not specified. Duplicate topic handling is also not defined.

---

## 13. Rewards Endpoints

### 13.1 Empty body vs empty array ambiguity

**File:** `apis/beacon/rewards/attestations.yaml:40-50`

The body is `required: false`. Sending `[]` vs omitting the body entirely may or may not be equivalent. The `ideal_rewards` default depends on body presence, but whether `[]` counts as "provided" is undefined.

### 13.2 Unknown validators in rewards requests

If a validator list is provided but none exist, should `total_rewards` be `[]` or should a 400 be returned?

### 13.3 GET vs POST inconsistency

- `getBlockRewards`: GET with `block_id`
- `getAttestationsRewards`: POST with `epoch` + body
- `getSyncCommitteeRewards`: POST with `block_id` + body

Attestation rewards use epoch while others use block ID. How epoch resolves to state (beginning? end?) is unstated.

---

## 14. Node and Config Endpoints

### 14.1 `getSyncingStatus` -- `head_slot` meaning when synced

**File:** `apis/node/syncing.yaml:23-24`

Described as "Head slot node is trying to reach" (implies target), but when synced, is it the actual head slot? The semantics change based on `is_syncing`.

### 14.2 `getPeer` -- disconnected peer ambiguity

**File:** `apis/node/peer.yaml`

The `Peer` object has a `state` field with value `disconnected`. Whether a formerly-connected peer returns 200 with `state: disconnected` or 404 "not found" is unspecified.

### 14.3 `getSpec` -- unbounded response schema

**File:** `apis/config/spec.yaml:28-40`

Response `data` is `type: object` with no defined properties, only examples. Different nodes can return different keys. Required keys are not specified.

### 14.4 `getPeers` `meta.count` vs `getPeerCount` types

`getPeers` uses `type: number` for count. `getPeerCount` uses `$ref: Uint64` (string). Same concept, different types.

### 14.5 `Peer.enr` nullable without context

**File:** `types/p2p.yaml:50-52`

Uses `oneOf` with `type: "null"` but never documents when ENR would be null.

### 14.6 `MetaData` optional fields without version guidance

**File:** `types/p2p.yaml:24-41`

Fields like `syncnets` ("not present in phase0") and `custody_group_count` ("present from Fulu") are not `required`, but there is no discriminator. Clients cannot tell whether absence means "pre-fork" or "node omitted it."

### 14.7 `getNodeVersion` v1 vs v2 structural differences

**Files:** `apis/node/version.yaml`, `apis/node/version.v2.yaml`

V1 returns `{data: {version: string}}` (deprecated) while V2 returns `{data: {beacon_node: ClientVersionV1, execution_client?: ClientVersionV1}}`. There is no guidance on when V1 will be removed.

### 14.8 `getSyncingStatus` not cross-referenced from other endpoints

**File:** `apis/node/syncing.yaml`

The syncing status endpoint exists so clients can check if the node is syncing, but no other endpoint cross-references it. There is no guidance telling clients "you should check syncing status first" or "this endpoint may return stale data if `is_syncing` is true."

### 14.9 Proposer lookahead data semantics underspecified

**File:** `apis/beacon/states/proposer_lookahead.yaml:33-37`

The response `data` is an array of `Uint64` values with `maxItems: 64`. There is no description of what these integers represent (presumably validator indices for upcoming proposer slots), the ordering convention (presumably sequential slots), or the starting slot. A consumer cannot interpret this data without out-of-band knowledge.

---

## 15. Type-Level Issues

### 15.1 `Uint64` lacks pattern validation

**File:** `types/primitive.yaml:37-39`

```yaml
Uint64:
  type: string
  example: "1"
```

No `pattern` constraint. Any string passes: `"-1"`, `"abc"`, `""`, or values exceeding 2^64. Same issue for `Uint256` and `Int64`.

### 15.2 `Uint8` pattern is incorrect

**File:** `types/primitive.yaml:99-103`

```yaml
pattern: "^[1-2]?[0-9]{1,2}$"
```

This pattern accepts `"256"` and higher values that overflow uint8. It should be something like `^(25[0-5]|2[0-4][0-9]|[01]?[0-9]{1,2})$`.

### 15.3 `ClientCode` enum is a hardcoded list

**File:** `types/primitive.yaml:20-23`

```yaml
enum: [BU, EJ, EG, GE, GR, LH, LS, NM, NB, TE, TK, PM, RH]
```

New clients not in this list will fail validation. No guidance on handling unknown codes or update process.

### 15.4 `ExtraData` name collision

`types/fork_choice.yaml:30-32` defines `ExtraData` as `type: object` (freeform). `types/primitive.yaml:119-124` defines `ExtraData` as a hex string. Same name, different types.

### 15.5 `ExecutionOptimistic` vs `Finalized` absence semantics

**File:** `types/primitive.yaml:53-61`

`ExecutionOptimistic`: "If the field is not present, assume the False value." (absent = safe)
`Finalized`: "If the field is not present, additional calls are necessary..." (absent = unknown)

These are asymmetric fallback conventions that will confuse implementers.

### 15.6 Validator aggregate status mapping only in external link

**File:** `types/api.yaml:56`

The `ValidatorStatus` enum supports fine-grained statuses (`pending_initialized`, `active_ongoing`, etc.) and coarse filter values (`active`, `pending`, `exited`, `withdrawal`). The mapping between these is documented only in an external HackMD link, not in the spec itself.

---

## 16. Schema Keyword Inconsistencies

### 16.1 `oneOf` vs `anyOf` used inconsistently

| Endpoint | Schema Keyword |
|---|---|
| `getBlockV2` (GET data) | `anyOf` |
| `publishBlockV2` (POST body) | `anyOf` |
| `submitPoolAttestationsV2` (POST body) | **`oneOf`** |
| All other polymorphic endpoints | `anyOf` |

Since fork-specific schemas are mutually exclusive, `oneOf` is semantically more correct everywhere, but only one endpoint uses it. Code generators will produce different validation logic.

### 16.2 Inline vs `$ref` parameter definitions

**File:** `apis/beacon/blocks/root.yaml:8-16`

`getBlockRoot` defines `block_id` inline rather than using the shared `$ref`. While functionally similar, this risks drift and makes the spec harder to maintain.

### 16.3 `required: []` is unusual

**File:** `apis/beacon/states/validators.yaml:113`

The POST body schema has `required: []` (empty array), which is unusual and may not be handled consistently by all OpenAPI tools.

### 16.4 SSZ list limits apply only to SSZ, not JSON

**File:** `apis/validator/aggregate_and_proofs.v2.yaml:29`

References `MAX_COMMITTEES_PER_SLOT * TARGET_AGGREGATORS_PER_COMMITTEE` as the SSZ list limit, but this limit is not applied to the JSON variant.

### 16.5 GET maxItems=64 but POST has no limit

**File:** `apis/beacon/states/validators.yaml`

The GET variant limits `id` to `maxItems: 64` and defines a 414 response. The POST variant has no such limit and no 414 response.

### 16.6 POST body shape inconsistency across state query endpoints

- `postStateValidators`: Body is an object `{ids: [], statuses: []}`
- `postStateValidatorBalances`: Body is a flat array `["id1", "id2"]`
- `postStateValidatorIdentities`: Body is a flat array

Same concept (list of validator IDs), different shapes.

### 16.7 `validator_identities` is POST-only

**File:** `apis/beacon/states/validator_identities.yaml`

This endpoint only defines a POST operation, while the similar `validators.yaml` and `validator_balances.yaml` both define GET and POST. No documentation explains why `validator_identities` lacks a GET variant.

### 16.8 `pending_consolidations.yaml` response title is wrong (bug)

**File:** `apis/beacon/states/pending_consolidations.yaml:22`

The response schema `title` is `GetPendingDepositsResponse` -- it should be `GetPendingConsolidationsResponse`. This is a copy-paste error.

### 16.9 Tag classification inconsistencies (`ValidatorRequiredApi` vs `Validator`)

Several endpoints are tagged `ValidatorRequiredApi` + `Validator` (indicating essential for validators), while others are tagged only `Validator`:

**`Validator` only (not `ValidatorRequiredApi`):**
- `apis/validator/beacon_committee_selections.yaml`
- `apis/validator/sync_committee_selections.yaml`
- `apis/validator/register_validator.yaml`
- `apis/validator/liveness.yaml`
- `apis/validator/execution_payload_bid.yaml`

The classification rationale is not documented. Additionally, `apis/validator/duties/ptc.yaml` has reversed tag ordering compared to all other dual-tagged files.

---

## Top 20 Interoperability Risks

Ranked by potential to cause divergent implementations, silent failures, or validator/client breakage.

| Rank | Ref | Issue | Risk |
|---|---|---|---|
| 1 | 4.4 | `Eth-Consensus-Version` header mismatch with body -- no error specified | SSZ payloads are uninterpretable without this header. If header and body disagree, implementations will deserialize garbage or crash. Every SSZ-accepting endpoint is affected. |
| 2 | 7.1 | SSZ responses lack type mapping documentation | Clients cannot know which SSZ container to deserialize per fork (e.g., `SignedBeaconBlock` vs `SignedBlockContents`). Wrong guess = deserialization failure. Affects all SSZ block endpoints. |
| 3 | 1.1 | `BlockId` as slot for a skipped slot -- behavior undefined | Affects 8+ endpoints. Some implementations may return 404, others may return the nearest block. Clients relying on one behavior will break on the other. |
| 4 | 5.1 | `gloas` missing from most version enums | When the Gloas fork activates, endpoints returning `version: "gloas"` will violate their own schema. Clients doing strict enum validation will reject valid responses. |
| 5 | 4.7 | Overloaded 503 semantics (syncing vs. optimistic) | Validators cannot distinguish "try again later" from "this data is untrustworthy." Wrong interpretation could lead to attesting to an optimistic head, risking slashing. |
| 6 | 11.1 | `builder_boost_factor` default unspecified | Validators who omit this parameter get implementation-defined behavior for block value selection. Could silently lose MEV revenue or always prefer builder payloads. |
| 7 | 15.1 | `Uint64` lacks pattern validation | Every numeric parameter (`slot`, `epoch`, `validator_index`, `gwei`) accepts any string. Implementations must independently decide how to handle `"-1"`, `"abc"`, or overflow values. |
| 8 | 9.2 | Non-existent validator indices -- behavior undefined | Clients don't know if the response array is 1:1 with the request. Some implementations return empty entries, others omit them, others return 400. Breaks duty-tracking logic. |
| 9 | 8.4 | `getDebugForkChoice` violates the `data` wrapper contract | The only endpoint that puts data at the top level instead of under `data`. Clients written against the documented convention will fail to parse this response. |
| 10 | 5.2 | `getBlindedBlock` version enum includes `fulu` but no Fulu data schema | Requesting a Fulu blinded block returns a version the schema can validate but data that matches no listed `anyOf` variant. Code generators will reject the response. |
| 11 | 7.3 | SSZ responses: `data` only vs envelope -- never stated | If one implementation SSZ-encodes the full envelope and another encodes only `data`, clients will fail to deserialize responses from one of them. |
| 12 | 4.5 | `InternalError` and `CurrentlySyncing` schema examples show wrong codes | Schema-level `example: 404` on a 500/503 response. Test suites and code generators using schema examples will produce incorrect expectations. |
| 13 | 15.2 | `Uint8` pattern is incorrect | Accepts `"256"` which overflows uint8. Validators submitting `custody_group_count` values could trigger different behavior across implementations. |
| 14 | 2.2 | `getBlockHeaders` -- both parameters supplied simultaneously | AND vs OR vs error -- three valid interpretations. Block explorers and monitoring tools will get inconsistent results across nodes. |
| 15 | 3.1 | POST endpoints with empty arrays -- no defined behavior | 9 endpoints affected. Some implementations return 200 (no-op), others 400. Clients that send `[]` as "clear all" or "no filter" will get unpredictable behavior. |
| 16 | 4.1 | `getBlockHeaders` 400 error references non-existent `block_id` (bug) | Outright spec bug. Implementations copying the error message will confuse clients. Implementations checking their behavior against the spec will produce wrong errors. |
| 17 | 16.8 | `pending_consolidations.yaml` response title is wrong (bug) | Code generators using schema titles for class names will generate `GetPendingDepositsResponse` for the consolidations endpoint. |
| 18 | 8.3 | `getPoolAttestationsV2` version header during fork transition | Pool may contain mixed-fork attestations but only one `Eth-Consensus-Version` header can be returned. Clients may misinterpret attestation format. |
| 19 | 14.9 | Proposer lookahead data semantics underspecified | Response is an array of Uint64 with no description of what the values represent, their ordering, or starting slot. Consumers must guess the semantics. |
| 20 | 4.6 | Missing 503 across most endpoints | Syncing nodes silently serve stale data on state, block, reward, pool, debug, and light client endpoints. Clients have no signal that data may be incomplete or outdated. |
