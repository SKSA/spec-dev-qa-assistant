# Subscription Manager

Manage subscription lifecycle for test users.

## Overview

Handles the complete subscription flow:
1. Cart creation with meal preferences
2. Checkout with test payment
3. State transitions (cancel/pause)
4. Loyalty enrollment (optional)

## Subscription Flow

### Skip for 'new' State

```bash
if [[ "$STATE" == "new" ]]; then
  echo "⏭️  Skipping subscription (state: new)"
  exit 0
fi
```

### Step 1: Create Cart

See `subscription/cart-creator.md` for:
- Initialize shopping cart
- Add meal preferences
- Get cart ID

```bash
echo "🛒 Step 2/4: Creating subscription cart..."

CART_RESPONSE=$(curl -s --max-time 10 -X POST \
  "https://www-staging.hellofresh.com/gw/carts" \
  -H 'content-type: application/json' \
  -H "authorization: Bearer ${ACCESS_TOKEN}" \
  -d "{
    \"meals\": ${MEALS},
    \"people\": ${PEOPLE},
    \"preferences\": [\"chefschoice\"]
  }")

CART_ID=$(echo "$CART_RESPONSE" | jq -r '.id')
echo "✅ Cart created (ID: ${CART_ID})"
```

### Step 2: Complete Checkout

See `subscription/payment-handler.md` for:
- Process test payment
- Complete checkout flow
- Get subscription details

```bash
echo "💳 Step 3/4: Completing checkout..."

CHECKOUT_RESPONSE=$(curl -s --max-time 30 -X POST \
  "https://www-staging.hellofresh.com/gw/checkout" \
  -H 'content-type: application/json' \
  -H "authorization: Bearer ${ACCESS_TOKEN}" \
  -d "{
    \"cartId\": \"${CART_ID}\",
    \"paymentMethod\": \"fake-valid-nonce\",
    \"address\": {
      \"line1\": \"123 Test St\",
      \"city\": \"New York\",
      \"state\": \"NY\",
      \"zip\": \"10001\"
    }
  }")

SUBSCRIPTION_ID=$(echo "$CHECKOUT_RESPONSE" | jq -r '.subscriptionId')
PLAN_ID=$(echo "$CHECKOUT_RESPONSE" | jq -r '.planId')

echo "✅ Subscription created (ID: ${SUBSCRIPTION_ID})"
```

### Step 3: State Management

See `subscription/state-manager.md` for handling state transitions.

#### Cancel Subscription

```bash
if [[ "$STATE" == "cancelled" ]]; then
  echo "🚫 Cancelling subscription..."
  
  curl -s --max-time 10 -X POST \
    "https://www-staging.hellofresh.com/gw/subscriptions/${SUBSCRIPTION_ID}/cancel" \
    -H "authorization: Bearer ${ACCESS_TOKEN}"
  
  echo "✅ Subscription cancelled"
fi
```

#### Pause Subscription

```bash
if [[ "$STATE" == "paused" ]]; then
  echo "⏸️  Pausing subscription..."
  
  curl -s --max-time 10 -X POST \
    "https://www-staging.hellofresh.com/gw/subscriptions/${SUBSCRIPTION_ID}/pause" \
    -H "authorization: Bearer ${ACCESS_TOKEN}"
  
  echo "✅ Subscription paused"
fi
```

### Step 4: Loyalty Enrollment (Optional)

```bash
if [[ -n "$LOYALTY_TIER" ]]; then
  echo "🎁 Enrolling in loyalty program (tier: ${LOYALTY_TIER})..."
  
  curl -s --max-time 10 -X POST \
    "https://www-staging.hellofresh.com/gw/loyalty/enroll" \
    -H 'content-type: application/json' \
    -H "authorization: Bearer ${ACCESS_TOKEN}" \
    -d "{
      \"customerId\": \"${CUSTOMER_ID}\",
      \"tier\": \"${LOYALTY_TIER}\"
    }"
  
  echo "✅ Loyalty enrollment attempted"
fi
```

## Output Variables

After subscription creation:
- `SUBSCRIPTION_ID` - Subscription ID
- `PLAN_ID` - Plan ID
- `CART_ID` - Cart ID (intermediate)

## Error Handling

### Cart Creation Failed
- **Cause:** Invalid meal preferences or quota exceeded
- **Message:** "Failed to create cart"
- **Action:** Check meal preferences are valid

### Checkout Failed
- **Cause:** Payment processing or address validation
- **Message:** "Failed to complete checkout"
- **Action:** Verify test payment method

### State Transition Failed
- **Cause:** Invalid state or timing issue
- **Message:** "Failed to ${action} subscription"
- **Action:** Retry or check subscription status

## Notes

- Uses official Braintree test nonce: `fake-valid-nonce`
- Test payments never charge real money
- State transitions are immediate in staging
- Loyalty enrollment is best-effort (may not be available in all markets)
