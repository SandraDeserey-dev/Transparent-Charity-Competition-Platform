# Transparent Charity Competition Platform (TCCP)

## Overview

The Transparent Charity Competition Platform (TCCP) is a Web3 project built on the Stacks blockchain using Clarity smart contracts. It addresses real-world problems in the charity sector, such as lack of transparency in donation handling, donor distrust due to mismanagement scandals, and unfair competition among charities for limited funds. By leveraging blockchain, TCCP ensures that charities compete fairly for donations through a transparent, verifiable process.

### Core Idea
- **Fair Competition**: Charities register and submit verifiable impact metrics (e.g., via oracles). Donors contribute to a shared donation pool.
- **Transparency via Blockchain**: All donations, votes, and distributions are recorded on-chain, allowing anyone to audit the process.
- **Solving Real-World Problems**:
  - **Donor Trust**: Immutable records prevent fraud and ensure funds reach intended causes.
  - **Efficient Allocation**: Competition based on community votes and real impact metrics rewards effective charities.
  - **Inclusivity**: Small charities can compete with larger ones on merit, reducing monopolies.
  - **Global Accessibility**: Permissionless participation for donors and charities worldwide.

The platform involves 6 core smart contracts written in Clarity, handling registration, donations, voting, oracles, distribution, and governance.

## Architecture

TCCP operates on the Stacks blockchain, which settles on Bitcoin for security. Users interact via a dApp frontend (not included here; assume integration with wallets like Hiro Wallet).

### Smart Contracts
All contracts are written in Clarity. Below is a high-level description of each, followed by code snippets. The contracts interact via traits and public functions for modularity.

1. **CharityRegistry.clar**: Manages charity registration and verification.
   - Allows charities to register with details (name, description, wallet address).
   - Admins or community can verify charities to prevent scams.
   - Stores charity metadata on-chain.

2. **DonationPool.clar**: Handles incoming donations.
   - Accepts STX (Stacks token) or SIP-10 tokens as donations.
   - Locks funds in a pool until distribution periods.
   - Tracks total donations and per-donor contributions.

3. **VotingMechanism.clar**: Enables donor voting for charities.
   - Donors get voting power proportional to their donations (e.g., 1 vote per STX donated).
   - Quadratic voting to prevent whale dominance.
   - Time-bound voting rounds.

4. **ImpactOracle.clar**: Integrates real-world impact data.
   - Uses oracles (e.g., via Chainlink or custom feeds) to fetch off-chain metrics like "number of meals provided" or "trees planted."
   - Validates and stores impact scores to influence distribution.

5. **FundDistributor.clar**: Distributes funds based on votes and impact.
   - At the end of a cycle, calculates shares using a formula (e.g., 70% votes + 30% impact).
   - Automatically transfers funds to winning charities' wallets.
   - Emits events for transparency.

6. **GovernanceToken.clar**: Issues a governance token (TCCP-TOKEN).
   - SIP-10 compliant fungible token.
   - Donors receive tokens as rewards for participation.
   - Token holders propose and vote on platform upgrades (e.g., changing distribution weights).

### Contract Interactions
- A donor sends STX to `DonationPool`, which mints voting rights in `VotingMechanism` and rewards via `GovernanceToken`.
- Charities register in `CharityRegistry` and submit data to `ImpactOracle`.
- At cycle end, `FundDistributor` queries votes and impacts, then distributes funds.
- All contracts use traits for interoperability (e.g., a `donation-trait` for pool interactions).

## Installation and Deployment

### Prerequisites
- Stacks CLI (clarinet) for local development.
- Hiro Wallet or Leather for testing.
- Node.js for any frontend integration (optional).

### Setup
1. Clone the repo: `git clone https://github.com/yourusername/tccp.git`
2. Install Clarinet: Follow [Stacks docs](https://docs.stacks.co/clarity).
3. Navigate to project: `cd tccp`
4. Initialize: `clarinet integrate`
5. Deploy contracts locally: `clarinet deploy`

### Directory Structure
```
tccp/
├── contracts/
│   ├── CharityRegistry.clar
│   ├── DonationPool.clar
│   ├── VotingMechanism.clar
│   ├── ImpactOracle.clar
│   ├── FundDistributor.clar
│   ├── GovernanceToken.clar
├── tests/
│   ├── charity-registry-test.clar
│   └── ... (tests for each contract)
├── Clarinet.toml
└── README.md
```

## Smart Contract Code Snippets

Below are simplified Clarity code examples for each contract. Full implementations would include error handling, maps, and more functions.

### 1. CharityRegistry.clar
```clarity
(define-trait charity-trait
  {
    (register-charity (principal string string) (response bool uint))
  }
)

(define-map charities principal { name: (string-ascii 50), desc: (string-utf8 200), verified: bool })

(define-public (register-charity (owner principal) (name (string-ascii 50)) (desc (string-utf8 200)))
  (map-insert charities owner { name: name, desc: desc, verified: false })
  (ok true)
)

(define-public (verify-charity (owner principal))
  (match (map-get? charities owner)
    some-charity (map-set charities owner (merge some-charity { verified: true })) (ok true)
    (err u1)
  )
)
```

### 2. DonationPool.clar
```clarity
(define-data-var total-donations uint u0)
(define-map donor-contributions principal uint)

(define-public (donate (amount uint))
  (try! (stx-transfer? amount tx-sender (as-contract tx-sender)))
  (var-set total-donations (+ (var-get total-donations) amount))
  (map-set donor-contributions tx-sender (+ (default-to u0 (map-get? donor-contributions tx-sender)) amount))
  (ok amount)
)
```

### 3. VotingMechanism.clar
```clarity
(define-map votes { donor: principal, charity: principal } uint)
(define-map voting-power principal uint)

(define-public (vote (charity principal) (amount uint))
  (let ((power (default-to u0 (map-get? voting-power tx-sender))))
    (asserts! (>= power amount) (err u2))
    (map-set votes { donor: tx-sender, charity: charity } (+ (default-to u0 (map-get? votes { donor: tx-sender, charity: charity })) amount))
    (map-set voting-power tx-sender (- power amount))
    (ok true)
  )
)
```

### 4. ImpactOracle.clar
```clarity
(define-map impact-scores principal uint)

(define-public (submit-impact (charity principal) (score uint) (oracle principal))
  (asserts! (is-eq oracle (var-get trusted-oracle)) (err u3))
  (map-set impact-scores charity score)
  (ok true)
)
```

### 5. FundDistributor.clar
```clarity
(define-public (distribute-funds)
  (let ((total-votes (fold sum-votes charities u0))
        (total-impact (fold sum-impacts charities u0)))
    ;; Logic to calculate shares and transfer STX
    (fold distribute-to-charity charities (ok true))
  )
)

(define-private (distribute-to-charity (charity principal) (acc (response bool uint)))
  ;; Calculate share = (0.7 * votes / total-votes) + (0.3 * impact / total-impact) * total-donations
  ;; Then stx-transfer? to charity
)
```

### 6. GovernanceToken.clar
```clarity
;; SIP-10 Fungible Token
(define-fungible-token tccp-token)
(define-data-var token-uri (string-utf8 256) u"")

(define-public (transfer (amount uint) (sender principal) (recipient principal) (memo (optional (buff 34))))
  (ft-transfer? tccp-token amount sender recipient)
)

(define-public (mint (amount uint) (recipient principal))
  (ft-mint? tccp-token amount recipient)
)
```

## Testing
- Use Clarinet to run unit tests: `clarinet test`
- Example test: Simulate donation, voting, and distribution.

## Future Enhancements
- Integrate with real oracles for impact data.
- Add DAO governance for parameter changes.
- Frontend dApp for user-friendly interactions.

## License
MIT License. See LICENSE file for details.