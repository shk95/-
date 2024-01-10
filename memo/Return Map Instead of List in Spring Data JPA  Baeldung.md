---
created: 2024-01-08T17:55:49 (UTC +09:00)
tags: []
source: https://www.baeldung.com/spring-data-return-map-instead-of-list
author: 
---

# Return Map Instead of List in Spring Data JPA | Baeldung

> ## Excerpt
> Learn how to use custom collections as return types in Spring Data JPA make the design more straightforward and less cluttered with mapping and filtering logic.

---
![announcement - icon](https://www.baeldung.com/wp-content/uploads/2022/04/announcement-icon.png)

Spring Data JPA is a great way to handle the **complexity of JPA with the powerful simplicity of Spring Boot**.

Get started with Spring Data JPA through the guided reference course:

**[\>> CHECK OUT THE COURSE](https://www.baeldung.com/courselsd-NPI-EA-1fOru)**

## 1\. Overview[](https://www.baeldung.com/spring-data-return-map-instead-of-list#overview)

[Spring JPA](https://www.baeldung.com/the-persistence-layer-with-spring-data-jpa) provides a very flexible and convenient API for interaction with databases. However, sometimes, we need to customize it or add more functionality to the returned collections.

Using _[Map](https://www.baeldung.com/java-hashmap)_ as a return type from JPA repository methods might help to create more straightforward interactions between services and databases. **Unfortunately, Spring doesn’t allow this conversion to happen automatically.** In this tutorial, we’ll check how to overcome this and learn some interesting techniques to make our repositories more functional.

## 2\. Manual Implementation[](https://www.baeldung.com/spring-data-return-map-instead-of-list#manual-implementation)

The most apparent approach to the problem when a framework doesn’t provide something, is to implement it ourselves. In this case, JPA allows us to implement the repositories from scratch, skip the entire generation process, or use default methods to get the best of both worlds.

### 2.1. Using _List_[](https://www.baeldung.com/spring-data-return-map-instead-of-list#1-using-list)

We can implement a method to map the resulting list into the map. [Stream API](https://www.baeldung.com/java-8-streams) helps greatly with this task, allowing almost one-liner implementation:

```java
default Map<Long, User> findAllAsMapUsingCollection() { return findAll().stream() .collect(Collectors.toMap(User::getId, Function.identity())); }
```

### 2.2. Using _Stream_[](https://www.baeldung.com/spring-data-return-map-instead-of-list#2-using-stream)

We can do a similar thing but use _Stream_ directly. To do so, we can identify a custom method that will return a stream of users. Luckily, Spring JPA supports such return types, and we can benefit from autogeneration:

```java
@Query("select u from User u") Stream<User> findAllAsStream();
```

After that, we can implement a custom method that would map the results into the data structure we need:

```java
@Transactional default Map<Long, User> findAllAsMapUsingStream() { return findAllAsStream() .collect(Collectors.toMap(User::getId, Function.identity())); }
```

The repository methods that return _Stream_ should be called inside a transaction. In this case, we directly added a [_@Transactional_](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring) annotation to the default method.

### 2.3. Using _Streamable_[](https://www.baeldung.com/spring-data-return-map-instead-of-list#3-using-streamable)

This is a similar approach to the one discussed previously. The only change is that we’ll be using _Streamable_. We need to create a custom method to return it first:

```java
@Query("select u from User u") Streamable<User> findAllAsStreamable();
```

Then, we can map the result appropriately:

```java
default Map<Long, User> findAllAsMapUsingStreamable() { return findAllAsStreamable().stream() .collect(Collectors.toMap(User::getId, Function.identity())); }
```

## 3\. Custom _Streamable_ Wrapper[](https://www.baeldung.com/spring-data-return-map-instead-of-list#custom-streamable-wrapper)

Previous examples showed us quite simple solutions to the problem. However, suppose we have several different operations or data structures to which we want to map our results. **In that case, we can end up with unwieldy mappers scattered around our code or multiple repository methods that do similar things.**

A better approach might be to create a dedicated class representing a collection of entities and place all the methods connected to the operations on the collection inside. To do so, we’ll be using _Streamable_.

As was shown previously, Spring JPA understands _Streamable_ and can map the result to it. Interestingly, we can extend _Streamable_ and provide it with convenient methods. Let’s create a _Users_ class that would represent a collection of _User_ objects:

```java
public class Users implements Streamable<User> { private final Streamable<User> userStreamable; public Users(Streamable<User> userStreamable) { this.userStreamable = userStreamable; } @Override public Iterator<User> iterator() { return userStreamable.iterator(); } // custom methods }
```

To make it work with JPA, we should follow a simple convention. **First, we should implement _Streamable_, and secondly, provide the way Spring will be able to initialize it.** The initialization part can be addressed either by a public constructor that takes _Streamable_ or static factories with names _of(Streamable<T>)_ or _valueOf(Streamable<T>)_.

After that, we can use _Users_ as a return type of JPA repository methods:

```java
@Query("select u from User u") Users findAllUsers();
```

Now, we can place the method we kept in the repository directly in the _Users_ class:

```java
public Map<Long, User> getUserIdToUserMap() { return stream().collect(Collectors.toMap(User::getId, Function.identity())); }
```

The best part is that we can use all the methods connected to the processing or mapping of the _User_ entities. Let’s say we want to filter out users by some criteria:

```java
@Test void fetchUsersInMapUsingStreamableWrapperWithFilterThenAllOfThemPresent() { Users users = repository.findAllUsers(); int maxNameLength = 4; List<User> actual = users.getAllUsersWithShortNames(maxNameLength); User[] expected = { new User(9L, "Moe", "Oddy"), new User(25L, "Lane", "Endricci"), new User(26L, "Doro", "Kinforth"), new User(34L, "Otho", "Rowan"), new User(39L, "Mel", "Moffet") }; assertThat(actual).containsExactly(expected); }
```

Also, we can group them in some way:

```java
@Test void fetchUsersInMapUsingStreamableWrapperAndGroupingThenAllOfThemPresent() { Users users = repository.findAllUsers(); Map<Character, List<User>> alphabeticalGrouping = users.groupUsersAlphabetically(); List<User> actual = alphabeticalGrouping.get('A'); User[] expected = { new User(2L, "Auroora", "Oats"), new User(4L, "Alika", "Capin"), new User(20L, "Artus", "Rickards"), new User(27L, "Antonina", "Vivian")}; assertThat(actual).containsExactly(expected); }
```

This way, we can hide the implementation of such methods, remove clutter from our services, and unload the repositories.

## 4\. Conclusion[](https://www.baeldung.com/spring-data-return-map-instead-of-list#conclusion)

Spring JPA allows customization, but sometimes it’s pretty straightforward to achieve this. Building an application around the types restricted by a framework might affect the quality of the code and even the design of an application.

Using custom collections as return types might make the design more straightforward and less cluttered with mapping and filtering logic. **Using dedicated wrappers for the collections of entities can improve the code even further.**

As usual, all the code used in this tutorial is available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/persistence-modules/spring-data-jpa-query-3).
