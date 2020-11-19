---
title: 'Android Then and Now: Callbacks'
author: Jamie Sanson
date: 2020-11-20T00:00:00+13:00
hero_image: "/src/assets/content/images/elena-koycheva-bgeupv246bm-unsplash.jpg"

---
# Android Then and Now: Callbacks
Welcome to the second instalment in this series of byte-sized blog posts, where we look at Android Then and Now. We'll go through bits and pieces of Android development that have changed, and look at why and how they are what they are today in less than five minutes of reading. We'll cover things like layouts, dependency management, background work, camera usage, navigation and much more!

Today we’re changing tack, and looking at a style of programming which is slowly becoming less prevalent in Android development: Callbacks.

## Callbacks are still cool
Callbacks are all over the place in Android Development. That’s simply because they do a job, and they do it well! By definition:
> A callback is a function passed into another function as an argument, which is then invoked inside the outer function to complete some kind of routine or action.

[Wikipedia](https://en.wikipedia.org/wiki/Callback_(computer_programming)) states a couple of key benefits for callbacks: They can be called multiple times, and they can capture information the outer function doesn’t need to worry about. Handling view clicks in Android is a great example of this.

```kt
myButton.setOnClickListener { view -> 
	viewModel.onMyButtonClicked()
}
```

Here, we pass an `OnClickListener` to the `setOnClickListener` function. Kotlin flattens this into a lambda which takes the view clicked as a parameter. Both of these benefits of coroutines become immediately obvious:
* You could click a button more than once, so we need to be able to call back more than one.
* All the implementation of `setOnClickListener` needs to know is that we need a reference to the view clicked. We keep details of what to do when the button is clicked to ourselves.

All in all, callbacks are cool. They’re an elegant solution to a communication problem which is actually quite tricky, and our programming language of choice makes them simple to implement with lambdas.

## Sometimes, callbacks could be cooler
Callbacks are a very general, and very useful programming style. They can be used to solve any number of problems — however in some cases, thanks to improvements to technology, callback might not be the best approach. One such case are callbacks which are called no more than once. How about an example!

Let’s think about a common problem, asynchronous REST API calls. We want to hit some network endpoint, download some information, then continue our program. We don’t want to do work on the main thread, as we’d block the app from running, making it unresponsive to user input. To address this, we move the work of doing a network call to a different thread. We can then talk back to the main thread by posting messages, telling us to resume what we were doing now that the other thread has finished its work. 

In practice, writing the code to post these messages back to the main thread is at the very least error prone, and becomes a pain to maintain. Instead we could delegate the threading work to a library, and wait for a result to be delivered to a callback function! This is one of the ways in which we can use the ever-popular Retrofit library:

```kt
myApi.getThings().enqueue(object: Callback<Things> {
	override fun onResponse(
		call: Call<String>, 
		response: Response<Things>) {}

    override fun onFailure(
		call: Call<String>,
		t: Throwable) {}
})
```

This greatly simplifies how we make network calls, allowing us to avoid the cursed `AsyncTask`. However, with Kotlin introducing coroutines, we can now do one better.

## Suspend funs beat one-shot callbacks
If you think about it, Kotlin coroutines solves the exact same problem Retrofit was solving with `enqueue`. We’re calling a function, waiting for a result which might have come from a different thread, and continuing. There are a couple of key differences, which describes why I think suspend funs are the obvious winner:
* It limits misdirection, the program reads like any other sequential code.
* It simplifies the problem of thread isolation even further, while also being more obvious, as [suspend funs have to be called from a suspendible context](https://kotlinlang.org/docs/reference/coroutines/basics.html#your-first-coroutine).

Let’s look at the same example, but now with coroutines:

```kt
try {
	val myThings = myApi.suspendGetThings()
} catch (e: Exception) {
	// The same as onFailure
}
```

What was two callbacks is now either a result, or an exception. We’ve gone from implementing our own functions to drive control from callbacks to code, to a simple try-catch. Not only is this easier to write, it’s far easier to read too! On top of this there's the added benefit of built in lifecycle management using coroutine scopes and contexts, which makes cancelling things at the right time almost automatic.

Converting callbacks to suspend funs is easy, and [Roman Elizarov does a great job at explaining you how](https://medium.com/@elizarov/callbacks-and-kotlin-flows-2b53aa2525cf). He even goes further, to show how you can replace multi-shot callbacks with other coroutines primitives! 

---
I hope you enjoyed the second instalment of Android Then and Now! Next week: Navigation. If you want to see more, follow me on [Medium](https://medium.com/@jamiesanson). Alternatively, I’ll be cross posting to my [own personal blog](https://jamie.sanson.dev) and announcing each instalment on [Twitter](https://twitter.com/jamiesanson)