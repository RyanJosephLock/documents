# Setup VS Code

Install VS Code Extensions
 - Gitlens
 - Git History
 - Salesforce Extension Pack (Expanded)

PC Downloads
 - Node.js (to install Salesforce CLI)
 - JDK 21 or higher (https://adoptium.net) (for salesforce extension commands - SDFX)


----
----

# Salesforce CLI (Hint: view more command options with --help)

--

# Connection & Deploy

## Login via Terminal (SF User)
sf org login web --alias srgSIT --instance-url https://superretailgroupservicesptyltd--sit.sandbox.my.salesforce.com/

## View logged in users
sf org list

## Connect Local SF Project (Requires SF project to be in files)
sf config set target-org <username of org>

## Open a previously connected org
sf org open

## Project Deployment (manual and pipeline)
sf project deploy

--

# Generate Components

## Create Apex Class
sf apex generate class --name AccountService --output-dir force-app/main/default

## Create Apex Test Class
sf apex generate class --name AccountServiceTest --output-dir force-app/main/default --template ApexUnitTest

## Create Apex Trigger
sf apex generate trigger --name AccountTrigger --output-dir force-app/main/default/triggers

## Create Objects
sf schema generate sobject --label "Initiatives"

## Create Object Fields
sf schema generate field --label "Start Date" --object force-app/main/default/objects/Initiative__C

--

# Deploy Metadata To Org

## Compare local files with target org, check for components to be deployed (add to .forceignore to remove)
sf project deploy preview --target-org <target org alias>

## Validate all components in project
sf project deploy validate --target-org <target org alias> --source-dir force-app/main/default

## Deploy specific folders or components (Hint: can include multiple --source-dir for multiple files)
sf project deploy start --target-org <target org alias> --source-dir force-app/main/default/classes/AccountService.cls

## Deploy specific metadata
sf project deploy start --target-org <target org alias> --metadata ApexTrigger --metadata ApexClass:AccountService

## Deploy using manifest file
DO: Edit the file manifest/package.xml to include list of what should be deployed (note: can have multiple manifest files)
sf project deploy start --target-org <target org alias> --manifest manifest/package.xml

--

# Retrieve Metadata From Org

## Compare what's in org vs local files
sf project retrieve preview --target-org <target org alias>

## Retrieve ALL metadata from salesforce org (this retrieves everything and WON'T be used 99% of time)
sf project retrieve start --target-org <target org alias>

## Retrieve specific metadata types from salesforce org
sf project retrieve start --target-org <target org alias> --metadata ApexClass (or ApexClass:ClassName)

## Retrieve metadata based on manifest file
DO: Edit the file manifest/package.xml to include list of what should be retrieved (note: can have multiple manifest files)
sf project retrieve start --target-org <target org alias> --manifest manifest/package.xml

--

# Run SOQL & SOQL Using sf cli

## Run SOQL query
sf data query --query "SELECT id, Name, Account.Name FROM Opportunity Order By Amount DESC LIMIT 10" --target-org ExactPath

## Run SOQL query in multiple lines, use backslashes
sf data query \
--query "SELECT id, Name, Account.Name FROM Opportunity Order By Amount DESC LIMIT 10" \
--target-org ExactPath

## Store SOQL query as CSV (ensure save folders exist)
sf data query --query "SELECT id, Name,  Account.Name FROM Opportunity Order By Amount DESC LIMIT 10" --target-org ExactPath --result-format csv --output-file data/query.csv

## Find specific records
sf data search --query "FIND {Anna} IN Name Fields RETURNING Contact(Id, Name, Email)" --target-org ExactPath

Notes: Data command can also get, create, delete, upsert and more (check --help)...

--

# Import & Export data using sf cli (useful for moving data between sandboxes/envs)






!!! CHEAT SHEET STATUS !!!
CONTINUE WRITING FROM UDEMY VIDEO 87 IMPORT & EXPORT DATA USING SF CLI






----
----



# Salesforce Extension Pack Commands (Create Projects and Components)

## Prerequisites to use commands (input into VS Code command prompt )
 - Salesforce CLI
 - Salesforce Extension Pack (Expanded)
 - JDK 21 or higher

## Create New Project
>SFDX: Create Project with Manifest

## Authorize an org (this is the same as the above sf org login web)
>SFDX: Authorize an Org

## Run Diff on Local file vs Org File
Right Click on code > >SFDX: Run Diff...
