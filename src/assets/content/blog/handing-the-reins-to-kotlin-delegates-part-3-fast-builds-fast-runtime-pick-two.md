---
title: 'Handing the Reins to Kotlin Delegates — Part 3: Fast builds, fast runtime.
  Pick two'
author: Jamie Sanson
date: 2019-10-01T10:00:00+13:00
hero_image: "/src/assets/content/images/kotlin-delegates-3-hero.jpeg"

---
In the previous articles we’ve seen what property delegation is, how we can use it to separate concerns, and implement complex getting and setting functionality. This article takes a different approach, looking at a real world problem and evaluating a bunch of different solutions. In this article we’ll see how Property Delegation, in some cases, can be comparable to Dynamic Proxies and Annotation Processing + Code Generation.

# Configuring Android Apps

One common component required for large Android applications is some form of application configuration. This allows developers and product owners to roll out or tweak features remotely, and measure user impact. One choice for configuration framework is Firebase Remote Config. It integrates nicely with Firebase A/B testing, giving those working on the app confidence that the features they’re deploying are performing well. It’s relatively easy to set up, but gets complicated as you scale to larger teams with more refined processes.

## Background: The problem

At Trade Me we use Firebase Remote Config for both feature toggling and A/B testing. For the most part it was working well, however we realised as our testing practice matured that we had no good way of testing these remote properties. The method our testers employed at the time was to modify values in the production Remote Config project and force a refresh in the app to test that the changes work as expected. We also found that as developers we weren’t providing default values for every config item, leading to inconsistencies in value access. We boiled this down to the fact that we were defining remote config properties in one file and defaults in another (`remote_config_default.xml`), where the second file could easily be overlooked.

We wanted to change these practices. The goal was to make it such that all Remote Config values could be overridden locally, such that testers could check these feature toggles within the app. This means we could lock down write access to the production remote config project to only those who absolutely need it. Additionally we wanted to make it easier for developers to define and use these config items, without having to go to multiple files. The requirements for the solution were therefore:

* Must expose config items in UI for testing
* Must be simple to define new items (no more than a few LoC)
* Default values must be defined in the same place

# Config Abstraction — The Contenders

## Dynamic Proxies

Dynamic Proxies are a concept from Java where you can dynamically implement interfaces. You specify a listener known as an `InvocationHandler` which handles method invocations. You can then do whatever you want within that handler, such as retrieve config items from different sources!

    Proxy.newProxyInstance(classLoader, arrayOf(Config::class.java), handler)

There are libraries out there which use this construct under the hood to provide a good developer experience, one of the more well known ones being [Retrofit](https://github.com/square/retrofit). This library allows us to specify interfaces with method calls, annotated with information around how that method should act. The library then inspects those annotations at runtime using reflection, and creates a proxy instance which you can then use — nice!

## Annotation Processing + Code Gen

Annotation processing is a pre-build step in the Java build pipeline. We can define annotation processors which are called in rounds by the toolchain, which allows us to inspect annotation on classes, methods and properties before compilation happens. As it happens before compiling, we can pair annotation processing with code generation to magically provide functionality with less strain on the developer!

One well known Android example of this method is [Butter Knife](https://github.com/JakeWharton/butterknife). Although now obsolete, this library solved the problem of accessing views in Android well before first party alternatives were available. Using Butter Knife would look something like this:

    @BindView(R.id.some_view)
    public View something;

Butter Knifes annotation processor would run before compilation, see these definitions and generate code to bind the view with ID `some_view` to the `something` field — easy!

## Kotlin Property Delegation

We’ve already seen a lot about Kotlin Property Delegation, so I won’t cover much here. One relatively well known example of Kotlin Property Delegation in a library is [Koin](https://github.com/InsertKoinIO/koin). This library aims to help developers with dependency management. It uses a function called `inject()` for property delegation, where it will internally look for dependencies which match the type of the property, and optionally scoped based of both the context the inject function is called from, and additional information which can be provided in the functions arguments. Seems to do the job!

# Weighing up the options

We investigated all three options, and turns out they all did everything we needed. We could define everything in one place! In the case of proxies and annotation processing we defined default values, keys etc. in annotation. In the case of property delegation we defined these via the a delegation functions arguments.

The APIs we prototyped for all three approaches gave us relatively similar developer experiences, so we had to look at more fundamental differences. We looked at performance first, which ended up being a good idea. There’s some commonly known drawbacks to annotation processing and proxy classes:

* Annotation Processing and code generation adds time to builds. This can be mitigated somewhat by developing the processor as incremental, but regardless would increase build times for clean builds — something we wanted to avoid.
* Using dynamic proxies isn’t expensive at runtime, but inspecting annotation _is._ Reflection has been known to be horribly slow on Android, and it’s something we weren’t that interested in depending on.

After assessing these drawbacks, it became clear that using Kotlin Properties for what we wanted to do would give the best performance at both build time and run time, so that’s where we landed!

# Konfigure — The Solution

We ran with the prototype API we were investigating when comparing our options, and fleshed out a library comprised of multiple modules. [Konfigure](https://github.com/TradeMe/konfigure) is the configuration library we use in our apps now, which has managed to solve all the problems we identified. Let’s see how it works:

## Ease of use and default values

We strived towards coming up with the most simple API we could, and we ended up with something where we could define everything we needed in one line. A simple, boolean config item is defined like this:

    val isSomethingEnabled: Boolean by config()

This looks for a `isSomethingEnabled` key which maps to a boolean value in Firebase Remote Config, and exposes it when you use the property. If it doesn’t exist in the remote, the `defaultValue` (which is false in this case) is returned. Easy!

## Overriding in UI

These config items are scoped to live within a parent `Config` class. When this function is called, these items are registered within the class, and can be retrieved by other components. This allowed us to build a UI on top of it, where display metadata can be tacked on to config items, and those items can be overridden! These overrides persist in memory by default, but can be extended to instead persist these overrides in more persistent memory, like shared preferences.

![](/src/assets/content/images/kotlin-delegates-3-ui.png)  
We bundled this UI in a separate dependency, so it’s optional to use it. Using it is as simple as implementing `ConfigProvider` in your `Application` class, then starting the `ConfigActivity` using a static `start` function.

It’s extensible too, if you want to add further functionality! The sample in the Konfigure repo shows examples where config changes may require apps to restart, or you want to filter specific items from being displayed — all of which we use!

## Firebase and other sources

Konfigure works by looking for key-value pairs in a [ConfigSource](https://github.com/TradeMe/konfigure/blob/master/konfigure/src/main/java/nz/co/trademe/konfigure/api/ConfigSource.kt). This allows for configuration to be pulled from any generic String to String map, including existing shared preferences implementations! You can even add multiple config sources, if you have some remote and local config source you want to combine.

We use Firebase at Trade Me, so we provide an implementation of a [`FirebaseRemoteConfigSource`](https://github.com/TradeMe/konfigure/blob/master/konfigure-firebase/src/main/java/nz/co/trademe/konfigure/firebase/FirebaseRemoteConfigSource.kt) for users of Firebase to simply drop in and use, in the `konfigure-firebase` dependency. If you want to add support for some other configuration framework, simply create your own `ConfigSource` and plug it in!

***

# Conclusions

This series of articles focused on Kotlin Property Delegation. We started by looking into basic usage, and continued to see how we can use Property Delegation to it’s fullest extent. We explored building simple, targeted property delegates to share code and make development a breeze. We looked into how the Kotlin language designers have done it, with extension operator functions for things like map delegation.

In this article we looked at a real world problem, and some real world solutions to that problem. Kotlin Property Delegation provided a similar developer experience to other Java + Kotlin solutions. We saw that in this case, Property Delegation came out on top due to less impact on build times, and less impact on runtime performance.

# What you should take away

Kotlin Property Delegation isn’t some magical language feature that will revolutionise the way you write code. Instead, I think it’s a very useful and powerful tool which could be used in more situations, but is perhaps unknown to some. This series of articles should give you insight in to what Kotlin Property Delegation can do — from simple String resolving, to complex configuration libraries.

As with all tools and alternatives, there are trade offs. Before you jump in to using Property Delegation to solve every problem you’re faced with, show good engineering practice and consider all the other alternatives. You may find, now that you know how it works, that Property Delegation is the right tool for the job.