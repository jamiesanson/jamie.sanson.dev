---
title: App Links in a multi-app ecosystem
author: Jamie Sanson
date: 2019-04-02T00:00:00+13:00
hero_image: "/src/assets/content/images/app-links.jpeg"

---
Android App Links helps users get straight to the content they want on Android 6.0 and above. This takes them straight to the app which handles the URL they just clicked on, as opposed to prompting them for their preferred app.

What happens if you have more than one app which handles similar URLs? Here’s some handy tips and tricks to get your apps handling URLs the way you want them to.

# Tip 1. Use more than one intent filter (if you need to)

For bigger companies, there comes a time where you have multiple hosts you need to handle. These might be things like in-house marketing domains, new websites etc. For these kind of situations, it’s valid to add all your hosts into one intent filter element, but you should try to avoid this _as much as possible._ By having more than one host definition in the same intent filter, you can complicate things by overlapping the URL paths you handle for the separate domains.

    <!-- Caution: Catching too many domains makes it hard to be specific. -->
    <data android:host="*mydomain.co.nz">
    <data android:host="*mydomain.nz">

This can cause issues when the URL structures supported by these domains differ. For example:

* [https://www.mydomain.co.nz/view/categories](https://www.mydomain.co.nz/view/categories "https://www.mydomain.co.nz/view/categories")
* [https://mydomain.nz/categories/view](https://mydomain.nz/categories/view "https://mydomain.nz/categories/view")

These URLs could lead the user to a page with similar functionality, but if we swap the paths around we get into a situation where we’re catching URLs that don’t exist. In some cases doing this can make sense, but in general it’s safer to split things up.

We could handle these two URLs above using two separate intent filters, being specific in what paths we catch.

    <!-- Do: Split your Intent Filters into host-specific ones to be more specific. -->
    <intent-filter>
    	...
        <data android:host="*mydomain.co.nz"/>
        <data android:path="/view/categories"/>
    </intent-filter>
    
    <intent-filter>
    	...
        <data android:host="*mydomain.nz"/>
        <data android:path="/categories/view"/>
    </intent-filter>

# Tip 2. Check the logs for App Link Verification

App Links in Android only work correctly when the system can verify that the app you’ve built belongs to the same people that host the website you’re trying to catch URLs from. This is pretty well explained in the [App Links developer docs](https://developer.android.com/training/app-links/verify-site-associations#testing), but one thing they miss out is how to see exactly _which_ domain(s) are causing your verification woes. For this, you want to look at the logs on the device you’re testing on from a _fresh install or upgrade._ A fresh install is important, as the system won’t re-verify the app from incremental changes. To see what the system is doing, you’ll want to filter your logs on one specific class.

`IntentFilterIntentOp` — This class gives you all the results, telling you whether or not your app passed verification, and if not what domains failed. An example of this working correctly can be seen below.

    I/IntentFilterIntentOp: Verification 10 complete. Success:false. Failed host:mydomain.co.nz

If you need more information around specifically _why_ verification may have failed, here’s a few bonus tips on debugging.

* Check out [Google’s Digital Asset Link API](https://developer.android.com/training/app-links/verify-site-associations#test-dal-files). You can call this endpoint to check if your domain verifies properly and returns the right payload.
* Try looking at the logs from `SingleHostAsyncVerifier`. This class is the one responsible for querying your domains for your `assetlinks.json` file, and matching up signatures. This may give you more insight in to why certain hosts are failing.

# Tip 3. Use android:priority to favour the more relevant app

Alright, you’ve structured your app’s intent filters nicely, and have your app links verifying correctly. Cool. But what if you have a main application — one used for most functionality, as well as a couple of other, more specialised apps? This probably isn’t a common use case, but it’s something which may be useful.

Picture this. Your main app catches a URL and provides the user with a pretty alright experience, but your more specialised app has a much more detailed and better user experience on offer. You want to handle the URL in both places, as it’s more likely users have the main app installed. You also want App Links to keep doing their thing, as that’s a much better user experience too.

## Problem: How do you get the system to favour your more specialised app?

**Solution**: Say hello to the lesser-known intent filter property: `android:priority`. This is a property on the `intent-filter` element that allows you to change the order in which Android considers your intent filters. The [documentation](https://developer.android.com/guide/topics/manifest/intent-filter-element#priority) warns to use it sparingly:

> _Use this attribute only if you really need to impose a specific order in which the broadcasts are received, or want to force Android to prefer one activity over others._

Turns out, we can use this to hint to Android that we want to prefer our more specialised apps! Set the priority in your main app to be the minimum value, meaning your more specific apps will be chosen above it without any change to them.

    <intent-filter
    	android:autoVerify="true"
        android:priority="-1000"
        tools:targetApi="m">

# Tip 4. (Optional) Use Plunge to match patterns up to incoming URLs and test your intent-filters

[Plunge](https://github.com/TradeMe/Plunge) is a library we wrote at Trade Me to help with how we create and test deep links in our apps. We found that there were two different parts to deep linking:

* The `intent-filter` side of things, where we need to use basic glob matching to catching things.
* The code side of things where we handle these and extract the information we need to know about to take the user to where they’re going.

These things are quite different — one’s an Android thing we’re stuck with and the other is something we have a lot of control over. The thing is, **they both do the same things.** They catch URLs, and do things.

Plunge tries to unify both components of deep linking by doing two key things:

* It allows you to write simple matching patterns for catching URLs in your code which are very similar to your intent filter patterns.
* It allows you to **write tests once** and use them to test both implementations**.**

Here’s an example of how you’d catch a simple URL with a bit of information:

1. _Define your `pathPattern` in your manifest_

    <data android:pathPattern="/item/..*" />

2\. _Catch the pattern in your code_

    pattern("/item/{itemId}") { launchItem(it.params["itemId"]) }

3\. _Write a test case_

    {  
    	"url": "https://plunge.example.com/item/12345",  
        "description": "The page for buying a submarine",  
        "params": [    
        	{      
            	"name": "itemId",      
                "value": "12345"    
            }  
         ]
    }

Those three simple things are all it takes to handle a new URL. With those additions you can catch a URL, pull out some relevant information, and test both the catching, and the information extracting.

There’s a bit more setup as detailed [in the repo](https://github.com/TradeMe/Plunge), and it has a lot more about how you can use it for testing both positive and negative scenarios. We’ve found it’s simplified our deep linking logic considerably, and gives developers much more confidence in adding new links. Want to know more about the library? Check out [our intro post](https://medium.com/default-to-open/plunge-better-deep-linking-for-android-apps-f331d0bb4648)!