# CLAIMWEAVE

**Decentralized Identity and Verifiable Credential Protocol on Stellar**

## 1. Project Overview
CLAIMWEAVE is a decentralized identity (DID) and verifiable credential (VC) issuance protocol natively built on the Stellar network. It serves as the trust layer for the internet of value, allowing organizations (Issuers) to sign digital assertions about individuals (Holders) that can be verified instantly by any third party (Verifiers) without contacting the original issuer. The current identity infrastructure is a patchwork of siloed databases and centralized honeypots. CLAIMWEAVE solves this by empowering users to hold their own credentials in a self-sovereign wallet, proving specific facts on demand while revealing nothing else. By utilizing Stellar's high-speed, low-cost ledger, CLAIMWEAVE provides a global revocation registry and a directory of trusted issuers, enabling unbanked individuals to build verifiable financial identities to access DeFi loans and cross-border remittances, while allowing users in the developed world to cryptographically halt the surveillance economy.

## 2. System Design Principles
1. **Privacy by Default:** The system strictly guarantees that no personally identifiable information (PII) is ever stored on the Stellar blockchain. The smart contracts only record cryptographic hashes of credentials and boolean revocation status bits.
2. **Standardized Interoperability:** The protocol adheres strictly to the W3C Verifiable Credential (VC) and Decentralized Identifier (DID) data models, ensuring that credentials issued on CLAIMWEAVE are universally recognizable by external global identity systems.
3. **Low-Barrier Recovery:** Utilizing Stellar's native account abstractions and multisig capabilities, the Credential Wallet is designed to be recoverable through social recovery or trusted custodians without requiring non-technical users to manage complex seed phrases.
4. **Instant Revocation:** The architecture implements a deterministic Revocation List contract that allows issuers to instantly invalidate a credential (e.g., a revoked license or an expired compliance check) via a single ledger close, ensuring verifiers always query real-time validity states.

## 3. Technology Stack
1. **Smart Contract Language:** Rust version 1.76.0. Chosen for memory safety, deterministic execution, and compilation to Soroban WebAssembly (WASM).
2. **Smart Contract Framework:** Soroban SDK version 20.0.0. Chosen as the mandatory toolkit for compiling network-compatible Stellar smart contracts.
3. **Frontend Framework:** Next.js version 14.1.0 using the App Router. Chosen for server-side rendering, optimized static delivery, and robust API route handling.
4. **Frontend Language:** TypeScript version 5.4.2. Chosen for strict type safety across the application and protocol boundaries.
5. **Decentralized Storage:** `nft.storage`. Chosen for immutable, IPFS-backed hosting of the Issuer's public profile metadata and JSON-LD credential schemas.
6. **Backend API Environment:** Node.js version 20.11.1 LTS with Express version 4.18.2. Chosen for high-throughput asynchronous handling of credential schemas and issuer verification endpoints.
7. **Database:** PostgreSQL version 16. Chosen for relational integrity of the off-chain issuer directory caching.
8. **ORM:** Prisma version 5.10.0. Chosen for type-safe database migrations and querying aligned with the TypeScript backend.
9. **Infrastructure/Hosting:** Vercel (Frontend/API) and Supabase (PostgreSQL Database). Chosen for managed scaling and seamless CI/CD pipelines.

## 4. Smart Contract Architecture

### Contract 1: `claimweave_registry.rs`
**Responsibility:** The decentralized public directory. It stores the mapping between a Stellar Address and an Issuer's metadata CID (which resolves to Name, Logo, and Supported Schemas on IPFS).
**Public Functions:**
1. `register_issuer(env: Env, admin: Address, metadata_cid: String) -> Result<(), RegistryError>`
2. `update_issuer_metadata(env: Env, admin: Address, new_cid: String) -> Result<(), RegistryError>`
3. `get_issuer(env: Env, issuer: Address) -> Result<String, RegistryError>`
4. `remove_issuer(env: Env, admin: Address, issuer: Address) -> Result<(), RegistryError>`

**Storage Keys:**
1. Key: `IssuerProfile(Address)` | Type: Persistent | Value: `String` (IPFS CID) | TTL Strategy: Maintained indefinitely, funded by the issuer.
2. Key: `Admin` | Type: Instance | Value: `Address` | TTL Strategy: Instance storage, bumped automatically.

**Error Types:**
1. `RegistryError::Unauthorized`
2. `RegistryError::IssuerAlreadyRegistered`
3. `RegistryError::IssuerNotFound`

**Events Emitted:**
1. `IssuerRegisteredEvent { issuer: Address, metadata_cid: String }`
2. `IssuerUpdatedEvent { issuer: Address, new_metadata_cid: String }`

### Contract 2: `claimweave_revocation.rs`
**Responsibility:** Tracks the real-time validity of credentials. It stores a boolean status mapped to a specific cryptographic hash of a credential.
**Public Functions:**
1. `issue_credential(env: Env, issuer: Address, cred_hash: BytesN<32>) -> Result<(), RevocationError>`
2. `revoke_credential(env: Env, issuer: Address, cred_hash: BytesN<32>) -> Result<(), RevocationError>`
3. `is_valid(env: Env, issuer: Address, cred_hash: BytesN<32>) -> bool`

**Storage Keys:**
1. Key: `CredentialStatus(Address, BytesN<32>)` | Type: Persistent | Value: `bool` | TTL Strategy: Maintained indefinitely to serve as a permanent historical record of validity.

**Error Types:**
1. `RevocationError::UnauthorizedIssuer`
2. `RevocationError::CredentialAlreadyIssued`
3. `RevocationError::CredentialNotIssued`

**Events Emitted:**
1. `CredentialIssuedEvent { issuer: Address, cred_hash: BytesN<32> }`
2. `CredentialRevokedEvent { issuer: Address, cred_hash: BytesN<32> }`

## 5. Data Flow Diagrams

**Action 1: Credential Issuance**
1. An organization accesses the CLAIMWEAVE Issuer Dashboard.
2. The organization constructs a JSON object containing the user's claims.
3. The `@claimweave/core` SDK formats the JSON object into a W3C-compliant Verifiable Credential document.
4. The organization signs the Verifiable Credential using their Soroban account private key.
5. The SDK generates a SHA-256 hash of the signed Verifiable Credential.
6. The organization transmits the plain JSON Verifiable Credential to the user's wallet via a secure off-chain channel.
7. The organization submits a transaction to `claimweave_revocation.rs` calling `issue_credential` with the generated hash.
8. The contract records the credential hash as `true` (valid) on the Stellar ledger.

**Action 2: Credential Verification**
1. A user attempts to access a gated service.
2. The user's wallet transmits their Verifiable Credential JSON payload to the verifier's application.
3. The verifier utilizes the `@claimweave/core` SDK to locally verify the cryptographic signature against the issuer's public key.
4. The SDK independently generates the SHA-256 hash of the provided Verifiable Credential.
5. The SDK queries the Stellar Horizon RPC to call `is_valid` on `claimweave_revocation.rs` using the issuer address and the locally generated hash.
6. The contract returns `true`, proving the credential was issued and has not been revoked.
7. The verifier queries `claimweave_registry.rs` to fetch the IPFS CID, confirming the issuer's identity and reputation.
8. The verifier grants access to the user.

**Action 3: Credential Revocation**
1. An organization determines a credential is no longer valid.
2. The organization logs into the CLAIMWEAVE Issuer Dashboard and selects the specific credential record.
3. The organization submits a transaction to `claimweave_revocation.rs` calling `revoke_credential` with the targeted credential hash.
4. The contract updates the persistent storage value for that hash from `true` to `false`.
5. The ledger closes, and any subsequent queries to `is_valid` immediately return `false`.

## 6. SDK Architecture

### TypeScript SDK: `@claimweave/core`
**Class:** `ClaimweaveClient`
**Exported Functions:**
1. `constructor(network: string, rpcUrl: string, registryAddress: string, revocationAddress: string) -> ClaimweaveClient`
2. `createCredential(issuerDid: string, subjectDid: string, claims: Record<string, any>, schemaUrl: string) -> UnsignedCredential`
3. `signCredential(credential: UnsignedCredential, keypair: Keypair) -> SignedCredential`
4. `generateCredentialHash(signedCred: SignedCredential) -> string`
5. `verifySignatureOffChain(signedCred: SignedCredential) -> boolean`
6. `checkRevocationStatusOnChain(issuerAddress: string, credHash: string) -> Promise<boolean>`
7. `resolveIssuerMetadata(issuerAddress: string) -> Promise<IssuerMetadata>`

### Python SDK: `claimweave-py`
**Class:** `ClaimweaveClient`
**Exported Functions:**
1. `__init__(self, network: str, rpc_url: str, registry_address: str, revocation_address: str) -> None`
2. `create_credential(self, issuer_did: str, subject_did: str, claims: Dict, schema_url: str) -> Dict`
3. `sign_credential(self, credential: Dict, secret_key: str) -> Dict`
4. `generate_credential_hash(self, signed_cred: Dict) -> str`
5. `verify_signature_off_chain(self, signed_cred: Dict) -> bool`
6. `check_revocation_status_on_chain(self, issuer_address: str, cred_hash: str) -> bool`
7. `resolve_issuer_metadata(self, issuer_address: str) -> Dict`

## 7. Frontend Architecture
1. **Page: `/` (Landing Page):** Explains the CLAIMWEAVE mission for financial inclusion and self-sovereign identity.
2. **Page: `/issuer/dashboard` (Organization Portal):** The secure interface for organizations to generate, sign, and revoke credentials.
3. **Page: `/issuer/profile` (Registry Management):** Allows organizations to update their public metadata and publish schemas to `nft.storage`.
4. **Page: `/verify` (Public Verification Tool):** A drag-and-drop interface where anyone can drop a JSON credential file to instantly verify its signature and on-chain revocation status.
5. **Component: `CredentialBuilder.tsx`:** A dynamic form generator that builds JSON payloads based on standardized W3C schemas.
6. **Component: `IssuerProfileCard.tsx`:** Fetches and renders an issuer's logo, name, and verification status from IPFS.
7. **Component: `RevocationToggle.tsx`:** A button component that triggers the on-chain revocation transaction via the Freighter wallet.
8. **Hook: `useClaimweave.ts`:** Instantiates and memoizes the `@claimweave/core` client for global application access.
9. **Context: `IssuerAuthContext.tsx`:** Manages the connected wallet state and verifies if the connected address holds an active profile in the `claimweave_registry` contract.

## 8. Off-Chain Infrastructure
1. **Service: `claimweave-api`:** An Express.js REST API providing fast read access to publicly indexed issuer profiles and caching IPFS metadata to reduce latency.
2. **Endpoint: `GET /api/issuers/:address`:** Fetches the cached issuer profile from the PostgreSQL database.
3. **Endpoint: `POST /api/ipfs/pin`:** A secure proxy endpoint that accepts JSON schema definitions and pins them to `nft.storage` using the platform's backend API key.
4. **Service: `claimweave-indexer`:** A background Node.js service that polls the Stellar Horizon API for `IssuerRegisteredEvent` and `IssuerUpdatedEvent`, updating the Prisma database accordingly.
5. **Operational Specs:** Hosted on Vercel with database management handled via Supabase.

## 9. Security Model
1. **Threat Vector: PII Data Leakage.** An organization accidentally includes personal data directly in an on-chain transaction memo or payload.
   **Mitigation:** The `claimweave_revocation.rs` contract strictly accepts `BytesN<32>` hashes. The `@claimweave/core` SDK enforces this typing, physically preventing the submission of raw string data or JSON payloads to the network.
2. **Threat Vector: Unauthorized Revocation.** A malicious actor attempts to revoke another organization's credentials.
   **Mitigation:** The `revoke_credential` function requires explicit `env.auth(issuer)` authorization. A transaction will strictly revert if the signature does not match the issuer address that originally logged the credential hash.
3. **Threat Vector: Signature Forgery.** An attacker generates a fake credential and attempts to present it as valid.
   **Mitigation:** The verification standard requires cryptographically matching the signature on the Verifiable Credential to the Soroban public key. If the signature is invalid, the off-chain verification fails before any on-chain query is made.
4. **Threat Vector: Key Loss.** An individual loses access to their local Credential Wallet holding their JSON files.
   **Mitigation:** Credentials are fundamentally data files. The architecture encourages wallets to utilize encrypted cloud backups (e.g., iCloud, Google Drive). If the file is lost, the user must request a re-issuance from the organization; the protocol does not hold a backup of the claims.

## 10. Integration Points
1. **W3C Verifiable Credentials Data Model 1.1:** CLAIMWEAVE SDKs construct JSON structures that strictly adhere to this global standard, ensuring compatibility with external digital wallets (e.g., Apple Wallet, Google Wallet).
2. **nft.storage API:** Integrates with Filecoin/IPFS to pin and resolve decentralized public issuer profiles and schemas.
3. **Stellar Horizon RPC:** Used heavily for broadcasting revocation state changes and querying contract storage.

## 11. Testing Strategy
1. **Unit Tests (`test_registry.rs`, `test_revocation.rs`):** Tests individual contract functions in isolation using `Env::default()`. Validates that non-admins cannot update the registry and that the revocation state correctly toggles between true and false.
2. **Integration Tests (`test_issuance_flow.rs`):** Tests the end-to-end Soroban flow: registering an issuer, issuing a hash under that issuer's authority, and immediately revoking it to assert the boolean state transitions.
3. **SDK Tests (`client.test.ts`):** Uses Jest to test the TypeScript SDK. Validates the JSON normalization process, the SHA-256 hashing algorithms, and the Ed25519 signature verification against standard test vectors.
4. **End-to-End Tests (Playwright):** Simulates a user dragging a valid JSON credential into the `/verify` portal and asserts that the UI correctly reads the Horizon RPC and displays a green validation state.

## 12. Deployment Architecture
1. **Testnet Deployment Sequence:**
   1. Compile WASM binaries using `soroban contract build --profile release`.
   2. Optimize WASM using `soroban contract optimize`.
   3. Deploy `claimweave_registry.wasm` to Stellar Testnet and capture the contract ID.
   4. Initialize `claimweave_registry` with a Testnet Admin public key.
   5. Deploy `claimweave_revocation.wasm` to Stellar Testnet.
   6. Publish `@claimweave/core` with the `testnet` environment configuration pointing to the newly deployed addresses.
2. **Mainnet Deployment Sequence:**
   1. Mirror the exact steps from Testnet on Stellar Mainnet.
   2. Admin keys are generated via offline hardware wallets and configured as a multi-signature account before initialization.
3. **Post-Deploy Verification:** Execute the Python script to register a test issuer, issue a mock hash, and verify its validity state on the Mainnet ledger.

## 13. Upgrade and Governance Strategy
1. The `claimweave_registry` and `claimweave_revocation` contracts implement the standard Soroban upgrade interface.
2. The upgrade function requires authorization from the protocol's Governance Multisig.
3. The Governance Multisig is a 3-of-5 Gnosis-style multisig scheme involving the core founding team and trusted ecosystem partners.
4. Any proposed WASM binary upgrade must be verified on Testnet, audited, and publicly disclosed to the community 7 days prior to the multisig executing the upgrade transaction on Mainnet.

---

## PART 2 — FULL PRODUCTION PROGRESS TRACKER

### Phase 0 — Repository & Environment Setup
- [ ] Initialize Git repository named `claimweave-monorepo`.
- [ ] Configure Turborepo for monorepo management.
- [ ] Create `contracts`, `sdk`, `api`, and `app` subdirectories.
- [ ] Initialize a new Rust workspace in the `contracts` directory.
- [ ] Configure `Cargo.toml` with workspace members `claimweave-registry` and `claimweave-revocation`.
- [ ] Install Rust toolchain version 1.76.0.
- [ ] Install `soroban-cli` version 20.0.0 globally.
- [ ] Add `wasm32-unknown-unknown` target to the Rust toolchain.
- [ ] Set up GitHub Actions workflow file `.github/workflows/rust-tests.yml` to run `cargo test` on pull requests.
- [ ] Set up `.env.example` file with placeholder variables for Horizon RPC URLs and network passphrases.

### Phase 1 — Smart Contract Development
- [ ] Write the `RegistryError` enum definition in `claimweave-registry/src/errors.rs`.
- [ ] Implement the `register_issuer(env: Env, admin: Address, metadata_cid: String)` function in `claimweave-registry/src/lib.rs`.
- [ ] Implement the `update_issuer_metadata(env: Env, admin: Address, new_cid: String)` function in `claimweave-registry/src/lib.rs`.
- [ ] Implement the `get_issuer(env: Env, issuer: Address)` function in `claimweave-registry/src/lib.rs`.
- [ ] Implement the `remove_issuer(env: Env, admin: Address, issuer: Address)` function in `claimweave-registry/src/lib.rs`.
- [ ] Write the `RevocationError` enum definition in `claimweave-revocation/src/errors.rs`.
- [ ] Implement the `issue_credential(env: Env, issuer: Address, cred_hash: BytesN<32>)` function in `claimweave-revocation/src/lib.rs`.
- [ ] Implement the `revoke_credential(env: Env, issuer: Address, cred_hash: BytesN<32>)` function in `claimweave-revocation/src/lib.rs`.
- [ ] Implement the `is_valid(env: Env, issuer: Address, cred_hash: BytesN<32>)` function in `claimweave-revocation/src/lib.rs`.

### Phase 2 — Contract Testing
- [ ] Write unit test `test_register_issuer_requires_admin_auth` in `claimweave-registry/tests/test_registry.rs`.
- [ ] Write unit test `test_update_issuer_metadata_stores_correct_string` in `claimweave-registry/tests/test_registry.rs`.
- [ ] Write unit test `test_issue_credential_sets_boolean_to_true` in `claimweave-revocation/tests/test_revocation.rs`.
- [ ] Write unit test `test_revoke_credential_sets_boolean_to_false` in `claimweave-revocation/tests/test_revocation.rs`.
- [ ] Write unit test `test_revoke_credential_requires_issuer_auth` in `claimweave-revocation/tests/test_revocation.rs`.
- [ ] Write integration test `test_full_issuance_and_revocation_lifecycle` in `claimweave-revocation/tests/test_integration.rs`.

### Phase 3 — SDK Development (TypeScript)
- [ ] Initialize a new Node.js package in the `sdk/typescript` directory.
- [ ] Install dependencies: `@stellar/stellar-sdk` and `tweetnacl`.
- [ ] Configure `tsconfig.json` for strict type checking and ESModule compilation.
- [ ] Write the `ClaimweaveClient` class constructor in `sdk/typescript/src/client.ts`.
- [ ] Write the `createCredential` method to map inputs into a W3C-compliant JSON object.
- [ ] Write the `signCredential` method to sign the normalized JSON string using Ed25519.
- [ ] Write the `generateCredentialHash` method to compute the SHA-256 digest of the signed payload.
- [ ] Write the `verifySignatureOffChain` method to cryptographically validate the payload.
- [ ] Write the `checkRevocationStatusOnChain` method to interface with Horizon RPC.
- [ ] Set up Jest testing framework in the SDK directory.
- [ ] Write Jest test `test_create_credential_matches_w3c_schema`.
- [ ] Write Jest test `test_generate_credential_hash_is_deterministic`.

### Phase 4 — SDK Development (Python)
- [ ] Initialize a new Poetry project in the `sdk/python` directory.
- [ ] Install dependency `stellar-sdk` for Python.
- [ ] Write the `ClaimweaveClient` class constructor in `sdk/python/claimweave/client.py`.
- [ ] Write the `create_credential` method.
- [ ] Write the `sign_credential` method.
- [ ] Write the `generate_credential_hash` method.
- [ ] Write the `verify_signature_off_chain` method.
- [ ] Write the `check_revocation_status_on_chain` method.
- [ ] Set up `pytest` configuration.
- [ ] Write `pytest` test for credential hashing logic.
- [ ] Write `pytest` test for off-chain signature verification.

### Phase 5 — Frontend Development
- [ ] Initialize Next.js 14 App Router project in the `app` directory.
- [ ] Install Tailwind CSS, Framer Motion, and `@stellar/freighter-api`.
- [ ] Configure global styling and color variables in `globals.css` and `tailwind.config.ts`.
- [ ] Create the `IssuerAuthContext.tsx` provider to handle Freighter login and registry checks.
- [ ] Build the landing page layout in `app/page.tsx`.
- [ ] Build the `app/issuer/profile/page.tsx` view for metadata management.
- [ ] Build the `IssuerProfileCard.tsx` component to parse IPFS metadata.
- [ ] Build the `app/issuer/dashboard/page.tsx` view for credential management.
- [ ] Build the `CredentialBuilder.tsx` dynamic form component.
- [ ] Build the `RevocationToggle.tsx` component to handle smart contract transactions.
- [ ] Build the `app/verify/page.tsx` view.
- [ ] Build the `CredentialVerifier.tsx` drag-and-drop verification interface.

### Phase 6 — Off-Chain Infrastructure
- [ ] Initialize an Express.js project in the `api` directory.
- [ ] Initialize Supabase project and obtain PostgreSQL connection strings.
- [ ] Initialize Prisma ORM with `npx prisma init`.
- [ ] Write Prisma schema in `prisma/schema.prisma` defining the `IssuerProfile` cache model.
- [ ] Run `npx prisma db push` to synchronize the database schema.
- [ ] Write the `indexer.ts` background script to poll Horizon RPC for `IssuerRegisteredEvent`.
- [ ] Write Prisma insertion logic inside `indexer.ts` to save cached IPFS data to the database.
- [ ] Write Express route `GET /api/issuers/:address` to fetch cached profiles.
- [ ] Write Express route `POST /api/ipfs/pin` to handle secure uploads to `nft.storage`.
- [ ] Create a `Dockerfile` for the API and Indexer services.

### Phase 7 — Documentation
- [ ] Write `README.md` at the monorepo root explaining project structure and startup commands.
- [ ] Write `contracts/README.md` detailing the Soroban architecture and revocation data structures.
- [ ] Write `sdk/typescript/README.md` containing installation instructions and W3C data model examples.
- [ ] Write `sdk/python/README.md` containing pip installation instructions and usage examples.
- [ ] Create the `docs/architecture.md` file pasting this entire architecture specification.
- [ ] Create the `docs/api-reference.md` file documenting the Express API endpoints.
- [ ] Write inline JSDoc comments for all exported methods in the TypeScript SDK.

### Phase 8 — Security & Audit Preparation
- [ ] Run `cargo clippy --all-targets` and resolve all warnings in the Rust contracts.
- [ ] Run `npm run lint` across the Next.js and API codebases and resolve warnings.
- [ ] Add explicit boundary checks in `claimweave-registry` to ensure `metadata_cid` strings do not exceed length limits.
- [ ] Document the exact SHA-256 hashing methodology applied to Verifiable Credentials in `docs/security.md`.
- [ ] Compile a frozen version of the WASM binaries and generate SHA-256 checksums for the audit package.
- [ ] Write a comprehensive threat model document outlining signature forgery mitigations.

### Phase 9 — Testnet Deployment & QA
- [ ] Fund a Stellar Testnet account using the Friendbot faucet.
- [ ] Deploy `claimweave_registry.wasm` to Testnet and capture the contract ID.
- [ ] Initialize the Testnet `claimweave_registry` contract with Testnet admin credentials.
- [ ] Deploy `claimweave_revocation.wasm` to Testnet.
- [ ] Update `.env.testnet` files in the frontend and API directories with the new contract IDs.
- [ ] Deploy the Next.js frontend to a Vercel staging environment.
- [ ] Deploy the API and Indexer to a staging AWS ECS cluster.
- [ ] Perform a manual end-to-end test on the staging URL: register issuer profile, build credential, verify off-chain, record on-chain, and verify on-chain state.
- [ ] Fix any bugs discovered during manual QA testing.

### Phase 10 — Mainnet Deployment
- [ ] Generate strict offline hardware wallet keys for the Mainnet Admin Multisig.
- [ ] Configure the multisig account parameters on Stellar Mainnet.
- [ ] Deploy `claimweave_registry.wasm` to Mainnet.
- [ ] Initialize Mainnet `claimweave_registry` with the Mainnet Multisig address.
- [ ] Deploy `claimweave_revocation.wasm` to Mainnet.
- [ ] Update production environment variables across Vercel and AWS to point to Mainnet contract IDs and Horizon endpoints.
- [ ] Push the final production release to the master branch to trigger live deployment.
- [ ] Perform a micro-transaction recording a test hash on Mainnet to verify end-to-end viability.

### Phase 11 — Wave Program Onboarding
- [ ] Register the `claimweave-monorepo` organization on GitHub.
- [ ] Scope the specific repositories (`contracts`, `sdk/typescript`, `app`) applying to the Stellar Drips Wave Program platform as a maintainer.
- [ ] Populate the `.github/ISSUE_TEMPLATE/contributor_task.md` file to enforce standardized PR submissions.
- [ ] Create exactly 15 specific "Good First Issue" tickets targeting the upcoming short, one-week sprint (March 23 - March 30).
- [ ] Tag the issues with the appropriate Wave Program labels and specific point values aligning with the sprint budget pool.
- [ ] Write a `CONTRIBUTING.md` guide specifically tailored to the "Fix, Merge, and Earn" rhythm, explaining how to run local tests before submitting PRs during the Wave sprint.

### Phase 12 — Post-Launch Maintenance Setup
- [ ] Set up Datadog or Sentry error tracking in the Next.js frontend to monitor UI crashes during verification flows.
- [ ] Set up PM2 or Docker health checks for the `claimweave-indexer` service to alert on downtime.
- [ ] Configure PagerDuty alerts for any Supabase database connection failures or `nft.storage` API gateway timeouts.
- [ ] Establish a monitor for the `Admin` address balance to ensure it remains funded for future registry updates.