# Interesting aspects for developers

This chapter describes everything that goes beyond general or specific Dogu development, but helps new developers get started or gives ideas for broadening horizons.

## Containers

Building container images is central to Dogu development.  
The following sections address common best practices.  
These best practices come from both general container and Dogu development.

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
- build root-less containers if possible
- as few statements as possible in the Dockerfile
- use LABELs for metadata
  - `LABEL maintainer="hello@cloudogu.com"`
  - `NAME="namespace/dogu-name"` pure
  - `VERSION ="w.x.y-z"`, will be taken over by the automatic release process
- the Dockerfile contains a healthcheck
- downloads (with curl/wget or similar) are checked with checksums/hashes

## Resources folder & Dogu scripts

For Dogu development at Cloudogu, we use a `resources` folder.
We also recommend other Dogu development teams to use this structure.
However, this decision is up to the team itself.

This contains all the important resources that a Dogu needs at runtime.
The folder is built using the same structure as the container file system.

Example:
- `resources/`
  - `create-sa.sh` (optional)
  - `remove-sa.sh` (optional)
  - `post-upgrade.sh` (optional)
  - `pre-upgrade.sh` (optional)
  - `upgrade-notification.sh` (optional)
  - `startup.sh` (encouraged)

This example shows the contents of a resource folder of a Dogu.

The complete folder is copied to the root directory of the dogu container in the Dockerfile.  
This would overwrite an existing configuration of the app in the path `/etc/your_app/config.json`.  
Using this approach, it is possible to customize all files of the container.

### Bash scripts

The creation of bash scripts in a dogu is necessary to control various processes.  
This includes starting and upgrading the dogu, as well as creating a service account.

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

#### Line-Endings

Unix-based line endings should be used for all scripts at all times.

### Dogu startup script: startup.sh

The [dogu-startup-script][startup-sh] is the entry point of a dogu container and manages all the steps necessary to start the dogu smoothly.
Conventionally, the startup script is created with the name `startup.sh` in the `resources` folder of the Dogu.

[startup-sh]: ../important/relevant_functionalities_en.md#structure-and-best-practices-of-startupsh-

## Fast feedback cycles through tool development without CES.

In development, rapid insight into whether the right development path has been followed is important.  
Fast feedback is even more important the more complex the environment becomes.

In Dogu development, rapid feedback can be achieved in a number of ways:

### Multi-Stage Build

For effective development, the use of a multi-stage build for the Dogu image is recommended.
This offers advantages such as more effective caching of downloads, faster build times, effective layering and small image sizes.
More information on multi-stage build is available on the [official Docker website][docker-multi-stage].

[docker-multi-stage]: https://docs.docker.com/develop/develop-images/multistage-build/

### CES-free development environment.

Building the Dogu and delivering it to the local EcoSystem not only takes a lot of time, but also takes away the ability to debug the software effectively.  
Therefore, it makes sense to build a local development environment independent of the CES to effectively run, test and debug the Dogu software.

The local development environment must provide all dependent CES services through locally started alternatives.
We recommend that a local development environment is defined in the form of a `docker-compose.yml` file and can be easily started at any time with `docker-compose up` and shut down with `docker-compose down`.  
For an example [see CAS][local-cas-example].

[local-cas-example]: https://github.com/cloudogu/cas/blob/develop/app/docker-compose.yml

## Quality Assurance for Dogus

### Container Validation (GOSS)

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
- Accessibility of the software: TCP health checks

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
To allow easy integration into our existing documentation infrastructure, the documentation should be placed in a `/docs` folder located in the repository root.

By default, the docs include the following:
- Features of the Dogu
- Installation of the Dogu
- Configuration of the Dogu
- Development on the Dogu
- Readme
- Changelog

### License

The license for using the Dogu should be clearly stated.
This can be specified in the Readme, for example.

### Readme

A Readme is useful both for your own developments and in a partner context with external developers.  
In the case of public repositories, however, a higher standard should be applied to a Readme.

A Readme file is located in the repository root and concisely describes notes on the general purpose of the Dogu.
This can be in the form of a quickstart guide, for example, describing how to set up a local development environment.
It should also include a reference to the documentation folder and can also refer to specific pages of the documentation.
Finally, it is recommended to specify required resources for use and a person responsible for the Dogu.

### Changelog

Changelogs are written for people who want to understand what's new in releases.

The format of the changelog should follow **[keep a changelog](https://keepachangelog.com)**.

New changes are always added to the top section `## [Unreleased]`.  
When a new release is made, these changes move to a new section containing the corresponding version number.

For all changes, references to the operations of the issue tracker used should be added.  
In addition, if more complex procedures are involved, a reference to documentation is useful.
