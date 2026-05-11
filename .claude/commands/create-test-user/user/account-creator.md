# Account Creator

Create user accounts in the staging environment.

## Credential Generation

See `user/credential-generator.md` for details on generating unique credentials.

### Generate Email

```bash
# Generate unique email with date-time format: hfuser-DD-HHMMSS@hf.com
EMAIL="hfuser-$(date +%d-%H%M%S)@hf.com"
```

### Set User Details

```bash
PASSWORD="qwerty123"
FIRST_NAME="Test"
LAST_NAME="User"
```

## Account Creation

### 1. Prepare Request

```bash
echo "👤 Step 1/1: Creating user account..."

SIGNUP_DATA=$(cat <<EOF
{
  "email": "${EMAIL}",
  "password": "${PASSWORD}",
  "firstName": "${FIRST_NAME}",
  "lastName": "${LAST_NAME}",
  "country": "${MARKET}",
  "locale": "en-US"
}
EOF
)
```

### 2. Call Signup API

```bash
SIGNUP_RESPONSE=$(curl -s --max-time 10 -X POST \
  "https://www-staging.hellofresh.com/gw/users/signup" \
  -H 'content-type: application/json' \
  -H "authorization: Bearer ${BEARER_TOKEN}" \
  -d "$SIGNUP_DATA")
```

### 3. Extract Customer ID

**Using jq:**
```bash
if [[ "$JSON_PARSER" == "jq" ]]; then
  CUSTOMER_ID=$(echo "$SIGNUP_RESPONSE" | jq -r '.id // empty')
fi
```

**Using python3:**
```bash
if [[ "$JSON_PARSER" == "python3" ]]; then
  CUSTOMER_ID=$(echo "$SIGNUP_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('id', ''))")
fi
```

### 4. Validate Creation

```bash
if [[ -z "$CUSTOMER_ID" ]] || [[ "$CUSTOMER_ID" == "null" ]]; then
  echo "❌ Failed to create user account"
  echo "Response: ${SIGNUP_RESPONSE:0:500}"
  exit 1
fi

echo "✅ User account created (ID: ${CUSTOMER_ID})"
```

### 5. Get Access Token

User's access token for subsequent API calls:

```bash
if [[ "$JSON_PARSER" == "jq" ]]; then
  ACCESS_TOKEN=$(echo "$SIGNUP_RESPONSE" | jq -r '.accessToken // empty')
else
  ACCESS_TOKEN=$(echo "$SIGNUP_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('accessToken', ''))")
fi
```

## Output Variables

After successful account creation:
- `EMAIL` - Generated email address
- `PASSWORD` - User password
- `FIRST_NAME` - User first name
- `LAST_NAME` - User last name
- `CUSTOMER_ID` - User's customer ID
- `ACCESS_TOKEN` - User's access token

## Error Handling

### Duplicate Email
- **Cause:** Email already exists (unlikely with timestamp)
- **Action:** Retry with new timestamp
- **Message:** "Email already exists"

### API Error
- **Cause:** Invalid data or API issue
- **Action:** Display response and exit
- **Message:** "Failed to create user account"

### Invalid Response
- **Cause:** Malformed API response
- **Action:** Display raw response
- **Message:** "Unexpected response format"
