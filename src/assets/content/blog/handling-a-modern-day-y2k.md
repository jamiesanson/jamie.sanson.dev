---
title: Handling a Modern Day Y2K
author: Jamie Sanson
date: 2019-05-29T10:00:00+12:00
hero_image: "/src/assets/content/images/android-y2k.jpeg"

---
Trade Me is an old company, about 20 years old. Throughout its time it’s been one of the largest eCommerce sites in New Zealand, allowing kiwis to connect and trade goods in a safe manner. Recently we’ve reached a huge milestone — more than 2 billion listings created on the site since Trade Me’s inception. Just over 2 billion actually, 2,147,483,647…

Google I/O 2019 introduced In-App Updates in Android via the Play Core library. Using it you can ensure users keep up to date by prompting them passively, and you also have the ability to force the user to immediately update their app. This is the story of what prompted us to consider using it, and how we integrated In-App Updates with our applications.

# Trade Me’s Y2K

We’ve just surpassed a huge milestone at Trade Me. More than 2,147,483,647 listings created on the site over the past 20 years. This is obviously something we’re incredibly proud of, but it came with some challenges. The programming-versed reader may have recognised that number. It happens to be 2³¹-1, the maximum value of a primitive integer in most programming languages.

Early on in computer science you learn a bit about bits and bytes, and how numbers are held in memory. 2³¹-1 happens to be the largest signed (+/-) integer value you can hold across four bytes. Trying to fit a bigger number into the same space does different things depending on language. In low level languages like C you may or may not experience integer wraparound. It’s officially undefined behaviour. In higher level languages like Java for example, a few more defined things can happen. The primitive `int` wraps around, so `MAX_INT + 1` is equal to `MIN_INT`. In other cases like string formatting, you get descriptive exceptions. For example, the following code will throw a `NumberFormatException`, as `%d` expects a standard integer.

    String.format("Biiiig number: %d", Integer.MAX_INT + 1);

We knew things would stop working in our apps when our Y2K eventually hit. Our listings have a unique ID, which happens to be the integer describing which numbered listing it is. Turns out there were a few places in our mobile applications where we do things we shouldn’t — trying to format a listing ID as an integer, then displaying it somewhere. We knew when listing ID Y2K struck, any screen which was still trying to do that would crash with a `NumberFormatException`.

We got to work addressing this issue months before this large number was forecast to hit. We rolled out updates to all of our applications in July of 2018, with the 32-bit rollover expected to hit on the 14th of May 2019. We did thorough testing of the older versions of the applications which didn’t have the fix to make sure we knew the behaviour of these broken apps ahead of time. Turns out we weren’t thorough enough. The 14th of May rolled round, and our overall crash rate more than doubled. Turns out almost 20,000 users were still on versions of the app which were almost a year old, and the most we could feasibly do was send them an email trying to describe the situation. We’re still waiting for some of these users to update, and we expect that not all will.

For incidents like this at Trade Me, we have a procedure known as post-mortems. These are essentially JIRA cases where we document what went wrong, who was affected, how we fixed it, and how we’ll avoid it in future. One clear thing we identified is that we need a better way to sunset old versions of our applications.

# Post-post Mortem — Leveraging Play Core

The main outcome of our post mortem was that we realised we needed a better way to deprecate our older app versions, keeping users up to date with the usual “Bug fixes and improvements”. Luckily, the Play Core library had our back. Google I/O 2019 introduced the [in-app updates API](https://developer.android.com/guide/app-bundle/in-app-updates), which as mentioned allows us to force users to upgrade. We didn’t want this for every update, as it feels like a lame user experience. The Play Core library offers an alternative via flexible updates, but for now we were primarily concerned around immediate updates.

Our implementation of in-app updates is blacklist based. It has two main parts:

1. Remotely configurable blacklisted version ranges
2. A synchronous(-ish) update check on app start

## Configuring blacklisted ranges

For a future-proof solution, we wanted to be able to specify numerous different blacklisted ranges in case of a few critical issues across a few different releases. Modelling this in JSON is pretty trivial, all we need is an array of objects which hold a minimum and maximum value. We settled on something that looked like this.

    [{"min": 234, "max": 235}]

We decided on minimum and maximum values both being inclusive, so the configuration above would disable our apps with version code 234 and 235. We then parse this into a set of `IntRange`s, a Kotlin core class for modelling a range of numbers, and check if the current apps `VERSION_CODE` is found in any of them.

    parseConfig(config).any { BuildConfig.VERSION_CODE in it }

Our `config` String is currently held in Firebase Remote config, which is easy to change without breaking deserialisation thanks to the built in JSON validation when editing it 🎉

## Checking for updates

We do a check for updates when the app starts. We have a splash screen activity, which handles a number of different startup scenarios. It handles push notifications, deep links, app actions, as well as standard launcher launches. Following along from the [docs](https://developer.android.com/guide/app-bundle/in-app-updates#install_flexible), we’ve defined this extension function.

    private fun AppUpdateInfo.immediateUpdateRequired(config: String): Boolean = 
    	(updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE || updateAvailability() == UpdateAvailability.DEVELOPER_TRIGGERED_UPDATE_IN_PROGRESS) &&
        isUpdateTypeAllowed(IMMEDIATE) &&
        parseConfig(config).any { BuildConfig.VERSION_CODE in it }

Finally, all we do now is kick off an asynchronous task which calls `AppUpdateManager.getAppUpdateInfo`. With the result of this task, we can check if there’s an immediate update required based off the extension function above. If so, we start the immediate update flow. If not, we continue app start as per usual. Easy!

With all this put in place, we’re now prepared for future show-stopping issues. If we eventually run into some other Y2K scenario, all it takes is a simple Remote Config update to ensure our users stay up to date with a working version of our apps.