# Kubernetes

### Configure Kubernetes TokenReview SA

https://www.vaultproject.io/docs/auth/kubernetes#configuring-kubernetes

### Get Kubernetes Data

```
export K8S_HOST="K8S_IP_ADDR"
export SA_NAME=$(kubectl get serviceaccount vault-auth -o go-template='{{ (index .secrets 0).name }}')
export SA_JWT_TOKEN=$(kubectl get secret ${SA_NAME} -o go-template='{{ .data.token }}' | base64 --decode)
export SA_CA_CRT=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode)
```

### Enable Kubernetes Auth in both Namespaces

```
vault auth enable -namespace=engineering kubernetes
```

```
vault auth enable -namespace=operations kubernetes
```

### Create Policies

```
vault policy write -namespace=engineering app-a app-a.hcl
vault policy write -namespace=engineering app-b app-b.hcl

vault policy write -namespace=operations app-a app-a.hcl
vault policy write -namespace=operations app-b app-b.hcl
```

### Write Kubernetes Auth Config

```
vault write -namespace=engineering auth/kubernetes/config token_reviewer_jwt="$SA_JWT_TOKEN" kubernetes_host="https://$K8S_HOST:443" kubernetes_ca_cert="$SA_CA_CRT"
```

```
vault write -namespace=operations auth/kubernetes/config token_reviewer_jwt="$SA_JWT_TOKEN" kubernetes_host="https://$K8S_HOST:443" kubernetes_ca_cert="$SA_CA_CRT"
```

### Create Kubernetes Roles

```
vault write -namespace=operations auth/kubernetes/role/app-a \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=app-a \
        ttl=24h
```
```
vault write -namespace=operations auth/kubernetes/role/app-b \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=app-b \
        ttl=24h
```
```
vault write -namespace=engineering auth/kubernetes/role/app-a \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=app-a \
        ttl=24h
```
```
vault write -namespace=engineering auth/kubernetes/role/app-b \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=app-b \
        ttl=24h
```

# Database

### Enable Database Secret Engine

```
vault secrets enable -namespace=engineering database
vault secrets enable -namespace=operations database
```

### Create DB Config

https://www.vaultproject.io/docs/secrets/databases/postgresql

```
vault write -namespace=engineering database/config/my-postgresql-database \
    plugin_name=postgresql-database-plugin \
    allowed_roles="postgres-root-role" \
    connection_url="postgresql://{{username}}:{{password}}@IP_ADDRESS_GOES_HERE:5432/" \
    username="postgres" \
    password="vaultpassword"
```

```
vault write -namespace=operations database/config/my-postgresql-database \
    plugin_name=postgresql-database-plugin \
    allowed_roles="postgres-root-role" \
    connection_url="postgresql://{{username}}:{{password}}@IP_ADDRESS_GOES_HERE:5432/" \
    username="postgres" \
    password="vaultpassword"
```

### Create Role

```
vault write -namespace=engineering database/roles/postgres-root-role \
    db_name=my-postgresql-database \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

```
vault write -namespace=operations database/roles/postgres-root-role \
    db_name=my-postgresql-database \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

### Get Credentials

```
vault read -namespace=operations database/creds/postgres-root-role
```

# AppRole

### Enable AppRole Auth

```
vault auth enable -namespace=operations approle
```

### Create AppRole

```
vault write -namespace=operations auth/approle/role/pipeline \
    secret_id_ttl=10m \
    token_num_uses=10 \
    token_ttl=20m \
    token_max_ttl=30m \
    secret_id_num_uses=40 \
    token_policies="app-a,app-b"
```

### Get Role ID

```
vault read -namespace=operations auth/approle/role/pipeline/role-id
```

### Get Secret ID

```
vault write -namespace=operations -f auth/approle/role/pipeline/secret-id
```

### Login

```
vault write auth/approle/login \
    role_id=db02de05-fa39-4855-059b-67221c5c2f63 \
    secret_id=6a174c20-f6de-a53c-74d2-6018fcceff64
```

# Demo

```
vault write -namespace=operations secret/test key=value
```

```
kubectl apply -f app-kv.yaml
```

```
kubectl apply -f app-db.yaml
```