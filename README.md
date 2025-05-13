#!/usr/bin/env bash
#
# vm_connect.sh — Looping SSH wrapper using creds in ~/.ssh/creds_enc.
#                Connect to any number of VMs in one run.
#
# Usage: ./vm_connect.sh
# Requires: openssl, expect, ssh

set -euo pipefail
CREDS_FILE="${HOME}/.ssh/creds_enc"

# ——— Step 1: create creds_enc if missing ———
if [ ! -f "$CREDS_FILE" ]; then
  read -p "No creds file at $CREDS_FILE. Create it now? (y/n) " yn
  case "$yn" in
    [Yy]*)
      read -p "SSH username: " NEW_USER
      read -s -p "SSH password: " NEW_PASS; echo
      read -s -p "Passphrase to encrypt creds file: " ENC_PASS; echo

      mkdir -p "${HOME}/.ssh" && chmod 700 "${HOME}/.ssh"
      openssl enc -aes-256-cbc -salt \
        -pass pass:"$ENC_PASS" \
        -out "$CREDS_FILE" <<EOF
$NEW_USER:$NEW_PASS
EOF
      chmod 600 "$CREDS_FILE"
      echo "→ Created $CREDS_FILE"
      ;;
    *)
      echo "Aborted; credentials file is required." >&2
      exit 1
      ;;
  esac
fi

# ——— Step 2: decrypt creds once ———
read -s -p "Passphrase to unlock $CREDS_FILE: " FILE_PASS; echo
CREDS=$(openssl enc -aes-256-cbc -d -in "$CREDS_FILE" -pass pass:"$FILE_PASS") \
  || { echo "ERROR: bad passphrase"; exit 2; }

USER=${CREDS%%:*}
PASSWORD=${CREDS#*:}

export USER PASSWORD

# ——— Step 3: loop for multiple hosts ———
while true; do
  read -p $'\nEnter host (or type "quit" to exit): ' HOST
  [[ "$HOST" == "quit" ]] && { echo "Goodbye."; break; }
  read -p 'Initial command to run (leave blank for none): ' INIT_CMD

  export HOST INIT_CMD

  /usr/bin/env expect <<'EOF'
    log_user 1
    set timeout -1

    # grab from env
    set host     $env(HOST)
    set user     $env(USER)
    set password $env(PASSWORD)
    set init_cmd $env(INIT_CMD)

    # SSH in (password-only; ask to trust new hosts)
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

    # wait for prompt
    expect -re {[$#] $}

    # optional initial command
    if {\$init_cmd != ""} {
      send -- "\$init_cmd\r"
      expect -re {[$#] $}
    }

    # REPL: type commands; 'exit' drops back to this script
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

    expect eof
EOF

done
