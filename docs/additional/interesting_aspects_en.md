# Interesting aspects for developers

This chapter describes everything that goes beyond general or specific Dogu development, but helps new developers get started or gives ideas for broadening horizons.

## Containers

Building container images is central to Dogu development.
The following sections address common best practices.
These best practices come both from general container and Dogu development.

### 12-Factor App

The term ["12 Factor App"][12-factor] refers to a collection of methods for building software-as-a-service applications.
These methods aim to make applications portable and fail-safe.
In doing so, differences between **development environment and execution environment** should be minimized to the extent that production problems can be more easily understood and managed.

The 12 factors are as follows:
1. **Codebase:** One codebase tracked in revision control, many deploys
2. **Dependencies:** Explicitly declare and isolate dependencies
3. **Config:** Store config in the environment
4. **Backing services:** Treat backing services as attached resources
5. **Build, release, run:** Strictly separate build and run stages
6. **Processes:** Execute the app as one or more stateless processes
7. **Port binding:** Export services via port binding
8. **Concurrency:** Scale out via the process model
9. **Disposability:** Maximize robustness with fast startup and graceful shutdown
10. **Dev/prod parity:** Keep development, staging, and production as similar as possible
11. **Logs:** Treat logs as event streams
12. **Admin processes:** Run admin/management tasks as one-off processes

[12-factor]: https://12factor.net/

### Container images

When building images, the following aspects should be considered:
- build [root-less containers][rootless-container] if possible
- if possible use current version of all used [tools][container-tools] / [base-images][base-images] or commits
  - Attention: New alpine software version possible by increasing the base image version!
  - Update the tools of the base-image
    - e.g. `apk update && apk upgrade`
- keep the image size small
  - compared to smaller images, large images consume more time during the image download and container instantiation and impede the time of deployment
  - it's a good idea to remove unnecessary files or packages, or even avoid their installiation in the first place
- focus on fast (re-)build times
  - as [few statements][minimize-number-of-layers] as possible in the Dockerfile
  - **one** [COPY statement][copy-statement] for container file system only
  - [Multi-Stage-Builds][multistage-build] may help to reduce the number of image layers
- use LABELs for metadata
  - `LABEL maintainer="hello@cloudogu.com"` instead of `MAINTAINER` statement
  - `NAME="namespace/dogu-name"` pure
  - `VERSION ="w.x.y-z"`, will be taken over by the automatic release process
- the Dockerfile contains a [healthcheck][healthcheck]
  - e.g.: `HEALTHCHECK CMD doguctl healthy nexus || exit 1`
- downloads (with curl/wget or similar) are checked with checksums/hashes
  - verifying downloaded artifacts increases the security of later builds in case an attacker replaced a file with malware
- Dogu startup must be done via a fixed startup script (e.g. [`startup.sh`][startup-sh])
  - Dogus do not accept external commands or parameters via the Docker CLI

[rootless-container]: https://docs.docker.com/engine/security/rootless/
[minimize-number-of-layers]: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#minimize-the-number-of-layers
[multistage-build]: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#use-multi-stage-builds
[container-tools]: https://github.com/cloudogu/base/blob/3466b5e95c25a6c5ac569069167e513c71815797/Dockerfile#L10
[base-images]: https://github.com/cloudogu/sonar/blob/8e389605d1f2fa7720d725a1cca6692f4c6b77e3/Dockerfile#L1
[healthcheck]: https://github.com/cloudogu/sonar/blob/8e389605d1f2fa7720d725a1cca6692f4c6b77e3/Dockerfile#L3
[copy-statement]: https://github.com/cloudogu/sonar/blob/8e389605d1f2fa7720d725a1cca6692f4c6b77e3/Dockerfile#L33

## Resources folder & Dogu scripts

For Dogu development at Cloudogu, we use a `resources` folder.
We also recommend other Dogu development teams to use this structure.
However, this decision is up to the team itself.

This contains all the important resources that a Dogu needs at runtime.

The complete folder is copied to the root directory of the dogu container in the Dockerfile.
Therefore, the structure of the resources folder and the container file system is identical.

Example:
- `resources/`
  - [`create-sa.sh` (optional)][create-service-account]
  - [`remove-sa.sh` (optional)][delete-service-account]
  - [`post-upgrade.sh` (optional)][post-upgrade]
  - [`pre-upgrade.sh` (optional)][pre-upgrade]
  - [`upgrade-notification.sh` (optional)][upgrade-notification]
  - [`startup.sh` (encouraged)][startup-sh]

[create-service-account]: ../important/relevant_functionalities_de.md#service-account-anlegen
[delete-service-account]: ../important/relevant_functionalities_de.md#service-account-löschen
[post-upgrade]: ../important/relevant_functionalities_de.md#post-upgrade---führt-alle-aktionen-nach-dem-upgrade-des-dogus-durch
[pre-upgrade]: ../important/relevant_functionalities_de.md#pre-upgrade---führt-alle-aktionen-vor-dem-upgrade-des-dogus-durch
[upgrade-notification]: ../important/relevant_functionalities_de.md#upgrade-notification---zeigt-eine-benachrichtigung-vor-der-upgradebestätigung-eines-dogus
[startup-sh]: ../important/relevant_functionalities_de.md#aufbau-und-best-practices-von-startupsh

### Bash scripts

The creation of bash scripts in a dogu is necessary to control various processes.
This includes starting and upgrading the dogu, as well as [creating a service account][create-service-account].

In general, all scripts should conform to the following conventions:

#### bash-strict-mode

Bash scripts should always use [bash-strict-mode][strict-mode].
This prevents further execution of the script in the event of a misbehavior.
In general, it makes sense to catch all potential errors in the startup script and report them with a separate error method.
This could look like this:

```shell
# This function prints an error to the console and waits 5 minutes before exiting the process.
function exitOnErrorWithMessage() {
  message=${1}
  echo -e "ERROR: ${message}. Exiting in 300 seconds."
  sleep 300
  exit 1
}
```

Such a method makes the whole script more readable and slows down the restart loop of the Dogu as well as the mass of generated logs.

[strict-mode]: http://redsymbol.net/articles/unofficial-bash-strict-mode/

#### Line Breaks

Unix-based line breaks should be used for all scripts at all times (`\n`).

#### Use doguctl

Doguctl is a command line tool to simplify the configuration of a dogu.
Info on its functionalities can be found in ["Usage of doguctl"][doguctl-usage].

Where applicable, always use doguctl in bash scripts like `startup.sh`.
Examples of reasonable usage can be found in the [Nexus startup script][doguctl-example].

[doguctl-usage]: ../important/relevant_functionalities_en.md#usage-of-doguctl
[doguctl-example]: https://github.com/cloudogu/nexus/blob/4d2de3733eca684df1363179c527363a4d31526e/resources/startup.sh

## Fast feedback cycles

In development, rapid insight into whether the right development path has been followed is important.
Fast feedback is even more important the more complex the environment becomes.

In Dogu development, rapid feedback can be achieved in a number of ways:

### Multi-Stage Build

For effective development, the use of a multi-stage build for the Dogu image is recommended.
This offers advantages such as more effective caching of downloads, faster build times, effective layering and small image sizes.
More information on multi-stage build is available on the [official Docker website][docker-multi-stage].

[docker-multi-stage]: https://docs.docker.com/develop/develop-images/multistage-build/

### Development environment without a Cloudogu EcoSystem

Building the Dogu and delivering it to the local EcoSystem not only takes a lot of time, but also takes away the ability to debug the software effectively.
Therefore, it makes sense to build a local development environment independent of the CES to effectively run, test and debug the Dogu software.

The local development environment must provide all dependent CES services through locally started alternatives.
We recommend that a local development environment is defined in the form of a `docker-compose.yml` file and can be easily started at any time with `docker-compose up` and shut down with `docker-compose down`.
For an example [see CAS][local-cas-example].

[local-cas-example]: https://github.com/cloudogu/cas/blob/develop/app/docker-compose.yml

## Quality Assurance for Dogus

### Container Validation

Container validation assures the correct configuration of the container, which is necessary for the smooth execution of the software.

For this purpose, we use the server validation program [goss][goss] in our Dogus.
The actual goss tests are created in the following data structure in the dogu:
- `<root directory of the Dogu>/`
  - `spec/`
    - `goss/`
      - `goss.yml` (contains all goss tests)

The goss tests should ideally check all important aspects for the smooth start of a dogu.
This includes aspects like:
- Checking permission for scripts (startup.sh), volumes.
- Checking the UID & GID for files (startup.sh, resources, volumes, write-dependent paths)
- Readiness of the software: TCP health checks

For pipeline integration, we recommend using the `ecoSystem.verify()` method from the [dogu-build-lib][dogu-build-lib].
An example call looks like the following in a Jenkinsfile:

```groovy
stage('Verify') {
    ecoSystem.verify("/dogu")
}
```

This internally calls the `cesapp verify` command, which runs the goss tests.

[goss]: https://github.com/goss-org/goss
[dogu-build-lib]: https://github.com/cloudogu/dogu-build-lib

## Documentation

Every tool should have a reasonable documentation about used features, which supports usage or support.
To allow easy integration into our existing documentation infrastructure, the documentation (except Readme and changelog) should be placed in a `/docs` folder located in the repository root.

By default, the docs include the following:
- Features of the Dogu
- Installation of the Dogu
- Configuration of the Dogu
- Development on the Dogu
- System requirements of the Dogu (nice to have)
- Readme
- Changelog

### License

The license for using the Dogu should be clearly stated.
This can be specified in the Readme, for example.
In case of a public repo, the license file should be included in the repository root.

### Readme

A Readme is a file consisting of diverse and often important information about the given software.
It is useful both for your own developments and in a partner context with external developers.
In the case of public repositories, however, a higher standard should be applied to a Readme.

A Readme file is located in the repository root and concisely describes notes on the general purpose of the Dogu.
For users this can be some sorts of quickstart guide of how to use the Dogu, or even a feature list.
For Dogu developers the readme may provide insights of how collaboration is possible or which copyright license was chosen.
Additional docs can be linked to provide further information for both users and developers.

Administrators may want to find information about the resource consumption of this dogu.
Finally, the readme should clarify who is responsible for this Dogu.

### Changelog

Changelogs are written for people who want to understand what's new in releases.

The format of the changelog should follow **[keep a changelog](https://keepachangelog.com)**.

New changes are always added to the top section `## [Unreleased]`.
When a new release is made, these changes move to a new section containing the corresponding version number.

For all changes, references to the operations of the issue tracker used should be added.
In addition, if more complex procedures are involved, a reference to documentation is useful.

### System requirements

The system requirements of a Dogu (CPU, RAM, storage) can either be recorded in the Dogu's documentation
or should be otherwise communicated to Cloudogu directly (e.g. in case of a private repository).

## Release of a Dogus

### Cloudogu account and permissions

First, you need to create an account on https://account.cloudogu.com.
After that, just send us a request with your account name / email address and the Dogu namespace you want (e.g. _yourcompany_).
We will create the namespace and grant your account permission to push to the namespace.

This step is necessary only once per account.

### Release process

Now you can log in with your Cloudogu account via `cesapp login`.
The Dogu can then be easily pushed to the Dogu registry using `cesapp push`.

We recommend to perform these steps automatically in the CI/CD process for your production release branch.

## Dogu checklist

Use this checklist to determine if your Dogu meets all the requirements to be published.

### Required components

| Component                     | Description                                                                                                                               |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| **[dogu.json][dogujson]**     |                                                                                                                                           |
| Name                          | The dogu.json contains a Name field. See [name][doguname]                                                                                 |
| Version                       | The dogu.json contains a Version field. See [Version][doguversion]                                                                        |
| Image                         | The dogu.json contains an Image field. See [Image][doguimage]                                                                             |
| DisplayName                   | The dogu.json contains a DisplayName field. See [DisplayName][dogudisplayname]                                                            |
| Description                   | The dogu.json contains a Description field. See [Description][dogudescription]                                                            |
| Category                      | The dogu.json contains a Category field. See [Category][dogucategory]                                                                     |
|                               |                                                                                                                                           |
| **[Dockerfile][dockerfile]**  |                                                                                                                                           |
| Healthcheck                   | The Dockerfile contains a healthcheck. See [HealthChecks][healthchecks]                                                                   |
| MAINTAINER label              | The Dockerfile contains a label "MAINTAINER". See [Dockerfile][dockerfile]                                                                |
|                               |                                                                                                                                           |
| **Central Authentication**    |                                                                                                                                           |
| Single Sign-On (SSO)          | The Dogu handles centralized login via SSO. See [authentication] [authentication]                                                         |
| Single Logout (SLO)           | The Dogu masters central logout via SLO. See [authentication][authentication]                                                             |
|                               |                                                                                                                                           |
| **Dogu behavior**             |                                                                                                                                           |
| CES admin group replication   | CES admin group permissions are replicated to the Dogu. See [Admin group changeability][admingroup]                                       |
| CES admin group changeability | If the CES admin group is changed, the Dogu adapts to it. See [Admin group changeability][admingroup]                                     |
| CES FQDN changeability        | If the CES FQDN is changed, the Dogu adapts to it. See [Changeability of FQDN][fqdnchange]                                                |
| Backup/Restore Capability     | All production data of the Dogus should be stored inside volumes for backup and restore. See [Backup & Restore Capability][backuprestore] |
| Upgrade Capability            | Upgrade steps between Dogu versions are automated by script. See [Dogu upgrades][doguupgrades]                                            |
| Memory Limits                 | The memory usage of the Dogu container can be set. See [Memory/swap limit][memorylimit]                                                   |
| Logging behavior              | The amount and verbosity of log data can be set. See [Control logging behavior][logging]                                                  |


### Recommended components

| Component                          | Description                                                                                                                                               |
|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| startup.sh                         | A script is used to start the Dogu. See [startup.sh][startup_sh]                                                                                          |
| changelog                          | Changes to the dogu are noted in the changelog. See [CHANGELOG.md][changelog_md]                                                                          |
| Best Practices for Dogu containers | Best practices for container building are followed. See [container images][containerimages]                                                               |
| Best Practices for scripts         | Best practices for scripts are followed. See [resources folder & dogu scripts][resourcesdir]                                                              |
| container validation               | Finished containers are validated with `goss`. See [Container Validation][containervalidation]                                                            |
| Documentation                      | All features and configuration options of Dogus are documented. See [documentation][documentation] and [Cloudogu documentation rules][documentationrules] |
| Code Quality Assurance, Tests      | Code quality should be assured by various (automated) test, lint and review operations. See [quality assurance][qualitycontrol]                           |
| Issue Tracking                     | Bugs and improvement suggestions should be able to be submitted and tracked. See [issue tracking][issuetracking]                                          |
| Telemetry configurable             | If the Dogu collects telemetry data, this feature should be disengageable.                                                                                |

[dockerfile]: ../core/basics_en.md#3-dockerfile
[authentication]: ../important/relevant_functionalities_en.md#authentication
[startup_sh]: ../important/relevant_functionalities_en.md#structure-and-best-practices-of-startupsh
[changelog_md]: ../additional/interesting_aspects_en.md#changelog
[admingroup]: ../important/relevant_functionalities_en.md#changeability-of-the-admin-group
[dogujson]: ../core/basics_en.md#4-dogujson
[doguname]: ../core/compendium_en.md#name-2
[doguversion]: ../core/compendium_en.md#version
[fqdnchange]: ../important/relevant_functionalities_en.md#changeability-of-the-fqdn
[dogudisplayname]: ../core/compendium_en.md#displayname
[doguimage]: ../core/compendium_en.md#image
[dogudescription]: ../core/compendium_en.md#description-1
[dogucategory]: ../core/compendium_en.md#category
[healthchecks]: ../core/compendium_en.md#healthchecks
[backuprestore]: ../important/relevant_functionalities_en.md#backup--restore-capability
[doguupgrades]: ../important/relevant_functionalities_en.md#dogu-upgrades
[memorylimit]: ../important/relevant_functionalities_en.md#memoryswap-limit
[logging]: ../important/relevant_functionalities_en.md#controlling-the-logging-behavior
[containerimages]: ../additional/interesting_aspects_en.md#container-images
[resourcesdir]: ../additional/interesting_aspects_en.md#resources-folder--dogu-scripts
[containervalidation]: ../additional/interesting_aspects_en.md#container-validation
[documentation]: ../additional/interesting_aspects_en.md#documentation
[documentationrules]: ../additional/internal_aspects_en.md#cloudogu-documentation-rulebook
[qualitycontrol]: ../additional/internal_aspects_en.md#quality-assurance
[issuetracking]: ../additional/internal_aspects_en.md#issue-tracking