---
layout: post
title: 'YelpQL: Learn GraphQL By Building Yelp App'
description: One of the best way to learn new technology is to build something. This technique is super effective and I have learned a lot by building…
author: pranay
date: '2019-05-18T17:39:35.278Z'
categories: [ GraphQL, Android, Tutorial ]
keywords: [GraphQL, Android, Tutorial, Apollo Android, Apollo GraphQL, Yelp API, Yelp GraphQL]
image: https://cdn-images-1.medium.com/max/1200/1*Av9FavOvzFcxp8e1JW54HQ.png
featured: true
hidden: true
---

Design inspiration from: [https://medium.com/hh-design/an-unsolicited-app-redesign-yelp-396c41947776](https://medium.com/hh-design/an-unsolicited-app-redesign-yelp-396c41947776)

One of the best way to learn new technology is to build something. This technique is super effective and I have learned a lot by building apps and releasing it to users.

In my [last post](https://android.jlelse.eu/hello-apollo-writing-your-first-android-app-with-graphql-d8edabb35a2), we saw how GraphQL can enable faster API and in turn faster apps, and how we can use [Apollo Android](https://github.com/apollographql/apollo-android) to consume GraphQL API in an Android apps. If you haven’t read my last post please take some time and read it [here](https://android.jlelse.eu/hello-apollo-writing-your-first-android-app-with-graphql-d8edabb35a2).

I wanted to explore more and move beyond Hello World with an actual app, that uses an actual GraphQL API not just hello world.

I began my search on some cool GraphQL API’s and found this great list [https://github.com/APIs-guru/graphql-apis](https://github.com/APIs-guru/graphql-apis). There are some cool API’s supporting GraphQL but the most interesting API in this list was from [Yelp](https://www.yelp.com/developers/graphql/guides/intro).

> Yelp’s [GraphQL API](https://www.yelp.com/developers/graphql/guides/intro)’s are currently in Beta, but they are sufficient for creating an awesome Android App that talks GraphQL.

In this post, we will see how we can consume Yelp api, challenges and cool stuff that we can do with GraphQL.

### App Design

![](https://cdn-images-1.medium.com/max/800/1*TsUXENuu-uzrREL7Xx3IPw.png)

For the app design, I decided to pick [Kenny Lopez](https://medium.com/u/5506821b37ce) [Yelp’s Redesign](https://medium.com/hh-design/an-unsolicited-app-redesign-yelp-396c41947776) and mashed it together with Matt Dellac [redesign](https://www.uplabs.com/posts/yelp-redesign).

### API

Yelp’s [GraphQL API](https://www.yelp.com/developers/graphql/guides/intro)’s are similar to their fusion V2 API, but currently it doesn’t support all the functionality. For our [YelpQL](https://github.com/pranayairan/YelpQL) app we will use

*   [Search API](https://www.yelp.com/developers/graphql/query/search), to search for near by business.
*   [Business API](https://www.yelp.com/developers/graphql/query/business) to get details of business, reviews, ratings etc.

Yelp’s GraphQL api is currently in beta. To get started, sign up for Yelp Developer account [here](https://www.yelp.com/developers). Once you signed up as a developer you can join the beta program by going [here](https://www.yelp.com/developers/v3/manage_app.).

### Authentication

To query Yelp API’s you will need authentication token. This token can be reused and valid for 180 days. To obtain the authentication token

*   Make a POST call to [https://api.yelp.com/oauth2/token](https://api.yelp.com/oauth2/token) with your client\_id and client\_secret. You can obtain both from your yelp developer account.
*   This call will return a response with access\_token and expires\_in time until the token is valid. As of now, expiration is 180 days.

Check [YelpService.java](https://github.com/pranayairan/YelpQL/blob/master/app/src/main/java/binarybricks/com/yelpql/network/YelpService.java) file for the authentication calls we make in YelpQL.

### Location and Permission

Yelp API’s will not work without passing a location. This location can be an Address or Latitude, Longitude. Let’s get the current location of the user and also request for location permission.

Android Permission model comes with its own boilerplate code, to make things simple and reactive, we will be using [**RxPermission**](https://github.com/tbruyelle/RxPermissions)  library. Add [**RxPermission**](https://github.com/tbruyelle/RxPermissions) library in your code:

compile 'com.tbruyelle.rxpermissions2:rxpermissions:0.9.4@aar'

Now to request a permission all you need to do is make this call :

Once you get the location permission, we will need to request for users location, for our work we will need only users last known location.

We will be using [**RxLocation**](https://github.com/patloew/RxLocation)  library. This library wraps all the GoogleAPI client and Location boilerplate code into nice RXJava2 observables.

compile 'com.patloew.rxlocation:rxlocation:1.0.3'

To get users last known location just add this

With very few lines of code, we are able to request for location permission and users last known location. Let’s query the yelp api now.

### Search

Yelp’s [search api](https://www.yelp.com/developers/graphql/query/search) returns 16 fields for each business, in REST world you don’t have a choice but to get all this data on the wire, even if you care about only a few fields. But GraphQL allows us to pick and choose our response fields.

For our app, we only care about 7 fields, let’s construct the GraphQL query for our business search API.

If you notice in the above query, we are only interested in few fields, Yelp API with the power of GraphQL will return us only those fields, in turn, making the response smaller and app faster.

You can run above query in Yelp [GraphiQL](https://www.yelp.com/developers/graphiql?query=query%20SearchYelp%7B%0A%20%20search%28term%3A%22Burgers%22%2Cradius%3A%2010000%2C%20latitude%3A%2037.4220%2Clongitude%3A-122.0840%2C%20%0A%20%20%20%20limit%3A%2020%2C%20offset%3A%2020%29%20%7B%0A%20%20%20%20total%0A%20%20%20%20business%20%7B%0A%20%20%20%20%20%20id%0A%20%20%20%20%20%20name%0A%20%20%20%20%20%20rating%0A%20%20%20%20%20%20photos%0A%20%20%20%20%20%20price%0A%20%20%20%20%20%20coordinates%20%7B%0A%20%20%20%20%20%20%20%20latitude%0A%20%20%20%20%20%20%20%20longitude%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20categories%20%7B%0A%20%20%20%20%20%20%20%20title%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D%0A&operationName=SearchYelp) tool.

Since we are using Apollo for consuming GraphQL Queries, when you add this query and **rebuild** your project, Apollo will generate the **concrete class** for query. This concrete class will be named after your query name, in our case that would be SearchYelp.java.

#### Network call

Now we have the query and Apollo generated the concrete class, let’s make the actual network call that will get us the results.

> You can pass dynamic parameters in GraphQL queries by using **$** sign and the variable name and type, all parameters should be defined at the time of query creation. An **!** sign makes the parameter mandatory and will cause a run time exception if not provided.

Apollo will convert all the parameters into fields that can be set at the time of constructing the queries. Let’s construct our Search query:

```java
final SearchYelp searchYelp = SearchYelp._builder_()  
        .term(searchTerm)  
        .latitude(latitude)  
        .longitude(longitude)  
        .radius(10000d)  
        .offset(offset)  
        .build();
```

To construct a query, we will create the object of our query class that apollo generated using the builder.

All parameters that are defined in the **GraphQL query** then can be set making the query dynamic. Once we have the query object generated we can use apollo to make the network call.

To make network call using Apollo, we will create an **apollo client** object defining the URL and setting all the required headers. **Apollo uses OKHTTP** as it’s HTTP layer, so all headers will be defined using okhttp client object. Check [this file](https://github.com/pranayairan/YelpQL/blob/master/app/src/main/java/binarybricks/com/yelpql/network/GraphQLClient.java) for the complete code.

```java
ApolloQueryCall<SearchYelp.Data> query = apolloClient.query(searchYelp);
```

Once we have the Apollo client, we can call **.query** passing our query to get ApolloQueryCall object. This object will be then used for making the final request.

Apollo supports both **RXJava2 calls** and **straight call backs.** If you want to use straight callbacks you can just call **enqueue** on ApolloQueryCall object similar to Retrofit. But if you want RXJava calls you need to add this dependency and **wrap your query into Rx2Apollo.from** the  method. This returns a **RxJava2 Single**.

```ruby
compile 'com.apollographql.apollo:apollo-rx2-support:0.3.2'
```

Check [SearchAPI.java](https://github.com/pranayairan/YelpQL/blob/master/app/src/main/java/binarybricks/com/yelpql/network/SearchAPI.java) for complete network call for our search query. Once we have the response, we will add it to our adapter and display the list. We also added support for infinite scrolling since API restricts the number of businesses in the response.

### Business Details

Now we have the list of business, we would like to provide the functionality to get business details when some one clicks on any business. Let’s get that query finalize.

Again similar to our search query, we have the flexibility to pick and choose fields that you need, we can **further optimize** this query by not getting fields that we already got from Search query.

Rest of the process is similar, once you add this query, apollo will generate a concrete class and you will able to query the api in the same way you did for Search Query. You can look at [BusinessDetailsAPI.java](https://github.com/pranayairan/YelpQL/blob/master/app/src/main/java/binarybricks/com/yelpql/network/BusinessDetailsAPI.java) for complete code.

### Apollo Gotchas

Now we have an android app that consumes Yelp’s GraphQL API, let’s talk about some of the **gotchas** that you should be aware of when using Apollo with your app.

#### Schema

If you noticed above, after adding both the queries we recompiled the project, this was done due to fact that **Apollo converts your GraphQL queries into concrete classes.** What that means is Apollo needs to know everything about your API. This includes all the fields supported, their data types, nullability etc.

To enable Apollo to do this, you will need to provide **schema.json** file to Apollo. This files will have your GraphQL Queries schema. Apollo will use this schema to generate concrete classes for your queries.

> Yelp developer website doesn’t provide you direct access to schema. You will need to make a special query to get it.

You can download **Schema** of any GrahpQL API endpoint using [apollo-codegen](http://dev.apollodata.com/ios/downloading-schema.html). You can get complete instructions on how to use it [here](http://dev.apollodata.com/ios/downloading-schema.html)

Since YelpAPI needs authentication, you will need to pass that in headers for downloading the schema. You can get the schema that we are using by referring to [schema.json](https://github.com/pranayairan/YelpQL/blob/master/app/src/main/graphql/yelp/schema.json) file.

#### Models

If you open [SearchAPI.java](https://github.com/pranayairan/YelpQL/blob/master/app/src/main/java/binarybricks/com/yelpql/network/SearchAPI.java) you will notice that once we get the response back from Apollo we map it to a new model. We are doing similar things in [BusinessDetailsAPI.java](https://github.com/pranayairan/YelpQL/blob/master/app/src/main/java/binarybricks/com/yelpql/network/BusinessDetailsAPI.java) api as well.

To understand why we are doing this, we will need to understand how apollo generates the concrete class. For apollo every query is **unique** and it generates **inner classes** to represents response models.

So Apollo will represent the **business** in Search api as **SearchYelp.Business** and **business** in Business Details api as **BusinessDetails.Business.** Take a moment and read the above 2 lines again.

What this means is even though both search and business details query asking for **Business** and even though **Business as an entity** is referring to same underlying business on YELP. **Apollo treats both business response as different.**

Due to this limitation, it is always a good idea to have an underlying **conversion** that converts apollo **response model** to our apps UI model. This mapping also gives us another advantage, **since graphql response can be deeply nested,** this mapping helps us to **flattened** the response into more logical representation.

### Wrap up

In This post, we saw how we can make an actual android app that consumes GraphQL API using [**Apollo**](https://github.com/apollographql/apollo-android)**.** We also saw how we can use selective fields querying in GraphQL and how it can help us make apps fast.

We saw some of the gotchas that I learned while developing this app and how it can save you some time with Apollo.

You can find the complete source code of a working Yelp App on my repo [https://github.com/pranayairan/YelpQL](https://github.com/pranayairan/YelpQL)

_Cross Posted From Medium [https://android.jlelse.eu/yelpql-learn-graphql-by-building-yelp-app-da2a71f16c77_](https://android.jlelse.eu/yelpql-learn-graphql-by-building-yelp-app-da2a71f16c77_)