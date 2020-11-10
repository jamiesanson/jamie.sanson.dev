---
title: 'Android Then and Now: Intro, @IntDef & Enums'
author: Jamie Sanson
date: 2020-11-10T00:00:00+13:00
hero_image: "/src/assets/content/images/possessed-photography-tojix_nyzfo-unsplash.jpg"

---
Earlier this week I mentioned to a colleague that for the longest time Google didn’t recommend using `enum`s in Android Development. I’ve been an Android Developer for 5 years and didn’t think much of it, but to this fact is apparently pretty mind-boggling. This got me thinking, what else has changed over the years? As it turns out, modern Android development is very different to Android development of old.

Join me for a series of byte-sized blog posts where we look at Android Then and Now. We’ll go through bits and pieces of Android development that have changed, and look at why and how they are what they are today in less than five minutes of reading. We’ll cover things like layouts, dependency management, background work, camera usage and much more.

We’re starting the series off with a bang by looking at `enum`s, and why they were so highly discouraged.

## Back then, memory was tight

The title of this section really sums up why enums were frowned upon. We’re skipping quite a few beats here though — let me explain.

In the early days, mobile devices were constrained hardware-wise. Most people kept hold of a phone for a year or two, meaning in 2015 one of the most popular devices in New Zealand was the Samsung Galaxy S3. This device was really one of the big drivers of Android in the early days, delivering incredible performance on Android 4.X with… 1GB of RAM. As it turns out, 1GB of RAM doesn’t leave much free when considering the overhead of running an OS, as well as multiple apps.

Memory was a sought after resource. We all wanted to be good mobile citizens and to be app that people kill last. The focus therefore had to be on app memory consumption.

## Enums are classes, and classes consume memory

The main issue with enums is that they’re classes under the hood, and classes have overhead. If you’ve got 5 minutes to spare and want to know exactly how bad this can be, I’d recommend looking back at [The Price of Enums](https://www.youtube.com/watch?v=Hzs6OBcvNQE) from back in 2015. For a TL;DW: enums consume about 13x more dex (Dalvik Executable) memory space than standard integer constants, with additional runtime allocation overhead from an array maintained under the hood to keep track of values.

For example, this definition sets you back just over 1.6KB in dex space:

```java
public static enum Value {
	VALUE1,
	VALUE2,
	VALUE3
}
```

When instead making use of static integer constants, we’re down to just 125B:

```java
public static final int VALUE1 = 1;
public static final int VALUE2 = 2;
public static final int VALUE3 = 3;
```

As you can imagine, apps which make extensive use of enums could quite easily blow up in memory usage. This puts pressure on the system, meaning these apps are more likely to be killed.

## Introducing @IntDef, turning enums into ints

So we’ve seen enums are bad, and ints are good. The Android team ran with this, and implemented tooling to help developers make use of ints over enum. This took the form of the `@IntDef` annotation and accompanying lint rules. Using these we’ll extend our integer example from above like so, giving us (mostly) the same build time constant-group benefits without the memory overhead!

```java
// We define our set of ints in the @IntDef annotation,
// giving us similar build time assurances as with enums
@Retention(CLASS)
@IntDef({VALUE1, VALUE2, VALUE3})
public @interface Value {}

public static final int VALUE1 = 1;
public static final int VALUE2 = 2;
public static final int VALUE3 = 3;

public void func(@Value int value) {}

// The following is a lint error! If enabled, this will fail
// your builds 💥
func(4);
```

Oh and also in some circumstances [R8](https://developer.android.com/studio/build/shrink-code) does this for you! If you’re not accessing the enum values array and you haven’t implemented any methods on the enum, it can be flattened into a set of int constants.

## Android Now: Just use enums already

There have been quite a number of improvements to Android ecosystem over the past 5 years. ART (the Android RunTime) has improved, allowing dex space usage to shrink considerably. Additionally, the average amount of RAM available on devices has shot up considerably as device manufacturers move to cater for more performance-hungry apps. Some devices are now firmly in laptop territory, with the Galaxy S20 Ultra packing 16GB of RAM.

The official guidance stating not to use enums no longer exists. Use enums whenever you want! They’re a much nicer abstraction than `@IntDef` allows and devices now have silly amounts of memory, so they don’t mind. If you want to know more about how ART has helped things further, Chet Haase has a great segment in a [this talk from Google I/O 2018](https://youtu.be/IrMw7MEgADk?t=608) talking about runtime improvements.

All in all, memory usage is something most apps no longer have to pay too much attention to. Apps which allocate bitmaps still do, and as do apps which might load ML models into memory, but the majority of apps get off fairly lightly. The [official guidance](https://developer.android.com/topic/performance/memory#Abstractions) nowadays is at a much higher level: Use abstractions as you see fit. However, they can be expensive, so avoid them if you’re not seeing significant benefit.

***

I hope you enjoyed the first instalment of Android Then and Now! I’ve got a backlog of topics, and I’ll be sticking to a cadence of roughly once a week. If there’s any weird old Android thing you remember, reach out and I’ll cover it!

If you want to see more, follow me on [Medium](https://medium.com/@jamiesanson). Alternatively, I’ll be cross posting to my [own personal blog](https://jamie.sanson.dev) and announcing each instalment on [Twitter](https://twitter.com/jamiesanson)
