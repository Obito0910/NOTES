# Azure Cloud Security — Concept Notes (from CryptoCabana challenge)

**Goal of these notes:** understand _why_ each step worked, so that next time you can solve any "Azure Storage / Key Vault" style CTF yourself — not just by memorizing commands, but by actually understanding the reasoning.

---

## 1. Azure Storage Account — Basics

An Azure Storage Account is a container that holds multiple **services**:

- **Blob storage** — files/objects (similar to an S3 bucket in AWS)
- **File storage** — SMB file shares
- **Queue storage** — messaging
- **Table storage** — NoSQL key-value store

Every storage account has its own unique DNS name:

```
https://<account-name>.blob.core.windows.net/
```

Inside a storage account you have **Containers** (like folders/buckets), and inside each container you have **Blobs** (files).

```
storage account (cryptocabanaf5scjagc)
 ├── container: $web       ← static website files
 ├── container: backups    ← the kiosk writes here
 └── container: vault      ← hidden, never referenced by the kiosk
```

**Key learning:** Just because a container's name is never written anywhere on the page doesn't mean it's _access-protected_. If you have an account-level credential, you can see everything — whether or not the UI points to it. This is called **"security by obscurity"** — a flawed approach that is a common vulnerability in real-world systems too.

---

## 2. SAS Tokens (Shared Access Signature) — The most important concept here

A SAS token is a **temporary, scoped credential** that Azure Storage gives you without sharing your actual account key. It's attached via a query string in the URL:

```
https://account.blob.core.windows.net/container/blob?sv=...&sig=...
```

### There are 3 types of SAS:

|Type|Scope|When it's used|
|---|---|---|
|**User Delegation SAS**|Linked to an Azure AD identity|Most secure, modern approach|
|**Service SAS**|Within one specific service, scoped to one specific resource (e.g. just one container/blob)|When the client only needs access to one thing|
|**Account SAS**|Across the whole storage account, one or more services|Broad access — much more powerful, and therefore risky|

### Learn to decode a SAS token's query parameters:

Our token looked like this:

```
?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=...
```

|Param|Full form|Value|Meaning|
|---|---|---|---|
|`sv`|Storage version|`2022-11-02`|API version|
|`ss`|**S**ignatur**e** **s**ervices|`b`|blob service only|
|`srt`|**S**ignature **r**esource **t**ypes|`sco`|**s**ervice + **c**ontainer + **o**bject — valid at all three levels!|
|`sp`|**S**ignature **p**ermissions|`rl`|**r**ead + **l**ist|
|`se`|Signature **e**xpiry|2099|expiry date (extremely far out — a red flag in itself)|
|`st`|Signature **st**art|2024|valid from this date|
|`spr`|Protocol|`https`|HTTPS only|
|`sig`|Signature|(hash)|the actual cryptographic signature that verifies everything|

**The single most important thing to learn here:** `srt=sco` means this token isn't limited to one container — it can **list/read across the entire account**. Whenever you find a SAS token in frontend JS, the first thing to check is the `srt` and `sp` parameters — if `srt` includes `s` (service) or the full `sco`, that token's reach extends well beyond the one resource it was meant for.

**Practical rule:** Never use an Account SAS client-side (in browser JS) — always use a Service SAS scoped to one specific blob/container, with minimum necessary permissions. This was the core "vulnerability" in this challenge.

---

## 3. Blob Storage REST API — Manual enumeration

Azure Blob Storage has a REST API that can be used directly with curl, without needing the `az` CLI — you just need the right query parameters.

### List containers in an account:

```
GET https://<account>.blob.core.windows.net/?comp=list
```

### List blobs in a container:

```
GET https://<account>.blob.core.windows.net/<container>?restype=container&comp=list
```

### Download/read a blob:

```
GET https://<account>.blob.core.windows.net/<container>/<blob-name>
```

Every request needs the SAS token (`?...&sig=...`) appended at the end if the resource isn't publicly accessible.

**Worth remembering:** `comp=list` and `restype=container` — these two query params are what trigger Azure Blob API's "listing" mode. Without them, fetching the bare root URL gets you nothing.

---

## 4. Azure AD App Registrations / Service Principals

When we found `backup-service-account.json`, it had these fields:

```json
{
  "client_id": "...",       // The App's (Service Principal's) identity
  "client_secret": "...",   // Password / secret
  "tenant_id": "...",       // Azure AD tenant (organization) ID
  "key_vault_uri": "..."
}
```

This is a **Service Principal** — essentially a "robot user account" created for automation/scripts (no human logs in, a script does). Three things are needed to authenticate as one:

1. `client_id` — who it is
2. `client_secret` — its password
3. `tenant_id` — which organization it belongs to

With these three, you can log in to Azure without any human account, using `az login --service-principal`. This is called **"Service Principal authentication"**, or the OAuth2 **client credentials flow**.

**Real-world danger:** If this JSON is ever accidentally placed somewhere publicly readable (like a blob storage container), anyone can reach whatever Azure resources that service has access to. That's exactly why the JSON contained the note: _"Rotate this if it ever leaves the vault."_

---

## 5. Authenticating with Azure CLI (`az`)

```bash
az login --service-principal -u <client_id> -p <client_secret> --tenant <tenant_id>
```

This command gets an **access token** from Azure AD, which then gets automatically attached to every subsequent `az` command. After logging in, you can check which subscription is active:

```bash
az account show
```

And if multiple subscriptions are available:

```bash
az account list -o table
az account set --subscription "<name-or-id>"
```

---

## 6. Azure Key Vault — Secrets and Versioning

**Key Vault** is Azure's secrets-management service — for storing passwords, API keys, certificates, etc., encrypted at rest and access-controlled via Azure AD.

### Important concept: every secret in Key Vault is **versioned**

Whenever you update a secret's value, the old value isn't deleted — a new **version** is created instead. By default, the old version stays readable (unless it's explicitly disabled/purged).

```bash
# List all secrets
az keyvault secret list --vault-name <vault-name> -o table

# List all versions of one secret
az keyvault secret list-versions --vault-name <vault-name> --name <secret-name> -o json

# Get the latest version's value
az keyvault secret show --vault-name <vault-name> --name <secret-name> --query value -o tsv

# Get a specific (older) version's value
az keyvault secret show --vault-name <vault-name> --name <secret-name> --version <version-id> --query value -o tsv
```

**This was the "twist" in this challenge:** `key-shard-2` had 2 versions, created only 2 seconds apart — this pattern indicates some kind of "trigger on access, then rotate" automation. The current/latest version was just a taunting message, but **the real secret was in the older version**.

**The general principle to take away:** Whenever you get access to Key Vault (or any versioned secret store — like Git history, S3 versioning, Kubernetes secrets), don't just check "latest" — always check **history/versions** too. A lot of CTFs and real-world leaks follow exactly this pattern (a secret leaked once, got "rotated," but the old version remained accessible).

---

## 7. Overall Methodology — General flow for cloud recon

This pattern roughly repeats across almost every Azure/AWS/GCP cloud challenge:

1. **Explore the static frontend** — page source, JS files, network tab. Look for hardcoded credentials/tokens/URLs.
2. **Understand the scope of any credential you find** — is it limited to one resource, or broader?
3. **Enumerate using that credential** — list containers/buckets/resources that weren't directly shown to you.
4. **Check for hidden/unlinked resources** — "not linked in the UI" ≠ "not accessible."
5. **If you find another service credential (service principal, IAM role, access key), escalate** — log in with the new credential and see what further access it grants.
6. **Check history/versions** — rotated secrets, deleted-but-recoverable data, git history, backups.
7. **Repeat the same loop at every new access level** — list → enumerate → escalate.

This same mindset applies to AWS (S3 buckets + IAM roles) and GCP (Cloud Storage + Service Accounts) too — only the commands/tools differ.

---

## 8. Quick Command Cheatsheet

```bash
# --- Blob Storage (SAS-based, no az needed) ---
curl -s "https://ACCOUNT.blob.core.windows.net/?comp=list&SAS_TOKEN"
curl -s "https://ACCOUNT.blob.core.windows.net/CONTAINER?restype=container&comp=list&SAS_TOKEN"
curl -s "https://ACCOUNT.blob.core.windows.net/CONTAINER/BLOB?SAS_TOKEN"

# --- Azure CLI auth ---
az login --service-principal -u CLIENT_ID -p CLIENT_SECRET --tenant TENANT_ID
az account show

# --- Key Vault ---
az keyvault secret list --vault-name VAULT_NAME -o table
az keyvault secret list-versions --vault-name VAULT_NAME --name SECRET_NAME -o json
az keyvault secret show --vault-name VAULT_NAME --name SECRET_NAME --query value -o tsv
az keyvault secret show --vault-name VAULT_NAME --name SECRET_NAME --version VERSION_ID --query value -o tsv

# --- Useful for other Azure recon (bonus, not used this time but good to know) ---
az keyvault key list --vault-name VAULT_NAME -o table       # keys (not secrets)
az keyvault certificate list --vault-name VAULT_NAME -o table
az resource list -o table                                    # all resources this SP can access
az role assignment list --assignee CLIENT_ID -o table         # SP's roles/permissions
```

---

## 9. Terms Glossary (for quick revision)

- **SAS Token** — Temporary, scoped credential for accessing Azure Storage without the master key.
- **Account SAS vs Service SAS** — Account SAS = broad (multiple services/resources); Service SAS = narrow (one resource).
- **Container** — Azure Blob Storage's equivalent of a "folder"/bucket.
- **Blob** — a file/object stored inside a container.
- **Service Principal** — a non-human Azure AD identity, used for automation (essentially a "bot account").
- **Tenant** — an Azure AD organization/directory ID.
- **Client Credentials Flow** — OAuth2 method where a token is obtained directly using client_id + client_secret (no user interaction involved).
- **Key Vault** — Azure's secrets/keys/certificates management service.
- **Secret Versioning** — every update creates a new version; the old one remains recoverable by default.
- **`$web` container** — a special reserved container name that Azure Static Website hosting uses internally.

---

_Notes prepared for Obito Uchiha — Team AKATSUKI, based on the CryptoCabana (Byte Lotus Hotel) challenge._