---
title: Developing a GraphQL API in Go
description: ''
tags:
  - go
  - tutorial
  - graphql
series: Developing a GraphQL service
cover_image: ''
canonical_url: null
published: false
id: 1309557
---

The previous post discussed how to set up a project workflow for a
Go-based API.

The project leverages a folder structure which
separates packages based on responsibility and whether the packages
should be accessed externally. `cmd` is used for the application
entrypoint. `pkg` is for externally accessible packages (i.e. database)
models. `internal` is for internal application business logic and is
not accessible form external. `test` contains integration tests.

Coding standards and build pipeline set-up were also discussed. The
topic for this post will be developing out the API. We will be
discussing setting up the API entrypoints with gin-gonic, an API server
framework. We will be setting up our GraphQL API with gqlgen. A data
layer will be set up with Gorm for data, and OpenSearch to make the
data searchable.

## API Server with Gin-Gonic


