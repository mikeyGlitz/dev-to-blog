---
title: "Building Robust Applications in Go: Integrating Envconfig, Gorm, and OpenSearch"
description: "Learn how to build robust and scalable applications in Go by integrating essential tools and technologies such as Envconfig for configuration management, Gorm for data persistence, and OpenSearch for efficient data indexing. Explore step-by-step implementation and best practices to enhance your Go development skills."
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
Welcome to the second entry in our multi-part blog series on building a powerful application with Go! In our previous post, we laid the foundation by creating a Go-based project skeleton using GitLab, Docker, and Go-based linting and build tools. Now, we're ready to delve deeper into the development process and explore three crucial topics: creating an application configuration with envconfig, integrating a data persistence layer with Gorm, and indexing data with OpenSearch.

Efficiently managing application configuration, persisting data, and enabling powerful search capabilities are vital aspects of building robust and scalable applications. In this article, we'll guide you through the process of implementing these features in your Go application, equipping you with the necessary tools to develop high-performing and feature-rich software.

Specifically, we'll cover the following topics:

1. **Creating an Application Configuration with envconfig**: We'll revisit the envconfig package to establish a flexible and centralized configuration system for our application. By leveraging environment variables, we'll make our application easily configurable across different deployment environments.

2. **Integrating a Data Persistence Layer with Gorm**: Gorm, a popular ORM library for Go, provides powerful abstractions for interacting with databases. We'll explore how to integrate Gorm into our application, enabling seamless data persistence operations. You'll learn about defining models, performing CRUD operations, and establishing relationships between entities.

3. **Indexing Data with OpenSearch**: OpenSearch, a high-performance search engine, allows us to efficiently index and search through large datasets. We'll integrate OpenSearch into our application and demonstrate how to leverage its indexing capabilities for quick and accurate search operations. You'll discover techniques for indexing data and executing various types of searches.

By the end of this article, you'll have a solid understanding of application configuration management, data persistence with Gorm, and data indexing with OpenSearch in your Go-based projects. Armed with these essential skills, you'll be well-equipped to develop robust, scalable, and search-enabled applications using Go.

Let's jump right in and explore the powerful combination of envconfig, Gorm, and OpenSearch to enhance your Go applications!

## Creating an Application Configuration with envconfig

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

## Integrating a Data Persistence Layer with Gorm

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

## Indexing Data with OpenSearch

With a basic data layer established, it's time to explore the next step: making the data searchable. In this regard, we will leverage OpenSearch to index our data. OpenSearch, a fork of Elasticsearch created by Amazon Web Services, provides a powerful search engine that enhances data retrieval capabilities. While OpenSearch inherits many features from Elasticsearch, it stands out as a fully open-source solution that offers more features for free.

To kickstart our OpenSearch implementation, we first need to model the OpenSearch connection. Our goal is to implement four primary operations in OpenSearch: inserting a new record, updating an existing record, deleting a record, and retrieving search results.

```go
package index

type OpenSearchConnection interface {
  Insert(index string, payload interface{}) error
  Delete(index string, id string) error
  Update(index string, id string, updates interface{}) error
}
```

With a basic data layer established, it's time to explore the next step: making the data searchable. In this regard, we will leverage OpenSearch to index our data. OpenSearch, a fork of Elasticsearch created by Amazon Web Services, provides a powerful search engine that enhances data retrieval capabilities. While OpenSearch inherits many features from Elasticsearch, it stands out as a fully open-source solution that offers more features for free.

To kickstart our OpenSearch implementation, we first need to model the OpenSearch connection. Our goal is to implement four primary operations in OpenSearch: inserting a new record, updating an existing record, deleting a record, and retrieving search results.

### Implementing the OpenSearch Client

The `Client` struct implements the `OpenSearchConnection` interface and serves as a container for the OpenSearch client object. Additionally, we introduce a `handleBodyError` function to handle error messages received from OpenSearch.

```go
type Client struct {
  DB *opensearch.Client
}

func handleBodyError(status string, responseBytes []byte) error {
  res := gjson.GetManyBytes(responseBytes, "error.type", "error.reason")
  return fmt.Errorf(
    "[ %s ] %s: %s",
    status,
    res[0].String(),
    res[1].String(),
  )
}

func (idx *Client) Insert(index string, payload interface{}) error {
  log.Debugf("[opensearch.insert] inserting record into index %s:\n%v", index, payload)
  res, err := idx.DB.Index(index, opensearchutil.NewJSONReader(&payload))
  if err != nil {
    log.Errorf("[opensearch.insert] unable to insert document into OpenSearch => %v", err)
    return err
  }
  defer res.Body.Close()
  responseBytes, err := io.ReadAll(res.Body)
  if err != nil {
    log.Errorf("[opensearch.insert] unable to parse JSON response => %v", err)
    return err
  }
  if res.IsError() {
    err := handleBodyError(res.Status(), responseBytes)
    log.Errorf("[opensearch.insert] received error response from OpenSearch => %v", err)
    return err
  }
  inserted := gjson.GetBytes(responseBytes, "result")
  log.Debugf("[opensearch.insert] insertion result => %s", inserted.String())

  return nil
}

// ... (rest of the functions)
```

The provided code defines the `Client` struct, which implements the `OpenSearchConnection` interface. It includes methods for inserting, updating, deleting, and searching records in OpenSearch. The `handleBodyError` function is used to handle error messages received from OpenSearch.

In the `Insert` method, a record is inserted into the specified index using the OpenSearch client. The response is processed, and any errors or relevant information are logged accordingly.

The `Update` and `Delete` methods follow a similar pattern, leveraging the OpenSearch client to perform the respective operations and handling the responses appropriately.

The `Search` method relies on a couple of additional helpers:
A `SearchResponse` struct to store the JSON structure of the search result,
and a function `parseSearchResults` to parse the OpenSearch response into
usable search results.

```go
type SearchResponse struct {
 Hits     int                       // The number of results returned from the search
 Results  []map[string]gjson.Result // An array of results
 LastItem string                    // The ID of the last item in the list
}

func parseSearchResult(responseBytes []byte, isGeneric bool) (*SearchResponse, error) {
 res := gjson.GetManyBytes(responseBytes, "hits.total.value", "hits.hits")
 hits, err := strconv.Atoi(res[0].String())
 if err != nil {
  return nil, err
 }
 results := res[1].Array()
 mapped := make([]map[string]gjson.Result, 0)
 if len(results) > 0 {
  mapped = funk.Map(results, func(elem gjson.Result) map[string]gjson.Result {
   source := elem.Map()["_source"]
   if !isGeneric {
    return source.Map()
   }

   index := elem.Map()["_index"]
   sourceFields := source.Map()
   document := make(map[string]gjson.Result)
   document["index"] = index
   for key, element := range sourceFields {
    document[key] = element
   }
   return document
  }).([]map[string]gjson.Result)
  log.Debugf("Received results => %v", mapped)
 }

 response := &SearchResponse{
  Hits:    hits,
  Results: mapped,
 }

 if len(results) > 0 {
  lastElem := funk.Last(results).(gjson.Result)
  lastId := gjson.Get(lastElem.Raw, "id").String()
  response.LastItem = lastId
 }

 return response, nil
}

func (idx *Client) Search(index string, payload *map[string]interface{}, order string, startAt int, size int) (*SearchResponse, error) {
 sort := []map[string]interface{}{{
  "updatedAt": map[string]interface{}{"order": order},
 }}

 pageSize := 10
 if size != 0 {
  pageSize = size
 }

 query := &map[string]interface{}{
  "match_all": map[string]interface{}{},
 }

 reqBody := map[string]interface{}{
  "query": query,
  "sort":  sort,
  "size":  pageSize,
  "from":  startAt,
 }

 res, err := idx.DB.Search(idx.DB.Search.WithIndex(index), idx.DB.Search.WithBody(opensearchutil.NewJSONReader(&reqBody)))
 if err != nil {
  log.Errorf("[opensearch.search] encountered error while attempting to search:\n%v", err)
  return nil, err
 }
 defer res.Body.Close()
 bodyBytes, err := io.ReadAll(res.Body)
 if err != nil {
  log.Errorf("[opensearch.search] unable to read in response body:\n%v", err)
  return nil, err
 }
 if res.IsError() {
  err := handleBodyError(res.Status(), bodyBytes)
  log.Errorf("[opensearch.search] OpenSearch responded with error:\n%v", err)
  return nil, err
 }
 // If we're searching in all indices, we want to change the shape of the response
 genericResult := index == "_all"
 parsedResult, err := parseSearchResult(bodyBytes, genericResult)
 if err != nil {
  log.Errorf("[opensearch.search] unable to parse search response:\n%v", err)
  return nil, err
 }
 return parsedResult, nil
}
```

By encapsulating the OpenSearch functionality within the `Client` struct and implementing the `OpenSearchConnection` interface, we ensure a modular and extensible design for interacting with OpenSearch in our application.

### Connecting to OpenSearch

Much like we saw in the previous section on how to establish a connection to our data persistance layer using Gorm, we'll now examine how to establish a connection to OpenSearch and leverage its capabilities within our application. We'll begin by introducing the `InitializeClient()` function, which plays a crucial role in initializing the OpenSearch connection. This function retrieves configuration values from the environment and sets up the necessary components for communication with OpenSearch. Let's dive into the details.

```go
var IndexConnection OpensearchConnection

func InitializeClient() error {
    esConfig := config.Config.Opensearch

    config := opensearch.Config{
        Addresses: esConfig.Addresses,
        Username:  esConfig.Username,
        Password:  esConfig.Password,
    }

 if len(esConfig.CertificatePath) > 0 {
  log.Infof("Reading certificate from %s...", esConfig.CertificatePath)
  esCert, err := certificateFileReader(esConfig.CertificatePath)
  if err != nil {
   log.Errorf("[opensearch.initialization] Unable to read in CA certificate file at %s => %v", esConfig.CertificatePath, err)
   return err
  }
  esCa := x509.NewCertPool()
  if ok := esCa.AppendCertsFromPEM(esCert); !ok {
   err := errors.New("unable to add certificate to certificate pool")
   log.Errorf("[opensearch.initialization] Encountered an error adding certificate to pool => %v", err)
   return err
  }

  config.Transport = &http.Transport{
   TLSClientConfig: &tls.Config{RootCAs: esCa},
  }
  log.Info("Successfully loaded certificate")
 }

    client, err := opensearch.NewClient(config)
    if err != nil {
        log.Errorf("[opensearch.initialization] Unable to create elasticsearch connection => %v", err)
        return err
    }

    IndexConnection = &Client{DB: client}

    return nil
}
```

In the given code block, the `InitializeClient()` function demonstrates how to establish a connection to OpenSearch. It starts by retrieving the OpenSearch configuration from the environment settings. The configuration includes addresses, username, and password required for the connection.

To ensure secure communication, the function supports certificate-based authentication. If a certificate path is provided in the configuration, the function reads the certificate file and adds it to the certificate pool.

Once the necessary configurations are set, the function creates an OpenSearch client using the specified settings. The created client is assigned to the `DB` field of the `Client` struct, which is then assigned to the `IndexConnection` variable. This package-level variable allows other parts of the application to access the OpenSearch client conveniently.

By utilizing the `InitializeClient()` function, we can establish a connection to OpenSearch and prepare the client for subsequent database operations.

### Synching with the Database

Having set up the `OpenSearchConnection` interface, we can now leverage [gorm hooks](https://gorm.io/docs/hooks.html) to synchronize database operations with OpenSearch. This synchronization ensures that when CRUD operations are performed on the data model, the corresponding changes are also reflected in OpenSearch.

We will begin by integrating hooks into our data models. Gorm offers the following hooks to handle specific events:

- `AfterCreate`
- `AfterUpdate`
- `AfterDelete`

For example, let's consider the `AfterDelete` hook implementation below:

```go
func (i *Ingredient) AfterDelete(txn *gorm.DB) error {
  client := index.IndexConnection
  err := client.Delete("ingredients", i.ID)
  if err != nil {
    log.Errorf("Unable to delete ingredient from OpenSearch: %v", err)
  }
}
```

In this code snippet, the `AfterDelete` hook is used to synchronize the deletion of an ingredient from the OpenSearch index. The hook retrieves the OpenSearch connection from `index.IndexConnection` and calls the `Delete` method to remove the corresponding ingredient using its ID. If any error occurs during the deletion process, an error message is logged.

By leveraging Gorm hooks, we can seamlessly integrate OpenSearch with our data models and ensure that database operations trigger synchronized actions in OpenSearch, maintaining data consistency between the two systems.

## Summary

In this article, we have covered the essential topics that form the foundation of developing robust applications in Go. We started by creating a Go-based project skeleton using GitLab, Docker, and Go-based linting and build tools, ensuring a standardized and efficient development process. We then delved into creating a flexible application configuration using envconfig, enabling us to easily manage different deployment environments.

Next, we explored Gorm, a powerful ORM library, and integrated it into our application to provide seamless data persistence capabilities. With Gorm, we were able to interact with databases effortlessly, abstracting away the complexities of database operations.

Lastly, we discussed OpenSearch and learned how to index and search data efficiently, enhancing the search capabilities of our applications. By integrating OpenSearch, we empowered our applications with robust search functionality, enabling users to find relevant information quickly.

Building upon the knowledge gained from this post, we are now ready to take the next step in our journey. In the upcoming article, we will implement a web server using Gin-Gonic, a fast and flexible HTTP framework for Go. With Gin-Gonic, we will be able to create a powerful and scalable API layer, seamlessly integrating our previous topics into a comprehensive application architecture.

By combining the concepts of project structure, application configuration, data persistence with Gorm, and search capabilities with OpenSearch, we have established a solid foundation for developing sophisticated applications in Go. Join us in the next post as we continue to leverage the strengths of Go's ecosystem and build powerful applications with Gin-Gonic as our web server framework.

## References

- gorm documentation - <https://gorm.io>
- gormigrate - <https://github.com/go-gormigrate/gormigrate>
- Creating an Opinionated GraphQL server with Go - Part 3 - <https://dev.to/cmelgarejo/creating-an-opinionated-graphql-server-with-go-part-3-3aoi>
- The Go client for Elasticsearch: Working with data - <https://www.elastic.co/blog/the-go-client-for-elasticsearch-working-with-data>
