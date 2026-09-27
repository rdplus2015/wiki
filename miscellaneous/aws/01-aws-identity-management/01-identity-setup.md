# AWS Identity Management — Command Reference

Quick reference for AWS CLI v2 setup, identity/credentials handling, and profile management.
Companion to `AWS_CLI_SETUP.md` (concepts, rationale, credential model — not yet written).

---

## Install & verify

```bash
# Official installer (Linux x86_64) — avoid apt/snap packages, they lag behind
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws/

# Verify installation
aws --version
# → aws-cli/2.x.x Python/3.x.x Linux/...

# Update to latest version (re-run the same installer with --update)
sudo ./aws/install --update
```

---

## Identity checks

```bash
# Confirm which account/user/role is currently active — run this before ANY
# deploy or destructive operation. Never assume the active identity.
aws sts get-caller-identity

# Same, against a specific profile
aws sts get-caller-identity --profile <profile-name>

# Get the caller's account ID only (useful in scripts)
aws sts get-caller-identity --query "Account" --output text
```

---

## Profile management

A **profile** is a named, isolated set of credentials stored locally in
`~/.aws/credentials` (secrets) and `~/.aws/config` (region, output format,
role settings). Working with named profiles instead of relying on `default`
is the professional standard once you touch more than one account or project.

```bash
# Create or update a profile interactively
aws configure --profile <profile-name>
# Prompts: AWS Access Key ID / AWS Secret Access Key / Default region name / Default output format

# List all configured profiles
aws configure list-profiles

# Show the resolved config/credentials for a given profile
aws configure list --profile <profile-name>

# Set a single value without the interactive prompt
aws configure set region us-east-1 --profile <profile-name>
aws configure set output json --profile <profile-name>

# Same thing, interactively — re-running `aws configure` on an existing profile
# shows the current value in brackets; press Enter to keep it, or type a new one
aws configure --profile <profile-name>
# AWS Access Key ID [****************XXXX]:
# AWS Secret Access Key [****************XXXX]:
# Default region name [us-east-1]:
# Default output format [json]:

# Use a profile for a single command (no need to export anything)
aws s3 ls --profile <profile-name>

# Use a profile for an entire shell session
export AWS_PROFILE=<profile-name>
```

**Renaming a profile** — there is no `aws configure rename-profile` command.
Edit `~/.aws/credentials` and `~/.aws/config` directly, or manually:

```bash
cat ~/.aws/credentials   # copy the aws_access_key_id / aws_secret_access_key
nano ~/.aws/credentials
```

```ini
[old-profile-name]
aws_access_key_id = ...
aws_secret_access_key = ...

[new-profile-name]
aws_access_key_id = ...
aws_secret_access_key = ...
```

Then remove the old section once the new one is verified working.

---

## Access key rotation

Rotating keys regularly (and immediately after any suspected exposure) is
standard practice — never reuse a leaked key, always create-then-delete.

```bash
# Create a new access key for a user (a user can have max 2 active keys at once —
# this is intentional, it's what makes zero-downtime rotation possible)
aws iam create-access-key --user-name <iam-user-name>

# List existing keys for a user (to identify which one to retire)
aws iam list-access-keys --user-name <iam-user-name>

# Update the local profile with the new key
aws configure set aws_access_key_id <new-access-key-id> --profile <profile-name>
aws configure set aws_secret_access_key <new-secret-access-key> --profile <profile-name>

# Verify the new key works before deleting the old one
aws sts get-caller-identity --profile <profile-name>

# Delete the old (retired) key
aws iam delete-access-key --access-key-id <old-access-key-id> --user-name <iam-user-name>
```

---

## Global options (useful on every command)

```bash
--profile <profile-name>     # target a specific credential set
--region <region>            # override the configured region for this call only
--output json|table|text|yaml  # control output format
--query "<JMESPath expression>"  # filter/reshape the response client-side
--no-cli-pager                # print directly instead of piping through a pager
--dry-run                     # simulate the call without executing it (supported by most EC2/mutating commands)
```

Examples:

```bash
# Table output — readable at a glance in the terminal
aws ec2 describe-instances --profile <profile-name> --output table

# Extract a single field with a JMESPath query
aws s3api list-buckets --query "Buckets[].Name" --output text

# Combine profile + region + query for a scripted, portable command
aws lambda list-functions \
  --profile <profile-name> \
  --region us-east-1 \
  --query "Functions[].FunctionName" \
  --output text
```

---

## Temporary / federated credentials (STS)

Relevant whenever you're not using long-lived IAM user keys — sandbox
environments, assumed roles, SSO.

```bash
# Assume a role and get temporary credentials (valid 15min–12h depending on role config)
aws sts assume-role \
  --role-arn arn:aws:iam::<account-id>:role/<role-name> \
  --role-session-name <session-name>

# Check how long the current session's credentials remain valid
aws sts get-caller-identity
# (expiration is not shown here directly — check the session token issuance response,
# or re-run get-caller-identity: an expired token fails with ExpiredToken)
```

---

## Housekeeping

```bash
# Clear all cached SSO/STS sessions
rm -rf ~/.aws/cli/cache

# See exactly which credentials/config file paths the CLI is reading
aws configure list
```

---

## Security notes

- Never commit `~/.aws/credentials` or any raw access key to Git.
- Never hardcode an access key ID or secret in a command example that gets shared or published — always use placeholders (`<access-key-id>`, `<account-id>`).
- If a key is ever exposed (committed, pasted, logged), rotate it immediately (create new → verify → delete old) rather than just deleting it first.

---

## References

- [AWS CLI v2 official install guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [AWS CLI configuration basics](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [Named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html)
- [STS AssumeRole reference](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)
- [JMESPath query syntax (used by --query)](https://jmespath.org/tutorial.html)
