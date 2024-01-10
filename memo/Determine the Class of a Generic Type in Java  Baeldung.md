---
created: 2024-01-09T09:52:03 (UTC +09:00)
tags: []
source: https://www.baeldung.com/java-generic-type-find-class-runtime
author: 
---

# Determine the Class of a Generic Type in Java | Baeldung

> ## Excerpt
> Learn about generics and type erasure, along with its benefits and limitations, and explore various workarounds for getting a class of generic type information at runtime.

---
## 1\. Overview[](https://www.baeldung.com/java-generic-type-find-class-runtime#overview)

[Generics](https://www.baeldung.com/java-generics), which was released in Java 5, allowed developers to create classes, interfaces, and methods with typed parameters, enabling the writing of type-safe code. Extracting this type of information at runtime allows developers to write more flexible code.

In this tutorial, we’ll learn how to get the class of a generic type.

## 2\. Generics and Type Erasure[](https://www.baeldung.com/java-generic-type-find-class-runtime#generics-and-type-erasure)

**Generics were introduced in Java with the major goal of providing compile-time type-safety checks along with flexibility and reusability of code.** The introduction of Generics made the [Collections](https://www.baeldung.com/java-collections) framework undergo significant enhancements and improvements. Before generics, Java collections used raw type, which was kind of error-prone, and developers often faced typecast exceptions.

To demonstrate this, let’s consider a simple example where we create a [_List_](https://www.baeldung.com/java-arraylist) and add data to it:

```java
void withoutGenerics(){ List container = new ArrayList(); container.add(1); container.add("2"); container.add("string"); for (int i = 0; i < container.size(); i++) { int val = (int) container.get(i); //For "string", we get java.lang.ClassCastException: class String cannot be cast to class Integer } }
```

In the above example, the _List_ contains raw data. Hence, we’re able to add _Integer_s and _String_s. When we read the list using _get(),_ we’re type-casting it to _Integer_, but for the _String_ type, we get a type-casting exception.

With generics, we define a type parameter for a collection. If we try to add any other data type than the defined type parameter, then the compiler complains about it.

For example, let’s create a generic _List_ with an _Integer_ type and try adding different types of data to it:

```java
void withGenerics(){ List<Integer> container = new ArrayList(); container.add(1); container.add("2"); // compiler won't allow this since we cannot add string to list of integer container. container.add("string"); // compiler won't allow this since we cannot add string to list of integer container. for (int i = 0; i < container.size(); i++) { int val = container.get(i); // not casting required since we defined type for List container. } }
```

In the above code, when we’re trying to add a _String_ data type to the _List_ of integers, the compiler complains about it.

In Generics, the type of information is only available at compile-time. Java compiler erases type information during compilation and it’s not available at runtime. This is called [type erasure](https://www.baeldung.com/java-type-erasure). 

**Due to type erasure, all the type parameter information is replaced with the bound (if the upper bound is defined) or the object type (if the upper bound isn’t defined)**.

We can confirm this by using the _[javap](https://www.baeldung.com/java-class-view-bytecode)_ utility, which inspects the _.class_ file and helps to examine the bytecode. Let’s compile the code containing the _withGenerics()_ method above and inspect it with the _javap_ utility:

```java
javac CollectionWithAndWithoutGenerics.java // compiling java file
```

```java
javap -v CollectionWithAndWithoutGenerics // read bytecode using javap tool
```

```plaintext
// bytecode mnemonics public static void withGenerics(); descriptor: ()V flags: (0x0009) ACC_PUBLIC, ACC_STATIC Code: stack=2, locals=3, args_size=0 0: new #12 // class java/util/ArrayList 3: dup 4: invokespecial #14 // Method java/util/ArrayList."<init>":()V 7: astore_0 8: aload_0 9: iconst_1 10: invokestatic #15 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer; 13: invokeinterface #21, 2 // InterfaceMethod java/util/List.add:(Ljava/lang/Object;
```

As we can see in _bytecode_ mnemonics, line #13,  _List.add_ method was passed with _Object_ instead of _Integer_ type.

Type erasure was a design choice made by Java designers to support backward compatibility.

## 3\. Getting Class Information[](https://www.baeldung.com/java-generic-type-find-class-runtime#getting-class-information)

**The unavailability of type information at runtime makes it challenging to capture type information at runtime.** However, there are certain workarounds to get the type information at runtime.

### 3.1. **Using _Class<T>_ Parameter**[](https://www.baeldung.com/java-generic-type-find-class-runtime#1-using-classlttgt-parameter)

In this approach, we explicitly pass the class of generic type _T_ at runtime, and this information is retained so that we can access it at runtime. In the below example, we’re passing _Class<T>_ at runtime to the constructor, which assigns it to the _clazz_ variable. Then, we can access class information using the _getClazz()_ method:

```java
public class ContainerTypeFromTypeParameter<T> { private Class<T> clazz; public ContainerTypeFromTypeParameter(Class<T> clazz) { this.clazz = clazz; } public Class<T> getClazz() { return this.clazz; } }
```

Our test verifies that we’re successfully storing and retrieving the class information at runtime:

```java
@Test public void givenContainerClassWithGenericType_whenTypeParameterUsed_thenReturnsClassType(){ var stringContainer = new ContainerTypeFromTypeParameter<>(String.class); Class<String> containerClass = stringContainer.getClazz(); assertEquals(String.class, containerClass); }
```

### 3.2. **Using Reflection**[](https://www.baeldung.com/java-generic-type-find-class-runtime#2-using-reflection)

Using a non-generic field with [reflection](https://www.baeldung.com/java-reflection) is another workaround that allows us to get generic information at runtime.

Basically, we use reflection to obtain the runtime class of a generic type. In the below example, we use _content.getClass(),_ which gets class information of content at runtime using reflection:

```java
public class ContainerTypeFromReflection<T> { private T content; public ContainerTypeFromReflection(T content) { this.content = content; } public Class<?> getClazz() { return this.content.getClass(); } }
```

Our test verifies that it works for the _ContainerTypeFromReflection_ class and gets the type information:

```java
@Test public void givenContainerClassWithGenericType_whenReflectionUsed_thenReturnsClassType() { var stringContainer = new ContainerTypeFromReflection<>("Hello Java"); Class<?> stringClazz = stringContainer.getClazz(); assertEquals(String.class, stringClazz); var integerContainer = new ContainerTypeFromReflection<>(1); Class<?> integerClazz = integerContainer.getClazz(); assertEquals(Integer.class, integerClazz); }
```

### 3.3. Using _TypeToken_[](https://www.baeldung.com/java-generic-type-find-class-runtime#3-using-typetoken)

**Type tokens are a popular way to capture generic type information at runtime. It was made popular by Joshua Bloch in his book “Effective Java”**.

In this approach, we first create an abstract class called _TypeToken,_ where we pass the type information from the client code. Inside the abstract class, we then use the _getGenericSuperClass()_ method to retrieve the passed type argument at runtime:

```java
public abstract class TypeToken<T> { private Type type; protected TypeToken(){ Type superClass = getClass().getGenericSuperclass(); this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0]; } public Type getType() { return type; } }
```

As we can see in the above example, Inside our TokenType abstract class, we are capturing _Type_ info at runtime using _getGenericSupperClass(),_ which we are returning using the _getType()_ method.

Our test verifies that it works for the sample class that extends the abstract _TypeToken_ with _String_ as the type parameter:

```java
@Test public void giveContainerClassWithGenericType_whenTypeTokenUsed_thenReturnsClassType(){ class ContainerTypeFromTypeToken extends TypeToken<List<String>> {} var container = new ContainerTypeFromTypeToken(); ParameterizedType type = (ParameterizedType) container.getType(); Type actualTypeArgument = type.getActualTypeArguments()[0]; assertEquals(String.class, actualTypeArgument); }
```

## **4\. Conclusion**[](https://www.baeldung.com/java-generic-type-find-class-runtime#conclusion)

In this article, we discuss generics and type erasure, along with its benefits and limitations. We also explored various workarounds for getting a class of generic type information at runtime, along with code examples.

As always, the example code is available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-lang-oop-generics).
