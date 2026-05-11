# State Manager

Handle subscription state transitions (cancel, pause).

## State Transitions

### Active State

No action needed - subscription starts as active after checkout:

```bash
if [[ "$STATE" == "active" ]]; then
  echo "✅ Subscription is active"
  # No additional action needed
fi
```

### Cancel Subscription

```bash
if [[ "$STATE" == "cancelled" ]]; then
  echo "🚫 Cancelling subscription..."
  
  CANCEL_RESPONSE=$(curl -s --max-time 10 -X POST \
    "https://www-staging.hellofresh.com/gw/subscriptions/${SUBSCRIPTION_ID}/cancel" \
    -H 'content-type: application/json' \
    -H "authorization: Bearer ${ACCESS_TOKEN}" \
    -d '{
      "reason": "Test user - automated cancellation",
      "feedback": "Testing cancellation flow"
    }')
  
  # Validate cancellation
  STATUS=$(echo "$CANCEL_RESPONSE" | jq -r '.status')
  
  if [[ "$STATUS" == "cancelled" ]]; then
    echo "✅ Subscription cancelled"
  else
    echo "⚠️  Cancellation may have failed"
    echo "Response: ${CANCEL_RESPONSE:0:200}"
  fi
fi
```

### Pause Subscription

```bash
if [[ "$STATE" == "paused" ]]; then
  echo "⏸️  Pausing subscription..."
  
  # Calculate pause dates (pause for 2 weeks)
  PAUSE_START=$(date -u +%Y-%m-%d)
  PAUSE_END=$(date -u -d '+14 days' +%Y-%m-%d)
  
  PAUSE_RESPONSE=$(curl -s --max-time 10 -X POST \
    "https://www-staging.hellofresh.com/gw/subscriptions/${SUBSCRIPTION_ID}/pause" \
    -H 'content-type: application/json' \
    -H "authorization: Bearer ${ACCESS_TOKEN}" \
    -d "{
      \"startDate\": \"${PAUSE_START}\",
      \"endDate\": \"${PAUSE_END}\",
      \"reason\": \"Test user - automated pause\"
    }")
  
  # Validate pause
  STATUS=$(echo "$PAUSE_RESPONSE" | jq -r '.status')
  
  if [[ "$STATUS" == "paused" ]]; then
    echo "✅ Subscription paused (until ${PAUSE_END})"
  else
    echo "⚠️  Pause may have failed"
    echo "Response: ${PAUSE_RESPONSE:0:200}"
  fi
fi
```

## State Validation

After state change, verify the new state:

```bash
validate_state() {
  local expected_state=$1
  local subscription_id=$2
  
  # Fetch current subscription state
  CURRENT_STATE=$(curl -s --max-time 10 \
    "https://www-staging.hellofresh.com/gw/subscriptions/${subscription_id}" \
    -H "authorization: Bearer ${ACCESS_TOKEN}" | jq -r '.status')
  
  if [[ "$CURRENT_STATE" == "$expected_state" ]]; then
    echo "✅ State verified: ${expected_state}"
    return 0
  else
    echo "⚠️  Expected state: ${expected_state}, got: ${CURRENT_STATE}"
    return 1
  fi
}
```

## Cancellation Reasons

Standard cancellation reasons for test users:

```json
{
  "reason": "Test user - automated cancellation",
  "feedback": "Testing cancellation flow",
  "category": "test_automation"
}
```

## Pause Duration

Standard pause period for test users:
- **Duration:** 2 weeks
- **Start:** Immediate
- **End:** +14 days from start

## Error Handling

### Invalid State Transition
- **Cause:** Subscription already in target state
- **Message:** "Cannot transition from X to Y"
- **Action:** Check current state first

### Timing Restrictions
- **Cause:** Too close to delivery cutoff
- **Message:** "Cannot modify subscription at this time"
- **Action:** Adjust pause dates or retry

### API Error
- **Cause:** Network or server issue
- **Message:** "Failed to update subscription"
- **Action:** Retry or check subscription status

## Notes

- State changes are immediate in staging
- Cancellations are permanent (cannot reactivate)
- Pauses have start/end dates
- No prorating or refunds in staging
- State transitions trigger webhook events
