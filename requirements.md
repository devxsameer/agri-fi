# Requirements Document

## Introduction

The Agric-onchain Finance Platform digitizes agricultural trade by connecting farmers, traders, and investors through a blockchain-backed system built on Stellar. The MVP enables farmer registration with KYC, trade deal creation and tokenization, document upload to decentralized storage, investor funding of tokenized deals, and automated escrow payment release upon delivery confirmation. The goal is to make agricultural trade transparent and accessible to small farmers who are currently excluded from trade financing.

## Glossary

- **Farmer**: A registered user who lists agricultural produce and receives financing through trade deals
- **Trader**: A registered user who creates purchase contracts and manages shipments
- **Investor**: A registered user who funds tokenized trade deals and earns returns upon completion
- **Trade_Deal**: A tokenized agricultural trade agreement between a farmer and a trader, fundable by investors
- **Trade_Token**: A Stellar-issued asset representing fractional ownership in a Trade_Deal (e.g., COCOA-TRADE-1002)
- **Escrow**: A Stellar-based holding account that locks investor funds until delivery is confirmed
- **Document**: A digitized trade document (purchase agreement, bill of lading, export certificate, warehouse receipt) stored on IPFS/S3 with its hash recorded on Stellar
- **KYC**: Know Your Customer — identity verification required before a user can participate in trade deals
- **Milestone**: A verified logistics checkpoint in the shipment journey (Farm → Warehouse → Port → Importer)
- **Platform**: The Agric-onchain Finance web application
- **Stellar_SDK**: The Stellar blockchain SDK used for asset issuance, escrow, and payments
- **IPFS**: InterPlanetary File System used for decentralized document storage
- **Wallet**: A Stellar-compatible wallet used to sign transactions and hold assets

---

## Requirements

### Requirement 1: User Registration and KYC

**User Story:** As a farmer, trader, or investor, I want to register on the platform with my role and identity, so that I can participate in agricultural trade deals with verified credentials.

#### Acceptance Criteria

1. WHEN a user submits a registration form with name, email, password, role (farmer, trader, or investor), and country, THE Platform SHALL create a new user account with a `pending_kyc` status
2. WHEN a user submits valid KYC documents (government ID and proof of address), THE Platform SHALL update the user's `kyc_status` to `verified`
3. IF a user attempts to create or fund a trade deal with `kyc_status` not equal to `verified`, THEN THE Platform SHALL reject the action and return an error message indicating KYC is required
4. WHEN a user connects a Stellar-compatible wallet, THE Platform SHALL associate the wallet address with the user's account
5. THE Platform SHALL enforce that each email address is associated with at most one user account
6. IF a user submits a registration with an email that already exists, THEN THE Platform SHALL return an error indicating the email is already registered
7. WHEN a user's KYC is verified, THE Platform SHALL notify the user via email

---

### Requirement 2: Trade Deal Creation

**User Story:** As a trader, I want to create a purchase contract for agricultural produce, so that I can formalize a trade agreement with a farmer and open it for investor funding.

#### Acceptance Criteria

1. WHEN a verified trader submits a trade deal with commodity type, quantity (in kg or tons), total value (in USD), farmer ID, and expected delivery date, THE Platform SHALL create a new Trade_Deal record with status `draft`
2. WHEN a trader publishes a Trade_Deal, THE Platform SHALL issue a Stellar asset (Trade_Token) representing the deal, with the token symbol derived from the commodity and deal ID (e.g., COCOA-1002)
3. THE Platform SHALL calculate the number of tokens as `floor(total_value / 100)`, where each token has a face value of $100 USD
4. WHEN a Trade_Deal is published, THE Platform SHALL set the deal status to `open` and make it visible to investors
5. IF a trader attempts to publish a Trade_Deal without at least one associated document, THEN THE Platform SHALL reject the publication and return an error
6. WHILE a Trade_Deal has status `open`, THE Platform SHALL display the remaining funding amount as `total_value - total_invested`
7. WHEN a Trade_Deal is fully funded (total invested equals total value), THE Platform SHALL automatically update the deal status to `funded`
8. THE Platform SHALL associate each Trade_Deal with exactly one farmer and one trader

---

### Requirement 3: Document Upload and Storage

**User Story:** As a trader or farmer, I want to upload trade documents to the platform, so that all parties have access to verified, tamper-proof records of the trade agreement.

#### Acceptance Criteria

1. WHEN a user uploads a document file (PDF, PNG, or JPEG, maximum 10 MB), THE Platform SHALL store the file on IPFS or S3 and record the resulting content hash in the database
2. WHEN a document is successfully stored, THE Platform SHALL record the IPFS/S3 hash as a transaction memo or data entry on the Stellar blockchain, linking it to the Trade_Deal
3. THE Platform SHALL support the following document types: purchase agreement, bill of lading, export certificate, and warehouse receipt
4. IF a user uploads a file exceeding 10 MB, THEN THE Platform SHALL reject the upload and return an error specifying the size limit
5. IF a user uploads a file with an unsupported format (not PDF, PNG, or JPEG), THEN THE Platform SHALL reject the upload and return an error specifying accepted formats
6. WHEN a document is uploaded, THE Platform SHALL associate it with a specific Trade_Deal and record the uploader's user ID
7. WHEN a document hash is recorded on Stellar, THE Platform SHALL store the Stellar transaction ID alongside the document record for auditability

---

### Requirement 4: Investor Funding

**User Story:** As an investor, I want to fund a tokenized trade deal by purchasing trade tokens, so that I can earn a return when the trade is completed.

#### Acceptance Criteria

1. WHEN a verified investor selects a Trade_Deal with status `open` and specifies a token quantity, THE Platform SHALL calculate the investment amount as `token_quantity * 100` USD
2. WHEN an investor confirms a funding transaction, THE Platform SHALL transfer the investment amount from the investor's wallet to the Trade_Deal's Escrow account on Stellar
3. WHEN the Stellar escrow transaction is confirmed, THE Platform SHALL issue the corresponding Trade_Tokens to the investor's Stellar wallet and create an investment record
4. IF an investor attempts to purchase more tokens than are available in a Trade_Deal, THEN THE Platform SHALL reject the transaction and return an error indicating insufficient tokens remaining
5. IF the Stellar transaction fails, THEN THE Platform SHALL not create an investment record and SHALL return an error with the Stellar transaction failure reason
6. WHEN an investor funds a deal, THE Platform SHALL update the Trade_Deal's `total_invested` field by adding the investment amount
7. THE Platform SHALL allow multiple investors to fund the same Trade_Deal until it is fully funded
8. WHEN a Trade_Deal reaches full funding, THE Platform SHALL notify all participating investors via email

---

### Requirement 5: Shipment Tracking

**User Story:** As a trader, I want to record and track logistics milestones for a shipment, so that all parties can verify the progress of the trade from farm to importer.

#### Acceptance Criteria

1. WHEN a trader records a shipment milestone (Farm, Warehouse, Port, or Importer) with a timestamp and optional notes, THE Platform SHALL create a milestone record linked to the Trade_Deal
2. WHEN a milestone is recorded, THE Platform SHALL submit a Stellar transaction with the milestone data encoded in the transaction memo, creating a verifiable on-chain event
3. THE Platform SHALL enforce that milestones are recorded in the order: Farm → Warehouse → Port → Importer
4. IF a trader attempts to record a milestone out of sequence, THEN THE Platform SHALL reject the action and return an error indicating the expected next milestone
5. WHEN the `Importer` milestone is recorded, THE Platform SHALL update the Trade_Deal status to `delivered`
6. WHILE a Trade_Deal has status `funded`, THE Platform SHALL allow the assigned trader to record milestones

---

### Requirement 6: Escrow Payment Release

**User Story:** As a farmer and investor, I want payments to be released automatically when delivery is confirmed, so that funds are distributed fairly and without manual intervention.

#### Acceptance Criteria

1. WHEN a Trade_Deal status changes to `delivered`, THE Platform SHALL initiate the escrow release process on Stellar
2. WHEN the escrow is released, THE Platform SHALL transfer the farmer's payment (total_value minus platform fee) from the Escrow account to the farmer's Stellar wallet
3. THE Platform SHALL calculate the platform fee as 2% of the total_value
4. WHEN the escrow is released, THE Platform SHALL distribute investor returns to each investor's Stellar wallet proportional to their token holdings
5. WHEN all payments are successfully distributed, THE Platform SHALL update the Trade_Deal status to `completed`
6. IF a Stellar payment transaction fails during escrow release, THEN THE Platform SHALL log the failure, retain the funds in escrow, and notify the platform administrator
7. WHEN a Trade_Deal is completed, THE Platform SHALL notify the farmer, trader, and all investors via email
8. THE Platform SHALL record each payment transaction's Stellar transaction ID for auditability

---

### Requirement 7: Platform Dashboard and Visibility

**User Story:** As any registered user, I want to view a dashboard relevant to my role, so that I can monitor my trade deals, investments, and payments in one place.

#### Acceptance Criteria

1. WHEN a farmer logs in, THE Platform SHALL display a list of Trade_Deals associated with the farmer, including status, funded amount, and shipment milestones
2. WHEN a trader logs in, THE Platform SHALL display all Trade_Deals created by the trader, including funding progress and current shipment milestone
3. WHEN an investor logs in, THE Platform SHALL display all investments made by the investor, including token holdings, deal status, and expected returns
4. WHEN any user views a Trade_Deal detail page, THE Platform SHALL display the commodity, quantity, value, funding progress, document list, and shipment milestones
5. THE Platform SHALL display only Trade_Deals with status `open` on the public investment marketplace
