# TAP Go Implementation (`tap-go`)

**Repository:** [TransactionAuthorizationProtocol/tap-go](https://github.com/TransactionAuthorizationProtocol/tap-go)
**Module:** `github.com/TransactionAuthorizationProtocol/tap-go`
**License:** MIT

A Go library providing typed wrappers for all 20 TAP message types, built on top of [`github.com/Notabene-id/go-didcomm`](https://github.com/Notabene-id/go-didcomm) for DIDComm v2.

---

## Install

```bash
go get github.com/TransactionAuthorizationProtocol/tap-go
```

## Dependencies

- `github.com/Notabene-id/go-didcomm` — DIDComm v2 message handling (packing, unpacking, signing, encryption, DID resolution)
- `github.com/google/uuid` — message ID generation

---

## Package Structure

Single flat `tap` package. One `.go` file per message type with corresponding tests. A CLI binary lives at `cmd/tap/`.

---

## Core Interface

All 20 TAP body structs implement the `TAPBody` interface:

```go
type TAPBody interface {
    TAPType() string
}
```

### Client

Wraps `didcomm.Client` to handle DIDComm envelope + TAP body parsing:

```go
type Client struct {
    DIDComm *didcomm.Client
}

func NewClient(dc *didcomm.Client) *Client

// Receive unpacks a DIDComm envelope and parses the TAP body
func (c *Client) Receive(ctx context.Context, envelope []byte) (*TAPResult, error)
```

### TAPResult

```go
type TAPResult struct {
    Message   *didcomm.Message
    Body      TAPBody
    Encrypted bool
    Signed    bool
    Anonymous bool
}
```

### Parsing

```go
// Parse a DIDComm message into a typed TAP body
func ParseBody(msg *didcomm.Message) (TAPBody, error)

// Check if a message is a TAP message
func IsTAPMessage(msg *didcomm.Message) bool

// List all TAP type URIs
func AllTypes() []string
```

---

## Message Types and Constructors

Every constructor validates required fields, sets `@context` and `@type` automatically, and returns a `*didcomm.Message`.

**Initiating messages** (no `thid` — they start a new thread):

| Constructor | Body Struct | TAIP |
|-------------|------------|------|
| `NewTransferMessage(from, to, body)` | `TransferBody` | TAIP-3 |
| `NewPaymentMessage(from, to, body)` | `PaymentBody` | TAIP-14 |
| `NewRFQMessage(from, to, body)` | `RFQBody` | TAIP-18 (renamed from `NewExchangeMessage`/`ExchangeBody` 2026-05-01) |
| `NewLockMessage(from, to, body)` | `LockBody` | TAIP-17 (renamed from `NewEscrowMessage`/`EscrowBody` 2026-05-01) |
| `NewConnectMessage(from, to, body)` | `ConnectBody` | TAIP-15 |

**Reply messages** (require `thid` for thread correlation):

| Constructor | Body Struct | TAIP |
|-------------|------------|------|
| `NewAuthorizeMessage(from, to, thid, body)` | `AuthorizeBody` | TAIP-4 |
| `NewSettleMessage(from, to, thid, body)` | `SettleBody` | TAIP-4 |
| `NewRejectMessage(from, to, thid, body)` | `RejectBody` | TAIP-4 |
| `NewCancelMessage(from, to, thid, body)` | `CancelBody` | TAIP-4 |
| `NewRevertMessage(from, to, thid, body)` | `RevertBody` | TAIP-4 |
| `NewAuthorizationRequiredMessage(from, to, thid, body)` | `AuthorizationRequiredBody` | TAIP-4 |
| `NewCaptureMessage(from, to, thid, body)` | `CaptureBody` | TAIP-17 |
| `NewQuoteMessage(from, to, thid, body)` | `QuoteBody` | TAIP-18 |
| `NewAddAgentsMessage(from, to, thid, body)` | `AddAgentsBody` | TAIP-5 |
| `NewReplaceAgentMessage(from, to, thid, body)` | `ReplaceAgentBody` | TAIP-5 |
| `NewRemoveAgentMessage(from, to, thid, body)` | `RemoveAgentBody` | TAIP-5 |
| `NewUpdateAgentMessage(from, to, thid, body)` | `UpdateAgentBody` | TAIP-5 |
| `NewUpdatePartyMessage(from, to, thid, body)` | `UpdatePartyBody` | TAIP-6 |
| `NewUpdatePoliciesMessage(from, to, thid, body)` | `UpdatePoliciesBody` | TAIP-7 |
| `NewConfirmRelationshipMessage(from, to, thid, body)` | `ConfirmRelationshipBody` | TAIP-9 |

---

## Shared Data Types (`types.go`)

### Party

```go
type Party struct {
    ID       string             `json:"@id"`
    Type     string             `json:"@type,omitempty"`
    Name     string             `json:"name,omitempty"`
    MCC      string             `json:"mcc,omitempty"`
    LEICode  string             `json:"leiCode,omitempty"`
    IVMS101  *json.RawMessage   `json:"ivms101,omitempty"`
    Account  string             `json:"account,omitempty"`
    URL      string             `json:"url,omitempty"`
    Email    string             `json:"email,omitempty"`
}
```

### Agent

```go
type Agent struct {
    ID         string    `json:"@id"`
    Type       string    `json:"@type,omitempty"`
    Role       string    `json:"role,omitempty"`
    For        ForField  `json:"for,omitempty"`
    Policies   []Policy  `json:"policies,omitempty"`
    ServiceURL string    `json:"serviceUrl,omitempty"`
    Name       string    `json:"name,omitempty"`
    URL        string    `json:"url,omitempty"`
    LEICode    string    `json:"leiCode,omitempty"`
}
```

### ForField

Custom type that handles JSON marshal/unmarshal of a single DID string or array of DIDs:

```go
// Single party
agent.For = NewForField("did:web:alice.example.com")

// Multiple parties
agent.For = NewForField("did:web:alice.example.com", "did:web:bob.example.com")
```

### Policy

```go
type Policy struct {
    Type                   string   `json:"@type"`
    Context                string   `json:"@context,omitempty"`
    PresentationDefinition string   `json:"presentationDefinition,omitempty"`
    Codes                  []string `json:"codes,omitempty"`
    Nonce                  string   `json:"nonce,omitempty"`
}
```

### TransactionConstraints

```go
type TransactionConstraints struct {
    Purposes                   []string `json:"purposes,omitempty"`
    Limits                     *Limits  `json:"limits,omitempty"`
    AllowedBeneficiaries       []string `json:"allowedBeneficiaries,omitempty"`
    AllowedAssets              []string `json:"allowedAssets,omitempty"`
    AllowedSettlementAddresses []string `json:"allowedSettlementAddresses,omitempty"`
}

type Limits struct {
    PerTransaction string `json:"per_transaction,omitempty"`
    Day            string `json:"day,omitempty"`
    Week           string `json:"week,omitempty"`
    Month          string `json:"month,omitempty"`
    Year           string `json:"year,omitempty"`
    Currency       string `json:"currency"`
}
```

### Invoice (TAIP-16)

```go
type Invoice struct {
    ID           string     `json:"id"`
    IssueDate    string     `json:"issueDate"`
    CurrencyCode string     `json:"currencyCode"`
    Total        string     `json:"total"`
    LineItems    []LineItem `json:"lineItems"`
    SubTotal     string     `json:"subTotal,omitempty"`
    TaxTotal     string     `json:"taxTotal,omitempty"`
    DueDate      string     `json:"dueDate,omitempty"`
    Note         string     `json:"note,omitempty"`
}
```

---

## Error Types

```go
var ErrInvalidBody      = errors.New("invalid body")       // missing required fields
var ErrUnknownMessageType = errors.New("unknown message type")
```

---

## Usage Examples

### Creating a Transfer

```go
import tap "github.com/TransactionAuthorizationProtocol/tap-go"

msg, err := tap.NewTransferMessage(
    "did:web:originator.vasp",
    []string{"did:web:beneficiary.vasp"},
    &tap.TransferBody{
        Asset:  "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        Amount: "1000.00",
        Originator: tap.Party{
            ID:   "did:pkh:eip155:1:0xSender...",
            Type: "Individual",
            Name: "Alice Smith",
        },
        Beneficiary: tap.Party{
            ID:   "did:pkh:eip155:1:0xReceiver...",
            Type: "Individual",
        },
        Agents: []tap.Agent{
            {
                ID:   "did:web:originator.vasp",
                Type: "VASP",
                Role: "OriginatorVASP",
                For:  tap.NewForField("did:pkh:eip155:1:0xSender..."),
            },
        },
    },
)
```

### Parsing a Received Message

```go
body, err := tap.ParseBody(msg)
switch b := body.(type) {
case *tap.TransferBody:
    fmt.Printf("Transfer: %s %s\n", b.Amount, b.Asset)
case *tap.PaymentBody:
    fmt.Printf("Payment: %s\n", b.Amount)
case *tap.AuthorizeBody:
    fmt.Printf("Authorized, settle to: %s\n", b.SettlementAddress)
}
```

### End-to-End with Client

```go
client := tap.NewClient(didcommClient)
result, err := client.Receive(ctx, envelope)

fmt.Printf("Encrypted: %v, Signed: %v\n", result.Encrypted, result.Signed)

switch b := result.Body.(type) {
case *tap.TransferBody:
    // handle incoming transfer
}
```

---

## CLI (`cmd/tap/`)

The repo includes a CLI binary wrapping `go-didcomm` commands with TAP-specific functionality:

```bash
# DID operations
tap did generate-key
tap did generate-web

# Message operations
tap message transfer --from did:web:myvasp.com --to did:web:other.com --body @transfer.json
tap message authorize --from did:web:other.com --to did:web:myvasp.com --thid <transfer-id> --body @auth.json

# DIDComm packing (supports piping)
tap message transfer ... | tap pack authcrypt --from ... --to ...

# Receive and parse
tap receive --key-file keys.json < envelope.json

# Direct send
tap pack signed ... | tap send --url https://tap.other.com/receive
```
