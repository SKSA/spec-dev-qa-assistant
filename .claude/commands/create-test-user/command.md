---
name: /create-test-user
argument-hint: '[--state new]'
description: Create US test users in staging with auto-authentication
mode: single-agent
parameters:
  - name: state
    description: "User subscription state: new (no subscription), active, cancelled, or paused"
    required: false
    default: "new"
  - name: plan
    description: "Subscription plan (e.g., 2-meals-2-people, 3-meals-4-people)"
    required: false
    default: "2-meals-2-people"
  - name: loyalty_tier
    description: "Loyalty tier (bronze, silver, gold, platinum)"
    required: false
dependencies:
  - curl (required)
  - jq or python3 (required for JSON parsing)
  - VPN connection to HelloFresh staging (required)
---

# Create Test User

## Purpose
Create US test users in staging environment using direct backend API calls with automatic authentication from username/password.

**Features:**
- Auto-generates bearer token from your staging credentials
- Creates user accounts (with optional subscriptions)
- US market only

**Base URL**: `https://www-staging.hellofresh.com`

## User States

**`new` (Default - Recommended for Speed)** ⭐
- Account only, no subscription
- 100% success rate
- Fast (~5-10 seconds)
- Use for most E2E tests that don't require active subscriptions

**`active`, `cancelled`, `paused`** (With Subscription)
- Creates subscription via checkout with official Braintree test nonces
- Uses static sandbox tokens: `fake-valid-nonce` (Braintree), `test_` cards (Adyen), `card_test_` (ProcessOut)
- Should work reliably in staging (uses official payment processor test values)
- Slower (~20-30 seconds) due to full checkout flow
- Use when tests specifically require active subscription state

## Usage

Run the command:

```bash
/create-test-user
```

**Optional Parameters:**
- `--state`: Account type - default: `new` (no subscription)
  - `new`: Account only (fastest, most reliable)
  - `active`: With active subscription (requires payment)
  - `cancelled`: With cancelled subscription
  - `paused`: With paused subscription
- `--plan`: Subscription plan - default: 2-meals-2-people

**Authentication:**
Uses hardcoded staging credentials (`er+391+1@hf.com` / `qwerty123`) to automatically generate bearer tokens. No manual authentication required.
- `--plan`: Subscription plan - default: 2-meals-2-people
- `--token`: Use existing bearer token instead of username/password

## Setup

```bash
# Check for jq or python3
if command -v jq >/dev/null 2>&1; then
  JSON_PARSER="jq"
elif command -v python3 >/dev/null 2>&1; then
  JSON_PARSER="python3"
else
  echo "❌ No JSON parser available (need jq or python3)"
  echo "   Install: brew install jq"
  exit 1
fi

echo ""
echo "⚠️  WARNING: This command requires an active VPN connection to HelloFresh staging."
echo "   Make sure you're connected to the VPN before proceeding."
echo ""

# Show configuration
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 Configuration"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  Market: $MARKET"
echo "  State: ${STATE:-active}"
echo "  Plan: ${PLAN:-2-meals-2-people}"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

# Hardcoded staging credentials for automatic token generation
STAGING_USERNAME="er+391+1@hf.com"
STAGING_PASSWORD="qwerty123"

echo "🔐 Authenticating with staging credentials..."

MARKET="US"

LOGIN_AUTH_RESPONSE=$(curl -s --max-time 10 -X POST \
  "https://www-staging.hellofresh.com/gw/login?country=us" \
  -H 'content-type: application/json' \
  -d "{\"username\":\"${STAGING_USERNAME}\",\"password\":\"${STAGING_PASSWORD}\"}")

if [[ "$JSON_PARSER" == "jq" ]]; then
  BEARER_TOKEN=$(echo "$LOGIN_AUTH_RESPONSE" | jq -r '.access_token // empty')
else
  BEARER_TOKEN=$(echo "$LOGIN_AUTH_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('access_token', ''))")
fi

if [[ -z "$BEARER_TOKEN" ]] || [[ "$BEARER_TOKEN" == "null" ]]; then
  echo "❌ Failed to authenticate with staging credentials"
  echo "Response: ${LOGIN_AUTH_RESPONSE:0:500}"
  echo ""
  echo "Please check:"
  echo "  • VPN is connected to staging"
  echo "  • Staging credentials are still valid"
  exit 1
fi

echo "✅ Authentication successful"
echo "✅ Bearer token ready"
```

### Step 1: Parse Input & Validate

**Optional Arguments:**
- `--state`: User state (new, active, cancelled, paused) - default: new
- `--plan=X-meals-Y-people`: Plan configuration (default: 2-meals-2-people)
- `--loyalty-tier=basic|gold|platinum`: Enroll in loyalty program

**Validation:**

```bash
# Market is hardcoded to US
MARKET="US"

# Validate state
if [[ ! "$STATE" =~ ^(new|active|cancelled|paused)$ ]]; then
  echo "❌ Invalid state: $STATE"
  echo "Valid states: new, active, cancelled, paused"
  exit 1
fi

# Parse plan option
PLAN="${PLAN:-2-meals-2-people}"
MEALS=$(echo "$PLAN" | cut -d'-' -f1)
PEOPLE=$(echo "$PLAN" | cut -d'-' -f3)
```

### Step 2: Generate User Credentials

```bash
# Generate unique email with date-time format: hfuser-DD-HHMMSS@hf.com
EMAIL="hfuser-$(date +%d-%H%M%S)@hf.com"
PASSWORD="qwerty123"
FIRST_NAME="Test"
LAST_NAME="User"

# Get business division for market
get_business_division() {
  local market=$1
  local brand=""
  
  case "$market" in
    MR) brand="BRAND_GOODCHOP" ;;
    GN) brand="BRAND_GREENCHEF" ;;
    YE) brand="BRAND_YOUFOODZ" ;;
    CK) brand="BRAND_CHEFS_PLATE" ;;
    FJ) brand="BRAND_FACTOR" ;;
    ER) brand="BRAND_EVERYPLATE" ;;
    *) brand="BRAND_HELLOFRESH" ;;
  esac
  
  echo "{\"brand\":\"${brand}\",\"region_code\":\"${market}\"}"
}

BUSINESS_DIVISION=$(get_business_division "$MARKET")

echo ""
echo "🎯 Creating test user:"
echo "   Market: $MARKET"
echo "   State: $STATE"
echo "   Token: ${BEARER_TOKEN:0:20}..."
if [[ "$STATE" != "new" ]]; then
  echo "   Plan: $MEALS meals for $PEOPLE people"
fi
if [[ -n "$LOYALTY_TIER" ]]; then
  echo "   Loyalty: $LOYALTY_TIER tier"
fi
echo ""
```

### Step 3: Create User Account

Using the provided bearer token to create customer and login.

```bash
echo "📝 Step 1/4: Creating user account..."

# Step 3a: Create customer using provided bearer token (with retry on transient errors)
MAX_RETRIES=2
RETRY_COUNT=0

while [[ $RETRY_COUNT -le $MAX_RETRIES ]]; do
  CUSTOMER_RESPONSE=$(curl -s --max-time 8 -X POST \
    "https://www-staging.hellofresh.com/gw/api/customers?country=${MARKET}" \
    -H "Authorization: Bearer ${BEARER_TOKEN}" \
    -H 'content-type: application/json' \
    -d "{\"customer\":{\"email\":\"${EMAIL}\",\"password\":\"${PASSWORD}\",\"trackingMetadata\":{},\"status\":\"registered_checkout\",\"isTest\":true,\"firstName\":\"${FIRST_NAME}\",\"lastName\":\"${LAST_NAME}\"}}")

  # Parse customer response
  if [[ "$JSON_PARSER" == "jq" ]]; then
    CUSTOMER_ID=$(echo "$CUSTOMER_RESPONSE" | jq -r '.id // empty')
  else
    CUSTOMER_ID=$(echo "$CUSTOMER_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('id', ''))")
  fi

  # Check for success or retryable errors
  if [[ -n "$CUSTOMER_ID" ]] && [[ "$CUSTOMER_ID" != "null" ]]; then
    echo "✅ Customer created (ID: ${CUSTOMER_ID})"
    break
  elif echo "$CUSTOMER_RESPONSE" | grep -q "profile-service\|timeout\|502\|503"; then
    RETRY_COUNT=$((RETRY_COUNT + 1))
    if [[ $RETRY_COUNT -le $MAX_RETRIES ]]; then
      echo "⚠️  Transient error, retrying... ($RETRY_COUNT/$MAX_RETRIES)"
      sleep 1
    else
      echo "❌ Failed to create customer after $MAX_RETRIES retries"
      echo "Response (first 500 chars): ${CUSTOMER_RESPONSE:0:500}"
      exit 1
    fi
  else
    echo "❌ Failed to create customer"
    echo "Response (first 500 chars): ${CUSTOMER_RESPONSE:0:500}"
    exit 1
  fi
done

# Step 3b: Login to get user-specific access token
LOGIN_RESPONSE=$(curl -s --max-time 8 -X POST \
  "https://www-staging.hellofresh.com/gw/login?country=${MARKET}" \
  -H 'content-type: application/json' \
  -d "{\"username\":\"${EMAIL}\",\"password\":\"${PASSWORD}\"}")

if [[ "$JSON_PARSER" == "jq" ]]; then
  ACCESS_TOKEN=$(echo "$LOGIN_RESPONSE" | jq -r '.access_token // empty')
else
  ACCESS_TOKEN=$(echo "$LOGIN_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('access_token', ''))")
fi

if [[ -z "$ACCESS_TOKEN" ]] || [[ "$ACCESS_TOKEN" == "null" ]]; then
  echo "❌ Failed to login"
  echo "Response (first 500 chars): ${LOGIN_RESPONSE:0:500}"
  exit 1
fi

echo "✅ User logged in successfully"
```

### Step 4: Create Subscription (if needed)

Uses cart + checkout flow (from JMeter working approach).

**Payment Token Strategy:**
Uses official Braintree sandbox test nonces from [Braintree Testing Documentation](https://developer.paypal.com/braintree/docs/reference/general/testing/ruby/#payment-method-nonces):

- `fake-valid-nonce` - Official static test nonce for Braintree
- `test_` prefixed card details - For Adyen tokenization
- `card_test_` tokens - For ProcessOut

These are **static, non-expiring test values** provided by the payment processors for sandbox testing.

**Meal Preference Strategy:**
Cart metadata includes `"mealsPreset": "chefschoice"` which is:
- A valid preset for US market (verified via shopping-platform-service)
- The highest priority/weight preference (weight: 8)
- "A handpicked selection of Fit, Quick, & Variety recipes"
- Stored in cart metadata (how UI does it)
- Read by OPS when calling profile-service during checkout

**Valid US Presets** (from shopping-platform-service):
- `chefschoice` (weight: 8) ✅ Using this
- `veggie` (weight: 7)
- `family` (weight: 6)
- `fit` (weight: 5)
- `quick` (weight: 4)
- `porkfree` (weight: 3)
- `pescatarian` (weight: 2)

**Note:** Profile-service validates against shopping-platform-service. Using an invalid preset (like `"classic"`) will cause a 400 error.

Skip if state is "new":

```bash
if [[ "$STATE" == "new" ]]; then
  echo "⏭️  Skipping subscription (state: new)"
  SUBSCRIPTION_ID=""
  PLAN_ID=""
else
  echo "📦 Step 2/4: Creating subscription..."
  
  # Get SKU and address from config
  SKU=$(get_plan_sku "$MARKET" "$MEALS" "$PEOPLE")
  ADDRESS_JSON=$(get_test_address "$MARKET")
  
  # Get payment method configuration for market
  PAYMENT_METHOD=$(get_payment_method_config "$MARKET")
  echo "✅ Payment method: $PAYMENT_METHOD"
  
  # Parse address fields for checkout (reduce command substitution)
  ADDRESS_LINE1=""
  ADDRESS_CITY=""
  ADDRESS_ZIP=""
  ADDRESS_REGION=""
  
  if [[ "$JSON_PARSER" == "jq" ]]; then
    read -r ADDRESS_LINE1 ADDRESS_CITY ADDRESS_ZIP ADDRESS_REGION <<< "$(echo "$ADDRESS_JSON" | jq -r '[.line1, .city, (.zip // .postcode // .eircode), (.region // .state // "")] | @tsv')"
  else
    ADDRESS_LINE1=$(echo "$ADDRESS_JSON" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('line1', ''))")
    ADDRESS_CITY=$(echo "$ADDRESS_JSON" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('city', ''))")
    ADDRESS_ZIP=$(echo "$ADDRESS_JSON" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('zip') or d.get('postcode') or d.get('eircode', ''))")
    ADDRESS_REGION=$(echo "$ADDRESS_JSON" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('region') or d.get('state', ''))")
  fi
  
  # Step 4a & 4b: Create cart and fetch delivery slots in parallel (saves ~2-3s)
  TIMESTAMP_MS=$(( $(date +%s) * 1000 ))
  
  # Create temporary files for parallel execution
  CART_TMP="/tmp/cart_response_$$"
  DELIVERY_TMP="/tmp/delivery_response_$$"
  
  # Run in parallel
  curl -s --max-time 8 -X POST \
    "https://www-staging.hellofresh.com/gw/cart/items" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" \
    -H 'content-type: application/json' \
    -d "{\"items\":[{\"sku\":\"${SKU}\",\"quantity\":1,\"metadata\":{\"mealsPreset\":\"chefschoice\",\"timestamp\":${TIMESTAMP_MS},\"productFamily\":\"classic\"}}],\"country\":\"${MARKET}\"}" > "$CART_TMP" &
  CART_PID=$!
  
  # Fetch delivery slots with parameters matching browser request
  # Using family=classic-box-t25 and customerPriority=active_oneoff (critical for checkout flow)
  curl -s --max-time 8 -X GET \
    "https://www-staging.hellofresh.com/gw/api/delivery_dates_options?country=${MARKET}&customerPriority=active_oneoff&family=classic-box-t25&locale=en-US&numDeliveries=20&zip=${ADDRESS_ZIP}" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" > "$DELIVERY_TMP" &
  DELIVERY_PID=$!
  
  # Wait for both to complete
  wait $CART_PID
  wait $DELIVERY_PID
  
  # Read responses
  CART_RESPONSE=$(cat "$CART_TMP")
  DELIVERY_RESPONSE=$(cat "$DELIVERY_TMP")
  
  # Cleanup temp files
  rm -f "$CART_TMP" "$DELIVERY_TMP"
  
  # DEBUG: Log delivery slots API response
  echo ""
  echo "🔍 DEBUG: Delivery Slots API Response"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "Query: country=US&customerPriority=active_oneoff&family=classic-box-t25&locale=en-US&numDeliveries=20&zip=${ADDRESS_ZIP}"
  echo ""
  
  if [[ "$JSON_PARSER" == "jq" ]]; then
    SLOT_COUNT=$(echo "$DELIVERY_RESPONSE" | jq -r '.items | length // 0')
    echo "Total slots returned: $SLOT_COUNT"
    
    if [[ "$SLOT_COUNT" -gt 0 ]]; then
      echo ""
      echo "Slot details (first 5):"
      echo "$DELIVERY_RESPONSE" | jq -r '.items[0:5][] | {
        date: .deliveryDate.deliveryDate,
        availableOnCheckout: .deliveryDate.deliveryOption.availableOnCheckout,
        status: .deliveryDate.status,
        handle: .deliveryDate.deliveryOption.handle,
        day: .deliveryDate.deliveryOption.deliveryDay
      }' 2>/dev/null || echo "  (Could not parse slot details)"
    else
      echo ""
      echo "Raw response preview:"
      echo "${DELIVERY_RESPONSE:0:500}"
    fi
  else
    SLOT_COUNT=$(echo "$DELIVERY_RESPONSE" | python3 -c "import sys, json; print(len(json.load(sys.stdin).get('items', [])))" 2>/dev/null || echo "0")
    echo "Total slots returned: $SLOT_COUNT"
    echo ""
    echo "Raw response preview:"
    echo "${DELIVERY_RESPONSE:0:500}"
  fi
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""
  
  # Parse cart ID
  CART_ID=""
  if [[ "$JSON_PARSER" == "jq" ]]; then
    CART_ID=$(echo "$CART_RESPONSE" | jq -r '.id // empty')
  else
    CART_ID=$(echo "$CART_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('id', ''))")
  fi
  
  if [[ -z "$CART_ID" ]] || [[ "$CART_ID" == "null" ]]; then
    echo "❌ Failed to create cart"
    echo "Response: ${CART_RESPONSE:0:200}"
    STATE="new"
    SUBSCRIPTION_ID=""
    PLAN_ID=""
  else
    echo "✅ Cart created: $CART_ID"
    
    # Extract first AVAILABLE delivery slot with proper filtering
    # Check for availableOnCheckout=true and good utilizationFactor
    DELIVERY_DATE=""
    DELIVERY_OPTION=""
    DELIVERY_DAY=""
    
    if [[ "$JSON_PARSER" == "jq" ]]; then
      # Filter for available slots: availableOnCheckout=true and status=RUNNING
      read -r DELIVERY_DATE DELIVERY_OPTION DELIVERY_DAY <<< "$(echo "$DELIVERY_RESPONSE" | jq -r '
        .items[]? 
        | select(.deliveryDate.deliveryOption.availableOnCheckout == true) 
        | select(.deliveryDate.status == "RUNNING" or .deliveryDate.status == null)
        | [.deliveryDate.deliveryDate, .deliveryDate.deliveryOption.handle, .deliveryDate.deliveryOption.deliveryDay] 
        | @tsv
      ' | head -1)"
    else
      # Python fallback - try to find first available slot
      DELIVERY_DATE=$(echo "$DELIVERY_RESPONSE" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for item in data.get('items', []):
  dd = item.get('deliveryDate', {})
  opt = dd.get('deliveryOption', {})
  if opt.get('availableOnCheckout', True):
    print(dd.get('deliveryDate', ''))
    break
" 2>/dev/null)
      
      if [[ -n "$DELIVERY_DATE" ]] && [[ "$DELIVERY_DATE" != "null" ]]; then
        DELIVERY_OPTION=$(echo "$DELIVERY_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d['items'][0]['deliveryDate']['deliveryOption']['handle'])" 2>/dev/null)
        DELIVERY_DAY=$(echo "$DELIVERY_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d['items'][0]['deliveryDate']['deliveryOption']['deliveryDay'])" 2>/dev/null)
      fi
    fi
    
    if [[ -z "$DELIVERY_DATE" ]] || [[ "$DELIVERY_DATE" == "null" ]]; then
      echo "❌ No available delivery slots found"
      echo "ℹ️  ZIP code queried: ${ADDRESS_ZIP}"
      echo "ℹ️  Staging may not have delivery slots configured for upcoming dates"
      echo "ℹ️  Check Logistics Configurator: https://staging-operations.hellofresh.com"
      STATE="new"
      SUBSCRIPTION_ID=""
      PLAN_ID=""
    else
      echo "✅ Delivery slot found: $DELIVERY_DATE"
      
      # Generate HFPublicID (user tracking UUID) - required for price calculation
      HF_PUBLIC_ID=$(python3 -c "import uuid; print(str(uuid.uuid4()))")
      echo "✅ Generated HFPublicID: $HF_PUBLIC_ID"
      
      # Build payment JSON based on market
      if [[ "$PAYMENT_METHOD" == "Braintree_CreditCard" ]]; then
        PAYMENT_JSON='"payment": {
            "method": "Braintree_CreditCard",
            "paymentMethodNonce": "fake-valid-nonce"
          }'
      elif [[ "$PAYMENT_METHOD" == "ProcessOut_CreditCard" ]]; then
        PAYMENT_JSON='"payment": {
            "method": "ProcessOut_CreditCard",
            "card_id": "card_test_4242424242424242"
          }'
      else
        PAYMENT_JSON='"payment": {
            "method": "Adyen_CreditCard",
            "shouldSendExtra3DSParms": false,
            "checkout_integration": true,
            "payment_details": {
              "type": "scheme",
              "encryptedCardNumber": "test_4111111111111111",
              "encryptedExpiryYear": "test_2030",
              "encryptedExpiryMonth": "test_03",
              "encryptedSecurityCode": "test_737"
            }
          }'
      fi
      
      # Step 4c: Checkout with retry logic (retries checkout only, not full flow)
      MAX_CHECKOUT_RETRIES=2
      CHECKOUT_ATTEMPT=0
      CHECKOUT_SUCCESS=false
      
      while [[ $CHECKOUT_ATTEMPT -le $MAX_CHECKOUT_RETRIES ]]; do
        CHECKOUT_ATTEMPT=$((CHECKOUT_ATTEMPT + 1))
        
        # Recreate cart if this is a retry
        if [[ $CHECKOUT_ATTEMPT -gt 1 ]]; then
          echo "⚠️  Retry $CHECKOUT_ATTEMPT/$((MAX_CHECKOUT_RETRIES + 1)): Creating new cart..."
          TIMESTAMP_MS=$(( $(date +%s) * 1000 ))
          CART_RESPONSE=$(curl -s --max-time 8 -X POST \
            "https://www-staging.hellofresh.com/gw/cart/items" \
            -H "Authorization: Bearer ${ACCESS_TOKEN}" \
            -H 'content-type: application/json' \
            -d "{\"items\":[{\"sku\":\"${SKU}\",\"quantity\":1,\"metadata\":{\"mealsPreset\":\"chefschoice\",\"timestamp\":${TIMESTAMP_MS},\"productFamily\":\"classic\"}}],\"country\":\"${MARKET}\"}")
          
          if [[ "$JSON_PARSER" == "jq" ]]; then
            CART_ID=$(echo "$CART_RESPONSE" | jq -r '.id // empty')
          else
            CART_ID=$(echo "$CART_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('id', ''))")
          fi
          
          if [[ -z "$CART_ID" ]] || [[ "$CART_ID" == "null" ]]; then
            echo "❌ Failed to recreate cart for retry"
            break
          fi
          echo "✅ New cart created: $CART_ID"
          sleep 1
        fi
        
        CHECKOUT_RESPONSE=$(curl -s --max-time 12 -X POST \
          "https://www-staging.hellofresh.com/gw/order/legacy/process" \
          -H "Authorization: Bearer ${ACCESS_TOKEN}" \
          -H 'content-type: application/json' \
          -d "{\"cartID\":\"${CART_ID}\",\"customerEmail\":\"${EMAIL}\",\"hfPublicId\":\"${HF_PUBLIC_ID}\",\"useSameAddressForBilling\":true,\"personal\":{\"firstName\":\"${FIRST_NAME}\",\"lastName\":\"${LAST_NAME}\"},${PAYMENT_JSON},\"address\":{\"address1\":\"${ADDRESS_LINE1}\",\"city\":\"${ADDRESS_CITY}\",\"postcode\":\"${ADDRESS_ZIP}\",\"region\":\"${ADDRESS_REGION}\",\"firstName\":\"${FIRST_NAME}\",\"lastName\":\"${LAST_NAME}\",\"phone\":\"+1234567890\",\"customerId\":\"${CUSTOMER_ID}\",\"isBilling\":true,\"isOffice\":false,\"billToSameAddress\":true,\"isDefaultBilling\":false,\"isDefaultShipping\":false},\"selectedAddress\":{\"isBilling\":true,\"isOffice\":false,\"billToSameAddress\":true,\"isDefaultBilling\":false,\"isDefaultShipping\":false},\"billingAddress\":{\"isBilling\":true,\"isOffice\":false,\"billToSameAddress\":true,\"isDefaultBilling\":false,\"isDefaultShipping\":false},\"deliveryMoments\":{\"${TIMESTAMP_MS}\":{\"deliveryTime\":\"${DELIVERY_OPTION}\",\"firstDelivery\":\"${DELIVERY_DATE}\",\"sku\":\"${SKU}\"}},\"subscription\":{\"delivery_time\":\"\",\"delivery_weekday\":${DELIVERY_DAY}},\"project\":\"HF\",\"country\":\"${MARKET,,}\",\"forwardURL\":\"https://www-staging.hellofresh.com/checkout-api/finish/placeorder/\",\"customerId\":\"${CUSTOMER_ID}\",\"couponCode\":null,\"freeAddOn\":false}")
        
        # Check for errors and classify them
        if echo "$CHECKOUT_RESPONSE" | grep -q '"code":9904'; then
          # 9904 = validation error, don't retry
          echo "❌ Checkout validation error (9904) - check payload"
          echo "Response: ${CHECKOUT_RESPONSE:0:500}"
          break
        elif echo "$CHECKOUT_RESPONSE" | grep -q "profile-service"; then
          # Profile-service error - retryable
          if [[ $CHECKOUT_ATTEMPT -le $MAX_CHECKOUT_RETRIES ]]; then
            echo "⚠️  Profile-service error, retrying..."
            sleep 2
            continue
          else
            echo "❌ Profile-service error persists after $((MAX_CHECKOUT_RETRIES + 1)) attempts"
            echo "Response: ${CHECKOUT_RESPONSE:0:500}"
            break
          fi
        elif echo "$CHECKOUT_RESPONSE" | grep -q '"code":9900'; then
          # 9900 generic error - fail fast
          echo "❌ Checkout error 9900 (non-retryable)"
          echo "Response: ${CHECKOUT_RESPONSE:0:500}"
          break
        elif echo "$CHECKOUT_RESPONSE" | grep -q '"code":9901'; then
          # HFPublicID error - shouldn't happen but fail fast
          echo "❌ HFPublicID error (shouldn't happen)"
          echo "Response: ${CHECKOUT_RESPONSE:0:500}"
          break
        else
          # Success or unknown response
          CHECKOUT_SUCCESS=true
          echo "✅ Checkout completed"
          break
        fi
      done
      
      if [[ "$CHECKOUT_SUCCESS" != "true" ]]; then
        STATE="new"
        SUBSCRIPTION_ID=""
        PLAN_ID=""
      else
        # Poll for subscription with timeout (saves 1-2s on fast staging)
        SUB_COUNT=0
        SUBSCRIPTION_ID=""
        PLAN_ID=""
        
        for poll_attempt in 1 2 3; do
          sleep 1
          
          SUBS_CHECK=$(curl -s --max-time 8 -X GET \
            "https://www-staging.hellofresh.com/gw/api/customers/me/subscriptions?country=${MARKET}" \
            -H "Authorization: Bearer ${ACCESS_TOKEN}")
          
          if [[ "$JSON_PARSER" == "jq" ]]; then
            SUB_COUNT=$(echo "$SUBS_CHECK" | jq -r '.count // 0')
            if [[ "$SUB_COUNT" -gt 0 ]]; then
              read -r SUBSCRIPTION_ID PLAN_ID <<< "$(echo "$SUBS_CHECK" | jq -r '[.items[0].id, (.items[0].customerPlanId // .items[0].customer_plan_id // .items[0].planId)] | @tsv')"
              echo "✅ Subscription found after ${poll_attempt}s (ID: ${SUBSCRIPTION_ID})"
              break
            fi
          else
            SUB_COUNT=$(echo "$SUBS_CHECK" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('count', 0))")
            if [[ "$SUB_COUNT" -gt 0 ]]; then
              SUBSCRIPTION_ID=$(echo "$SUBS_CHECK" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d['items'][0]['id'])")
              PLAN_ID=$(echo "$SUBS_CHECK" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d['items'][0].get('customerPlanId') or d['items'][0].get('customer_plan_id') or d['items'][0].get('planId', ''))")
              echo "✅ Subscription found after ${poll_attempt}s (ID: ${SUBSCRIPTION_ID})"
              break
            fi
          fi
        done
        
        if [[ "$SUB_COUNT" -eq 0 ]]; then
          echo "⚠️  Checkout completed but subscription not found after 3s"
          echo "Checkout response: ${CHECKOUT_RESPONSE:0:300}"
          STATE="new"
        fi
      fi
    fi
  fi
fi
```

### Step 5: Manipulate State

See `api/state-management.md` for API details.

**Cancel subscription:**

```bash
if [[ "$STATE" == "cancelled" ]] && [[ -n "$PLAN_ID" ]]; then
  echo "🚫 Step 3/4: Cancelling subscription..."
  
  CANCEL_RESPONSE=$(curl -s --max-time 10 -X POST "https://www-staging.hellofresh.com/gw/api/plans/${PLAN_ID}/cancel?country=${MARKET}" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" \
    -H 'content-type: application/json' \
    -d "{\"reason\":{\"text\":\"Test user creation\"}}")
  
  echo "✅ Subscription cancelled"
fi
```

**Pause subscription:**

```bash
if [[ "$STATE" == "paused" ]] && [[ -n "$PLAN_ID" ]]; then
  echo "⏸️  Step 3/4: Pausing subscription..."
  
  PAUSE_RESPONSE=$(curl -s --max-time 10 -X POST "https://www-staging.hellofresh.com/gw/api/plans/${PLAN_ID}/pause?country=${MARKET}" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" \
    -H 'content-type: application/json' \
    -d "{\"weeks\":4}")
  
  echo "✅ Subscription paused"
fi
```

**Enroll in loyalty:**

```bash
if [[ -n "$LOYALTY_TIER" ]]; then
  echo "🎁 Enrolling in loyalty program (${LOYALTY_TIER} tier)..."
  
  # API call for loyalty enrollment (if available in market)
  # Note: Not all markets support loyalty API
  
  echo "✅ Loyalty enrollment attempted"
fi
```

### Step 7: Display Summary

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ Test user created successfully!"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "📧 Email:          ${EMAIL}"
echo "🔑 Password:       ${PASSWORD}"
echo "🆔 Customer ID:    ${CUSTOMER_ID}"
if [[ -n "$SUBSCRIPTION_ID" ]]; then
  echo "🎫 Subscription ID: ${SUBSCRIPTION_ID}"
  echo "📦 Plan:           ${MEALS} meals for ${PEOPLE} people"
fi
echo "🌍 Market:         ${MARKET}"
echo "📊 State:          ${STATE}"
if [[ -n "$LOYALTY_TIER" ]]; then
  echo "🎁 Loyalty:        ${LOYALTY_TIER} tier"
fi
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

## Helper Functions

**IMPORTANT:** These functions must be defined before running the workflow steps.

```bash
# Get business division for market
get_business_division() {
  local market=$1
  local brand=""
  
  case "$market" in
    MR) brand="BRAND_GOODCHOP" ;;
    GN) brand="BRAND_GREENCHEF" ;;
    YE) brand="BRAND_YOUFOODZ" ;;
    CK) brand="BRAND_CHEFS_PLATE" ;;
    FJ) brand="BRAND_FACTOR" ;;
    ER) brand="BRAND_EVERYPLATE" ;;
    *) brand="BRAND_HELLOFRESH" ;;
  esac
  
  echo "{\"brand\":\"${brand}\",\"region_code\":\"${market}\"}"
}

# Generate plan SKU
get_plan_sku() {
  local market=$1
  local meals=$2
  local people=$3
  
  # Whitelabel-specific SKU formats
  case "$market" in
    MR)
      # Good Chop - use MIX product
      echo "${market}-MIX-${meals}-${people}-0"
      ;;
    FJ|YE)
      # Ready-to-eat - use portions instead of people
      echo "${market}-RTE-${meals}-1-0"
      ;;
    *)
      # Standard meal kit format
      echo "${market}-CBT25-${meals}-${people}-0"
      ;;
  esac
}

# Get test address for market
get_test_address() {
  local market=$1
  
  case "$market" in
    US|MR|GN|FJ|ER)
      cat <<'EOF'
{
  "line1": "123 Test Street",
  "city": "Schenectady",
  "region": "NY",
  "zip": "12345",
  "country": "US"
}
EOF
      ;;
    
    DE)
      cat <<'EOF'
{
  "line1": "Teststraße 123",
  "city": "Berlin",
  "zip": "10115",
  "country": "DE"
}
EOF
      ;;
    
    GB|KN)
      cat <<'EOF'
{
  "line1": "123 Test Street",
  "city": "London",
  "postcode": "SW1A 1AA",
  "country": "GB"
}
EOF
      ;;
    
    AU|YE|AO)
      cat <<'EOF'
{
  "line1": "123 Test Street",
  "city": "Sydney",
  "region": "NSW",
  "postcode": "2000",
  "country": "AU"
}
EOF
      ;;
    
    CA|CK|CG|CF)
      cat <<'EOF'
{
  "line1": "123 Test Street",
  "city": "Toronto",
  "province": "ON",
  "postalCode": "M5H 2N2",
  "country": "CA"
}
EOF
      ;;
    
    NL)
      cat <<'EOF'
{
  "line1": "Teststraat 123",
  "city": "Amsterdam",
  "zip": "1012 AB",
  "country": "NL"
}
EOF
      ;;
    
    BE)
      cat <<'EOF'
{
  "line1": "Teststraat 123",
  "city": "Brussels",
  "zip": "1000",
  "country": "BE"
}
EOF
      ;;
    
    AT)
      cat <<'EOF'
{
  "line1": "Teststraße 123",
  "city": "Wien",
  "zip": "1010",
  "country": "AT"
}
EOF
      ;;
    
    CH)
      cat <<'EOF'
{
  "line1": "Teststrasse 123",
  "city": "Zürich",
  "zip": "8001",
  "country": "CH"
}
EOF
      ;;
    
    FR)
      cat <<'EOF'
{
  "line1": "123 Rue de Test",
  "city": "Paris",
  "zip": "75001",
  "country": "FR"
}
EOF
      ;;
    
    IT)
      cat <<'EOF'
{
  "line1": "Via del Test 123",
  "city": "Roma",
  "zip": "00100",
  "country": "IT"
}
EOF
      ;;
    
    ES)
      cat <<'EOF'
{
  "line1": "Calle de Test 123",
  "city": "Madrid",
  "zip": "28001",
  "country": "ES"
}
EOF
      ;;
    
    SE)
      cat <<'EOF'
{
  "line1": "Testgatan 123",
  "city": "Stockholm",
  "zip": "111 20",
  "country": "SE"
}
EOF
      ;;
    
    DK)
      cat <<'EOF'
{
  "line1": "Testvej 123",
  "city": "København",
  "zip": "1000",
  "country": "DK"
}
EOF
      ;;
    
    NO)
      cat <<'EOF'
{
  "line1": "Testveien 123",
  "city": "Oslo",
  "zip": "0010",
  "country": "NO"
}
EOF
      ;;
    
    IE)
      cat <<'EOF'
{
  "line1": "123 Test Street",
  "city": "Dublin",
  "eircode": "D01 F5P2",
  "country": "IE"
}
EOF
      ;;
    
    NZ)
      cat <<'EOF'
{
  "line1": "123 Test Street",
  "city": "Auckland",
  "postcode": "1010",
  "country": "NZ"
}
EOF
      ;;
    
    *)
      echo "❌ No test address configured for market: $market" >&2
      return 1
      ;;
  esac
}

# Get test payment method configuration per market
get_payment_method_config() {
  local market=$1
  
  # US and US-based brands use Braintree
  case "$market" in
    US|MR|GN|FJ|ER|CG)
      echo "Braintree_CreditCard"
      ;;
    IT|SE|BE|DE|AT|DK|ES|CH|AU)
      # ProcessOut markets
      echo "ProcessOut_CreditCard"
      ;;
    *)
      # All other markets use Adyen (CA, FR, IE, GB, NO, NL, CK, etc.)
      echo "Adyen_CreditCard"
      ;;
  esac
}

# Get payment details JSON per provider
get_payment_details() {
  local payment_method=$1
  
  if [[ "$payment_method" == "Braintree_CreditCard" ]]; then
    # Braintree uses paymentMethodNonce
    cat <<'EOF'
{
  "method": "Braintree_CreditCard",
  "paymentMethodNonce": "fake-valid-nonce"
}
EOF
  elif [[ "$payment_method" == "ProcessOut_CreditCard" ]]; then
    # ProcessOut uses test card token
    cat <<'EOF'
{
  "method": "ProcessOut_CreditCard",
  "card_id": "card_test_4242424242424242"
}
EOF
  else
    # Adyen uses encrypted test card details
    cat <<'EOF'
{
  "method": "Adyen_CreditCard",
  "shouldSendExtra3DSParms": false,
  "checkout_integration": true,
  "payment_details": {
    "type": "scheme",
    "encryptedCardNumber": "test_4111111111111111",
    "encryptedExpiryYear": "test_2030",
    "encryptedExpiryMonth": "test_03",
    "encryptedSecurityCode": "test_737"
  }
}
EOF
  fi
}
```

## Error Handling

**Common Errors:**

1. **VPN Not Connected:**
   ```
   ❌ VPN not connected or staging gateway unreachable
   → Connect to staging VPN and try again
   ```

2. **API Error:**
   ```
   ❌ Failed to create user account
   Response: {"error": "..."}
   → Check API response for details
   ```

3. **Subscription Creation Failed:**
   ```
   ❌ Failed to create subscription
   → May need to update test payment method or address
   ```

## Usage Examples

### Example 1: Basic New User (No Subscription - Default)
```bash
/create-test-user
```
Creates US user account without subscription (fastest option).

### Example 2: Active User with Subscription
```bash
/create-test-user --state active
```
Creates US user with active subscription (default 2-meals-2-people plan).

### Example 3: Cancelled User with Custom Plan
```bash
/create-test-user --state cancelled --plan "3-meals-4-people"
```
Creates US user with 3 meals for 4 people, then cancels.

### Example 4: Loyalty-Enrolled User
```bash
/create-test-user --state active --loyalty-tier "gold"
```
Creates active US user and attempts loyalty enrollment.

### Example 5: Paused Subscription
```bash
/create-test-user --state paused
```
Creates US user with paused subscription.

## Important Notes

1. **VPN Required:** Must be connected to staging VPN
2. **Unique Emails:** Uses timestamp to ensure unique emails
3. **Test Cards:** Uses test payment methods for US market
4. **Payment Tokens:** Uses official Braintree sandbox nonces ([Braintree Testing Docs](https://developer.paypal.com/braintree/docs/reference/general/testing/ruby/#payment-method-nonces))
5. **Meal Preferences:** Cart includes `"chefschoice"` preset (validated via shopping-platform-service for US market)
6. **State Changes:** Cancel/pause operations are destructive
7. **Access Tokens:** Tokens expire, use promptly for API testing
8. **US Only:** Currently only supports US market

## Payment Test Values

The command uses official payment processor test values for US market:

**Braintree (US):**
- `fake-valid-nonce` - Static sandbox nonce (never expires)
- Official Braintree test value for sandbox transactions

Reference: [Braintree Testing Documentation](https://developer.paypal.com/braintree/docs/reference/general/testing/ruby/#payment-method-nonces)

## Related Commands

- `/setup-qa-assistant` - Setup prerequisites and dependencies
- `/generate-e2e-tests` - Generate tests that can use created users
- `/verify-ac` - Verify ACs using test users
