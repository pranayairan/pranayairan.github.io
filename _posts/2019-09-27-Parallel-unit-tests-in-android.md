---
layout: post
title: 'Parallel Unit Tests in Android'
description:  Running Unit Test and Robo Electric tests in parallel is hard in Android. We can simplify this by using Junit Categories and a custom gradle task. This article shows how we can shard the unit test on android using jUnit categories. 
author: pranay
date: '2019-09-26T20:30:35.278Z'
categories: [ Testing, Android ]
keywords: [Testing, Android, Roboelectric, Unit test, Test Sharding, Junit Categories]
image: assets/images/parallel.jpg
featured: true
hidden: true
---
As mobile apps are getting more complex so are the code written to test the functionality. Testing on android involves various layers and there are multiple tools to test specific layer but broadly we can categorize these tests into 3 buckets. 

 1. Unit Test (fast, runs on JVM)
 2. UI test (slow, runs using an emulator)
 3. RoboElectric test (moderately fast, runs on JVM with simulated android env)

JVM tests usually run super fast, but as project complexity grows, the test suit becomes bigger and slower. On Android, all unit test runs **Sequentally**. The larger the test suits the longer it will take for them to complete. 

With project [Nitrogen](https://medium.com/androiddevelopers/write-once-run-everywhere-tests-on-android-88adb2ba20c5) and Roboelectric 4, it has become easier to write RoboElectric tests to test the UI behavior. RoboElectric tests are faster than UI test and run on JVM, but since it depends on simulated android env, these tests are usually slower than vanilla Junit tests. 

As our android apps start growing in complexity, the test execution starts slowing impacting overall build performance and developer productivity.  There are ways in gradle to execute the unit test in parallel by setting 

   ``` groovy
tasks.withType(Test) {
        maxParallelForks = Runtime.runtime.availableProcessors()
    }
```

This options is rigid and can crash your build machine. 

To get the most advantage of your build machine and CI we need to **Shard** this tests into multiple parallel executions. For example, we can shard the tests into 2, one running just unit test and other shard running RoboElectric test in parallel reducing the overall test time. 

> **Sharding** is splitting the test suite into multiple threads, so they can be easily run as a group. 

Unfortunately Android and Gradle don't allow an easy way to shard or break your tests. In this post, we will discuss 2 different approaches we can use to achieve this. 

##  Command line options

This is the most simple option available with **gradle**, when it comes to executing 1 test, all tests in a class or a package of tests. 
Let's take an example test class 

```kotlin
package com.dexterapps.testsharding

class ConverterUtilTest {

    @Test
    fun testConvertFahrenheitToCelsius() {
        val actual = ConverterUtil.convertCelsiusToFahrenheit(100F)
        val expected = 212f
        
        assertEquals("Conversion from celsius to fahrenheit failed",
            expected.toDouble(), actual.toDouble(), 0.001)
    }
    @Test
    fun testConvertCelsiusToFahrenheit() {
     // test code here
    }
}
```
### Executing Single Test
**testConvertFahrenheitToCelsius**  can be executed with this command
```
 ./gradlew :ProjectName:testVariantNameUnitTest --tests com.dexterapps.testsharding.ConverterUtilTest.testConvertFahrenheitToCelsius

```
Relpace `:ProjectName:testVariantNameUnitTest` with yours like  `:app:testDebugUnitTest` or the variant you like to test.


### Executing all tests in class
To run all tests in a class like **ConverterUtilTest** we can run this command
```
./gradlew :ProjectName:testVariantNameUnitTest --tests com.dexterapps.testsharding.ConverterUtilTest
```

### Executing all tests in a package
To run all tests in a package like **com.dexterapps.testsharding** we can run this command
```
./gradlew :ProjectName:testVariantNameUnitTest --tests com.dexterapps.testsharding.*
```
Even though this option works for a lot of cases, the downside of this approach is each engineer would need to set up something on their machine whether that's a run configuration in Android Studio or a shell script, etc. 

This approach also doesn't help us to **Shard** the test cleanly.  Say if we want to just separate **unit test** and **roboelectric test**  we need to force everyone to put this test in specific package or something similar.  Let's look at another approach that might give us the flexibility we need. 

##  Junit Categories
JUnit 4.12 introduced a nifty feature called **Categories **. With Categories we can organize the test cases into different categories, label them and run those categorized test cases. 

Though this seems simple, there is no straight forward support for Junit Category in Gradle and Gradle Android. We need to write a custom gradle task to make it work. Let's look at the code. 

Sample Android app with Unit Test Sharding using Junit  `Categories` can be found here [https://github.com/pranayairan/android-unit-test-sharding](https://github.com/pranayairan/android-unit-test-sharding)

### Marker Interface 
To represent the categories we need to create marker interfaces. This is simple interfaces that we will use to differentiate between various test types. Here is how I defined interfaces for my app
```kotlin
interface RoboElectricTests

interface UnitTests

interface FastTests

interface SlowTests
```
This interface will represent different categories that we will use to classify our tests. 

### Category Example
Once we have a marker interface, it is trivial to add them as categories. We just need to annotate tests we want to categorize with `@category` annotation. Let's look at an example

```kotlin
 @Test
 @Category(UnitTests::class)
 fun testConvertFahrenheitToCelsius() {
     val actual = ConverterUtil.convertCelsiusToFahrenheit(100F)
     val expected = 212f
     
     assertEquals("Conversion from celsius to fahrenheit failed",
         expected.toDouble(), actual.toDouble(), 0.001)
```
We can even add multiple categories or categories at class level like this 
```kotlin
 @Category(FastTests::class)  
 class ConverterUtilTest {
    @Test  
    @Category(FastTests::class,UnitTests::class)  
    fun testConvertCelsiusToFahrenheit() {
    }
 }
```

### Running the category tests
Android gradle plugin doesn't support running the category tests easily. To get this annotation work we need to use a custom gradle task.  Let's look at some code to execute a test with category `roboelectric`
```groovy
if (project.gradle.startParameter?.taskRequests?.args[0]?.remove("--robolectric")) {
    subprojects
            .find { it.name == "app" }
            .tasks
            .withType(Test)
            .configureEach {
                useJUnit {
                    includeCategories 'com.dexterapps.testsharding.RoboElectricTests'
                    //excludeCategories if we want to exclude any test
                }
            }
}
```
We need to add this code to root `build.gradle` file to make it work for a module with the name _app_. If you have multiple modules, you need to change the find block. 

With this gradle code, we check if the command has `--robolectric` parameter. If it does, we look for the Junit task and add `includeCategories` for all test that has Category `com.dexterapps.testsharding.RoboElectricTests` all other tests would not be excluded. We can also use `excludeCategories `to do the reverse. 

Once this task is added, we can easily run Unit test with `@Category(RoboElectricTests::class)` using following command. 
```
./gradlew :app:testDebugUnitTest --robolectric
```
Similarly we can run other test like 
```
./gradlew :app:testDebugUnitTest --unit
or
/gradlew :app:testDebugUnitTest --fasttest
```

### Wrap up

Sharding is not a new concept and people are doing it in UI testing by using tools like [Spoon](https://square.github.io/spoon/) or [Flank](https://github.com/TestArmada/flank) for a long time. But with the increasing use of RoboElectric and other JVM based test, it is beneficial to shard Unit tests and run them in parallel. 

In this post, we saw how we can take advantage of **Junit Categories** to shard our unit test and run them using a custom gradle command. 

I created a sample app that shard unit tests into 3 different categories. You can find the complete source code at  [https://github.com/pranayairan/android-unit-test-sharding](https://github.com/pranayairan/android-unit-test-sharding)

_I would like to give shot out to my Colleague [Martin Feldsztejn](https://github.com/martofeld) who wrote the custom gradle command to get categories to work in android. I would also like to thanks Sriram Santosh for proofreading this and providing his feedback_

