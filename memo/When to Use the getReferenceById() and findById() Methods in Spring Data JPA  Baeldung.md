---
created: 2024-01-09T09:19:33 (UTC +09:00)
tags: []
source: https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods
author: 
---

# When to Use the getReferenceById() and findById() Methods in Spring Data JPA | Baeldung

> ## Excerpt
> Learn the difference between getReferenceById(ID) and findById(ID) and find the situation when each might be more suitable.

---
## 1\. Overview[](https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods#overview)

[_JpaRepository_](https://www.baeldung.com/spring-data-repositories) provides us with basic methods for CRUD operations. However, some of them are not so straightforward, and sometimes, it’s hard to identify which method would be the best for a given situation.

**_getReferenceById(ID)_ and _findById(ID)_ are the methods that often create such confusion.** These methods are new API names for _getOne(ID), findOne(ID),_ and _getById(ID)._

In this tutorial, we’ll learn the difference between them and find the situation when each might be more suitable.

Let’s start with the simplest one out of these two methods. This method does what it says, and usually, developers don’t have any issues with it. It simply finds an entity in a repository given a specific ID:

```java
@Override Optional<T> findById(ID id);
```

The method returns an _[Optional](https://www.baeldung.com/java-optional)._ Thus, assuming it would be empty if we passed a non-existent ID is correct.

The method uses eager loading under the hood, so we’ll send a request to our database whenever we call this method. Let’s check an example:

```java
public User findUser(long id) { log.info("Before requesting a user in a findUser method"); Optional<User> optionalUser = repository.findById(id); log.info("After requesting a user in a findUser method"); User user = optionalUser.orElse(null); log.info("After unwrapping an optional in a findUser method"); return user; }
```

This method will generate the following logs:

```plaintext
[2023-12-27 12:56:32,506]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.SimpleUserService - Before requesting a user in a findUser method [2023-12-27 12:56:32,508]-[main] DEBUG org.hibernate.SQL - select user0_."id" as id1_0_0_, user0_."first_name" as first_na2_0_0_, user0_."second_name" as second_n3_0_0_ from "users" user0_ where user0_."id"=? [2023-12-27 12:56:32,508]-[main] TRACE org.hibernate.type.descriptor.sql.BasicBinder - binding parameter [1] as [BIGINT] - [1] [2023-12-27 12:56:32,510]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.SimpleUserService - After requesting a user in a findUser method [2023-12-27 12:56:32,510]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.SimpleUserService - After unwrapping an optional in a findUser method
```

**Spring might batch requests in a transaction but will always execute them.** Overall, _findById(ID)_ doesn’t try to surprise us and does what we expect from it. However, the confusion arises because it has a counterpart that does something similar.

## 3\. _getReferenceById()_[](https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods#getreferencebyid)

This method has a similar signature to _findById(ID):_

```java
@Override T getReferenceById(ID id);
```

Judging by the signature alone, we can assume that this method would throw an exception if the entity doesn’t exist. **It’s true, but it’s not the only difference we have. The main difference between these methods is that _getReferenceById(ID)_ is lazy.** Spring won’t send a database request until we explicitly try to use the entity within a transaction.

### 3.1. Transactions[](https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods#1-transactions)

**Each transaction has a dedicated persistence context it works with.** Sometimes, we can expand the persistence context outside the transaction scope, but it’s not common and useful only for specific scenarios. Let’s check how the persistence context behaves regarding the transactions:

[![](https://www.baeldung.com/wp-content/uploads/2023/12/Eager-Loading-With-findById.png)](https://www.baeldung.com/wp-content/uploads/2023/12/Eager-Loading-With-findById.png)

Within a transaction, all the entities inside the persistence context have a direct representation in the database. This is a [managed state](https://www.baeldung.com/java-optional). Thus, all the changes to the entity will be reflected in the database. Outside the transaction, the entity moved to a [detached state](https://www.baeldung.com/hibernate-entity-lifecycle#detached-entity), and changes won’t be reflected until the entity is moved back to the managed state.

Lazy-loaded entities behave slightly differently. Spring won’t load them until we explicitly use them in the persistence context:

[![](https://www.baeldung.com/wp-content/uploads/2023/12/Lazy-Loading-without-using.png)](https://www.baeldung.com/wp-content/uploads/2023/12/Lazy-Loading-without-using.png)Spring will allocate an empty proxy placeholder to fetch the entity from the database lazily. **However, if we don’t do this, the entity will remain an empty proxy outside the transaction, and any call to it will result in a** _**[LazyInitializationException](https://www.baeldung.com/hibernate-initialize-proxy-exception).**_ However, if we do call or interact with the entity in the way it will require the internal information, the actual request to the database will be made:

[![](https://www.baeldung.com/wp-content/uploads/2023/12/Lazy-Loading-with-using.png)](https://www.baeldung.com/wp-content/uploads/2023/12/Lazy-Loading-with-using.png)

### 3.2. Non-transactional Services[](https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods#2-non-transactional-services)

Knowing the behavior of transactions and the persistence context, let’s check the following non-transactional service, which calls the repository. **The _findUserReference_ doesn’t have a persistence context connected to it, and _getReferenceById_ will be executed in a separate transaction:**

```java
public User findUserReference(long id) { log.info("Before requesting a user"); User user = repository.getReferenceById(id); log.info("After requesting a user"); return user; }
```

This code will generate the following log output:

```plaintext
[2023-12-27 13:21:27,590]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.TransactionalUserReferenceService - Before requesting a user [2023-12-27 13:21:27,590]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.TransactionalUserReferenceService - After requesting a user
```

As we can see, there’s no database request. **After understanding the lazy loading, Spring assumes that we might not need it if we don’t use the entity within.** Technically, we cannot use it because our only transaction is one inside the _getReferenceById_ method. Thus, the _user_ we returned will be an empty proxy, which will result in an exception if we access its internals:

```java
public User findAndUseUserReference(long id) { User user = repository.getReferenceById(id); log.info("Before accessing a username"); String firstName = user.getFirstName(); log.info("This message shouldn't be displayed because of the thrown exception: {}", firstName); return user; }
```

### 3.3. Transactional Service[](https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods#3-transactional-service)

Let’s check the behavior if we’re using a [_@Transactional_](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring) service:

```java
@Transactional public User findUserReference(long id) { log.info("Before requesting a user"); User user = repository.getReferenceById(id); log.info("After requesting a user"); return user; }
```

This will give us a similar result for the same reason as in the previous example, as we don’t use the entity inside our transaction:

```plaintext
[2023-12-27 13:32:44,486]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.TransactionalUserReferenceService - Before requesting a user [2023-12-27 13:32:44,486]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.TransactionalUserReferenceService - After requesting a user
```

Also, any attempts to interact with this user outside of this transactional service method would cause an exception:

```java
@Test void whenFindUserReferenceUsingOutsideServiceThenThrowsException() { User user = transactionalService.findUserReference(EXISTING_ID); assertThatExceptionOfType(LazyInitializationException.class) .isThrownBy(user::getFirstName); }
```

However, now, the _findUserReference_ method defines the scope of our transaction. This means that we can try to access the _user_ in our service method, and it should cause the call to the database:

```java
@Transactional public User findAndUseUserReference(long id) { User user = repository.getReferenceById(id); log.info("Before accessing a username"); String firstName = user.getFirstName(); log.info("After accessing a username: {}", firstName); return user; }
```

The code above would output the messages in the following order:

```plaintext
[2023-12-27 13:32:44,331]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.TransactionalUserReferenceService - Before accessing a username [2023-12-27 13:32:44,331]-[main] DEBUG org.hibernate.SQL - select user0_."id" as id1_0_0_, user0_."first_name" as first_na2_0_0_, user0_."second_name" as second_n3_0_0_ from "users" user0_ where user0_."id"=? [2023-12-27 13:32:44,331]-[main] TRACE org.hibernate.type.descriptor.sql.BasicBinder - binding parameter [1] as [BIGINT] - [1] [2023-12-27 13:32:44,331]-[main] INFO com.baeldung.spring.data.persistence.findvsget.service.TransactionalUserReferenceService - After accessing a username: Saundra
```

The request to the database wasn’t made when we called _getReferenceById(),_ but when we called _user.getFirstName()._ 

### 3.3. Transactional Service With a New Repository Transaction[](https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods#3-transactional-service-with-a-new-repository-transaction)

Let’s check a bit more complex example. Imagine we have a repository method that creates a separate transaction whenever we call it:

```java
@Override @Transactional(propagation = Propagation.REQUIRES_NEW) User getReferenceById(Long id);
```

**_[Propagation.REQUIRES\_NEW](https://www.baeldung.com/spring-transactional-propagation-isolation#6requiresnew-propagation)_ means the outer transaction won’t propagate, and the repository method will create its persistence context.** In this case, even if we use a transactional service, Spring will create two separate persistence contexts that won’t interact, and any attempts to use the _user_ will cause an exception:

```java
@Test void whenFindUserReferenceUsingInsideServiceThenThrowsExceptionDueToSeparateTransactions() { assertThatExceptionOfType(LazyInitializationException.class) .isThrownBy(() -> transactionalServiceWithNewTransactionRepository.findAndUseUserReference(EXISTING_ID)); }
```

We can use a couple of different propagation configurations to create more complex interactions between transactions, and they can yield different results.

## 4\. Conclusion[](https://www.baeldung.com/spring-data-jpa-getreferencebyid-findbyid-methods#conclusion)

The main difference between _findById()_ and _getReferenceById()_ is when they load the entities into the persistence context. Understanding this might help to implement optimizations and avoid unnecessary database lookups. This process is tightly connected to the transactions and their propagation. That’s why the relationships between transactions should be observed.

As usual, all the code used in this tutorial is available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/persistence-modules/spring-data-jpa-repo-4).
