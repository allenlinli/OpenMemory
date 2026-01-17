# Ansible Vault Setup for OpenMemory

This project uses Ansible Vault to encrypt sensitive environment files.

## Quick Start

### First Time Setup

1. **Get the vault password** from your team or create a new one:
   ```bash
   # Create new vault password file (if setting up fresh)
   openssl rand -base64 32 > .vault_pass
   chmod 600 .vault_pass
   ```

2. **Decrypt .env.vault to .env** for local development:
   ```bash
   ansible-vault decrypt .env.vault --output=.env
   ```

   The `.env` file is gitignored and safe to use locally.

## Daily Usage

### View Encrypted File
```bash
ansible-vault view .env.vault
```

### Edit Encrypted File
```bash
ansible-vault edit .env.vault
```

### Decrypt for Local Development
```bash
# Decrypt to .env (gitignored, for local use)
ansible-vault decrypt .env.vault --output=.env

# Or decrypt in place (WARNING: removes encryption!)
ansible-vault decrypt .env.vault
```

### Re-encrypt After Changes
```bash
# If you edited .env locally, re-encrypt it
ansible-vault encrypt .env --output=.env.vault

# Or encrypt in place
ansible-vault encrypt .env.vault
```

## Configuration

The `ansible.cfg` file automatically loads the vault password from `.vault_pass`, so you don't need to type it every time.

## Security Notes

⚠️ **NEVER commit these files:**
- `.vault_pass` - Contains the encryption password
- `.env` - Local plaintext copy of environment variables
- `vault-password.sh` - Bitwarden retrieval script (if using)

✅ **Safe to commit:**
- `.env.vault` - Encrypted environment file
- `.env.example` - Template without secrets
- `ansible.cfg` - Vault configuration

## Vault Password Management

### Option 1: Simple Password File (Current)
The `.vault_pass` file contains your encryption password. Keep it secure and never commit it.

### Option 2: Bitwarden Integration (Recommended for Teams)
Store the vault password in Bitwarden Secrets Manager:

```bash
# 1. Store vault password in Bitwarden
bws secret create "OPENMEMORY_VAULT_PASSWORD" "$(cat .vault_pass)" \
  --project <project-id>

# 2. Create retrieval script
cat > vault-password.sh <<'EOF'
#!/bin/bash
bws secret get OPENMEMORY_VAULT_PASSWORD -o json | jq -r '.value'
EOF
chmod +x vault-password.sh

# 3. Update ansible.cfg
sed -i '' 's|vault_password_file = .vault_pass|vault_password_file = ./vault-password.sh|' ansible.cfg

# 4. Delete the plaintext password file
rm .vault_pass
```

## Deployment with Ansible

The encrypted `.env.vault` file can be deployed using Ansible playbooks:

```yaml
# deploy.yml
---
- name: Deploy OpenMemory
  hosts: production
  tasks:
    - name: Deploy encrypted environment file
      ansible.builtin.copy:
        src: .env.vault
        dest: /opt/openmemory/.env
        mode: '0600'
        owner: openmemory
      # Ansible automatically decrypts during copy!
```

Run with:
```bash
ansible-playbook deploy.yml
```

## Troubleshooting

### "ERROR! Encrypted vault file format or password is incorrect"
- Check that `.vault_pass` contains the correct password
- Verify the file hasn't been corrupted

### "The following paths are ignored by one of your .gitignore files"
- This is expected for `.env` (local file)
- `.env.vault` should NOT be ignored and can be committed

### Need to Change Vault Password
```bash
ansible-vault rekey .env.vault
```

## File Structure

```
OpenMemory/
├── .env.vault          # Encrypted (committed to git)
├── .env.example        # Template (committed to git)
├── .env                # Local plaintext (gitignored)
├── .vault_pass         # Encryption password (gitignored)
├── ansible.cfg         # Vault config (committed to git)
└── VAULT_SETUP.md      # This file
```
