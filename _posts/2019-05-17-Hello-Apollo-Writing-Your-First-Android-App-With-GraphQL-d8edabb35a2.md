---
layout: post
title: Hello Apollo Writing Your First Android App With GraphQL
author: pranay
description: >-
  Create your first GraphQL Android App with Apollo Android and explore how
  GrahQL can make your app fast.
date: '2019-05-17T17:39:35.278Z'
categories: [ GraphQL, Android, Tutorial ]
keywords: [GraphQL, Android, Tutorial, Apollo Android, Apollo GraphQL]
image: https://cdn-images-1.medium.com/max/2560/1*uKVfsFREG2HMCL8hJdcZ5Q.jpeg
featured: true
hidden: true
---

![](https://cdn-images-1.medium.com/max/2560/1*uKVfsFREG2HMCL8hJdcZ5Q.jpeg)

If you are a mobile developer, you understand the importance of faster API and faster apps. Especially in countries where data is costly, it becomes important to get only required data.

But API’s are designed for multiple clients each with different requirements and more often you will find that those API’s are not optimized for mobile apps. That is where newer technologies like GraphQL can help.

GraphQL is a query language for you API, that gives clients the power to ask for exactly what they need and nothing more. If you have not read about GraphQL explore more at [http://graphql.org/](http://graphql.org/).

### **GraphQL on Android**

Let’s explore how we can consume GraphQL API’s on Android. Although there are multiple client libraries for the web, so far there is only 1 GraphQL client for Android [Apollo](https://github.com/apollographql/apollo-android).

[Apollo Android](https://github.com/apollographql/apollo-android) is an awesome GraphQL client that makes consuming GraphQL API easy. It has 2 main components

*   Apollo Codegen, this component is a gradle plugin to generate code like ButterKnife, apollo codegen generates Java models from standard graphql queries at compile time.
*   Networking/Caching, the other component of Apollo android is the networking and caching piece, this takes care of all the network communication with your GraphQL API, parsing the response to correct model, enabling you to pass dynamic data to your GraphQL queries and response caching.

### Hello World

Now we know what is GraphQL and how Apollo Android works let’s see how we can integrate apollo in our android app.

Create an empty android project if you don’t already have one. Now in your project level **build.gradle** file add this line

classpath 'com.apollographql.apollo:gradle-plugin:0.3.2'

this should be placed after **com.android.tools.** Now open your app’s **build.gradle** and add this line on top

apply plugin: 'com.apollographql.android'

this should go below **com.android.application,** if you want to use apollo for your Kotlin project, add apollo plugin before you kotlin plugin. With this 2 dependencies, we added apollo in our app.

### **CodeGen**

Apollo takes your graphql queries, takes the schema and generate Java classes from it. Let’s explore how it works

*   Create a folder under **src/main** on the same level as your **java/res** folder. You can name the folder anything, but I name it graphql.
*   Add your **schema.json** file to this folder. Schema.json if the file that describes your GraphQL API, all fields, input arguments etc. Sample schema file [here](https://github.com/apollographql/apollo-android/blob/master/apollo-sample/src/main/graphql/com/apollographql/apollo/sample/schema.json)
*   Add your GraphQL query file with a **.graphql extension** in the same folder, this queries will be then used by apollo codegen to generate the data model for the response. Sample GraphQL files [here](https://github.com/apollographql/apollo-android/blob/master/apollo-sample/src/main/graphql/com/apollographql/apollo/sample/GithuntFeedQuery.graphql)

After adding your schema and .graphql files, rebuild the project. Apollo will parse the queries and schema and generate code for you. Once your build is complete, you can explore the generated files by going to **app/build/generated/source/apollo** folder.

### Wiring up

Now we have the apollo dependencies added and codegen working, let’s wire up everything and make our first GraphQL network call.

For this sample app, we will be consuming a sample GraphQL API of GitHub [https://githunt-api.herokuapp.com/graphql](https://githunt-api.herokuapp.com/graphql)

Apollo uses [OkHTTP](http://square.github.io/okhttp/) as its networking client, if you are already familiar with OkHttp you know we can use OkHTTP to add any **headers and interceptors.**

> Add all require headers for your api in okhttp client

OkHttpClient okHttpClient = new OkHttpClient.Builder()  
        .addInterceptor(logging)  
        .build();

If you want to enable caching, apollo comes with 3 level of cache, read more about caching here [https://github.com/apollographql/apollo-android](https://github.com/apollographql/apollo-android). If you decide to use caching you will need to add the following dependency.

compile "com.apollographql.apollo:apollo-android-support:0.3.2"

Now we have the OkHttp Client and Cache, we can construct the Apollo client object.

apolloClient = ApolloClient._builder_()  
        .serverUrl(_BASE\_URL_)  
        .okHttpClient(okHttpClient)  
        .normalizedCache(normalizedCacheFactory, cacheKeyResolver)  
        .build();

Cache part is **optional**, Once apollo client is build, you can use the same client for all your network request.

### First GraphQL Request

In our sample app apollo generated **FeedQuery class** from our sample .graphql file. We will use this class to make the network call, using our newly created apolloClient object.

**Build query passing all the parameters**

FeedQuery feedQuery = FeedQuery._builder_()  
        .limit(_FEED\_SIZE_)  
        .type(FeedType._HOT_)  
        .build();

**Limit and Type** are dynamic parameters in our graphQL queries, apollo automatically create setters for those parameters enabling us to pass the values from our Java code.

**Create ApolloCall.**

ApolloCall<FeedQuery.Data> githuntFeedCall = apolloClient  
                                             .query(feedQuery);

Apollo supports both normal **callback** and **RxJava.** Sample app will have 2 API 1 consumed via a normal callback and 1 via RxJava. For the article let’s see how the callback method works.

githuntFeedCall.enqueue(new ApolloCall.Callback<FeedQuery.Data>() {  
    @Override  
    public void onResponse(@Nonnull Response<FeedQuery.Data> response) {  
        FeedQuery.Data data = response.data();  
    }  
  
    @Override  
    public void onFailure(@Nonnull ApolloException e) {  
  
    }  
});

To use the normal callback method, all you need to do is call enqueue with a callback, similar style as retrofit.

That’s it, you have successfully consumed your first GraphQL API in your android app.

GraphQL is powerful and a great alternative to REST based API. If you are dealing with complex API and if it supports GraphQL you can simplify your app by switching to GraphQL.

You can find a complete working sample app here [https://github.com/apollographql/apollo-android/tree/master/apollo-sample](https://github.com/apollographql/apollo-android/tree/master/apollo-sample). But apollo sample app adds all project level dependencies and doesn’t work properly (PR in progress).

So I modified the sample app and removed all the project level dependencies, a fully working app with apollo and GraphQL can be found here [https://github.com/pranayairan/HelloApolloAndroid](https://github.com/pranayairan/HelloApolloAndroid)

_If you liked this article, click the💚 below so other people will see this here on Medium._