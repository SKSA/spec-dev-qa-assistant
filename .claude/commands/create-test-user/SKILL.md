---
name: create-test-user
description: Create US test users with active subscriptions in staging
argument-hint: '[--state active] [--plan 2-meals-2-people]'
---

# Create Test User - EXECUTABLE SKILL

This skill uses the **proven working script** from `command.md` as the executable implementation.

## How This Works

This SKILL.md acts as an orchestrator that:
1. Reads the complete, tested bash script from `command.md`
2. Executes it as-is (no interpretation or reconstruction)
3. The script is modular internally but executes as one reliable unit

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

## Execution Strategy

**IMPORTANT:** Execute the complete working script from `command.md`.

The script contains:
1. ✅ Setup and JSON parser detection
2. ✅ Authentication with staging credentials
3. ✅ Input validation (state, plan, loyalty tier)
4. ✅ User account creation with retry logic
5. ✅ Cart creation and delivery slot fetching (parallel)
6. ✅ Checkout with proper payment methods
7. ✅ Subscription polling with timeout
8. ✅ State management (cancel/pause)
9. ✅ Result formatting and display

## How to Execute

Read and execute the bash code blocks from `command.md` in this order:

### Step 1: Setup (lines 74-132)
- Detect JSON parser (jq or python3)
- Show VPN warning
- Authenticate with staging credentials
- Get bearer token

### Step 2: Validation (lines 143-158)  
- Hardcode MARKET="US"
- Validate state parameter
- Parse plan into meals/people

### Step 3: Generate Credentials (lines 162-200)
- Generate unique email with timestamp
- Set password, first/last name
- Get business division

### Step 4: Create User (lines 207-268)
- Create customer with retry logic
- Login to get user-specific access token

### Step 5: Create Subscription if needed (lines 304-575)
- Skip if state="new"
- Get SKU and address
- Create cart and fetch delivery slots (parallel)
- Complete checkout with payment
- Poll for subscription
- Handle cancel/pause state changes

### Step 6: Display Results (lines 590-625)
- Format success message
- Show credentials
- Show subscription details if applicable

## Helper Functions (lines 639-871)

The script includes these tested helper functions:
- `get_business_division()` - Returns brand for market
- `get_plan_sku()` - Generates SKU from market/meals/people
- `get_test_address()` - Returns test address for market (US only currently)
- `get_payment_method_config()` - Returns payment method (Braintree for US)

## Key Success Factors

1. **Uses exact working API endpoints** from successful runs
2. **Parallel cart + delivery fetch** for speed
3. **Retry logic** for transient errors
4. **Proper subscription polling** with timeout
5. **Official Braintree test nonce** (`fake-valid-nonce`)

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

## Documentation Modules

The modular documentation files in subdirectories explain each part:
- `validation/` - Input validation logic
- `auth/` - Authentication process
- `user/` - User creation process
- `subscription/` - Subscription lifecycle
- `output/` - Result formatting

**These are for reference only.** The executable code is in `command.md`.
