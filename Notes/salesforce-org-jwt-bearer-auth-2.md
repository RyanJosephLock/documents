## Follow the below steps to generate the Certificate file
Step1 - openssl genpkey -aes-256-cbc -algorithm RSA -pass pass:DEV_SANDBOX -out assets/dev/server.pass.key -pkeyopt rsa_keygen_bits:2048

Step2 - openssl rsa -passin pass:DEV_SANDBOX -in assets/dev/server.pass.key -out assets/dev/server.key

Step3 - openssl req -new -key assets/dev/server.key -out assets/dev/server.csr

Step4 - openssl x509 -req -sha256 -days 365 -in assets/dev/server.csr -signkey assets/dev/server.key -out assets/dev/server.crt

CREATE External Client APP ( use http://localhost:1717/OauthRedirect as Callback in the External Client App)

## run the below comment to authenticate Salesforce using JWT
  sf org login jwt --client-id <YOUR-CLIENT-ID>
  --jwt-key-file assets/dev/server.key
  --username <YOUR-USERNAME-ID>
  --set-default --alias DEV_INT_ORG
  --instance-url https://test.salesforce.com

## Generate the Encryption Key & IV
openssl enc -aes-256-cbc -k GITHUBACTIONS_DEV_SANDBOX -P -md sha1 -nosalt

## Run the following command to Encrypt the server.key file
openssl enc -nosalt -aes-256-cbc -in assets/dev/server.key -out assets/dev/server.key.enc -base64 -K -iv

### Run the below comment to
DECRYPT the enc file to key file

openssl enc -nosalt -aes-256-cbc -d -in assets/dev/server.key.enc -out assets/dev/server.key -base64 -K -iv

### Congigure the Github Environments
ENVIRONMENT SPECIFIC SECRETS TO CREATE

1. DEPLOYMENT_USER_USERNAME - the salesforce username that needs to perform the validation/deployment. ( This should be the deployment username )
2. DECRYPTION_KEY - is the value of the Key file to decrypt the server.key.inc file
3. DECRYPTION_IV - is the value of the IV file to decrypt the server.key.inc file
4. CONSUMER_KEY - the salesforce connected application id
5. ENCRYPTION_KEY_FILE - the location of the encrypted file that is assets/dev/server.key.inc
6. JWT_KEY_FILE - the location to place the decrypted key file and the value should be asset/dev/server.key
7. ENVIRONMENT SPECIFIC VARIABLES TO CREATE
8. HUB_LOGIN_URL - the salesforce login url depending upon whether it is salesforce sandbox or production
ORG_DEFAULT_ALIAS
