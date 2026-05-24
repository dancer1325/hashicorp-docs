# TODO:
TODO:

# How-To
## install Vault?
### ways to customize
#### `helm install vault hashicorp/vault --values <PATH_TO_YAML>`
* `helm install vault hashicorp/vault --values override-values.yml`
#### `helm install vault hashicorp/vault --set "<KEY>=<VALUE>"`
* `helm install vault hashicorp/vault --set "server.dev.enabled=true"`
### Dev mode
TODO:
### TODO:

## View the Vault UI
### ❌NOT exposed -- as -- service❌
TODO:
### ways to expose it
#### -- via -- port-forwarding
* `kubectl port-forward vault-0 8200:8200`
* http://localhost:8200
#### -- via -- `ui` configuration value
TODO:

## Initialize & unseal Vault

* `kubectl get pods -l app.kubernetes.io/name=vault`
  * 's return: 0 READY pods
    * Reason:🧠they are NOT initialized🧠
* `kubectl exec -ti vault-0 -- vault operator init`
  * 's return: unseal keys + rook token
    * ⚠️copy ALL of them⚠️
* unseal the Vault server

    ```sh
    ## Unseal the first vault server until it reaches the key threshold
    $ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 1
    $ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 2
    $ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 3
    ```
* `kubectl get pods -l app.kubernetes.io/name=vault`
  * 's return: Vault server READY

# TODO:
TODO:


