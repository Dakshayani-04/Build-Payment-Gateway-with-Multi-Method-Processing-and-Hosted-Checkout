# Backend Documentation

## Architecture Overview

The backend is built with Node.js and Express.js, implementing a RESTful API for payment processing.

### Core Modules

#### 1. **Authentication Layer** (`middleware/auth.js`)
- API Key & Secret validation
- JWT token generation
- Request signature verification
- Merchant identification

#### 2. **Payment Processing** (`services/paymentService.js`)
- Order creation and management
- Payment initiation and processing
- State machine implementation (processing → success/failed)
- Simulated bank processing delay

#### 3. **Validation Module** (`utils/validation.js`)
- UPI VPA format validation regex: `/^[a-zA-Z0-9._-]+@[a-zA-Z]{3,}$/`
- Luhn algorithm for card validation
- Card network detection (BIN ranges)
- Expiry date verification

#### 4. **Database Layer** (`models/`)
- Merchants table (API credentials)
- Orders table (transaction metadata)
- Payments table (payment records)
- Indexes on: `merchantId`, `orderId`, `status` for performance

## File Structure

```
backend/
├── src/
│   ├── config/
│   │   ├── database.js          # PostgreSQL connection
│   │   ├── env.js               # Environment variables
│   │   └── constants.js          # App constants
│   ├── controllers/
│   │   ├── authController.js    # Authentication endpoints
│   │   ├── orderController.js   # Order management
│   │   ├── paymentController.js # Payment processing
│   │   └── transactionController.js # History/stats
│   ├── middleware/
│   │   ├── auth.js              # API key validation
│   │   ├── errorHandler.js      # Global error handling
│   │   └── requestLogger.js     # Request logging
│   ├── models/
│   │   ├── Merchant.js          # Merchant schema
│   │   ├── Order.js             # Order schema
│   │   └── Payment.js           # Payment schema
│   ├── routes/
│   │   ├── auth.js              # Auth endpoints
│   │   ├── orders.js            # Order endpoints
│   │   ├── payments.js          # Payment endpoints
│   │   └── transactions.js      # History endpoints
│   ├── services/
│   │   ├── authService.js       # Auth logic
│   │   ├── paymentService.js    # Payment logic
│   │   └── merchantService.js   # Merchant logic
│   ├── utils/
│   │   ├── validation.js        # Card/UPI validation
│   │   ├── helpers.js           # Utility functions
│   │   └── logger.js            # Logging utility
│   └── index.js                 # Express app setup
├── .env.example
├── Dockerfile
├── package.json
└── server.js                    # Entry point
```

## API Endpoints

### Authentication
```
POST /api/auth/login
  Body: { apiKey, apiSecret }
  Returns: { token, merchantId, email }
```

### Orders
```
POST /api/orders
  Body: { amount, currency, customerId, customerEmail, description }
  Returns: { orderId, amount, status, createdAt }

GET /api/orders/:orderId
  Returns: Order details

GET /api/orders?merchantId=xxx
  Returns: List of orders
```

### Payments
```
POST /api/payments
  Body: { orderId, method, upiId || cardNumber+expiry+cvv }
  Returns: { paymentId, status, orderId, amount }

GET /api/payments/:paymentId
  Returns: { paymentId, status, orderId, method, amount }

GET /api/payments?orderId=xxx
  Returns: Payments for order
```

### Transactions
```
GET /api/transactions?limit=20&offset=0
  Returns: List of all transactions for merchant

GET /api/transactions/stats
  Returns: { totalTransactions, totalAmount, successCount, failureCount }
```

## Database Schema

### Merchants
```sql
CREATE TABLE merchants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  apiKey VARCHAR(255) UNIQUE NOT NULL,
  apiSecret VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  status VARCHAR(50) DEFAULT 'active',
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_merchants_apikey ON merchants(apiKey);
CREATE INDEX idx_merchants_email ON merchants(email);
```

### Orders
```sql
CREATE TABLE orders (
  id VARCHAR(50) PRIMARY KEY,
  merchantId UUID NOT NULL REFERENCES merchants(id),
  amount BIGINT NOT NULL,
  currency VARCHAR(3) DEFAULT 'INR',
  customerId VARCHAR(255),
  customerEmail VARCHAR(255),
  description TEXT,
  status VARCHAR(50) DEFAULT 'created',
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_merchantid ON orders(merchantId);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_customerid ON orders(customerId);
```

### Payments
```sql
CREATE TABLE payments (
  id VARCHAR(50) PRIMARY KEY,
  orderId VARCHAR(50) NOT NULL REFERENCES orders(id),
  merchantId UUID NOT NULL REFERENCES merchants(id),
  method VARCHAR(20) NOT NULL,
  amount BIGINT NOT NULL,
  status VARCHAR(50) DEFAULT 'processing',
  paymentData JSONB,
  errorMessage TEXT,
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_orderId ON payments(orderId);
CREATE INDEX idx_payments_merchantid ON payments(merchantId);
CREATE INDEX idx_payments_status ON payments(status);
```

## Payment Validation Logic

### UPI Validation
```javascript
function validateUPI(upiId) {
  const upiRegex = /^[a-zA-Z0-9._-]+@[a-zA-Z]{3,}$/;
  return upiRegex.test(upiId);
}
// Example: customer@upi, user.name@bank
```

### Card Validation (Luhn Algorithm)
```javascript
function validateCardNumber(cardNumber) {
  const digits = cardNumber.replace(/\D/g, '');
  let sum = 0;
  for (let i = digits.length - 1; i >= 0; i--) {
    let digit = parseInt(digits[i]);
    if ((digits.length - 1 - i) % 2 === 1) {
      digit *= 2;
      if (digit > 9) digit -= 9;
    }
    sum += digit;
  }
  return sum % 10 === 0;
}
```

### Card Network Detection
```javascript
function detectCardNetwork(cardNumber) {
  const bin = cardNumber.substring(0, 6);
  if (/^4[0-9]{12}(?:[0-9]{3})?$/.test(cardNumber)) return 'Visa';
  if (/^5[1-5][0-9]{14}$/.test(cardNumber)) return 'Mastercard';
  if (/^3[47][0-9]{13}$/.test(cardNumber)) return 'Amex';
  if (/^508[0-9]{12}$/.test(cardNumber)) return 'RuPay';
  return 'Unknown';
}
```

## State Machine Implementation

```javascript
const paymentStates = {
  CREATED: 'created',
  PROCESSING: 'processing',
  SUCCESS: 'success',
  FAILED: 'failed',
  REFUNDED: 'refunded'
};

const validTransitions = {
  created: ['processing'],
  processing: ['success', 'failed'],
  success: ['refunded'],
  failed: [],
  refunded: []
};
```

## Payment Processing Flow

1. **Initiate**: Payment created in 'processing' state
2. **Simulate Bank Delay**: 2-5 second wait
3. **Validate**:
   - Card: Check Luhn, expiry, CVV
   - UPI: Check VPA format
4. **Determine Outcome**:
   - Valid: Transition to 'success'
   - Invalid: Transition to 'failed'
5. **Persist**: Update database with final status

## Error Handling

```javascript
const errorResponses = {
  INVALID_API_KEY: { code: 401, message: 'Invalid API credentials' },
  INVALID_CARD: { code: 400, message: 'Invalid card number' },
  PAYMENT_FAILED: { code: 402, message: 'Payment processing failed' },
  DATABASE_ERROR: { code: 500, message: 'Database error' },
  INVALID_ORDER: { code: 404, message: 'Order not found' }
};
```

## Security Considerations

1. **Never log sensitive data** (card numbers, secrets)
2. **Use HTTPS only** in production
3. **Rate limit** payment endpoints (100 req/minute)
4. **Hash API secrets** before storage
5. **Validate all inputs** before database queries
6. **Use parameterized queries** for SQL
7. **Implement request signing** for webhooks

## Performance Optimization

- **Connection pooling**: 20-30 connections to PostgreSQL
- **Database indexes**: On frequently queried columns
- **Query optimization**: Select only needed fields
- **Caching**: Redis for merchant credentials (optional)
- **Async processing**: Use Promise/async-await

## Testing

### Unit Tests
```bash
npm test -- validation.test.js
npm test -- paymentService.test.js
```

### Integration Tests
```bash
npm test -- api.integration.test.js
```

### Manual Testing with Postman
1. Import collection from `/postman/Payment-Gateway.postman_collection.json`
2. Set environment variables
3. Run requests sequentially
