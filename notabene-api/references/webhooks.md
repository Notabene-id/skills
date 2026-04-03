# Notabene Webhooks Documentation

This document provides comprehensive information about all webhook events sent by the Notabene platform to receiving developers. Webhooks are delivered via [Svix](https://svix.com/) with automatic retries, exponential backoff, and signature verification.

## Table of Contents

- [Overview](#overview)
- [Webhook Envelope Structure](#webhook-envelope-structure)
- [Webhook Categories](#webhook-categories)
- [Authentication & Security](#authentication--security)
- [Webhook Events](#webhook-events)
  - [Transfer Management Events](#transfer-management-events)
  - [TAP Policy Requirement Events](#tap-policy-requirement-events)
  - [TAP Policy Satisfaction Events](#tap-policy-satisfaction-events)
  - [TAP Policy Status Events](#tap-policy-status-events)
  - [Flow Payment Events](#flow-payment-events)
- [Common Fields](#common-fields)
- [Error Handling](#error-handling)
- [Testing](#testing)

## Overview

Notabene webhooks notify your application about important events in real-time during the transaction authorization process. These events are critical for maintaining compliance with travel rule requirements and managing the transfer lifecycle.

**Delivery Method**: HTTP POST to your configured endpoint URLs
**Delivery Service**: Svix (reliable delivery with retries)
**Content-Type**: `application/json`
**Signature Verification**: All webhooks are signed for security

## Webhook Envelope Structure

All webhook events are delivered inside a standard envelope. The `message` field contains the event type string, the `payload` field contains the event-specific data, and the `version` field indicates the payload format version.

```json
{
  "message": "notification.transferCreated",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123"
  },
  "version": "1.0.0"
}
```

| Field     | Type   | Description                                      |
| --------- | ------ | ------------------------------------------------ |
| `message` | string | The webhook event type identifier                |
| `payload` | object | Event-specific data (see individual event docs)  |
| `version` | string | Payload format version (currently `"1.0.0"`)     |

> **Note**: The event type is only present in the `message` field of the envelope. It is **not** included inside the `payload` object.

## Webhook Categories

### 1. Transfer Management Events

These webhooks notify you about changes in transfer state and agent management:

- **notification.transferCreated**: A new transfer has been initiated
- **notification.transferStatusChanged**: Transfer status has been updated
- **notification.transferAgentAdded**: A new agent has been added to a transfer
- **notification.transferAgentReplaced**: An agent has been replaced in a transfer
- **notification.transferAgentStatusChanged**: An agent's status has been updated

### 2. TAP Policy Requirement Events

These webhooks are sent when the Notabene system requires external actions to continue processing a transfer:

- **tap.requirePresentationRequested**: Request for credential/document presentation
- **tap.requireAuthorizationRequested**: Request for transfer authorization decision
- **tap.requireRelationshipConfirmationRequested**: Request to confirm entity relationships

### 3. TAP Policy Satisfaction Events

These webhooks confirm that previously requested actions have been completed:

- **tap.requireAuthorizationSatisfied**: Authorization requirement has been fulfilled
- **tap.requirePresentationSatisfied**: Presentation requirement has been fulfilled
- **tap.requireRelationshipConfirmationSatisfied**: Relationship confirmation requirement has been fulfilled

### 4. TAP Policy Status Events

These webhooks provide updates on the overall status of policy processing:

- **tap.policySatisfied**: A policy has been fully satisfied
- **tap.requirePresentationPartiallySatisfied**: A presentation requirement has been partially fulfilled (some but not all requested credentials received)

### 5. Flow Payment Events

These webhooks are sent for Notabene Flow payment processing workflows. Flow provides enhanced payment processing with customer interaction capabilities for payouts and payins.

**Payin Events** (Customer paying merchant):

- **flow.payin.created**: Payin request created
- **flow.payin.opened**: Customer opened payment link
- **flow.payin.assetSelected**: Customer selected payment asset
- **flow.payin.agentAdded**: Agent added to payin transfer
- **flow.payin.settlementAddressSelected**: Settlement address selected
- **flow.payin.authorizationRequired**: Payin requires authorization
- **flow.payin.fundingAdded**: Funding address information added to payin

**Payout Events** (Merchant paying customer):

- **flow.payout.created**: Payout request created
- **flow.payout.opened**: Customer opened payout link
- **flow.payout.assetSelected**: Customer selected payout asset
- **flow.payout.agentAdded**: Agent added to payout transfer
- **flow.payout.settlementAddressSelected**: Settlement address selected
- **flow.payout.authorizationRequired**: Payout requires authorization
- **flow.payout.fundingAdded**: Funding address information added to payout

---

## Authentication & Security

All webhooks include Svix signatures for verification. You should validate these signatures to ensure the webhook originated from Notabene:

```javascript
import { Webhook } from "svix";

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

// Verify webhook signature
try {
  const payload = webhook.verify(body, headers);
  // Process the webhook...
} catch (err) {
  // Invalid signature
  console.error("Webhook signature verification failed");
}
```

---

## Webhook Events

---

### Transfer Management Events

#### notification.transferCreated

**Purpose**: Notifies that a new transfer has been created in the system.

**When Sent**: Immediately after a transfer is successfully created and enters the processing pipeline.

**Expected Action**: Optional - log the event, update internal records, or trigger business workflows.

**Payload Structure**:

```json
{
  "message": "notification.transferCreated",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field | Type   | Required | Description                       |
| ----- | ------ | -------- | --------------------------------- |
| `for` | string | Yes      | Entity DID receiving this webhook |
| `id`  | string | Yes      | Transfer ID                       |

**Response Required**: None (informational only).

---

#### notification.transferStatusChanged

**Purpose**: Notifies that a transfer's status has changed.

**When Sent**: When a transfer moves between states (e.g., from "PENDING" to "AUTHORIZED", "AUTHORIZED" to "SETTLED").

**Expected Action**: Update your system's transfer status tracking.

**Payload Structure**:

```json
{
  "message": "notification.transferStatusChanged",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "fromStatus": "PENDING",
    "toStatus": "AUTHORIZED",
    "settlementId": "settlement-456"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field          | Type   | Required | Description                                  |
| -------------- | ------ | -------- | -------------------------------------------- |
| `for`          | string | Yes      | Entity DID receiving this webhook            |
| `id`           | string | Yes      | Transfer ID                                  |
| `fromStatus`   | string | Yes      | Previous transfer status                     |
| `toStatus`     | string | Yes      | New transfer status                          |
| `settlementId` | string | No       | Settlement identifier (when status involves settlement) |

**Response Required**: None (informational only).

**Common Status Values**: `PENDING`, `AUTHORIZED`, `REJECTED`, `SETTLED`, `FLAGGED`, `REVERTED`, `FROZEN`, `CLEARED`

---

#### notification.transferAgentAdded

**Purpose**: Notifies that a new agent has been added to a transfer.

**When Sent**: When an intermediary entity or service provider is added to facilitate the transfer.

**Expected Action**: Update transfer tracking to include the new agent.

**Payload Structure**:

```json
{
  "message": "notification.transferAgentAdded",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-789",
    "agent": {
      "id": "agent-789",
      "for": "did:web:intermediary.com",
      "role": "VASP"
    }
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field        | Type   | Required | Description                       |
| ------------ | ------ | -------- | --------------------------------- |
| `for`        | string | Yes      | Entity DID receiving this webhook |
| `id`         | string | Yes      | Transfer ID                       |
| `agentID`    | string | Yes      | ID of the added agent             |
| `agent`      | object | Yes      | Agent details                     |
| `agent.id`   | string | No       | Agent identifier                  |
| `agent.for`  | string | No       | Entity DID the agent acts for     |
| `agent.role` | string | No       | Agent role                        |

**Response Required**: None (informational only).

**Agent Roles**: `VASP`, `Custodian`, `SettlementAddress`, `SourceAddress`, `Gateway`

---

#### notification.transferAgentReplaced

**Purpose**: Notifies that an agent in a transfer has been replaced with a different agent.

**When Sent**: When an intermediary or service provider is changed during transfer processing.

**Expected Action**: Update your records to reflect the agent change.

**Payload Structure**:

```json
{
  "message": "notification.transferAgentReplaced",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "fromAgentID": "old-agent-123",
    "toAgentID": "new-agent-456",
    "toAgent": {
      "id": "new-agent-456",
      "for": "did:web:new-intermediary.com",
      "role": "VASP"
    }
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field          | Type   | Required | Description                         |
| -------------- | ------ | -------- | ----------------------------------- |
| `for`          | string | Yes      | Entity DID receiving this webhook   |
| `id`           | string | Yes      | Transfer ID                         |
| `fromAgentID`  | string | Yes      | ID of the agent being replaced      |
| `toAgentID`    | string | Yes      | ID of the new agent                 |
| `toAgent`      | object | Yes      | New agent details                   |
| `toAgent.id`   | string | No       | Agent identifier                    |
| `toAgent.for`  | string | No       | Entity DID the agent acts for       |
| `toAgent.role` | string | No       | Agent role                          |

**Response Required**: None (informational only).

---

#### notification.transferAgentStatusChanged

**Purpose**: Notifies that an agent's status within a transfer has changed.

**When Sent**: When an agent's processing state changes (e.g., from "processing" to "completed").

**Expected Action**: Update agent status tracking.

**Payload Structure**:

```json
{
  "message": "notification.transferAgentStatusChanged",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-456",
    "fromStatus": "processing",
    "toStatus": "completed"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field        | Type   | Required | Description                       |
| ------------ | ------ | -------- | --------------------------------- |
| `for`        | string | Yes      | Entity DID receiving this webhook |
| `id`         | string | Yes      | Transfer ID                       |
| `agentID`    | string | Yes      | Agent identifier                  |
| `fromStatus` | string | Yes      | Previous agent status             |
| `toStatus`   | string | Yes      | New agent status                  |

**Response Required**: None (informational only).

**Common Agent Statuses**: `processing`, `completed`, `failed`, `pending`

---

### TAP Policy Requirement Events

#### tap.requirePresentationRequested

**Purpose**: Requests the recipient to provide required credentials or documents for travel rule compliance.

**When Sent**: When a transfer requires additional documentation (identity verification, transaction details, etc.) to proceed.

**Expected Action**: Your system should facilitate the presentation of the required credentials to continue the transfer process.

**Payload Structure**:

```json
{
  "message": "tap.requirePresentationRequested",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "originatorPresentationDefinition": "...",
    "beneficiaryPresentationDefinition": "...",
    "callbackUrl": "/entities/did%3Aweb%3Aexample.com/transfers/transfer-123/presentations/callback",
    "encryptionKey": "...",
    "policyId": "policy-789"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                                | Type   | Required | Description                                                        |
| ------------------------------------ | ------ | -------- | ------------------------------------------------------------------ |
| `for`                                | string | Yes      | Entity DID receiving this webhook                                  |
| `id`                                 | string | Yes      | Transfer ID                                                        |
| `originatorAgentID`                  | string | Yes      | Originator agent identifier                                        |
| `beneficiaryAgentID`                 | string | No       | Beneficiary agent identifier (included when beneficiary presentation is required) |
| `originatorPresentationDefinition`   | string | Yes      | Presentation definition for originator credentials                 |
| `beneficiaryPresentationDefinition`  | string | No       | Presentation definition for beneficiary credentials (only present when beneficiary data is required) |
| `callbackUrl`                        | string | Yes      | Relative URL path to POST presentation data                        |
| `encryptionKey`                      | object | No       | Encryption key for securing presentation data                      |
| `policyId`                           | string | Yes      | Policy identifier                                                  |

**Response Required**: POST presentation data to the `callbackUrl` to satisfy the requirement.

---

#### tap.requireAuthorizationRequested

**Purpose**: Requests authorization decision for a transfer (approve or reject).

**When Sent**: When a transfer requires manual approval due to compliance policies, risk factors, or business rules.

**Expected Action**: Review the transfer details and provide an authorization decision (approve/reject).

**Payload Structure**:

```json
{
  "message": "tap.requireAuthorizationRequested",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "authorizeCallbackUrl": "/entities/did%3Aweb%3Aexample.com/transfers/transfer-123/authorize",
    "rejectCallbackUrl": "/entities/did%3Aweb%3Aexample.com/transfers/transfer-123/reject",
    "purpose": "High-value transfer requires manual review"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                  | Type   | Required | Description                                      |
| ---------------------- | ------ | -------- | ------------------------------------------------ |
| `for`                  | string | Yes      | Entity DID receiving this webhook                |
| `id`                   | string | Yes      | Transfer ID                                      |
| `originatorAgentID`    | string | Yes      | Originator agent identifier                      |
| `beneficiaryAgentID`   | string | Yes      | Beneficiary agent identifier                     |
| `authorizeCallbackUrl` | string | Yes      | Relative URL path to POST approval               |
| `rejectCallbackUrl`    | string | Yes      | Relative URL path to POST rejection              |
| `purpose`              | string | No       | Reason the authorization is required             |

**Response Required**: POST to either `authorizeCallbackUrl` (to approve) or `rejectCallbackUrl` (to reject) the transfer.

> **Note**: The callback URLs are relative paths. Prepend the Notabene API base URL (e.g., `https://api.notabene.id`) to form the full endpoint.

---

#### tap.requireRelationshipConfirmationRequested

**Purpose**: Requests confirmation of the relationship between entities in a transfer.

**When Sent**: When the system needs to verify that the relationship between originator and beneficiary entities is legitimate.

**Expected Action**: Confirm the relationship exists and is valid for this transfer.

**Payload Structure**:

```json
{
  "message": "tap.requireRelationshipConfirmationRequested",
  "payload": {
    "for": "did:web:example.com",
    "from": "did:web:counterparty.com",
    "id": "transfer-123",
    "originatorAgentID": "originator-agent-123",
    "beneficiaryAgentID": "beneficiary-agent-456",
    "confirmCallbackUrl": "/entities/did%3Aweb%3Aexample.com/relationships?from=did%3Aweb%3Acounterparty.com&to=confirmed",
    "purpose": "Confirm relationship between entities"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                | Type   | Required | Description                                    |
| -------------------- | ------ | -------- | ---------------------------------------------- |
| `for`                | string | Yes      | Entity DID receiving this webhook              |
| `from`               | string | Yes      | DID of the counterparty agent                  |
| `id`                 | string | Yes      | Transfer ID                                    |
| `originatorAgentID`  | string | Yes      | Originator agent identifier                    |
| `beneficiaryAgentID` | string | Yes      | Beneficiary agent identifier                   |
| `confirmCallbackUrl` | string | Yes      | Relative URL path to POST confirmation         |
| `purpose`            | string | No       | Reason the confirmation is required            |

**Response Required**: POST confirmation to `confirmCallbackUrl`.

> **Note**: The callback URL is a relative path. Prepend the Notabene API base URL to form the full endpoint.

---

### TAP Policy Satisfaction Events

#### tap.requireAuthorizationSatisfied

**Purpose**: Confirms that a previously requested authorization has been completed.

**When Sent**: After an authorization decision has been made and processed (following a tap.requireAuthorizationRequested event).

**Expected Action**: Optional - log the completion, update audit trails.

**Payload Structure**:

```json
{
  "message": "tap.requireAuthorizationSatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-456"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field     | Type   | Required | Description                       |
| --------- | ------ | -------- | --------------------------------- |
| `for`     | string | Yes      | Entity DID receiving this webhook |
| `id`      | string | Yes      | Transfer ID                       |
| `agentID` | string | Yes      | Agent that satisfied the requirement |

**Response Required**: None (informational only).

---

#### tap.requirePresentationSatisfied

**Purpose**: Confirms that a previously requested presentation has been completed. Includes callback URLs for completing beneficiary self-policy checks (beneficiary name matching and internal checks).

**When Sent**: After required credentials/documents have been successfully presented and PII validation has passed (following a tap.requirePresentationRequested event).

**Expected Action**: Use the provided callback URLs to confirm beneficiary name matching and internal checks.

**Payload Structure**:

```json
{
  "message": "tap.requirePresentationSatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "agentID": "agent-456",
    "confirmBeneficiaryCallbackUrl": "/entities/did%3Aweb%3Aexample.com/tx/transfer-123/checks/beneficiary-name-matching",
    "confirmInternalChecksCallbackUrl": "/entities/did%3Aweb%3Aexample.com/tx/transfer-123/checks/internal-checks"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                              | Type   | Required | Description                                         |
| ---------------------------------- | ------ | -------- | --------------------------------------------------- |
| `for`                              | string | Yes      | Entity DID receiving this webhook                   |
| `id`                               | string | Yes      | Transfer ID                                         |
| `agentID`                          | string | Yes      | Agent that satisfied the requirement                |
| `confirmBeneficiaryCallbackUrl`    | string | Yes      | Relative URL path to confirm beneficiary name matching |
| `confirmInternalChecksCallbackUrl` | string | Yes      | Relative URL path to confirm internal checks        |

**Response Required**: POST to `confirmBeneficiaryCallbackUrl` and `confirmInternalChecksCallbackUrl` to complete the self-policy checks. Both checks must be completed before the transfer can proceed to authorization.

> **Note**: The callback URLs are relative paths. Prepend the Notabene API base URL to form the full endpoint.

---

#### tap.requireRelationshipConfirmationSatisfied

**Purpose**: Confirms that a previously requested relationship confirmation has been completed.

**When Sent**: After entity relationships have been successfully confirmed (following a tap.requireRelationshipConfirmationRequested event).

**Expected Action**: Optional - log the completion, update relationship records.

**Payload Structure**:

```json
{
  "message": "tap.requireRelationshipConfirmationSatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "relationshipId": "relationship-789",
    "agentID": "agent-456",
    "from": "did:web:counterparty.com",
    "to": "confirmed",
    "status": "confirmed"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field            | Type   | Required | Description                       |
| ---------------- | ------ | -------- | --------------------------------- |
| `for`            | string | Yes      | Entity DID receiving this webhook |
| `id`             | string | Yes      | Transfer ID                       |
| `relationshipId` | string | Yes      | Relationship identifier           |
| `agentID`        | string | Yes      | Agent that confirmed the relationship |
| `from`           | string | Yes      | Source entity DID                 |
| `to`             | string | Yes      | Target/confirmation status        |
| `status`         | string | Yes      | Relationship confirmation status  |

**Response Required**: None (informational only).

---

### TAP Policy Status Events

#### tap.policySatisfied

**Purpose**: Notifies that a policy has been fully satisfied for a transfer.

**When Sent**: When all requirements of a policy have been met and the policy evaluation is complete.

**Expected Action**: Optional - log the completion, update compliance tracking.

**Payload Structure**:

```json
{
  "message": "tap.policySatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "policyType": "RequirePresentation"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field        | Type   | Required | Description                                              |
| ------------ | ------ | -------- | -------------------------------------------------------- |
| `for`        | string | Yes      | Entity DID receiving this webhook                        |
| `id`         | string | Yes      | Transfer ID                                              |
| `policyType` | string | Yes      | Type of policy that was satisfied (e.g., `RequirePresentation`, `RequireAuthorization`, `RequireRelationshipConfirmation`) |

**Response Required**: None (informational only).

---

#### tap.requirePresentationPartiallySatisfied

**Purpose**: Notifies that a presentation requirement has been partially satisfied. Some but not all of the requested credentials have been received.

**When Sent**: When a subset of the requested credentials/documents has been presented, but additional credentials are still needed to fully satisfy the requirement.

**Expected Action**: Optional - track presentation progress. The system will continue to collect remaining credentials. A `tap.requirePresentationSatisfied` event will follow once all credentials are received.

**Payload Structure**:

```json
{
  "message": "tap.requirePresentationPartiallySatisfied",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "policyId": "policy-789"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field       | Type   | Required | Description                                     |
| ----------- | ------ | -------- | ----------------------------------------------- |
| `for`       | string | Yes      | Entity DID receiving this webhook               |
| `id`        | string | Yes      | Transfer ID                                     |
| `policyId`  | string | Yes      | UUID of the policy being partially satisfied    |

**Response Required**: None (informational only).

---

### Flow Payment Events

Flow events are sent for Notabene Flow payment processing workflows. Events are differentiated by transaction type (payin vs payout) to allow subscribers to filter based on their needs.

---

#### flow.payin.created / flow.payout.created

**Purpose**: Notifies that a new Flow payment request has been created.

**When Sent**: Immediately after a payin or payout request is successfully created through the Flow API, or when a Payment message is received via DIDComm (for responders).

**Expected Action**: Optional - log the event, trigger business workflows, or update internal records.

**Payload Structure**:

```json
{
  "message": "flow.payin.created",
  "payload": {
    "for": "did:web:example.com",
    "initiatedBy": "did:web:merchant.com",
    "id": "transfer-123",
    "transactionType": "PAYIN",
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com",
    "customer": {
      "@id": "did:email:customer@example.com",
      "name": "John Doe",
      "email": "customer@example.com"
    },
    "merchant": {
      "@id": "did:web:merchant.com",
      "name": "Example Merchant"
    },
    "paymentLink": "https://connect.notabene.id/payin/abc123",
    "amount": "500.00",
    "currency": "USD",
    "supportedAssets": [
      "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "eip155:137/erc20:0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"
    ],
    "fallbackSettlementAddresses": [
      "eip155:1:0x742d35Cc6634C0532925a3b8D16e3E7B9F4F1234",
      "eip155:137:0x742d35Cc6634C0532925a3b8D16e3E7B9F4F1234"
    ],
    "memo": "Payment for Order #12345",
    "invoice": {
      "id": "INV-2025-001",
      "dueDate": "2025-12-31"
    },
    "expiry": "2025-12-31T23:59:59Z",
    "agents": [
      { "@id": "did:web:wallet.com", "role": "SettlementAddress", "for": "did:web:merchant.com" }
    ]
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                        | Type     | Required | Description                                                |
| ---------------------------- | -------- | -------- | ---------------------------------------------------------- |
| `for`                        | string   | Yes      | Entity DID receiving this webhook                          |
| `initiatedBy`                | string   | Yes      | Entity DID that created the payment                        |
| `id`                         | string   | Yes      | Transfer ID                                                |
| `transactionType`            | string   | Yes      | `PAYIN` or `PAYOUT`                                        |
| `customerDid`                | string   | Yes      | Customer's DID                                             |
| `merchantDid`                | string   | Yes      | Merchant's DID                                             |
| `customer`                   | object   | Yes      | Customer details (`@id`, optional `name`, `email`)         |
| `merchant`                   | object   | Yes      | Merchant details (`@id`, optional `name`)                  |
| `paymentLink`                | string   | No       | URL for customer payment interaction                       |
| `amount`                     | string   | No       | Payment amount                                             |
| `currency`                   | string   | No       | Currency code (e.g., `USD`)                                |
| `supportedAssets`            | string[] | No       | CAIP-19 asset identifiers merchant accepts                 |
| `fallbackSettlementAddresses`| string[] | No       | CAIP-10 settlement addresses if responder doesn't provide  |
| `memo`                       | string   | No       | Payment memo/description                                   |
| `invoice`                    | object   | No       | Invoice details (structure varies)                         |
| `expiry`                     | string   | No       | ISO 8601 timestamp when payment request expires            |
| `agents`                     | array    | No       | Transfer agents (`@id`, `role`, `for`)                     |

**Response Required**: None (informational only).

---

#### flow.payin.opened / flow.payout.opened

**Purpose**: Notifies that a customer has viewed a Flow payment link.

**When Sent**: When a customer accesses the payment link URL.

**Expected Action**: Optional - track customer engagement or trigger follow-up workflows.

**Payload Structure**:

```json
{
  "message": "flow.payout.opened",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "transactionType": "PAYOUT",
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field             | Type   | Required | Description                       |
| ----------------- | ------ | -------- | --------------------------------- |
| `for`             | string | Yes      | Entity DID receiving this webhook |
| `id`              | string | Yes      | Transfer ID                       |
| `transactionType` | string | Yes      | `PAYIN` or `PAYOUT`               |
| `customerDid`     | string | Yes      | Customer's DID                    |
| `merchantDid`     | string | Yes      | Merchant's DID                    |

**Response Required**: None (informational only).

---

#### flow.payin.assetSelected / flow.payout.assetSelected

**Purpose**: Notifies that a customer has selected an asset for the Flow payment.

**When Sent**: When customer chooses an asset from the payment link interface.

**Expected Action**: Optional - track payment progress or prepare for settlement.

**Payload Structure**:

```json
{
  "message": "flow.payin.assetSelected",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "transactionType": "PAYIN",
    "selectedAsset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "selectedSettlementAddress": "0x742d35Cc6634C0532925a3b8D16e3E7B9F4F1234",
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                      | Type   | Required | Description                             |
| -------------------------- | ------ | -------- | --------------------------------------- |
| `for`                      | string | Yes      | Entity DID receiving this webhook       |
| `id`                       | string | Yes      | Transfer ID                             |
| `transactionType`          | string | Yes      | `PAYIN` or `PAYOUT`                     |
| `selectedAsset`            | string | Yes      | CAIP-19 asset identifier selected       |
| `selectedSettlementAddress`| string | Yes      | Settlement address for the selected asset |
| `customerDid`              | string | Yes      | Customer's DID                          |
| `merchantDid`              | string | Yes      | Merchant's DID                          |

**Response Required**: None (informational only).

---

#### flow.payin.agentAdded / flow.payout.agentAdded

**Purpose**: Notifies that an agent has been added to a Flow transfer.

**When Sent**: When a wallet or intermediary agent is added to the Flow transfer.

**Expected Action**: Optional - update transfer tracking with agent information.

**Payload Structure**:

```json
{
  "message": "flow.payout.agentAdded",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "transactionType": "PAYOUT",
    "agentID": "agent-789",
    "agent": {
      "id": "agent-789",
      "for": "did:web:wallet.com",
      "role": "VASP"
    },
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field             | Type   | Required | Description                       |
| ----------------- | ------ | -------- | --------------------------------- |
| `for`             | string | Yes      | Entity DID receiving this webhook |
| `id`              | string | Yes      | Transfer ID                       |
| `transactionType` | string | Yes      | `PAYIN` or `PAYOUT`               |
| `agentID`         | string | Yes      | Agent identifier                  |
| `agent`           | object | Yes      | Agent details                     |
| `agent.id`        | string | No       | Agent identifier                  |
| `agent.for`       | string | No       | Entity DID the agent acts for     |
| `agent.role`      | string | No       | Agent role                        |
| `customerDid`     | string | Yes      | Customer's DID                    |
| `merchantDid`     | string | Yes      | Merchant's DID                    |

**Response Required**: None (informational only).

---

#### flow.payin.settlementAddressSelected / flow.payout.settlementAddressSelected

**Purpose**: Notifies that a settlement address has been selected for the Flow transfer.

**When Sent**: When a blockchain address is selected for receiving settlement funds.

**Expected Action**: Optional - prepare for settlement or update payment records.

**Payload Structure**:

```json
{
  "message": "flow.payin.settlementAddressSelected",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "transactionType": "PAYIN",
    "settlementAddress": "0x742d35Cc6634C0532925a3b8D16e3E7B9F4F1234",
    "selectedAsset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field               | Type   | Required | Description                             |
| ------------------- | ------ | -------- | --------------------------------------- |
| `for`               | string | Yes      | Entity DID receiving this webhook       |
| `id`                | string | Yes      | Transfer ID                             |
| `transactionType`   | string | Yes      | `PAYIN` or `PAYOUT`                     |
| `settlementAddress` | string | Yes      | Selected settlement address             |
| `selectedAsset`     | string | Yes      | CAIP-19 asset identifier                |
| `customerDid`       | string | Yes      | Customer's DID                          |
| `merchantDid`       | string | Yes      | Merchant's DID                          |

**Response Required**: None (informational only).

---

#### flow.payin.authorizationRequired / flow.payout.authorizationRequired

**Purpose**: Notifies that a Flow payment requires authorization.

**When Sent**: When a Flow payment needs approval before proceeding.

**Expected Action**: Review and authorize or reject the payment.

**Payload Structure**:

```json
{
  "message": "flow.payout.authorizationRequired",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "transactionType": "PAYOUT",
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com",
    "authorizationUrl": "https://api.notabene.id/v1/flow/authorize/abc123",
    "purpose": "High-value payout requires approval"
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field              | Type   | Required | Description                                |
| ------------------ | ------ | -------- | ------------------------------------------ |
| `for`              | string | Yes      | Entity DID receiving this webhook          |
| `id`               | string | Yes      | Transfer ID                                |
| `transactionType`  | string | Yes      | `PAYIN` or `PAYOUT`                        |
| `customerDid`      | string | Yes      | Customer's DID                             |
| `merchantDid`      | string | Yes      | Merchant's DID                             |
| `authorizationUrl` | string | No       | URL to authorize or reject the payment     |
| `purpose`          | string | No       | Reason the authorization is required       |

**Response Required**: Authorize or reject via the provided authorization URL.

---

#### flow.payin.fundingAdded / flow.payout.fundingAdded

**Purpose**: Notifies that funding address information has been added to a Flow payment. This occurs when a party provides the address where funds should be sent or received.

**When Sent**: When funding address details are added via a TAP Fund message, typically after asset selection when settlement details are finalized.

**Expected Action**: Update payment records with funding details. For payins, this indicates where the customer should send funds. For payouts, this indicates where funds will be sent.

**Payload Structure**:

```json
{
  "message": "flow.payin.fundingAdded",
  "payload": {
    "for": "did:web:example.com",
    "id": "transfer-123",
    "transactionType": "PAYIN",
    "customerDid": "did:email:customer@example.com",
    "merchantDid": "did:web:merchant.com",
    "fundingAddress": {
      "amount": "500.00",
      "currency": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "fundingAddress": "eip155:1:0x742d35Cc6634C0532925a3b8D16e3E7B9F4F1234",
      "for": "did:web:merchant.com",
      "fee": "0.5",
      "addedBy": "did:web:wallet.com",
      "addedAt": "2025-12-23T10:30:00Z"
    }
  },
  "version": "1.0.0"
}
```

**Payload Fields**:

| Field                           | Type   | Required | Description                                           |
| ------------------------------- | ------ | -------- | ----------------------------------------------------- |
| `for`                           | string | Yes      | Entity DID receiving this webhook                     |
| `id`                            | string | Yes      | Transfer ID                                           |
| `transactionType`               | string | Yes      | `PAYIN` or `PAYOUT`                                   |
| `customerDid`                   | string | Yes      | Customer's DID                                        |
| `merchantDid`                   | string | Yes      | Merchant's DID                                        |
| `fundingAddress`                | object | Yes      | Funding address details                               |
| `fundingAddress.amount`         | string | Yes      | Amount to be funded                                   |
| `fundingAddress.currency`       | string | Yes      | Currency/asset (CAIP-19 format for crypto)            |
| `fundingAddress.fundingAddress` | string | No       | Crypto address (CAIP-10 format)                       |
| `fundingAddress.fundingAccount` | string | No       | Fiat account (PayTo URI format)                       |
| `fundingAddress.for`            | string | Yes      | DID of party receiving the funds                      |
| `fundingAddress.fee`            | string | No       | Fee percentage (e.g., "0.5" for 0.5%)                 |
| `fundingAddress.addedBy`        | string | Yes      | DID of agent who added funding info                   |
| `fundingAddress.addedAt`        | string | Yes      | ISO 8601 timestamp when funding info was added        |

**Notes**:
- Either `fundingAddress` (for crypto) or `fundingAccount` (for fiat) will be present, not both
- The `for` field indicates who will receive funds at this address

**Response Required**: None (informational only).

---

## Flow Payment Events (Planned - Not Yet Implemented)

The following Flow events are planned for future implementation:

### Flow Settlement Events

- **flow.payin.settled**: Payin completed successfully
- **flow.payout.settled**: Payout completed successfully

### Flow Customer Events

- **flow.payin.customerApproved**: Customer approved the payment
- **flow.payin.customerRejected**: Customer rejected the payment
- **flow.payout.customerApproved**: Customer approved the payout
- **flow.payout.customerRejected**: Customer rejected the payout

### Flow Batch Processing Events

- **flow.batch.created**: Batch processing started
- **flow.batch.completed**: Batch processing finished successfully
- **flow.batch.partialFailure**: Some items in batch failed

### TAIP-15 Connection Events (Partially Implemented)

The following connection events have internal type definitions and transformers but are not yet wired to the webhook delivery pipeline:

- **connection.requested**: Connection request initiated
- **connection.authorized**: Connection approved by customer
- **connection.rejected**: Connection rejected by customer
- **connection.cancelled**: Connection terminated
- **connection.updated**: Connection resource updated

Planned but not yet implemented:

- **flow.connection.limitExceeded**: Transaction rejected due to limits
- **flow.connection.expired**: Connection expired

---

## Common Fields

All webhook `payload` objects include these standard fields:

- **`for`**: String DID of the entity this webhook is intended for
- **`id`**: String transfer ID or transaction identifier (except connection events which use `connectionId`)
- **`agentID`** (when applicable): String identifier of the agent involved
- **Timestamp fields**: Automatically added by Svix to the HTTP headers (e.g., `svix-timestamp`, `svix-id`, etc.)

The webhook envelope always includes:

- **`message`**: The event type string (e.g., `notification.transferCreated`)
- **`payload`**: The event-specific data
- **`version`**: Currently `"1.0.0"`

## Error Handling

### Webhook Delivery Failures

Svix automatically handles webhook delivery failures with:

- **Exponential backoff**: Increasing delays between retry attempts
- **Multiple retries**: Up to several attempts over 3 days
- **Dead letter queue**: Failed webhooks are available in Svix dashboard

### Recommended Error Handling

```javascript
app.post("/notabene-webhook", async (req, res) => {
  try {
    // Verify signature first
    const verified = webhook.verify(req.body, req.headers);

    // The verified object has the envelope structure
    const { message, payload, version } = verified;

    // Process the webhook based on the message (event type)
    await processWebhook(message, payload);

    // Return success
    res.status(200).send("OK");
  } catch (error) {
    console.error("Webhook processing failed:", error);

    // Return error status to trigger Svix retry
    res.status(500).send("Processing failed");
  }
});
```

### HTTP Response Codes

- **200-299**: Success - webhook processed successfully
- **400-499**: Client error - webhook will NOT be retried
- **500-599**: Server error - webhook will be retried

## Testing

### Webhook Testing Tools

1. **Svix Dashboard**: View webhook delivery status and payloads
2. **ngrok**: Create public tunnels for local development
3. **webhook.site**: Simple webhook testing service
4. **Postman/Insomnia**: Test webhook endpoints manually

### Example Test Setup

```javascript
// Test webhook endpoint
app.post("/test-webhook", (req, res) => {
  console.log("Received webhook:", JSON.stringify(req.body, null, 2));
  console.log("Headers:", req.headers);
  res.status(200).send("Received");
});
```

### Development Tips

1. **Always verify signatures** in production
2. **Handle idempotency** - webhooks may be delivered multiple times
3. **Log webhook payloads** for debugging and audit purposes
4. **Return 200 status** as quickly as possible to avoid timeouts
5. **Process webhooks asynchronously** if they trigger complex operations
6. **Monitor webhook failures** in the Svix dashboard

## Integration Examples

### Node.js Express Example

```javascript
import express from "express";
import { Webhook } from "svix";

const app = express();
const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.use("/webhook", express.raw({ type: "application/json" }));

app.post("/webhook", async (req, res) => {
  const rawPayload = req.body;
  const headers = req.headers;

  try {
    // Verify the webhook signature
    const event = webhook.verify(rawPayload, headers);

    // event has the envelope structure: { message, payload, version }
    const { message, payload } = event;

    // Handle different event types using the `message` field
    switch (message) {
      case "tap.requireAuthorizationRequested":
        await handleAuthorizationRequest(payload);
        break;
      case "notification.transferStatusChanged":
        await handleStatusChange(payload);
        break;
      case "flow.payin.created":
        await handleFlowPayinCreated(payload);
        break;
      case "flow.payout.created":
        await handleFlowPayoutCreated(payload);
        break;
      // ... handle other event types
    }

    res.status(200).send("OK");
  } catch (err) {
    console.error("Webhook signature verification failed:", err);
    res.status(400).send("Invalid signature");
  }
});

async function handleAuthorizationRequest(payload) {
  // Implement your authorization logic
  console.log(`Authorization required for transfer ${payload.id}`);

  // Make decision and callback to Notabene
  const decision = await makeAuthorizationDecision(payload);

  // Prepend the Notabene API base URL to the relative callback paths
  const baseUrl = "https://api.notabene.id";

  if (decision.approved) {
    await fetch(`${baseUrl}${payload.authorizeCallbackUrl}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ decision: "approved", reason: decision.reason }),
    });
  } else {
    await fetch(`${baseUrl}${payload.rejectCallbackUrl}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ decision: "rejected", reason: decision.reason }),
    });
  }
}

async function handleFlowPayinCreated(payload) {
  console.log(`Flow payin created: ${payload.id}`);
  console.log(`Customer: ${payload.customerDid}`);
  console.log(`Payment link: ${payload.paymentLink}`);
  // Update your system with the new payin request
}

async function handleFlowPayoutCreated(payload) {
  console.log(`Flow payout created: ${payload.id}`);
  console.log(`Customer: ${payload.customerDid}`);
  console.log(`Payment link: ${payload.paymentLink}`);
  // Update your system with the new payout request
}
```

## Support

For webhook integration support:

- **Documentation**: Review this guide and the Svix documentation
- **Dashboard**: Use the Svix dashboard to monitor webhook delivery
- **Support**: Contact Notabene support for integration assistance
- **Status**: Check webhook delivery status in the Svix interface

---

_This documentation covers all webhook events currently supported by the Notabene platform. Webhook payloads and event types may be extended in future versions._
