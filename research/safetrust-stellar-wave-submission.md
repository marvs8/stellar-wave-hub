# SafeTrust — Stellar Wave Research Submission

## Project Selected

- **Project:** SafeTrust
- **Wave source:** `safetrustcr/frontend-SafeTrust` and `safetrustcr/backend-SafeTrust` — both listed in Stellar Wave repos on Drips (4x Points tier)
- **Domain:** Payments / P2P Escrow / DeFi
- **Organization:** SafeTrust CR — Costa Rica
- **Repositories:**
  - Frontend: https://github.com/safetrustcr/frontend-SafeTrust
  - Backend: https://github.com/safetrustcr/backend-SafeTrust
  - dApp MVP monorepo: https://github.com/safetrustcr/dApp-SafeTrust
  - Landing page: https://github.com/safetrustcr/landing-SafeTrust
- **Telegram:** https://t.me/safetrustcr
- **X / Twitter:** https://x.com/SafeTrustCR
- **Figma Design:** https://www.figma.com/design/CVg9hoim0f1FIlozIar7ZZ/SafeTrust

## Why This Matches the Task

SafeTrust is a verified Stellar Wave Program participant with two repos approved at the **4x Points** tier — the highest multiplier available on Drips. The project has accumulated 21 stars and 92 forks on its frontend repo alone, with 817 commits across active development. It is a full-stack, open-source decentralized escrow platform that uses the Stellar blockchain as its core trust layer through the TrustlessWork API. The project was not previously submitted to the Hub at the time of this submission.

## What SafeTrust Does

SafeTrust is a decentralized platform for secure peer-to-peer (P2P) transactions — primarily rental deposits and payments — powered by blockchain escrow on the Stellar network. The core problem it solves is the trust gap in P2P rental markets: when a tenant pays a deposit to a landlord, there is no neutral party enforcing the return of that deposit if the landlord acts in bad faith. Traditional platforms like Airbnb solve this by being a centralized intermediary, but they charge fees and hold custody of funds.

SafeTrust removes the intermediary entirely. Funds are locked in a Soroban smart contract escrow on Stellar via the TrustlessWork API. Neither the tenant nor the landlord can unilaterally access the funds — release is governed by the agreed contract terms. If the rental completes without dispute, the owner receives payment. If there is a dispute, the tenant recovers the deposit. The entire flow is on-chain, auditable, and requires no trust in SafeTrust itself as a company.

The platform targets the Latin American rental market, where informal P2P rental arrangements are common but deposit disputes are a persistent problem. By using USDC on Stellar (a stablecoin pegged to the US dollar), SafeTrust avoids the volatility risk that would make crypto-denominated deposits impractical for everyday renters.

## Technical Architecture

SafeTrust is organized across four repositories that together form a complete product:

### 1. Frontend (`frontend-SafeTrust`) — Next.js 15

The main user-facing application. Built with:
- **Next.js 15** (App Router) with TypeScript
- **Tailwind CSS** for styling
- **Apollo Client 4** for GraphQL queries and real-time subscriptions
- **Firebase Authentication** (Email/Password + Google OAuth) for user identity
- **Freighter, Albedo, and LOBSTR** wallet integrations for Stellar key management
- **TrustlessWork API** client for escrow lifecycle operations

The frontend communicates with the Hasura GraphQL layer for all data reads. Write operations (escrow deployment, auth sync) go through the backend webhook service. This separation keeps the frontend stateless with respect to business logic.

### 2. Backend (`backend-SafeTrust`) — Hasura + PostgreSQL

The data and middleware layer. Built with:
- **Hasura GraphQL Engine** — auto-generates a full GraphQL API from PostgreSQL tables, including row-level permissions per role (`tenant`, `landlord`)
- **PostgreSQL** as the primary database
- **Karate framework** for API testing (BDD-style `.feature` files)
- **Docker Compose** for local development orchestration
- **Multi-tenant metadata architecture** — `metadata/tenants/` allows the same Hasura instance to serve multiple tenant configurations

The backend tracks these core tables: `users`, `user_wallets`, `apartments`, `bid_requests`, `trustless_work_escrows`, `escrow_milestones`, `escrow_transactions`, `trustless_work_webhook_events`.

### 3. dApp MVP Monorepo (`dApp-SafeTrust`) — Turborepo

The integration monorepo that wires everything together for the MVP. Uses:
- **Turborepo** as the build system
- **pnpm workspaces** for package management
- `apps/frontend` — Next.js 14 (App Router), port 3001
- `apps/api` — Node.js + Express, port 3000 (handles escrow deploy via TrustlessWork)
- `services/webhook` — Firebase Auth sync service (runs in Docker)
- `infra/hasura` — Docker Compose stack (PostgreSQL + Hasura + webhook)
- `packages/types` — shared TypeScript interfaces
- `packages/graphql` — codegen output (Apollo typed hooks)

The architecture diagram from the repo:

```
Stellar Blockchain (TrustlessWork API)
         │ HTTP (signed XDR)
services/webhook (Node.js + Express)
  Firebase Auth sync · POST /api/auth/sync-user
         │ SQL (pg pool)
infra/hasura (MIDDLEWARE)
  Auto-generated GraphQL API · JWT validation
  Tables: users · escrows · bid_requests · milestones
         │ GraphQL (queries · mutations · subscriptions)
apps/frontend (Next.js 14)
  Apollo Client · Firebase Auth · Freighter Wallet
  /login → /register → /dashboard → PAY → escrow live
```

### 4. Landing Page (`landing-SafeTrust`) — Next.js

Marketing and onboarding site. 12 stars, 74 forks, actively maintained.

## Stellar Integration

SafeTrust integrates with Stellar in two distinct ways:

**1. Escrow via TrustlessWork API**

The core escrow mechanism delegates to [TrustlessWork](https://trustlesswork.com), a Stellar Wave participant that provides Escrow-as-a-Service (EaaS) built on Soroban smart contracts. SafeTrust calls the TrustlessWork REST API to:
- Deploy a new escrow contract for each rental agreement
- Fund the escrow with USDC (the tenant's deposit)
- Release funds to the landlord on agreement completion
- Return funds to the tenant on dispute resolution

This is a deliberate architectural choice: rather than writing and auditing custom Soroban escrow contracts, SafeTrust leverages a battle-tested, permissionless escrow primitive. The TrustlessWork contracts run on Stellar testnet (and mainnet for production flows).

**2. Wallet Integration**

Users connect their Stellar wallets (Freighter, Albedo, or LOBSTR) to sign escrow transactions. The wallet address is stored in `public.user_wallets` and linked to the user's Firebase identity. Freighter is the recommended wallet — it is a browser extension that signs Stellar XDR transactions without exposing the private key to the application.

**On-chain verification:**

SafeTrust uses the TrustlessWork API endpoint `https://api.trustlesswork.com` (mainnet) and `https://dev.api.trustlesswork.com` (testnet). The TrustlessWork escrow contract on Stellar testnet is the underlying on-chain primitive. The `NEXT_PUBLIC_TRUSTLESS_NETWORK=testnet` environment variable confirms the current deployment targets Stellar testnet.

Stellar Horizon verification for TrustlessWork escrow activity:
- `https://horizon-testnet.stellar.org/accounts/` (escrow contract accounts)
- `https://api.stellar.expert/explorer/testnet/` (contract invocations)

## Escrow Lifecycle

The complete P2P rental flow:

1. **Tenant finds property** — browses apartment listings on the dashboard
2. **Bid request** — tenant submits a bid (`PENDING` status in `bid_requests` table)
3. **Landlord accepts** — bid moves to `ACTIVE`
4. **Escrow deployed** — tenant clicks PAY, connects Freighter wallet, `apps/api` calls TrustlessWork `/escrow/initiate` to deploy a Soroban escrow contract on Stellar
5. **Escrow funded** — tenant signs the funding transaction via Freighter; USDC locked on-chain
6. **Rental agreement active** — both parties can see escrow status in real-time via Hasura GraphQL subscriptions
7. **Completion** — on successful rental, landlord triggers release; TrustlessWork `/escrow/complete` releases USDC to landlord
8. **Dispute** — if disputed, TrustlessWork handles the dispute resolution flow and returns deposit to tenant

## Community & Ecosystem

| Metric | Value |
|---|---|
| Stellar Wave tier | 4x Points (highest tier) |
| frontend-SafeTrust stars | 21 |
| frontend-SafeTrust forks | 92 |
| frontend-SafeTrust commits | 817+ |
| backend-SafeTrust stars | 11 |
| backend-SafeTrust forks | 84 |
| backend-SafeTrust commits | 592+ |
| dApp-SafeTrust forks | 35 |
| dApp-SafeTrust commits | 336+ |
| landing-SafeTrust stars | 12 |
| landing-SafeTrust forks | 74 |
| Open issues (frontend) | 8 |
| Open PRs (frontend) | 9 |
| Organization location | Costa Rica |
| Telegram | https://t.me/safetrustcr |

The high fork count relative to stars (92 forks vs 21 stars on the frontend) is characteristic of a Wave program project where contributors fork to work on open issues. The 8 open issues and 9 open PRs on the frontend repo confirm active, ongoing development.

## Independent Research Notes

Several things stand out from independent analysis beyond the project's own README:

**Architectural maturity:** The multi-tenant Hasura metadata structure in `backend-SafeTrust` (`metadata/tenants/`, `metadata/base/`, `metadata/build/`) is more sophisticated than a typical hackathon project. The `setup-tenant.sh` script that merges tenant-specific config with shared base config suggests the team is building toward a multi-tenant SaaS product, not just a single-tenant demo.

**Deliberate escrow delegation:** Choosing TrustlessWork over writing custom Soroban contracts is a sound security decision. Custom escrow contracts are a common attack surface. By delegating to a specialized, audited escrow primitive, SafeTrust reduces its attack surface and time-to-market simultaneously.

**Testing infrastructure:** The backend uses the Karate BDD testing framework with Docker-based test isolation (`docker-compose-test.yml`). This is a production-grade testing setup, not a stub. The frontend uses Jest + React Testing Library + Cypress for unit, integration, and E2E coverage.

**Known gaps (from the dApp README's "Known State" section):** The Freighter wallet integration is partially stubbed — `useWalletDetection` hook exists but the actual connect flow is not fully wired. The escrow deploy frontend button is not yet connected to `apps/api`. These are documented gaps, not hidden issues, which reflects good contributor hygiene.

**Firebase + Stellar identity model:** The hybrid identity model (Firebase for auth, Stellar wallet for on-chain signing) is a pragmatic choice for onboarding non-crypto-native users. Users can register with email/password and add a wallet later, lowering the barrier to entry compared to wallet-only auth.

## Submission Details

- **Category:** `payments`
- **Tags:** `escrow, soroban, p2p, stellar-wave, defi, rental, trustlesswork, usdc, freighter, latam`
- **Stellar Account / Contract:** TrustlessWork escrow contracts on Stellar testnet (via `https://api.trustlesswork.com`)
- **Verification:** `https://horizon-testnet.stellar.org` — escrow accounts created per rental agreement via TrustlessWork API
- **Network:** Stellar Testnet (development) / Stellar Mainnet (production via `https://api.trustlesswork.com`)
