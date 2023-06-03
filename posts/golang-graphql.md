---
title: Supercharge Your API Development with GraphQL and Go
description: My article description
tags:
  - go
  - graphql
  - tutorial
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

After executing the `gqlgen generate` command, you will notice several new files in the `internal/graph` folder. Each model defined in the previous section will have a corresponding `.resolvers.go` file, which contains the implementation of the schema resolvers. Initially, these files will contain stubbed-out resolver methods.

For example, let's consider the integration of mutations with the `Ingredient` type:

```go
package graph

func (r *ingredientOpsResolver) Update(ctx context.Context, obj *model.IngredientOps, id string, input map[string]interface{}) (*model.Ingredient, error) {
    // Resolver implementation here
}

type IngredientOpsResolver struct {
    *Resolver
}
```

In the above snippet, we have the resolver implementation for the `Update` mutation of the `Ingredient` type. Additionally, a `schema.resolvers.go` file is generated, which includes resolver functions for both queries and mutations:

```go
package graph

func (r *queryResolver) Ingredient(ctx context.Context, id string) (model.Ingredient, error) {
    // Resolver implementation here
}

func (r *mutationResolver) Ingredient(ctx context.Context) (*model.IngredientOps, error) {
    return &model.IngredientOps{}, nil
}

func (r *Resolver) Mutation() generated.MutationResolver {
    return &mutationResolver{r}
}

func (r *Resolver) Query() generated.QueryResolver {
    return &queryResolver{r}
}

type mutationResolver struct {
    *Resolver
}

type queryResolver struct {
    *Resolver
}
```

In the `schema.resolvers.go` file, we have resolver methods for the `Ingredient` query and the `Ingredient` mutation. These methods will be implemented with the required functionality to handle the corresponding GraphQL operations.

Overall, after generating the code with `gqlgen`, the resolver files provide a starting point for implementing the actual logic for the resolvers, allowing you to customize the behavior of your GraphQL API.

## Abstracting the Data Model: Implementing Services

While we could proceed to call our data model methods directly from the resolvers we just generated, as a general best practice, we'll wrap our data model with a layer of abstraction called services. This approach provides several benefits and allows us to maintain separation of concerns. In the `internal/graph/services` folder, we'll create a service for each data model defined in the previous article.

Each service will define an interface, `IIngredientService` in the case of the `Ingredient` model, which outlines the operations that can be performed on that specific data model. This interface serves as a contract and provides a generic abstraction for our resolvers. By working with interfaces, we can easily change the implementation of each service without impacting the resolvers. This flexibility comes in handy, especially when writing unit tests, where we can swap out a service implementation for a mock implementation.

In the example of the `IngredientService`, we define two package aliases: `dbmodel` refers to the database model package created with gorm, and `apimodel` refers to the generated GraphQL model created with gqlgen. The `IngredientService` struct holds the necessary dependencies such as the database connection (`DB`), the OpenSearch connection (`ES`), and the context (`Context`).

Within the `IngredientService`, we implement the methods defined in the `IIngredientService` interface. For instance, the `Create` method takes input from the GraphQL API, creates a new `dbmodel.Ingredient` instance, and saves it to the database using `svc.DB.Create()`. We can also perform additional logic or transformations as needed before returning the result converted to the GraphQL model (`apimodel.Ingredient`).

```go
package services

type IIngredientService interface {
    Create(input apimodel.NewIngredient) (*apimodel.Ingredient, error)
    Update(id string, input map[string]interface{}) (*apimodel.Ingredient, error)
    Delete(id string) (*apimodel.Ingredient, error)
    FetchIngredient(id string) (*apimodel.Ingredient, error)
    FetchIngredients(id string) (*apimodel.IngredientList, error)
}

type IngredientService struct {
    DB      *gorm.DB
    ES      index.OpenSearchConnection
    Context context.Context
}

func (svc *IngredientService) Create(input apiModel.NewIngredient) (*apimodel.Ingredient, error) {
    ingredient := dbmodel.Ingredient{
        Name:        input.Name,
        Measurement: input.Measurement,
    }

    if err := svc.DB.Create(&ingredient); err != nil {
        log.Errorf("[IngredientService] could not save Ingredient => %v", err)
        return nil, err
    }

    return ingredient.ConverToGraphQLModel(), nil
}

// Implement remaining service methods

type Services struct {
    UserService IUserService
}
```

The `Services` struct acts as a container for all the defined services, providing a convenient way to manage and access them in our application.

By introducing this services layer, we achieve better organization, maintainability, and testability of our codebase.

### Integrating Services Using Middleware

With our services implemented, the next step is to integrate them into our GraphQL implementation through the use of middleware. We'll leverage the HTTP server framework, gin-gonic, to set up the middleware. To begin, let's create a new folder called `internal/middleware`, which will contain our middleware files. Within this folder, we'll create a file named `services.middleware.go`. In this file, we'll define a function called `Services` that will serve as the middleware function, enhancing the server context with the services we defined earlier. Additionally, we'll create a function named `ForServices` that will be used to fetch the services from the server context.

```go
package middleware

var serviceKey = &contextKey{name: "services"}

// Services enhances the request context with the services
func Services(db *gorm.DB, es index.OpensearchConnection) gin.HandlerFunc {
    return func(c *gin.Context) {
        services := services.Services{
            UserService:       &services.UserService{DB: db, ES: es, Context: c.Request.Context()},
            PantryService:     &services.PantryService{DB: db, ES: es, Context: c.Request.Context()},
            IngredientService: &services.IngredientService{DB: db, ES: es, Context: c.Request.Context()},
        }
        // Enhance the context with the services
        c.Request = c.Request.WithContext(context.WithValue(c.Request.Context(), serviceKey, &services))
        c.Next()
    }
}

// ForServices is used to retrieve the service directory from the context
func ForServices(ctx context.Context) *services.Services {
    return ctx.Value(serviceKey).(*services.Services)
}
```

The `Services` function is a gin.HandlerFunc that takes a database connection (`db`) and an OpenSearch connection (`es`) as parameters. Within this function, we create instances of the services we defined earlier, passing the necessary dependencies. The middleware then enhances the request context by adding the services to it using `context.WithValue()`. The `ForServices` function retrieves the services from the context based on the `serviceKey`.

Within the context of our resolvers, we can now leverage our service implementation. For example, let's consider the implementation of an ingredient resolver:

```go
func (r *ingredientOpsResolver) CreateIngredient(ctx context.Context, obj *apimodel.IngredientOps, input apimodel.NewIngredient) (*apimodel.Ingredient, error) {
    services := getServices(ctx)
    return services.IngredientService.Create(input)
}
```

In this code snippet, we define the resolver method `CreateIngredient` which takes the necessary context, the object being resolved (`obj`), and the input data (`input`). By calling `getServices(ctx)`, we retrieve the services from the context. With the services available, we can then invoke the `Create` method of the `IngredientService` to add a new ingredient based on the provided input. This integration of services within our resolvers allows us to easily interact with the underlying data models and perform the necessary operations.

## Schema-Level Validation

To ensure the stability and security of your API, it is crucial not to blindly trust user input. One of the primary steps in handling incoming requests is validating the request body. By validating the request body, you can ensure that only expected and valid requests are processed, while also safeguarding your API against potential malicious code or attacks.

Validating the request body serves as a defense mechanism to filter out any inappropriate or malformed input that could potentially compromise the integrity and security of your system. It allows you to verify the data's format, check for required fields, enforce data constraints, and perform any necessary sanitization or normalization.

By implementing robust input validation mechanisms, you establish a strong line of defense against common security vulnerabilities such as injection attacks, cross-site scripting (XSS), and other forms of malicious exploits. It helps to mitigate the risk of unauthorized access, data breaches, and manipulation of sensitive information.

In addition to ensuring security, proper input validation also contributes to the overall stability and reliability of your API. By rejecting invalid or unexpected requests early in the process, you can prevent errors, inconsistencies, and potential crashes that could occur when handling malformed data. This improves the overall robustness and performance of your API.

Transitioning to the built-in validations provided by GraphQL, you can leverage the native validation capabilities of the GraphQL specification itself. GraphQL offers a range of predefined validation rules that can be applied to your schema and queries, ensuring data integrity and consistency.

By utilizing these built-in validations, you can further enhance the validation process within your API. The GraphQL validation rules not only validate the incoming request body but also leverage the GraphQL schema to ensure the validity of the entire operation.

The GraphQL schema acts as a blueprint for your API, defining the types, fields, and relationships that your API supports. During the validation process, GraphQL's built-in validations compare the structure and semantics of the incoming request against the schema. This schema-based validation ensures that the requested fields, arguments, and operations conform to the defined types and their respective constraints.

By incorporating the schema in the validation process, you benefit from a powerful mechanism that checks not only the basic syntax but also the semantic correctness of the request. The validation rules analyze the query's structure, field usage, argument requirements, input object validation, and more, ensuring that the request aligns with the schema's expectations.

The benefit of using GraphQL's built-in validations, which are schema-aware, is that they provide a standardized and declarative approach to ensure data quality. The validations are performed before executing the resolver functions, guaranteeing that the incoming request aligns with the schema's definitions.

This schema-based validation approach offers several advantages. First, it allows you to catch and handle potential issues early on, reducing the chances of executing invalid or inconsistent operations. Second, it enables you to provide meaningful and precise error messages to clients, guiding them in making valid requests by highlighting specific violations against the schema. Third, it helps maintain the consistency and coherence of your API's responses by ensuring that the data returned adheres to the defined schema and meets the expected structure.

Moreover, GraphQL's validation rules are extensible, allowing you to add custom validation logic tailored to your specific business requirements. This flexibility enables you to enforce additional business rules, perform complex validations, and integrate with any existing validation frameworks or libraries, all while leveraging the schema as the foundation for validation.

We will extend GraphQL's built-in validation logic by introducing a custom directive that enables field-level validation, allowing us to impose additional constraints on specific fields. To begin, we'll define the new directive in our `schemas.graphqls` file, as shown in the snippet below:

```graphql
directive @validate(constraint: String!) on INPUT_FIELD_DEFINITION | ARGUMENT_DEFINITION
```

This directive, named `validate`, will be utilized on query arguments and fields within our `input` types. By applying this directive, we can specify a constraint that must be satisfied by the input provided for the annotated field.

## Faster Data Retrievers with dataloaden

## Integrating gqlgen with gin-gonic

## Authenticate your API with Keycloak

## Service Monitoring with goctuator

## Summary

## References

- Keycloak golang webservices - <https://mikebolshakov.medium.com/keycloak-with-go-web-services-why-not-f806c0bc820a>
- gqlgen - <https://gqlgen.com/>
- Using GraphQL with Golang - <https://www.apollographql.com/blog/graphql/golang/using-graphql-with-golang/>
- Custom GraphQL validation - <https://david-yappeter.medium.com/gqlgen-custom-data-validation-part-1-7de8ef92de4c>
- Creating an Opinionated GraphQL Server with Go - <https://dev.to/cmelgarejo/creating-an-opinionated-graphql-server-with-go-part-2-46io>
