It does three things:

generates the JWT key pair for Salesforce
gives you the public certificate to upload to the Salesforce Connected App
pushes your secrets into GitHub environment secrets for dev, uat, and prod
Before you run it

You need:

gh installed and authenticated
openssl installed
a GitHub repo already created
a Salesforce Connected App created first, because you’ll need its Consumer Key / Client ID
Script: setup-github-salesforce-secrets.sh
#!/usr/bin/env bash
set -euo pipefail

# ==========================================
# CONFIGURE THESE VALUES
# ==========================================
GITHUB_REPO="${GITHUB_REPO:-your-org/your-repo}"

DEV_USERNAME="${DEV_USERNAME:-dev-user@example.com}"
UAT_USERNAME="${UAT_USERNAME:-uat-user@example.com}"
PROD_USERNAME="${PROD_USERNAME:-prod-user@example.com}"

DEV_INSTANCE_URL="${DEV_INSTANCE_URL:-https://test.salesforce.com}"
UAT_INSTANCE_URL="${UAT_INSTANCE_URL:-https://test.salesforce.com}"
PROD_INSTANCE_URL="${PROD_INSTANCE_URL:-https://login.salesforce.com}"

SF_CLIENT_ID="${SF_CLIENT_ID:-}"

KEY_DIR="${KEY_DIR:-./jwt}"
KEY_NAME="${KEY_NAME:-server.key}"
CERT_NAME="${CERT_NAME:-server.crt}"

# ==========================================
# CHECKS
# ==========================================
command -v gh >/dev/null 2>&1 || {
  echo "Error: GitHub CLI (gh) is not installed."
  exit 1
}

command -v openssl >/dev/null 2>&1 || {
  echo "Error: openssl is not installed."
  exit 1
}

if [[ -z "${SF_CLIENT_ID}" ]]; then
  echo "Error: SF_CLIENT_ID is empty."
  echo "Set it like this:"
  echo '  export SF_CLIENT_ID="3MVG9..."'
  exit 1
fi

# Check gh auth
if ! gh auth status >/dev/null 2>&1; then
  echo "Error: gh is not authenticated."
  echo "Run: gh auth login"
  exit 1
fi

mkdir -p "${KEY_DIR}"

PRIVATE_KEY_PATH="${KEY_DIR}/${KEY_NAME}"
CERT_PATH="${KEY_DIR}/${CERT_NAME}"

# ==========================================
# GENERATE JWT KEYPAIR IF MISSING
# ==========================================
if [[ ! -f "${PRIVATE_KEY_PATH}" || ! -f "${CERT_PATH}" ]]; then
  echo "Generating RSA private key and self-signed certificate..."
  openssl req -x509 -newkey rsa:2048 -keyout "${PRIVATE_KEY_PATH}" -out "${CERT_PATH}" -sha256 -days 3650 -nodes -subj "/CN=Salesforce JWT CI"
else
  echo "Using existing keypair in ${KEY_DIR}"
fi

echo
echo "=========================================="
echo "PUBLIC CERTIFICATE CREATED"
echo "Upload this certificate to your Salesforce Connected App:"
echo "  ${CERT_PATH}"
echo "=========================================="
echo

# ==========================================
# READ PRIVATE KEY CONTENT
# ==========================================
SF_JWT_KEY_CONTENT="$(cat "${PRIVATE_KEY_PATH}")"

# ==========================================
# HELPER FUNCTION
# ==========================================
set_env_secret() {
  local env_name="$1"
  local secret_name="$2"
  local secret_value="$3"

  echo "Setting ${secret_name} in environment ${env_name}..."
  gh secret set "${secret_name}" \
    --env "${env_name}" \
    --repo "${GITHUB_REPO}" \
    --body "${secret_value}"
}

# ==========================================
# CREATE ENVIRONMENTS IF NEEDED
# NOTE:
# gh CLI does not provide a simple built-in command
# to create Actions environments directly in all setups,
# so create dev/uat/prod in GitHub UI first if missing.
# ==========================================
echo "Make sure these GitHub environments already exist:"
echo "  - dev"
echo "  - uat"
echo "  - prod"
echo

# ==========================================
# DEV SECRETS
# ==========================================
set_env_secret "dev" "SF_USERNAME" "${DEV_USERNAME}"
set_env_secret "dev" "SF_CLIENT_ID" "${SF_CLIENT_ID}"
set_env_secret "dev" "SF_JWT_KEY" "${SF_JWT_KEY_CONTENT}"
set_env_secret "dev" "SF_INSTANCE_URL" "${DEV_INSTANCE_URL}"

# ==========================================
# UAT SECRETS
# ==========================================
set_env_secret "uat" "SF_USERNAME" "${UAT_USERNAME}"
set_env_secret "uat" "SF_CLIENT_ID" "${SF_CLIENT_ID}"
set_env_secret "uat" "SF_JWT_KEY" "${SF_JWT_KEY_CONTENT}"
set_env_secret "uat" "SF_INSTANCE_URL" "${UAT_INSTANCE_URL}"

# ==========================================
# PROD SECRETS
# ==========================================
set_env_secret "prod" "SF_USERNAME" "${PROD_USERNAME}"
set_env_secret "prod" "SF_CLIENT_ID" "${SF_CLIENT_ID}"
set_env_secret "prod" "SF_JWT_KEY" "${SF_JWT_KEY_CONTENT}"
set_env_secret "prod" "SF_INSTANCE_URL" "${PROD_INSTANCE_URL}"

echo
echo "Done."
echo
echo "Next steps:"
echo "1. Upload ${CERT_PATH} to the Salesforce Connected App."
echo "2. Confirm the GitHub environments dev/uat/prod exist."
echo "3. Run a workflow to test JWT auth."
How to run it
chmod +x setup-github-salesforce-secrets.sh

export GITHUB_REPO="your-org/your-repo"
export SF_CLIENT_ID="3MVG9xxxxxxxxxxxxxxxx"
export DEV_USERNAME="dev-user@example.com"
export UAT_USERNAME="uat-user@example.com"
export PROD_USERNAME="prod-user@example.com"

./setup-github-salesforce-secrets.sh
What this script does not do

It does not create the Salesforce Connected App for you. That still needs to be done in Salesforce first.

What you do in Salesforce

After the script runs, take the generated file:

./jwt/server.crt

Upload that certificate into your Salesforce Connected App.

Then use the Connected App’s Consumer Key as:

SF_CLIENT_ID
Good setup for your team of 5 on one shared dev sandbox

Use:

one integration user for dev
one integration user for uat
one integration user for prod

That keeps deployments stable and avoids using personal user accounts.
