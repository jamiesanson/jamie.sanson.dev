---
title: 'Android Then and Now: Navigation'
author: Jamie Sanson
date: 2021-01-28T00:00:00.000+13:00
hero_image: "/src/assets/content/images/compass-3408928_1920.jpg"

---
Welcome to the third instalment in this series of byte-sized blog posts, where we look at Android Then and Now. We'll go through bits and pieces of Android development that have changed, and look at why and how they are what they are today in less than five minutes of reading. We'll cover things like layouts, dependency management, background work, camera usage, navigation and much more!

Today we’re looking at Navigation, from `ActivityGroup` to `Composable`s.

## Multi-Activity

Multi-Activity navigation is where it all started. An `Activity` gives you access to a `Window`, and the intent system lets you move between Activities without much of a hassle. However, as apps got more complex, one Activity per screen didn’t help you much if you wanted to split up logic for different sections or views, leading to the fabled [God Activity Architecture](https://medium.com/@taylorcase19/god-activity-architecture-one-architecture-to-rule-them-all-62fcd4c0c1d5).

To address this, the Android framework originally shipped with something called [`ActivityGroup`](https://developer.android.com/reference/android/app/ActivityGroup), which “solved” this problem by letting you run multiple activities under another parent activity - each with it’s own Window. With a bit of programmatic View manipulation you can even show them all at the same time! As you probably can guess, this was a terrible idea. Activities are complicated beasts, lifecycles are weird, and getting them to talk to each other is hard. Why not just use one Activity, and something else to control your Views?

## Single-Activity + View Controllers

`android.app.Fragment` came along in API 11, as the replacement for `ActivityGroup` ([deprecated in API 13](https://cs.android.com/android/_/android/platform/frameworks/base/+/2f04883ff880966d63d1aa4a1c7b05e497cfcc58)). Fragments tried to solve the same problem, originally phrased as

> Basic implementation of an API for organizing a single activity into separate, discrete pieces. - from Dianne Hackborn memo, Apr 15 2010

The intention behind this was brilliant, and this solved a very real problem, but as the API surface grew so did the complexity. The Fragment lifecycle ended up being too comprehensive and notoriously difficult to understand. ![](https://i.stack.imgur.com/1llRw.png)

Over time we mostly came to understand Fragments, and best practices formed around callbacks we should be using by default. They weren’t perfect, were sometimes thought of as [footguns](https://en.wiktionary.org/wiki/footgun), and anti-patterns emerged to solve other complex framework problems (looking at you, [setRetainInstance](https://android-review.googlesource.com/c/platform/frameworks/support/+/1159084)). However, overall `Fragment`s introduced a much better alternative to `ActivityGroup`.

For a lot of use-cases, Fragments are to this day a good approach for splitting up your Activities, and are vastly different to the Fragments of old. You can quite easily use a single activity and multiple fragments via the Navigation library, and the recently re-written and re-documented `androidx.app.Fragment` offers a fantastic developer experience thanks to [Ian Lake](https://twitter.com/ianhlake) and co.

As with all things in software, we continue to iterate, and the next logical (?) step is to use as few Views & view controllers as possible.

## Single-View

Want better assurances around how things look across various different Android devices and OS versions? Still don’t like Fragments, even though they’re pretty nice now? Well you’re in luck, as single-view navigation & layout frameworks have you covered.

Single-view navigation frameworks have been around for quite some time in Android, with some arriving as early as [_2008_](https://en.wikipedia.org/wiki/Adobe_AIR). Mobile web frameworks (think Ionic, Xamarin etc.) typically present your entire application within a single (Web)View, and navigation entirely occurs within that View. Pitfalls with web-based frameworks became increasingly evident over the years, and other frameworks began popping up to address these.

Flutter released in 2017, with a lot of the same promised other frameworks were touting - cross-platform functionality, ease and speed of development etc. with the selling point of being faster to develop with, and more performant for users. The problem is, Flutter is written with Dart, and as Android developers we know that [Kotlin is a superior language](https://developers.mews.com/13-reasons-why-dart-is-worse-than-kotlin/). Someone at Google decided we should have a Kotlin version of Flutter, and thus [Jetpack Compose](https://developer.android.com/jetpack/compose) was born.

Late-October 2020 saw the release of [Navigation - Compose](https://developer.android.com/jetpack/androidx/releases/navigation#compose-1.0.0-alpha01), a first party solution to single-View navigation. Although it’s still in development, it’s safe to assume that single-Views and `Composable`s with tend towards becoming the recommended approach within the next few years.

## Conclusion

Build Android apps has changed dramatically since the days of Android 1.X, and how we do navigation is no different. We’ve worked our way through Activities (more than one at a time!), Fragments, and finally to single Views.

What’s next? Will Compose take off, and we entirely forget about the View system? Can we go deeper than single-View? Who knows.

***

1. [IntDefs and Enums](https://jamie.sanson.dev/blog/android-then-and-now-intro-intdef-enums/) 
2. [Callbacks](https://jamie.sanson.dev/blog/android-then-and-now-callbacks/) 
3. [Navigation](https://jamie.sanson.dev/blog/android-then-and-now-navigation/) <--- you are here!
4. [Nullness](https://jamie.sanson.dev/blog/android-then-and-n-caught-exception-nullpointerexception/)

I hope you enjoyed this belated third instalment of Android Then and Now! **Next time:-; Caught exception: NullPointerException**.

If you want to see more, follow me on Medium. Alternatively, I’ll be cross posting to my own personal blog and announcing each instalment on Twitter