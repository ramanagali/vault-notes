# Vault Commands


## 1 Vault Initialization

```
vault operator init -key-shares=5 -key-threshold=3

vault operator unseal key1
vault operator unseal key2
vault operator unseal key3


vault login 
vault login token=<<root_token>>


vault operator rekey \
    -init \
    -key-shares=3 \
    -key-threshold=2 \
    -verify

vault operator rekey \
    -target=recovery \
    -init \
    -key-shares=3 \
    -key-threshold=2
    
vault server -dev

export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN="s.XmpNPoi9sRhYtdKHaQhkHP6x"

vault status
```

## 2 SECRETS

```
vault secrets list
vault secrets list -detailed


vault write cubbyhole/test 
vault read cubbyhole/testt
```

### 2.1 Secrets - Key Value 

```
vault secrets enable kv
vault secrets enable -path=gvr kv

vault kv put gvr/webui user=venkat pass=xyz
vault kv get gvr/webui
vault kv list gvr/webui

vault secrets tune -default-lease-ttl=72h gvr/

vault kv delete gvr/webui

vault secrets disable gvr

```
### 2.2 Secrets - With Version

```
vault secrets enable -version=2 -path=gvr kv

vault write kv/config max_versions=4
vault read kv/config

vault kv put kv/cus name="gvr" pass="1111"
vault kv put kv/cus name="gvr" pass="2222"
vault kv put kv/cus name="gvr" pass="3333"

vault kv get -version=1 kv/cus
vault kv get -version=2 kv/cus

vault kv delete -versions=1 kv/cus
vault kv undelete -versions=1 kv/cus

vault kv destroy -versions=2 kv/cus


vault kv metadata get kv/cus
vault kv metadata put -mount=kv/cus -max-versions 3 -delete-version-after="1h1m1s" kv

vault kv metadata delete kv/cus

vault secrets disable gvr
```

### 2.2 Dynamic Secrets - AWS

```
vault secrets enable -path=aws aws

export AWS_ACCESS_KEY_ID=""
export AWS_SECRET_ACCESS_KEY=""

vault write aws/config/root access_key=$AWS_ACCESS_KEY_ID secret_key=$AWS_SECRET_ACCESS_KEY region=ap-southeast-1

vault write aws/roles/ec2-role \
        credential_type=iam_user \
        policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF

vault read aws/creds/ec2-role
vault lease revoke aws/creds/ec2-role/0bce0782-32aa-25ec-f61d-c026ff22106

vault secrets disable -path=aws aws
```
## 2 Vault AUTHENTICATION

### 2.1 token
```
vault login
vault login token=hvs.f6OTpenqZRwL4tKGapFM8yoP

vault auth list

vault token create
vault login token=hvs.f6OTpenqZRwL4tKGapFM8yoP
vault token revoke s.xxxxxx

vault path-help auth/token
```

### 2.2 userpass
```
vault auth enable userpass

vault write auth/userpass/users/gvr password=12345 
vault read auth/userpass/users/gvr

vault login -method=userpass username=gvr password=12345


vault auth disable userpass	

vault path-help auth/userpass
```

### 2.3 github
```
vault auth enable github

vault write auth/github/config organization=learnwithgvr

vault write auth/github/map/teams/dev value=default,applicationsvault write auth/github/map/users/gvr value=gvr-policy

vault read auth/github/map/users/gvr

vault auth disable github
vault path-help auth/github
```

### 2.4 aws
```
vault auth enable aws
vault path-help auth/aws

vault write auth/aws/config/client \
    secret_key=******************** \
    access_key=********************

AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY

vault write auth/aws/role/dev-role \
    auth_type=ec2 \
    bound_ami_id=ami-fce3c696 \
    policies=prod,dev \
    max_ttl=500h

vault write auth/aws/role/dev-role-iam \
    auth_type=iam \
    bound_iam_principal_arn=arn:aws:iam::123456789012:role/MyRole \
    policies=prod,dev \
    max_ttl=500h

vault auth disable aws
vault path-help auth/aws
```
# 3 app role pull authentication
```
vault policy write jenkins -<<EOF
# Read-only permission on secrets stored at 'secret/mysql/webapp'
path "secret/mysql/webapp" {
  capabilities = [ "read" ]
}
EOF

vault secrets enable -path=secret/mysql kv
vault kv put secret/mysql/webapp db_name="users" username="admin" password="passw0rd"
vault kv get secret/mysql/webapp

vault auth enable approle

vault write auth/approle/role/jenkins \
    token_policies="jenkins" \
    token_ttl=1h \
    token_max_ttl=4h \
    secret_id_num_uses=10

vault read auth/approle/role/jenkins
vault read auth/approle/role/jenkins/role-id
vault read auth/approle/role/jenkins/role-id -format=json | jq -r ".data.role_id > role_id.txt


vault write -force auth/approle/role/jenkins/secret-id
vault write -f auth/approle/role/jenkins/secret-id -format=json| jq -r ".data.secret_id" > secretid
vault write -wrap-ttl=60s -force auth/approle/role/jenkins/secret-id -format=json

vault write auth/approle/login role_id=  secret_id=

VAULT_TOKEN="" vault kv get secret/mysql/webapp

vault auth disable approle
```

# 4 Policies
```
vault secrets list
vault auth list

vault kv get gvr/webui
vault read auth/userpass/users/gvr


vault policy write my-policy ./my-policy.hcl

vault policy write web-policy - << EOF
path "gvr/dev/*" {
  capabilities = ["create", "update", "read"]
}

path "gvr/webui" {
  capabilities = ["read"]
}
EOF

vault token create -policy=test-policy.hcl
vault login token=
vault token capabilities gvr/webui/


vault kv put gvr/dev/ user="venkat"
vault kv put gvr/webui/ user="venkat"

vault token capabilities $ADMIN_TOKEN sys/auth/approle
vault token capabilities $ADMIN_TOKEN identity/entity
```
# 5 TOKENS

### 5.1. Tokens with use limit  
```
vault token create -help

vault token create -ttl=1h -use-limit=2 -policy=default

export USE_LIMIT_TOKEN=

VAULT_TOKEN= vault token lookup

VAULT_TOKEN= vault write cubbyhole/token value=1234567890

VAULT_TOKEN= vault read cubbyhole/token
```

### 5.2 Periodic service tokens
```
-- Root or sudo users have the ability to generate periodic tokens ---

vault token create -policy="default" -period=24h

vault token create -policy="default" -period=24h -format=json | jq -r ".auth.client_token" > periodic_token.txt

vault token lookup $(cat periodic_token.txt)
```

### 5.3 Short-lived tokens
```
vault token create -ttl=60s -format=json | jq -r ".auth.client_token" > short-lived_token.txt
vault token lookup $(cat short-lived_token.txt)
```

### 5.4 Orphan tokens
```
vault token create -orphan -format=json | jq -r ".auth.client_token" > orphan_token.txt
vault token lookup $(cat orphan_token.txt)
```

### 5.5 Token Role
```
vault write auth/token/roles/zabbix \
    allowed_policies="policy1, policy2, policy3" \
    orphan=true \
    period=8h

vault token create -role=zabbix
```

### 5.5 Renew service tokens
```
vault token create -ttl=45 -explicit-max-ttl=120 -policy=default -format=json | jq -r ".auth.client_token" > test_token.txt

vault token renew $(cat test_token.txt)
vault token renew -increment=60 $(cat test_token.txt)
```

### 5.6 Revoke service tokens
```
vault policy write test -<<EOF
path "auth/token/create" {
   capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF

vault token create -ttl=60 -policy=test -format=json | jq -r ".auth.client_token" > parent_token.txt

VAULT_TOKEN=$(cat parent_token.txt) \
   vault token create -ttl=180 -policy=default -format=json \
    | jq -r ".auth.client_token" > child_token.txt


VAULT_TOKEN=$(cat parent_token.txt) \
   vault token create -orphan -ttl=180 -policy=default -format=json \
    | jq -r ".auth.client_token" > orphan_token.txt

vault token revoke $(cat parent_token.txt)
vault token lookup $(cat parent_token.txt)
vault token lookup $(cat child_token.txt)
vault token lookup $(cat orphan_token.txt)
```

### 5.7 apply token types
```
vault auth enable approle
vault write auth/approle/role/jenkins policies="jenkins" period="24h"

vault read -format=json auth/approle/role/jenkins/role-id \
    | jq -r ".data.role_id" > role_id.txt

vault write -f -format=json auth/approle/role/jenkins/secret-id \
    | jq -r ".data.secret_id" > secret_id.txt

vault write auth/approle/login role_id=$(cat role_id.txt) \
     secret_id=$(cat secret_id.txt)

vault token lookup <returned_token>
```

# 6. Leases
```
export AWS_ACCESS_KEY_ID=********************
export AWS_SECRET_ACCESS_KEY=********************=********************

vault secrets enable -path=aws aws
vault secrets tune -default-lease-ttl=3m -max-lease-ttl=5m /aws

vault write aws/config/root \
    access_key=$AWS_ACCESS_KEY_ID \
    secret_key=$AWS_SECRET_ACCESS_KEY \
    region=ap-southeast-1


vault write aws/roles/test-role credential_type=iam_user \
policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF

vault read aws/creds/test-role

vault list sys/leases/lookup/aws/creds/my-role

vault lease renew aws/creds/my-role/MChg3hoGrotBL3JlRoMDIvj4

vault lease revoke aws/creds/my-role/0bce0782-32aa-25ec-f61d-c026ff22106

vault secrets disable aws

vault write -f aws/config/rotate-root

```
# 7. Transit Engine
```
vault secrets enable transit

vault write -f transit/keys/my-key

vault write transit/encrypt/my-key plaintext=$(base64 <<< "test123")

vault write transit/decrypt/my-key ciphertext=

echo "dGVzdDEyMwo=" | base64 -d

vault write -f transit/keys/my-key/rotate

vault write transit/rewrap/my-key ciphertext=

vault write transit/decrypt/my-key ciphertext=

vault write -f transit/datakey/plaintext/my-key
```

# 8 Vault API

```
curl http://127.0.0.1:8200/v1/sys/init

curl \
    --request POST \
    --data '{"key": "fwqMMIc12dYP1H14XQo+EN0VMVd42FSDkz9rqigN0oY4"}' \
    http://127.0.0.1:8200/v1/sys/unseal | jq


curl -H "X-Vault-Token: $(vault print token)" -X LIST http://127.0.0.1:8200/v1/auth/userpass/users
curl -H "X-Vault-Token: $(vault print token)" -X LIST http://127.0.0.1:8200/v1/auth/userpass/users | jq '.'

curl -H "X-Vault-Token: $(vault print token)" -X LIST http://127.0.0.1:8200/v1/auth/token/accessors
curl -H "X-Vault-Token: $(vault print token)" -X LIST http://127.0.0.1:8200/v1/auth/token/accessors | jq '.'

curl -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/auth/token/lookup-self
curl -H "X-Vault-Token: $(vault print token)" -X POST http://127.0.0.1:8200/v1/auth/token/create
curl -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/auth/token/lookup-self


curl -H "X-Vault-Token: $(vault print token)" https://127.0.0.1:8200/v1/secret/config

curl -H "X-Vault-Request: true" -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/auth/userpass/users/gvr

curl --header "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/auth/userpass/users/gvr | jq

curl -H "X-Vault-Request: true" -H "X-Vault-Token: $(vault print token)" http://127.0.0.1:8200/v1/gvr/webui | jq '.data.username'

```

## Generete curl API command using CLI

```
vault auth enable -output-curl-string userpass

vault read -output-curl-string auth/userpass/users/gvr

vault secrets list -output-curl-string

vault kv get -output-curl-string gvr/webui

```
 <hr />
 
* [Provision Vault Server using Vagrant](https://github.com/ramanagali/vault-server)
* [Vault youtube playlist - my youtube channel](https://www.youtube.com/playlist?list=PLFkEchqXDZx7CuMTbxnlGVflB7UKwf_N3) 
