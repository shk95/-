---
created: 2024-01-09T09:34:25 (UTC +09:00)
tags: []
source: https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value
author: 
---

# Bind Case Insensitive @Value to Enum in Spring Boot | Baeldung

> ## Excerpt
> Learn how to leverage Spring autoconfiguration to map these values to Enum instances.

---
## 1\. Overview[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#overview)

Spring provides us with autoconfiguration features that we can use to bind components, configure beans, and set values from a property source.

[_@Value_](https://www.baeldung.com/spring-value-annotation) annotation is useful when we don’t want to cannot hardcode the values and prefer to provide them using property files or the system environment.

In this tutorial, we’ll learn how to leverage Spring autoconfiguration to map these values to [_Enum_](https://www.baeldung.com/a-guide-to-java-enums) instances.

## 2\. _Converters<F,T>_[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#convertersltftgt)

Spring uses converters to map the _String_ values from _@Value_ to the required type. A dedicated [_BeanPostPorcessor_](https://www.baeldung.com/spring-beanpostprocessor) goes through all the components and checks if they require additional configuration or, in our case, injection. **After that, a suitable converter is found, and the data from the source converter is sent to the specified target.** Spring provides a _String_ to _Enum_ converter out of the box, so let’s review it.

### 2.1. _LenientToEnumConverter_[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#1-lenienttoenumconverter)

As the name suggests, this converter is quite free to interpret the data during conversion. Initially, it assumes that the values are provided correctly:

```java
@Override public E convert(T source) { String value = source.toString().trim(); if (value.isEmpty()) { return null; } try { return (E) Enum.valueOf(this.enumType, value); } catch (Exception ex) { return findEnum(value); } }
```

However, it tries a different approach if it cannot map the source to an _Enum_. It gets the canonical names for both _Enum_ and the value:

```java
private E findEnum(String value) { String name = getCanonicalName(value); List<String> aliases = ALIASES.getOrDefault(name, Collections.emptyList()); for (E candidate : (Set<E>) EnumSet.allOf(this.enumType)) { String candidateName = getCanonicalName(candidate.name()); if (name.equals(candidateName) || aliases.contains(candidateName)) { return candidate; } } throw new IllegalArgumentException("No enum constant " + this.enumType.getCanonicalName() + "." + value); }
```

The _getCanonicalName(String)_ filters out all special characters and converts the string to lowercase:

```java
private String getCanonicalName(String name) { StringBuilder canonicalName = new StringBuilder(name.length()); name.chars() .filter(Character::isLetterOrDigit) .map(Character::toLowerCase) .forEach((c) -> canonicalName.append((char) c)); return canonicalName.toString(); }
```

**This process makes the converter quite adaptive, so it might introduce some problems if not considered.** At the same time, it provides excellent support for case-insensitive matching for _Enum_ for free, without any additional configuration required.

### 2.2. Lenient Conversion[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#2-lenient-conversion)

Let’s take a simple _Enum_ class as an example:

```java
public enum SimpleWeekDays { MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY }
```

We’ll inject all these constants into a dedicated class-holder using _@Value_ annotation:

```java
@Component public class WeekDaysHolder { @Value("${monday}") private WeekDays monday; @Value("${tuesday}") private WeekDays tuesday; @Value("${wednesday}") private WeekDays wednesday; @Value("${thursday}") private WeekDays thursday; @Value("${friday}") private WeekDays friday; @Value("${saturday}") private WeekDays saturday; @Value("${sunday}") private WeekDays sunday; // getters and setters }
```

Using lenient conversion, we can not only pass the values using a different case, but as was shown previously, we can add special characters around and inside these values, and the converter will still map them:

```java
@SpringBootTest(properties = { "monday=Mon-Day!", "tuesday=TuesDAY#", "wednesday=Wednes@day", "thursday=THURSday^", "friday=Fri:Day_%", "saturday=Satur_DAY*", "sunday=Sun+Day", }, classes = WeekDaysHolder.class) class LenientStringToEnumConverterUnitTest { @Autowired private WeekDaysHolder propertyHolder; @ParameterizedTest @ArgumentsSource(WeekDayHolderArgumentsProvider.class) void givenPropertiesWhenInjectEnumThenValueIsPresent( Function<WeekDaysHolder, WeekDays> methodReference, WeekDays expected) { WeekDays actual = methodReference.apply(propertyHolder); assertThat(actual).isEqualTo(expected); } }
```

It’s not necessarily a good thing to do, especially if it’s hidden from developers. **Incorrect assumptions can create subtle problems that are hard to identify.**

### 2.3. Extremely Lenient Conversion[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#3-extremely-lenient-conversion)

At the same time, this type of conversion works for both sides and won’t fail even if we break all the naming conventions and use something like this:

```java
public enum NonConventionalWeekDays { Mon$Day, Tues$DAY_, Wednes$day, THURS$day_, Fri$Day$_$, Satur$DAY_, Sun$Day }
```

The issue with this case is that it might yield the correct result and map all the values to their dedicated enums:

```java
@SpringBootTest(properties = { "monday=Mon-Day!", "tuesday=TuesDAY#", "wednesday=Wednes@day", "thursday=THURSday^", "friday=Fri:Day_%", "saturday=Satur_DAY*", "sunday=Sun+Day", }, classes = NonConventionalWeekDaysHolder.class) class NonConventionalStringToEnumLenientConverterUnitTest { @Autowired private NonConventionalWeekDaysHolder holder; @ParameterizedTest @ArgumentsSource(NonConventionalWeekDayHolderArgumentsProvider.class) void givenPropertiesWhenInjectEnumThenValueIsPresent( Function<NonConventionalWeekDaysHolder, NonConventionalWeekDays> methodReference, NonConventionalWeekDays expected) { NonConventionalWeekDays actual = methodReference.apply(holder); assertThat(actual).isEqualTo(expected); } }
```

**Mapping _“Mon-Day!”_ to _“Mon$Day”_ without failing might hide issues and suggest developers skip the established conventions.** Although it works with case-insensitive mapping, the assumptions are too frivolous.

## 3\. Custom Converters[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#custom-converters)

The best way to address specific rules during mappings is to create our implementation of a _Converter._ After witnessing what _LenientToEnumConverter_ is capable of, let’s take a few steps back and create something more restrictive.

### 3.1. _StrictNullableWeekDayConverter_[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#1-strictnullableweekdayconverter)

Imagine that we decided to map values to the enums only if the properties correctly identify their names. This might cause some initial problems with not respecting the uppercase convention, but overall, this is a bulletproof solution:

```java
public class StrictNullableWeekDayConverter implements Converter<String, WeekDays> { @Override public WeekDays convert(String source) { try { return WeekDays.valueOf(source.trim()); } catch (IllegalArgumentException e) { return null; } } }
```

This converter will make minor adjustments to the source string. Here, the only thing we do is trim whitespace around the values. **Also, note that returning null isn’t the best design decision, as it would allow the creation of a context in an incorrect state.** However, we’re using nulls here to simplify testing:

```java
@SpringBootTest(properties = { "monday=monday", "tuesday=tuesday", "wednesday=wednesday", "thursday=thursday", "friday=friday", "saturday=saturday", "sunday=sunday", }, classes = {WeekDaysHolder.class, WeekDayConverterConfiguration.class}) class StrictStringToEnumConverterNegativeUnitTest { public static class WeekDayConverterConfiguration { // configuration } @Autowired private WeekDaysHolder holder; @ParameterizedTest @ArgumentsSource(WeekDayHolderArgumentsProvider.class) void givenPropertiesWhenInjectEnumThenValueIsNull( Function<WeekDaysHolder, WeekDays> methodReference, WeekDays ignored) { WeekDays actual = methodReference.apply(holder); assertThat(actual).isNull(); } }
```

At the same time, if we provide the values in uppercase, the correct values would be injected. To use this converter, we need to tell Spring about it:

```java
public static class WeekDayConverterConfiguration { @Bean public ConversionService conversionService() { DefaultConversionService defaultConversionService = new DefaultConversionService(); defaultConversionService.addConverter(new StrictNullableWeekDayConverter()); return defaultConversionService; } }
```

In some Spring Boot versions or configurations, a similar converter may be a default one, which makes more sense than _LenientToEnumConverter._

### 3.2. _CaseInsensitiveWeekDayConverter_[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#2-caseinsensitiveweekdayconverter)

Let’s find a happy middle ground where we’ll be able to use case-insensitive matching but at the same time won’t allow any other differences:

```java
public class CaseInsensitiveWeekDayConverter implements Converter<String, WeekDays> { @Override public WeekDays convert(String source) { try { return WeekDays.valueOf(source.trim()); } catch (IllegalArgumentException exception) { return WeekDays.valueOf(source.trim().toUpperCase()); } } }
```

We’re not considering the situation when _Enum_ names aren’t in uppercase or using mixed case. **However, this would be a solvable situation and would require only an additional couple of lines and try-catch blocks.** We could create a lookup map for the _Enum_ and cache it, but let’s do it.

The tests would look similar and would correctly map the values. For simplicity, let’s check only the properties that would be correctly mapped using this converter:

```java
@SpringBootTest(properties = { "monday=monday", "tuesday=tuesday", "wednesday=wednesday", "thursday=THURSDAY", "friday=Friday", "saturday=saturDAY", "sunday=sUndAy", }, classes = {WeekDaysHolder.class, WeekDayConverterConfiguration.class}) class CaseInsensitiveStringToEnumConverterUnitTest { // ... }
```

Using custom converters, we can adjust the mapping process based on our needs or conventions we want to follow.

## 4\. SpEL[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#spel)

[SpEL](https://www.baeldung.com/spring-expression-language) is a powerful tool that can do almost anything. **In the context of our problem, we’ll try to adjust the values we receive from a property file before we try to map _Enum_.** To achieve this, we can explicitly change the provided values to upper-case:

```java
@Component public class SpELWeekDaysHolder { @Value("#{'${monday}'.toUpperCase()}") private WeekDays monday; @Value("#{'${tuesday}'.toUpperCase()}") private WeekDays tuesday; @Value("#{'${wednesday}'.toUpperCase()}") private WeekDays wednesday; @Value("#{'${thursday}'.toUpperCase()}") private WeekDays thursday; @Value("#{'${friday}'.toUpperCase()}") private WeekDays friday; @Value("#{'${saturday}'.toUpperCase()}") private WeekDays saturday; @Value("#{'${sunday}'.toUpperCase()}") private WeekDays sunday; // getters and setters }
```

To check that the values are mapped correctly, we can use the _StrictNullableWeekDayConverter_ we created before:

```java
@SpringBootTest(properties = { "monday=monday", "tuesday=tuesday", "wednesday=wednesday", "thursday=THURSDAY", "friday=Friday", "saturday=saturDAY", "sunday=sUndAy", }, classes = {SpELWeekDaysHolder.class, WeekDayConverterConfiguration.class}) class SpELCaseInsensitiveStringToEnumConverterUnitTest { public static class WeekDayConverterConfiguration { @Bean public ConversionService conversionService() { DefaultConversionService defaultConversionService = new DefaultConversionService(); defaultConversionService.addConverter(new StrictNullableWeekDayConverter()); return defaultConversionService; } } @Autowired private SpELWeekDaysHolder holder; @ParameterizedTest @ArgumentsSource(SpELWeekDayHolderArgumentsProvider.class) void givenPropertiesWhenInjectEnumThenValueIsNull( Function<SpELWeekDaysHolder, WeekDays> methodReference, WeekDays expected) { WeekDays actual = methodReference.apply(holder); assertThat(actual).isEqualTo(expected); } }
```

Although the converter understands only upper-case values, by using SpEL, we convert the properties to the correct format. **This technique might be helpful for simple translations and mappings, as it’s present directly in the _@Value_ annotation and is relatively straightforward to use.** However, avoid putting a lot of complex logic into SpEL.

## 5\. Conclusion[](https://www.baeldung.com/spring-boot-enum-bind-case-insensitive-value#conclusion)

_@Value_ annotation is powerful and flexible, supporting SpEL and property injection. Custom converters might make it even more powerful, allowing us to use it with custom types or implement specific conventions.

As usual, all the code in this tutorial is available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/spring-boot-modules/spring-boot-properties-4).
