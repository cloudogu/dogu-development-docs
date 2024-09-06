# Relevant functionalities

This chapter describes the features and possible implementation ideas/solutions of those functionalities that make up a real Dogu in the first place and solve recurring problems.

Therefore, the following sections deal with recurring functionalities on how a Dogu can embed itself into the Cloudogu EcoSystem landscape.

## Authentication

Given a Dogu provides this, users can enjoy the benefits of enterprise-level single sign-on (SSO) and single log-out (SLO). Authentication is enabled by the Central Authentication Service (CAS). Dogu developers have several authentication options at their disposal. The following three sections deal with methods that have already been successfully used, namely:

- Authentication using CAS protocol
- Authentication using OAuth 2.0 protocol
- Authentication using OpenID Connect protocol

The basis for CAS is always a correct configuration of a user/group directory, be it the [LDAP](https://github.com/cloudogu/ldap) dogu or an external LDAP-compatible service.

### CAS protocol

Authentication within the Cloudogu EcoSystem is done using the [CAS protocol](https://apereo.github.io/cas/6.6.x/protocol/CAS-Protocol.html), which enables single sign-on and single log-out. Different CAS protocol versions are supported (2.0 and 3.0). When implementing using CAS protocol, it is recommended to use version 3.0 because (compared to version 2.0) it can return important user attributes after a successful authentication.

The following diagram shows the parties involved in the authentication configuration. Before a dogu (here using Redmine as an example) can participate in this process, the dogu must internally configure a set of CAS URLs:

- CAS log-in URL (for redirecting users to the web form).
- CAS validation URL (for validating service tickets)
- CAS log-out URL (to invalidate an SSO session)
- The service recognition in CAS against the Dogus happens automatically during the Dogu installation and will not be
  considered further in the following.

![Participants in the authentication configuration](../images/important/chapter3_auth_cas_config.png)

The actual authentication happens via a sequence of HTTP redirects and the exchange of session cookies in the background, of which users are unaware. The display of the CAS login form as an active login step stands out strongly.

The following figure provides a rough overview of the login process with the parties involved.

![CAS authentication workflow](../images/important/chapter3_auth_cas_sequencediag.png)

The SSO of the CAS significantly reduces this process when logging in to additional dogus.

For more information and a more detailed illustration before, during and after an authentication, see
the [CAS documentation](https://apereo.github.io/cas/6.6.x/protocol/CAS-Protocol.html).

**entry for a normal CAS client:**

```json
{
  "ServiceAccounts": [
    {
      "Type": "cas",
      "Params": [
        "cas"
      ]
    }
  ]
}
```

### OAuth protocol

CAS offers OAuth/OIDC as a protocol for authentication including SSO/SLO. The following describes the specification of OAuth 2.0 protocol in CAS.

#### Creating an OAuth Service Account for a dogu

In order for a Dogu to use the CAS's OAuth endpoints, it must log in to the CAS as a client. To do this, the OAuth-specific CAS service can be requested via the `dogu.json` of the respective dogu.

**entry for an OAuth client:**

```json
{
  "ServiceAccounts": [
    {
      "Type": "cas",
      "Params": [
        "oauth"
      ]
    }
  ]
}
```

The service account credentials are randomly generated (see [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) and stored encrypted in the registry under the path `/config/<dogu>/sa-cas/oauth` and `/config/<dogu>/sa-cas/oauth_client_secret`.

The credentials are composed of the `CLIENT_ID` and the `CLIENT_SECRET`. For the CAS, the `CLIENT_SECRET` is stored as a hash in the Cloudogu EcoSystem Registry under the path `/config/cas/service_accounts/oauth/<CLIENT_ID>/secret`.

#### OAuth endpoints and authentication flow

The following steps describe a successful OAuth authentication flow.

1. request a short term token: see section "OAuth authorize endpoint" below.
2. swap short-term token for long-term token: see section below "AccessToken endpoint".
3. long term token can now be used to authenticate against resources.
   Currently, CAS only offers the user's profile as a resource: See section [OAuth user profile](#oauth-user-profile).

![Authentication sequence with OAuth 2.0](../images/important/chapter3_auth_oauth_sequencediag.png)

Note: Resource is a part of CAS.

#### OAuth authorize endpoint

This endpoint serves as the initial start of OAuth authorization.
The `authorize` endpoint is used to request a short-lived token from the CAS.

**URL** : `<fqdn>/oauth2.0/authorize`

**Method** : `GET`

**Condition of the data**

```
response_type = code
client_id = Valid clientID from the dogu
state = Any string
redirect_url = <URL to which the short-term token will be redirected upon successful authentication>
```

**Data example**

```
response_type = code
client_id = portainer
state = b8c57125-9281-4b67-b857-1559cdfcdf31
redirect_url = http://local.cloudogu.com/portainer/
```

**call example**

```
https://local.cloudogu.com/cas/oauth2.0/authorize?client_id=portainer&redirect_uri=http%3A%2F%2Flocal.cloudogu.com%2Fportainer%2F&response_type=code&state=b8c57125-9281-4b67-b857-1559cdfcdf31
```

##### Successful response

Automatically redirects the user to the CAS login screen.
After successful login the `redirect_url` is passed as `code` GET parameter.

Example for `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

#### OAuth access token

This endpoint is used to exchange a short term token (`code`) for a long term token (`access_token`).

**URL** : `<fqdn>/oauth2.0/accessToken`

**Method** : `GET`

**Data constraints**

```
grant_type = authorization_code
code = Valid code from `authorize` endpoint
client_id = Valid clientID from the dogu
client_secret = Valid secret from the dogu
redirect_url = <URL to which the long term token will be sent upon successful authentication>
```

**Data example**

```
grant_type = authorization_code
code = ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local
client_id = portainer
client_secret = sPJtcNrmROZ3sZu3
redirect_url = https://local.cloudogu.com/portainer/
```

**Request example**

```
https://local.cloudogu.com/cas/oauth2.0/accessToken?grant_type=authorization_code&code=ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local&client_id=portainer&client_secret=sPJtcNrmROZ3sZu3&redirect_uri=https%3A%2F%2Flocal.cloudogu.com%2Fportainer%2F
```

##### Successful response

**Status:** 200 OK

**Example response:**

```json
{
    "access_token": "TGT-1-m2gUNJwEqXyV7aAEXekihcVnFc5iI4mpfdZGOTSiiHzEbwr1cr-cas.ces.local",
    "expires_in": "7196",
    "token_type": "Bearer"
}
```

##### Unsuccessful response

**Error:** The short-term token is invalid or has already expired.

**Status:** 500 Bad request

**Example response:**

```json
{
  "message": "invalid_grant"
}
```

#### OAuth user profile

This endpoint is used to retrieve the user profile from the logged in user using a long term token (`access_token`).

**URL** : `<fqdn>/oauth2.0/profile`

**Method** : `GET`

**Request header**

```
authorization = Valid Access Token as Bearer
```

**Request header example**

```
authorization: Bearer TGT-1-m2gUNJwEqXyV7aAEXekihcVnFc5iI4mpfdZGOTSiiHzEbwr1cr-cas.ces.local
```

##### Successful response

**Status:** 200 OK

**Example response:**

```json
{
  "id": "cesadmin",
  "attributes": {
    "username": "cesadmin",
    "cn": "admin",
    "mail": "cesadmin@localhost.de",
    "givenName": "admin",
    "surname": "admin",
    "displayName": "admin",
    "groups": [
      "cesManager",
      "cesadmin"
    ]
  }
}
```

##### Unsuccessful response

**Error:** The long-term token is invalid or has already expired.

**Status:** 500 Bad request

**Example response:**

```json
{
  "message": "expired_accessToken"
}
```

#### OAuth logout endpoint

This endpoint is used to terminate the OAuth session.
The logout endpoint is used to invalidate the long term token from CAS.

**URL** : `<fqdn>/cas/logout`

### OpenID Connect protocol

CAS provides OAuth/OpenID Connect (OIDC) as a protocol for authentication including SSO/SLO. The following describes the specification of the OpenID Connect protocol in CAS.

#### Creating an OIDC Service Account for a dogu

In order for a dogu to use the CAS's OIDC endpoints, it must log in to the CAS as a client. To do this, the request for an OIDC-specific CAS service account can be stored in the `dogu.json` of the dogu in question.

**entry for an OIDC client:**
```json
{
  "ServiceAccounts": [
    {
      "Type": "cas",
      "Params": [
        "oidc"
      ]
    }
  ]
}
```

The service account credentials are randomly generated (see [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) and stored encrypted in the registry under the path `/config/<dogu>/sa-cas/oidc` and `/config/<dogu>/sa-cas/oidc_client_secret`.

The credentials are composed of the `CLIENT_ID` and the `CLIENT_SECRET`. For the CAS, the `CLIENT_SECRET` is stored as a hash in the Cloudogu EcoSystem Registry under the path `/config/cas/service_accounts/oidc/<CLIENT_ID>/secret`.

#### OIDC-Authorize-Endpoint

This endpoint serves as the initial start of the OpenID Connect authorization.
The `authorize` endpoint is used to request a short-lived token from the CAS.

**URL** : `<fqdn>/oidc/authorize`

**Method** : `GET`

**Condition of the data**

```
response_type = code
client_id = Valid clientID from the dogu
state = Any string
redirect_url = <URL to which the short term token of successful authentication will be redirected>.
```

**Data example**

```
response_type = code
client_id = teamscale
state = b8c57125-9281-4b67-b857-1559cdfcdf31
redirect_url = http://local.cloudogu.com/teamscale/
```

**Request example**

```
https://local.cloudogu.com/cas/oidc/authorize?client_id=portainer&redirect_uri=http%3A%2F%2Flocal.cloudogu.com%2Fteamscale%2F&response_type=code&state=b8c57125-9281-4b67-b857-1559cdfcdf31
```

##### Successful response

Automatically redirects one to the CAS login screen.
After successful login the `redirect_url` is passed with a `code` as GET parameter.

Example for `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

#### OIDC access token

This endpoint is used to exchange a short term token (`code`) for a long term token (`access_token`).

**URL** : `<fqdn>/oidc/accessToken`

**Method** : `GET`

**Data constraints**

```
grant_type = authorization_code
code = Valid code from `authorize` endpoint
client_id = Valid clientID from the dogu
client_secret = Valid secret from the dogu
redirect_url = <URL to which the long term token of successful authentication is sent>
```

**Data example**

```
grant_type = authorization_code
code = ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local
client_id = teamscale
client_secret = sPJtcNrmROZ3sZu3
redirect_url = https://local.cloudogu.com/teamscale/
```

**Request example**

```
https://local.cloudogu.com/cas/oidc/accessToken?grant_type=authorization_code&code=ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local&client_id=teamscale&client_secret=sPJtcNrmROZ3sZu3&redirect_uri=https%3A%2F%2Flocal.cloudogu.com%2Fteamscale%2F
```

##### Successful response

**Status:** 200 OK

**Example response:**

```json
{
    "access_token": "TGT-1-m2gUNJwEqXyV7aAEXekihcVnFc5iI4mpfdZGOTSiiHzEbwr1cr-cas.ces.local",
    "expires_in": "7196",
    "token_type": "Bearer"
}
```

##### Unsuccessful response

**Error:** The short-term token is invalid or has already expired.

**Status:** 500 Bad request

**Example response:**

```json
{
    "message": "invalid_grant"
}
```

#### OIDC logout endpoint

This endpoint is used to terminate the OpenID Connect session.
The logout endpoint is used to invalidate the long term token from the CAS.

**URL** : `<fqdn>/oidc/logout`

**Method** : `GET`

**Condition of the data**

```
response_type = code
client_id = Valid clientID from the dogu
state = Any string
redirect_url = <URL to which the short term token of successful authentication will be redirected>.
```

**Data example**

```
response_type = code
client_id = teamscale
state = b8c57125-9281-4b67-b857-1559cdfcdf31
redirect_url = http://local.cloudogu.com/teamscale/
```

**Request example**

```
https://local.cloudogu.com/cas/oidc/authorize?client_id=teamscale&redirect_uri=http%3A%2F%2Flocal.cloudogu.com%2Fteamscale%2F&response_type=code&state=b8c57125-9281-4b67-b857-1559cdfcdf31
```

##### Successful response

Automatically redirects to the CAS login screen.
After successful login the `redirect_url` is passed with a `code` as GET parameter.

Example for `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

## Accessing the Registry

The Cloudogu EcoSystem Registry is a key-value database that is also a core element for Dogus. The configuration of a Dogu is performed via the Registry. In addition, values that are also of a global nature are stored in the registry.

### Registry access from the dogu

The following image focuses on parts that play a role in the communication between Dogu (that is: Redmine) and the Registry:

![Involved components that play a role in the interaction between Dogu and Registry](../images/important/chapter3_ces_registry_components.png)

- Registry: It runs beyond the Dogu container and can be reached via an IP address from the container network.
   - After installation, the registry contains the public key of each dogu among other keys.
- File `/etc/ces/node_master`.
   - This is automatically mounted in the dogu during the dogu installation. It contains the IP address of the registry so that the dogu can access the registry.
- File `/private/private.pem`
   - This file contains the private key of the dogu. This is used, among other things, to decrypt encrypted registry
     values.
   - Often these are [service accounts](../core/compendium_en.md#serviceaccounts) to other Dogus.

The Dogu specific configuration is located in the registry path `/config/<dogu>/`. [Registry keys](../core/compendium_en.md#configuration) should be written in `snake_case`, i.e. lowercase with underscores.

A valuable help is the command line tool `doguctl` among other things also in the start phase of the Dogu container. This tool simplifies access to the registry, e.g. by automatically reading the `node_master` file or simply addressing Dogu's own registry keys. A list of the exact commands can be found under [The use of doguctl](#usage-of-doguctl).

The `dogu.json` allows you to define your own configuration values, which can also be validated automatically.

```bash
# reads configuration value from the key /config/<dogu>/my_key
$ doguctl config my_key
my old value

# reads global configuration value from /config/_global/fqdn
$ doguctl config -g fqdn
your-ecosystem.example.com
```

### Other registry entries

There are other areas beyond the dogu-specific registry values that are of interest in the dogu startup phase.

The global configuration is located in the registry path `/config/_global/` and can be managed with [doguctl](#usage-of-doguctl).

- Global values `/config/_global`
   - `/config/_global/fqdn`
      - the FQDN of this Cloudogu EcoSystem instance.
      - e.g. eco.example.com
      - This value may change during operation. See also the section on [Changeability of the FQDN](#changeability-of-the-fqdn).
   - `/config/_global/domain`
      - the domain portion of this Cloudogu EcoSystem instance, often identical with the mail domain.
      - e.g. example.com
   - `/config/_global/mail_address`
      - the email address of the instance administrator
      - e.g. jane.doe@example.com
   - `/config/_global/admin_group`
      - the current name of the LDAP group whose members administer the Cloudogu EcoSystem instance in the UI.
      - This value may change during operation. See also the section on [admin group changeability](#changeability-of-the-admin-group).
- Dogu states `/state/<dogu>`
   - if the dogu defines a [HealthCheck](../core/compendium_en.md#healthchecks) of type `state`, then it allows
     administrators and other dogus to get health hints on the dogu.
   - In your own dogu e.g. set `doguctl state installing` when a longer installation routine is started. Just before the main process is started then use `doguctl state ready` to indicate a proper operating state.
   - In the EcoSystem host this can be checked with `cesapp healthy <dogu>`.
   - In other dogus this can be checked with `doguctl healthy <dogu>`.
      - With the `--timeout <seconds>` switch, you can thus wait for dogus on which your own dogu depends.
      - A non-zero exit code (0) indicates that either the dogu did not finish its startup process in time or the registry communication failed.

## Structure and best practices of `startup.sh`.

A simple Dogu actually only needs a `dogu.json` and a container image. But what happens when the reality of continuous operation differs from the assumptions made during Dogu development?

At Cloudogu, we quickly discovered that it is more clever to be able to deal with such changes during operation. The spectrum here is broad. It can be waiting times compared to other Dogus, or changing originally fixed identifiers to glitches in the container engine.

Recurring tasks at container startup can be found in the section "Typical Dogu Features" in the sections:

- [changeability-of-admin-group](#changeability-of-the-admin-group).
- [Changeability of FQDN](#changeability-of-the-fqdn)
- [control-logging-behavior](#controlling-the-logging-behavior)

In order to dynamically respond to these conditions, Cloudogu has adopted the practice of not starting the main process in the container directly. Instead, error cases are first checked in a startup script, new configuration values are set if necessary, and only then the main process is started, e.g. like this:

1. container starts `startup.sh`.
2. `startup.sh`
   1. sets own options
   2. calls own commands to implement specific functionalities, e.g. to
      - wait for dogu dependencies
      - perform one-time installation processes
      - generate a temporary admin account
      - apply the current dogu log level to the dogu application
      - customize the registration the Cloudogu EcoSystem's TLS certificates in the container
      - Prepare system changes to the registry with API accesses to prepare the software
         - Replication of LDAP groups/permissions
      - Configure the software (log level, other configuration, e.g. by `doguctl template`)
      - React on system changes of the registry: e.g. [admin-group](#changeability-of-the-admin-group) has changed, which needs further action in the Dogu
   3. set the Dogu state to `ready` (appropriate [HealthCheck](../core/compendium_en.md#healthchecks) provided)
   4. start the main process

This section therefore addresses findings and _best practices_ related to such startup scripts: The `startup.sh`.

By the way, diligent developers can gather inspiration in Cloudogu`s own startup
scripts [e.g. in the Redmine dogu](https://github.com/cloudogu/redmine/blob/develop/resources/startup.sh).

### Script interpreter

In order to run a script in a dogu, a script interpreter must exist in the container image. This can be an official package (like `bash`), but there is nothing against using your own script interpreter.

There is a broad landscape of available scripting languages. While interpreters like Python, Ruby, Groovy, or Kotlin provide a certain developer experience, they also bring a larger binary size to the table. Since the [Bash interpreter](https://www.gnu.org/software/bash/) is widely used in the Linux community and because of its small footprint regarding the container image size, Bash syntax is used here.

### Error handling

Depending on the dogu, errors can occur in different places and (stupidly) be swallowed again. It is a good practice to abort the script with an appropriate exit code when errors occur, in order to identify the cause of the error more quickly.

The first thing to do is to set the following options

```bash
#!/bin/bash
set -o errexit # terminate the whole script (and thus the container) on an uncaught error
set -o nounset # find uninitialized variables
set -o pipefail # don't ignore errors on pipe usage
```

How an error is caught is an implementation detail. Sometimes it is convenient to test an error case and send a separate message, sometimes the error itself is sufficient.

If you already expect errors, then this construct can be helpful:

```bash
yourCommandExitCode=0
your-command || yourCommandExitCode=$?
if [[ ${yourCommandExitCode} -ne 0 ]]; then
  echo "ERROR: Oh no. An error occurred during the execution of your-command.";
  doguctl state "ErrorYourFunctionality".
  sleep 300
  exit 1
fi
```

The error of the command `your-command` is caught. A separate error text is printed. The dogu-state is set to a non-`ready` value, which could be useful for debugging. Finally, five minutes (= 300 seconds) are waited until `exit 1` restarts the dogu as faulty.

### Divide and conquer with Bash

The longer you develop an application, the more functionality you add to it. The same can happen with `startup.sh`. Apart from easier testability (e.g. with [BATS](https://github.com/bats-core/bats-core)), smaller functions and components are easier to understand.

It is therefore a good idea to design Dogu functions in `startup.sh` in the same way from the point of view of testability, readability or refactorability.

#### Shell functions

Bash functions help here:

```bash
function setDoguLogLevel() {
  echo "Mapping dogu specific log level..."
  currentLogLevel=$(doguctl config --default "WARN" "logging/root")

  # map here the log level to the log configuration of the dogu
}
```

Recurring functions can be abstracted and parameterized:

```bash
function deleteSetupArtifact() {
  local artifactFile="${1}"
  rm -f "${artifactFile}"
}

function cleanUpSetup() {
  deleteSetupArtifact "/tmp/tracker-extract-3-files.127" }
  deleteSetupArtifact "/usr/share/lib/intermediateFile.bin"
}
```

#### Traceability during the startup phase

As complexity increases, it is worth an idea to add a `echo` to relevant steps to speed up a search for the error in log output in case of an error.

At the same time, the [Dogu-State-HealthCheck](../core/compendium_en.md#healthchecks) provides the possibility to mark custom execution phases during Dogu startup. For example `doguctl state 'aSpecialInstallPhase'` at the beginning or `doguctl state 'ready'` at the end of the script.

A HealthCheck of type `state` will only succeed if the state contains the string `ready`.

#### Code swapping

Another possibility is to outsource functions to other script files, which are then included in the actual startup script using `source`. It should be noted, however, that even `source` can fail and should be handled.

```bash
sourcingExitCode=0
source /resources/util.sh || sourcingExitCode=$?
if [[ ${sourcingExitCode} -ne 0 ]]; then
  echo "ERROR: An error occurred while sourcing /resources/util.sh: Exit code ${sourcingExitCode}"
  doguctl state "ErrorSourcingUtilSh"
  sleep 300
  exit 1
fi
```

### Quality assurance of startup scripts

For more complex tasks, a startup script such as `startup.sh` may gain complexity. Like any other program, it should be tested and analyzed for robustness. There are several ways to do this. [BATS](https://github.com/bats-core/bats-core) can assist in unit testing shell functions.

Analysis tools such as [Shellcheck](https://www.shellcheck.net/) can detect errors in the use of shell calls.

### Usage of `doguctl`

The section [about registry access](#registry-access-from-the-dogu) has already touched on the subject of `doguctl`. `doguctl` is a command line tool that bundles and simplifies recurring interactions with the Cloudogu EcoSystem registry. This section describes possible calls.

With `--help`, each subcommand of `doguctl` prints a help page.

#### doguctl config

This call reads and writes configuration values.

```bash
# reads configuration value from the key /config/<dogu>/my_key
$ doguctl config my_key
my old value

# reads global configuration value from /config/_global/fqdn
$ doguctl config -g fqdn
your-ecosystem.example.com

# reads configuration value from /config/<dogu>/my_key and returns default value if not set
$ doguctl config --default NOVALUE my_key2
NOVALUE

# reads encrypted DB name /config/<dogu>/sa-postgresql/db_name
$ doguctl config -e sa-postgresql/db_name
your-dogu-database-1234

# writes configuration value into /config/<dogu>/my_key
$ doguctl config my_key 'my new value'

# writes encrypted secret /config/<dogu>/secret/credential
$ doguctl config -e secret/credential '$up3r$3cre7'

# deletes the key /config/<dogu>/delete_me
$ doguctl config --rm delete_me
```

#### doguctl validate

This call validates registry configuration values. The command `validate` returns an exit code == 0 if all health checks of the dogus under consideration return a positive result. Otherwise it returns an exit code == 1.

```bash
$ doguctl validate logging/root # validate single value, good case: value=WARN from ERROR,WARN,INFO,DEBUG
Key logging/root: OK
$ echo $?
0

$ doguctl validate logging/root # validates single value, error case: value=foo from ERROR,WARN,INFO,DEBUG
Key logging/root: INVALID
at least one key was invalid
$ echo $?
1

$ doguctl validate --all # validates all values with output
...
Key container_config/memory_limit: OK
Key container_config/swap_limit: OK
Key container_config/java_max_ram_percentage: OK
Key container_config/java_min_ram_percentage: OK
Key logging/root: INVALID
Key caching/workdir/size: OK (No Validator)
at least one key was invalid
$ echo $?
1

$ doguctl validate --silent # validates values without output (can be combined with --all), good case
$ echo $?
0

$ doguctl validate --silent # validates values without output (can be combined with --all), error case
at least one key was invalid
echo $?
1
```

#### doguctl healthy

This call checks whether a given dogu is healthy. The command `healthy` returns an exit code == 0 if the given dogu is healthy. Otherwise, it returns an exit code == 1.

```bash
if ! doguctl healthy --wait --timeout 120 postgresql; then
   echo "timeout reached by waiting of ldap to get healthy"
   exit 1
fi
```

#### doguctl state

This call reads and writes dogu state values. It is used in combination with a [HealthCheck](../core/compendium_en.md#healthchecks).
It is lost upon restart of the container. For persistent values that need to be present even after restarts, `doguctl config` 
should be used.

```bash
$ doguctl state "installing" # write the value to the state
# Do installation

$ doguctl state # reads the value from the state
installing

$ doguctl state "ready" # set the state to the default healthy value
# start the application
```

#### doguctl random

This call generates random strings, suitable for generating passwords or other credentials.

```bash
$ doguctl random # generates random string with length 16
9HoF4nYmlLYtf6Ju
$ doguctl random -l 5 # generates random string with length 5
W6Wmj
```

#### doguctl template

This call creates a file from a [golang template](https://pkg.go.dev/text/template). Different registry values and environment variables can be processed directly in this.

The call takes one or two parameters:

```bash
$ doguctl template <template file> [output file]
```

If the `template` command is used without an output file, then the rendered output is printed to `stdout`.

Supported template parameters:

- Environment variables
   - `.Env.Get <environment variable>` - uses previously exported environment variable "ADMIN_USERNAME"
- Dogu configuration
   - `.Config.Get <Dogu configuration key>` - uses unencrypted Dogu configuration value under `/config/<dogu>/<key>`
   - `.Config.GetOrDefault <Dogu configuration key> <default value>` - uses unencrypted Dogu configuration value or
     provided default value
   - `.Config.GetAndDecrypt <encrypted Dogu configuration key>` - decrypts Dogu configuration value and uses it
   - `.Config.Exists <Dogu configuration key>` - returns a `bool` value if a Dogu configuration value exists
     under `/config/<dogu>/<key>`
- Global configuration
   - `.GlobalConfig.Get <global configuration key>` - uses global configuration value `/config/_global/<key>`
   - `.GlobalConfig.GetOrDefault <global configuration key> <default value>` - uses unencrypted global configuration
     value or provided default value
   - `.GlobalConfig.Exists <Dogu configuration key>` - returns a `bool` value if a global configuration value exists
     at `/config/_global/<key>`
- Dogu queries
   - `.Dogus.IsEnabled <Dogu-name>` - returns a `bool` value if a Dogu is installed
   - `.Dogus.Get <Dogu-Name>` - returns a [Dogu](../core/compendium_en.md#type-dogu) struct
   - `.Dogus.GetAll` - return a slice of all [Dogus](../core/compendium_en.md#type-dogu)

Example of a template file:

```gotemplate
# my-pseudo-config.conf
[Global]
admin.username = {{ .Env.Get "ADMIN_USERNAME" }}
funny.name = {{ .Config.Get "something_funny" }}
log.level = {{ .Config.GetOrDefault "log_level" "WARN" }}
url.jdbc = jdbc:postgresql://postgresql:5432/{{ .Config.GetAndDecrypt "sa-postgresql/database" }}
url.fqdn = https://{{ .GlobalConfig.Get "fqdn" }}/my-dogu

{{ if .Config.Exists "notification" }}
notification = mail
{{ end }}

{{ if .Dogus.IsEnabled "redmine" }}
ticketsystem.url = https://{{ .GlobalConfig.Get "fqdn" }}/redmine
{{ end }}
...
```

#### doguctl wait-for-tcp

With this call `doguctl` waits until a given TCP port is open or a timeout (in seconds) occurs. An exit code != 0 signals an error.

```bash
if ! doguctl wait-for-tcp --timeout 120 8080; then
  echo "Received timeout after waiting 120 seconds for port 8080. Exiting."
  exit 1
fi
```

#### doguctl wait-for-http

With this call `doguctl` waits until a given HTTP(s) URL is ready or a timeout (in seconds) occurs. An exit code != 0 signals an error.

```bash
if ! doguctl wait-for-tcp --host postgresql --timeout 120 5432; then
   echo "Received timeout after waiting 120 seconds for port 5432 of host postgresql. Exiting."
   exit 1
fi
```

## Service Accounts

Service Accounts represent a mechanism for access accounts that Dogus need to secure or store their data, but do not want to provide this functionality themselves. Dogus can represent themselves as producers and/or consumers of Service Accounts. In each case, both are optional features of a Dogu.

Generally, the nature of the application in the dogu itself dictates whether or which of the two makes the most sense. For example, a database management dogu such as PostgreSQL that does not offer other dogus access to its database would not be useful. On the other hand, it would be fatal for a dogu like Redmine not to have a dogu into which it can store its data via SQL.

Dogus like PostgreSQL that can provide access accounts to other dogus are also called _service account producers_. Dogus like Redmine that require an access account to another dogu are called _service account consumers_.

### Producing service accounts

Service accounts are not requested by a service account consumer itself, because a dogu should not care about this form of orchestration. This is instead the job of a client like `cesapp` or `k8s-dogu-operator` during the installation of the service account consumer. The `ExposedCommand` for this is `service-account-create`.

Similarly, the deletion of service accounts is handled by a client like `cesapp` or `k8s-dogu-operator` during the uninstallation of a dogu to delete its requested service accounts. The `ExposedCommand` for this is `service-account-remove`.

In order for a dogu to act as a service account producer, it must provide two executables in its `dogu.json`:

```json
{
  "ExposedCommands": [
    {
      "Name": "service-account-create",
      "Description": "Creates a new service account",
      "Command": "/create-sa.sh"
    },
    {
      "Name": "service-account-remove",
      "Description": "Removes a service account",
      "Command": "/remove-sa.sh"
    }
  ]
}
```

The terms `service-account-create` and `service-account-remove` are protected properties and are used by clients interpreting `dogu.json` as commands to create and delete service accounts.

The following two sections discuss the creation and deletion of service accounts. More information about exposed commands can be found in the [ExposedCommands](../core/compendium_en.md#exposedcommands) section of the compendium.

#### Create Service Account

The `service-account-create` script manages the creation of a new service account. It is always executed when a new consumer is installed. In any case, during the call, the `service-account-create` script receives the name of consumer dogu as its first argument. A second argument to fine-tune the service account is optional and depends on the support of the producer dogu.

A `service-account-create` script consists of three steps:

1. create the access data
2. creating an account for our software with the information
3. returning the access data
   - This results in them being automatically written in encrypted form to the consumer's dogu-specific configuration area.

An example of the `service-account-create` script could look like this:

`create-sa.sh`

```bash
#!/bin/bash
set -o errexit
set -o nounset
set -o pipefail
doguName="${1}"
{
#1) Create the credentials
   USER="${doguName}-$(doguctl random)"
   PASSWORD=$(doguctl random)

#2) Creates an account for our software with the information
# ... Implement the dogu specific logic here
} >/dev/null 2>&1 # suppress other output on stdout, because only credentials or errors are expected

#3) Return credentials. The client takes care of storage and encryption.
echo "username: ${USER}"
echo "password: ${PASSWORD}"
```

In step 1, credentials are randomly generated. The name of the consumer dogus can be included in the credentials, but this is not mandatory.

In step 2, an account must be created in the Dogu software with the generated information.

In step 3, the credentials get printed in the format `<registrykey>: <registryvalue>`. `registrykey` and `registryvalue` must be separated by a colon and a space. The whole line must be terminated by a newline.

These credentials are automatically read by the processing client and stored in the consumer's dogu-specific configuration area in encrypted form. Following the example above, the consumer dogu would set up two registry entries containing the respective secret as a value:

- `/config/<consumer>/sa-<producer>/username`
- `/config/<consumer>/sa-<producer>/password`

The consumer dogu can now read and decrypt these e.g. by `doguctl config -e sa-<producer>/username`.

#### Deleting a Service Account

The `service-account-remove` script deletes an existing service account of a consumer dogu.

It is executed whenever the consumer dogu gets uninstalled. During the call, the `service-account-remove` script receives the name of the consumer dogu as its first argument. Since no output from the script is processed here, error messages can also be printed to `stdout`.

A `service-account-remove` script consists of two steps:

1. identify the consumer
2. delete the data and the service account of the consumer

An example of the `service-account-remove` script might look like this:

`remove-sa.sh`

```bash
#!/bin/bash
set -o errexit
set -o nounset
set -o pipefail

#1) identify the consumer
doguName="${1}"
if [ X"${doguName}" = X"" ]; then
  echo "usage remove-sa.sh dogu-name"
  exit 1
fi

#2) Delete the service account of the consumer
# ... Implement the Dogu specific logic here
```

In step 1, the name of the consumer dogu is read (e.g. "redmine") to identify the service account of the dogu.

In step 2 the account is deleted from the producer's database including access data.

### Consuming service accounts

It is very easy to request a service account from a producer dogu, since the main work is done by the client and the producer. A consumer dogu just needs to name the desired producer dogu as [service account](../core/compendium_en.md#serviceaccounts) in its `dogu.json`:

```json
{
  "ServiceAccounts": [
    {
      "Type": "your-producer-dogu",
      "Kind": "dogu"
    }
  ]
}
```

A service account is a special form of dependency. Therefore, it makes sense to keep the producer dogu in its [dependency list](../core/compendium_en.md#dependencies).  In the different phases of a Dogu installation, this ensures that the producer dogu is really available for use:

```json
{
  "Dependencies": [
    {
      "Name": "your-producer-dogu",
      "Type": "dogu"
    }
  ],
  "ServiceAccounts": [
    {
      "Type": "your-producer-dogu",
      "Kind": "dogu"
    }
  ]
}
```

## Dogu upgrades

Dogus can be upgraded in the Cloudogu EcoSystem using simple means (e.g. `cesapp upgrade dogu <dogu name>`). The entire upgrade process must be automated beforehand during dogu development and can then be run through by the CES tools (here: `cesapp`) without much assistance from the admins.

The upgrade process includes the following steps:
- Displaying upgrade notifications, if any
- Downloading the new Dogu image
- Ensuring that the Dogu is healthy
- Ensuring that all dependent dogus are available
- Running the pre-upgrade script on the old dogu, if available
- Replacing the old dogu with the new one
- Starting the new dogu
- Running the post-upgrade script in the new dogu, if any

If it is necessary to perform pre- or post-upgrade steps during a dogu update, optional pre- and post-upgrade scripts can be created. As an example, consider software that requires a different database schema after an update. In such a case it is possible to write upgrade scripts to start a migration of the database after a Dogu update.

In addition, it is possible to be notified by an upgrade notification script about important changes when upgrading Dogus.

All upgrade scripts described below should be placed in the `resources` folder (besides e.g. the `startup.sh` script).

### Determining the upgrade paths

When developing a Dogu, it must be ensured that upgrading a Dogu also works across multiple versions. For this purpose, the upgrade scripts must be designed in such a way that they determine the upgrade path and automatically perform any necessary migration steps.

For this purpose, all upgrade scripts can read the previous (`FROM`) and new (`TO`) Dogu version:

```bash
FROM_VERSION="${1}"
TO_VERSION="${2}"
```

### `pre-upgrade` - Perform actions before upgrading a dogu

The `pre-upgrade` script can be defined as [Exposed Command](../core/compendium_en.md#exposedcommands) in the `dogu.json` of a Dogu:

```json
{
   "ExposedCommands": [
    {
      "Name": "pre-upgrade",
      "Command": "/pre-upgrade.sh"
    }
  ]
}
```

This script is executed before the actual upgrade of the Dogu in the old Dogu container.

#### Important info on the pre-upgrade process

- The script is not necessarily executed from the same path where it is located in the new Dogu container. Accordingly, absolute paths must be used in the script.
- The complete pre-upgrade must be **a single** script, which can be copied accordingly into the old container. If the script uses additional files, these must already be present on the old container.

### `post-upgrade` - Perform actions after the upgrade of a dogu

The `post-upgrade` script can be defined as [Exposed Command](../core/compendium_en.md#exposedcommands) in the `dogu.json` of a dogu:

```json
{
  "ExposedCommands": [
    {
      "Name": "post-upgrade",
      "Command": "/post-upgrade.sh"
    }
  ]
}
```

As soon as it boots up, this script will be started in the new dogu container after the dogu has been upgraded. So normally both the `startup.sh` and the `post-upgrade` script will run at the same time. If it should be necessary that the `post-upgrade` script is executed before the `startup.sh` script, the `startup.sh` script must be provided with a waiting mechanism. This can be done, for example, by waiting for a registry key to be set by the `post-upgrade` script once it has run completely.

### `upgrade-notification` - Show a notification before the dogu upgrade confirmation

The `upgrade-notification` script can be defined as [Exposed Command](../core/compendium_en.md#exposedcommands) in the `dogu.json` of a dogus:

```json
{
  "ExposedCommands": [
    {
      "Name": "upgrade-notification",
      "Command": "/upgrade-notification.sh"
    }
  ]
}
```

This script is executed before the upgrade process and should only output information (via `echo`) that may be relevant for the administrator before the upgrade. For example, it may indicate breaking changes or that a backup should be made before the upgrade.

## Typical Dogu features

This chapter describes features that integrate Dogus deeper into the Cloudogu EcoSystem and make them uniformly administrable.

### Resource requirements

The resource requirements for a Dogu can be applied to the following resources:

* Memory (RAM)
* Swap
* CPU cores
* Ephemeral storage

A distinction is made between resource requests and limits:

- **Resource Requests:** Specify the minimum resources (CPU cores, memory, ephemeral storage) required by a dogu for it to be functional.
  In the Multinode-EcoSystem, the Kubernetes scheduler ensures that the Dogu is started on a node with sufficient resources.
- **Resource Limits:** Specify the maximum amount of resources a Dogu is allowed to use.

#### Memory

Memory limits can be used to restrict the memory consumption (RAM) of Dogus.

If a Dogu exceeds its memory limit, the largest process in the container is terminated.
This is usually the main process of the Dogus and causes the container to be restarted.
If no value is set for the memory limit, this will not happen.

Memory requests can be used to specify the minimum memory requirement for Dogus to be fully functional.

**Important:**
Memory-Requests are only applied in the Multinode-EcoSystem.

#### Swap

Swap limits can be used to restrict the memory consumption (swap space) of Dogus.

If a Dogu exceeds its swap limit, the largest process in the container is terminated.
This is usually the main process of the Dogus and causes the container to be restarted.
If no value is set on swap limiting, it will not take place.
For swap limiting `0b` is the default value and thus does not provide swap.

**Important:**
Swap limits are **NOT** applied in the Multinode-EcoSystem!

##### Preparation of the storage limits in the host

In order for the ClouDogu EcoSystem host to limit memory **and** swap, the following settings must be made beforehand:
1. open the file `/etc/default/grub`.
2. the following value must be added to the variable `GRUB_CMDLINE_LINUX_DEFAULT`:
   `cgroup_enable=memory swapaccount=1`.
3. save the changes
4. execute the command `sudo update-grub` or `sudo update-bootloader --refresh`.
5. reboot the machine

**Warning!**
The above activation with `cgroup_enable=memory swapaccount=1` is expected to result in a memory overhead of 1% and a performance penalty of 10% even if Docker is not running.

#### CPU cores

CPU limits can be used to restrict the maximum CPU cores available to Dogus.
When the CPU cores limit is exceeded, the container runtime throttles the available CPU resources for the container.

CPU requests can be used to specify the minimum CPU cores required for a Dogu to be fully functional.

**Important:**
CPU limits and requests are only applied in the Multinode-EcoSystem.

#### Ephemeral-Storage

Ephemeral storage limits can be used to limit the maximum available ephemeral storage
(see [Kubernetes-Ephemeral-Storage](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage))
of Dogus.
When the limit is exceeded, the container is restarted.

Ephemeral storage requests can be used to specify the minimum ephemeral storage required for a Dogu to be fully functional.

**Important:**
Ephemeral-Storage-Limits and -Requests are only applied in the Multinode-EcoSystem.


#### Configuration in the Dogu

To be able to set resource requests, the `dogu.json` of the Dogus must contain the following entries:

```json
{
  "Configuration": [
    {
      "Name": "container_config/memory_limit",
      "Description":"Limits the container's memory usage. Use a positive integer value followed by one of these units [b,k,m,g] (byte, kibibyte, mebibyte, gibibyte).",
      "Optional": true,
      "Validation": {
        "Type": "BINARY_MEASUREMENT"
      }
    },
    {
      "Name": "container_config/memory_request",
      "Description":"Requests the container's minimal memory requirement. Use a positive integer value followed by one of these units [b,k,m,g] (byte, kibibyte, mebibyte, gibibyte). This configuration is only applicable to the Multinode-EcoSystem.",
      "Optional": true,
      "Validation": {
        "Type": "BINARY_MEASUREMENT"
      }
    },
    {
      "Name": "container_config/swap_limit",
      "Description":"Limits the container's swap memory usage. Use zero or a positive integer value followed by one of these units [b,k,m,g] (byte, kibibyte, mebibyte, gibibyte). 0 will disable swapping.",
      "Optional": true,
      "Validation": {
        "Type": "BINARY_MEASUREMENT"
      }
    },
    {
      "Name": "container_config/cpu_core_limit",
      "Description":"Limits the container's CPU core usage. Use a positive floating value describing a fraction of 1 CPU core. When you define a value of '0.5', you are requesting half as much CPU time compared to if you asked for '1.0' CPU. This configuration is only applicable to the Multinode-EcoSystem.",
      "Optional": true
    },
    {
      "Name": "container_config/cpu_core_request",
      "Description":"Requests the container's minimal CPU core requirement. Use a positive floating value describing a fraction of 1 CPU core. When you define a value of '0.5', you are requesting half as much CPU time compared to if you asked for '1.0' CPU. This configuration is only applicable to the Multinode-EcoSystem.",
      "Optional": true
    },
    {
      "Name": "container_config/storage_limit",
      "Description":"Limits the container's ephemeral storage usage. Use a positive integer value followed by one of these units [b,k,m,g] (byte, kibibyte, mebibyte, gibibyte). This configuration is only applicable to the Multinode-EcoSystem.",
      "Optional": true,
      "Validation": {
        "Type": "BINARY_MEASUREMENT"
      }
    },
    {
      "Name": "container_config/storage_request",
      "Description":"Requests the container's minimal ephemeral storage requirement. Use a positive integer value followed by one of these units [b,k,m,g] (byte, kibibyte, mebibyte, gibibyte). This configuration is only applicable to the Multinode-EcoSystem.",
      "Optional": true,
      "Validation": {
        "Type": "BINARY_MEASUREMENT"
      }
    }
  ]
}
```

This creates the registry entries for the resource requirements in the respective Dogu configuration.

**Memory**

- key for request: `config/<DOGU_NAME>/container_config/memory_request`
- Key for limit: `config/<DOGU_NAME>/container_config/memory_limit`
- Optional
- Description: Sets the memory resource requirement of the Dogus.
- Format: The configurable values for the keys are each a string of the form `<number value><unit>` and describe the maximum amount of memory that can be used by the Dogu.
  Note here that there must be no space between the numeric value and the unit.
  Available units are `b`, `k`, `m` and `g` (for byte, kibibyte, mebibyte and gibibyte).

**Swap**

- Key for limit: `config/<DOGU_NAME>/container_config/swap_limit`
- Optional
- Description: Sets the swap limit of the dogus.
- Format: The configurable values for the keys are each a string of the form `<number value><unit>` and describe the maximum amount of memory that can be used by the dogu.
  Note here that there must be no space between the numeric value and the unit.
  Available units are `b`, `k`, `m` and `g` (for byte, kibibyte, mebibyte and gibibyte).

**CPU cores**

- Key for request: `config/<DOGU_NAME>/container_config/cpu_core_request`.
- Key for limit: `config/<DOGU_NAME>/container_config/cpu_core_limit`
- Optional
- Description: Sets the CPU resource request of the dogus.
- Format:
  See https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes

**Ephemeral-Storage**

- Key for request: `config/<DOGU_NAME>/container_config/storage_request`
- Key for limit: `config/<DOGU_NAME>/container_config/storage_limit`
- Optional
- Description: Sets the ephemeral storage resource requirement of the Dogus.
- Format: The configurable values for the keys are each a string of the form `<number value><unit>` and describe the maximum amount of storage that can be used by the Dogu.
  Note here that there must be no space between the numeric value and the unit.
  Available units are `b`, `k`, `m` and `g` (for byte, kibibyte, mebibyte and gibibyte).

  
Setting the values can be done in the following ways:
- `doguctl config container_config/memory_limit 1g`
- `cesapp edit-config <doguname>` (only from the host)
- `etcdctl set /config/<doguname>/container_config/memory_limit "1g"` (only from the host)

##### Apply configured resource requests

To apply the limits in the Singlenode-EcoSystem, the Dogu must be recreated (`cesapp recreate <doguname>`) and then restarted (`cesapp start <doguname>`).

In the Multinode-EcoSystem, the global etcd key `config/_global/sync_resource_requirements` must be created/modified/deleted. 
Any change to the key will start an automatic update process for all dogus. 
In this update process resource requirements are applied to all dogus and dogus are restarted if new resource requirements are set. 
Unchanged dogus are not restarted. 
In general, the update process can be started with the following command:

```bash
etcdctl set /config/_global/sync_resource_requirements true
```

#### Checking the resource requirements

**Singlenode-EcoSystem:**
The limitation of memory (RAM only, no swap) can be checked using `docker stats <doguname>`.
The `MEM USAGE / LIMIT` column should correctly show the memory limit set.

**Multinode-EcoSystem:**
The resource requirements are contained in the Kubernetes pod resource and can be checked there.
For example, with:

```shell
kubectl get pod ldap-6ccb6c78fd-rp86t -o yaml | yq -r '.spec.containers[0].resources'
```

#### Limit in Java Dogus

A special case is the limiting of a Java process. If a Dogu contains a Java process, the following additional entries can be added to `dogu.json`:

```json
{
  "Configuration": [
    {
      "Name": "container_config/java_max_ram_percentage",
      "Description": "Limits the heap stack size of the Java process to the configured percentage of the available physical memory when the container has more than approx. 250 MB of memory available. Is only considered when a memory_limit is set. Use a valid float value with decimals between 0 and 100 (f. ex. 55.0 for 55%). Default value: 25%",
      "Optional": true,
      "Default": "25.0",
      "Validation": {
        "Type": "FLOAT_PERCENTAGE_HUNDRED"
      }
    },
    {
      "Name": "container_config/java_min_ram_percentage",
      "Description": "Limits the heap stack size of the Java process to the configured percentage of the available physical memory when the container has less than approx. 250 MB of memory available. Is only considered when a memory_limit is set. Use a valid float value with decimals between 0 and 100 (f. ex. 55.0 for 55%). Default value: 50%",
      "Optional": true,
      "Default": "50.0",
      "Validation": {
        "Type": "FLOAT_PERCENTAGE_HUNDRED"
      }
    }
  ]
}
```

**Notes:**
- The percentages always refer to the limited memory of a container. If no limit is set, the percentages are also ignored.
- A Java process should not be allocated too large a share of the memory. One should always consider any other processes that also need memory. If this does not happen, the actual program may crash.
- The values that can be configured with this must be given as parameters to the corresponding Java process in the start scripts of the Dogus. A reference implementation can be found in the [Nexus Dogu](https://github.com/cloudogu/nexus/blob/77bdcfdbe0787c85d2d9b168dc38ff04b225706d/resources/util.sh#L52).

### Backup & restore capability

One feature of the Cloudogu EcoSystem is that Dogus can be backed up and restored via a central backup system. For this to work, all Dogus must offload their mutable data to volumes. This procedure is already described in the [Compendium](../core/compendium_en.md#volumes).

When developing a Dogu, care must be taken to ensure that the Dogu continues to function normally after a successful backup and restore operation and that all data and functions are available. To do this, all volumes in the `dogu.json` that contain production data must first be marked with the [NeedsBackup](../core/compendium_en.md#needsbackup) flag. After that the Dogu should be built and filled with test data.

After that, one performs a backup and a restore of the system. The commands [cesapp backup](https://docs.cloudogu.com/de/docs/system-components/cesapp/operations/backup/) and [cesapp restore](https://docs.cloudogu.com/de/docs/system-components/cesapp/operations/restore/) can be used for this.

Once all Dogus have been restarted, test whether your own Dogu is running normally and all test data is still available.

### Changeability of the admin group

When installing the Cloudogu EcoSystem, a global admin group is set. All members of this group should be given admin rights in all Dogus. This must be ensured during the development of the Dogus. The name of the globally defined admin group can be queried from within the Dogu using `doguctl`: `doguctl config --global 'admin_group'`.

Additionally, the admin group can be changed afterwards. The Dogu must then react (after a reboot, if necessary) in such a way that only members of the new admin group are granted admin rights. Users in the old admin group will only receive normal user rights unless they are also in the new admin group.

### Changeability of the FQDN

The Fully Qualified Domain Name (FQDN) of the Cloudogu EcoSystem is stored globally in the registry and can be read by the Dogus using [`doguctl`](#usage-of-doguctl). If necessary, it can be integrated into the Dogu configuration (see also [Creating an exemplary application](../core/basics_en.md#5-creating-an-exemplary-application). In addition, care must be taken during Dogu development to ensure that the FQDN is changeable. So the Dogu should be able (after a restart if necessary) to read a new FQDN and adapt its configuration to this new FQDN.

### Controlling the logging behavior

This section shows how to create a uniform, abstracted setting option that can be used to set the log behavior in all Dogus in the same way. Four different log levels that can be set via the registry are used for this purpose: `ERROR`, `WARN`, `INFO` and `DEBUG`.

These abstract log levels help the administrator as he does not need to know what a dogu-specific log level is called, but can instead fall back on four basic values and reuse them safely in all Dogus.

Dogu developers must take care that these four log levels are mapped meaningfully to the log levels of the software in the Dogu, which may have different names (e.g. `TRACE` or `FATAL`).

#### Log level backgrounds

Nobody reads logs. If someone reads them, it is not with pleasure, for two reasons:
1. standard logs contain mostly unimportant information for the viewer
2. logs are only read when a problem occurs.

Log level control addresses these circumstances.

If a dogu logs too many lines by default and permanently, then administrators or support staff have their lives made harder than necessary.
This is because they have to work through a mountain of lines and separate the logs into relevant and irrelevant information.
In addition, excessive logging (e.g. long stack traces combined with container crash loops) can lead to so much data that the hard disk fills up and log rotation is no longer possible.

#### Log levels and their margins

The log levels are sorted by level of detail from coarse to fine and contain the respective level above. For example, log outputs of the level `WARN` thus also contain log outputs of the level `ERROR`.

This gives the developer a great deal of freedom as to which logs are output at which log level.

To distinguish between relevant and irrelevant, there is of course room for interpretation.
Cloudogu as a Dogu provider supports the customer in making logging as low and as concise as possible for their productive systems.
This means, based on our experience as both software developers and Dogu users, we decide for the customer,
which log sources, log lines etc. are appropriate per Dogu and log level.

##### `ERROR`

Log levels analogous to `ERROR` are very often found on production systems. They are used so that operations or support do not have to go through superfluous additional information, but can efficiently get to the core of problems.

Log output of this level should only contain output that indicates acute errors, operational prevention, or security violations, etc.

Examples:
- Repair of corrupt data in a database
- application error
- unplanned stop or restart of services or applications
- acute connectivity errors
   - e.g. Postgresql dogu is not reachable even after several minutes
- _repeated_, failed login attempts
- acutely threatening or acutely existing lack of resources
- messages with `FATAL` or `SEVERE` log level

##### `WARN`

Log levels analogous to `WARN` are frequently found on production or staging systems.

Log outputs of this level are very similar to those of `ERROR`. Unlike these, however, they contain output indicating imminent errors.

Examples:
- faulty or optimizable configuration was detected
- update of application or libraries is suggested
- imminent lack of resources in the medium term
- singular connectivity errors, which are limited in time
   - e.g. Postgresql dogu is not reachable at the first attempt, but after a few minutes it is reachable again
- _single_ failed login attempts
- scheduled start and stop of services or applications

##### `INFO`

Outputs that might be of interest in special but non-critical situations. A _catch-all_ for all the output that is not important enough for `WARN`, but does not yet go deep into the details of the application.

Examples:
- a subsystem was completely loaded
- Error-free result of background jobs
- Successful start/stop of a component

##### `DEBUG`

Output that may be of interest in critical situations and/or used for debugging.

Examples:
- a relevant decision for a configuration was made
- Messages with `TRACE` log level

#### Mapping per Dogu

These four abstract log levels occasionally have no direct relationship to concrete log levels in the tools or plugins in the Dogu and must be mapped manually in Dogu development. The reason for this is that both the tools and the associated plugins and frameworks have log levels that differ too greatly in quantity and quality.

The mapping of an abstract log level is thus in the hands of the Dogu development. Here the developers can decide, for example, to map the `TRACE` log level from the software to the `DEBUG` log level of the Dogu.

This mapping from abstract to concrete log levels has the advantage of flexibility, e.g. if several configuration files have to be filled with different log levels.

#### Value range and default

To set the log level, an optional key `/config/${dogu}/logging/root` is introduced per Dogu, which can take one of the following abstract log level values.

- `ERROR`
- `WARN` (default value)
- `INFO`
- `DEBUG`

If no `/config/${dogu}/logging/root` key exists, then `doguctl` will automatically use the `WARN` value if properly configured [as default value](#validation-and-default-values).

#### Validation and default values

`doguctl` supports dogu developers defining a log level for a dogu in a sturdy way. This leaves only the mapping of a found log level to the respective logging framework in the dogu.

In the `dogu.json` a log level configuration value can be set, which both can be validated and also provides a default value on the Dogu.

```json
{
  "Configuration": [
    {
      "Name": "logging/root",
      "Description": "Set the root log level to one of ERROR, WARN, INFO, DEBUG.",
      "Optional": true,
      "Default": "WARN",
      "Validation": {
        "Type": "ONE_OF",
        "Values": [
          "WARN",
          "DEBUG",
          "INFO",
          "ERROR"
        ]
      }
    }
  ]
}
```

This value can be checked with `doguctl validate`:

```
if [[ ! doguctl validate logging/root --silent ]]; then
   echo "WARNING: Found invalid value in logging/root. Resetting it to WARN"
   doguctl config logging/root WARN
fi
```

The set or default value from `dogu.json` can be retrieved as usual. `doguctl` takes care of getting a default value if no value has been set:

```
rootLogLevel=$(doguctl config logging/root)
```

#### Extensibility

If there is a need for additional Dogu-specific log levels, the Dogu log levels can be supplemented with additional keys (also under `/config/<doguname>/logging/`).
Such log levels are very specific by their nature.
Therefore, different rules may apply to the use of such specific keys.

## Supported Communication Protocols

In the Cloudogu EcoSystem, communication protocols play a key role in data exchange, bei it the connection between a user and a dogu, or a dogu to another dogu or another actively running service. The protocols being used strongly depend on each dogu. This chapter enumerates protocols that are supported and have been successfully tested. 

Further information about troubleshooting network communication can be found in the [troubleshooting document](troubleshooting_en.md#troubleshooting-network-problems-in-a-dogu)

### User-to-dogu communication

There are two ways to make a connection from a user client to a dogu:

1. the nginx reverse-proxy dogu
   - generally, all network traffic from the user's web client to the target dogu is reverse-proxied by the nginx dogu
   - nginx takes care of the TLS termination and the internal URL rewriting
2. exposed dogu ports
   - a dogu might also [expose its ports to the outside](../core/compendium_en.md#exposedports)
   - the nginx dogu will not handle or re-route any requests made to this port
   - generally speaking, this is a dangerous matter because the dogu must solve all security measures by itself
     - f. i. authorization, SSO/SLO, request throttling, access logging, etc.

Currently, these communication protocols have been successfully tested from outside into a Cloudogu EcoSystem:
- via nginx:
  - HTTP 1 and 2
  - Websockets
- via port exposing
  - SSH

### Dogu-to-dogu/xyz communication

Of course, dogus can communicate with other dogus over the internal network, too! Usually this is done in an unsecured 
fashion, i. e. HTTP (and not HTTPS).

Currently, these communication protocols have been successfully tested from inside the Cloudogu EcoSystem:
- HTTP 1 and 2
- Websockets
- gRPC
