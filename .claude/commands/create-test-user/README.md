# Create Test User Plugin

Create test users in the HelloFresh staging environment with automatic authentication and subscription management.

## Overview

This plugin provides the `/create-test-user` command that creates test user accounts with optional subscriptions for E2E testing. Currently supports US market only.

## Features

- **Auto-Authentication**: Uses hardcoded staging credentials to generate bearer tokens
- **Subscription Creation**: Always creates users with active subscriptions
- **Multiple States**: Support for active, cancelled, or paused subscription states
- **Fast Execution**: ~20-30 seconds per user with full subscription setup
- **Debug Logging**: Comprehensive delivery slot API debugging

## Installation

```bash
claude plugin install hellofresh/claude-plugins-marketplace/external_plugins/create-test-user
```

## Usage

```bash
# Create user with active subscription (default)
/create-test-user

# Create with custom plan
/create-test-user --plan 3-meals-4-people

# Create and cancel subscription
/create-test-user --state cancelled

# Create and pause subscription
/create-test-user --state paused
```

## User States

### `active` (Default) ⭐
- Creates user with active subscription
- Full checkout flow
- ~20-30 seconds
- Ready for immediate testing

### `cancelled`
- Creates subscription then cancels it
- For testing cancellation flows

### `paused`
- Creates subscription then pauses it (4 weeks)
- For testing pause flows

## Output

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Test user created successfully!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📧 Email:          hfuser-11-153045@hf.com
🔑 Password:       qwerty123
🆔 Customer ID:    12345678
🎫 Subscription ID: 87654321
📦 Plan:           2 meals for 2 people
🌍 Market:         US
📊 State:          active
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Module Structure

The command is organized into composable modules:

### Detection & Validation
- `validation/input-validator.md` - Validate user input parameters

### Authentication
- `auth/token-generator.md` - Generate bearer tokens from staging credentials

### User Management
- `user/account-creator.md` - Create user accounts with retry logic
- `user/credential-generator.md` - Generate unique credentials

### Subscription Management
- `subscription/subscription-manager.md` - Manage subscription lifecycle
- `subscription/payment-handler.md` - Handle test payment methods
- `subscription/state-manager.md` - Handle state transitions

### Configuration
- `config/plans.md` - Plan SKU configuration
- `config/addresses.md` - Test address configuration
- `config/markets.md` - Market configuration

### Output
- `output/result-formatter.md` - Format and display results

### API Reference
- `api/signup.md` - User signup endpoint
- `api/subscription.md` - Subscription endpoints
- `api/state-management.md` - State management endpoints

## Dependencies

- **curl**: HTTP requests to staging APIs
- **jq or python3**: JSON parsing
- **VPN**: Connection to HelloFresh staging environment

## Configuration

### Staging Credentials
Uses hardcoded credentials for auto-authentication:
- Username: `er+391+1@hf.com`
- Password: `qwerty123`

### Test Payment Methods
Uses official Braintree sandbox test nonce:
- Nonce: `fake-valid-nonce`
- No real money charged

## Important Notes

1. **VPN Required**: Must be connected to staging VPN
2. **US Only**: Currently only supports US market
3. **Unique Emails**: Uses timestamp to ensure unique emails (format: `hfuser-DD-HHMMSS@hf.com`)
4. **Test Payments**: Uses official Braintree test nonces
5. **Always Creates Subscriptions**: All users are created with active subscriptions by default

## Troubleshooting

### No Delivery Slots Available
If subscription creation fails due to "no delivery slots available", this is a staging Logistics Configurator limitation. The debug logs will show which ZIP code was queried and what slots were returned.

### VPN Connection
Ensure you're connected to the HelloFresh staging VPN before running the command.

## Author

**Squad**: Share  
**Tribe**: Loyalty and Virality  
**Email**: sharesquad@hellofresh.com

## References

- [Braintree Testing Documentation](https://developer.paypal.com/braintree/docs/reference/general/testing/ruby/#payment-method-nonces)
- Staging environment: `https://www-staging.hellofresh.com`
