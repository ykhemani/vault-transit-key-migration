# Securely migration a Vault Transit encryption key from one Vault cluster to another

If you're in a situation where you need to migrate a [Vault](https://vaultproject.io) [Transit](https://developer.hashicorp.com/vault/docs/secrets/transit) encryption key from one Vault cluster to another, we can do this securely using the steps described below.

To do this, we'll be using the Vault [CLI](https://developer.hashicorp.com/vault/docs/install) as well as the Vault API. Although, you may do everything we describe below using the Vault API.

## Prequisites

In order to perform the steps below, you'll need the Vault CLI and the Vault.

1. On the Source Cluster: Create the key to be exported if it doesn't already exist. Note that the key must be exportable.

```
curl \
  -sk \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  --request POST \
  --data "{ \"exportable\" : true }" \
  $VAULT_ADDR/v1/transit/keys/key_to_export | \
  jq
```


2. On the Destination Cluster: Enable the Transit secret engine if it isn't already enabled.

```
vault secrets enable transit
```

3. On the Destination Cluster: Export the wrapping key.

```
vault read \
  transit/wrapping_key \
  -format=json
```

4. On the Source Cluster: Create the JSON payload for importing the wrapping key exported from the Destination Cluster (vault2_wrapping_key.json)
```
{
  "type" : "rsa-4096",
  "public_key" : "<public_key from the Destination Cluster>"
}
```

5. On the Source Cluster: Import the wrapping key from the Destination Cluster.

```
curl \
  -sk \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  --request POST \
  --data @vault2_wrapping_key.json \
  $VAULT_ADDR/v1/transit/keys/vault2_wrapping_key/import | \
  jq -r .
```

6. On the Source Cluster: Securely export the key you want to import into the Destination Cluster.

```
curl -sk -H "X-Vault-Token: $VAULT_TOKEN" --request GET $VAULT_ADDR/v1/transit/byok-export/vault2_wrapping_key/key_to_export | jq
```

7. On the Destination Cluster: Import the key exported on the Source Cluster.

```
vault write transit/keys/import-key/import ciphertext="<ciphertext>" type="<key type"
```

---
