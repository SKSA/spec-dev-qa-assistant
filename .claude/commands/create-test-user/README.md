# Create Test User Command

Create test users in the HelloFresh staging environment with automatic authentication and subscription management.

## Overview

This command provides the `/create-test-user` skill that creates test user accounts with optional subscriptions for E2E testing. Currently supports US market only.

## Features

- **Auto-Authentication**: Uses hardcoded staging credentials to generate bearer tokens
- **Multiple User States**: Create users in new, active, cancelled, or paused states
- **Subscription Management**: Optional subscription creation with configurable plans
- **Loyalty Enrollment**: Optional loyalty program enrollment
- **Fast Execution**: New users created in ~5-10 seconds, subscriptions in ~20-30 seconds

## Usage

```bash
# Create new user (no subscription - fastest)
/create-test-user

# Create user with active subscription
/create-test-user --state active

# Create cancelled user with custom plan
/create-test-user --state cancelled --plan 3-meals-4-people

# Create user with loyalty enrollment
/create-test-user --state active --loyalty-tier gold
```

## Workflow

1. **Validation**: Validates input parameters (state, plan, loyalty tier)
2. **Authentication**: Generates bearer token from staging credentials
3. **User Creation**: Creates user account with unique email
4. **Subscription Setup** (optional): Creates cart, completes checkout, manages state
5. **Output**: Displays user credentials and subscription details

## Module Structure

The skill is organized into composable modules:

### Validation Modules
- `validation/input-validator.md` - Validate user input parameters

### Authentication Modules
- `auth/token-generator.md` - Generate bearer tokens from credentials

### User Management Modules
- `user/account-creator.md` - Create user accounts
- `user/credential-generator.md` - Generate unique credentials

### Subscription Modules
- `subscription/subscription-manager.md` - Manage subscription lifecycle
- `subscription/payment-handler.md` - Handle test payments
- `subscription/state-manager.md` - Handle state transitions

### Output Modules
- `output/result-formatter.md` - Format and display results

## User States

### `new` (Default - Recommended) ⭐
- Account only, no subscription
- 100% success rate
- Fast (~5-10 seconds)
- Use for most E2E tests that don't require subscriptions

### `active`
- Creates subscription with active state
- Uses test payment methods
- Slower (~20-30 seconds)
- Use when tests require active subscriptions

### `cancelled`
- Creates subscription then cancels it
- For testing cancellation flows

### `paused`
- Creates subscription then pauses it
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
- Always succeeds in sandbox

## Important Notes

1. **VPN Required**: Must be connected to staging VPN
2. **US Only**: Currently only supports US market
3. **Unique Emails**: Uses timestamp to ensure unique emails
4. **Test Payments**: Uses official Braintree test nonces (never charges real money)
5. **State Changes**: Cancel/pause operations are destructive

## Error Handling

### Common Errors

**VPN Not Connected**
```
❌ VPN not connected or staging gateway unreachable
→ Connect to staging VPN and try again
```

**Authentication Failed**
```
❌ Failed to authenticate with staging credentials
→ Check VPN connection and verify credentials
```

**API Error**
```
❌ Failed to create user account
→ Check API response for details
```

## Related Commands

- `/generate-e2e-tests` - Generate tests that can use created users
- `/verify-ac` - Verify acceptance criteria using test users

## References

- [Braintree Testing Documentation](https://developer.paypal.com/braintree/docs/reference/general/testing/ruby/#payment-method-nonces)
- Staging environment: `https://www-staging.hellofresh.com`
