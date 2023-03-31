# Relevant functionalities

This chapter describes the features and possible implementation ideas/solutions of those functionalities that make up a
real Dogu in the first place and solve recurring problems.

Therefore, the following sections deal with recurring functionalities on how a Dogu can embed itself into the Cloudogu
EcoSystem landscape.

## Authentication

Provided Dogus provide this, user:ins can enjoy the benefits of enterprise-level single sign-on (SSO) and single
logout (SLO). Authentication is enabled by the Central Authentication Service (CAS). Dogu developers have several
authentication options at their disposal. The following three sections deal with methods that have already been
successfully used, namely:

- Authentication using CAS protocol
- Authentication using OAuth
- Authentication using OpenID Connect

The basis for CAS here is always a correct configuration of a user/group directory, be it
the [LDAP](https://github.com/cloudogu/ldap) dogu or an external LDAP-compatible service.

### CAS protocol

Authentication within the Cloudogu EcoSystem is done using
the [CAS protocol](https://apereo.github.io/cas/6.5.x/protocol/CAS-Protocol.html), which enables single sign-on and
single log-out. Different CAS protocol versions are supported (2.0 and 3.0). When implementing using CAS protocol, it is
recommended to use version 3.0 because (compared to version 2.0) it can return important user attributes after a
successful authentication.

The following diagram shows the parties involved in the authentication configuration. Before a dogu (here using Redmine
as an example) can participate in this process, the dogu must internally configure a set of CAS URLs:

- CAS log-in URL (for redirecting users to the web form).
- CAS validation URL (for validating service tickets)
- CAS log-out URL (to invalidate an SSO session)
- The service recognition in CAS against the Dogus happens automatically during the Dogu installation and will not be
  considered further in the following.

![Participants in the authentication configuration](../images/important/chapter3_auth_cas_config.png)

The actual authentication happens via a sequence of HTTP redirects and the exchange of session cookies in the
background, of which users are unaware. The display of the CAS login form as an active login step stands out strongly.

The following figure provides a rough overview of the login process with the parties involved.

![CAS authentication workflow](../images/important/chapter3_auth_cas_sequencediag.png)

The SSO of the CAS significantly reduces this process when logging in to additional dogus.

For more information and a more detailed illustration before, during and after an authentication, see
the [CAS documentation](https://apereo.github.io/cas/6.5.x/protocol/CAS-Protocol.html).

### OAuth protocol

CAS offers OAuth/OIDC as a protocol for authentication including SSO/SLO. The following describes the specification of
OAuth 2.0 protocol in CAS.

#### Creating an OAuth Service Account for a dogu

In order for a Dogu to use the CAS's OAuth endpoints, it must log in to the CAS as a client.
To do this, the request for an OAuth-specific CAS service account can be stored in the `dogu.json` of the dogu in
question.

**entry for an OAuth client:**

```json
{
  ...,
  "ServiceAccounts": [
    {
      "Type": "cas",
      "Params": [
        "oauth"
      ]
    }
  ],
  ...
}
```

The service account credentials are randomly generated (
see [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) and stored encrypted in the
registry under the path `/config/<dogu>/sa-cas/oauth` and `/config/<dogu>/sa-cas/oauth_client_secret`.

The credentials are composed of the `CLIENT_ID` and the `CLIENT_SECRET`. For the CAS, the `CLIENT_SECRET` is stored as a
hash in the Cloudogu EcoSystem Registry under the path `/config/cas/service_accounts/oauth/<CLIENT_ID>`.

#### OAuth endpoints and authentication flow

The following steps describe a successful OAuth authentication flow.

1. request a short term token: see section below "OAuth authorize endpoint".
2. swap short-term token for long-term token: see section below "AccessToken endpoint".
3. long term token can now be used to authenticate against resources.
   Currently, CAS only offers the user's profile as a resource: See section [OAuth user profile](#oauth-user-profile).

![Authentication sequence with OAuth 2.0](../images/important/chapter3_auth_oauth_sequencediag.png)

#### OAuth authorize endpoint

This endpoint serves as the initial start of OAuth authorization.
The `authorize` endpoint is used to request a short-lived token from the CAS.

**URL** : `<fqdn>/oauth2.0/authorize`

**Method** : `GET`

**Condition of the data**

```
?response_type = code
?client_id = Valid clientID from the dogu
?state = Any string
?redirect_url = <URL to which the short-term token will be redirected upon successful authentication>
```

**Data example**

```
?response_type = code
?client_id = portainer
?state = b8c57125-9281-4b67-b857-1559cdfcdf31
?redirect_url = http://local.cloudogu.com/portainer/
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
?grant_type = authorization_code
?code = Valid code from `authorize` endpoint
?client_id = Valid clientID from the dogu
?client_secret = Valid secret from the dogu
?redirect_url = <URL to which the long term token will be sent upon successful authentication>
```

**Data example**

```
?grant_type = authorization_code
?code = ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local
?client_id = portainer
?client_secret = sPJtcNrmROZ3sZu3
?redirect_url = https://local.cloudogu.com/portainer/
```

**Call example**

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

#### Create OIDC service account for dogu.

In order for a dogu to use the CAS's OIDC endpoints, it must log in to the CAS as a client.
To do this, the request for an OIDC-specific CAS service account can be stored in the `dogu.json` of the dogu in question.

**entry for an OIDC client:**
```json
{
  ...,
  "ServiceAccounts": [
    {
      "Type": "cas",
      "Params": [
        "oidc"
      ]
    }
  ],
  ...
}
```

The service account credentials are randomly generated (
see [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) and stored encrypted in the
registry under the path `/config/<dogu>/sa-cas/oidc` and `/config/<dogu>/sa-cas/oidc_client_secret`.

The credentials are composed of the `CLIENT_ID` and the `CLIENT_SECRET`. For the CAS, the `CLIENT_SECRET` is stored as a
hash in the Cloudogu EcoSystem Registry under the path `/config/cas/service_accounts/oidc/<CLIENT_ID>`.

#### OIDC-Authorize-Endpoint.

This endpoint serves as the initial start of the OpenID Connect authorization.
The `authorize` endpoint is used to request a short-lived token from the CAS.

**URL** : `<fqdn>/oidc/authorize`

**Method** : `GET`

**Condition of the data**

```
?response_type = code
?client_id = Valid clientID from the dogu
?state = Any string
?redirect_url = <URL to which the short term token of successful authentication will be redirected>.
```

**Data example**

```
?response_type = code
?client_id = teamscale
?state = b8c57125-9281-4b67-b857-1559cdfcdf31
?redirect_url = http://local.cloudogu.com/teamscale/
```

**call example**

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
?grant_type = authorization_code
?code = Valid code from `authorize` endpoint
?client_id = Valid clientID from the dogu
?client_secret = Valid secret from the dogu
?redirect_url = <URL to which the long term token of successful authentication is sent>
```

**Data example**

```
?grant_type = authorization_code
?code = ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local
?client_id = teamscale
?client_secret = sPJtcNrmROZ3sZu3
?redirect_url = https://local.cloudogu.com/teamscale/
```

**Call example**

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
The logout endpoint is used to invalidate a the long term token from the CAS.

**URL** : `<fqdn>/oidc/logout`

**Method** : `GET`

**Condition of the data**

```
?response_type = code
?client_id = Valid clientID from the dogu
?state = Any string
?redirect_url = <URL to which the short term token of successful authentication will be redirected>.
```

**Data example**

```
?response_type = code
?client_id = teamscale
?state = b8c57125-9281-4b67-b857-1559cdfcdf31
?redirect_url = http://local.cloudogu.com/teamscale/
```

**call example**

```
https://local.cloudogu.com/cas/oidc/authorize?client_id=teamscale&redirect_uri=http%3A%2F%2Flocal.cloudogu.com%2Fteamscale%2F&response_type=code&state=b8c57125-9281-4b67-b857-1559cdfcdf31
```

##### Successful response

Automatically redirects one to the CAS login screen.
After successful login the `redirect_url` is passed with a `code` as GET parameter.

Example for `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

## Accessing the Registry

The Cloudogu EcoSystem Registry is a key-value database that is also a core element for Dogus. The configuration of a
Dogus is performed via the Registry. In addition, values that are also of a global nature are stored in the registry.

### Registry access from the dogu

The following image focuses on parts that play a role in the communication between Dogu (exemplary here: Redmine) and
the Registry:

![Involved components that play a role in the interaction between Dogu and Registry](../images/important/chapter3_ces_registry_components.png)

- Registry: It runs beyond the Dogu container and can be reached via an IP address from the container network.
   - After installation, the registry contains the public key of each dogu among other keys.
- File `/etc/ces/node_master`.
   - This is automatically mounted in the dogu during the dogu installation. It contains the IP address of the registry
     so that the dogu can access the registry.
- File `/private/private.pem`
   - This file contains the private key of the dogu. This is used, among other things, to decrypt encrypted registry
     values.
   - Often these are [service accounts](../core/compendium_en.md#serviceaccounts) to other Dogus.

The Dogu specific configuration is located in the registry
path `/config/<dogu>/`. [Registry keys](../core/compendium_en.md#configuration) should be written in `snake_case`, i.e.
lowercase with underscores.

A valuable help is the command line tool `doguctl` among other things also in the start phase of the Dogu container.
This tool simplifies access to the registry, e.g. by automatically reading the `node_master` file or simply addressing
Dogu's own registry keys. A list of the exact commands can be found under [The use of doguctl](#the-use-of-doguctl).

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

The global configuration is located in the registry path `/config/_global/` and can be managed
with [doguctl](#the-use-of-doguctl).

- Global values `/config/_global`
   - `/config/_global/fqdn`
      - the FQDN of this Cloudogu EcoSystem instance.
      - e.g. eco.example.com
      - This value may change during operation. See also the section on [Changeability of FQDN](#changeability-of-fqdn).
   - `/config/_global/domain`
      - the domain portion of this Cloudogu EcoSystem instance that dogus can use to send mail.
      - e.g. example.com
   - `/config/_global/mail_address`
      - the email address of the instance administrator
      - e.g. petra.mustermann@example.com
   - `/config/_global/admin_group`
      - the current name of the LDAP group whose members administer the Cloudogu EcoSystem instance in the UI.
      - This value may change during operation. See also the section
        on [admin group changeability](#changeability-admin-group).
- Dogu states `/state/<dogu>`
   - if the dogu defines a [HealthCheck](../core/compendium_en.md#healthchecks) of type `state`, then it allows the
     administrator and other dogus to get health hints on the dogu
   - In your own dogu e.g. set `doguctl state installing` when a longer installation routine is started. Just before the
     main process is started then use `doguctl state ready` to indicate a proper operating state.
   - In the EcoSystem host this can be checked with `cesapp healthy <dogu>`.
   - In other dogus this can be checked with `doguctl healthy <dogu>`.
      - With the `--timeout <seconds>` switch, you can thus wait for dogus on which your own dogu depends.
      - A non-zero exit code (0) indicates that either the dogu did not finish its startup process in time or the
        registry communication failed.

## Structure and best practices of `startup.sh`.

A simple Dogu actually only needs a `dogu.json` and a container image. But what happens when the reality of continuous
operation differs from the assumptions made during Dogu development?

At Cloudogu, we quickly discovered that it is more clever to be able to deal with such changes during operation. The
spectrum here is broad. It can be waiting times compared to other Dogus, or changing originally fixed identifiers to
glitches in the container engine.

Recurring tasks at container startup can be found in the section "Typical Dogu Features" in the sections:

- [changeability-of-admin-group](#changeability-of-admin-group).
- [Changeability of FQDN](#changeability-of-fqdn)
- [control-logging-behavior](#control-logging-behavior)

In order to dynamically respond to these conditions, Cloudogu has adopted the practice of not starting the main process
in the container directly. Instead, error cases are first checked in a startup script, new configuration values are set
if necessary, and only then the main process is started, e.g. like this:

1. container starts `startup.sh`.
2. `startup.sh`
   1. sets own options
   2. calls own commands to implement specific functionalities, e.g. to
      - wait for dogu dependencies
      - perform one-time installation processes
      - generate a temporary admin account
      - implement the current log level
      - register the CES-TLS certificates in the software/target system
      - Prepare system changes to the registry with API accesses to prepare the software
         - Replication of LDAP groups/rights
      - Setting the software (log level, other configuration, e.g. by `doguctl template`)
      - React on system changes of the registry: e.g. [admin-group](#changeability-of-admin-group) has changed, which
        needs further action in the Dogu
   3. set the Dogu state to `ready` (appropriate [HealthCheck](../core/compendium_en.md#healthchecks) provided)
   4. start the main process

This section therefore addresses findings and _best practices_ related to such startup scripts: The `startup.sh`.

By the way, diligent developer:s can gather inspiration in Cloudogu`s own startup
scripts [e.g., in Redmine-Dogu](https://github.com/cloudogu/redmine/blob/develop/resources/startup.sh).

### Script interpreter

In order to run a script in a dogu, a script interpreter must exist in the container image. This can be an official
package (like `bash`), but there is nothing against using your own script interpreter. Since bash scripts are widely
used, bash syntax is used here.

### Error handling

Depending on the dogu, errors can occur in different places and (stupidly) be swallowed again. It is a good practice to
abort the script with an appropriate exit code when errors occur, in order to identify the cause of the error more
quickly.

The first thing to do is to set the following options

```bash
#!/bin/bash
set -o errexit # terminate the whole script (and thus the container) on an uncaught error 
set -o nounset # find uninitialized variables
set -o pipefail # don't swallow errors on pipe usage
```

How an error is caught is an implementation detail. Sometimes it is convenient to test an error case and send a separate
message, sometimes the error itself is sufficient.

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

The error of the command `your-command` is caught. A separate error text is printed. The dogu-state is set to a
non-`ready`-value, which could be useful for debugging. Finally, five minutes (= 300 seconds) are waited until `exit 1`
restarts the dogu as faulty.

### Divide and conquer with Bash

The longer you develop an application, the more functionality you add to it. The same can happen with `startup.sh`.
Apart from easier testability (e.g. with [BATS](https://github.com/bats-core/bats-core)), small execution parts are
easier to understand.

It is therefore a good idea to design Dogu functions in `startup.sh` in the same way from the point of view of
testability, readability or refactorability.

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

As complexity increases, it is worth an idea to add a `echo` to relevant steps to speed up a search for the error in log
output in case of an error.

At the same time, the [Dogu-State-HealthCheck](../core/compendium_en.md#healthchecks) provides the possibility to mark
custom execution phases during Dogu startup. For example `doguctl state 'aSpecialInstallPhase'` at the beginning
or `doguctl state 'ready'` at the end of the script.

A HealthCheck of type `state` will only succeed if the state contains the string `ready`.

#### Code swapping

Another possibility is to outsource functions to other script files, which are then included in the actual start script
using `source`. It should be noted, however, that even such a `source`n can fail and should be handled.

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

For more complex tasks, a startup script such as `startup.sh` may gain complexity. Like any other program, it should be
tested and analyzed for robustness. There are several ways to do this. [BATS](https://github.com/bats-core/bats-core)
can assist in unit testing shell functions.

Analysis tools such as [Shellcheck](https://www.shellcheck.net/) can detect errors in the use of shell calls.

### The use of `doguctl`

The section [about registry access](#registry-access-from-dogu-here) has already touched on the subject
of `doguctl`. `doguctl` is a command line tool that bundles and simplifies recurring interactions with the Cloudogu
EcoSystem registry. This section describes possible calls.

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

This call validates registry configuration values. The command `validate` returns an exit code == 0 if all HealthChecks
of the dogus under consideration return a positive result. Otherwise it returns an exit code == 1.

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

This call checks whether a given dogu is healthy. The command `healthy` returns an exit code == 0 if the given dogu is
healthy. Otherwise, it returns an exit code == 1.

```bash
if ! doguctl healthy --wait --timeout 120 postgresql; then
   echo "timeout reached by waiting of ldap to get healthy".
   exit 1
fi
```

#### doguctl state

This call reads and writes dogu state values. It is used in combination with
a [HealthCheck](../core/compendium_en.md#healthchecks).

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

This call creates a file from a [golang template](https://pkg.go.dev/text/template). Different registry values and
environment variables can be processed directly in this.

The call takes one or two parameters:

```bash
$ doguctl template <template file> [output file]
```

If the `template` command is used without an output file, then the rendered output is printed to `stdout`.

Supported template parameters:

- Environment variables
   - `.Env.Get <environment variable>` - uses previously exported environment variable "ADMIN_USERNAME".
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
   - `.Dogus.Get <Dogu-Name>` - returns a [Dogu](../core/compendium_en.md#type-dogu)-Struct
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

With this call `doguctl` waits until a given HTTP(s) URL is ready or a timeout (in seconds) occurs. An exit code != 0
signals an error.

```bash
if ! doguctl wait-for-tcp --host postgresql --timeout 120 5432; then
   echo "Received timeout after waiting 120 seconds for port 5432 of host postgresql. Exiting."
   exit 1
fi
```

## Service Accounts

Service Accounts represent a mechanism for access accounts that Dogus need to secure or store their data, but do not
want to provide this functionality themselves. Dogus can represent themselves as producers and/or consumers of Service
Accounts. In each case, both are optional features of a Dogus.

Generally, the nature of the application in the dogu itself dictates whether or which of the two makes the most sense.
For example, a database management dogu such as PostgreSQL that does not offer other dogus access to its database would
not be useful. On the other hand, it would be fatal for a dogu like Redmine not to have a dogu into which it can store
its data via SQL.

Dogus like PostgreSQL that can provide access accounts to other dogus are also called _service account producers_. Dogus
like Redmine that require an access account to another dogu are called _service account consumers_.

### Producing service accounts

Service accounts are not requested by a service account consumer itself, because a dogu should not care about this form
of orchestration. This is instead the job of a client like `cesapp` or `k8s-dogu-operator` during the installation of
the service account consumer. The `ExposedCommand` for this is `service-account-create`.

Similarly, the deletion of service accounts is handled by a client like `cesapp` or `k8s-dogu-operator` during the
uninstallation of a dogus to delete its requested service accounts. The `ExposedCommand` for this
is `service-account-remove`.

In order for a dogu to act as a service-account producer, it must provide two executables in its `dogu.json`:

```json
{
  ...,
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
  ],
  ...
}
```

The terms `service-account-create` and `service-account-remove` are protected properties and are used by clients interpreting `dogu.json` as commands to create and delete service accounts.

The following two sections discuss the creation and deletion of service accounts. More information about ExposedCommands can be found in the [ExposedCommands](../core/compendium_en.md#exposedcommands) section of the compendium.

#### Create Service Account

The `service-account-create` script manages the creation of a new service account. It is always executed when a new consumer is installed. In any case, during the call, the `service-account-create` script receives the name of the consumer dogus as its first argument. A second argument to fine-tune the service account is optional and depends on the support of the producer dogus.

A `service-account-create` script consists of three steps:

1. create the access data
2. creating an account for our software with the information
3. returning the access data
   - This results in them being automatically written in encrypted form to the consumer's Dogu-specific configuration area.

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

In step 1, credentials are randomly generated. The name of the consumer dogus can be included in the credentials, but
this is not mandatory.

In step 2, an account must be created in the Dogu software with the generated information.

In step 3, the credentials are output with the format `echo "<registrykey>: <registryvalue>"`. `registrykey`
and `registryvalue` must be separated by a colon and a space. The whole line must be terminated by a newline.

These credentials are automatically read by the processing client and stored in the consumer's Dogu-specific
configuration area in encrypted form.
Following the example above, the consumer dogu would set up two registry entries containing the respective secret as a
value:

- `/config/<consumer>/sa-<producer>/username`
- `/config/<consumer>/sa-<producer>/password`

The verbaucher-dogu can now read and decrypt these e.g. by `doguctl config -e sa-<produzentdogu>/username`.

Delete #### service account

The `service-account-remove` script deletes an existing service account of a consumer dogu.

It is executed whenever the consumer dogu is uninstalled. During the call, the `service-account-remove` script receives
the name of the consumer dogu as its first argument. Since no output from the script is processed here, error messages
can also be printed to `stdout`.

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
SERVICE="${1}"
if [ X"${SERVICE}" = X"" ]; then
  echo "usage remove-sa.sh servicename"
  exit 1
fi

#2) Delete the service account of the consumer
# ... Implement the Dogu specific logic here
```

In step 1, the name of the consumer dogu is read (e.g. "redmine") to identify the service account of the dogu.

In step 2 the account is deleted from the producer's database including access data.

### Consume service accounts

It is very easy to request a service account from a producer-dogu, since the main work is done by the client and the
producer. A consumer dogu just needs to name the desired producer dogu
as [service account](../core/compendium_en.md#serviceaccounts) in its `dogu.json`:

```json
{
  ...,
  "ServiceAccounts": [
    {
      "Type": "producerdogu",
      "Kind": "dogu"
    }
  ],
  ...
}
```

A service account is a special form of dependency. Therefore, it makes sense to keep the producer Dogu in
its [dependency list](../core/compendium_en.md#dependencies). This ensures in the different phases of a Dogu
installation that the producer Dogu is really available for use:

```json
{
  ...,
  "Dependencies": [
    {
      "Name": "producerdogu",
      "Type": "dogu"
    }
  ],
  "ServiceAccounts": [
    {
      "Type": "producerdogu",
      "Kind": "dogu"
    }
  ],
  ...,
}
```
