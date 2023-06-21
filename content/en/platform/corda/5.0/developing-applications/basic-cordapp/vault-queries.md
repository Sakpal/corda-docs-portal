---
date: '2023-05-03'
title: "Vault Queries"
version: 'Corda 5.0'
menu:
  corda5:
    identifier: corda5-develop-first-cordapp-vault-queries
    parent: corda5-develop-first-cordapp
    weight: 6000
---

# Vault Queries

## What is a Vault Named Query?

A vault named query is essentially a database query that can be defined by Corda users. The user can define the following:

* The name of the query
* The query functionality (state type the query will work on and the `WHERE` clause)
* Optional filtering logic of the result set
* Optional transformation logic of the result set
* Optional collection logic of the result set
* The query creator needs to follow a few pre-defined steps in order for the query to be registered and to be usable.

## How a State is Represented in the Database?

Each state type can be represented as a JSON string (`custom_representation` column in the database), we can use that JSON representation to write our vault named queries. However, this representation needs to be pre-defined.

 In order to do that we need an implementation of the `net.corda.v5.ledger.utxo.query.json.ContractStateVaultJsonFactory<T>` interface. The `<T>` parameter is the type of the state we want to represent.

Let’s say we have a very simple state type called `TestState` and a simple contract called `TestContract`. We’ll use this state throughout this documentation. It would look like this:

```kotlin
package com.r3.corda.demo.contract

class TestContract : Contract {
    override fun verify(transaction: UtxoLedgerTransaction) {}
}

@BelongsToContract(TestContract::class)
class TestState(
    val testField: String,
    private val participants: List<PublicKey>
) : ContractState {
    override fun getParticipants(): List<PublicKey> = participants
}
```

```java
package com.r3.corda.demo.contract;

class TestContract implements Contract {
    @Override
    public void verify(@NotNull UtxoLedgerTransaction transaction) {}
}

@BelongsToContract(TestContract.class)
public class TestState implements ContractState {

    private final List<PublicKey> participants;
    private final String testField;

    public TestState(@NotNull String testField, @NotNull List<PublicKey> participants) {
        this.testField = testField;
        this.participants = participants;
    }

    @NotNull
    @Override
    public List<PublicKey> getParticipants() {
        return new LinkedList<>(this.participants);
    }

    @NotNull
    public String getTestField() {
        return testField;
    }
}
```

This contract has no verification logic and should only be used for testing purposes. The state itself has a `testField` property defined that we use for JSON representation and when constructing our query.

We want to represent our state as a JSON string. Since our state has only one property (`testField`) the `ContractStateVaultJsonFactory` implementation is rather simple:

```kotlin
class TestStateJsonFactory : ContractStateVaultJsonFactory<TestState> {
    override fun getStateType(): Class<TestState> = TestState::class.java

    override fun create(state: TestState, jsonMarshallingService: JsonMarshallingService): String {
        return jsonMarshallingService.format(state)
    }
}
```

```java
class TestUtxoStateJsonFactory implements ContractStateVaultJsonFactory<TestUtxoState> {
    @NotNull
    @Override
    public Class<TestUtxoState> getStateType() {
        return TestUtxoState.class;
    }

    @Override
    @NotNull
    public String create(@NotNull TestUtxoState state, @NotNull JsonMarshallingService jsonMarshallingService) {
        return jsonMarshallingService.format(state);
    }
}
```

After the output state has been finalised it will be represented as the following in the database (`custom_representation column`):

```java
{
  "net.corda.v5.ledger.utxo.ContractState" : {
    "stateRef": "<TransactionID>:<StateIndex>"
  },
  "com.r3.corda.demo.contract.TestState" : {
    "testField": ""
  } 
}
```

 {{< note >}}
    The `net.corda.v5.ledger.utxo.ContractState` field will always be part of the JSON representation no matter which state type we are using.
 {{< /note >}}

We will use this representation to create our vault named queries in the next section.

## How to Create and Register a Vault Named Query?

The vault named queries has two crucial elements. The first one is registration, this means the query will be stored on sandbox creation time and can be executed later on.

Vault named queries need to be part of a contract CPK. The contract CPK needs to be uploaded in order for a vault named query to be installed.

To create and register a query the `net.corda.v5.ledger.utxo.query.VaultNamedQueryFactory` interface must be implemented.

### Basic Vault Named Query Registration Example

A simple vault named query implementation for this `TestState` would look like this:

{{< note >}}
This example has no filtering, mapping, or collecting logic.
{{< /note >}}

```kotlin
class DummyCustomQueryFactory : VaultNamedQueryFactory {
    override fun create(vaultNamedQueryBuilderFactory: VaultNamedQueryBuilderFactory) {
        vaultNamedQueryBuilderFactory.create("DUMMY_CUSTOM_QUERY")
            .whereJson(
                "WHERE visible_states.custom_representation -> 'com.r3.corda.demo.contract.TestState' " +
                        "->> 'testField' = :testField"
            )
            .register()
    }
}
```

```java
public class JsonQueryFactory implements VaultNamedQueryFactory {
    @Override
    public void create(@NotNull VaultNamedQueryBuilderFactory vaultNamedQueryBuilderFactory) {
        vaultNamedQueryBuilderFactory.create("DUMMY_CUSTOM_QUERY")
                .whereJson(
                        "WHERE visible_states.custom_representation -> 'com.r3.corda.demo.contract.TestState' " +
                                "->> 'testField' = :testField"
                )
                .register();
    }
}
```

Let’s analyse this code snippet a bit more in depth. 
As mentioned above, we are inheriting from the interface called `VaultNamedQueryFactory`, this interface has one method:

`void create(@NotNull VaultNamedQueryBuilderFactory vaultNamedQueryBuilderFactory);`

This function will be called on startup and we should define how our query will operate in it with the following steps:

* To create a vault named query with a given name, in our case it's `DUMMY_CUSTOM_QUERY`, call `vaultNamedQueryBuilderFactory.create()`.
* To define how a query's `WHERE` clause will work, call `whereJson()`.
    {{< note >}}
    Always start with the actual `WHERE` statement and then write the rest of the clause. Fields need to be prefixed with the `visible_states. qualifier`. Since `visible_states.custom_representation` is a JSON column, we can use some JSON specific  operations, more info here.
     * Parameters can be used in the query in a :parameter format. We are using a parameter called :testField which we will be able to set when executing this query. This works similarly to popular Java SQL libraries such as Hibernate.
    {{< /note >}}
* To finalise query creation and to store the created query in the registry to be executed later, call `register()`. This call needs to be the last step when defining a query.

### Filtering, transforming and collecting (a complex query example)

We are going to use the same state as in the first example just evolve our query a little bit. Let’s say we want to add some extra logic to our query:

* Only keep the results that have “Alice” in their participant list.
* Transform the result set to only keep the transaction IDs.
* Collect the result set into one single integer (number of results).

These optional logics will always be applied in the following order (no matter in which order they are called):

1. Filtering
2. Transforming
3. Collecting

The query `whereJson` will return `StateAndRef` objects and the data going into the filtering and transforming logic consists of `StateAndRefs`.

#### Filtering

To create a filtering logic we need to implement the `net.corda.v5.ledger.utxo.query.VaultNamedQueryStateAndRefFilter<T>` interface. The `<T>` type here is the state type that will be returned from the database, in our case it’s `TestState`. This interface has only one function:

`@NotNull Boolean filter(@NotNull StateAndRef<T> data, @NotNull Map<String, Object> parameters);`

This will define whether or not to keep the given element (`row`) from the result set. Elements returning true will be kept and the rest will be discarded.

In our example we want to keep the elements that have “Alice” in their participant list. This filter would look like this:

```kotlin
class DummyCustomQueryFilter : VaultNamedQueryStateAndRefFilter<TestState> {
    override fun filter(data: StateAndRef<TestUtxoState>, parameters: MutableMap<String, Any>): Boolean {
        return data.state.contractState.participantNames.contains("Alice")
    }
}
```

```java
class DummyCustomQueryFilter implements VaultNamedQueryStateAndRefFilter<TestUtxoState> {

    @NotNull
    @Override
    public Boolean filter(@NotNull StateAndRef<TestUtxoState> data, @NotNull Map<String, Object> parameters) {
        return data.getState().getContractState().getParticipantNames().contains("Alice");
    }
}
```

#### Transforming

We can create a transformer class to make sure to only keep the transaction IDs of each record. Transformer classes need to implement the `VaultNamedQueryStateAndRefTransformer<T, R>` interface. The `<T>` is the type of results returned from the database, in our case `TestState` and `<R>` is the type we are going to transform our results into and we want to have transaction IDs which are `Strings`.

This interface has one function:

```java
@NotNull`
`R transform(@NotNull T data, @NotNull Map<String, Object> parameters);
```

This will define how each record (“row”) will be transformed (mapped).
Our transformer will look something like this:

```kotlin

class DummyCustomQueryTransformer : VaultNamedQueryStateAndRefTransformer<TestState, String> {
    override fun transform(data: StateAndRef<TestState>, parameters: MutableMap<String, Any>): String {
        return data.ref.transactionId.toString()
    }
}
```

```java

class DummyCustomQueryMapper implements VaultNamedQueryStateAndRefTransformer<TestUtxoState, String> {
    @NotNull
    @Override
    public String transform(@NotNull StateAndRef<TestUtxoState> data, @NotNull Map<String, Object> parameters) {
        return data.getRef().getTransactionId().toString();
    }
}
```

This will transform each element to a `String` object which is the given state’s transaction ID.

#### Collecting

We want to collect our result set into one single integer.

For this, we need the `net.corda.v5.ledger.utxo.query.VaultNamedQueryCollector<R, T>` interface. The `<R>` parameter is the type of the original result set (in our case `String` because of our transformation) and `<T>` is the type we’ll collect into (in our case this is an `Int`).

This interface has only one method:

```java
@NotNull
Result<T> collect(@NotNull List<R> resultSet, @NotNull Map<String, Object> parameters);
```

This will define how we collect our result set, the collector class should look like the following:

```kotlin
class DummyCustomQueryCollector : VaultNamedQueryCollector<String, Int> {
    override fun collect(
        resultSet: MutableList<String>,
        parameters: MutableMap<String, Any>
    ): VaultNamedQueryCollector.Result<Int> {
        return VaultNamedQueryCollector.Result(
            listOf(resultSet.size),
            true
        )
    }
}
```

```java
class DummyCustomQueryCollector implements VaultNamedQueryCollector<String, Integer> {

    @NotNull
    @Override
    public Result<Integer> collect(@NotNull List<String> resultSet, @NotNull Map<String, Object> parameters) {
        return new Result<>(
                List.of(resultSet.size()),
                true
        );
    }
}
```

{{< note >}}
 The query `isDone` should only be set to true if the result set is complete.
{{< /note >}}

#### Registering our Complex Query

In the following example we will register a complex query with a filter, a transformer, and a collector:

```kotlin
class DummyCustomQueryFactory : VaultNamedQueryFactory {
    override fun create(vaultNamedQueryBuilderFactory: VaultNamedQueryBuilderFactory) {
        vaultNamedQueryBuilderFactory.create("DUMMY_CUSTOM_QUERY")
            .whereJson(
                "WHERE visible_states.custom_representation -> 'com.r3.corda.demo.contract.TestState' " +
                        "->> 'testField' = :testField"
            )
            .filter(DummyCustomQueryFilter())
            .map(DummyCustomQueryMapper())
            .collect(DummyCustomQueryCollector())
            .register()
    }
}
```

```java
public class JsonQueryFactory implements VaultNamedQueryFactory {
    @Override
    public void create(@NotNull VaultNamedQueryBuilderFactory vaultNamedQueryBuilderFactory) {
        vaultNamedQueryBuilderFactory.create("DUMMY_CUSTOM_QUERY")
                .whereJson(
                        "WHERE visible_states.custom_representation -> 'com.r3.corda.demo.contract.TestState' " +
                                "->> 'testField' = :testField"
                )
                .filter(new DummyCustomQueryFilter())
                .map(new DummyCustomQueryMapper())
                .collect(new DummyCustomQueryCollector())
                .register();
    }
}
```

{{< note >}}
The collector always need to be the last one in the chain as all previous filtering and mapping functionality will be applied before collecting.
{{< /note >}}

Now that we have our query registered, it’s time to execute it.

### How to Execute a Vault Named Query?

To execute a query we need to use the `UtxoLedgerService`. This can be injected to a flow via `@CordaInject`.
To instantiate our query we need to call the following:

```kotlin
utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Int::class.java)
```

or

```java
utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Integer.class)
```

We provide the name of our query (in our case `DUMMY_CUSTOM_QUERY`) and the return type. Since we collected our result set into an integer in the complex query example we will use `Int` (or `Integer` in Java).

Before executing we can define a couple things:

* Which index our result set should start (`setOffset`), default 0.
* How many results should our query return (`setLimit`), default Int.MAX (2,147,483,647).
* Define named parameters that are in our query and the actual value for them.
    {{< note >}}
    All parameters must be defined otherwise the execution will fail. (`setParameter` or `setParameters`)
    {{</ note >}}
* Each state in the database has a timestamp value for when it was inserted. With this we can set an * upper limit to only return states that were inserted before a given time. (`setTimestampLimit`)

In our case this would look like this:

```kotlin
val resultSet = utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Int::class.java) // instantiate the query
            .setOffset(0) // Start from the beginning
            .setLimit(1000) // Only return 1000 records
            .setParameter("testField", "dummy") // Set the parameter to a dummy value
            .setCreatedTimestampLimit(Instant.now()) // Set the timestamp limit to the current time
            .execute() // execute the query
```

```java
PagedQuery.ResultSet<Integer> resultSet = utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Integer.class) // instantiate the query
                .setOffset(0) // Start from the beginning
                .setLimit(1000) // Only return 1000 records
                .setParameter("testField", "dummy") // Set the parameter to a dummy value
                .setCreatedTimestampLimit(Instant.now()) // Set the timestamp limit to the current time
                .execute(); // execute the query
```

{{< note >}} 
A dummy value is assigned for the `testField` parameter in our query but this can be replaced. We only have one parameter in our example query which is `:testField`.
{{</ note >}} 

Results can be acquired by calling `getResults()` on the `ResultSet`.
Paging can be achieved by increasing the offset until the result set has elements:

```kotlin
var currentOffset = 0;

val resultSet = utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Integer.class) // instantiate the query
                .setOffset(0) // Start from the beginning
                .setLimit(1000) // Only return 1000 records
                .setParameter("testField", "dummy") // Set the parameter to a dummy value
                .setCreatedTimestampLimit(Instant.now()) // Set the timestamp limit to the current time
                .execute()

while (resultSet.results.isNotEmpty()) {
    currentOffset += 1000
    query.setOffset(currentOffset)
    resultSet = query.execute()
}
```

```java
int currentOffset = 0;

ResultSet<Integer> resultSet = utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Integer.class) // instantiate the query
                .setOffset(0) // Start from the beginning
                .setLimit(1000) // Only return 1000 records
                .setParameter("testField", "dummy") // Set the parameter to a dummy value
                .setCreatedTimestampLimit(Instant.now()) // Set the timestamp limit to the current time
                .execute();

while (resultSet.results.isNotEmpty()) {
    currentOffset += 1000;
    query.setOffset(currentOffset);
    resultSet = query.execute();
}
```

Or just calling the `hasNext()` and `next()` functionality:

```kotlin
val resultSet = utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Integer.class) // instantiate the query
                .setOffset(0) // Start from the beginning
                .setLimit(1000) // Only return 1000 records
                .setParameter("testField", "dummy") // Set the parameter to a dummy value
                .setCreatedTimestampLimit(Instant.now()) // Set the timestamp limit to the current time
                .execute()
                
var results = resultSet.results

while (resultSet.hasNext()) {
    results = resultSet.next()
}
```

```java
ResultSet<Integer> resultSet = utxoLedgerService.query("DUMMY_CUSTOM_QUERY", Integer.class) // instantiate the query
                .setOffset(0) // Start from the beginning
                .setLimit(1000) // Only return 1000 records
                .setParameter("testField", "dummy") // Set the parameter to a dummy value
                .setCreatedTimestampLimit(Instant.now()) // Set the timestamp limit to the current time
                .execute();
                
List<Integer> results = resultSet.getResults();

while (resultSet.hasNext()) {
    results = resultSet.next();
}
```
