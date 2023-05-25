---
title: Supercharge Your API Development with GraphQL and Go
description: My article description
tags: ''
cover_image: ''
series: Developing a GraphQL API with Go
canonical_url: null
published: false
id: 1479976
---
## Setting up gqlgen for GraphQL Integration

To begin integrating GraphQL into your project, you will need to set up [gqlgen](https://gqlgen.com). gqlgen is a popular Go library for automatically generating a GraphQL server based on a schema definition. It provides a seamless way to build robust and type-safe GraphQL APIs. Here's how you can install and initialize gqlgen:

Start by adding the following line to your `tools.go` file:

```go
import _ "github.com/99designs/gqlgen"
```

Next, run the initialization script for gqlgen using the following commands:

```bash
go mod tidy
go run github.com/99designs/gqlgen init
```

Running `gqlgen init` will generate the `gqlgen.yml` file, which serves as the base configuration for gqlgen. It will also create a `graph` folder containing the necessary files and folders for GraphQL. Some important artifacts that are generated include `schema.graphqls`, which contains the GraphQL schema definition for your API, and the `generated` and `model` folders, which provide generated structs and functions to support your GraphQL implementation.

With the gqlgen skeleton established, let's customize it to fit the Go project convention we've set up in the previous entry of this series.

## Customizing gqlgen for the Go Project Convention

Begin by moving the `graph` folder into the `internal` folder. This ensures that the generated code follows the project's internal structure.

Next, we need to update the gqlgen configuration by modifying the `gqlgen.yml` file. We want gqlgen to recognize the updated `graph` folder and generate the code accordingly. Here's an example of the updated configuration:

```yml
# Where are all the schema files located?
schema:
  - internal/graph/schemas/*.graphqls

# Where should the generated server code go?
exec:
  filename: internal/graph/generated/generated.go
  package: generated

# Where should the generated models go?
model:
  filename: internal/graph/model/models_gen.go
  package: model

# Where should the resolver implementations go?
resolver:
  layout: follow-schema
  dir: internal/graph
  package: graph

# Other gqlgen configuration values
```

The modified configuration allows gqlgen to recognize multiple `.graphqls` files located in the `internal/graph/schemas` directory, enabling better compartmentalization of the schema.

## Defining the GraphQL Schema

In GraphQL, we need to define several primary elements: model types, connection types, input types for CRUD operations, and resolvers. Let's focus on an example for the `Ingredient` type:

```graphql
type Ingredient {
  id: ID!
  name: String!
  measurement: Measurement!
  createdAt: Time!
  updatedAt: Time!
}

type IngredientList {
  ingredients: [Ingredient!]
  lastItem: String!
  hits: Int!
}

input NewIngredient {
  name: String!
  measurement: String!
}

input UpdateIngredient {
  name: String
  measurement: String
}

type IngredientOps {
  update(id: String!, input: UpdateIngredient!): Ingredient! @goField(forceResolver: true)
  delete(id: String!): Ingredient! @goField(forceResolver: true)
}
```

The above snippet represents the `Ingredient` type in our schema. The schema file for `Ingredient` would be located at `internal/graph/schemas/ingredient.graphqls`. It includes definitions for the `Ingredient` type itself, an `IngredientList` type used for retrieving multiple records, input types for insertions and updates, and a type to compartmentalize mutations related to `Ingredient`.

By defining these elements in the schema, we establish the structure and behavior of our GraphQL API for the `Ingredient` type.

Once all the data model types are defined, it's time to integrate them with the root schema, which is located in the `schema.graphqls` file.

```graphql
directive @goField(
  forceResolver: Boolean
  name: String
) on INPUT_FIELD_DEFINITION | FIELD_DEFINITION

scalar Time

type Query {
  ingredient(id: String!): Ingredient!
  ingredients(name: String): IngredientList!
}

type Mutation {
  ingredient: IngredientOps! @goField(forceResolver: true)
}
```

In the schema, we define a custom scalar `Time`, which represents a `time.Time` object in Go within the context of our GraphQL schema. This scalar is built-in and supported by gqlgen.

GraphQL specifies three root types: `Query`, `Mutation`, and `Subscription`. In our example, we focus on `Query` and `Mutation`. The `Query` type is responsible for data retrieval operations, while the `Mutation` type handles updates to the data in the schema. We won't be using the `Subscription` type for our API server in this example.

gqlgen provides a built-in directive called `@goField`, which allows us to override the default behavior in GraphQL. We can use `@goField` to force specific fields to be treated as resolvers, enabling us to compartmentalize our mutations and customize their behavior as needed.

We can update the generated code by executing the following command:

```bash
go run github.com/99designs/gqlgen generate
```

This command triggers gqlgen to regenerate the necessary code based on the schema and configuration files. It ensures that any changes made to the schema or configuration are reflected in the generated code, providing an up-to-date implementation for our GraphQL API.

## Implementing Resolvers

## Schema-Level Validation

## Faster Data Retrievers with dataloaden

## Integrating gqlgen with gin-gonic

## Authenticate your API with Keycloak

## Service Monitoring with goctuator

## Summary

## References

- Keycloak golang webservices - <https://mikebolshakov.medium.com/keycloak-with-go-web-services-why-not-f806c0bc820a>
- gqlgen - <https://gqlgen.com/>
- Custom GraphQL validation - <https://david-yappeter.medium.com/gqlgen-custom-data-validation-part-1-7de8ef92de4c>
- Creating an Opinionated GraphQL Server with Go - <https://dev.to/cmelgarejo/creating-an-opinionated-graphql-server-with-go-part-2-46io>
