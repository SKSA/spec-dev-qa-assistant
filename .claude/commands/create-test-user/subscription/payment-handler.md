# Payment Handler

Handle test payment processing for subscriptions.

## Test Payment Methods

### Braintree (US Market)

Uses official Braintree sandbox test nonce:

```bash
PAYMENT_NONCE="fake-valid-nonce"
```

**Properties:**
- Static sandbox nonce (never expires)
- Official Braintree test value
- No real money charged
- Always succeeds in sandbox

**Reference:** [Braintree Testing Documentation](https://developer.paypal.com/braintree/docs/reference/general/testing/ruby/#payment-method-nonces)

## Payment Processing

### Build Payment JSON

```bash
get_payment_json() {
  local market=$1
  
  # US uses Braintree
  cat <<EOF
{
  "paymentType": "braintree",
  "nonce": "fake-valid-nonce"
}
EOF
}

PAYMENT_JSON=$(get_payment_json "$MARKET")
```

### Address Configuration

See `config/addresses.md` for market-specific test addresses.

**US Test Address:**
```json
{
  "address1": "123 Test Street",
  "city": "New York",
  "postcode": "10001",
  "region": "NY",
  "firstName": "Test",
  "lastName": "User",
  "phone": "+1234567890"
}
```

## Checkout Flow

### Complete Checkout Request

```bash
CHECKOUT_DATA=$(cat <<EOF
{
  "cartID": "${CART_ID}",
  "customerEmail": "${EMAIL}",
  "hfPublicId": "${HF_PUBLIC_ID}",
  "useSameAddressForBilling": true,
  "personal": {
    "firstName": "${FIRST_NAME}",
    "lastName": "${LAST_NAME}"
  },
  ${PAYMENT_JSON},
  "address": {
    "address1": "${ADDRESS_LINE1}",
    "city": "${ADDRESS_CITY}",
    "postcode": "${ADDRESS_ZIP}",
    "region": "${ADDRESS_REGION}",
    "firstName": "${FIRST_NAME}",
    "lastName": "${LAST_NAME}",
    "phone": "+1234567890"
  },
  "deliveryMoments": {
    "${TIMESTAMP_MS}": {
      "deliveryTime": "${DELIVERY_OPTION}",
      "firstDelivery": "${FIRST_DELIVERY_DATE}"
    }
  }
}
EOF
)

CHECKOUT_RESPONSE=$(curl -s --max-time 30 -X POST \
  "https://www-staging.hellofresh.com/gw/payment/checkout" \
  -H 'content-type: application/json' \
  -H "authorization: Bearer ${ACCESS_TOKEN}" \
  -d "$CHECKOUT_DATA")
```

### Parse Response

```bash
if [[ "$JSON_PARSER" == "jq" ]]; then
  SUBSCRIPTION_ID=$(echo "$CHECKOUT_RESPONSE" | jq -r '.subscriptionId // empty')
  PLAN_ID=$(echo "$CHECKOUT_RESPONSE" | jq -r '.planId // empty')
else
  SUBSCRIPTION_ID=$(echo "$CHECKOUT_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('subscriptionId', ''))")
  PLAN_ID=$(echo "$CHECKOUT_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('planId', ''))")
fi
```

## Validation

```bash
if [[ -z "$SUBSCRIPTION_ID" ]] || [[ "$SUBSCRIPTION_ID" == "null" ]]; then
  echo "❌ Failed to create subscription"
  echo "Response: ${CHECKOUT_RESPONSE:0:500}"
  exit 1
fi
```

## Error Handling

### Payment Declined
- **Cause:** Invalid test nonce (shouldn't happen with fake-valid-nonce)
- **Message:** "Payment declined"
- **Action:** Verify using correct test nonce

### Address Validation Failed
- **Cause:** Invalid postal code or state
- **Message:** "Invalid address"
- **Action:** Check address format for market

### Timeout
- **Cause:** Slow staging environment
- **Message:** "Checkout timeout"
- **Action:** Increase timeout or retry

## Notes

- Test nonces never charge real money
- Sandbox environment only
- No 3DS authentication required
- Payments process immediately
- No fraud checks in staging
