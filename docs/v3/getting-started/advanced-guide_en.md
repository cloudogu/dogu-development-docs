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

Update the chart:

```shell
helm upgrade --install xwiki . \
  --namespace ecosystem \
  --wait \
  --timeout 15m
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

Because the EcoSystem must also be able to run in air-gapped environments, image references and their use in `values.yaml` must be maintained in `chart-patch-tpl.yaml`. These can then be mirrored, and the image references in `values.yaml` can be adjusted.

The chart uses an XWiki image and a MySQL image.
Therefore, `chart-patch-tpl.yaml` looks as follows and is placed in the same directory as `Chart.yaml`:

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
```

The patches to the `values.yaml` are specified as Go templates.
The following additional functions are available for processing image references:

- `registryFrom` returns the registry, for example `docker.io`.
- `repositoryFrom` returns the repository, for example `library/xwiki`.
- `tagFrom` returns the tag, for example `18.6.0-mysql-tomcat`.

## Push the chart to the local registry

The local registry was set up together with the quickstart environment. It is available at `localhost:5001`.

First, download the dependencies and package the chart:

```shell
helm dependency build .
helm lint .

mkdir -p dist
helm package . --destination dist
```

Push the generated package to the local OCI registry:

```shell
helm push dist/xwiki-1.7.1-1.tgz \
  oci://localhost:5001/quickstart/charts \
  --plain-http
```

The chart can now also be installed directly from the local registry:

```shell
helm upgrade --install xwiki \
  oci://localhost:5001/quickstart/charts/xwiki \
  --version 1.7.1-1 \
  --plain-http \
  --namespace ecosystem \
  --wait \
  --timeout 15m
```

## Next step

The chart is now ready for distribution. [Release a Dogu V3](../guides/release_en.md) explains how to request registry access, publish the chart and verify the release.
