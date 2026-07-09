# Salesforce CLI & VS Code Cheat Sheet

> Tip: run `sf <command> --help` on any command below to see the full list of flags/options.

---

## 1. Environment Setup

### VS Code Extensions
- **GitLens**
- **Git History**
- **Salesforce Extension Pack (Expanded)**

### Required Installs
| Tool | Purpose | Link |
|---|---|---|
| Node.js | Needed to install the Salesforce CLI | https://nodejs.org |
| JDK 21+ | Needed for Salesforce Extension Pack / SFDX commands | https://adoptium.net |

---

## 2. Connection & Deploy

### Login via Terminal
```bash
sf org login web --alias <alias> --instance-url https://superretailgroupservicesptyltd--sit.sandbox.my.salesforce.com/
```

### View Logged-in Orgs
```bash
sf org list
```

### Connect/Deploy a Project to an Org
> Requires the SFDX project files to already exist locally.
```bash
sf config set target-org <username-or-alias>
```

### Open a Previously Connected Org
```bash
sf org open
```

### Deploy a Project (manual or pipeline)
```bash
sf project deploy
```

### Deploy a Project with Test Classes (all or selected)
```bash
sf project deploy start --source-dir force/app/main/default --target-org <alias> --test-level RunAllTestsInOrg/RunLocalTests/RunSpecifiedTests/RunRelevantTests/...
sf project deploy start --source-dir force/app/main/default --target-org <alias> --test-level RunSpecifiedTests AccountServiceTest CaseEmailServiceTest
```

### Login using sfdx-url (external application, i.e. ci/cd)
```bash
sf org login sfdx-url tmp/authFile.txt --set-default --alias <alias>
```

---

## 3. Generate Components

| What | Command |
|---|---|
| Apex Class | `sf apex generate class --name AccountService --output-dir force-app/main/default` |
| Apex Test Class | `sf apex generate class --name AccountServiceTest --output-dir force-app/main/default --template ApexUnitTest` |
| Apex Trigger | `sf apex generate trigger --name AccountTrigger --output-dir force-app/main/default/triggers` |
| Custom Object | `sf schema generate sobject --label "Initiatives"` |
| Object Field | `sf schema generate field --label "Start Date" --object force-app/main/default/objects/Initiative__c` |

---

## 4. Deploy Metadata to an Org

### Preview Changes (local vs. org)
> Add unwanted components to `.forceignore` to exclude them.
```bash
sf project deploy preview --target-org <alias>
```

### Validate Only (no deploy)
```bash
sf project deploy validate --target-org <alias> --source-dir force-app/main/default
```

### Deploy Specific File(s)
> Repeat `--source-dir` for multiple files/folders.
```bash
sf project deploy start --target-org <alias> --source-dir force-app/main/default/classes/AccountService.cls
```

### Deploy Specific Metadata Type(s)
```bash
sf project deploy start --target-org <alias> --metadata ApexTrigger --metadata ApexClass:AccountService
```

### Deploy via Manifest File
> Edit `manifest/package.xml` to list what should be deployed. You can maintain multiple manifest files for different purposes.
```bash
sf project deploy start --target-org <alias> --manifest manifest/package.xml
```

---

## 5. Retrieve Metadata from an Org

### Preview Changes (org vs. local)
```bash
sf project retrieve preview --target-org <alias>
```

### Retrieve Everything
> ⚠️ Rarely needed — pulls **all** metadata from the org.
```bash
sf project retrieve start --target-org <alias>
```

### Retrieve Specific Metadata Type(s)
```bash
sf project retrieve start --target-org <alias> --metadata ApexClass
sf project retrieve start --target-org <alias> --metadata ApexClass:ClassName
```

### Retrieve via Manifest File
```bash
sf project retrieve start --target-org <alias> --manifest manifest/package.xml
```

---

## 6. SOQL & SOSL Queries

### Basic SOQL Query
```bash
sf data query --query "SELECT Id, Name, Account.Name FROM Opportunity ORDER BY Amount DESC LIMIT 10" --target-org ExactPath
```

### Multi-line Query (use trailing backslashes)
```bash
sf data query \
  --query "SELECT Id, Name, Account.Name FROM Opportunity ORDER BY Amount DESC LIMIT 10" \
  --target-org ExactPath
```

### Save Query Results as CSV
> Ensure the target output folder already exists.
```bash
sf data query --query "SELECT Id, Name, Account.Name FROM Opportunity ORDER BY Amount DESC LIMIT 10" \
  --target-org ExactPath --result-format csv --output-file data/query.csv
```

### SOSL Search (find specific records)
```bash
sf data search --query "FIND {Anna} IN Name Fields RETURNING Contact(Id, Name, Email)" --target-org ExactPath
```

> 💡 The `sf data` command family also supports `get`, `create`, `delete`, and `upsert` — check `sf data --help` for the full list.

---

## 7. View and Update System Data

### Salesforce CLI - Example Assign Permission Set
```bash
sf org assign permset \
  --name Example_permissionset Stripe_Object_Permissions \
  --on-behalf-of user1@example.com \
  --on-behalf-of user1@example.com \
  --target-org <alias>
```

### Run Apex - Example Assign Permission Set
```bash
sf apex run --file scripts/apex/hello.apex --target-org <alis>
```

> 💡 Running Apex can execute a range of things in pipelines or run manually, including assigning Permission Sets.

### Check the API version and the available limits in the org (i.e credits, scratch org, etc)
```bash
sf limits api display
sf limits api display --target-org <alias>
sf limits api display -o order-management | grep Scratch
```

---

## 8. Import & Export Data

### Export Data (Tree Format)
> Exports records as JSON tree files — useful for **migrating data between orgs/sandboxes** since it preserves parent-child relationships.
```bash
sf data export tree --query "SELECT Id, Name FROM Account" --target-org ExactPath --output-dir data
```

### Import Data (Tree Format)
```bash
sf data import tree --files data/Account.json --target-org ExactPath
```

### Generate an Export Plan
> `--plan` generates a plan definition file that groups multiple related objects (e.g. Account + Contact) into a single, ordered import/export job.
```bash
sf data export tree --query "SELECT Id, Name FROM Account" --target-org ExactPath --output-dir data --plan
```

> 💡 SOQL queries can be referenced from a file instead of typed inline — handy for long or reused queries.
> ```bash
> sf data export tree --query-file queries/accounts.soql --target-org ExactPath --output-dir data
> ```

### Bulk API — Export Data
```bash
sf data export bulk --query "SELECT Id, Name FROM Account" --target-org ExactPath --output-file data/accounts.csv
```

### Bulk API — Import Data
```bash
sf data import bulk --sobject Account --file data/accounts.csv --target-org ExactPath
```

> 💡 Bulk API commands (`export bulk` / `import bulk`) are best for **large data volumes** (CSV-based), while `export tree` / `import tree` are best for **small, relational datasets** (JSON-based, preserves record relationships).

---

## 9. Scratch Org

> 💡 Scratch orgs are temporary, configurable Salesforce environments used for development and CI/CD. Created in minutes and lasting up to 30 days. Limit 3 scratch orgs at a time.

### Create Scratch Org
```bash
1. Ensure Dev Hub and Source Tracking in Developer and Developer Pro Sandboxes is enabled. Dev Hub enables ActiveStratchOrg and ScratchOrgInfo objects (note custom fields can be added to ScratchOrgInfo to customise created scratch orgs).

2. Authorize an existing Salesforce Org (SFDX command or CLI)
>SFDX: Authorize a Dev Hub
sf org login web --set-default-dev-hub --alias MAIN_ORG

3. Scratch org definition file example (Docs: https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm)
{
  "orgName": "PantherSchools",
  "edition": "Enterprise",
  "hasSampleData": true,
  "adminEmail": "youremail@gmail.com",
  "description": "This is sample org used for the deployment validation purpose!",
  "workItem__c": "TICKET-23245",
  "userStory__c": "US-12345",
  "features": [
    "EnableSetPasswordInApi"
  ],
  "settings": {
    "lightningExperienceSettings": {
      "enableS1DesktopEnabled": true
    },
    "mobileSettings": {
      "enableS1EncryptedStoragePref2": false
    }
  }
}

4. Create scratch org
sf org create scratch --target-dev-hub MAIN_ORG --definition-file config/project-scratch-def.json --set-default --duration-days 1

5. Deploy project to scratch org (the target org is already set to scratch org)
sf project deploy start --source-dir force-app

6. Scratch Org can be viewed in UI "App Launcher > Active Scratch Orgs" and "App Launcher > Active Scratch Org Info"

```

### Useful Scratch Org Commands
```bash
CHANGE PASSWORD
sf org generate password --length 25 --complexity 5 --target-org test-jksdhfds23@example.com

DELETE SCRATCH ORG
sf org delete scratch --target-org test-jksdhfds23@example.com
```

---

## 10. Salesforce Extension Pack Commands (VS Code Command Palette)

**Prerequisites:**
- Salesforce CLI
- Salesforce Extension Pack (Expanded)
- JDK 21+

| Action | Command Palette Entry |
|---|---|
| Create a new project with manifest | `SFDX: Create Project with Manifest` |
| Authorize an org (same as `sf org login web`) | `SFDX: Authorize an Org` |
| Diff a local file against the org version | Right-click file → `SFDX: Run Diff...` |
