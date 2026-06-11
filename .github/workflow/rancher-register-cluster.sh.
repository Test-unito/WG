#!/usr/bin/env bash
#
# rancher-register-cluster.sh - Mint a Rancher cluster import URL via the API.
#
# Lets the infra pipeline register/import a downstream cluster WITHOUT any human
# logging into Rancher as admin. A scoped Rancher API token (created once by
# DevOps) is used to:
#   1. Find or create an "imported" cluster by name in Rancher.
#   2. Find or create its cluster registration token.
#   3. Print the import command in the exact quoted form Terraform expects:
#        "kubectl apply -f https://<rancher-host>/v3/import/<id>.yaml"
#
# The printed value can be fed straight into TF_VAR_rancher_import_url.
#
# Requires: curl, jq.

set -euo pipefail

RANCHER_URL="${RANCHER_URL:-}"        # e.g. https://rancher.dev.mosip.net
RANCHER_TOKEN="${RANCHER_TOKEN:-}"    # token-xxxxx:yyyyy (Bearer)
CLUSTER_NAME="${CLUSTER_NAME:-}"
INSECURE="${INSECURE:-false}"         # use insecureCommand (self-signed Rancher TLS)

usage() {
  cat <<'EOF'
Usage: rancher-register-cluster.sh --rancher-url <url> --token <token> --cluster-name <name> [--insecure]

Required (flags or env vars RANCHER_URL / RANCHER_TOKEN / CLUSTER_NAME):
  --rancher-url <url>     Rancher base URL (https://rancher.<env>.mosip.net)
  --token <token>        Rancher API bearer token (scoped, created once by DevOps)
  --cluster-name <name>  Cluster name to register/import in Rancher

Optional:
  --insecure             Emit the insecure import command (self-signed Rancher TLS)
  -h, --help             Show help

Output (stdout, last line): "kubectl apply -f https://<host>/v3/import/<id>.yaml"
EOF
}

err() { echo "[rancher-register][ERROR] $*" >&2; }
die() { err "$*"; exit 1; }
log() { echo "[rancher-register] $*" >&2; }

while [[ $# -gt 0 ]]; do
  case "$1" in
    --rancher-url)   RANCHER_URL="$2"; shift 2 ;;
    --token)         RANCHER_TOKEN="$2"; shift 2 ;;
    --cluster-name)  CLUSTER_NAME="$2"; shift 2 ;;
    --insecure)      INSECURE="true"; shift ;;
    -h|--help)       usage; exit 0 ;;
    *)               die "Unknown argument: $1 (use --help)" ;;
  esac
done

[[ -n "$RANCHER_URL" ]]   || die "--rancher-url is required"
[[ -n "$RANCHER_TOKEN" ]] || die "--token is required"
[[ -n "$CLUSTER_NAME" ]]  || die "--cluster-name is required"
command -v curl >/dev/null 2>&1 || die "curl is required"
command -v jq   >/dev/null 2>&1 || die "jq is required"

RANCHER_URL="${RANCHER_URL%/}"

api() {
  # api <method> <path> [json-body]
  local method="$1" path="$2" body="${3:-}"
  local args=(-fsSL -X "$method"
    -H "Authorization: Bearer ${RANCHER_TOKEN}"
    -H "Content-Type: application/json"
    -H "Accept: application/json")
  [[ "$INSECURE" == "true" ]] && args+=(-k)
  [[ -n "$body" ]] && args+=(-d "$body")
  curl "${args[@]}" "${RANCHER_URL}${path}"
}

log "Looking up cluster '${CLUSTER_NAME}' in Rancher ..."
CLUSTER_ID="$(api GET "/v3/clusters?name=${CLUSTER_NAME}" | jq -r '.data[0].id // empty')"

if [[ -z "$CLUSTER_ID" ]]; then
  log "Cluster not found; creating imported cluster '${CLUSTER_NAME}' ..."
  CLUSTER_ID="$(api POST "/v3/clusters" \
    "{\"type\":\"cluster\",\"name\":\"${CLUSTER_NAME}\",\"import\":true}" \
    | jq -r '.id // empty')"
  [[ -n "$CLUSTER_ID" ]] || die "Failed to create cluster in Rancher"
  log "Created cluster id=${CLUSTER_ID}"
else
  log "Found existing cluster id=${CLUSTER_ID}"
fi

log "Resolving cluster registration token ..."
TOKEN_JSON="$(api GET "/v3/clusterregistrationtokens?clusterId=${CLUSTER_ID}")"
MANIFEST_URL="$(echo "$TOKEN_JSON" | jq -r '.data[0].manifestUrl // empty')"
INSECURE_CMD="$(echo "$TOKEN_JSON" | jq -r '.data[0].insecureCommand // empty')"
COMMAND="$(echo "$TOKEN_JSON" | jq -r '.data[0].command // empty')"

if [[ -z "$MANIFEST_URL" && -z "$COMMAND" ]]; then
  log "No registration token yet; creating one ..."
  CREATE_JSON="$(api POST "/v3/clusterregistrationtoken" \
    "{\"type\":\"clusterRegistrationToken\",\"clusterId\":\"${CLUSTER_ID}\"}")"
  MANIFEST_URL="$(echo "$CREATE_JSON" | jq -r '.manifestUrl // empty')"
  INSECURE_CMD="$(echo "$CREATE_JSON" | jq -r '.insecureCommand // empty')"
  COMMAND="$(echo "$CREATE_JSON" | jq -r '.command // empty')"
fi

# Prefer the explicit command fields; fall back to building from manifestUrl.
IMPORT_CMD=""
if [[ "$INSECURE" == "true" && -n "$INSECURE_CMD" ]]; then
  IMPORT_CMD="$INSECURE_CMD"
elif [[ -n "$COMMAND" ]]; then
  IMPORT_CMD="$COMMAND"
elif [[ -n "$MANIFEST_URL" ]]; then
  IMPORT_CMD="kubectl apply -f ${MANIFEST_URL}"
fi

[[ -n "$IMPORT_CMD" ]] || die "Could not determine import command from Rancher API response"

# Normalise to: kubectl apply -f https://<host>/v3/import/<id>.yaml
IMPORT_CMD="$(echo "$IMPORT_CMD" | sed -E 's/.*(kubectl apply -f https:\/\/[^ ]+\.yaml).*/\1/')"

log "Import command resolved."
# Final stdout line, quoted to match the Terraform variable validation.
printf '"%s"\n' "$IMPORT_CMD"
