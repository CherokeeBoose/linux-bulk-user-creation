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

ls -l userlist.csv
echo "------"; cat userlist.csv; echo "------"
```

Expected: the file lists and prints exactly as above.

---

## 2) Create the provisioning script

```bash
cat > provision_users.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

CSV_FILE="${1:-}"
[[ -z "$CSV_FILE" ]] && { echo "Usage: sudo $0 userlist.csv"; exit 1; }
[[ $EUID -ne 0 ]] && { echo "Please run with sudo/root."; exit 1; }
[[ ! -f "$CSV_FILE" ]] && { echo "CSV not found: $CSV_FILE"; exit 1; }

DEFAULT_SHELL="/bin/bash"

tail -n +2 "$CSV_FILE" | while IFS=',' read -r FirstName LastName UserID JobRole StartingPassword; do
  FirstName="$(echo "$FirstName" | xargs)"
  LastName="$(echo "$LastName" | xargs)"
  UserID="$(echo "$UserID" | xargs)"
  JobRole="$(echo "$JobRole" | tr '[:upper:] ' '[:lower:]_' | xargs)"
  StartingPassword="$(echo "$StartingPassword" | xargs)"

  [[ -z "$UserID" ]] && { echo "Skipping row with empty UserID."; continue; }

  GROUP="${JobRole:-$UserID}"
  if ! getent group "$GROUP" >/dev/null; then
    groupadd "$GROUP"
    echo "Created group: $GROUP"
  fi

  if id "$UserID" &>/dev/null; then
    echo "User exists, updating: $UserID"
    usermod -g "$GROUP" -s "$DEFAULT_SHELL" -c "$FirstName $LastName" "$UserID"
  else
    echo "Creating user: $UserID ($FirstName $LastName)"
    useradd -m -c "$FirstName $LastName" -g "$GROUP" -s "$DEFAULT_SHELL" "$UserID"
  fi

  echo "${UserID}:${StartingPassword}" | chpasswd
  chage -d 0 "$UserID"
done

echo "Bulk user provisioning complete."
SCRIPT

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
getent passwd jdoe
id jdoe
sudo chage -l jdoe
getent group sales_manager
```

Expected:
- `jdoe` exists with a home under `/home/jdoe`
- primary group reflects JobRole
- `Password must be changed at next login`

---

## 5) Optional cleanup/reset

```bash
cat > remove_users.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
CSV_FILE="${1:-}"
[[ -z "$CSV_FILE" ]] && { echo "Usage: sudo $0 userlist.csv"; exit 1; }
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
