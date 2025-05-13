#!/usr/bin/env bash
#
# vm_connect.sh — SSH into a VM using creds in ~/.ssh/creds_enc,
#                run a one-off initial command, then give you an interactive
#                REPL.  Type `exit` at its prompt to drop out of the VM
#                without having to re-enter your decryption passphrase.
#
# Usage: ./vm_connect.sh <host> "<initial-command>"
# Requires: openssl, expect, ssh

set -euo pipefail

CREDS_FILE="${HOME}/.ssh/creds_enc"

# 1) If creds file missing, offer to create it
if [ ! -f "$CREDS_FILE" ]; then
  read -p "No creds file at $CREDS_FILE. Create it now? (y/n) " yn
  case "$yn" in
    [Yy]*)
      read -p "SSH username: " NEW_USER
      read -s -p "SSH password: " NEW_PASS; echo
      read -s -p "Passphrase to encrypt creds: " ENC_PASS; echo

      mkdir -p "${HOME}/.ssh" && chmod 700 "${HOME}/.ssh"
      # one-liner "user:pass" → encrypted file
      openssl enc -aes-256-cbc -salt \
        -pass pass:"$ENC_PASS" \
        -out "$CREDS_FILE" <<EOF
$NEW_USER:$NEW_PASS
EOF
      chmod 600 "$CREDS_FILE"
      echo "→ Created $CREDS_FILE"
      ;;
    *)
      echo "Aborted; creds file is required." >&2
      exit 1
      ;;
  esac
fi

# 2) Parse args
if [ $# -lt 1 ]; then
  echo "Usage: $0 <host> [\"<initial-command>\"]"
  exit 1
fi
HOST=$1
INIT_CMD=${2-}

# 3) Decrypt creds once
read -s -p "Passphrase to unlock $CREDS_FILE: " FILE_PASS; echo
CREDS=$(openssl enc -aes-256-cbc -d -in "$CREDS_FILE" -pass pass:"$FILE_PASS") \
  || { echo "ERROR: bad passphrase"; exit 2; }

USER=${CREDS%%:*}
PASSWORD=${CREDS#*:}

export HOST USER PASSWORD INIT_CMD

# 4) Hand off to Expect
/usr/bin/env expect <<'EOF'
  log_user 1
  set timeout -1

  # grab from env
  set host     $env(HOST)
  set user     $env(USER)
  set password $env(PASSWORD)
  set init_cmd $env(INIT_CMD)

  # SSH in (password only; ask to add new hosts)
  spawn ssh \
    -o PubkeyAuthentication=no \
    -o StrictHostKeyChecking=ask \
    $user@$host

  expect {
    "(yes/no)?" {
      send "yes\r"; exp_continue
    }
    "*?assword:*" {
      send "$password\r"
    }
  }

  # wait for shell prompt
  expect -re {[$#] $}

  # if they passed an initial command, run it
  if {$init_cmd != ""} {
    send -- "$init_cmd\r"
    expect -re {[$#] $}
  }

  # now a little REPL
  while {1} {
    send_user "remote> "
    expect_user -re "(.*)\n"
    set cmd \$expect_out(1,string)
    if {\$cmd eq "exit"} {
      send "exit\r"
      break
    }
    send -- "\$cmd\r"
    expect -re {[$#] $}
  }

  # cleanup: wait for SSH to close
  expect eof
EOF

