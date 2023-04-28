# Internal aspects

This section describes aspects that must be observed by internal developers of Cloudogu GmbH.
They contain requirements for quality assurance or, for example, basic definitions to a finished, final Dogu.
Thus, it results that this information is not necessarily relevant for external developers.
However, the aspects can be used to define or improve quality standards.

## Quality assurance

Dogus can consist both of self-developed software and of encapsulated foreign software.
In the case of self-development, common quality assurance standards should apply, which are mentioned in the following
sections below.
When a Dogu captures third-party software, it is often not practical to test all features of the actual
tool to be tested.
The test focus is then delineated to the added Dogu features, which typically represent the added benefit of a Dogu's
represent.

### Quality assurance of the tool

Quality assurance is the responsibility of the owner.
For our software, we implement the following aspects for quality assurance:

- Unit tests (80% code coverage)
- Integration tests
- End-to-end tests
- Tools for static analysis e.g. Reviewdog, Linter (style, smells)
- SonarQube (code coverage, smells, bugs, security vulnerabilities)
- Reviews according to 4-eyes principle

### Quality assurance of the Dogus

In order to ensure a flawless integration of the Dogus into the Cloudogu EcoSystem, the quality of the Dogus should be 
guaranteed through be ensured by various methods.
For this purpose, the Dogus are tested in two different phases.

#### Automation through Makefiles

With the help of Makefiles, processes can be standardized. As support, we offer a collection of
[makefiles][makefiles].

[makefiles]: https://github.com/cloudogu/makefiles

#### Automation through CI/CD

The majority of features under test should be tested automatically in a build pipeline.
Successfully passing all CI/CD tests are a prerequisite for a release.
As support, we offer our [ces-build-lib][ces-build-lib] and [dogu-build-lib][dogu-build-lib].
Both libraries are effectively used in all CI/CD pipelines of our Dogus.
It should be noted that the `dogu-build-lib` is dependent on the Google Cloud.
It is able to provision instances of the Cloudogu EcoSystem in the cloud so that they can be used for testing.

In the following, we describe the different methods that can be used in a pipeline for testing.
We specifically address the support for our libraries.

[ces-build-lib]: https://github.com/cloudogu/ces-build-lib
[dogu-build-lib]: https://github.com/cloudogu/dogu-build-lib

##### Shell tests & linting

With the development of Dogus, several bash scripts are likely to be created.
These should be tested and checked for smells.
For linting, the use of the `ShellCheck()` method from the `dogu-build-lib` is recommended.
An example call looks like the following in a Jenkinsfile:

```groovy
stage('shellcheck') {
    shellCheck("resources/startup.sh")
}
```

For testing, [bats][bats] from the `ces-build-lib` can be used:

```groovy
Docker docker = new Docker(this)

stage('Bats Tests') {
    Bats bats = new Bats(this, docker)
    bats.checkAndExecuteTests()
}
```

[bats]: https://github.com/bats-core/bats-core

##### Dockerfile linting

The Dockerfile should be built according to best practices and, of course, should be syntactically correct.
For this, the use of the `lintDockerfile()` method from the `ces-build-lib` is recommended.
An example call looks like this in a Jenkinsfile:

```groovy
stage('lint') {
    lintDockerfile()
}
```

##### End-to-End Tests (UI, API, CAS Plugin)

The intent of end-to-end testing is to test the added value of the Dogus.
For example, one refers to the integration of the CAS Dogus:

- Front-channel login
- Back-channel login
- Front-channel logout
- Back-channel logout
- Changing the admin group

The `dogu-build-lib` provides several methods to execute this.
A call in the Jenkinsfile looks like this:

```groovy
stage('Integration tests') {
    ecoSystem.runCypressIntegrationTests()
}
```

The tests themselves must be located under the Dogu directory in the `integrationTests` folder.
There is also a [library][dogu-integration-lib] which is used to support the implementation of the tests.

[dogu-integration-lib]: https://github.com/cloudogu/dogu-integration-test-library

##### Trivy

It is important to identify and fix the security vulnerabilities in a Docker image, or the target system.
For this, the Trivy scanner is useful, which identifies and prioritizes security vulnerabilities in the system.
The `dogu-build-lib` provides a [component][trivy] for this purpose, which can be used as follows to scan the image of a
Dogu.

```groovy
EcoSystem ecoSystem = new EcoSystem(this, "gcloud-credentials-id", "ssh-credentials-id")
Trivy trivy = new Trivy(this, ecoSystem)

stage('Trivy scan') {
    trivy.scanDogu("/dogu", TrivyScanFormat.HTML, TrivyScanLevel.CRITICAL, TrivyScanStrategy.FAIL)
}
```

[trivy]: https://github.com/cloudogu/dogu-build-lib/blob/develop/docs/development/trivy_de.md

#### Manual testing on a test stage

In the manual testing phase, the Dogu should be tested in a staging environment.
This staging environment should represent a production environment as realisitcally as possible.
The Dogu should be installed using the `cesapp install` command.
In addition, it should be systematically tested manually in the browser and classified as functional.
The focus should be on the features that have the greatest relevance in production.

## Definition of Done

This section describes possible requirements for a final state of a development.
It creates a common understanding for developers when a feature is done and can include various requirements.

The following points describe basic requirements that can serve as the basis for a Definition of Done.

### Issue tracking

- An issue exists in a ticket system for every change.
- It describes the product increment to be developed.
- Tickets in public ticket systems are linked in internal systems.

### Dockerfile

Requirements for the Dockerfile can be found [here](interesting_aspects_en.md#container-images).

### Dogu.json

- Must contain configurable keys:
    - `container_config/memory_limit`
    - `container_config/swap_limit`

### Dogu scripting

Requirements for shell scripting can be found [here](interesting_aspects_en.md#bash-scripts).

### Dogu functionalities

- Consider backup & restore capability
- Consider upgrade capability
    - Upgrade scripts must be designed in such a way that it is possible to upgrade from all old versions of the Dogu
      (as long as this is not prevented by the software used in the Dogu).
    - Upgrade of Dogus from previous version must be tested.
- Admin group
    - Admin group must be changeable.
    - Admin group from the `etcd` is used and if the admin group is changed the Dogu reacts accordingly
      to it, then the old admin group will be stripped of admin rights. See e.g. SonarQube and Nexus Repository
      manager.
- Logging
    - Check log output of the Dogu for errors.
- Accepting a change of the CES FQDN
    - An FQDN change must be processed by a Dogu in a meaningful way.
    - The Dogu must start properly after a restart.
- Consider single sign-on & single sign-out

### Documentation

- Update Dogu documentation and check if it complies with the [documentation ruleset](#cloudogu-documentation-rulebook).
    - If there are any discrepancies, rework the structure accordingly. Leave the existing structures in place, and use 
    the files link to the new `docs` folder from the files stored there.
    - The old structures will be deleted after a reasonable time (1 year).

## Cloudogu documentation rulebook

The rules are intended to standardize the documentation for Cloudogu and to record decisions on how certain topics
are to be handled. The following points form a basis and can be supplemented by other aspects such as product notations,
a documentation plan or a style guide.

### Structure of documentation

The technical documentation is located in the `docs` folder in the respective repository.
In addition to the `docs` folder, each repository contains a `CHANGELOG` as well as a `README`.
Overarching instructions ("guides") do not need to be placed in the `docs` folder.

#### README.md

See [README](interesting_aspects_en.md#readme)

#### CHANGELOG

See [CHANGELOG](interesting_aspects_en.md#changelog)

#### The docs folder

The `docs` folder is structured according to this exemplary scheme:

```
/docs  
├── getting_started_en.md  
├── getting_started_en.md  
├── /gui  
│ ├── backup_button_en.md  
│ ├── backup_button_en.md  
│ ├── restore_button_en.md  
│ └── restore_button_en.md  
├── /operations  
│ ├── show_graph_feature_en.md  
│ ├── show_graph_feature_en.md  
│ ├── get_function_en.md  
│ ├── get_function_en.md  
│ ├── remove_function_en.md  
│ ├── remove_function_en.md  
│ ├── push_command_en.md  
│ ├── push_command_en.md  
│ ├── pull_command_en.md  
│ ├── pull_command_en.md  
│ ├── groups_endpoint_en.md  
│ ├── groups_endpoint_en.md  
│ ├── users_endpoint_en.md  
│ └── users_endpoint_en.md  
└── /development  
└── how_to_create_an_image.md  
└── how_to_build_a_plugin.md
```

(The names are chosen as examples and are not defaults).
The naming scheme is the name of the feature plus a suffix with the language version (de, en).
Single words in the name are connected with underscores (e.g. name_of_feature_en or name_of_feature_de).

#### operations folder

The operations folder is the core of the documentation. Every single feature, every function, every command is described
in three sections:

- What? What feature does the section describe? What is the general purpose?
- How? Detailed description of any options, flags, or settings.
- Why? Backgrounds, placement in the overall system, limitations of the implementation.

All contents in the folder 'operations' are available in German and English. The translation can be done automatically.

#### gui folder

The folder `gui` is optional for repositories that include a graphical user interface.
In this folder all elements of the graphical interface are explained.
The structure of the descriptions is equivalent to `operations`.

#### development folder

The folder `development` is optional for repositories that need further hints for the development, e.g. of plugins,
are needed.
The descriptions in this folder are written in English only.

#### getting_started files

In the `docs` folder, the files `getting_started_en.md` and `getting_started_en.md` describe the steps necessary
to make a repository locally ready for development.

#### Language versions

The goal is to provide all parts of the documentation (except `CHANGELOG`, `README` and development) in English and German.
`CHANGELOG`, `README` and development will be written in English only.

#### Images

If images are used in the documentation, they have to be placed in a new sub-folder `figures`.
For `operations`, `gui` and `development` a separate `figures` folder should be created for each.
To minimize naming conflicts for images, another sub-folder should be created in the `figures` folder,
which has the same name as the Markdown file for which the images are to be used.
The images can then be placed there.

### Product spellings Cloudogu

| Topic           | Spelling                          | Comment                                    |
|-----------------|-----------------------------------|--------------------------------------------|
| CES             | Cloudogu EcoSystem                | -                                          |
| Dōgu            | Dogu                              | Without superimposed macron                |
| SCM-Manager     | SCM-Manager                       | Hyphenated                                 |
| User Management | User Management                   | Not written together.                      |
| Warp Menu       | Warp Menu (de) and Warp Menu (en) | The CES drop-down menu in the right margin |