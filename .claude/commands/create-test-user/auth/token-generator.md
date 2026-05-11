# Token Generator

Generate bearer tokens for staging API authentication.

## Credentials

Uses hardcoded staging credentials for auto-authentication:

```bash
STAGING_USERNAME="er+391+1@hf.com"
STAGING_PASSWORD="qwerty123"
```

## Authentication Flow

### 1. Set Market

```bash
MARKET="US"
```

### 2. Request Bearer Token

```bash
echo "🔐 Authenticating with staging credentials..."

LOGIN_AUTH_RESPONSE=$(curl -s --max-time 10 -X POST \
  "https://www-staging.hellofresh.com/gw/login?country=us" \
  -H 'content-type: application/json' \
  -d "{\"username\":\"${STAGING_USERNAME}\",\"password\":\"${STAGING_PASSWORD}\"}")
```

### 3. Parse Token

**Using jq:**
```bash
if [[ "$JSON_PARSER" == "jq" ]]; then
  BEARER_TOKEN=$(echo "$LOGIN_AUTH_RESPONSE" | jq -r '.access_token // empty')
fi
```

**Using python3:**
```bash
if [[ "$JSON_PARSER" == "python3" ]]; then
  BEARER_TOKEN=$(echo "$LOGIN_AUTH_RESPONSE" | python3 -c "import sys, json; d=json.load(sys.stdin); print(d.get('access_token', ''))")
fi
```

### 4. Validate Token

```bash
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

## Output Variables

After successful authentication:
- `BEARER_TOKEN` - Valid bearer token for API calls
- `MARKET` - Set to "US"

## Error Handling

### Authentication Failure
- **Cause:** Invalid credentials or VPN not connected
- **Action:** Display error with troubleshooting steps
- **Exit:** Code 1

### VPN Not Connected
- **Symptom:** Timeout or connection refused
- **Message:** "VPN not connected or staging gateway unreachable"
- **Resolution:** Connect to staging VPN

## Security Notes

- Credentials are for staging environment only
- Bearer tokens expire after session
- No production credentials should be used
