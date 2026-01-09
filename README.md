# Telegram Gift Auction Clone

Multi-round digital gift auction system inspired by Telegram Gift Auctions. A production-grade backend implementation with concurrent bid handling, anti-sniping mechanics, and strict financial correctness.

## 📋 Table of Contents

- [Auction Mechanics](#auction-mechanics)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
- [API Documentation](#api-documentation)
- [Load Testing](#load-testing)
- [Assumptions & Design Decisions](#assumptions--design-decisions)

---

## Auction Mechanics

### Overview

This is **not** a classic single-deadline auction. Instead, it's a **multi-round system** where:

- An auction is divided into **N sequential rounds**.
- In each round, a fixed number of gifts (`giftsPerRound`) are distributed.
- Winners are the **top-N bidders** by bid amount (and earlier submission time on ties).
- Non-winners' bids **automatically carry over** to the next round without re-charging their balance.
- After the last round, unclaimed bids are **refunded automatically**.

### Round Lifecycle

```
Round 1              Round 2              Round 3
[-------|]           [-------|]           [-------|]
|       |            |       |            |       |
Open    Winner       Open    Winner       Open    Winner
Decided        Decided         Decided

Non-winners stay in system → carry bid to next round
```

### Anti-Sniping Mechanism

**Problem:** Without protection, bidders could place winning bids in the last millisecond.

**Solution (Our Approach):**

- Define `antiSnipingWindowSeconds` (e.g., 30 seconds).
- If a bid is placed within the last 30 seconds before round end:
  - The round end time is **extended** by `antiSnipingExtensionSeconds` (e.g., 60 seconds).
  - Maximum of `maxAntiSnipingExtensions` (e.g., 3) to prevent infinite loops.
- This forces decisive bidding well before the deadline.

### Bid Ranking

- **Primary sort:** Bid amount (descending).
- **Tiebreaker:** Timestamp of bid submission (ascending) — first come, first served.

### Example Scenario

```
Auction params:
- totalGifts: 100
- giftsPerRound: 40
- roundDurationSeconds: 300 (5 min)
- antiSnipingWindowSeconds: 30
- antiSnipingExtensionSeconds: 60

Round 1:
  Bids: Alice=50, Bob=48, Carol=45, Dave=40, Eve=35 (40 gifts distributed)
  Winners: Top 40 by amount
  Losers: Continue to Round 2
  
Round 2:
  (Same process for remaining 60 gifts)

Round 3:
  Final round — distributes remaining gifts.
  Unclaimed bids are refunded.
```

---

## Architecture

### Project Structure

```
telegram-gift-auction-clone/
├── backend/
│   ├── src/
│   │   ├── models/
│   │   │   ├── auction.ts
│   │   │   ├── bid.ts
│   │   │   └── user.ts
│   │   ├── services/
│   │   │   ├── auctionService.ts
│   │   │   ├── bidService.ts
│   │   │   ├── balanceService.ts
│   │   │   └── roundWorker.ts
│   │   ├── controllers/
│   │   │   ├── auctionController.ts
│   │   │   ├── bidController.ts
│   │   │   └── userController.ts
│   │   ├── middleware/
│   │   │   └── errorHandler.ts
│   │   ├── utils/
│   │   │   └── logger.ts
│   │   └── app.ts
│   ├── package.json
│   ├── tsconfig.json
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   └── App.tsx
│   └── package.json
├── load-test/
│   ├── bot.ts
│   ├── stress.ts
│   └── package.json
├── docs/
│   └── auction-spec.md
├── docker-compose.yml
├── .gitignore
└── README.md
```

### Tech Stack

- **Backend:** Node.js + TypeScript + Express/Fastify
- **Database:** MongoDB (auctions, bids, user balances)
- **Concurrency:** Redis (optional, for distributed locks & rate limiting)
- **Frontend:** React + TypeScript (minimal demo UI)
- **Load Testing:** Node.js scripts with concurrent requests

### Key Design Decisions

1. **Atomic Bid Operations:**
   - Bids are atomic: check balance → deduct → store bid or rollback.
   - Use MongoDB transactions for ACID guarantees.

2. **Round Worker (Background Job):**
   - Periodically checks for completed rounds.
   - Determines winners, distributes gifts, transitions to next round.
   - Processes refunds for unclaimed bids.

3. **Financial Correctness:**
   - Invariant: `Sum(user_balances) + Sum(locked_bids) = constant`
   - Every transaction is logged for audit.
   - No money is lost or duplicated.

4. **Bid Carry-Over:**
   - When a bid loses, a new record is created for the next round.
   - Original bid amount is **not re-debited** from balance.
   - This allows users to participate in multiple rounds simultaneously.

---

## Getting Started

### Prerequisites

- Node.js 18+
- Docker & Docker Compose
- MongoDB 5.0+

### Installation & Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/ph0que/telegram-gift-auction-clone.git
   cd telegram-gift-auction-clone
   ```

2. **Start services with Docker Compose:**
   ```bash
   docker-compose up --build
   ```

   This will spin up:
   - MongoDB on `localhost:27017`
   - Backend API on `localhost:3000`
   - Frontend on `localhost:3001`
   - (Optional) Redis on `localhost:6379`

3. **Backend Setup (Manual):**
   ```bash
   cd backend
   npm install
   npm run build
   npm start
   ```

4. **Frontend Setup:**
   ```bash
   cd frontend
   npm install
   npm run dev
   ```

### Environment Variables

Create `.env` in `backend/` root:

```env
MONGO_URI=mongodb://localhost:27017/auction
PORT=3000
NODE_ENV=development
REDIS_URL=redis://localhost:6379 (optional)
```

---

## API Documentation

### Base URL

```
http://localhost:3000/api
```

### Endpoints

#### Create Auction

```http
POST /auctions
Content-Type: application/json

{
  "name": "Exclusive NFT Gift",
  "totalGifts": 100,
  "giftsPerRound": 40,
  "minBid": 10,
  "roundDurationSeconds": 300,
  "antiSnipingWindowSeconds": 30,
  "antiSnipingExtensionSeconds": 60,
  "maxAntiSnipingExtensions": 3
}

Response (201):
{
  "id": "auction_uuid",
  "status": "pending",
  "currentRound": 0,
  "startedAt": "2026-01-09T...",
  "createdAt": "2026-01-09T..."
}
```

#### Get Auction Status

```http
GET /auctions/:id

Response (200):
{
  "id": "auction_uuid",
  "name": "Exclusive NFT Gift",
  "status": "active",
  "currentRound": 1,
  "roundEndAt": "2026-01-09T19:10:00Z",
  "topBids": [
    { "userId": "user_1", "amount": 150, "submittedAt": "..." },
    { "userId": "user_2", "amount": 145, "submittedAt": "..." }
  ],
  "minWinningBid": 140,
  "totalGifts": 100,
  "giftsDistributed": 40
}
```

#### Place / Increase Bid

```http
POST /auctions/:auctionId/bids
Content-Type: application/json

{
  "userId": "user_uuid",
  "amount": 150
}

Response (201 or 200):
{
  "bidId": "bid_uuid",
  "auctionId": "auction_uuid",
  "round": 1,
  "userId": "user_uuid",
  "amount": 150,
  "status": "pending",
  "submittedAt": "2026-01-09T...",
  "position": 2  // rank among current bids
}
```

#### Get User Balance

```http
GET /users/:userId/balance

Response (200):
{
  "userId": "user_uuid",
  "balance": 1000,
  "lockedInBids": 150,
  "available": 850
}
```

#### Deposit Funds

```http
POST /users/:userId/deposit
Content-Type: application/json

{
  "amount": 500
}

Response (200):
{
  "userId": "user_uuid",
  "balance": 1500,
  "transaction": "txn_uuid"
}
```

---

## Load Testing

Test the system under concurrent load and anti-sniping behavior.

### Run Bot Simulation

```bash
cd load-test
npm install
ts-node bot.ts --bots 50 --auctionId <AUCTION_ID>
```

### Stress Test

```bash
ts-node stress.ts --concurrent 100 --duration 60 --antiSnipingTest true
```

Metrics collected:
- Request success/failure rates
- Response times (avg, p95, p99)
- Balance integrity check
- Anti-sniping extension triggers

---

## Assumptions & Design Decisions

### 1. Bid Carry-Over

**Assumption:** When a user loses a round, their bid amount is **not re-debited**. Instead, a new bid record is created with a reference to the original.

**Why:** This simplifies refund logic and allows users to participate in multiple rounds without recharging balance.

### 2. Anti-Sniping Extension Limit

**Assumption:** Maximum 3 anti-sniping extensions per round to prevent infinite extensions.

**Why:** Ensures auctions eventually close while maintaining fairness.

### 3. First-Come-First-Served Tiebreaker

**Assumption:** If two users bid the same amount, the earlier bid wins.

**Why:** Encourages decisive early bidding, reduces last-second jostling.

### 4. No Partial Gifts

**Assumption:** Gifts are atomic — either won or not. No fractional ownership.

**Why:** Simplifies distribution logic and matches Telegram's model.

### 5. Automatic Refunds

**Assumption:** After the auction ends, non-winning bids are automatically refunded without user action.

**Why:** Improves UX and ensures no funds are indefinitely locked.

---

## Financial Invariants

### Guarantee

At any point in time:

```
Sum(user_balances) + Sum(locked_in_bids) + Sum(distributed_prizes_value) = CONSTANT
```

No money is created or destroyed — only transferred between states.

### Validation

The system includes a health-check endpoint:

```http
GET /admin/audit/balance

Response:
{
  "status": "healthy",
  "totalBalances": 50000,
  "totalLocked": 5000,
  "totalPrizes": 45000,
  "sum": 100000,
  "expectedSum": 100000,
  "ok": true
}
```

---

## Development

### Running Tests

```bash
cd backend
npm run test
```

### Code Style

- **Prettier:** Code formatting
- **ESLint:** Linting with TypeScript support

```bash
npm run lint
npm run format
```

### Debugging

Enable debug logs:

```bash
DEBUG=auction:* npm start
```

---

## License

MIT

---

## Contributing

Contributions welcome! Please follow:
1. Commit messages: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Submit a pull request with detailed description.

---

## Contact

Questions? Reach out via GitHub Issues or discussion.
