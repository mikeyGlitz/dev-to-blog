---
title: Go API Project Set-Up
description: ''
tags:
  - go
  - tutorial
  - beginners
  - pipelines
series: Developing a GraphQL service
cover_image: ''
canonical_url: null
published: true
id: 1306503
date: '2022-12-23T20:18:49Z'
---

Have you ever wondered what goes into creating a production-ready
workflow? Have you ever considered what kind of conventions to follow
when you're beginning a new project?

Recently I had the pleasure to experiment with producing a Go based
web server to serve an API. Here are some lessons I've learned along
the way.

## Conventions

As a general practice for working with any language, it is best to adhere
to conventions which are commonly recognized in the language's community
as much as possible. Before a single line of code is written, I set up
my project with the baseline tools that I think I'll need during the
development process.

### Project Structure

In the case of a Go-based project my starting point is the project
layout which is documented in the [golang-standards Project Layout](https://github.com/golang-standards/project-layout).

My project layout adheres to the following structure:

| Path       | Description                                                                                                    |
|------------|----------------------------------------------------------------------------------------------------------------|
| `cmd`      | Contains the `main` package which is the primary entry point for my application                                 |
| `pkg`      | Contains public packages which are intended to be utilized by other projects (i.e. shared struct declarations) |
| `test`     | Integration tests (i.e. testcontainers)                                                                        |
| `internal` | Any code which not intended to be utilized outside of the application or module package                        |

While the go-lang standards layout recommends additional folders like
`api` and `web` for the purposes of serving a web API, since this
project will leverage GraphQL to provide API functionality, I won't
have separate folders for JSON/OpenAPI specifications nor will I have
a separate folder for hosting endpoints which will later be explained
in a future gqlgen section.

Also, the convention for which unit tests are written in go with the
`test` package. Unit test files need to be siblings to the implementation
files in their respective hierarchies. Due to this sibling
relationship, I leverage the `test` folder specifically for integration
testing since the integration tests don't necessarily have a dependency
on the internal state of the packages themselves.

Since I develop using `GO111MODULE=on`, I do not have need for a `vendor` folder.

`go.mod` is a file which contains the dependency list for all the 3rd
party go modules I'll be using to develop my project.

`tools.go` is a file which contains all the CLI tools that my project depends on.

To set up `go.mod` I use the following command and answer all the prompts:

```bash
go mod init
```

### Enforcing Style

Generally when working on a project, I prefer to enforce a recognized style-guide.

For Java, I use the Google style-guide with checkstyle.
For Kotlin, I use the Pinterest style-guide with ktlint.
For Python, I use flake8 to enforce styles.
For Typescript/Javascript I use eslint with the AirBnB style guide.

For Go, I use [golangci-lint](https://golangci-lint.run/).

I add the following line to my `tools.go` file under `import`:

```go
_ "github.com/golangci/golangci-lint/cmd/golangci-lint"
```

### Mocks

Unit tests are leveraged to test individual units of code. As such it
is not recommended for a developer to scaffold entire dependencies for
the sake of testing a single object. Due to the way Go's specific
implementations work, I've learned over time to declare interfaces
for a lot of the structs that I use in Go. Interfaces not only define
a contract for which struct-based implementations should adhere,
but they also provide a mechanism for which struct methods can be
mocked. While I've experimented with the mock package in
[testify](https://pkg.go.dev/github.com/stretchr/testify/mock), I've come to prefer the mock functionality which is
provided by [mockgen](https://github.com/golang/mock).

Mockgen is a utility which attaches to the `go generate` command.
Mockgen will generate mocked structures to emulate behavior for
interfaces in your code so that you can short circuit behavior to
assist in unit testing.

To generate mocks, I add a line to the top of the file under the
`package` declaration to run a generate script I.E.:

```go
//go:generate go run github.com/golang/mock/mockgen@v1.6.0 -destination=./mocks/mock_index.go -package=mocks gitlab.com/pantry-organize/pantry-api/internal/index OpensearchConnection
```

The generate line has the following sections:

- `go run` - tells go to run the script which is provided in the next argument
- `github.com/golang/mock/mockgen@1.6.0` - the module which you want go to run
- `-destination=[relative-path]` - Where you want mockgen to produce the files relative to the running directory
- `-package=[package-name]` - The package name you want to create the mocked implementation in
- `[package-path]` - The package that you want to mock
- `[interface-name]` - The interface that you want to mock

I add the following line to my `tools.go` file under `import`:

```go
_ "github.com/golang/mock/mockgen"
```

### Test Reporting

Since my project is hosted on GitLab, I have to make some tool
integration considerations when it comes to test reporting.
The kind of test reporting we're interested in reporting to GitLab are
test reports for which tests passed and which tests failed as well as
reports for which lines of code are covered by tests, also known as coverage.

The `go test` command does not support test reporting out of the box.
To generate a test report, we have to use
[gotestsum to generate a JUnit report](https://github.com/gotestyourself/gotestsum).
We add `gotestsum` to our `tools.go` file:

````go
_ "github.com/gotestyourself/gotestsum"
```

To report coverage to GitLab we have to use a tool which can convert
the coverage report from `go test` to Cobertura.
We'll pull in [gocover-cobertura](https://github.com/t-yuki/gocover-cobertura) to generate our coverage report.

We add the following line to our `tools.go` file:

```go
_ "github.com/t-yuki/gocover-cobertura"
```

### Makefile

Every language ecosystem that I've worked with has had some mechanism
for which to run scripts. I leverage scripting as glue for running
various build tasks.

In the case of Java, there's usually a maven plugin (i.e. checkstyle,
spring, shadow) which I can use to accomplish a particular build phase,
or leverage Gradle for the ability to run a custom build script.

For Node-based projects, you can specify a build script using the
`scripts` block in `package.json`.

For Go, the general convention is to leverage good ol' Makefile.

Makefile is a tool which can be used to compile source files. My first
experience using make was for C/C++ projects in university. Make can
also be used to run command-line scripts which is what we'll be using
in the case of go.

Generally speaking my Makefiles consist of the build steps that I'm
familiar with in the case of other languages:

- `clean` - Removes files which were either compiled binaries, generated code, or otherwise not tracked in version control
- `lint` - Scans an analyzes code for style guide adherence or syntactic correctness
- `test` - Preforms unit tests
- `integration` - Preforms all tests: unit and integration
- `report` - Prepare test reports (pass/fail and test coverage)
- `build` - Compiles source code into a single binary

In the case of Go, for a majority of the language's history there was
no concept of generics in the language. A stop-gap measure to create
generic-like functionality in Go was to leverage tools to auto-generate code.
In Go projects I usually create a `gen` task in `Makefile` which is
used for the express case of running `go generate ./...`.

Here is an example Makefile:

```Makefile
clean:
    @rm -rf ./internal/dataloaders/*_gen.go
    @rm -rf ./internal/graph/model/models_gen.go
    @rm -rf ./internal/graph/generated
    @rm ./pantry-api

gen:
    @go generate ./...

lint: gen
    @go run github.com/golangci/golangci-lint/cmd/golangci-lint run

test: gen
    @go test -short ./...

integration: lint
    @go test ./... -coverprofile=coverage.txt -covermode count
    @go run gotest.tools/gotestsum --junitfile report.xml --format testname
    @go run github.com/t-yuki/gocover-cobertura < coverage.txt > coverage.xml

report: integration
    @go tool cover -html coverage.txt -o coverage.html

build: gen
    @go build -ldflags "$(LDFLAGS)" -o pantry-api ./cmd/pantry-api
```

## Containerization

As a developer, you generally want to take your user experience into
consideration. When the application you're developing is a web server,
one of your users will be the operations person or the site administrator
who will ensure that your application runs and functions properly in
its deployment environment.

While a lot of legacy applications run directly on the host environment,
the use of containerization is more prevelant than ever. With a
container, it is possible to the deployment time and the installation
and configuration of your application. With a container, your
application deploys inside a self-contained environment (a lightweight
VM) and usually contains the minimal pre-configuration (more on this
in subsequent sections) which is required to run.

[Docker](docker.com) is the most widely used container engine to date.
For this example, we will be deploying our application using Docker
containers. To create a Docker container you must first create a
`Docker` file. When creating a Dockerfile, there are certain
considerations which may be taken into account as part of your build
process. You can either copy your application package or binary
into your container pre-built, or you can create a build step within
your container context and copy the binary from the build stage in
`Dockerfile` to a destination runtime environment.

As a general rule of thumb, I prefer to build my applications within
the `docker build` process. Organizaing `Dockerfile` into a `build` stage
and a `ship` stage provides the added benefit of multi-architecture builds.

With the increasing popularity of ARM-based architectures (as seen in
the Apple Silicone Macs, Raspberry Pi, and AWS Graviton instances),
supporting both Intel and ARM platforms may be a consideration for
increased application portability as well as cost savings.

In my example, I will demonstrate releasing a multi-stage Docker image
which is compatible with `docker buildx` for releasing a multi-
architecture image. This example will contain a `build` stage for
compiling a Go application and a `ship` stage for the actual runtime
environment once the container starts

```Dockerfile
# TARGETPLATFORM specifies the build/runtime environment i.e. ARM/AMD64
# This argument defaults to linux/amd64
FROM --platform=${TARGETPLATFORM:-linux/amd64} golang:1.19-alpine3.16 as build

ARG TARGETPLATFORM
ARG TARGETOS
ARG TARGETARCH

# Enable golang modules
ENV CGO_ENABLED=1
ENV GO111MODULE=on

# LDFLAGS are used to set values for signing your go binary
ENV LDFLAGS=

# Install required C packages in Alpine
RUN apk update \
    && apk add --no-cache gcc g++ libstdc++ make ca-certificates binutils-gold git
RUN update-ca-certificates

# Create a temporary /build folder and use it to compile the runtime binary
RUN mkdir /build
COPY . /build
WORKDIR /build
RUN export GOOS=${TARGETOS} && \
    GOARCH=${TARGETARCH} && \
    make test && \
    make build

# Set up the ship build context for the final deployment
FROM --platform=${TARGETPLATFORM:-linux/amd64} alpine:3.14 as ship

# Create a folder from which the application will run
RUN mkdir -p /home/app
# Copy the runtime binary from the build stage
COPY --from=build /build/pantry-api /home/app/server
# Add execute permission to the runtime binary
RUN chmod +x /home/app/server

# Docker best practice: NEVER EVER RUN YOUR CONTAINER AS A ROOT USER
# Here we create an application user which is a system account for the
# express purpose of running our application
RUN adduser -S appuser
RUN addgroup -S appuser && adduser appuser appuser
# Assign ownership of /home/app and all of its contents to the appuser account
RUN chown -R appuser:appuser /home/app

WORKDIR /home/app

# Anything beyond this statement will be executed as appuser
USER appuser

# Expose 8080 because that is the port which the web server is listening on
EXPOSE 8080

# This is the container start-up command which is run when the container is started
CMD ["./server"]
```

## CI/CD Pipeline

Pipelines enable automated builds of your application to kick off
whenever an update is pushed to version control. There are several
pipeline providers which are available at multiple price points:
Bitbucket Pipelines, GitHub Actions, Travis, Circle CI, just to
name a few.

Ever since the Microsoft Acquisition of GitHub, my preference has
leaned more towards using GitLab as my version control host.
GitLab offers the same functionality as GitHub as well as a self-hosted
option if you don't feel comfortable with hosting your code on
the Internet. GitLab offers a better (IMO) experience with its CI/CD pipelines.

To set up CI/CD pipelines in GitLab, you'll need to add a `.gitlab-ci.yml`
file. `.gitlab-ci.yml` specifies the build phases under the `stages` block.
The project builds occur under two stages: `test` and `release`.

```yaml
stages:
  - test
  - release
```

The next block in `.gitlab-ci.yml` is the services block. Since our
tests use [testcontainers](https://golang.testcontainers.org) package and we're pushing a
docker container onto [Dockerhub](https://hub.docker.com), we will need to specify a `services`
block next. Services will enable our pipeline to leverage Docker-in-Docker DinD.

```yaml
services:
  - name: docker:dind
    command:
      - "--tls=false"
```

Test is a stage which triggers the `make test` command which will run the go code
generators, lint the code, run the unit tests, and run the integration tests. 

The next block is the `sast` block which will pull in GitLab's standard SAST template.
The SAST template will scan the Go code for vulnerabilities as well as preform static
analysis against the code.

```yaml
sast:
  stage: test
include:
  - template: Security/SAST.gitlab-ci.yml
```

The next block is `go-test` which will be used to explicitly run the go tests.

```yaml
go-test:
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
    DOCKER_DRIVER: overlay2
    CGO_ENABLED: "1"
    GO111MODULE: "on"
  image: golang:1.19-alpine3.16
  stage: test
  before_script:
    - apk update && apk add --no-cache ca-certificates make binutils-gold gcc g++ libstdc++ git
    - update-ca-certificates
  script:
    - make integration
    - make build
  artifacts:
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

The `artifacts` block specifies paths for GitLab to capture artifacts.
These artifacts come in the form of test reports. Our project creates
a pass/fail report which uses the JUnit test report format, `report.xml`,
and a `coverage.xml` file which is consumed by cobertura to upload a test
coverage report to GitLab.

The next stage is the release stage. Release steps encompass everything which
is needed to release the application. Release in this context refers to publishing
a release to the GitLab repository and publishing the Docker image to Docker Hub.

This block creates the release tag in the GitLab repository

```yaml
release_job:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - echo "running release_job for $TAG"
  release:
    tag_name: 'v0.$CI_PIPELINE_IID'
    description: '$CI_COMMIT_MESSAGE'
    ref: '$CI_COMMIT_SHA'
```

This block runs the docker image build and pushes it out to Docker Hub

```yaml
docker-build:
  # Use the official docker image.
  image: docker:latest
  stage: release
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin
  # Default branch leaves tag empty (= latest tag)
  # All other branches are tagged with the escaped branch name (commit ref slug)
  script:
    - |
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        tag=""
        echo "Running on default branch '$CI_DEFAULT_BRANCH': tag = 'latest'"
      else
        tag=":$CI_COMMIT_REF_SLUG"
        echo "Running on branch '$CI_COMMIT_BRANCH': tag = $tag"
      fi
    - docker buildx create --use
    - >
        docker buildx build --push 
        --platform linux/arm/v7,linux/arm64/v8,linux/amd64 
        --build-arg CI_COMMIT_SHA=${CI_COMMIT_SHA} 
        --build-arg CI_COMMIT_AUTHOR="${CI_COMMIT_AUTHOR}"
        --build-arg CI_COMMIT_TIMESTAMP="${CI_COMMIT_TIMESTAMP}" 
        --build-arg CI_REPOSITORY_URL="${CI_REPOSITORY_URL}" 
        --build-arg CI_COMMIT_BRANCH=${CI_COMMIT_BRANCH} 
        --build-arg CI_JOB_STARTED_AT="${CI_JOB_STARTED_AT}" 
        --tag "$CI_REGISTRY_IMAGE${tag}" .
```

This block pushes a readme out to the docker repo in Docker Hub

```yaml
readme:
  stage: release
  image:
    name: chko/docker-pushrm
    entrypoint: ["/bin/sh", "-c", "/docker-pushrm"]
  script: "/bin/true"
  variables:
    DOCKER_USER: $CI_REGISTRY_USER
    DOCKER_PASS: $CI_REGISTRY_PASSWORD
    PUSHRM_TARGET: docker.io/$CI_REGISTRY_IMAGE
    PUSHRM_FILE: $CI_PROJECT_DIR/README.md
```

In the next post we'll be going over how to build out the service

## References

- Release CI/CD Examples - https://docs.gitlab.com/ee/user/project/releases/release_cicd_examples.html#create-a-release-when-a-commit-is-merged-to-the-default-branch)
- Go Project Layout - https://github.com/golang-standards/project-layout
- Go test coverage visualization - https://docs.gitlab.com/ee/ci/testing/test_coverage_visualization.html#go-example
- Go unit test results - https://docs.gitlab.com/ee/ci/testing/unit_test_report_examples.html#go
- gotestsum - https://github.com/gotestyourself/gotestsum
- golangci lint - https://golangci-lint.run/
- gocover-cobertura - https://github.com/t-yuki/gocover-cobertura
- multiarch builds with Docker - https://www.docker.com/blog/multi-arch-build-what-about-gitlab-ci/
