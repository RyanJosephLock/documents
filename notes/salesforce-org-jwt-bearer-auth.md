# Salesforce JWT Bearer Authentication Cheat Sheet

Passwordless server-to-server authentication using OAuth 2.0 JWT Bearer Flow with External Client Apps (ECA)


## Why External Client Apps (ECA)?

Beginning with **Spring '26**, Salesforce requires **External Client Apps (ECAs)** for new integrations instead of Connected Apps.

### Benefits

* ✅ Fully metadata compliant
* ✅ CI/CD friendly
* ✅ Closed security model by default
* ✅ Separation of developer configuration and admin policies


## Best Practice

Create a dedicated **service (integration) user** specifically for automated deployments instead of using a developer's personal account.

Benefits include:

* Clear audit history for automated deployments
* Deployments continue if team members leave
* Least-privilege permissions can be assigned
* No dependency on personal credentials
* Easier management of profiles and permission sets

> **Important:** The External Client App does **not** replace the Salesforce user. It replaces the OAuth client (formerly the Connected App). A valid, active Salesforce user with the appropriate permissions is still required for every JWT Bearer authentication.


## Ideal Use Cases

* CI/CD Pipelines

  * GitHub Actions
  * Jenkins
  * Azure DevOps
  * Salesforce DevOps Center
* ETL integrations
* Middleware
* Scheduled API jobs
* Any server-side integration requiring passwordless authentication

  
## Prerequisites

* OpenSSL
  * macOS/Linux: preinstalled
  * Windows: install OpenSSL
* Salesforce CLI (`sf`)
* Salesforce Org access


## Authentication Identity

Although the JWT Bearer Flow is **passwordless**, it is **not userless**.

Every successful JWT authentication is performed **as a specific Salesforce user** (typically a dedicated service or integration user). The External Client App authenticates the application, while the Salesforce user provides the security context and permissions for all API operations.

The JWT includes the following key claims:

| Claim | Purpose                                    |
| ----- | ------------------------------------------ |
| `iss` | The External Client App Consumer Key       |
| `sub` | The Salesforce username to authenticate as |
| `aud` | The Salesforce login URL                   |
| `exp` | JWT expiration time                        |

Authentication flow:

```text
CI/CD Pipeline
      │
      │ Signs JWT using server.key
      ▼
External Client App
      │
      │ Validates certificate and Consumer Key
      ▼
Authenticates the specified Salesforce User
      │
      ▼
Returns an OAuth Access Token
      │
      ▼
All Metadata and API operations execute using that user's permissions

```


---

# Phase 1 — Generate Keys & Certificate

Run all commands from a dedicated folder.

```text
~/jwt/
```


## Step 1.1 (Windows Only)

Set the OpenSSL configuration path.

```cmd
set OPENSSL_CONF=C:\openssl\share\openssl.cnf
```

> Skip on macOS and Linux.


## Step 1.2 Generate Encrypted RSA Private Key

```bash
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
```

Produces:

```
server.pass.key
```

### Purpose

An encrypted 2048-bit RSA private key protected with Triple DES.


## Step 1.3 Create Plain Private Key

```bash
openssl rsa -passin pass:x -in server.pass.key -out server.key
```

Produces:

```
server.key
```

### Notes

* Used directly by Salesforce CLI
* Keep this file secret
* Delete `server.pass.key` after creation if no longer needed


## Step 1.4 Generate Certificate Signing Request (CSR)

```bash
openssl req -new -key server.key -out server.csr
```

Example values:

| Field               | Example                                           |
| ------------------- | ------------------------------------------------- |
| Country             | IN                                                |
| State               | Uttar Pradesh                                     |
| Locality            | Ghaziabad                                         |
| Organization        | Your Company Name                                 |
| Organizational Unit | DevOps                                            |
| Common Name         | Leave blank or org domain                         |
| Email               | [you@yourcompany.com](mailto:you@yourcompany.com) |

Produces:

```
server.csr
```

---

## Step 1.5 Create Self-Signed Certificate

```bash
openssl x509 \
  -req \
  -sha256 \
  -days 365 \
  -in server.csr \
  -signkey server.key \
  -out server.crt
```

Produces:

```
server.crt
```

This certificate is uploaded into Salesforce.

# Generated Files

| File              | Purpose                                         |
| ----------------- | ----------------------------------------------- |
| `server.pass.key` | Encrypted private key                           |
| `server.key`      | Plain RSA private key used for JWT signing      |
| `server.csr`      | Certificate Signing Request                     |
| `server.crt`      | Public X.509 certificate uploaded to Salesforce |

---

# Phase 2 — Create an External Client App (ECA)

## Step 2.1 Navigate

```
Setup
    ↓
Quick Find
    ↓
External Client App Manager
    ↓
New External Client App
```


## Step 2.2 Basic Information

Fill in:

| Field                    | Value                             |
| ------------------------ | --------------------------------- |
| External Client App Name | JWT Integration App               |
| API Name                 | Auto-generated                    |
| Distribution State       | Local (single org) or Packageable |
| Description              | Optional                          |


## Step 2.3 Enable OAuth

Navigate to:

```
API
    ↓
Enable OAuth Settings
```

Enable:

* OAuth

Callback URL

```text
http://localhost:1717/OauthRedirect
```

Flow Enablement

* ✅ JWT Bearer Flow

Upload

```
server.crt
```


## Step 2.4 OAuth Scopes

Recommended scopes:

* API (`api`)
* Web (`web`)
* Refresh Token (`refresh_token`)
* Offline Access (`offline_access`)

Add any additional scopes required by your integration.

Click **Create**.


## Step 2.5 OAuth Policies (Required)

After creation:

```
Policies
    ↓
Edit
```

### OAuth Policies

```
Permitted Users

→ Admin approved users are pre-authorized
```

### App Policies

Assign either:

* Integration User Profile

or

* Permission Set

Save.


### Common Errors

**User not pre-authorized**

```json
{
  "error":"invalid_grant",
  "error_description":"user hasn't approved this consumer"
}
```

**App not installed**

```json
{
  "error":"app_not_found",
  "error_description":"External client app is not installed in this org"
}
```


## Step 2.6 Retrieve Consumer Key

Navigate to:

```
Settings
    ↓
OAuth Settings
    ↓
Consumer Key and Secret
```

Verify your identity.

Copy the:

```
Consumer Key
```

You'll need it for:

* JWT claims
* Salesforce CLI authentication

---

# Phase 3 — Authenticate

## Salesforce CLI Login

```bash
sf org login jwt \
  --username <USERNAME> \
  --jwt-key-file server.key \
  --client-id <CONSUMER_KEY> \
  --alias pre-release-org \
  --set-default \
  --instance-url https://login.salesforce.com
```

Success:

```text
Successfully authorized your@salesforce.user
with org ID 00DXXXXXXXXXXXXXXX
```

---

# Deploy Example

```bash
sf project deploy start \
  --source-dir force-app
```

---

# Quick Reference

## Required Files

| File                | Used Where       |
| ------------------- | ---------------- |
| `server.key`        | CLI JWT login    |
| `server.crt`        | Upload to ECA    |
| Consumer Key        | CLI & JWT claims |
| Salesforce Username | CLI login        |


## JWT Authentication Checklist

* [ ] Install OpenSSL
* [ ] Install Salesforce CLI
* [ ] Generate encrypted key
* [ ] Generate `server.key`
* [ ] Generate CSR
* [ ] Generate `server.crt`
* [ ] Create External Client App
* [ ] Enable OAuth
* [ ] Enable JWT Bearer Flow
* [ ] Upload `server.crt`
* [ ] Select OAuth scopes
* [ ] Set **Admin approved users are pre-authorized**
* [ ] Assign Profile/Permission Set
* [ ] Copy Consumer Key
* [ ] Authenticate using `sf org login jwt`
* [ ] Deploy or invoke Salesforce APIs
