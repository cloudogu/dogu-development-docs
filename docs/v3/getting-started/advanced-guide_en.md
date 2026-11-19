# Integrate XWiki into the CES

This guide extends [From a Helm chart to a Dogu V3](quickstart_en.md). It adds XWiki to the Warp menu, describes requirements for central CES authentication, and prepares the chart for distribution.

## Add XWiki to the Warp menu

A `WarpMenuEntry` adds XWiki to the central CES navigation. Create `warp-menu-entry.yaml` in the `templates` directory:

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: WarpMenuEntry
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: xwiki
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  category: Documentation
  displayName:
    de: XWiki
    en: XWiki
  path: /xwiki
```

Update the Dogu:

```shell
cldk dogu dev-push
```

Wait for the entry and check the result:

```shell
kubectl wait \
  --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}'=True \
  warpmenuentry/xwiki \
  --namespace ecosystem \
  --timeout 2m

kubectl get warpmenuentry xwiki --namespace ecosystem --output wide
```

Reload the CES page. XWiki now appears in the Warp menu.

## Authentication and permissions

An `AuthRegistration` registers XWiki with the CES identity provider (CAS).
XWiki also needs a suitable authenticator to use authentication through CAS.

Installing the authenticator and mapping LDAP groups to XWiki permissions are not part of this guide yet. These steps depend on the application and must be defined before publishing the Dogu.

It must be ensured that the admin group configured in the global config is synchronized into the Dogu and receives admin permissions there. The Dogu must also react to changes to the admin group and synchronize them at the latest after a restart.

Further information:

- [Multinode environment and Dogu V3 resources](../concepts/multinode-environment_en.md)
- [XWiki OpenID Connect Authenticator](https://github.com/xwiki-contrib/oidc/tree/master/oidc-authenticator)
- [CAS as an OpenID Connect provider](https://apereo.github.io/cas/development/authentication/OIDC-Authentication.html)

In principle, the `AuthRegistration` for XWiki could look as follows:

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: AuthRegistration
metadata:
  name: {{ .Release.Name }}
spec:
  consumer: {{ .Release.Name }}
  protocol: OIDC
  secretRef: xwiki-oidc-credentials
```

The Auth Registration Operator registers XWiki with CAS. It then writes the client ID, client secret, and issuer URL to the `xwiki-oidc-credentials` Secret.

## Prepare images for later mirroring

Because the EcoSystem must also be able to run in air-gapped environments, image references and their use in `values.yaml` must be maintained in `mirror-patch-tpl.yaml`. These can then be mirrored, and the image references in `values.yaml` can be adjusted.

Image mirroring happens before installation. An existing `runtime-patch-tpl.yaml` remains unchanged; it is evaluated later by the Dogu Operator.

The chart uses an XWiki image and a MySQL image. The MySQL image also runs in the XWiki init container that waits for the database. The XWiki chart builds its address without a registry, so the patch sets the full address.
Therefore, `mirror-patch-tpl.yaml` looks as follows and is placed in the same directory as `Chart.yaml`:

```yaml
apiVersion: v1
values:
  # All container images used by the chart
  images:
    xwiki: "docker.io/library/xwiki:18.6.0-mysql-tomcat"
    mysql: "docker.io/bitnamilegacy/mysql:8.4.4-debian-12-r7"

patches:
  # Adjust the image addresses in values.yaml for the target registry
  values.yaml:
    xwiki:
      image:
        # Use the registry and repository of the XWiki image
        name: "{{ registryFrom .images.xwiki }}/{{ repositoryFrom .images.xwiki }}"
        tag: "{{ tagFrom .images.xwiki }}"
      mysql:
        image:
          # The MySQL chart stores the registry and repository separately
          registry: "{{ registryFrom .images.mysql }}"
          repository: "{{ repositoryFrom .images.mysql }}"
          tag: "{{ tagFrom .images.mysql }}"
      initContainers:
        database:
          # Full address for the init container that waits for the database
          image: "{{ .images.mysql }}"
```

The patches to the `values.yaml` are specified as Go templates.
The following additional functions are available for processing image references:

- `registryFrom` returns the registry, for example `docker.io`.
- `repositoryFrom` returns the repository, for example `library/xwiki`.
- `tagFrom` returns the tag, for example `18.6.0-mysql-tomcat`.

## Inject CES configuration with runtime patches

In addition to mirror patches for image addresses, runtime patches provide access to CES global configuration.

Runtime patches let the Dogu Operator insert CES global configuration into chart values without changing upstream chart templates.

Processing happens in three steps:

1. The Operator reads CES global configuration, evaluates the runtime patches and merges the value sources.
2. The values are checked against the chart schema, if present and schema validation is enabled. The chart is then rendered for a preflight check.
3. If the checks pass, the values are written to a HelmRelease. The Helm Controller uses Helm to install or update the chart in the cluster.

Place `runtime-patch-tpl.yaml` next to `Chart.yaml`. It directly contains the desired chart values.

`globalConfig` is a template function. It reads values from the `global-config` ConfigMap in the Dogu's namespace and inserts them into the runtime values.

### Add a runtime patch

Adapt the keys to your chart's `values.yaml`. Place subchart values under its name or alias (`backend` here). Nested maps and lists are supported.

```yaml
application:
  host: '{{ globalConfig "fqdn" }}'
  certificate: '{{ globalConfig "certificate/server.crt" }}'
backend:
  applicationHosts:
    - '{{ globalConfig "fqdn" }}'
```

The file must be valid YAML before the Operator evaluates it. Quote template expressions as shown above. The Operator evaluates only string values, including those in lists. Keys and structure remain unchanged.

The file must contain exactly one YAML document with a map at its root. If no runtime values are needed, omit the file. A file containing `{}` is also valid and represents an empty map. An empty file, a whitespace-only file or a file containing `null` is not allowed.

### Read configuration

- `/` separates path segments; dots remain part of a key. For example, `certificate/server.crt` reads the certificate at that path.
- The requested value must be a string; missing keys or other data types cause an error. An empty string is allowed.
- Inserted values remain strings. Quotes and line breaks, such as those in certificates, are preserved.

`globalConfig` is the only additional template function provided.

### Preserve templates for the application

The Operator evaluates `{{ ... }}` even in multiline strings (`|`). To leave an expression for the application to evaluate, output it as a string:

```yaml
logConfiguration: |
  date: {{ "{{ .Date }}" }}
  line: {{ "{{ .LogLineJSON }}" }}
```

The application receives `date: {{ .Date }}` and `line: {{ .LogLineJSON }}`. The Operator does not evaluate the result again. Quotes or `|` alone do not prevent template evaluation. Static application templates that need no CES configuration can stay in `values.yaml`.

### Precedence and configuration changes

Runtime values override the chart defaults and `global.cesConfig`. They are in turn overridden by `spec.values` and `spec.mappedValues` of the Dogu CR.

When `global-config` changes, the operator re-evaluates the runtime patches and rolls out the new values without restarting Pods. If a workload should restart on changed ConfigMaps or Secrets, it can be annotated, for example with `k8s.cloudogu.com/reload-auto: "true"`.

The complete precedence, the merging of values, and the behavior on configuration changes are described in [Configuring v3 dogus](https://github.com/cloudogu/k8s-dogu-operator/blob/develop/docs/operations/v3/configuring_v3_dogus_en.md).

### Test runtime patches

Only the Dogu Operator evaluates runtime patches; `helm template` and `helm install` do not. To test them, deploy the Dogu to the local environment with `cldk dogu dev-push` and check the values the operator wrote into the HelmRelease:

```shell
cldk dogu dev-push
helm get values xwiki --namespace ecosystem
```

## Next step

The chart is now ready for deployment. [Release a Dogu V3](../guides/release_en.md) explains how to request registry access, publish the Dogu with `cldk dogu release`, and verify the release.
