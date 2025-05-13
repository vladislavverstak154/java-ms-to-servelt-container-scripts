#!/usr/bin/env bash
#
# vm_connect.sh — Looping SSH wrapper for Git Bash using creds in ~/.ssh/creds_enc.
#                Connect to any number of VMs in one run.
#
# Usage: ./vm_connect.sh
# Requires: sshpass, openssl, ssh (all available in your PATH)

set -euo pipefail

CREDS_FILE="${HOME}/.ssh/creds_enc"

# — Step 1: create creds_enc if missing —
if [ ! -f "$CREDS_FILE" ]; then
  read -p "No credentials file at $CREDS_FILE. Create it now? (y/n) " yn
  case "$yn" in
    [Yy]*)
      read -p "SSH username: " NEW_USER
      read -s -p "SSH password: " NEW_PASS; echo
      read -s -p "Passphrase to encrypt creds: " ENC_PASS; echo

      mkdir -p "${HOME}/.ssh"
      chmod 700 "${HOME}/.ssh"
      # encrypt one-line "user:pass"
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

# — Step 2: decrypt creds once —
read -s -p "Passphrase to unlock $CREDS_FILE: " FILE_PASS; echo
CREDS=$(openssl enc -aes-256-cbc -d \
         -in "$CREDS_FILE" \
         -pass pass:"$FILE_PASS") \
       || { echo "ERROR: bad passphrase"; exit 2; }

USER=${CREDS%%:*}
PASSWORD=${CREDS#*:}

# — Step 3: ensure sshpass is available —
if ! command -v sshpass >/dev/null; then
  echo "ERROR: sshpass not found in PATH. Please install sshpass for Git Bash." >&2
  exit 3
fi

# — Step 4: loop for multiple hosts —
while true; do
  read -p $'\nEnter host (or type "quit" to exit): ' HOST
  [[ "$HOST" == "quit" ]] && { echo "Goodbye."; break; }

  read -p 'Initial command to run (leave blank to skip): ' INIT_CMD

  echo "Connecting to $HOST..."

  if [ -n "$INIT_CMD" ]; then
    # run initial, then exec an interactive shell
    sshpass -p "$PASSWORD" ssh -t \
      -o PubkeyAuthentication=no \
      -o StrictHostKeyChecking=ask \
      "$USER@$HOST" \
      "$INIT_CMD; exec \$SHELL -l"
  else
    # pure interactive login
    sshpass -p "$PASSWORD" ssh -t \
      -o PubkeyAuthentication=no \
      -o StrictHostKeyChecking=ask \
      "$USER@$HOST"
  fi

  echo "=== Disconnected from $HOST ==="
done
