# Result Formatter

Format and display test user creation results.

## Display Summary

### Success Header

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ Test user created successfully!"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
```

### User Credentials

```bash
echo "📧 Email:          ${EMAIL}"
echo "🔑 Password:       ${PASSWORD}"
echo "🆔 Customer ID:    ${CUSTOMER_ID}"
```

### Subscription Details (if applicable)

```bash
if [[ -n "$SUBSCRIPTION_ID" ]]; then
  echo "🎫 Subscription ID: ${SUBSCRIPTION_ID}"
  echo "📦 Plan:           ${MEALS} meals for ${PEOPLE} people"
fi
```

### Market and State

```bash
echo "🌍 Market:         ${MARKET}"
echo "📊 State:          ${STATE}"
```

### Loyalty Information (if applicable)

```bash
if [[ -n "$LOYALTY_TIER" ]]; then
  echo "🎁 Loyalty:        ${LOYALTY_TIER} tier"
fi
```

### Footer

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

## Complete Output Example

### New User (No Subscription)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Test user created successfully!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📧 Email:          hfuser-11-153045@hf.com
🔑 Password:       qwerty123
🆔 Customer ID:    12345678
🌍 Market:         US
📊 State:          new

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Active User with Subscription

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

### With Loyalty Enrollment

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Test user created successfully!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📧 Email:          hfuser-11-153045@hf.com
🔑 Password:       qwerty123
🆔 Customer ID:    12345678
🎫 Subscription ID: 87654321
📦 Plan:           3 meals for 4 people
🌍 Market:         US
📊 State:          cancelled
🎁 Loyalty:        gold tier

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Error Display

### Error Header

```bash
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "❌ Failed to create test user"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
```

### Error Details

```bash
echo "Error: ${ERROR_MESSAGE}"
echo ""
echo "Troubleshooting:"
echo "  • Check VPN connection to staging"
echo "  • Verify staging credentials are valid"
echo "  • Review error response above"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

## Output Variables Used

Required variables for formatting:
- `EMAIL` - User email
- `PASSWORD` - User password
- `CUSTOMER_ID` - Customer ID
- `MARKET` - Market code
- `STATE` - User state

Optional variables:
- `SUBSCRIPTION_ID` - Subscription ID (if subscription created)
- `MEALS` - Number of meals (if subscription created)
- `PEOPLE` - Number of people (if subscription created)
- `LOYALTY_TIER` - Loyalty tier (if enrolled)

## Notes

- Use emoji for visual clarity
- Keep output concise and scannable
- Display most important info first (email/password)
- Group related information together
- Use consistent formatting across all outputs
