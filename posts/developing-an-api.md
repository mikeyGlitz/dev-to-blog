---
title: 'Supercharge your API Development: Building a High-Performance API with Go'
description: ''
tags:
  - go
  - tutorial
  - elastic
  - postgres
series: Developing a GraphQL service
cover_image: ''
canonical_url: null
published: false
id: 1309557
---

## Configuration with Environment Variables Using Envconfig

One of the primary aspects I focus on is configuring my applications. I rely on configuration to store essential details for the various systems that the application integrates with, such as databases, identity provider services, indexing services, and listener ports. The API's target environment is a Kubernetes cluster. To provide configuration values to Kubernetes, I make use of ConfigMaps or Secrets to supply environment variables.

To extract values from the system environment, I utilize [envconfig](https://github.com/kelseyhightower/envconfig), a Go package. Envconfig facilitates mapping system environment variables to a Go struct. These Go structs are exposed through a config package, enabling other parts of the application to access them.

Below is an example code block that demonstrates a configuration model using [struct tags](https://www.digitalocean.com/community/tutorials/how-to-use-struct-tags-in-go):

```go
package config

type DatabaseConfiguration struct {
  Host      string  `required:"true"`
  Port      int     `required:"true"`
  Username  string
  Password  string
  Name      string
}

type OpensearchConfiguration struct {
  Username        string
  Password        string
  CertificatePath string    `split_words:"true"`
  Addresses       []string  `required:"true"`
}

type Configuration struct {
  Database    DatabaseConfiguration
  Opensearch  OpensearchConfiguration
}
```

Once the configuration model is established, I expose it through a package-level function called `GetConfiguration`. It is expected to run once during the package setup.

```go
package config

var Config Configuration

func GetConfiguration() error {
  var dbConfiguration DatabaseConfiguration
  if err := envconfig.Process("db", &dbConfiguration); err != nil {
    log.Errorf("unable to read database configuration => %v", err)
  }

  var indexConfiguration DatabaseConfiguration
  if err := envconfig.Process("os", &indexConfiguration); err != nil {
    log.Errorf("unable to read opensearch configuration => %v", err)
  }

  Config = Configuration {
    Database:   dbConfiguration
    Opensearch: indexConfiguration
  }
}
```

In the `GetConfiguration` function, I retrieve the configuration values for the database and OpenSearch. These values are populated into their respective structs. If any errors occur during the process, appropriate error logging is performed. The resulting configurations are assigned to the `Config` variable within the Configuration struct.

## Implementing a Data Layer with Gorm

After successfully configuring the application, it's time to delve into integrating the data layer. For this purpose, I will utilize [gorm](https://gorm.io), a powerful SQL ORM that facilitates rapid development of the data layer using model structs.

Similar to the usage of struct tags in envconfig, gorm also leverages structs with struct tags to create a data model. To begin, we establish a `Base` struct that serves as a foundation for other models. The Base struct allows for overriding ID creation using UUID and sets default values for `CreatedAt` and `UpdatedAt`.
The `BaseSoftDelete` struct serves as a mechanism for implementing soft deletion. It extends the Base struct and introduces the `DeletedAt` field, which utilizes the `gorm.DeletedAt` type for handling soft deletes. By applying a struct tag, the `DeletedAt` field is indexed for optimized querying.

```go
package model

type Base struct {
  ID        string    `json:"id" gorm:"primaryKey,type:uuid"`
  CreatedAt time.Time `json:"createdAt"`
  UpdatedAt time.Time `json:updatedAt"`
}

type BaseSoftDelete {
  Base
  DeletedAt gorm.DeletedAt  `json:"deletedAt" gorm"index"`
}

func (b *Base) BeforeCreate(txn *gorm.DB) error {
  b.ID = uuid.NewString()
  return nil
}
```

In addition, the `Base` struct contains a `BeforeCreate` function, which serves as a pre-create hook for any struct that embeds the Base struct. This hook is invoked automatically before the struct is saved into the database, allowing us to set the ID to a new UUID. By generating a new UUID, we ensure uniqueness and maintain consistency when persisting data.

By utilizing the Base and BaseSoftDelete structs along with the `BeforeCreate` function, we establish a solid foundation for managing soft deletion and automated UUID generation within our data models. These features contribute to the overall robustness and reliability of our application's data layer.

With the base types created, we go on to create the rest of the model.

```go
package model

type Measurement struct {
  BaseSoftDelete
  Unit string
  Amount float32
}

type Ingredient struct {
  BaseSoftDelete
  Name string
  Measurement Measurement
}

type Recipe struct {
  BaseSoftDelete
  Ingredients []Ingredient
  Steps       []string
}

type User struct {
  BaseSoftDelete
  Name    string
  Email   string
  Recipes []Recipe
}
```

These model structs define the data structure for various entities. For instance, the `Measurement` struct represents a measurement with its associated unit and amount. Similarly, the `Ingredient` struct represents an ingredient with its name and corresponding measurement. The `Recipe` struct captures a recipe, including its ingredients and steps. Lastly, the `User` struct represents a user with their name, email, and associated recipes.

With these model definitions, we establish a solid foundation for building our data layer using gorm. These structs provide a structured representation of our data entities, enabling efficient storage, retrieval, and manipulation of data within the application.

Once we have created the data models, the next step is to define the database connection. To accomplish this, we can implement an initialization function called `InitDB`. This function will facilitate the establishment of a database connection by leveraging the values from the config package we defined earlier.

```go
package database

func InitDB() (*gorm.DB, error) {
  dbc := config.Config.Database
  databaseConnection := fmt.Sprintf("host=%s user=%s password=%s dbname=%s port =%d", dbc.Host, dbc.Username, dbc.Password, dbc.Name, dbc.Port)
  db, err := gorm.Open(postgres.Open(databaseConnection), &gorm.Config{})
  if err != nil {
    log.Errorf("[database.connection] => %v", err)
    return nil, err
  }
  return db, nil
}
```

In the `InitDB` function, we extract the database configuration values from the config package. These values include the host, username, password, database name, and port. Using these values, we construct a connection string for the PostgreSQL database.

Next, we use the `gorm.Open` function to establish a connection to the database. We pass in the PostgreSQL driver (`postgres.Open`) along with the constructed connection string. Additionally, we provide an empty `&gorm.Config{}` as the second argument to indicate the default configuration for the gorm instance.

If the connection is successfully established, the db object is returned, allowing us to interact with the database in our application. However, if an error occurs during the connection process, an error message is logged.

By implementing the `InitDB` function, we create a reusable and efficient method for initializing the database connection within our application's data layer. This sets the stage for seamless integration with the data models we defined earlier.

## Enhanced Searchability: Indexing Data with OpenSearch

## Seamless Upgrades: Migrating Data

## References

- gormigrate - https://github.com/go-gormigrate/gormigrate
- testcontainers with elasticsearch - https://riferrei.com/using-testcontainers-go-with-elasticsearch/
- Keycloak golang webservices - https://mikebolshakov.medium.com/keycloak-with-go-web-services-why-not-f806c0bc820a
- Opinionated graphql server with go - https://dev.to/cmelgarejo/creating-an-opinionated-graphql-server-with-go-part-1-3g3l
- Custom GraphQL validation - https://david-yappeter.medium.com/gqlgen-custom-data-validation-part-1-7de8ef92de4c
- Custom error types with go - https://klotzandrew.com/blog/error-handling-in-golang
