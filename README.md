# Ledger API

A Node.js ledger service for user authentication, account management, balance tracking, and transactional money transfers using MongoDB.

The application uses an immutable double-entry ledger. Every transfer creates:

- One `DEBIT` entry for the sender
- One `CREDIT` entry for the receiver

Account balances are calculated from ledger entries instead of being stored directly.

## Features

- User registration and login
- Password hashing with bcrypt
- JWT authentication
- Cookie-based and Bearer-token authentication
- Token blacklist-based logout
- User account creation
- Account balance calculation
- Double-entry transactions
- Transaction idempotency keys
- MongoDB transactions for atomic updates
- System-user-only initial fund transfers
- Registration and transaction email notifications
- Immutable ledger entries

## Technology Stack

- Node.js
- Express 5
- MongoDB
- Mongoose
- JSON Web Tokens
- bcryptjs
- Nodemailer
- dotenv
- cookie-parser

## Requirements

- Node.js 18 or later
- MongoDB configured as a replica set
- Gmail OAuth2 credentials for email notifications

MongoDB transactions require replica-set support. You can use a local MongoDB replica set or MongoDB Atlas.

## Installation

Clone the repository and install the dependencies:

```bash
git clone https://github.com/vrt-fardeen/Ledger.git
cd Ledger
npm install
```

### Packages Used

| Package | Purpose |
| --- | --- |
| `express` | HTTP server and API routing |
| `mongoose` | MongoDB connection, schemas, and transactions |
| `bcryptjs` | Password hashing and verification |
| `jsonwebtoken` | JWT creation and verification |
| `cookie-parser` | Reading authentication cookies |
| `dotenv` | Loading environment variables |
| `nodemailer` | Sending email notifications |

For development, the project runs Nodemon through `npx`, so no separate Nodemon installation is required.

## Environment Variables

Create a `.env` file in the project root:

```env
MONGO_URI=mongodb://127.0.0.1:27017/ledger?replicaSet=rs0
JWT_SECRET=replace-with-a-long-random-secret

EMAIL_USER=your-email@gmail.com
CLIENT_ID=your-google-oauth-client-id
CLIENT_SECRET=your-google-oauth-client-secret
REFRESH_TOKEN=your-google-oauth-refresh-token
```

Do not commit `.env` or OAuth credentials to GitHub.

## Running the Application

Development mode:

```bash
npm run dev
```

Production mode:

```bash
npm start
```

The server starts on:

```text
http://localhost:3000
```

Health check:

```http
GET /
```

Response:

```text
Ledger Service is up and running
```

## Authentication

Authentication is implemented with JSON Web Tokens.

After registration or login, the server:

1. Creates a JWT containing the user's ID.
2. Signs it using `JWT_SECRET`.
3. Sets the token in a `token` cookie.
4. Also returns the token in the JSON response.
5. Sets the token expiration to three days.

Protected endpoints accept either authentication format:

```http
Cookie: token=<jwt>
```

or:

```http
Authorization: Bearer <jwt>
```

The authentication middleware:

1. Reads the token from the cookie or Authorization header.
2. Checks whether the token exists in the blacklist collection.
3. Verifies the token using `JWT_SECRET`.
4. Loads the associated user.
5. Stores the user in `req.user`.

Invalid, missing, expired, or blacklisted tokens return HTTP `401 Unauthorized`.

## Authorization

Regular protected routes use `authMiddleware`.

The initial-funds endpoint uses `authSystemUserMiddleware`. This middleware:

1. Verifies the JWT.
2. Loads the user's `systemUser` property.
3. Allows access only when `systemUser` is `true`.
4. Returns HTTP `403 Forbidden` for normal users.

The `systemUser` property is:

- Hidden by default in queries
- Immutable after creation
- Used to authorize system-only operations

Account balance access also checks that the requested account belongs to the authenticated user.

## API Endpoints

### Authentication

#### Register

```http
POST /api/auth/register
Content-Type: application/json
```

Request:

```json
{
  "email": "user@example.com",
  "name": "John Doe",
  "password": "password123"
}
```

The password is hashed with bcrypt before being stored.

Response: `201 Created`

```json
{
  "user": {
    "_id": "user-id",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "token": "jwt-token"
}
```

A registration email is sent after the response is created.

#### Login

```http
POST /api/auth/login
Content-Type: application/json
```

Request:

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

The submitted password is compared with the stored bcrypt hash.

Response: `200 OK`

```json
{
  "user": {
    "_id": "user-id",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "token": "jwt-token"
}
```

#### Logout

```http
POST /api/auth/logout
```

The token is stored in the blacklist collection and the `token` cookie is cleared.

Blacklisted tokens automatically expire after three days.

### Accounts

All account endpoints require authentication.

#### Create Account

```http
POST /api/accounts/
Authorization: Bearer <jwt-token>
```

Response: `201 Created`

```json
{
  "account": {
    "_id": "account-id",
    "user": "user-id",
    "status": "ACTIVE",
    "currency": "INR"
  }
}
```

#### Get User Accounts

```http
GET /api/accounts/
Authorization: Bearer <jwt-token>
```

Returns all accounts belonging to the authenticated user.

#### Get Account Balance

```http
GET /api/accounts/balance/:accountId
Authorization: Bearer <jwt-token>
```

The account must belong to the authenticated user.

Response:

```json
{
  "accountId": "account-id",
  "balance": 1000
}
```

The balance is calculated as:

```text
balance = total credits - total debits
```

### Transactions

#### Create Transaction

```http
POST /api/transactions/
Authorization: Bearer <jwt-token>
Content-Type: application/json
```

Request:

```json
{
  "fromAccount": "sender-account-id",
  "toAccount": "receiver-account-id",
  "amount": 250,
  "idempotencyKey": "unique-request-key-001"
}
```

The transaction flow is:

1. Validate the required request fields.
2. Confirm both accounts exist.
3. Check the idempotency key.
4. Check that both accounts are active.
5. Calculate the sender's balance from the ledger.
6. Reject the request if the balance is insufficient.
7. Start a MongoDB session and transaction.
8. Create a transaction with `PENDING` status.
9. Create a `DEBIT` ledger entry for the sender.
10. Create a `CREDIT` ledger entry for the receiver.
11. Mark the transaction as `COMPLETED`.
12. Commit the MongoDB transaction.
13. Send a transaction email to the authenticated user.

The same `idempotencyKey` prevents a completed request from being processed again.

Possible responses include:

- `201 Created`: Transaction completed
- `200 OK`: Transaction already processed
- `200 OK`: Transaction is still processing
- `400 Bad Request`: Invalid request, account, status, balance, or transaction failure
- `500 Internal Server Error`: Existing transaction failed or was reversed

Successful response:

```json
{
  "message": "Transaction completed successfully",
  "transaction": {
    "_id": "transaction-id",
    "fromAccount": "sender-account-id",
    "toAccount": "receiver-account-id",
    "amount": 250,
    "status": "COMPLETED",
    "idempotencyKey": "unique-request-key-001"
  }
}
```

#### Create Initial Funds

```http
POST /api/transactions/system/initial-funds
Authorization: Bearer <system-user-jwt>
Content-Type: application/json
```

Request:

```json
{
  "toAccount": "user-account-id",
  "amount": 1000,
  "idempotencyKey": "initial-funds-001"
}
```

This endpoint is restricted to users whose `systemUser` property is `true`.

It creates a transaction from the system user's account to the target account, then creates:

- A system-account debit
- A target-account credit

Response:

```json
{
  "message": "Initial funds transaction completed successfully",
  "transaction": {
    "_id": "transaction-id",
    "status": "COMPLETED"
  }
}
```

## Application Logic

### Users

Users have:

- Email address
- Name
- Hashed password
- System-user flag
- Creation and update timestamps

Passwords are never stored in plain text.

### Accounts

Each account belongs to one user and has:

- Owner
- Status: `ACTIVE`, `FROZEN`, or `CLOSED`
- Currency, defaulting to `INR`

### Transactions

Transactions have:

- Source account
- Destination account
- Amount
- Status
- Unique idempotency key
- Creation and update timestamps

Supported statuses:

- `PENDING`
- `COMPLETED`
- `FAILED`
- `REVERSED`

### Ledger

Ledger entries are immutable and contain:

- Account
- Amount
- Transaction reference
- Type: `DEBIT` or `CREDIT`

Ledger entries cannot be updated or deleted through Mongoose operations. This keeps the account history auditable.

## Project Structure

```text
.
├── server.js
├── package.json
└── src
    ├── app.js
    ├── config
    │   └── db.js
    ├── controllers
    │   ├── account.controller.js
    │   ├── auth.controller.js
    │   └── transaction.controller.js
    ├── middleware
    │   └── auth.middleware.js
    ├── models
    │   ├── account.model.js
    │   ├── blackList.model.js
    │   ├── ledger.model.js
    │   ├── transaction.model.js
    │   └── user.model.js
    ├── routes
    │   ├── account.routes.js
    │   ├── auth.routes.js
    │   └── transaction.routes.js
    └── services
        └── email.service.js
```

## Email Notifications

Nodemailer uses Gmail OAuth2 to send:

- Registration emails
- Successful transaction emails
- Failed transaction emails

Required variables:

```env
EMAIL_USER
CLIENT_ID
CLIENT_SECRET
REFRESH_TOKEN
```

Email failures are logged by the email service.

## Important Implementation Notes

- MongoDB transactions require replica-set support.
- The normal transaction flow currently waits 15 seconds between the debit and credit operations.
- The transaction idempotency key is unique in MongoDB.
- Ledger entries are immutable.
- The project currently has no automated tests configured.
- The `npm test` command currently exits with a placeholder message.
- A system user and system account must exist before initial funds can be issued.
- The regular transaction endpoint currently verifies that both account IDs exist, but it does not verify that `fromAccount` belongs to the authenticated user. This authorization check should be added before production use.
