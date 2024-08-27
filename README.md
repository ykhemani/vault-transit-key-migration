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

Reference: [https://developer.hashicorp.com/vault/api-docs/secret/transit#create-key](https://developer.hashicorp.com/vault/api-docs/secret/transit#create-key)

2. On the Destination Cluster: Enable the Transit secret engine if it isn't already enabled.

```
vault secrets enable transit
```

Reference: [https://developer.hashicorp.com/vault/docs/secrets/transit#setup](https://developer.hashicorp.com/vault/docs/secrets/transit#setup)

3. On the Destination Cluster: Export the wrapping key.

```
vault read \
  -format=json \
  transit/wrapping_key
```

Example response:
```
{
  "request_id": "352fa329-95c5-d401-f92f-b586724a81ea",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "public_key": "-----BEGIN PUBLIC KEY-----\nMIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA0m/bhuk3cTsbrodaQwbj\n7ulSTRXs6pzGzeTqZ2md+hDiOxGn/7ToxLvIj6nAnIvsS4HYBIeKjPLiM7btgF8j\ni391xatI0/tdTTybinnpR2AaweTcQnXiN6HlipDouEk/308E6+Ras6J18SaAGArs\nI1Iwu8Dzl5yig2BpLtgC9sa19tHolmpeqo5N6K8JClCUxKDi2gQYtNWDtfQwoX8s\nctAG5/jTnzQEYxAOkVaCvEWeCwh+IzRFpCNnTHHXYu9apKN9rL5snWKUWHz4neys\nGwM+6ZumE/y0mg5gY9AOM6AwqYlio2KFXR5AsjzcvFPzTMP2ePqsWRC+VTodlz0X\njT3hoQtDGWnL2lWNyFG5vokRW+SafP3rhU13No/guSfZmOGkFDI1uJ1mBPLA4UiT\nKt+TpJfG3Y/0JbgnjXfPP99vjeQ2OXJ15Lw6uqsR9oowJpRdEBBJRqvf1dIe92LK\nWgKj1nTzurxuLLk4wNYceW78pDrX4R7cCmBQ8QJb0ZwGtTJ0/vIYBJcKa1J76t6s\nbQNjZ2SWa3zdBXboK8VDafJo5A+7yPoVwZfhu2nFPUIOWUcqPS13FI6Q5LwbtU/f\n07Nw/tzTqEqlwtlAbD7bTHQxcZ7vc29oAb0PfaEbBxUwLZtGF9Yr/uVQZm9Ciirc\nh5hrDKqYGo3cpdteyfQW/TkCAwEAAQ==\n-----END PUBLIC KEY-----\n"
  },
  "warnings": null,
  "mount_type": "transit"
}
```

[https://developer.hashicorp.com/vault/docs/secrets/transit/key-wrapping-guide#retrieve-the-transit-wrapping-key](https://developer.hashicorp.com/vault/docs/secrets/transit/key-wrapping-guide#retrieve-the-transit-wrapping-key)

4. On the Source Cluster: Create the JSON payload for importing the wrapping key exported from the Destination Cluster (vault2_wrapping_key.json)
```
{
  "type" : "rsa-4096",
  "public_key" : "<public_key from the Destination Cluster>"
}
```

For example:
```
{
  "type" : "rsa-4096",
  "public_key" : "-----BEGIN PUBLIC KEY-----\nMIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA0m/bhuk3cTsbrodaQwbj\n7ulSTRXs6pzGzeTqZ2md+hDiOxGn/7ToxLvIj6nAnIvsS4HYBIeKjPLiM7btgF8j\ni391xatI0/tdTTybinnpR2AaweTcQnXiN6HlipDouEk/308E6+Ras6J18SaAGArs\nI1Iwu8Dzl5yig2BpLtgC9sa19tHolmpeqo5N6K8JClCUxKDi2gQYtNWDtfQwoX8s\nctAG5/jTnzQEYxAOkVaCvEWeCwh+IzRFpCNnTHHXYu9apKN9rL5snWKUWHz4neys\nGwM+6ZumE/y0mg5gY9AOM6AwqYlio2KFXR5AsjzcvFPzTMP2ePqsWRC+VTodlz0X\njT3hoQtDGWnL2lWNyFG5vokRW+SafP3rhU13No/guSfZmOGkFDI1uJ1mBPLA4UiT\nKt+TpJfG3Y/0JbgnjXfPP99vjeQ2OXJ15Lw6uqsR9oowJpRdEBBJRqvf1dIe92LK\nWgKj1nTzurxuLLk4wNYceW78pDrX4R7cCmBQ8QJb0ZwGtTJ0/vIYBJcKa1J76t6s\nbQNjZ2SWa3zdBXboK8VDafJo5A+7yPoVwZfhu2nFPUIOWUcqPS13FI6Q5LwbtU/f\n07Nw/tzTqEqlwtlAbD7bTHQxcZ7vc29oAb0PfaEbBxUwLZtGF9Yr/uVQZm9Ciirc\nh5hrDKqYGo3cpdteyfQW/TkCAwEAAQ==\n-----END PUBLIC KEY-----\n"
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

Reference: [https://developer.hashicorp.com/vault/api-docs/secret/transit#import-key](https://developer.hashicorp.com/vault/api-docs/secret/transit#import-key)

6. On the Source Cluster: Securely export the key you want to import into the Destination Cluster.

```
curl -sk -H "X-Vault-Token: $VAULT_TOKEN" --request GET $VAULT_ADDR/v1/transit/byok-export/vault2_wrapping_key/key_to_export | jq
```

Example resposne:
```
{
  "request_id": "e4f9e716-5196-8236-48cd-7cd2a1327159",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "keys": {
      "1": "QC8gE/ou+IRFmH3/d76bNJSmof7rNjveRc+j9KulGv/EcIrNeLZOUAenvX2eEIJmYEoLyGrImOm8gTZDyB6GTdWxLMR4uzRDcmbPoBX8tTR8U+x1ZOYTYkUmaBr8/9bIX36qSV0u96Cae/jjswhtXpt6CwXp7Yy30J7DTCmiyhAqifTU9BlsOKrqkaNBfhkX+zGXAFWC+K5mnAo7y1pwR/yy/kEfYLTh7YSM8RnFTa0Yw2549pJjEZcy0qdW8oHRf5W4f57tvw6e4ueqffmAAQzx/S7Va37rWNPWpJz+cVJOKY1S2n14SxrZnjJL5w1sLyqshtCXBxk1tyEg3A80rq7qrgQy8oTno9L9v96c8yGGw4FlLsG+NKVsCiHtTj14gHzZB5Ybf1vvrx1jq/EJWptSejY4td/ObBdgj2d1izltXUze0saxxkQMchcsPCTv+FY7yQ6f8bKT/mJPTqd4S83Ut5YxQ9FUQUkbK8iSkjHjhzWwKe2BSTX/glWEdkSlDr5qdHpIKU0jxXet7nyS1/CpWjid3k6encr6FgiAjo2OytnFvf6Og3KmHj/dW4TwszDWHidqXkV630ivWySRMdbAIHUrWgQAiJU08CTIS+TKgfJc2FEkHripFKx3n8PLI1Wq3HOmRLRKQD4Kik314nCOqWAWZMcVMKr7Befu2sAYh5Q8spJ8zsj8EL3fej8iSK5WUFf7oWWREfzXvwzwaVUq0cSrp0dJ"
    },
    "name": "key_to_export",
    "type": "aes256-gcm96"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null,
  "mount_type": "transit"
}
```

Reference: [https://developer.hashicorp.com/vault/api-docs/secret/transit#securely-export-key](https://developer.hashicorp.com/vault/api-docs/secret/transit#securely-export-key)

7. On the Destination Cluster: Import the key exported on the Source Cluster.

```
vault write transit/keys/import-key/import ciphertext="<ciphertext>" type="<key type>"
```

For example: 
```
vault write transit/keys/import-key/import ciphertext="QC8gE/ou+IRFmH3/d76bNJSmof7rNjveRc+j9KulGv/EcIrNeLZOUAenvX2eEIJmYEoLyGrImOm8gTZDyB6GTdWxLMR4uzRDcmbPoBX8tTR8U+x1ZOYTYkUmaBr8/9bIX36qSV0u96Cae/jjswhtXpt6CwXp7Yy30J7DTCmiyhAqifTU9BlsOKrqkaNBfhkX+zGXAFWC+K5mnAo7y1pwR/yy/kEfYLTh7YSM8RnFTa0Yw2549pJjEZcy0qdW8oHRf5W4f57tvw6e4ueqffmAAQzx/S7Va37rWNPWpJz+cVJOKY1S2n14SxrZnjJL5w1sLyqshtCXBxk1tyEg3A80rq7qrgQy8oTno9L9v96c8yGGw4FlLsG+NKVsCiHtTj14gHzZB5Ybf1vvrx1jq/EJWptSejY4td/ObBdgj2d1izltXUze0saxxkQMchcsPCTv+FY7yQ6f8bKT/mJPTqd4S83Ut5YxQ9FUQUkbK8iSkjHjhzWwKe2BSTX/glWEdkSlDr5qdHpIKU0jxXet7nyS1/CpWjid3k6encr6FgiAjo2OytnFvf6Og3KmHj/dW4TwszDWHidqXkV630ivWySRMdbAIHUrWgQAiJU08CTIS+TKgfJc2FEkHripFKx3n8PLI1Wq3HOmRLRKQD4Kik314nCOqWAWZMcVMKr7Befu2sAYh5Q8spJ8zsj8EL3fej8iSK5WUFf7oWWREfzXvwzwaVUq0cSrp0dJ" type="aes256-gcm96"
Success! Data written to: transit/keys/import-key/import
```

Reference: [https://developer.hashicorp.com/vault/docs/commands/transit/import](https://developer.hashicorp.com/vault/docs/commands/transit/import)

---
