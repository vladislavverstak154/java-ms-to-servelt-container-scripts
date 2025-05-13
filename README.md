#!/usr/bin/env bash
#
# vm_connect.sh — Git Bash wrapper using creds in ~/.ssh/creds_enc + plink.exe
#
# Requirements:
#   • Git for Windows (ssh, openssl available)
#   • plink.exe (PuTTY) somewhere in your PATH

set -euo pipefail

CREDS_FILE="${HOME}/.ssh/creds_enc"

# — 1) Create creds_enc if missing —
if [ ! -f "$CREDS_FILE" ]; then
  read -p "No credentials file at $CREDS_FILE. Create it now? (y/n) " yn
  case "$yn" in
    [Yy]*)
      read -p "SSH username: " NEW_USER
      read -s -p "SSH password: " NEW_PASS; echo
      read -s -p "Passphrase to encrypt creds: " ENC_PASS; echo

      mkdir -p "${HOME}/.ssh"
      chmod 700 "${HOME}/.ssh"
      openssl enc -aes-256-cbc -salt \
        -pass pass:"$ENC_PASS" \
        -out "$CREDS_FILE" <<EOF
$NEW_USER:$NEW_PASS
EOF
      chmod 600 "$CREDS_FILE"
      echo "→ Created encrypted credentials at $CREDS_FILE"
      ;;
    *)
      echo "Aborted; credentials file is required." >&2
      exit 1
      ;;
  esac
fi

# — 2) Decrypt once —
read -s -p "Passphrase to unlock $CREDS_FILE: " FILE_PASS; echo
CREDS=$(openssl enc -aes-256-cbc -d \
         -in "$CREDS_FILE" \
         -pass pass:"$FILE_PASS") \
       || { echo "ERROR: bad passphrase"; exit 2; }

USER=${CREDS%%:*}
PASSWORD=${CREDS#*:}

# — 3) Ensure plink.exe is in PATH —
if ! command -v plink.exe &>/dev/null; then
  echo "ERROR: plink.exe not found in PATH. Download PuTTY's plink.exe and add it to PATH." >&2
  exit 3
fi

# — 4) Loop for multiple connections —
while true; do
  read -p $'\nEnter host (or type "quit" to exit): ' HOST
  [[ "$HOST" == "quit" ]] && { echo "Goodbye."; break; }

  read -p 'Initial command to run (leave blank to skip): ' INIT_CMD

  echo "Connecting to $HOST..."

  if [ -n "$INIT_CMD" ]; then
    # Run init command, then drop you into an interactive bash shell
    plink.exe -ssh -pw "$PASSWORD" -t \
      -o "StrictHostKeyChecking=ask" \
      "$USER@$HOST" \
      "$INIT_CMD; exec bash --login"
  else
    # Pure interactive login
    plink.exe -ssh -pw "$PASSWORD" -t \
      -o "StrictHostKeyChecking=ask" \
      "$USER@$HOST"
  fi

  echo "=== Disconnected from $HOST ==="
done
