---
layout: docs
page_title: Run Vault on Kubernetes
description: >-
  Run Vault directly on Kubernetes in various configurations. For
  pure-Kubernetes workloads, this enables Vault to also exist purely within
  Kubernetes.
# START AUTO GENERATED METADATA, DO NOT EDIT
created_at: 2026-04-15T00:15:10.000Z
last_modified: 2026-04-15T00:15:10.000Z
# END AUTO GENERATED METADATA
---

# Run Vault | Kubernetes

* Vault modes / 
  * work with Kubernetes
    * `dev`
    * `standalone`
    * `ha`
    * `external`

## How-To

### install Vault?

* requirements
  * install Helm

* steps
  * [add Helm Hashicorp repository](/content/vault/v2.x/content/partials/helm/repo.md)
  * [install Vault Helm chart](/content/vault/v2.x/content/partials/helm/install.md)

* ways to customize
  * `helm install vault hashicorp/vault --values <PATH_TO_YAML>`
  * `helm install vault hashicorp/vault --set "<KEY>=<VALUE>"`

#### Dev mode

* == run a Vault server in development
* == 1! Vault server / memory storage backend

* use cases
  * learn 
* ❌NOT use case❌
  * production environment

* set `server.dev.enabled=true`

  ```shell-session
  $ helm install vault hashicorp/vault \
      --set "server.dev.enabled=true"
  ```

#### Standalone mode

* default installation
* == 1! Vault server + file storage backend
* ❌| production, NOT recommended❌
  * Reason:🧠
    * less secure
    * less resilient🧠

#### HA mode

TODO:
The Helm chart may be run in high availability (HA) mode. This installs three
Vault servers with an existing Consul storage backend. It is suggested that
Consul is installed via the [Consul Helm
chart](https://github.com/hashicorp/consul-k8s).

Install the latest Vault Helm chart in HA mode.

```shell-session
$ helm install vault hashicorp/vault \
    --set "server.ha.enabled=true"
```

Refer to the [Vault Installation to Minikube via
Helm](/vault/tutorials/kubernetes/kubernetes-minikube-consul) tutorial
to learn how to set up Consul and Vault in HA mode.

#### External mode

The Helm chart may be run in external mode. This installs no Vault server and
relies on a network addressable Vault server to exist.

Install the latest Vault Helm chart in external mode.

```shell-session
$ helm install vault hashicorp/vault \
    --set "injector.externalVaultAddr=http://external-vault:8200"
```

Refer to the [Integrate a Kubernetes Cluster with an
External Vault](/vault/tutorials/kubernetes/kubernetes-external-vault)
tutorial to learn how to use an external Vault within a Kubernetes cluster.

### View the Vault UI

* enabled
* ❌NOT exposed -- as -- service❌
  * Reason:🧠security reasons🧠
* ways to expose it
  * -- via -- port-forwarding
  * -- via -- [`ui` configuration value](configuration.md)

### Initialize & unseal Vault

* [initialize a Vault server](/content/vault/v2.x/content/docs/commands/operator/init.md)
* [unseal](/content/vault/v2.x/content/docs/concepts/seal.mdx)

#### Google KMS auto unseal

TODO:
The Helm chart may be run with [Google KMS for Auto
Unseal](/vault/docs/configuration/seal/gcpckms). This enables Vault server pods to
auto unseal if they are rescheduled.

Vault Helm requires the Google Cloud KMS credentials stored in
`credentials.json` and mounted as a secret in each Vault server pod.

##### Create the secret

First, create the secret in Kubernetes:

```bash
kubectl create secret generic kms-creds --from-file=credentials.json
```

Vault Helm mounts this to `/vault/userconfig/kms-creds/credentials.json`.

##### Config example

This is a Vault Helm configuration that uses Google KMS:

```yaml
global:
  enabled: true

server:
  extraEnvironmentVars:
    GOOGLE_REGION: global
    GOOGLE_PROJECT: <PROJECT NAME>
    GOOGLE_APPLICATION_CREDENTIALS: /vault/userconfig/kms-creds/credentials.json

  volumes:
    - name: userconfig-kms-creds
      secret:
        defaultMode: 420
        secretName: kms-creds

  volumeMounts:
    - mountPath: /vault/userconfig/kms-creds
      name: userconfig-kms-creds
      readOnly: true

  ha:
    enabled: true
    replicas: 3

    config: |
      ui = true

      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
      }

      seal "gcpckms" {
        project     = "<NAME OF PROJECT>"
        region      = "global"
        key_ring    = "<NAME OF KEYRING>"
        crypto_key  = "<NAME OF KEY>"
      }

      storage "consul" {
        path = "vault"
        address = "HOST_IP:8500"
      }
```

#### Amazon KMS auto unseal

The Helm chart may be run with [AWS KMS for Auto
Unseal](/vault/docs/configuration/seal/awskms). This enables Vault server pods to auto
unseal if they are rescheduled.

Vault Helm requires the AWS credentials stored as environment variables that
are defined in each Vault server pod.

##### Create the secret

First, create a secret with your KMS access key/secret:

```shell-session
$ kubectl create secret generic kms-creds \
    --from-literal=AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID?}" \
    --from-literal=AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY?}"
```

##### Config example

This is a Vault Helm configuration that uses AWS KMS:

```yaml
global:
  enabled: true

server:
  extraSecretEnvironmentVars:
    - envName: AWS_ACCESS_KEY_ID
      secretName: kms-creds
      secretKey: AWS_ACCESS_KEY_ID
    - envName: AWS_SECRET_ACCESS_KEY
      secretName: kms-creds
      secretKey: AWS_SECRET_ACCESS_KEY

  ha:
    enabled: true
    config: |
      ui = true

      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
      }

      seal "awskms" {
        region     = "KMS_REGION_HERE"
        kms_key_id = "KMS_KEY_ID_HERE"
      }

      storage "consul" {
        address = "HOST_IP:8500"
        path = "vault/"
      }
```

### Probes

Probes are essential for detecting failures, rescheduling and using pods in
Kubernetes. The helm chart offers configurable readiness and liveliness probes
which can be customized for a variety of use cases.

Vault's [/sys/health`](/vault/api-docs/system/health) endpoint can be customized to
change the behavior of the health check. For example, we can change the Vault
readiness probe to show the Vault pods are ready even if they're still uninitialized
and sealed using the following probe:

```yaml
server:
  readinessProbe:
    enabled: true
    path: '/v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204'
```

Using this customized probe, a `postStart` script could automatically run once the
pod is ready for additional setup.

### Upgrading Vault | kubernetes

To upgrade Vault on Kubernetes, we follow the same pattern as
[generally upgrading Vault](/vault/docs/upgrading), except we can use
the Helm chart to update the Vault server StatefulSet. It is important to understand
how to [generally upgrade Vault](/vault/docs/upgrading) before reading this
section.

The Vault StatefulSet uses `OnDelete` update strategy. It is critical to use `OnDelete` instead
of `RollingUpdate` because standbys must be updated before the active primary. A
failover to an older version of Vault must always be avoided.

!> **IMPORTANT NOTE:** Always back up your data before upgrading! Vault does not
make backward-compatibility guarantees for its data store. Simply replacing the
newly-installed Vault binary with the previous version may not cleanly
downgrade Vault, as upgrades may perform changes to the underlying data
structure that make the data incompatible with a downgrade. If you need to roll
back to a previous version of Vault, you should roll back your data store as
well.

#### Upgrading Vault servers

!> **IMPORTANT NOTE:** Helm will install the latest chart found in a repo by default.
It's recommended to specify the chart version when upgrading.

To initiate the upgrade, set the `server.image` values to the desired Vault
version, either in a values yaml file or on the command line. For illustrative
purposes, the example below uses `vault:123.456`.

```yaml
server:
  image:
    repository: 'vault'
    tag: '123.456'
```

Next, list the Helm versions and choose the desired version to install.

```bash
$ helm search repo hashicorp/vault
NAME           	CHART VERSION	APP VERSION	DESCRIPTION
hashicorp/vault	0.32.0       	1.21.2     	Official HashiCorp Vault Chart
```

* `helm upgrade vault hashicorp/vault ... --dry-run`
  * TODO: 
  * Reason of `--dry-run`:🧠
    * Helm chart is NEW & under significant development
    * verify changes 🧠
  * _Example:_

    ```shell-session
    $ helm upgrade vault hashicorp/vault --version=0.32.0 \
        --set='server.image.repository=vault' \
        --set='server.image.tag=123.456' \
        --dry-run
    ```

This should cause no changes (although the resources are updated). If
everything is stable, `helm upgrade` can be run.

The `helm upgrade` command should have updated the StatefulSet template for
the Vault servers, however, no pods have been deleted. The pods must be manually
deleted to upgrade. Deleting the pods does not delete any persisted data.

If Vault is not deployed using `ha` mode, the single Vault server may be deleted by
running:

```shell-session
$ kubectl delete pod <name of Vault pod>
```


If you deployed Vault in high availability (`ha`) mode, you must upgrade your
standby pods before upgrading the active pod:

1. Before deleting the standby pod, remove the associated node from the raft
   with `vault operator raft remove-peer <server_id>`.
1. Confirm Vault removed the node successfully from Raft with
   `vault operator raft list-peers`.
1. Once you confirm the removal, delete the pod.

<Warning title="Delete nodes to avoid unnecessary leader elections">

Removing a pod without first deleting the node from its cluster means that
Raft will not be aware of the correct number of nodes in the cluster. Not knowing
the correct number of nodes can trigger a leader election, which can potentially
cause unneeded downtime.

</Warning>

Vault has K8s service discovery built in (when enabled in the server configuration) and
will automatically change the labels of the pod with its current leader status. These labels
can be used to filter the pods.

For example, select all pods that are Vault standbys:

```shell-session
$ kubectl get pods -l vault-active=false
```

Select the active Vault pod:

```shell-session
$ kubectl get pods -l vault-active=true
```

Next, sequentially delete every pod that is not the active primary, ensuring the quorum is maintained at all times:

```shell-session
$ kubectl delete pod <name of Vault pod>
```

If auto-unseal is not being used, the newly scheduled Vault standby pods needs
to be unsealed:

```shell-session
$ kubectl exec -ti <name of pod> -- vault operator unseal
```

Finally, once the standby nodes have been updated and unsealed, delete the active
primary:

```shell-session
$ kubectl delete pod <name of Vault primary>
```

Similar to the standby nodes, the former primary also needs to be unsealed:

```shell-session
$ kubectl exec -ti <name of pod> -- vault operator unseal
```

After a few moments the Vault cluster should elect a new active primary. The Vault
cluster is now upgraded!

### Protecting sensitive Vault configurations

Vault Helm renders a Vault configuration file during installation and stores the
file in a Kubernetes configmap. Some configurations require sensitive data to be
included in the configuration file and would not be encrypted at rest once created
in Kubernetes.

The following example shows how to add extra configuration files to Vault Helm
to protect sensitive configurations from being in plaintext at rest using Kubernetes
secrets.

First, create a partial Vault configuration with the sensitive settings Vault
loads during startup:

```shell-session
$ cat <<EOF >>config.hcl
storage "mysql" {
username = "user1234"
password = "secret123!"
database = "vault"
}
EOF
```

Next, create a Kubernetes secret containing this partial configuration:

```shell-session
$ kubectl create secret generic vault-storage-config \
    --from-file=config.hcl
```

Finally, mount this secret as an extra volume and add an additional `-config` flag
to the Vault startup command:

```shell-session
$ helm install vault hashicorp/vault \
  --set='server.volumes[0].name=userconfig-vault-storage-config' \
  --set='server.volumes[0].secret.defaultMode=420' \
  --set='server.volumes[0].secret.secretName=vault-storage-config' \
  --set='server.volumeMounts[0].mountPath=/vault/userconfig/vault-storage-config' \
  --set='server.volumeMounts[0].name=userconfig-vault-storage-config' \
  --set='server.volumeMounts[0].readOnly=true' \
  --set='server.extraArgs=-config=/vault/userconfig/vault-storage-config/config.hcl'
```

## Architecture

We recommend running Vault on Kubernetes with the same
[general architecture](/vault/docs/internals/architecture)
as running it anywhere else. There are some benefits Kubernetes can provide
that eases operating a Vault cluster and we document those below. The standard
[production deployment](/vault/tutorials/operations/production-hardening) tutorial is still an
important read even if running Vault within Kubernetes.

### Production deployment checklist

_End-to-End TLS._ Vault should always be used with TLS in production. If
intermediate load balancers or reverse proxies are used to front Vault,
they should not terminate TLS. This way traffic is always encrypted in transit
to Vault and minimizes risks introduced by intermediate layers. See the
[official documentation](/vault/docs/platform/k8s/helm/examples/standalone-tls/)
for example on configuring Vault Helm to use TLS.

_Single Tenancy._ Vault should be the only main process running on a machine.
This reduces the risk that another process running on the same machine is
compromised and can interact with Vault. This can be accomplished by using Vault
Helm's `affinity` configurable. See the
[official documentation](/vault/docs/platform/k8s/helm/examples/ha-with-consul/)
for example on configuring Vault Helm to use affinity rules.

_Enable Auditing._ Vault supports several auditing backends. Enabling auditing
provides a history of all operations performed by Vault and provides a forensics
trail in the case of misuse or compromise. Audit logs securely hash any sensitive
data, but access should still be restricted to prevent any unintended disclosures.
Vault Helm includes a configurable `auditStorage` option that provisions a persistent
volume to store audit logs. See the
[official documentation](/vault/docs/platform/k8s/helm/examples/standalone-audit/)
for an example on configuring Vault Helm to use auditing.

_Immutable Upgrades._ Vault relies on an external storage backend for persistence,
and this decoupling allows the servers running Vault to be managed immutably.
When upgrading to new versions, new servers with the upgraded version of Vault
are brought online. They are attached to the same shared storage backend and
unsealed. Then the old servers are destroyed. This reduces the need for remote
access and upgrade orchestration which may introduce security gaps. See the
[upgrade section](#how-to) for instructions
on upgrading Vault on Kubernetes.

_Upgrade Frequently._ Vault is actively developed, and updating frequently is
important to incorporate security fixes and any changes in default settings such
as key lengths or cipher suites. Subscribe to the Vault mailing list and
GitHub CHANGELOG for updates.

_Restrict Storage Access._ Vault encrypts all data at rest, regardless of which
storage backend is used. Although the data is encrypted, an attacker with arbitrary
control can cause data corruption or loss by modifying or deleting keys. Access
to the storage backend should be restricted to only Vault to avoid unauthorized
access or operations.
