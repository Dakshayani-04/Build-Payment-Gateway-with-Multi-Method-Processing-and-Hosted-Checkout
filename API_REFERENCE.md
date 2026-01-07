# API Reference

## Base URL

```
Development: http://localhost:5000
Production: https://api.yourdomain.com
```

## Authentication

All API requests must include:

```http
X-API-Key: your_api_key
X-API-Secret: your_api_secret
Content-Type: application/json
```

## Response Format

Success Response (200-299):
```json
{
  "success": true,
  "data": { ... },
  "timestamp": "2024-01-07T10:00:00Z"
}
```

Error Response (4xx-5xx):
```json
{
  "success": false,
  "error": "Error message",
  "code": "ERROR_CODE",
  "timestamp": "2024-01-07T10:00:00Z"
}
```

## Endpoints

### Authentication

#### POST /api/auth/login

Merchant login endpoint.

**Request:**
```json
{
  "apiKey": "string",
  "apiSecret": "string"
}
```

**Response (200):**
```json
{
  "token": "jwt_token_here",
  "merchantId": "uuid",
  "email": "merchant@example.com",
  "expiresIn": 3600
}
```

**Errors:**
- 401: Invalid credentials
- 400: Missing required fields

---

### Orders

#### POST /api/orders

Create a new order.

**Request:**
```json
{
  "amount": 50000,
  "currency": "INR",
  "customerId": "cust_123",
  "customerEmail": "customer@example.com",
  "description": "Order for Product",
  "metadata": {
    "key": "value"
  }
}
```

**Response (201):**
```json
{
  "orderId": "order_abc123",
  "amount": 50000,
  "currency": "INR",
  "status": "created",
  "customerId": "cust_123",
  "customerEmail": "customer@example.com",
  "description": "Order for Product",
  "createdAt": "2024-01-07T10:00:00Z",
  "updatedAt": "2024-01-07T10:00:00Z"
}
```

**Errors:**
- 400: Invalid amount (must be > 0)
- 401: Unauthorized

**Curl Example:**
```bash
curl -X POST http://localhost:5000/api/orders \
  -H "X-API-Key: your_api_key" \
  -H "X-API-Secret: your_api_secret" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "currency": "INR",
    "customerId": "cust_123",
    "customerEmail": "customer@example.com",
    "description": "Test Order"
  }'
```

---

#### GET /api/orders/:orderId

Retrieve order details.

**Response (200):**
```json
{
  "orderId": "order_abc123",
  "amount": 50000,
  "currency": "INR",
  "status": "created",
  "customerId": "cust_123",
  "customerEmail": "customer@example.com",
  "createdAt": "2024-01-07T10:00:00Z"
}
```

---

#### GET /api/orders

List all orders for merchant.

**Query Parameters:**
- `status`: created|processing|success|failed (optional)
- `limit`: 1-100 (default: 20)
- `offset`: 0+ (default: 0)
- `sortBy`: createdAt|amount (default: createdAt)
- `sortOrder`: asc|desc (default: desc)

**Response (200):**
```json
{
  "orders": [...],
  "total": 150,
  "limit": 20,
  "offset": 0
}
```

---

### Payments

#### POST /api/payments

Create a payment for an order.

**UPI Payment Request:**
```json
{
  "orderId": "order_abc123",
  "method": "upi",
  "upiId": "customer@upi"
}
```

**Card Payment Request:**
```json
{
  "orderId": "order_abc123",
  "method": "card",
  "cardNumber": "4532015112830366",
  "cardHolderName": "John Doe",
  "expiryMonth": "12",
  "expiryYear": "25",
  "cvv": "123"
}
```

**Response (201):**
```json
{
  "paymentId": "pay_xyz789",
  "orderId": "order_abc123",
  "amount": 50000,
  "currency": "INR",
  "method": "card",
  "status": "processing",
  "createdAt": "2024-01-07T10:00:00Z"
}
```

**Errors:**
- 400: Invalid payment method
- 400: Invalid card number
- 400: Invalid UPI ID
- 404: Order not found
- 409: Payment already exists for order

---

#### GET /api/payments/:paymentId

Retrieve payment details and status.

**Response (200):**
```json
{
  "paymentId": "pay_xyz789",
  "orderId": "order_abc123",
  "amount": 50000,
  "currency": "INR",
  "method": "card",
  "status": "success",
  "cardNetwork": "Visa",
  "lastFourDigits": "0366",
  "createdAt": "2024-01-07T10:00:00Z",
  "updatedAt": "2024-01-07T10:00:05Z"
}
```

---

#### GET /api/payments

List payments for an order.

**Query Parameters:**
- `orderId`: Required
- `status`: processing|success|failed (optional)

**Response (200):**
```json
{
  "payments": [...],
  "total": 5
}
```

---

### Transactions

#### GET /api/transactions

Get transaction history.

**Query Parameters:**
- `limit`: 1-100 (default: 20)
- `offset`: 0+ (default: 0)
- `status`: all|success|failed (default: all)
- `startDate`: ISO 8601 (optional)
- `endDate`: ISO 8601 (optional)

**Response (200):**
```json
{
  "transactions": [
    {
      "transactionId": "txn_123",
      "orderId": "order_abc123",
      "paymentId": "pay_xyz789",
      "amount": 50000,
      "currency": "INR",
      "method": "card",
      "status": "success",
      "createdAt": "2024-01-07T10:00:00Z"
    }
  ],
  "total": 500,
  "successCount": 480,
  "failureCount": 20,
  "totalAmount": 24000000
}
```

---

#### GET /api/transactions/stats

Get transaction statistics.

**Response (200):**
```json
{
  "totalTransactions": 500,
  "successfulTransactions": 480,
  "failedTransactions": 20,
  "totalAmount": 24000000,
  "successAmount": 23500000,
  "failureAmount": 500000,
  "avgAmount": 48000,
  "successRate": 96,
  "dailyVolume": [
    {
      "date": "2024-01-07",
      "count": 50,
      "amount": 2400000
    }
  ]
}
```

---

### Refunds

#### POST /api/refunds

Refund a successful payment.

**Request:**
```json
{
  "paymentId": "pay_xyz789",
  "amount": 50000,
  "reason": "Customer request"
}
```

**Response (201):**
```json
{
  "refundId": "refund_123",
  "paymentId": "pay_xyz789",
  "amount": 50000,
  "status": "initiated",
  "reason": "Customer request",
  "createdAt": "2024-01-07T10:00:00Z"
}
```

---

#### GET /api/refunds/:refundId

Get refund status.

**Response (200):**
```json
{
  "refundId": "refund_123",
  "paymentId": "pay_xyz789",
  "amount": 50000,
  "status": "completed",
  "createdAt": "2024-01-07T10:00:00Z",
  "completedAt": "2024-01-07T10:05:00Z"
}
```

---

## Status Codes

| Code | Meaning |
|------|----------|
| 200 | OK - Request successful |
| 201 | Created - Resource created |
| 400 | Bad Request - Invalid input |
| 401 | Unauthorized - Auth failed |
| 403 | Forbidden - Permission denied |
| 404 | Not Found - Resource not found |
| 409 | Conflict - Resource conflict |
| 500 | Server Error - Internal error |
| 502 | Bad Gateway - Service unavailable |
| 503 | Service Unavailable - Maintenance |

## Rate Limiting

Rate limits are applied per merchant:

- **Auth endpoints**: 10 requests/minute
- **Payment endpoints**: 100 requests/minute
- **Query endpoints**: 1000 requests/hour

Headers in response:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704624000
```

## Testing

### Test Card Numbers

```
Visa:        4532015112830366
Mastercard:  5105105105105100
Amex:        374245455400126
RuPay:       508105000000000

Expiry:      Any future date (MM/YY)
CVV:         Any 3 digits
Holder Name: Any name
```

### Test UPI IDs

```
success@okhdfcbank
test@ybl
testuser@paytm
```

## Common Error Messages

```
INVALID_API_KEY
INVALID_API_SECRET
INVALID_CARD_NUMBER
INVALID_EXPIRY
INVALID_UPI_ID
INSUFFICIENT_FUNDS
PAYMENT_DECLINED
PAYMENT_TIMEOUT
DUPLICATE_PAYMENT
ORDER_NOT_FOUND
MERCHANT_INACTIVE
```

## Webhooks (Coming Soon)

```
POST /webhooks/payment.completed
POST /webhooks/payment.failed
POST /webhooks/refund.completed
```
