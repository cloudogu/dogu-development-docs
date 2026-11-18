# Dogu-development-docs Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- [#107] Add German and English guides for preparing V2 Dogus as migratable V3 charts, using Nexus examples for migration jobs, resource conventions and troubleshooting.
- [#103] Document the required label `k8s.cloudogu.com/dogu.name` on Dogu CRs and how to test backup & restore of a Dogu.
- [#98] Document runtime patches with quoted globalConfig expressions in German and English.

### Changed
- [#109] Revise the Dogu V3 docs against the current Dogu CR and Dogu operator: deploy with `cldk dogu dev-push` in the quick start and advanced guide, release with `cldk dogu release`
- [#98] Document separate mirror and runtime patch files in German and English.
