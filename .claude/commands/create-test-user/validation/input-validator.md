# Input Validator

Validate and parse user input parameters for test user creation.

## Parameters

### State Parameter
- **Valid values:** `new`, `active`, `cancelled`, `paused`
- **Default:** `new`
- **Purpose:** Determines subscription state

```bash
STATE="${STATE:-new}"

if [[ ! "$STATE" =~ ^(new|active|cancelled|paused)$ ]]; then
  echo "❌ Invalid state: $STATE"
  echo "Valid states: new, active, cancelled, paused"
  exit 1
fi
```

### Plan Parameter
- **Format:** `X-meals-Y-people` (e.g., `2-meals-2-people`)
- **Default:** `2-meals-2-people`
- **Purpose:** Subscription plan configuration

```bash
PLAN="${PLAN:-2-meals-2-people}"

# Parse meals and people count
MEALS=$(echo "$PLAN" | cut -d'-' -f1)
PEOPLE=$(echo "$PLAN" | cut -d'-' -f3)

# Validate format
if [[ ! "$PLAN" =~ ^[0-9]+-meals-[0-9]+-people$ ]]; then
  echo "❌ Invalid plan format: $PLAN"
  echo "Expected format: X-meals-Y-people (e.g., 2-meals-2-people)"
  exit 1
fi
```

### Loyalty Tier Parameter
- **Valid values:** `bronze`, `silver`, `gold`, `platinum`
- **Optional:** Only used when specified
- **Purpose:** Loyalty program enrollment

```bash
if [[ -n "$LOYALTY_TIER" ]]; then
  if [[ ! "$LOYALTY_TIER" =~ ^(bronze|silver|gold|platinum)$ ]]; then
    echo "❌ Invalid loyalty tier: $LOYALTY_TIER"
    echo "Valid tiers: bronze, silver, gold, platinum"
    exit 1
  fi
fi
```

## Market Configuration

Market is hardcoded to US:

```bash
MARKET="US"
```

## Output

After validation, the following variables are set:
- `STATE` - User state (validated)
- `PLAN` - Full plan string (validated)
- `MEALS` - Number of meals (parsed from plan)
- `PEOPLE` - Number of people (parsed from plan)
- `LOYALTY_TIER` - Loyalty tier (validated if provided)
- `MARKET` - Fixed to "US"

## Error Handling

All validation errors immediately exit with:
1. Clear error message
2. Expected format/values
3. Exit code 1
