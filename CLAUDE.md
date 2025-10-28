# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Gradle plugin repository that provides three Docker-related Gradle plugins:
- `com.palantir.docker` - Build and push Docker images
- `com.palantir.docker-compose` - Generate docker-compose files with resolved dependencies
- `com.palantir.docker-run` - Run, stop, and manage Docker containers

**Important**: This repository is End of Life - no new features will be accepted and bugs may not be fixed. It is no longer used internally at Palantir.

## Build Commands

### Running Tests
```bash
./gradlew check
```

### Running a Specific Test
```bash
./gradlew test --tests "com.palantir.gradle.docker.PalantirDockerPluginTests"
```

### Building (without tests)
```bash
./gradlew build -x test -x check
```

### Publishing
```bash
./gradlew publish
```

Note: `publishPlugins` task only runs if the current state is a clean tag.

## Code Architecture

### Plugin Structure
The codebase is organized into three main plugins, each with its own entry point:

1. **PalantirDockerPlugin** (`com.palantir.docker`)
   - Entry point: `src/main/groovy/com/palantir/gradle/docker/PalantirDockerPlugin.groovy`
   - Extension: `DockerExtension.groovy`
   - Creates tasks: `docker`, `dockerPrepare`, `dockerClean`, `dockerTag*`, `dockerPush*`, `dockerfileZip`
   - Builds Docker images based on configuration and Dockerfile

2. **DockerComposePlugin** (`com.palantir.docker-compose`)
   - Entry point: `src/main/groovy/com/palantir/gradle/docker/DockerComposePlugin.groovy`
   - Extension: `DockerComposeExtension.groovy`
   - Creates tasks: `generateDockerCompose`, `dockerComposeUp`, `dockerComposeDown`
   - Resolves Docker image dependencies and populates template files

3. **DockerRunPlugin** (`com.palantir.docker-run`)
   - Entry point: `src/main/groovy/com/palantir/gradle/docker/DockerRunPlugin.groovy`
   - Extension: `DockerRunExtension.groovy`
   - Creates tasks: `dockerRun`, `dockerStop`, `dockerRunStatus`, `dockerRemoveContainer`
   - Manages Docker container lifecycle

### Key Implementation Details

- **DockerExtension**: Holds configuration for building Docker images (name, dockerfile, tags, buildArgs, labels, pull, noCache, buildx, platform, etc.)
- **CopySpec Pattern**: Uses Gradle's CopySpec to collect files for the Docker build context
- **Task Dependencies**: The `dockerPrepare` task copies files into `build/docker/` before the `docker` task runs
- **Tag Management**: Supports both deprecated `tags()` method and newer `tag(taskName, tagName)` method for creating tagged images
- **Docker Component**: Implements a custom Gradle component for Maven publishing of Docker image dependencies

### Testing

Tests are located in `src/test/groovy/com/palantir/gradle/docker/`:
- `AbstractPluginTest.groovy` - Base test class with helper methods
- `PalantirDockerPluginTests.groovy` - Tests for main Docker plugin
- `DockerComposePluginTests.groovy` - Tests for compose plugin
- `DockerRunPluginTests.groovy` - Tests for run plugin

Tests use GradleTestKit and Spock framework. The `AbstractPluginTest` provides utilities like `with()` for running Gradle tasks and `exec()` for executing shell commands.

## Gradle Configuration

- **Java Target**: Library targets Java 17 (configured in `javaVersions { libraryTarget = 17 }`)
- **Daemon Target**: JDK 21 for Gradle daemon
- **Test Gradle Versions**: Tests run against Gradle 8.14.3
- **Parallel Builds**: Enabled via `org.gradle.parallel=true`
- **Dependency Management**: Uses `com.palantir.consistent-versions` plugin with versions defined in `versions.props`

## Palantir Infrastructure Plugins

This project uses several Palantir infrastructure plugins:
- `com.palantir.baseline` - Code quality and formatting standards
- `com.palantir.java-format` - Java code formatting (native formatter enabled)
- `com.palantir.git-version` - Version from git tags
- `com.palantir.external-publish` - Publishing configuration
- `com.palantir.failure-reports` - Test failure reporting
- `com.palantir.jdks` - JDK provisioning and management

## CI/CD

GitHub Actions is configured with a modular workflow structure:

### Workflow Structure
- **Entry Point Workflows:**
  - `pr-review.yml` - Runs on pull requests and pushes to `develop`/`main` branches
  - `publish.yml` - Runs on merge to `develop` branch and on tags (publishing step currently disabled)

- **Reusable Workflows:**
  - `reusable-check.yml` - Runs `./gradlew check` (tests, checkstyle, etc.) with parallel execution
  - `reusable-build.yml` - Runs `./gradlew build -x test -x check` to compile and package

### Key Features
- Uses Amazon Corretto JDK 17 for building (JDK 21 available for daemon)
- Gradle caching is enabled for faster builds
- Enforces that no git-tracked files are modified during the build process
- Uploads test results and build artifacts for inspection
- Publishes test results directly to PR for easy review

### Future Publishing
The `publish.yml` workflow has a commented-out publish job that can be enabled when artifact publishing is configured. It will require:
- Setting up `GRADLE_KEY` and `GRADLE_SECRET` as repository secrets
- Uncommenting the publish job in `.github/workflows/publish.yml`

### Original CircleCI
The upstream Palantir repository uses CircleCI, which is preserved in `.circleci/config.yml` for reference.
