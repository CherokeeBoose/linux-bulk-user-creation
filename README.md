# Lab: Bulk Linux User Creation from CSV (EC2 + Bash)

## Objective
Automate Linux user onboarding from a CSV on an EC2 instance: create CSV → run a provisioning script → verify → optional cleanup.

## Prerequisites
- Linux EC2 (Amazon Linux 2/Ubuntu)
- Shell access (SSH or AWS Systems Manager: Session Manager)
- `sudo` privileges

---

## 1) Create the CSV (no editor needed)

```bash
cat > userlist.csv <<'CSV'
FirstName,LastName,UserID,JobRole,StartingPassword
Alejandro,Rosalez,arosalez,sales_manager,P@ssword1234!
Efua,Owusu,eowusu,shipping_manager,P@ssword1234!
Jane,Doe,jdoe,shipping,P@ssword1234!
Mateo,Jackson,mjackson,CEO,P@ssword1234!
Nikki,Wolf,nwolf,sales_representative,P@ssword1234!
Mary,Major,mmajor,finance_manager,P@ssword1234!
CSV

# Quick sanity check
ls -l userlist.csv
echo "------"; cat userlist.csv; echo "------"
```

---

## 2) Create the provisioning script (with inline teaching comments)

```bash
cat > provision_users.sh <<'SCRIPT'
#!/usr/bin/env bash
# ^ Use the first bash found in PATH for portability across distros.

set -euo pipefail
# -e : exit immediately on any error (fail fast)
# -u : treat unset variables as errors (catches typos)
# pipefail : if any command in a pipeline fails, the whole pipeline fails

CSV_FILE="${1:-}"
# ^ Read the first argument as the CSV file path; default to empty if omitted.

# Guard clauses: validate usage, privileges, and file existence.
[[ -z "$CSV_FILE" ]] && { echo "Usage: sudo $0 userlist.csv"; exit 1; }
[[ $EUID -ne 0 ]] && { echo "Please run with sudo/root."; exit 1; }
[[ ! -f "$CSV_FILE" ]] && { echo "CSV not found: $CSV_FILE"; exit 1; }

DEFAULT_SHELL="/bin/bash"
# ^ Set the login shell for created/updated users.

# Read the CSV line-by-line, skipping the header row.
# IFS=','   : split fields on commas
# -r        : do not treat backslashes as escapes
tail -n +2 "$CSV_FILE" | while IFS=',' read -r FirstName LastName UserID JobRole StartingPassword; do

  # Normalize/trust-but-trim input from CSV (common if edited in Excel).
  FirstName="$(echo "$FirstName" | xargs)"                     # trim whitespace
  LastName="$(echo "$LastName" | xargs)"                       # trim whitespace
  UserID="$(echo "$UserID" | xargs)"                           # trim whitespace
  JobRole="$(echo "$JobRole" | tr '[:upper:] ' '[:lower:]_' | xargs)"
  # ^ Lowercase and replace spaces with underscores for safe group names.
  StartingPassword="$(echo "$StartingPassword" | xargs)"       # trim whitespace

  # Require a username; skip incomplete rows instead of failing the whole run.
  [[ -z "$UserID" ]] && { echo "Skipping row with empty UserID."; continue; }

  # Determine primary group: prefer JobRole; fallback to username if empty.
  GROUP="${JobRole:-$UserID}"

  # Ensure the group exists (idempotent: only create if missing).
  if ! getent group "$GROUP" >/dev/null; then
    groupadd "$GROUP"
    echo "Created group: $GROUP"
  fi

  # Create new user OR update existing user's metadata to match CSV.
  if id "$UserID" &>/dev/null; then
    echo "User exists, updating: $UserID"
    # -g : set primary group
    # -s : set login shell
    # -c : set GECOS (full name) for readability
    usermod -g "$GROUP" -s "$DEFAULT_SHELL" -c "$FirstName $LastName" "$UserID"
  else
    echo "Creating user: $UserID ($FirstName $LastName)"
    # -m : create home directory under /home/<user>
    # -c : full name
    # -g : primary group
    # -s : login shell
    useradd -m -c "$FirstName $LastName" -g "$GROUP" -s "$DEFAULT_SHELL" "$UserID"
  fi

  # Set the initial password non-interactively.
  # chpasswd reads "user:password" pairs from STDIN.
  echo "${UserID}:${StartingPassword}" | chpasswd

  # Force password change at next login (security best practice).
  chage -d 0 "$UserID"

done

echo "Bulk user provisioning complete."
SCRIPT

# Make the script executable
chmod +x provision_users.sh
```

---

## 3) Run the script

```bash
sudo ./provision_users.sh userlist.csv
```

---

## 4) Verify a user and group

```bash
getent passwd jdoe      # confirm account exists and shows home/shell
id jdoe                 # confirm UID/GID and primary group
sudo chage -l jdoe      # confirm "Password must be changed at next login"
getent group sales_manager
```

---

## 5) Optional cleanup/reset

```bash
cat > remove_users.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
CSV_FILE="${1:-}"
[[ -z "$CSV_FILE" ]] && { echo "Usage: sudo $0 userlist.csv"; exit 1; }

# Skip header and remove each user listed in the CSV; ignore if already absent.
tail -n +2 "$CSV_FILE" | while IFS=',' read -r _ _ UserID _ _; do
  UserID="$(echo "$UserID" | xargs)"
  [[ -z "$UserID" ]] && continue
  if id "$UserID" &>/dev/null; then
    echo "Deleting user and home: $UserID"
    userdel -r "$UserID" || true
  fi
done
echo "Cleanup complete."
SCRIPT

chmod +x remove_users.sh
sudo ./remove_users.sh userlist.csv
```

---

## Security notes
- CSV contains plaintext passwords → acceptable for lab only; force rotation with `chage -d 0`.
- Always run with `sudo`.
- Scripts are idempotent (safe to re-run).
