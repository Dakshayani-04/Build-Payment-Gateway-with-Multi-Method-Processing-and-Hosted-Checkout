# Full-Stack Payment Gateway System

## Overview

A production-ready payment gateway platform similar to Razorpay/Stripe, implementing UPI and Card payment processing with a merchant dashboard and hosted checkout interface. Built with Node.js, Express.js, PostgreSQL, React, and Docker.

## Features

### Core Functionality
- **Multi-Payment Methods**: UPI and Credit/Debit Card support
- **Merchant Authentication**: API Key & Secret-based authentication
- **Real-time Payment Processing**: State machine-based payment flow
- **Transaction Management**: Complete payment history and status tracking
- **Hosted Checkout**: Customer-facing payment interface
- **Merchant Dashboard**: Transaction analytics and API credentials management

### Technical Features
- **Payment Validation**:
  - UPI VPA format validation
  - Luhn algorithm for card validation
  - Card network detection (Visa, Mastercard, Amex, RuPay)
  - Expiry date validation
- **Security**: API Key/Secret authentication, HTTPS support
- **State Management**: Processing → Success/Failed payment states
- **Deterministic Testing**: Predictable payment outcomes via environment variables
- **Database**: PostgreSQL with proper relationships and indexing
- **Containerization**: Full Docker & Docker Compose setup

## Tech Stack

### Backend
- **Runtime**: Node.js
- **Framework**: Express.js
- **Database**: PostgreSQL
- **ORM**: Prisma (optional, can use raw queries)
- **Authentication**: JWT, API Keys

### Frontend
- **Library**: React
- **Styling**: CSS/Tailwind
- **HTTP Client**: Axios

### DevOps
- **Containerization**: Docker
- **Orchestration**: Docker Compose
- **Version Control**: Git

### Tools
- **API Testing**: Postman
- **Database Management**: pgAdmin (optional)

## Project Structure

```
payment-gateway/
├── backend/
│   ├── src/
│   │   ├── config/           # Database and app configuration
│   │   ├── controllers/       # Request handlers
│   │   ├── middleware/        # Auth, validation middleware
│   │   ├── models/            # Database models
│   │   ├── routes/            # API endpoints
│   │   ├── utils/             # Payment validation, helpers
│   │   ├── services/          # Business logic
│   │   └── index.js           # Express app setup
│   ├── .env.example           # Environment template
│   ├── Dockerfile             # Backend container config
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── components/        # React components
│   │   ├── pages/             # Page components
│   │   ├── hooks/             # Custom hooks
│   │   ├── services/          # API calls
│   │   ├── styles/            # CSS files
│   │   └── App.jsx
│   ├── Dockerfile             # Frontend container config
│   └── package.json
├── docker-compose.yml         # Multi-container orchestration
├── README.md                  # This file
└── .gitignore
```

## Prerequisites

- Docker & Docker Compose (for containerized setup)
- Node.js 18+ (for local development)
- PostgreSQL 13+ (for local development)
- npm or yarn

## Installation & Setup

### Option 1: Using Docker Compose (Recommended)

1. **Clone the repository**:
   ```bash
   git clone https://github.com/Dakshayani-04/Build-Payment-Gateway-with-Multi-Method-Processing-and-Hosted-Checkout.git
   cd payment-gateway
   ```

2. **Create environment files**:
   ```bash
   # Backend environment
   cp backend/.env.example backend/.env
   
   # Frontend environment (if applicable)
   cp frontend/.env.example frontend/.env
   ```

3. **Start all services**:
   ```bash
   docker-compose up -d
   ```

4. **Access the services**:
   - **Backend API**: http://localhost:5000
   - **Frontend Dashboard**: http://localhost:3000
   - **Hosted Checkout**: http://localhost:3000/checkout

### Option 2: Local Development

1. **Setup Backend**:
   ```bash
   cd backend
   npm install
   cp .env.example .env
   # Configure DATABASE_URL in .env
   npm run dev
   ```

2. **Setup Frontend**:
   ```bash
   cd frontend
   npm install
   cp .env.example .env
   # Configure REACT_APP_API_URL in .env
   npm start
   ```

## Environment Variables

### Backend (.env)
```
NODE_ENV=development
PORT=5000
DATABASE_URL=postgresql://user:password@localhost:5432/payment_gateway
JWT_SECRET=your_jwt_secret_key
API_KEY_SECRET=your_api_key_secret
TEST_MODE=true
FRONTEND_URL=http://localhost:3000
```

### Frontend (.env)
```
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ENVIRONMENT=development
```

## Default Test Credentials

A test merchant is automatically seeded on startup:

```
Merchant ID: test_merchant_001
API Key: test_pk_123456789
API Secret: test_sk_987654321
```

## API Documentation

### Authentication
All API requests require:
- `X-API-Key`: Merchant's API Key
- `X-API-Secret`: Merchant's API Secret

### Endpoints

#### 1. Create Order
```http
POST /api/orders
Content-Type: application/json

{
  "amount": 50000,
  "currency": "INR",
  "customerId": "cust_123",
  "customerEmail": "customer@example.com",
  "description": "Order for Product"
}
```

**Response**:
```json
{
  "orderId": "order_abc123",
  "amount": 50000,
  "currency": "INR",
  "status": "created",
  "createdAt": "2024-01-07T10:00:00Z"
}
```

#### 2. Create Payment
```http
POST /api/payments
Content-Type: application/json

{
  "orderId": "order_abc123",
  "method": "upi",
  "upiId": "customer@upi"
}
```

Or for card:
```json
{
  "orderId": "order_abc123",
  "method": "card",
  "cardNumber": "4532015112830366",
  "expiryMonth": "12",
  "expiryYear": "25",
  "cvv": "123"
}
```

**Response**:
```json
{
  "paymentId": "pay_xyz789",
  "status": "processing",
  "orderId": "order_abc123",
  "amount": 50000,
  "method": "upi"
}
```

#### 3. Get Payment Status
```http
GET /api/payments/:paymentId
```

**Response**:
```json
{
  "paymentId": "pay_xyz789",
  "status": "success",
  "orderId": "order_abc123",
  "amount": 50000,
  "method": "upi",
  "createdAt": "2024-01-07T10:00:00Z",
  "updatedAt": "2024-01-07T10:00:05Z"
}
```

#### 4. Merchant Authentication
```http
POST /api/auth/login
Content-Type: application/json

{
  "apiKey": "test_pk_123456789",
  "apiSecret": "test_sk_987654321"
}
```

#### 5. Get Transaction History
```http
GET /api/transactions?limit=20&offset=0
```

## Payment Flow

### State Machine Diagram
```
Order Created
    ↓
Payment Processing (immediate)
    ↓
┌───────────────────┐
│  Simulated Bank   │ (2-5 second delay)
│  Processing       │
└───────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Success (if valid card/UPI)         │
│ Failed (if invalid/test declined)   │
└─────────────────────────────────────┘
```

## Deterministic Test Mode

For predictable payment outcomes:

```bash
# In .env
TEST_MODE=true
TEST_PAYMENT_OUTCOME=success  # or 'failed'
```

This allows consistent testing without random failures.

## Frontend Features

### Merchant Dashboard
- View API credentials (Key & Secret)
- Transaction statistics (total, successful, failed, refunded)
- Payment history with filters
- Real-time status updates

### Hosted Checkout
- Customer payment form
- UPI & Card payment options
- Real-time payment processing
- Success/Failure status display

## Database Schema

### Tables

**merchants**
- id
- apiKey
- apiSecret
- email
- status
- createdAt
- updatedAt

**orders**
- id
- merchantId
- amount
- currency
- customerId
- customerEmail
- description
- status
- createdAt
- updatedAt

**payments**
- id
- orderId
- merchantId
- method (upi/card)
- amount
- status (processing/success/failed)
- paymentData (JSON)
- errorMessage
- createdAt
- updatedAt

## API Testing with Postman

1. **Import Collection**:
   - Open Postman
   - Import the provided collection file
   - Set environment variables (API_KEY, API_SECRET)

2. **Test Merchants Endpoint**:
   - Create Order
   - Create Payment
   - Get Payment Status

## Deployment

### Docker Compose Production
```bash
docker-compose -f docker-compose.yml up -d
```

### Manual Deployment
1. Build Docker images
2. Push to registry (Docker Hub, ECR, etc.)
3. Deploy to cloud platform (AWS, GCP, Azure, etc.)
4. Configure environment variables
5. Set up reverse proxy (Nginx)
6. Enable HTTPS with SSL certificates

## Performance Considerations

- **Database Indexing**: Indexes on `merchantId`, `orderId`, `status` for fast queries
- **Connection Pooling**: PostgreSQL connection pool (20-30 connections)
- **Caching**: Redis for session/payment status caching (optional)
- **Rate Limiting**: Implement rate limiting on payment endpoints

## Security Best Practices

1. **Secrets Management**:
   - Never commit .env files
   - Use environment variables for secrets
   - Rotate API keys regularly

2. **HTTPS**: Always use HTTPS in production

3. **Request Signing**: Implement webhook signature verification

4. **Input Validation**: Validate all inputs (Luhn for cards, VPA format for UPI)

5. **Rate Limiting**: Prevent brute force and DDoS attacks

6. **Database Security**: Use parameterized queries, least privilege DB user

## Troubleshooting

### Common Issues

**Payment stuck in "processing" state**
- Check backend logs: `docker logs payment-gateway-api`
- Verify database connection
- Ensure TEST_MODE is properly configured

**Database connection refused**
- Verify PostgreSQL is running
- Check DATABASE_URL in .env
- Ensure database user has proper permissions

**CORS errors on frontend**
- Update CORS settings in Express backend
- Verify FRONTEND_URL in backend .env

**Port conflicts**
- Change ports in docker-compose.yml or .env
- Check if ports are already in use: `netstat -ano`

## Development Workflow

1. **Feature Branch**:
   ```bash
   git checkout -b feature/new-feature
   ```

2. **Make Changes**:
   - Update backend/frontend code
   - Test locally with Docker Compose

3. **Test**:
   ```bash
   npm test
   ```

4. **Commit & Push**:
   ```bash
   git add .
   git commit -m "feat: add new feature"
   git push origin feature/new-feature
   ```

5. **Create Pull Request**: Open PR on GitHub for review

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Open a Pull Request

## License

MIT License - feel free to use this project for personal or commercial purposes.

## Support

For issues, questions, or suggestions:
- Open an issue on GitHub
- Contact: your-email@example.com

## Future Enhancements

- [ ] Webhook support for real-time notifications
- [ ] Refund functionality
- [ ] Multiple currency support
- [ ] Mobile app (React Native)
- [ ] Advanced analytics dashboard
- [ ] Payment reconciliation
- [ ] Dispute management
- [ ] Multi-merchant support with sub-accounts

## Acknowledgments

Built with inspiration from industry-leading payment gateways like Razorpay and Stripe.

---

**Last Updated**: January 7, 2024
**Status**: Production Ready
