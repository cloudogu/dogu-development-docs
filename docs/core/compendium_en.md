# Compendium
This document describes the compendium of the `dogu.json` and its fields including the depending types in detail,
the syntax and example contents. For new developers a good start is to study the [Dogu](<#type-dogu>) type which 
represents the `dogu.json`.


# package core

## Index

- [type ConfigurationField](<#type-configurationfield>)
- [type Dependency](<#type-dependency>)
- [type Dogu](<#type-dogu>)
- [type DoguJsonV2FormatProvider](<#type-dogujsonv2formatprovider>)
- [type EnvironmentVariable](<#type-environmentvariable>)
- [type ExposedCommand](<#type-exposedcommand>)
- [type ExposedPort](<#type-exposedport>)
- [type HealthCheck](<#type-healthcheck>)
- [type ServiceAccount](<#type-serviceaccount>)
- [type ValidationDescriptor](<#type-validationdescriptor>)
- [type Volume](<#type-volume>)
- [type VolumeClient](<#type-volumeclient>)




## type [ConfigurationField](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L320-L395>)

ConfigurationField describes a single dogu configuration field which is stored in the Cloudogu EcoSystem registry.

```go
type ConfigurationField struct {
    Name string

    Description string

    Optional bool

    Encrypted bool

    Global bool

    Default string

    Validation ValidationDescriptor
}
```

### Name

Name contains the name of the configuration key. The field is mandatory. It must not contain leading or trailing slashes "/", but it may contain directory keys delimited with slashes "/" within the name.

The Name syntax is encouraged to consist of:

- lower case latin characters
- special characters underscore "_"
- ciphers 0-9

Example:

- feedback_url
- logging/root

### Description

Description offers context and purpose of the configuration field in human-readable format. This field is optional, yet highly recommended to be set.

Example:

- "Set the root log level to one of ERROR, WARN, INFO, DEBUG or TRACE. Default is INFO"
- "URL of the feedback service"

### Optional

Optional allows to have this configuration field unset, otherwise a value must be set. This field is optional. If unset, a value of `false` will be assumed.

Example:

- true

### Encrypted

Encrypted marks this configuration field to contain a sensitive value that will be encrypted with the dogu's private key. This field is optional. If unset, a value of `false` will be assumed.

Example:

- true

### Global

Global marks this configuration field to contain a value that is available for all dogus. This field is optional. If unset, a value of `false` will be assumed.

Example:

- true

### Default

Default defines a default value that may be evaluated if no value was configured, or the value is empty or even invalid. This field is optional.

Example:

- "WARN"
- "true"
- "[https://scm-manager.org/plugins](https://scm-manager.org/plugins)"

### Validation

Validation configures a validator that will be used to mark invalid or out-of-range values for this configuration field. This field is optional.

Example:

```
"Validation": {
   "Type": "ONE_OF", // only allows one of these two values
   "Values": [
     "value 1",
     "value 2",
   ]
 }

"Validation": {
   "Type": "FLOAT_PERCENTAGE_HUNDRED" // valid values range between 0.0 and 100.0
}

"Validation": {
  "Type": "BINARY_MEASUREMENT" // only allows suffixed integer values measured in byte, kibibyte, mebibyte, gibibyte
 }
```

## type [Dependency](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L454-L503>)

Dependency describes the quality of a dogu dependency towards another entity.

Examples:

```
{
  "type": "dogu",
  "name": "postgresql"
}

{
  "name": "postgresql"
}

{
  "type": "client",
  "name": "k8s-dogu-operator",
  "version": ">=0.16.0"
}

{
  "type": "package",
  "name": "cesappd",
  "version": ">=3.2.0"
}
```

```go
type Dependency struct {
    Type string `json:"type"`

    Name string `json:"name"`

    Version string `json:"version"`
}
```

### Type

Type identifies the entity on which the dogu depends. This field is optional. If unset, a value of `dogu` is then assumed.

Valid values are one of these: "dogu", "client", "package".

A type of "dogu" references another dogu which must be present and running during the dependency check.

A type of "client" references the client which processes this dogu's "dogu.json". Several client dependencies of a different client type can be used f. i. to prohibit the processing of a certain client.

A type of "package" references a necessary operating system package that must be present during the dependency check.

Examples:

- "dogu"
- "client"
- "package"

### Name

Name identifies the entity selected by Type. This field is mandatory. If the Type selects another dogu, Name must use the simple dogu name (f. e. "postgres"), not the full qualified dogu name (not "official/postgres").

Examples:

- "postgresql"
- "k8s-dogu-operator"
- "cesappd"

### Version

Version selects the version of entity selected by Type. This field is optional. If unset, any version of the selected entity will be accepted during the dependency check.

Version accepts different version styles and compare operators.

Examples:

- ">=4.1.1-2" - select the entity version greater than or equal to version 4.1.1-2
- "<=1.0.1" - select the entity version less than or equal to version 1.0.1
- "1.2.3.4" - select exactly the version 1.2.3.4

With a non-existing version it is possible to negate a dependency.

Example:

- "<=0.0.0" - prohibit the selected entity being present

## type [Dogu](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L544-L985>)

Dogu describes properties of a containerized application for the Cloudogu EcoSystem. Besides the meta information and the [OCI container image](https://opencontainers.org/), Dogu describes all necessities for automatic container instantiation, f. i. volumes, dependencies towards other dogus, and much more.

Example:

```
{
 "Name": "official/newdogu",
 "Version": "1.0.0-1",
 "DisplayName": "My new Dogu",
 "Description": "Newdogu is a test application",
 "Category": "Development Apps",
 "Tags": ["warp"],
 "Url": "https://www.company.com/newdogu",
 "Image": "registry.cloudogu.com/official/newdogu",
 "Dependencies": [
   {
     "type":"dogu",
     "name":"nginx"
   }
 ],
 "Volumes": [
   {
     "Name": "temp",
     "Path":"/tmp",
     "Owner":"1000",
     "Group":"1000",
     "NeedsBackup": false
   }
 ],
 "HealthChecks": [
   {
     "Type": "tcp",
     "Port": 8080
   }
 ]
}
```

```go
type Dogu struct {
    Name string

    Version string

    DisplayName string

    Description string

    Category string

    Tags []string

    Logo string

    URL string

    Image string

    ExposedPorts []ExposedPort

    ExposedCommands []ExposedCommand

    Volumes []Volume

    HealthCheck HealthCheck

    HealthChecks []HealthCheck

    ServiceAccounts []ServiceAccount

    Privileged bool

    Configuration []ConfigurationField

    Properties Properties

    EnvironmentVariables []EnvironmentVariable

    Dependencies []Dependency

    OptionalDependencies []Dependency
}
```

### Name

Name contains the dogu's full qualified name which consists of the dogu namespace and the dogu simple name, delimited by a single forward slash "/". This field is mandatory.

The dogu namespace allows to regulate access to dogus in that namespace. There are three reserved dogu namespaces: The namespaces `official` and `k8s` are open to all users without any further costs. In contrast, the `premium` namespace is open to subscription users only.

The namespace syntax is encouraged to consist of:

- lower case latin characters
- special characters underscore "_", minus "-"
- ciphers 0-9
- an overall length of less than 200 characters

The dogu simple name allows to address in multiple ways. The simple name will be the part of the URI of the Cloudogu EcoSystem to address a URI part (if the dogu provides an exposed UI). Also, the simple name will be used to address the dogu after the installation process (f. i. to start, stop or remove a dogu), or to address generated resources that belong to the dogu.

The simple name syntax must be an DNS-compatible identifier and is encouraged to consist of

- lower case latin characters
- special characters underscore "_", minus "-"
- ciphers 0-9
- an overall length of less than 20 characters

It is recommended to use the same full qualified dogu name within the dogu's Dockerfile as a label named `NAME`.

Examples:

- official/redmine
- premium/confluence
- foo-1/bar-2

### Version

Version defines the actual version of the dogu. This field is mandatory.

The version follows the format from semantic versioning and additionally is split in two parts. The application version and the dogu version.

An example would be 1.7.8-1 or 2.2.0-4. The first part of the version (e.g. 1.7.8) represents the version of the application (e.g. the nginx version in the nginx dogu). The second part represents the version of the dogu and for an initial release it should start at 1 (e.g. 1.7.8-1).

If the application does not change but e.g. there are changes in the startup script of the dogu the new version should be 1.7.8-2. If the application itself changes (e.g. there is a nginx upgrade) the new version should be 1.7.9-1. Notice that in this case the version of the dogu should be set to 1 again.

Whereas the dogu struct is the core place for the version and is used by the cesapp in various processes like installation and release the version should also be placed as a label in the dockerfile from the dogu.

Example versions in the dogu.json:

- 1.7.8-1
- 2.2.0-4

Recommended example in the Dockerfile:

```
LABEL maintainer="hello@cloudogu.com" \
NAME="official/nginx" \
VERSION="1.23.2-1"
```

### DisplayName

DisplayName is the name of the dogu which is used in UI frontends to represent the dogu. This field is mandatory.

Usages: In the setup of the ecosystem the display name of the dogu is used to select it for installation.

the display name is used in the warp menu to let the user navigate to the web ui of the dogu.

Another usage is the textual output of tools like the cesapp or the k8s-dogu-operator where the name is used in commands like list upgradeable dogus.

The description may consist of:

- lower and upper case latin characters where the first is upper case
- any special characters
- ciphers 0-9

Description is encouraged to consist of less than 30 characters because it is displayed in the Cloudogu EcoSystem warp menu.

Examples:

- Jenkins CI
- Backup & Restore
- SCM-Manager
- Smeagol

### Description

Description contains a short explanation, what the dogu does. This field is mandatory.

It is used in the setup of the ecosystem in the dogu selection. Therefore, the description should give an uninformed user a brief hint what the dogu is and maybe the function the dogu fulfills.

The description may consist of:

- lower and upper case latin characters where the first is upper case
- any special characters
- ciphers 0-9

Description is encouraged to consist a readable sentence which explains shortly the dogu's main topic.

Examples:

- Jenkins Continuous Integration Server
- MySQL - Relational database
- The Nexus Repository is like the local warehouse where all the parts and finished goods used in your software supply chain are stored and distributed.

### Category

Category organizes the dogus in three categories. This field is mandatory.

These categories are fixed and must be either:

- `Development Apps` - For regular dogus which should be used by a regular user of the ecosystem,
- `Administration Apps` - For dogus which should be used by a user with administration rights
- `Base` - For dogus which are important for the overall system.

The categories "Development Apps" and "Administration Apps" are represented in the warp menu to order the dogus.

Example dogus for each category:

- `Development Apps`: Redmine, SCM-Manager, Jenkins
- `Administration Apps`: Backup & Restore, User Management
- `Base`: Nginx, Registrator, OpenLDAP

### Tags

Tags contains a list of one-word-tags which are in connection with the dogu. This field is optional.

If the dogu should be displayed in the warp menu the tag "warp" is necessary. Other tags won't be automatically processed.

Examples for e.g. Jenkins:

```
{"warp", "build", "ci", "cd"}
```

### Logo

Logo represents a URI to a web picture depicting the dogu tool. This field is optional.

Deprecated: The Cloudogu EcoSystem does not facilitate the logo URI. It is a candidate for removal. Other options of representing a tool or application can be:

- embed the logo in the dogu's Git repository (if public)
- provide the logo in the dogu UI (if the dogu provides one)

### URL

URL links the website to the original tool vendor. This field is optional.

The Cloudogu EcoSystem does not facilitate this information. Anyhow, in a public dogu repository in which a dogu vendor re-packages a third party application the URL may point users to resources of the original tool vendor.

Examples:

- [https://github.com/cloudogu/usermgt](https://github.com/cloudogu/usermgt)
- [https://www.atlassian.com/software/jira](https://www.atlassian.com/software/jira)

### Image

Image links to the [OCI container](https://opencontainers.org/) image which packages the dogu application. This field is mandatory.

The image must not contain image tags, like the image version or "latest" (use the field Version for this information instead). The image registry part of this field must point to `registry.cloudogu.com`.

It is good practice to apply the same name to the image repository as from the Name field in order to enable access strategies as well as to avoid storage conflicts.

Examples for official/redmine:

- registry.cloudogu.com/official/redmine

### ExposedPorts

ExposedPorts contains a list of [ExposedPort](#type-exposedport). This field is optional.

Exposed ports provide a way to route traffic from outside the Cloudogu EcoSystem towards the dogu. Dogus that provide a GUI must also expose a port that accepts requests and returns responses. Non-GUI dogus can also expose ports if the dogu provides services to a consumer (f. i. when the dogu wants to provide an API for a CLI tool).

Examples:

```
[ { "Type": "tcp", "Container": "2222", "Host":"2222" } ]
```

### ExposedCommands

ExposedCommands defines actions of type [ExposedCommand](#type-exposedcommand) which can be executed in different phases of the dogu lifecycle automatically triggered by a dogu client like the cesapp or the k8s-dogu-operator or manually from an administrative user. This field is optional.

Usually, commands which are automatically triggered by a dogu-client are those in upgrade or service account processes. These commands are: pre-upgrade, post-upgrade, upgrade-notification, service-account-create, service-account-remove

pre-upgrade: This command will be executed during an early stage of an upgrade process from a dogu. A dogu client will mount the pre-upgrade script from the new dogu version in the container of the old still running dogu and executes it. It is mainly used to prepare data migrations (e.g. export a database). Core of the script should be the comparison between the version and to determine if a migration is needed. For this, the dogu client will call the script with the old version as the first and the new version as the second parameter. In addition, it is recommended to set states like "upgrading" or "pre-upgrade done" in the script. This can be very useful because you can use it in the post-upgrade or the regular startup for a waiting functionality.

post-upgrade: This command will be executed after a regular dogu upgrade. Like in the pre-upgrade the old dogu version is passed as the first and the new dogu version is passed as the second parameter. They should be used to determine if an action is needed. Keep in mind that in this time the regular startup script of your new container will be executed as well. Use a state in the CES-registry to handle a wait functionality in the startup. If the post-upgrade ends reset this state and start the regular container.

upgrade-notification: If the upgrade process works, e.g. with really sensitive data an upgrade notification script can be implemented to inform the user about the risk of this process and e.g. give a backup instruction. Before an upgrade the dogu client executes the notification script, prints all upgrade steps and asks the user if he wants to proceed.

service-account-create: The service-account-create command is used in the installation process of a dogu. A service account in the Cloudogu EcoSystem represents credentials to authorize another dogu, like a database or an authentication server. The service-account-create command has to be implemented in the service account producer dogu which produces the credentials. If a service account consuming dogu will be installed and requires a service account (e.g. for postgresql) the dogu client will call the service-account-create script in the postgresql dogu with the service name as the first parameters and custom ones as additional parameters. See core.ServiceAccounts for how to define custom parameters. With this information the script should create a service account and save it (e.g. USER table in an underlying database or maybe encrypted in the CES-registry). After that, the credentials must be printed to console so that the dogu client saves the credentials for the dogu which requested the service account. It is also important that these outputs are the only ones from the script, otherwise the dogu client will use them as credentials.

Example output:

```
echo "database: ${DATABASE}"
echo "username: ${USER}"
echo "password: ${PASSWORD}"
```

service-account-delete: If a dogu will be deleted the used service account should be deleted too. Because of this, the service account producing dogu (like postgresql) must implement a service-account-delete command. This script will be executed in the deletion process of a dogu. The dogu processing client passes the service account name as the only parameter. In contrast to the service-account-create command the script can output logs because the dogu client won't use any of this data.

Example commands for service accounts:

```
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
```

Custom commands: Moreover, a dogu can specify commands which are not executed automatically in the dogu lifecycle (e.g. upgrade). For example, a dogu like Redmine can specify a command to delete or install a plugin. At runtime an administrator can call this command with the cesapp like:

```
cesapp command redmine plugin-install scrum-plugin
```

Example:

```
"ExposedCommands": [
   {
     "Name": "plugin-install",
     "Description": "Installs a plugin",
     "Command": "/installPlugin.sh"
   }
 ],
```

### Volumes

Volumes contains a list of [Volume](#type-volume) which defines [OCI container volumes](https://opencontainers.org/). This field is optional.

Volumes are created during the dogu creation or upgrade. Volumes provide a performance boost compared to in-container storage.

Examples:

```
[ { "Name": "temp", "Path":"/tmp", "Owner":"1000", "Group":"1000", "NeedsBackup": false} ]
```

### HealthCheck

HealthCheck defines a single way to check the dogu health for observability. This field is optional.

Deprecated: use HealthChecks instead

### HealthChecks

HealthChecks defines multiple ways to check the dogu health for observability. This field is optional.

HealthChecks are used in various use cases:

- to show the `dogu is starting` page until the dogu is healthy at startup
- for monitoring via `cesapp healthy <dogu-name>`
- for monitoring via the admin dogu
- to avoid backing up unhealthy dogus

There are different types of health checks:

- state, via the `/state/<dogu>` registry key set by ces-setup
- tcp, to check for an open port
- http, to check a status code of a http response

They must be executable without authentication required.

Example:

```
"HealthChecks": [
  {"Type": "state"}
  {"Type": "tcp",  "Port": 8080},
  {"Type": "http", "Port": 8080, "Path": "/my/health/path"},
]
```

### ServiceAccounts

ServiceAccounts contains a list of [ServiceAccount](#type-serviceaccount). This field is optional.

A ServiceAccount protects sensitive data used for another dogu. Service account consumers are distinguished from service account producers. To produce a service account a dogu has to implement service-account-create and service-account-delete scripts in core.ExposedCommands. To consume a service account a dogu should define a request in ServiceAccounts.

In the installation process a dogu client recognizes a service account request and executes the service-account-create script with this information from the service account producer. The credentials will be then stored in the Cloudogu EcoSystem registry.

Example paths:

- config/redmine/sa-postgresql/username
- config/redmine/sa-postgresql/password
- config/redmine/sa-postgresql/database

The dogu itself can then use the service account by read and decrypt the credentials with `doguctl` in the startup script.

Examples:

```
"ServiceAccounts": [
  {
    "Type": "ldap",
    "Params": [
      "rw"
    ]
  },
  {
   "Type": "k8s-dogu-operator",
   "Kind": "k8s"
  }
],
```

### Privileged

Privileged indicates whether the Docker socket should be mounted into the container file system. This field is optional. The default value is `false`.

For security reasons, it is highly recommended to leave Privileged set to false since almost no dogu should gain retrospective container insights.

Example:

- false

### Configuration

Configuration contains a list of [ConfigurationField](#type-configurationfield). This field is optional.

It describes generic properties of the dogu in the Cloudogu EcoSystem registry.

Examples:

```
{
  "Name": "plugin_center_url",
  "Description": "URL of SCM-Manager Plugin Center",
  "Optional": true
}

{
  "Name": "logging/root",
  "Description": "Set the root log level to one of ERROR, WARN, INFO, DEBUG or TRACE. Default is INFO",
  "Optional": true,
  "Default": "INFO",
  "Validation": {
    "Type": "ONE_OF",
    "Values": [
      "WARN",
      "ERROR",
      "INFO",
      "DEBUG",
      "TRACE"
    ]
  }
}
```

### Properties

Properties is a `map[string]string` of Properties. This field is optional. It describes generic properties of the dogu 
which are evaluated by a client like cesapp or k8s-dogu-operator.

Example:

```
{
  "key1": "value1",
  "key2": "value2"
}
```

### EnvironmentVariables

EnvironmentVariables contains a list of [EnvironmentVariable](#type-environmentvariable) that should be set in the container. This field is optional.

Example:

```
[
  {"Key": "my_key", "Value": "my_value"}
]
```

### Dependencies

Dependencies contains a list of [Dependency](#type-dependency). This field is optional.

This field defines dependencies that must be fulfilled during the dependency check. If the dependency cannot be fulfilled during the check an error will be thrown and the processing will be stopped.

Examples:

```
[
  {
    "type": "dogu",
    "name": "postgresql"
  },
  {
    "type": "client",
    "name": "cesapp",
    "version": ">=6.0.0"
  },
  {
    "type": "package",
    "name": "cesappd",
    "version": ">=3.2.0"
  }
]
```

### OptionalDependencies

OptionalDependencies contains a list of [Dependency](#type-dependency). This field is optional.

In contrast to core.Dependencies, OptionalDependencies allows to define dependencies that may be fulfilled if they are existent. There is no negative impact during the dependency check if an optional dependency does not exist. But if the optional dependency is present and the dependency cannot be fulfilled during the dependency check then an error will be thrown and the processing will be stopped.

Examples:

```
[
  {
    "type": "dogu",
    "name": "ldap"
  }
]
```

## type [DoguJsonV2FormatProvider](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L1211>)

DoguJsonV2FormatProvider provides methods to format Dogu results compatible to v2 API.

```go
type DoguJsonV2FormatProvider struct{}
```

## type [EnvironmentVariable](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L280-L283>)

EnvironmentVariable struct represents custom parameters that can change the behaviour of a dogu build process

```go
type EnvironmentVariable struct {
    Key   string
    Value string
}
```



## type [ExposedCommand](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L243-L276>)

ExposedCommand struct represents a command which can be executed inside the dogu

```go
type ExposedCommand struct {
    Name string

    Description string

    Command string
}
```

### Name

Name identifies the exposed command. This field is mandatory. It must be unique in all commands of the same dogu.

The name must consist of:

- lower case latin characters
- special characters underscore "_", minus "-"
- ciphers 0-9

Examples:

- service-account-create
- plugin-delete

### Description

Description describes the exposed command in a few words. This field is optional.

Example:

- Creates a new service account
- Delete a plugin

### Command

Command identifies the script to be executed for this command. This field is mandatory.

The name syntax must comply with the file system syntax of the respective host operating system and is encouraged to consist of:

- lower case latin characters
- special characters underscore "_", minus "-"
- ciphers 0-9

Examples:

- /resources/create-sa.sh
- /resources/deletePlugin.sh

## type [ExposedPort](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L204-L231>)

ExposedPort struct is used to define ports which are exported to the host.

Example:

```
{ "Type": "tcp", "Container": "2222", "Host":"2222" }
{ "Container": "2222", "Host":"2222" }
```

```go
type ExposedPort struct {
    Type string

    Container int

    Host int
}
```

### Type

Type contains the protocol type over which the container communicates (f. i. 'tcp'). This field is optional (the value of `tcp` is then assumed).

Example:

- tcp
- udp
- sctp

### Container

Container contains the mapped port on side of the container. This field is mandatory. Usual port range limitations apply.

Examples:

- 80
- 8080
- 65535

### Host

Host contains the mapped port on side of the host. This field is mandatory. Usual port range limitations apply.

Examples:

- 80
- 8080
- 65535

## type [HealthCheck](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L171-L196>)

HealthCheck provide readiness and health checks for the dogu container.

```go
type HealthCheck struct {
    Type string

    State string

    Port int

    Path string

    Parameters map[string]string
}
```

### Type

Type specifies the nature of the health check. This field is mandatory. It can be either tcp, http or state.

For Type "tcp" the given Port needs to be open to be healthy.

For Type "http" the service needs to return a status code between >= 200 and < 300 to be healthy. Port and Path are used to reach the service.

For Type "state" the /state/<dogu> key in the Cloudogu EcoSystem registry gets checked. This key is written by the installation process. A 'ready' value in the CES-registry means that the dogu is healthy.

### State

State contains the expected health check state of Type state. This field is optional, even for health checks of the type "state". The default is "ready".

### Port

Port is the tcp-port for health checks of Type tcp and http. This field is mandatory for the health check of type "tcp" and "http"

### Path

Path is the Http-Path for health checks of Type "http". This field is mandatory for the health check of type "http". The default is '/health'.

### Parameters

Parameters may contain key-value pairs for check specific parameters.

Deprecated: is not in use.



## type [ServiceAccount](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L299-L317>)

ServiceAccount struct can be used to get access to another dogu.

Example:

```
{
 "Type": "k8s-dogu-operator",
 "Kind": "k8s"
}
```

```go
type ServiceAccount struct {
    Type string

    Params []string

    Kind string `json:"Kind,omitempty"`
}
```

### Type

Type contains the name of the service on which the account should be created. This field is mandatory.

Example:

- postgresql
- your-dogu

### Params

Params contains additional arguments necessary for the service account creation. The optionality of this field depends on the desired service account producer dogu. Please consult the service account producer dogu in question.

### Kind

Kind defines the kind of service on which the account should be created, e.g. `dogu` or `k8s`. This field is optional. If empty, a default value of `dogu` should be assumed.

Reading this property and creating a corresponding service account is up to the client.

## type [ValidationDescriptor](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L398-L411>)

ValidationDescriptor describes how to determine if a config value is valid.

```go
type ValidationDescriptor struct {
    Type string

    Values []string
}
```

### Type

Type contains the name of the config value validator. This field is mandatory. Valid types are:

- ONE_OF
- BINARY_MEASUREMENT
- FLOAT_PERCENTAGE_HUNDRED

### Values

Values may contain values that aid the selected validator. The values may or may not be optional, depending on the Type being used. It is up to the selected validator whether this field is mandatory, optional, or unused.

## type [Volume](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L74-L136>)

Volume defines container volumes that are created during the dogu creation or upgrade.

Examples:

- { "Name": "data", "Path":"/usr/share/yourtool/data", "Owner":"1000", "Group":"1000", "NeedsBackup": true}
- { "Name": "temp", "Path":"/tmp", "Owner":"1000", "Group":"1000", "NeedsBackup": false}

```go
type Volume struct {
    Name string

    Path string

    Owner string

    Group string

    NeedsBackup bool

    Clients []VolumeClient `json:"Clients,omitempty"`
}
```

### Name

Name identifies the volume. This field is mandatory. It must be unique in all volumes of the same dogu.

The name syntax must comply with the file system syntax of the respective host operating system and is encouraged to consist of:

- lower case latin characters
- special characters underscore "_", minus "-"
- ciphers 0-9

The name must not be "_private" to avoid conflicts with the dogu's private key.

Examples:

- tooldata
- tool-data-0

### Path

Path to the directory or file where the volume will be mounted inside the dogu. This field is mandatory.

Path may consist of several directory levels, delimited by a forward slash "/". Path must comply with the file system syntax of the container operating system.

The path must not match `/private` to avoid conflicts with the dogu's private key.

Examples:

- /usr/share/yourtool
- /tmp
- /usr/share/license.txt

### Owner

Owner contains the numeric Unix UID of the user owning this volume. This field is optional.

For security reasons it is strongly recommended to set the Owner of the volume to an unprivileged user. Please note that container image must be then built in a way that the container process may own the path either by user or group ownership.

The owner syntax must consist of ciphers (0-9) only.

Examples:

- "1000" - an unprivileged user
- "0" - the root user

### Group

Group contains the numeric Unix GID of the group owning this volume. This field is optional.

For security reasons it is strongly recommended to set the Group of the volume to an unprivileged Group. Please note that container image must be then built in a way that the container process may own the path either by user or group ownership.

The Group syntax must consist of ciphers (0-9) only.

Examples:

- "1000" - an unprivileged group
- "0" - the root group

### NeedsBackup

NeedsBackup controls whether the Cloudogu EcoSystem backup facility backs up the whole the volume or not. This field is optional. If unset, a value of `false` will be assumed.

### Clients

Clients contains a list of client-specific (t. i., the client that interprets the dogu.json) configurations for the volume. This field is optional.

## type [VolumeClient](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L58-L67>)

VolumeClient adds additional information for clients to create volumes.

Example:

```
{
  "Name": "k8s-dogu-operator",
  "Params": {
    "Type": "configmap",
    "Content": {
      "Name": "k8s-ces-menu-json"
    }
  }
}
```

```go
type VolumeClient struct {
    Name string

    Params interface{}
}
```

### Name

Name identifies the client responsible to process this volume definition. This field is mandatory.

Examples:

- cesapp
- k8s-dogu-operator

### Params

Params contains generic data only interpretable by the client. This field is mandatory.

