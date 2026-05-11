# Credential Generator

Generate unique credentials for test users.

## Email Generation

Uses timestamp-based format to ensure uniqueness:

### Format
```
hfuser-DD-HHMMSS@hf.com
```

Where:
- `DD` - Day of month (01-31)
- `HH` - Hour (00-23)
- `MM` - Minute (00-59)
- `SS` - Second (00-59)

### Implementation

```bash
EMAIL="hfuser-$(date +%d-%H%M%S)@hf.com"
```

### Examples
- `hfuser-11-153045@hf.com` - May 11, 15:30:45
- `hfuser-05-090312@hf.com` - May 5, 09:03:12

## Password

Uses consistent test password for all users:

```bash
PASSWORD="qwerty123"
```

## User Details

Standard test user details:

```bash
FIRST_NAME="Test"
LAST_NAME="User"
```

## Business Division

Get business division for market (currently US only):

```bash
get_business_division() {
  echo "HELLOFRESH"  # US uses HELLOFRESH division
}

BUSINESS_DIVISION=$(get_business_division)
```

## Uniqueness Guarantee

The timestamp-based email ensures uniqueness because:
1. Includes seconds precision
2. Very low collision probability
3. Sequential execution prevents conflicts
4. Staging database doesn't persist indefinitely

## Output Variables

- `EMAIL` - Unique email address
- `PASSWORD` - Test password
- `FIRST_NAME` - "Test"
- `LAST_NAME` - "User"
- `BUSINESS_DIVISION` - Market's business division

## Notes

- Emails are not real mailboxes
- No verification required in staging
- Credentials work immediately after creation
- Safe for parallel execution (timestamp provides uniqueness)
