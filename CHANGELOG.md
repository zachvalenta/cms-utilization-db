# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
### Changed
### Removed
### Fixed

## [1.6.0](https://github.com/sudowing/cms-utilization-db/tree/v1.6.0) - 2019-01-09

### Fixed
- `start-with-prisma` make command. Added detach flag.

### Added
- Added Section in readme for distributing GraphQL Queries. Currently rely on storing them in GraphQL Bin and proving links.

## [1.5.0](https://github.com/sudowing/cms-utilization-db/tree/v1.5.0) - 2019-01-09

### Fixed
- Prisma install & deploy instructions.

## [1.4.0](https://github.com/sudowing/cms-utilization-db/tree/v1.4.0) - 2019-01-09

### Added
- Added Table of Contents to Readme

## [1.3.0](https://github.com/sudowing/cms-utilization-db/tree/v1.3.0) - 2019-01-08

### Added
- Added support for prisma service.

## [1.2.0](https://github.com/sudowing/cms-utilization-db/tree/v1.2.0) - 2019-01-08

### Fixed
- Fixed typo in readme.

## [1.1.0](https://github.com/sudowing/cms-utilization-db/tree/v1.1.0) - 2019-01-08

### Added
- Anchor links to readme.

## [1.0.0](https://github.com/sudowing/cms-utilization-db/tree/v1.0.0) - 2019-01-07

### Added
- bash script for entire ETL process
- bash script for managing `pg_dump` process
- bash script for managing `pg_restore` process
- `.sql` for building temp schema
- `.sql` for destroying temp schema
- `.sql` for building prod schema
- `.sql` for loading prod schema
- initial `cms.dump` file. Not stored in repo, but baked into Docker image.
