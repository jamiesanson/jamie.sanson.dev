---
title: 'Handing the Reins to Kotlin Delegates — Part 1: What and why?'
author: Jamie Sanson
date: 2019-07-04T22:00:00.000+00:00
hero_image: "/src/assets/content/images/kotlin-delegates-1-hero.jpeg"

---
Kotlin is an extraordinary language, which is gaining traction at an incredibly fast pace. From it’s 1.0 introduction in February of 2016, it’s quickly become the primary language for Android development for a lot of people.

In this series of short posts, we’re going to explore one specific part of Kotlin which I think is underutilised — Kotlin Property Delegates. We’ll start with the basics, go further to see how far we can take them, then compare them to what we had in Java.

# What is Delegation?

Delegation is the act of delegating, defined by a quick Google Search as

> Delegate: entrust (a task or responsibility) to another person

When we talk about [delegation in the context of programming](https://en.wikipedia.org/wiki/Delegation_pattern), we’re doing exactly that. We’re entrusting a task/responsibility to another “person” — where “person” is really another software component. This has a bunch of benefits, including easier separation of concerns and composition as opposed to inheritance to name a few.

In Kotlin, we have two types of delegation — [interface delegation](https://kotlinlang.org/docs/reference/delegation.html), and [property delegation](https://kotlinlang.org/docs/reference/delegated-properties.html). For this series we’ll focus on property delegation, but I encourage you to have look at interface delegation if it’s unfamiliar. A standard Kotlin delegated property looks something like this.

    class Example {  
    	var p: String by Delegate()
    }

Here we have a property, `p`, which is a `String`. We’re _delegating_ responsibility for getting and setting the value of this property to a class of our own, `Delegate`. This means the `Example` class doesn’t need to know about any special rules when setting values of our property, which can save us a lot of messy, unrelated code.

# How do we implement a Property Delegate?

Kotlin makes implementation of simple property delegates easy, by providing interfaces in the standard library that we can implement to perform this property delegation easily. There are two interfaces we could use, `ReadOnlyProperty`, and `ReadWriteProperty` , which are used for matching the behaviour of `val`s and `var`s respectively.

Let’s look at a simple problem, and try solve it with and without property delegation. We want a string field that is no longer than 16 characters. Setting it to a string of any length should work, but it must only return a string of less than or equal to 16 characters long. Let’s check it out!

## Without property delegation

Without property delegation, we put this logic in the class where we define the property. Kotlin makes this easy with it’s support for `get` and `set` functions on variables. Without using a property delegation it looks a little bit like this. Note, we’re making use of the properties backing field by using the `field` scoped variable.

    class Example {    
    	var p: String = ""        
        	get() = field.take(16)
    }

We could alternatively do this in a more java-centric way where the property is private and we have public getters and setters, however this would result in more code overall for no real gain.

This works well, and is pretty easy to understand. However, it is perhaps a line of code that isn’t needed in our `Example` class. Let’s write a `ReadWriteProperty` and see how it differs.

## With property delegation

With property delegation, we define our logic outside of the property definition by using the `ReadWriteProperty` interface.

    class Delegate: ReadWriteProperty<Any, String> {  
    	private var backingString: String = "" 
        
        override fun getValue(t: Any, p: KProperty<*>): String =    
        	backingString 
            
        override fun setValue(t: Any, p: KProperty<*>, value: String) {
        	backingString = value.take(16) 
        }
    }

And then our `Example` class becomes what we see earlier in the post, i.e

    class Example {  
    	var p: String by Delegate()
    }

Overall we now have more lines of code than we did without property delegates, but you can see from this that we now have something a lot more portable. If we required this behaviour on more than property, it’s easy to see that this approach allows us to not have to repeat ourselves.

This delegate might not be super useful, but it sets the scene for what’s to come. It’s pretty simple, allowing getting and setting of `String` `var`s with a couple of getter and setter functions. Easy!

# What’s next?

Rest assured, there’s a lot more we can do with this simple `ReadWriteProperty`. You’ll notice there’s one type parameter we’ve set to simply be `Any`. This represents the class we can use this delegate within, `Any` meaning we can use it anywhere. These getter and setter functions also have parameters — a reference to the class this property is being used in, and a reference to the property itself. These can be used for designing _much_ more advanced delegates, as we now have information around where and how it’s being used. We’ll dive more into how we can use this in follow up blog posts:

* [**Part 2: This and that**](/blog/handing-the-reins-to-kotlin-delegates-part-2-this-and-that/). Making use of `thisRef` and the type parameter.
* [**Part 3: Fast builds, fast runtime. Pick two**.](/blog/handing-the-reins-to-kotlin-delegates-part-3-fast-builds-fast-runtime-pick-two/) Comparing property delegation to java staples, like annotation processing + code gen., and proxy classes.