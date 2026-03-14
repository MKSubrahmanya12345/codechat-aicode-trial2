# CodeME.md - BlockPost: Decentralized Content Proof & Proof-of-Impact

## 1. Context & Business Goal

BlockPost is a revolutionary platform designed to empower creators by providing irrefutable, decentralized proof of content originality and a novel "Proof-of-Impact" mechanism for earning micro-royalties.

**Core Problem Solved:** Content creators struggle with proving ownership and monetizing their work effectively in an era of rampant unauthorized re-uploads and minimal transparency.

**BlockPost's Solution:**
1.  **Multi-Layer Originality Checks:**
    *   **SHA-256:** Detects exact re-uploads.
    *   **Perceptual Hashing (pHash):** Flags cropped, resized, or filtered versions.
    *   **Audio Fingerprinting:** Identifies reused audio even with different visuals.
    *   This ensures near-duplicate content is caught and attributed.
2.  **Decentralized Storage:** Media and metadata are immutably pinned on IPFS (via Pinata/NFT.Storage), removing single points of failure and ensuring content persistence.
3.  **On-Chain Proof of Originality:** A metadata hash + timestamp for each upload is recorded on a Polygon Testnet smart contract, serving as a public, tamper-proof receipt of publication by a specific wallet.
4.  **Proof-of-Impact (New Feature):** Content verified as original by BlockPost can automatically earn micro-royalties based on its engagement. An analytics service tracks user interactions, which feeds into an on-chain impact score, enabling creators to claim earned value.

**Business Goal:** To provide creators with a simple, intuitive platform to secure their intellectual property, verify originality, and monetize their content transparently through engagement-driven royalties, making every piece of content a verifiable asset. The system should feel instant to the user despite complex backend operations.

## 2. Exact Tech Stack

### Client Applications
*   **Creator App (ID: 1):** React + Vite
*   **User/Viewer App (ID: 2):** React + Vite
*   **Web3 Provider (ID: 7):** MetaMask (for wallet connection and signing transactions)
*   **Blockchain Interaction:** Ethers.js (integrated into both client apps for reading blockchain data, and potentially the Creator App for initiating transactions via MetaMask)

### Backend Service
*   **BlockPost Backend (ID: 3):** Node.js + Express
*   **Hosting:** Render
*   **Blockchain Interaction:** Ethers.js (for writing proofs, updating impact, and reading blockchain data)
*   **IPFS Interaction:** Pinata API / NFT.Storage API
*   **Hashing Libraries:** Node.js crypto module (SHA-256), a suitable library for pHash (e.g., `image-hash`), and an audio fingerprinting library (e.g., `audioprint` or a custom implementation using `ffmpeg` and hashing).

### Data Stores
*   **Database (ID: 4):** MongoDB Atlas (NoSQL DB)
*   **Decentralized Storage (ID: 5):** IPFS (Pinata/NFT.Storage)
*   **Blockchain (ID: 6):** Polygon Testnet (Smart Contracts written in Solidity)

### Analytics
*   **Analytics Service (ID: 8):** PostHog or a custom Node.js service for tracking engagement.

## 3. Database Schemas (MongoDB Atlas)

### `User` Collection
Stores user profiles linked to their blockchain wallets.
```json
{
  "_id": "ObjectId",
  "walletAddress": { "type": "String", "required": true, "unique": true },
  "username": { "type": "String", "default": null },
  "email": { "type": "String", "default": null },
  "createdAt": { "type": "Date", "default": "Date.now" },
  "updatedAt": { "type": "Date", "default": "Date.now" }
}
```

### `ContentMetadata` Collection
Stores metadata for each piece of content, including originality proofs and IPFS links.
```json
{
  "_id": "ObjectId",
  "creatorWalletAddress": { "type": "String", "required": true, "ref": "User" },
  "title": { "type": "String", "required": true },
  "description": { "type": "String", "default": "" },
  "ipfsMediaCid": { "type": "String", "required": true, "unique": true },
  "ipfsMetadataCid": { "type": "String", "required": true, "unique": true },
  "sha256Hash": { "type": "String", "required": true, "unique": true },
  "pHash": { "type": "String", "required": true },
  "audioFingerprint": { "type": "String", "default": null }, // Null if not audio/video
  "blockchainProofTxHash": { "type": "String", "required": true, "unique": true }, // Transaction hash for ProofOfOriginality
  "isOriginal": { "type": "Boolean", "default": true },
  "originalContentId": { "type": "ObjectId", "ref": "ContentMetadata", "default": null }, // Null if original, refers to original if duplicate
  "status": { "type": "String", "enum": ["pending", "verified", "flagged"], "default": "pending" },
  "engagementScore": { "type": "Number", "default": 0 }, // Aggregated score from analytics
  "royaltiesAccumulated": { "type": "Number", "default": 0 }, // Total royalties calculated (off-chain)
  "createdAt": { "type": "Date", "default": "Date.now" },
  "updatedAt": { "type": "Date", "default": "Date.now" }
}
```

### `EngagementData` Collection
Records individual user interactions for analytics and impact calculation.
```json
{
  "_id": "ObjectId",
  "contentId": { "type": "ObjectId", "required": true, "ref": "ContentMetadata" },
  "viewerWalletAddress": { "type": "String", "default": null, "ref": "User" }, // Can be null for anonymous viewers
  "eventType": { "type": "String", "enum": ["view", "like", "share", "comment"], "required": true },
  "timestamp": { "type": "Date", "default": "Date.now" },
  "metadata": { "type": "Object", "default": {} } // E.g., { durationWatched: 30, country: "US" }
}
```

### `RoyaltyPayouts` Collection
Logs actual royalty payout transactions.
```json
{
  "_id": "ObjectId",
  "contentId": { "type": "ObjectId", "ref": "ContentMetadata" }, // Can be null if payout is for total creator earnings
  "creatorWalletAddress": { "type": "String", "required": true, "ref": "User" },
  "amount": { "type": "Number", "required": true }, // Amount in native token (e.g., MATIC)
  "currency": { "type": "String", "default": "MATIC" },
  "txHash": { "type": "String", "required": true, "unique": true }, // Blockchain transaction hash for payout
  "status": { "type": "String", "enum": ["pending", "completed", "failed"], "default": "pending" },
  "timestamp": { "type": "Date", "default": "Date.now" }
}
```

### Smart Contracts (Polygon Testnet)

#### `ProofOfOriginality.sol`
```solidity
// Stores content hash to its original creator and timestamp
contract ProofOfOriginality {
    struct OriginalContent {
        address creator;
        uint256 timestamp;
    }
    mapping(bytes32 => OriginalContent) public contentProofs;

    event ContentRegistered(bytes32 indexed contentHash, address indexed creator, uint256 timestamp);

    function registerOriginalContent(bytes32 _contentHash, address _creator) public {
        require(contentProofs[_contentHash].creator == address(0), "Content already registered.");
        contentProofs[_contentHash] = OriginalContent(_creator, block.timestamp);
        emit ContentRegistered(_contentHash, _creator, block.timestamp);
    }

    function getOriginalCreator(bytes32 _contentHash) public view returns (address) {
        return contentProofs[_contentHash].creator;
    }

    function getOriginalTimestamp(bytes32 _contentHash) public view returns (uint256) {
        return contentProofs[_contentHash].timestamp;
    }
}
```

#### `ProofOfImpact.sol`
```solidity
// Stores impact scores and manages royalty distribution based on engagement
contract ProofOfImpact {
    address public owner; // Backend service address
    mapping(bytes32 => uint256) public contentImpactScores; // contentHash => total impact score
    mapping(address => uint256) public creatorRoyalties; // creatorAddress => accumulated royalties ready for withdrawal

    uint256 public constant ROYALTY_PER_IMPACT_UNIT = 100000000000000; // Example: 0.0001 MATIC per impact unit (10^14 wei)

    event ImpactScoreUpdated(bytes32 indexed contentHash, uint256 newScore);
    event RoyaltiesClaimed(address indexed creator, uint256 amount);

    constructor() {
        owner = msg.sender; // Set the deployer as the owner (backend service)
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function.");
        _;
    }

    function setOwner(address _newOwner) public onlyOwner {
        owner = _newOwner;
    }

    // Called by the backend to update impact score, which in turn updates creator royalties
    function updateImpactScore(bytes32 _contentHash, address _creator, uint256 _newScore) public onlyOwner {
        uint256 oldScore = contentImpactScores[_contentHash];
        contentImpactScores[_contentHash] = _newScore;

        if (_newScore > oldScore) {
            uint256 scoreIncrease = _newScore - oldScore;
            uint256 royaltiesToAdd = scoreIncrease * ROYALTY_PER_IMPACT_UNIT;
            creatorRoyalties[_creator] += royaltiesToAdd;
        }
        emit ImpactScoreUpdated(_contentHash, _newScore);
    }

    function getImpactScore(bytes32 _contentHash) public view returns (uint256) {
        return contentImpactScores[_contentHash];
    }

    function getCreatorRoyalties(address _creator) public view returns (uint256) {
        return creatorRoyalties[_creator];
    }

    // Allows a creator to claim their accumulated royalties
    function claimRoyalties() public {
        uint256 amount = creatorRoyalties[msg.sender];
        require(amount > 0, "No royalties to claim.");

        creatorRoyalties[msg.sender] = 0; // Reset balance before transfer to prevent reentrancy
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Failed to send MATIC.");

        emit RoyaltiesClaimed(msg.sender, amount);
    }
}
```

## 4. API Endpoint Definitions (BlockPost Backend)

All endpoints should be protected by JWT authentication where applicable.

### Authentication
*   **`POST /api/auth/challenge`**
    *   **Description:** Initiates MetaMask login by providing a signed message challenge.
    *   **Request:** `{ "walletAddress": "0x..." }`
    *   **Response:** `{ "message": "Sign this message to authenticate: BlockPost-nonce-..." }`
*   **`POST /api/auth/verify`**
    *   **Description:** Verifies the signed message and authenticates/registers the user.
    *   **Request:** `{ "walletAddress": "0x...", "signature": "0x..." }`
    *   **Response:** `{ "token": "jwt_token_string", "user": { "_id": "...", "walletAddress": "..." } }`

### Content Management & Originality
*   **`POST /api/content/upload`**
    *   **Description:** Uploads a file, performs multi-layer hashing, pins to IPFS, and registers on-chain proof.
    *   **Authentication:** Required
    *   **Request:** `multipart/form-data`
        *   `file`: The content file (image, video, audio)
        *   `title`: Content title
        *   `description`: Content description
    *   **Response:** `201 Created`
        *   `{ "contentId": "ObjectId", "ipfsMediaCid": "Qm...", "ipfsMetadataCid": "Qm...", "blockchainProofTxHash": "0x...", "isOriginal": true, "originalContentId": null, "status": "verified" }`
*   **`POST /api/content/check-originality`**
    *   **Description:** Checks the originality of an uploaded file without fully processing/storing it.
    *   **Authentication:** Optional (can be public for quick checks)
    *   **Request:** `multipart/form-data`
        *   `file`: The content file
    *   **Response:** `200 OK`
        *   `{ "isOriginal": false, "originalContentId": "ObjectId_of_duplicate", "originalCreatorWallet": "0x...", "originalContentTitle": "..." }`
*   **`GET /api/content`**
    *   **Description:** Retrieves a list of content items.
    *   **Authentication:** Optional
    *   **Query Params:** `?page=1&limit=10&creator=0x...&search=keyword`
    *   **Response:** `200 OK`
        *   `{ "data": [ { ...ContentMetadata }, ... ], "total": 100, "page": 1, "limit": 10 }`
*   **`GET /api/content/:contentId`**
    *   **Description:** Retrieves detailed metadata for a specific content item.
    *   **Authentication:** Optional
    *   **Response:** `200 OK`
        *   `{ ...ContentMetadata }`
*   **`GET /api/content/:contentId/media`**
    *   **Description:** Serves the actual media file from IPFS via a gateway.
    *   **Authentication:** Optional
    *   **Response:** `200 OK` (Redirect or direct stream of content)

### Proof-of-Impact & Royalties
*   **`POST /api/content/:contentId/track-engagement`**
    *   **Description:** Records an engagement event for a content item.
    *   **Authentication:** Optional (User wallet address can be null for anonymous tracking)
    *   **Request:** `{ "eventType": "view", "metadata": { "durationWatched": 30, "location": "US" } }`
    *   **Response:** `202 Accepted`
*   **`GET /api/content/:contentId/impact`**
    *   **Description:** Retrieves the current impact score and royalty information for a content item.
    *   **Authentication:** Optional
    *   **Response:** `200 OK`
        *   `{ "contentId": "ObjectId", "engagementScore": 1500, "royaltiesAccumulated": 0.15, "onChainImpactScore": 1500 }`
*   **`GET /api/user/:walletAddress/royalties`**
    *   **Description:** Retrieves the total accumulated royalties available for a specific creator.
    *   **Authentication:** Optional (Can be public to view others' earnings)
    *   **Response:** `200 OK`
        *   `{ "walletAddress": "0x...", "totalRoyaltiesAvailable": 0.5, "onChainRoyalties": 0.5 }`
*   **`POST /api/royalties/claim`**
    *   **Description:** Allows a creator to claim their accumulated royalties. Triggers a blockchain transaction.
    *   **Authentication:** Required (Wallet address must match authenticated user)
    *   **Request:** `{}` (Implicitly claims for the authenticated user)
    *   **Response:** `200 OK`
        *   `{ "txHash": "0x...", "amountClaimed": 0.5, "status": "pending" }`
*   **`GET /api/royalties/payouts`**
    *   **Description:** Retrieves a history of royalty payouts.
    *   **Authentication:** Required (User's own payouts)
    *   **Query Params:** `?creator=0x...` (Admin endpoint)
    *   **Response:** `200 OK`
        *   `{ "data": [ { ...RoyaltyPayouts }, ... ] }`

### Blockchain Interaction (Read-only, Backend Proxy)
*   **`GET /api/blockchain/proofs/:contentHash`**
    *   **Description:** Fetches on-chain proof of originality for a given content hash.
    *   **Authentication:** Optional
    *   **Response:** `200 OK`
        *   `{ "creator": "0x...", "timestamp": 1678886400 }`
*   **`GET /api/blockchain/impact/:contentHash`**
    *   **Description:** Fetches on-chain impact score for a given content hash.
    *   **Authentication:** Optional
    *   **Response:** `200 OK`
        *   `{ "impactScore": 1500 }`

## 5. Folder Scaffolding Structure

This structure assumes a monorepo setup, which is ideal for managing multiple related applications and shared components.

```
/blockpost-monorepo
├── /apps                                  # Frontend applications
│   ├── /creator-app                       # React + Vite application for creators
│   │   ├── public                         # Static assets
│   │   ├── src
│   │   │   ├── api                        # API client for BlockPost Backend
│   │   │   ├── assets
│   │   │   ├── components                 # Reusable React components
│   │   │   ├── hooks                      # Custom React hooks
│   │   │   ├── pages                      # Page-level components (e.g., Upload, Dashboard)
│   │   │   ├── services                   # Web3 interactions (MetaMask, Ethers.js), IPFS gateway
│   │   │   ├── utils                      # Utility functions
│   │   │   ├── App.jsx                    # Main application component
│   │   │   └── main.jsx                   # Entry point
│   │   ├── .env.development
│   │   ├── .env.production
│   │   ├── index.html
│   │   ├── package.json
│   │   ├── postcss.config.js
│   │   ├── tailwind.config.js
│   │   └── vite.config.js
│   └── /viewer-app                        # React + Vite application for content viewers
│       ├── public
│       ├── src
│       │   ├── api
│       │   ├── assets
│       │   ├── components
│       │   ├── hooks
│       │   ├── pages                      # E.g., Content Feed, Content Detail
│       │   ├── services                   # Web3 interactions (read-only), analytics tracking
│       │   ├── utils
│       │   ├── App.jsx
│       │   └── main.jsx
│       ├── .env.development
│       ├── .env.production
│       ├── index.html
│       ├── package.json
│       ├── postcss.config.js
│       ├── tailwind.config.js
│       └── vite.config.js
├── /services
│   └── /blockpost-backend                 # Node.js + Express backend service
│       ├── src
│       │   ├── config                     # Environment variables, constants, DB connection
│       │   ├── controllers                # Request handlers, orchestrates services
│       │   ├── middleware                 # Authentication, error handling, CORS
│       │   ├── models                     # Mongoose schemas (User, ContentMetadata, EngagementData, RoyaltyPayouts)
│       │   ├── routes                     # API endpoint definitions
│       │   ├── services                   # Business logic:
│       │   │   ├── authService.js         # Wallet authentication
│       │   │   ├── contentService.js      # Hashing, IPFS pinning, originality checks
│       │   │   ├── blockchainService.js   # Ethers.js interactions with Polygon contracts
│       │   │   ├── analyticsService.js    # Integration with PostHog or custom analytics
│       │   │   └── royaltyService.js      # Royalty calculation and payout logic
│       │   ├── utils                      # Helper functions (e.g., IPFS client, hashing utilities)
│       │   ├── app.js                     # Express application setup
│       │   └── server.js                  # Server bootstrap (connect to DB, start server)
│       ├── .env.example                   # Example environment variables
│       ├── package.json
│       ├── Dockerfile                     # For Render deployment
│       └── README.md
├── /packages
│   ├── /smart-contracts                   # Solidity smart contracts
│   │   ├── contracts
│   │   │   ├── ProofOfOriginality.sol
│   │   │   └── ProofOfImpact.sol
│   │   ├── scripts                        # Deployment scripts, interaction scripts
│   │   ├── test                           # Unit tests for contracts
│   │   ├── hardhat.config.js              # Hardhat configuration (or Foundry.toml)
│   │   └── package.json
│   └── /analytics-service                 # (Optional) If PostHog is not used, a simple custom service
│       ├── src
│       │   └── index.js                   # Example: simple event listener, aggregates data
│       ├── package.json
│       └── .env.example
├── /shared                                # Shared code/types across monorepo
│   ├── /types                             # TypeScript interfaces/types for data structures
│   ├── /config                            # Shared constants (e.g., contract addresses, IPFS gateway URL)
│   └── /utils                             # Generic utility functions
├── .env.example                           # Root-level example for shared env vars (e.g., MongoDB URI, IPFS keys)
├── package.json                           # Monorepo root package.json (with workspaces config)
└── README.md                              # Project README
```