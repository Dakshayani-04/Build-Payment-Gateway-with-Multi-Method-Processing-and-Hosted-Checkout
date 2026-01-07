# Frontend Documentation

## Overview

The frontend is built with React, providing two main interfaces:
1. **Merchant Dashboard** - For merchants to view credentials, stats, and transaction history
2. **Hosted Checkout** - Customer-facing payment interface

## Technology Stack

- **React 18+** - UI library
- **Axios** - HTTP client for API calls
- **React Router** - Page routing
- **CSS/Tailwind** - Styling
- **React Hooks** - State management

## Project Structure

```
frontend/
├── public/
│   ├── index.html
│   └── favicon.ico
├── src/
│   ├── components/
│   │   ├── Header.jsx              # Navigation header
│   │   ├── PaymentForm.jsx         # Payment input form
│   │   ├── CardInput.jsx           # Card entry component
│   │   ├── UPIInput.jsx            # UPI entry component
│   │   ├── StatusDisplay.jsx       # Payment status indicator
│   │   ├── TransactionTable.jsx    # History table
│   │   ├── StatsCard.jsx           # Statistics card
│   │   └── Modal.jsx               # Reusable modal
├── pages/
│   ├── Dashboard.jsx           # Merchant dashboard
│   ├── Checkout.jsx            # Hosted checkout page
│   └── Login.jsx               # Merchant login
├── hooks/
│   ├── useApi.js               # API calls hook
│   ├─┠ usePayment.js           # Payment processing hook
│   └── useAuth.js              # Authentication hook
├── services/
│   ├── api.js                  # Axios instance
│   ├── auth.js                 # Auth API calls
│   ├── payment.js              # Payment API calls
│   └── order.js                # Order API calls
├── styles/
│   ├── index.css               # Global styles
│   ├── dashboard.css           # Dashboard styles
│   └── checkout.css            # Checkout styles
├── utils/
│   ├── validation.js           # Form validation
│   ├── format.js               # Data formatting
│   └── constants.js            # App constants
├── App.jsx                 # Root component
├── App.css                 # App styles
├── index.js                # Entry point
├── .env.example
├── Dockerfile
└── package.json
```

## Components

### 1. **Merchant Dashboard** (`pages/Dashboard.jsx`)

Features:
- Display API credentials (Key & Secret)
- Transaction statistics
- Real-time status updates
- Payment history with filters
- Export transaction data

```jsx
// Usage
import Dashboard from './pages/Dashboard';

function App() {
  return <Dashboard merchantId={merchantId} />;
}
```

### 2. **Hosted Checkout** (`pages/Checkout.jsx`)

Features:
- Order summary display
- Payment method selection (UPI/Card)
- Real-time validation
- Payment processing status
- Success/Failure handling

```jsx
// Props
<Checkout
  orderId="order_123"
  amount={50000}
  currency="INR"
  onSuccess={handleSuccess}
  onFailure={handleFailure}
/>
```

### 3. **Payment Form** (`components/PaymentForm.jsx`)

Handles:
- Method selection (UPI/Card)
- Input validation
- Form submission
- Error display

### 4. **Card Input** (`components/CardInput.jsx`)

Fields:
- Card Number (with formatting)
- Expiry Date (MM/YY)
- CVV
- Cardholder Name

### 5. **UPI Input** (`components/UPIInput.jsx`)

Fields:
- UPI ID (e.g., user@bank)
- Mobile (optional)

### 6. **Status Display** (`components/StatusDisplay.jsx`)

States:
- Processing (spinner)
- Success (checkmark)
- Failed (error message)

## Custom Hooks

### `useApi()`
```javascript
const { data, loading, error, call } = useApi();

await call('POST', '/api/payments', payloadData);
```

### `usePayment()`
```javascript
const { 
  processing, 
  status, 
  error, 
  processPayment 
} = usePayment();

await processPayment(orderData);
```

### `useAuth()`
```javascript
const { 
  isAuthenticated, 
  merchantId, 
  login, 
  logout 
} = useAuth();

await login(apiKey, apiSecret);
```

## API Integration

### Authentication
```javascript
// services/auth.js
export const loginMerchant = async (apiKey, apiSecret) => {
  const response = await api.post('/auth/login', {
    apiKey,
    apiSecret
  });
  return response.data;
};
```

### Payment Processing
```javascript
// services/payment.js
export const createPayment = async (orderData, paymentMethod) => {
  const response = await api.post('/payments', {
    orderId: orderData.orderId,
    method: paymentMethod.type,
    ...paymentMethod.details
  });
  return response.data;
};
```

### Payment Status
```javascript
export const getPaymentStatus = async (paymentId) => {
  const response = await api.get(`/payments/${paymentId}`);
  return response.data;
};
```

## State Management

Using React Hooks and Context API:

```jsx
// AuthContext
const [auth, dispatch] = useReducer(authReducer, initialState);

const login = async (apiKey, apiSecret) => {
  const token = await loginMerchant(apiKey, apiSecret);
  dispatch({ type: 'LOGIN', payload: token });
};
```

## Form Validation

### Card Validation
```javascript
// utils/validation.js
export function validateCard(cardData) {
  return {
    number: validateCardNumber(cardData.number),
    expiry: validateExpiry(cardData.expiry),
    cvv: validateCVV(cardData.cvv),
    holder: cardData.holder.length > 0
  };
}
```

### UPI Validation
```javascript
export function validateUPI(upiId) {
  const upiRegex = /^[a-zA-Z0-9._-]+@[a-zA-Z]{3,}$/;
  return upiRegex.test(upiId);
}
```

## Real-Time Updates

Polling for payment status:
```javascript
const pollPaymentStatus = (paymentId, interval = 2000) => {
  const timer = setInterval(async () => {
    const status = await getPaymentStatus(paymentId);
    if (['success', 'failed'].includes(status.status)) {
      clearInterval(timer);
      handleStatusChange(status);
    }
  }, interval);
};
```

## Styling

### Global Styles (`styles/index.css`)
```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto';
  color: #333;
}
```

### Component Styles (CSS Modules)
```css
/* styles/checkout.css */
.payment-form {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.input-group {
  margin-bottom: 20px;
}

.input-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 600;
}

.input-group input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}
```

## Environment Variables

Create `.env` file:
```
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ENVIRONMENT=development
REACT_APP_MERCHANT_ID=test_merchant_001
```

## Development

### Start Development Server
```bash
npm install
npm start
```

### Build for Production
```bash
npm run build
```

### Run Tests
```bash
npm test
```

## Browser Support

- Chrome/Edge (latest)
- Firefox (latest)
- Safari (latest)
- Mobile browsers

## Performance Optimization

1. **Code Splitting**: React.lazy() for routes
2. **Memoization**: useMemo(), React.memo()
3. **Lazy Loading**: Images and components
4. **Bundle Size**: Tree-shaking, minification

## Accessibility

- ARIA labels on form inputs
- Keyboard navigation support
- Color contrast compliance
- Screen reader friendly

## Security

1. **Input Sanitization**: Prevent XSS
2. **HTTPS Only**: In production
3. **Token Storage**: localStorage for auth token
4. **CORS**: Properly configured headers
5. **CSP Headers**: Content Security Policy

## Future Enhancements

- [ ] Dark mode support
- [ ] Multi-language support
- [ ] Advanced charts and analytics
- [ ] Export to PDF/CSV
- [ ] Mobile app (React Native)
- [ ] Webhook notifications
- [ ] Refund interface
- [ ] Dispute management
