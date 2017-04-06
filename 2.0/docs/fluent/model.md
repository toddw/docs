# Model

Models are the Swift representations of the data in your database. As such, they are central to most of Fluent's APIs. 

This guide is an overview of the protocol requirements and methods associated with models.

!!! seealso
    Check out the [getting started](getting-started.md) guide for an introductory overview on using models.

## CRUD

Models have several basic methods for creating, reading, updating, and deleting.

### Save

Persists the entity into the data store and sets the `id` property.

```swift
let pet = Pet(name: "Spud", age: 2)
try pet.save()
```

### Find

Finds the model with the supplied identifier or returns `nil`.

```swift
guard let pet = try Pets.find(42) else {
    throw Abort.notFound
}
print(pet.name)
```

### Delete

Deletes the entity from the data store if the entity has previously been fetched or saved.

```swift
try pet.delete()
```

### All

Returns all entities for this model.

```swift
for pet in try Pets.all() {
    print(pet.name)
}
```

### Count

Returns a count of all entities for this model.

```swift
let count = try Pets.count()
```

### Chunk

Returns chunked arrays of a supplied size for all of the entities for this model.

This is a great way to parse through all models of a large data set.

```swift
try Pets.chunk(20) { pets in
    //
}
```

### Query

Creates a `Query` instance for this `Model`.

```swift
let query = try Pet.makeQuery()
```

To learn more about crafting complex queries, see the [query](query.md) section.

## Convenience

### Assert Exists

The identifier property of a model is optional since models may not have been saved yet.

You can get the identifier or throw an error if the model has not been saved yet by calling `assertExists()`.

```swift
let id = try pet.assertExists()
print(id) // not optional
```

## Life Cycle

The following life-cycle methods can be implemented on your model to hook into internal operations.

```swift
/// Called before the entity will be created.
/// Throwing will cancel the creation.
func willCreate() throws

/// Called after the entity has been created.
func didCreate()

/// Called before the entity will be updated.
/// Throwing will cancel the update.
func willUpdate() throws

/// Called after the entity has been updated.
func didUpdate()

/// Called before the entity will be deleted.
/// Throwing will cancel the deletion.
func willDelete() throws

/// Called after the entity has been deleted.
func didDelete()
```

!!! note
    Throwing in a `willFoo()` method will cancel the operation.

Here's an example of implementing the `didDelete` method.

```swift
final class Pet: Model {
    ... 

    func didDelete() {
        print("Deleted \(name)")
    }
}
```

## Entity

Entity is the base Fluent protocol that Model conforms to. It is responsible for providing all information the 
database or query may need when saving, fetching, or deleting your models.

### Name

The singular relational name of this model. Also used for internal storage. Example: Pet = "pet".

This value should usually not be overriden. 

```swift
final class Pet: Model {
    static let name = "pet"
}
```

### Entity

The plural relational name of this model. Used as the collection or table name. 

Example: Pet = "pets".

This value should be overriden if the table name for your model is non-standard.

```swift
final class Pet: Model {
    static let entity = "pets"
}
```

### ID Type

The type of identifier used for both the local and foreign id keys. 

Example: uuid, integer, etc.

This value should be overriden if a particular model in your database uses a different ID type.

```swift
final class Pet: Model {
    static let idType = .uuid
}
```

This can also be overridden at the database level using config.

`Config/fluent.json`

```json
{
    "idType": "uuid"
}
```

Or programatically.

```swift
drop.database?.idType = .uuid
```

### Key Naming Convention

The naming convetion to use for foreign id keys, table names, etc.

Example: snake_case vs. camelCase.

This value should be overridden if a particular model in your database uses a different key naming convention.

```swift
final class Pet: Model {
    static let keyNamingConvention = .snake_case
}
```

This can also be overridden at the database level using config.

`Config/fluent.json`

```json
{
    "keyNamingConvention": "snake_case"
}
```

Or programatically.

```swift
drop.database?.keyNamingConvention = .snake_case
```

### ID Key

The name of the column that corresponds to this entity's identifying key.

The default is 'database.driver.idKey', and then "id"


```swift
final class Pet: Model {
    static let idKey = "id"
}
```

### Foreign ID Key

The name of the column that points to this entity's id when referenced from other tables or collections.

Example: "foo_id".

```swift
final class Pet: Model {
    static let foreignIdKey = "pet_id"
}
```