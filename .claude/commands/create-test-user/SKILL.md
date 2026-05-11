---
name: create-test-user
description: Create US test users in staging environment with auto-authentication
argument-hint: '[--state new] [--plan 2-meals-2-people] [--loyalty-tier gold]'
---

# Create Test User

Create test users in staging environment using direct backend API calls with automatic authentication. Currently supports US market only.

**⚠️ Important:** Requires VPN connection to HelloFresh staging environment.

## Overview

This skill creates test user accounts in the staging environment with optional subscriptions. It handles authentication, user creation, subscription setup, and state management automatically.

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

### 1. Input Validation
See `validation/input-validator.md` for:
- Parse and validate state parameter (new, active, cancelled, paused)
- Parse plan configuration (meals-people format)
- Validate loyalty tier if provided

### 2. Authentication
See `auth/token-generator.md` for:
- Auto-generate bearer token from staging credentials
- Validate token and handle auth failures
- Set market to US

### 3. User Creation
See `user/account-creator.md` for:
- Generate unique email with timestamp
- Create user account via signup API
- Extract customer ID and access token

### 4. Subscription Management (if state != 'new')
See `subscription/subscription-manager.md` for:
- Create cart with meal preferences
- Complete checkout with test payment
- Handle state changes (cancel/pause)
- Enroll in loyalty program if requested

### 5. Output
See `output/result-formatter.md` for:
- Display user credentials
- Show subscription details
- Print success summary

## Modules

### Validation
Input validation modules:
- `validation/input-validator.md` - Validate user input parameters

### Authentication
Token management modules:
- `auth/token-generator.md` - Generate bearer tokens from credentials

### User Management
User creation modules:
- `user/account-creator.md` - Create user accounts
- `user/credential-generator.md` - Generate unique credentials

### Subscription
Subscription management modules:
- `subscription/subscription-manager.md` - Manage subscription lifecycle
- `subscription/payment-handler.md` - Handle test payments
- `subscription/state-manager.md` - Handle state transitions

### Output
Result display modules:
- `output/result-formatter.md` - Format and display results

## User States

**`new` (Default - Recommended)** ⭐
- Account only, no subscription
- 100% success rate
- Fast (~5-10 seconds)
- Use for most E2E tests

**`active`**
- Full subscription with active state
- Uses test payment methods
- Slower (~20-30 seconds)

**`cancelled`**
- Creates subscription then cancels it
- For testing cancellation flows

**`paused`**
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

- curl (HTTP requests)
- jq or python3 (JSON parsing)
- VPN connection to staging

## Notes

- Uses hardcoded staging credentials for auto-authentication
- Generates unique emails using timestamp
- Uses official Braintree sandbox test nonces
- Only supports US market currently
- Test payment methods work in staging environment
