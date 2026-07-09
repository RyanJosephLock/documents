# Salesforce JWT Bearer Authentication - Setup Guide

A complete reference for setting up passwordless, server-to-server Salesforce authentication using the OAuth 2.0 JWT Bearer flow with External Client Apps (ECA) - the modern replacement for Connected Apps in Salesforce.

> **Why External Client Apps?** Starting Spring '26, Salesforce requires new integrations to use External Client Apps (ECAs) instead of Connected Apps. ECAs are fully metadata-compliant, support modern CI/CD workflows, enforce a closed security model by default, and cleanly separate developer-controlled settings from admin-controlled subscriber policies.

## Overview

The JWT Bearer Token flow allows a server application to authenticate to Salesforce without any user interaction or browser redirect. The client proves its identity by signing a JSON Web Token (JWT) using a private key. Salesforce validates the signature using the public certificate registered in an External Client App.

```
App (server.key) → signs JWT → Salesforce verifies with server.crt → returns access_token
```

**When to use this flow:**

- CI/CD pipelines (GitHub Actions, Jenkins, Azure DevOps, Salesforce DevOps Center)
- ETL tools and middleware integrations
- Scheduled Apex-to-external or external-to-Salesforce API calls
- Any scenario where a human login is not possible

## Prerequisites

- **OpenSSL** - Pre-installed on macOS/Linux. Download for Windows
- **Salesforce CLI (SFDX)**
- **Salesforce org access**

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


## Phase 1 - Create Private Key and Certificate

All commands below are run in your terminal from a dedicated folder (e.g. `~/jwt/`).

### Step 1.1 - Set OpenSSL config path (Windows only)

```bash
set OPENSSL_CONF=C:\openssl\share\openssl.cnf
```

Skip this step on macOS and Linux.

### Step 1.2 - Generate an encrypted RSA private key

Creates a DES3-encrypted 2048-bit RSA private key. The passphrase `x` is temporary and is only used to produce the clean key in the next step.

```bash
openssl genrsa -des3 -passout pass:x -out server.pass.key 2048
```

**Output:** `server.pass.key` - the encrypted private key file.

> **What is the encrypted file?** `server.pass.key` is the encrypted form of your private key. The `-des3` flag applies Triple DES symmetric encryption using the passphrase you supply. This protects the raw key material at rest. You would use this file when a tool or workflow specifically requires an encrypted PEM key.

### Step 1.3 - Strip encryption and create the plain key

Extracts the raw RSA private key from the encrypted file. Most tools - including the Salesforce CLI - consume this unencrypted version directly.

```bash
openssl rsa -passin pass:x -in server.pass.key -out server.key
```

**Output:** `server.key` - the unencrypted RSA private key.

> You can safely delete `server.pass.key` after this step. Keep `server.key` secret - it acts as a password. Anyone with this file can authenticate as your ECA integration user.

### Step 1.4 - Generate the Certificate Signing Request (CSR)

```bash
openssl req -new -key server.key -out server.csr
```

You will be prompted to fill in identity details. These are embedded in the certificate:

| Field | Example |
|---|---|
| Country Name | `IN` |
| State or Province | `Uttar Pradesh` |
| Locality Name | `Ghaziabad` |
| Organization Name | `Your Company Name` |
| Organizational Unit | `DevOps` |
| Common Name | (leave blank or use your org domain) |
| Email Address | `you@yourcompany.com` |

**Output:** `server.csr` - the certificate signing request.

### Step 1.5 - Self-sign the X.509 certificate

Signs the CSR with your own private key to produce a self-signed certificate valid for 365 days.

```bash
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
```

**Output:** `server.crt` - the X.509 certificate. This is the file you upload to the External Client App.

### Files produced

| File | Description |
|---|---|
| `server.pass.key` | Encrypted private key |
| `server.key` | Plain RSA private key for JWT signing |
| `server.csr` | Certificate signing request |
| `server.crt` | X.509 certificate - upload to ECA |

## Phase 2 - Create an External Client App (ECA)

### Step 2.1 - Navigate to External Client App Manager

1. Go to Salesforce Setup
2. In the Quick Find box, type `External`
3. Select **External Client App Manager** from the dropdown
4. Click **New External Client App** (top right corner)

### Step 2.2 - Basic information

Fill in the following fields:

- **External Client App Name** - A human-readable name (e.g. `JWT Integration App`)
- **API Name** - Auto-filled based on the name; used in metadata and deployments
- **Distribution State** - `Local` for single-org use; `Packageable` for ISV/multi-org scenarios
- **Description** - (Optional) Describe the integration purpose

### Step 2.3 - Enable OAuth and upload certificate

1. Scroll down and expand the **API (Enable OAuth Settings)** section
2. Click **Enable OAuth**
3. Enter the Callback URL:

   ```
   http://localhost:1717/OauthRedirect
   ```

4. Under **Flow Enablement**, select **JWT Bearer Flow**
5. Upload your `server.crt` certificate file when prompted for a public certificate

### Step 2.4 - Select OAuth scopes and enable JWT flow

Select the following OAuth Scopes:

- Manage user data via APIs (`api`)
- Manage user data via Web browsers (`web`)
- Perform requests at any time (`refresh_token`, `offline_access`)

Add additional scopes based on your integration requirements.

Click **Create** to save the ECA.

### Step 2.5 - Configure OAuth policies (pre-authorization)

After creating the ECA, navigate to the **Policies** tab and click **Edit**:

1. Under **OAuth Policies**, set:
   - **Permitted Users** → `Admin approved users are pre-authorized`
   - Confirm the warning prompt if it appears
2. Under **App Policies**, add the appropriate Profile or Permission Set for the integration user
3. Click **Save**

> **Without this step, JWT authentication will fail with:**
>
> ```json
> {"error":"invalid_grant","error_description":"user hasn't approved this consumer"}
> ```
>
> Or the ECA-specific error:
>
> ```json
> {"error":"app_not_found","error_description":"External client app is not installed in this org"}
> ```

### Step 2.6 - Retrieve Consumer Key

1. On the ECA detail page, click the **Settings** tab
2. Under **OAuth Settings**, click **Consumer Key and Secret**
3. Verify your identity with the emailed verification code
4. Copy and securely store the **Consumer Key** - you will need it for JWT claims and CLI commands

## Phase 3 - Authenticate and Get an Access Token

### Salesforce CLI (SFDX)

```bash
sf org login jwt \
  --username <Your-username> \
  --jwt-key-file server.key \
  --client-id <Client-ID> \
  --alias pre-release-org \
  --set-default \
  --instance-url https://login.salesforce.com
```

**On success:**

```
Successfully authorized your@salesforce.user with org ID 00DXXXXXXXXXXXXXXX
```

**Deploy using Salesforce CLI:**

```bash
sf project deploy start --source-dir force-app
```
