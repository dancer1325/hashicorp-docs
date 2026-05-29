---
layout: docs
page_title: Vault Agent Injector
description: >-
  Learn how Vault Agent Injector works. It is a Kubernetes admission webhook that adds Vault Agent containers to pods for consuming Vault secrets.
# START AUTO GENERATED METADATA, DO NOT EDIT
created_at: 2026-04-15T00:15:11.000Z
last_modified: 2026-04-15T00:15:11.000Z
# END AUTO GENERATED METADATA
---

# Vault Agent Injector OR vault-k8s

* Vault Agent Injector/Vault k8s
  * [source code](https://github.com/hashicorp/vault-k8s)
  * == [Kubernetes Mutation Webhook Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) /
    * ⚠️requirements⚠️
      * the pod contains the annotation `vault.hashicorp.com/agent-inject: true`
    * how does it work?
      * intercept pod events
      * make -- , by including Vault Agent containers, -- changes | pod specifications 
      * render Vault secrets -- , via [Vault Agent Templates](../../../agent-and-proxy/agent/template.mdx), -- | shared memory volume (emptyDir in RAM, scoped to Pod)

```mermaid
graph LR
    subgraph infra-cluster["Cluster: Infra"]
        Vault[Vault Server]
    end
    subgraph app-cluster["Cluster: Application"]
        subgraph Pod
            VA[Vault Agent Sidecar] -->|writes secrets| Vol["/vault/secrets (shared volume)"]
            App[Application Container] -->|reads secrets| Vol
        end
    end
    Vault -->|authenticates & fetches secrets| VA
```

## Overview

TODO: 
The Vault Agent Injector works by intercepting pod `CREATE` and `UPDATE`
events in Kubernetes. The controller parses the event and looks for the metadata
annotation `vault.hashicorp.com/agent-inject: true`. If found, the controller will
alter the pod specification based on other annotations present.

### Mutations

At a minimum, every container in the pod will be configured to mount a shared
memory volume. This volume is mounted to `/vault/secrets` and will be used by the Vault
Agent containers for sharing secrets with the other containers in the pod.

Next, two types of Vault Agent containers can be injected: init and sidecar. The
init container will prepopulate the shared memory volume with the requested
secrets prior to the other containers starting. The sidecar container will
continue to authenticate and render secrets to the same location as the pod runs.
Using annotations, the initialization and sidecar containers may be disabled.

Last, two additional types of volumes can be optionally mounted to the Vault Agent
containers. The first is secret volume containing TLS requirements such as client
and CA (certificate authority) certificates and keys. This volume is useful when
communicating and verifying the Vault server's authenticity using TLS. The second
is a configuration map containing Vault Agent configuration files. This volume is
useful to customize Vault Agent beyond what the provided annotations offer.

### Authenticating with Vault

The primary method of authentication with Vault when using the Vault Agent Injector
is the service account attached to the pod. Other authentication methods can be configured
using annotations.

For Kubernetes authentication, the service account must be bound to a Vault role and a
policy granting access to the secrets desired.

A service account must be present to use the Vault Agent Injector with the Kubernetes
authentication method. It is _not_ recommended to bind Vault roles to the default service
account provided to pods if no service account is defined.

### Requesting secrets

There are two methods of configuring the Vault Agent containers to render secrets:

- the `vault.hashicorp.com/agent-inject-secret` annotation, or
- a configuration map containing Vault Agent configuration files.

Only one of these methods may be used at any time.

#### Secrets via annotations

To configure secret injection using annotations, the user must supply:

- one or more _secret_ annotations, and
- the Vault role used to access those secrets.

The annotation must have the format:

```yaml
vault.hashicorp.com/agent-inject-secret-<unique-name>: /path/to/secret
```

The unique name will be the filename of the rendered secret and must be unique if
multiple secrets are defined by the user. For example, consider the following
secret annotations:

```yaml
vault.hashicorp.com/agent-inject-secret-foo: database/roles/app
vault.hashicorp.com/agent-inject-secret-bar: consul/creds/app
vault.hashicorp.com/role: 'app'
```

The first annotation will be rendered to `/vault/secrets/foo` and the second
annotation will be rendered to `/vault/secrets/bar`.

It's possible to set the file format of the rendered secret using the annotation. For example the
following secret will be rendered to `/vault/secrets/foo.txt`:

```yaml
vault.hashicorp.com/agent-inject-secret-foo.txt: database/roles/app
vault.hashicorp.com/role: 'app'
```

The secret unique name must consist of alphanumeric characters, `.`, `_` or `-`.

##### Secret templates

~> Vault Agent uses the Consul Template project to render secrets. For more information
on writing templates, see the [Consul Template documentation](https://github.com/hashicorp/consul-template).

How the secret is rendered to the file is also configurable. To configure the template
used, the user must supply a _template_ annotation using the same unique name of
the secret. The annotation must have the following format:

```yaml
vault.hashicorp.com/agent-inject-template-<unique-name>: |
  <
    TEMPLATE
    HERE
  >
```

For example, consider the following:

```yaml
vault.hashicorp.com/agent-inject-secret-foo: 'database/creds/db-app'
vault.hashicorp.com/agent-inject-template-foo: |
  {{- with secret "database/creds/db-app" -}}
  postgres://{{ .Data.username }}:{{ .Data.password }}@postgres:5432/mydb?sslmode=disable
  {{- end }}
vault.hashicorp.com/role: 'app'
```

The rendered secret would look like this within the container:

```shell-session
$ cat /vault/secrets/foo
postgres://v-kubernet-pg-app-q0Z7WPfVN:A1a-BUEuQR52oAqPrP1J@postgres:5432/mydb?sslmode=disable
```

~> The default left and right template delimiters are `{{` and `}}`.

If no template is provided the following generic template is used:

```
{{ with secret "/path/to/secret" }}
    {{ range $k, $v := .Data }}
        {{ $k }}: {{ $v }}
    {{ end }}
{{ end }}
```

For example, the following annotation will use the default template to render
PostgreSQL secrets found at the configured path:

```yaml
vault.hashicorp.com/agent-inject-secret-foo: 'database/roles/pg-app'
vault.hashicorp.com/role: 'app'
```

The rendered secret would look like this within the container:

```shell-session
$ cat /vault/secrets/foo
password: A1a-BUEuQR52oAqPrP1J
username: v-kubernet-pg-app-q0Z7WPfVNqqTJuoDqCTY-1576529094
```

~> Some secrets such as KV are stored in maps. Their data can be accessed using `.Data.data.<NAME>`

### Renewals and updating secrets

For more information on when Vault Agent fetches and renews secrets, see the
[Agent documentation](/vault/docs/agent-and-proxy/agent/template#renewals-and-updating-secrets).

### Vault agent configuration map

For advanced use cases, it may be required to define Vault Agent configuration
files to mount instead of using secret and template annotations. The Vault Agent
Injector supports mounting ConfigMaps by specifying the name using the `vault.hashicorp.com/agent-configmap`
annotation. The configuration files will be mounted to `/vault/configs`.

The configuration map must contain either one or both of the following files:

- **config-init.hcl** used by the init container. This must have `exit_after_auth` set to `true`.
- **config.hcl** used by the sidecar container. This must have `exit_after_auth` set to `false`.

An example of mounting a Vault Agent configmap [can be found here](/vault/docs/platform/k8s/injector/examples#configmap-example).

### Injector telemetry

<Tip>

Set [`injector.metrics.enabled`](/vault/docs/platform/k8s/helm/configuration#metrics)
to `true` in the Helm chart to start collecting injector metrics.

</Tip>

Vault Agent injector collects the following Prometheus metrics in addition to
the default set of `golang` metrics:

- `vault_agent_injector_request_queue_length` - The number of pending webhook requests for the injector.

- `vault_agent_injector_request_processing_duration_ms` - A histogram of webhook
  request processing times in milliseconds.

- `vault_agent_injector_injections_by_namespace_total` - The total count of
  Agent container injections, grouped by Kubernetes `namespace` and `injection_type`.
  Vault Agent injector counts the following injection types:
    - `init_only`
    - `sidecar_only`
    - `init_and_sidecar`

- `vault_agent_injector_failed_injections_by_namespace_total` - The total count
  of failed Agent sidecar injections, grouped by Kubernetes `namespace`.

## Tutorial

* [how to inject secrets | Kubernetes pods, -- via -- Vault Helm Sidecar](https://github.com/dancer1325/hashicorp-learn-vault-kubernetes-sidecar)
