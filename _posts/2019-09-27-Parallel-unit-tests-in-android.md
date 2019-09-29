---
layout: post
title: 'Parallel Unit Tests in Android'
description:  Running Unit Test and Robo Electric tests in parallel is hard in Android. We can simplify it by using JUnit Categories and custom gradle commands. This article shows how we can shard the unit tests and save time by running them in parallel.
author: pranay
date: '2019-09-29T11:30:35.278Z'
categories: [ Testing, Android ]
keywords: [Testing, Android, Robolectric, Unit test, Test Sharding, JUnit Categories]
image: assets/images/parallel.jpg
featured: true
hidden: true
---

> ***TL;DR** Our unit tests were taking a lot of time to run and there was no way to shard them and run in parallel. We found a way to use **JUnit Categories** and custom gradle command to shard our tests and reduce test execution time by half.*

Unit tests are the fundamental tests in our app testing strategy. But the more tests we write, the more time it takes to execute them and for a code to merge. 

Android apps have various layers and we can test each layer using different tools. But we can categorize these tests into 3 buckets. 

 1. Unit Test (fast, runs on JVM)
 2. UI test (slow, runs using an emulator)
 3. Robolectric test (moderate, runs on JVM with simulated android env)

JVM tests usually run super fast but on android, all unit test runs **Sequentially**, the more tests we write, the longer it will take to execute them. 

With project [Nitrogen](https://medium.com/androiddevelopers/write-once-run-everywhere-tests-on-android-88adb2ba20c5), Robolectric tests are now supported in the Android X testing library. Robolectric tests run on JVM and are faster then espresso tests. But since they depend on simulated android env they are slower than vanilla JUnit tests.

Combining Robolectric and JUnit test makes test execution slow and can impact build time and developer productivity. Gradle has support for executing tests in parallel using a custom setting like this: 

   ``` groovy
tasks.withType(Test) {
        maxParallelForks = Runtime.runtime.availableProcessors()
    }
```

But this option is rigid and can cause a crash in our build machine.

Modern CI systems come with support for parallel builds. To get most out of our CI, we need a way to **Shard** our tests and execute them in parallel. For ex: we can divide our unit tests into 2 parts (**Robolectric and pure JVM**) and run them in parallel cutting test execution time in half. 


> **Sharding** is splitting the test suite into multiple threads, so they run as a group. 

Unfortunately Android and Gradle don't have an easy way to shard or break tests. In this post, we will discuss 2 different approaches we can use to achieve this. 

##  Command line
Command-line parameters are the most simple option available with **gradle**, when it comes to executing 1 test, all tests in a class or a package. 

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
We can execute single test like **testConvertFahrenheitToCelsius**  with this command
```shell
 ./gradlew :ProjectName:testVariantNameUnitTest --tests com.dexterapps.testsharding.ConverterUtilTest.testConvertFahrenheitToCelsius
```
Relpace `:ProjectName:testVariantNameUnitTest` with yours. Like  `:app:testDebugUnitTest` or the variant you will like to test. 


### Executing all tests in class
To run all tests in a class **ConverterUtilTest** we can run
```shell
./gradlew :ProjectName:testVariantNameUnitTest --tests com.dexterapps.testsharding.ConverterUtilTest
```

### Executing all tests in a package
To run all tests in a package **com.dexterapps.testsharding** we can
```shell
./gradlew :ProjectName:testVariantNameUnitTest --tests com.dexterapps.testsharding.*
```
Command line options work for a lot of cases but as a downside, each engineer would need to set up there own Android Studio run config or shell script. 

This approach doesn't **Shard** the tests. Let's say if we want to separate **Robolectric** and **JVM** tests, we need to use specific packages, forcing everyone to put tests in diff package from where they belong canonically. 

Can we do better? Yes, we can, Let's look at another approach that gives us the flexibility we need. 

##  JUnit Categories
JUnit 4.12 introduced a nifty feature **Categories**. Categories provide us with a mechanism to label and group tests and run these tests either by including or excluding the categories. 

JUnit categories are simple but there is no direct support for it in Gradle or Gradle Android. We need to write a custom Gradle code to make it work. Let's look at the code. 

> You can download all the code discussed in this post from
> [https://github.com/pranayairan/android-unit-test-sharding](https://github.com/pranayairan/android-unit-test-sharding)

### Marker Interface 
To represent the categories or label tests, we need to create marker interfaces. This simple interfaces will be will use to classify our tests and run them in parallel. 

```kotlin
interface RobolectricTests

interface UnitTests

interface FastTests

interface SlowTests
``` 
### Category Example
Once we have marker interfaces, it is trivial to add them as categories. To categorize a test annotate it with `@category` annotation and add interface name. Let's look at some code. 

```kotlin
 @Test
 @Category(UnitTests::class)
 fun testConvertFahrenheitToCelsius() {
     val actual = ConverterUtil.convertCelsiusToFahrenheit(100F)
     val expected = 212f
     
     assertEquals("Conversion from celsius to fahrenheit failed",
         expected.toDouble(), actual.toDouble(), 0.001)
 }
```
We can add many categories to single method or add categories at class level. 

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
There is no easy way to run category tests on Android. We can add support for Categories in Android gradle plugin by writing custom gradle code.

Let's look at the code to execute tests with category `robolectric`

```groovy
if (project.gradle.startParameter?.taskRequests?.args[0]?.remove("--robolectric")) {
    subprojects
            .find { it.name == "app" }
            .tasks
            .withType(Test)
            .configureEach {
                useJUnit {
                    includeCategories 'com.dexterapps.testsharding.RobolectricTests'
                    //excludeCategories if we want to exclude any test
                }
            }
}
```
We need to add this code to root `build.gradle` file to make it work for `app` module. For a multi-module project, we need to add support of all modules one by one with module names. 

This gradle code checks if any `Test` task is executed with `--robolectric` parameter. If we find this parameter we will *include* all tests with Category  `com.dexterapps.testsharding.RobolectricTests` for JUnit tasks and ignore others test. We can also use `excludeCategories` to do the reverse. 

To run Unit tests with `@Category(RobolectricTests::class)` using following command. 
```shell
./gradlew :app:testDebugUnitTest --robolectric
```
Similarly we can run other *category tests* with
```shell
./gradlew :app:testDebugUnitTest --unit
or
./gradlew :app:testDebugUnitTest --fasttest
```

### Results
*JUnit Categories* enabled us to divide and run multiple test jobs in parallel. Before `Categories` our test execution time was **5 min**. With `Categories` and parallel jobs, it takes only **3 min** (Unit Test 2 min, Robolectric 3 min) giving us **~40%** savings in test execution time. 

### Wrap up
Concept of test **sharding** is not new in android. UI testing frameworks like [Spoon](https://square.github.io/spoon/) or [Flank](https://github.com/TestArmada/flank) supported it for a long time. But sharding for a unit testing is non-existing. 

This post covered the use of  **JUnit Categories** and custom gradle code to break unit tests into different categories and run them in parallel. We achieved **~40%** improvement in test execution time and improved developer productivity. 

I created a sample android app that shard unit tests into 3 different categories. You can find the complete source code at **[https://github.com/pranayairan/android-unit-test-sharding](https://github.com/pranayairan/android-unit-test-sharding)**

_I would like to give shot out to my Colleague [Martin Feldsztejn](https://github.com/martofeld) who wrote the custom gradle command to get categories to work in android. I would also like to thanks [Sriram Santosh](https://github.com/sriramsantosh) for proofreading this and providing his feedback_
