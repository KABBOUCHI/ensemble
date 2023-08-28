## Introduction

Ensemble is a [Laravel](https://laravel.com)-inspired object-relational mapper (ORM) that makes it enjoyable to interact with your database. When using Ensemble, each database table has a corresponding "Model" that is used to interact with that table. In addition to retrieving records from the database table, Ensemble models allow you to insert, update, and delete records from the table as well.

## Model Conventions

Ensemble Models will generally live inside the `models` module. Let's examine a basic model and discuss some of Ensemble's key conventions:

```rust
use ensemble::Model;

#[derive(Debug, Model)]
struct Flight {
    pub id: u64,
    pub name: String,
}
```

### Table Names

After glancing at the example above, you may have noticed that we did not tell Ensemble which database table corresponds to our Flight model. By convention, the "snake case", plural name of the class will be used as the table name unless another name is explicitly specified. So, in this case, Ensemble will assume the Flight model stores records in the flights table, while an AirTrafficController model would store records in an air_traffic_controllers table.

If your model's corresponding database table does not fit this convention, you may manually specify the model's table name using the `#[ensemble(table)]` attribute:

```rust
use ensemble::Model;

#[derive(Debug, Model)]
#[ensemble(table = "my_flights")]
struct Flight {
    pub id: u64,
    pub name: String,
}
```

### Column Names

By default, Ensemble assumes that your table columns will match the names of the fields on your model. If you would like to specify a different column name, you can use the `#[model(column)]` attribute:

```rust
use ensemble::Model;

#[derive(Debug, Model)]
struct Flight {
    pub id: u64,
    #[model(column = "name")]
    pub flight_name: String,
}
```

### Primary Keys

Ensemble requires every model to have a primary key. If your model includes an `id` field, Ensemble will assume that this is your primary key. If necessary, you may mark any other field as the primary field using the `#[model(primary)]` attribute:

```rust
use ensemble::Model;

#[derive(Debug, Model)]
struct Flight {
    #[model(primary)]
    pub flight_id: u64,
    pub name: String,
}
```

In addition, if the primary key is of type `u64`, Ensemble will use the database's incrementing feature and automatically give it a value on new instances of the model. If you wish to use a non-incrementing primary key, you must use a different type or opt-out with the `#[model(incrementing = false)]` attribute:

```rust
use ensemble::Model;

#[derive(Debug, Model)]
struct Flight {
    #[model(incrementing = false)]
    pub id: u64,
    pub name: String,
}
```

#### "Composite" Primary Keys

Like we've said, Ensemble requires each model to have at least one uniquely identifying "ID" that can serve as its primary key. "Composite" primary keys are not supported by Ensemble models. However, you are free to add additional multi-column, unique indexes to your database tables in addition to the table's uniquely identifying primary key.

### UUID Keys

Instead of using auto-incrementing integers as your Ensemble model's primary keys, you may choose to use UUIDs instead. UUIDs are universally unique alpha-numeric identifiers that are 36 characters long.

If you would like a model to use a UUID key instead of an auto-incrementing integer key, you may use the `uuid::Uuid` type on the model's primary key, and mark it with the `#[model(uuid)]` attribute:

```rust
use uuid::Uuid;
use ensemble::Model;

#[derive(Debug, Model)]
struct Flight {
    #[uuid]
    pub id: Uuid,
    pub name: String,
}
```

By default, Ensemble will generate `v4` UUIDs (make sure you've enabled the `v4` feature of the `uuid` crate). If you'd rather use a different version, you may specify it as an argument to the `#[model(uuid)]` attribute:

```rust
use uuid::Uuid;
use ensemble::Model;

#[derive(Debug, Model)]
struct Flight {
    #[model(uuid = "v6")]
    pub id: Uuid,
    pub name: String,
}
```

### Timestamps

If your model includes created_at and updated_at fields, Ensemble will automatically set these column's values when models are created or updated, like so:

```rust
use ensemble::{types::DateTime, Model};

#[derive(Debug, Model)]
struct Flight {
    pub id: u64,
    pub name: String,
    pub created_at: DateTime,
    pub updated_at: DateTime,
}
```

If you need to customize the names of the fields used to store the timestamps, you may use the `#[model(created_at)]` and `#[model(updated_at)]` attributes:

```rust
use ensemble::{types::DateTime, Model};

#[derive(Debug, Model)]
struct Flight {
    pub id: u64,
    pub name: String,

    #[model(created_at)]
    pub creation_date: DateTime,
    #[model(updated_at)]
    pub updated_date: DateTime,
}
```

### Default Values

Ensemble models automatically implement the `Default`trait, which means you can use the `default` method to create a new instance of the model with all its fields set to their default values. You can customize the default value of a field by using the `#[model(default)]` attribute:

```rust
use ensemble::{types::DateTime, Model};

#[derive(Debug, Model)]
struct Flight {
    pub id: u64,
    pub name: String,
    #[model(default = true)]
    pub is_commercial: bool,
}
```

## Retrieving Models

Once you have created a model and its associated database table, you are ready to start retrieving data from your database. You can think of each Ensemble model as a powerful query builder allowing you to fluently query the database table associated with the model. The model's `all` method will retrieve all of the records from the model's associated database table:

```rust
use crate::models::Flight;

for flight in Flight::all().await.unwrap() {
    println!("{}", flight.name);
}
```

### Building Queries

The Ensemble `all` method will return all of the results in the model's table. However, since each Ensemble model serves as a query builder, you may add additional constraints to queries and then invoke the get method to retrieve the results:

```rust
let flights = Flight::query()
    .r#where("active", '=', 1)
    .orderBy("name", "asc")
    .take(10)
    .get().await.unwrap();
```

### Refreshing Models

If you already have an instance of an Ensemble model that was retrieved from the database, you can "refresh" the model using the `fresh` method. The fresh method will re-retrieve the model from the database. The existing model instance will not be affected:

```rust
let flight = Flight::query().r#where("number", "=", "FR 900").first().await().unwrap();

let fresh_flight = flight.fresh();
```

## Retrieving Single Models / Aggregates

In addition to retrieving all of the records matching a given query, you may also retrieve single records using the `find` or `first` methods. Instead of returning a collection of models, these methods return a single model instance:

```rust
use crate::models::Flight;

// Retrieve a model by its primary key...
let flight = Flight::find(1).await.unwrap();

// Retrieve the first model matching the query constraints...
let flight = Flight::query().r#where("active", "=", 1).first().await.unwrap();
```

## Inserting & Updating Models

### Inserts

Of course, when using Ensemble, we don't only need to retrieve models from the database. We also need to insert new records. Thankfully, Ensemble makes it simple. To insert a new record into the database, you should instantiate a new model instance and set attributes on the model. Then, call the `save` method on the model instance:

```rust
use crate::models::Flight;
use axum::{Json, response::IntoResponse, http::StatusCode};

#[derive(serde::Deserialize)]
struct FlightRequest {
    name: String,
}

async fn store_flight(Json(request): Json<FlightRequest>) -> impl IntoResponse {
    let flight = Flight {
        name: request.name,
    };
    flight.save().await.unwrap();

    (StatusCode::201, "Flight saved successfully".to_string())
}
```

In this example, we assign the name field from the incoming HTTP request to the name attribute of the model instance. When we call the `save` method, a record will be inserted into the database. The model's `created_at` and `updated_at` timestamps will automatically be set when the save method is called, so there is no need to set them manually.

Alternatively, you may use the `create` method to "save" a new model using a single statement. The inserted model instance will be returned to you by the `create` method:

```rust
use crates::models::Flight;

let flight = Flight::create(Flight {
    name: "London to Paris",
    ..Flight::default(),
}).await.unwrap();
```

### Updates

The `save` method may also be used to update models that already exist in the database. To update a model, you should retrieve it and set any attributes you wish to update. Then, you should call the model's `save` method. Again, the `updated_at` timestamp will automatically be updated, so there is no need to manually set its value:

```rust
use crates::models::Flight;

let mut flight = Flight::find(1).await.unwrap();

flight.name = "Paris to London";

flight.save().await.unwrap();
```

#### Mass Updates

Updates can also be performed against models that match a given query. In this example, all flights that are active and have a destination of San Diego will be marked as delayed:

```rust
Flight::query()
    .r#where("active", '=', 1)
    .r#where("destination", '=', "San Diego")
    .update(&[("delayed", 1.into())])
    .await.unwrap();
```

The update method expects an array of tuples containing representing column and value pairs for the columns that should be updated. The update method returns the number of affected rows.

## Deleting Models

To delete a model, you may call the delete method on the model instance:

```rust
use crate::models::Flight;

let flight = Flight::find(1).await.unwrap();

flight.delete().await.unwrap();
```

You may call the truncate method to delete all of the model's associated database records. The truncate operation will also reset any auto-incrementing IDs on the model's associated table:

```rust
Flight::query().truncate().await.unwrap();
```

### Deleting Models Using Queries

Of course, you may build an Ensemble query to delete all models matching your query's criteria. In this example, we will delete all flights that are marked as inactive:

```rust
let deleted = Flight::query().r#where("active", '=', 0).delete().await.unwrap();
```