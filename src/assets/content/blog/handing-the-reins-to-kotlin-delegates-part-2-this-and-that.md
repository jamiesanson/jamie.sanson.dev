---
title: 'Handing the Reins to Kotlin Delegates — Part 2: This and that'
author: Jamie Sanson
date: 2019-08-14T22:00:00.000+00:00
hero_image: "/src/assets/content/images/kotlin-delegates-hero-2.jpeg"

---
In the [first entry of this series](https://jamie.sanson.dev/blog/handing-the-reins-to-kotlin-delegates-part-1-what-and-why/) we looked at Kotlin Property Delegation in its most basic form, the `ReadOnlyProperty` and `ReadWriteProperty`. These provide a simple way of us extracting getter and setter functions into a separate, portable class. As we saw in the last post, there are some things we haven’t used yet — the `thisRef`, and `property` parameters. In this article we’ll look at what we can do with these, and put them together into useful property delegates.

# A closer look at ReadWriteProperty

Let’s first have a look at the definition of `ReadWriteProperty`, to get an idea of how these extra parameters come in to play.

    /**
     * Base interface that can be used for implementing property delegates of read-write properties.
     *
     * This is provided only for convenience; you don't have to extend this interface
     * as long as your property delegate has methods with the same signatures.
     *
     * @param R the type of object which owns the delegated property.
     * @param T the type of the property value.
     */
    public interface ReadWriteProperty<in R, T> {
        /**
         * Returns the value of the property for the given object.
         * @param thisRef the object for which the value is requested.
         * @param property the metadata for the property.
         * @return the property value.
         */
        public operator fun getValue(thisRef: R, property: KProperty<*>): T
    
        /**
         * Sets the value of the property for the given object.
         * @param thisRef the object for which the value is requested.
         * @param property the metadata for the property.
         * @param value the value to set.
         */
        public operator fun setValue(thisRef: R, property: KProperty<*>, value: T)
    }

The first thing we notice is the second type parameter, `R`. As the KDoc for the class suggests, this parameter refers to the type of object which owns this delegated property. In the previous entry of this series we set this to `Any`, which allows the delegate to be used in any class.

We can also see another parameter we can make use of in our delegates — the `property` parameter. This allows us to access metadata about the property we’re being delegated to, such as name and visibility. We‘ll explore how this can be useful later on in the article.

## The second type parameter — R

`R` is the type which owns the delegate. This allows us to build delegates that are targeted to specific classes, which means we can leverage whatever may be in that `R` class within our delegate — nice! This is especially useful when thinking about Android. We’re all familiar with using `Activity` as a base for our apps. They allow us access to various useful things, like resources and information about the context the app is running in.

Let’s build a simple delegate which leverages the `R` type parameter in the context of Android. We want a nicer way of looking up and using `String` resources. We’ll build a simple delegate which can do this for us! Note, we’re going to use a `ReadOnlyProperty` here as you can’t set the value of `String` resources programmatically.

    class StringDelegate(
        private val stringId: Int
    ): ReadOnlyProperty<AppCompatActivity, String> {
        private var resolvedString: String? = null
    
        override fun getValue(thisRef: AppCompatActivity, property: KProperty<*>): String =
        	resolvedString ?: thisRef.getString(stringId).also { resolvedString = it }
    }

We can define an extension function to make it more idiomatic. Think about Kotlin’s `lazy` delegate function for another example of this.

    fun AppCompatActivity.id(stringId: Int) = StringDelegate(stringId)

We can then use it in our Activity to get access to a string lazily!

    class TestActivity: AppCompatActivity {
        val testString by id(R.id.test_string)
    }

We can use this paradigm for a bunch of different things. In Android, a common problem is getting a reference to a view. We now have Kotlin Android Extensions, which streamlines things immensely. Prior to it’s introduction however, we were stuck in our old Butterknife ways. A proof-of-concept Kotlin Butterknife clone arose from the same author. [Kotterknife](https://github.com/JakeWharton/kotterknife) made use of `thisRef` when used within classes with views, exposing functions such as `bindView`, which could be used in the same way as our `id` delegate function.

## Using the property

We’ve seen how we can use the second type parameter to access functionality outside of our simple delegate class via an argument. The other argument we’re yet to look at is the property one. This gives us information about the property we’re doing the delegation for. For this, instead of building our own delegate, let’s look at an existing delegate in the Kotlin standard library.

Kotlin has a couple of interesting delegate functions when it comes to the standard library — one which uses the property argument is the map accessor delegate. Using the map delegate looks something like this:

    fun test(map: Map<String, String>) {
        val testString by map
    }

There’s a bit to unravel here, but the gist of it is that this will do a look-up on the map for the “testString” key whenever the `testString` variable is accessed. The map is typed to contain `String` values, therefore the type of `testString` will be `String`. Let’s have a look at (a trimmed-down version of) the delegate.

    operator fun <V> Map<in String, V>.getValue(thisRef: Any?, property: KProperty<*>): V =
        get(property.name) as V

This function calls `get` on the map, where the key is the name of the property. It then casts the value to whatever is specified by the map — easy!

You will have also noticed that this doesn’t instantiate and return a property — in fact it looks like a function from within a property! One thing you may have picked up on, an interesting tidbit from the KDoc of `ReadWriteProperty`, is that you don’t necessarily need to use a property interface as a base.

> _This is provided only for convenience; you don’t have to extend this interface as long as your property delegate has methods with the same signatures._

This extension function matches the signature of the `getValue` function we overrode in our String delegate from earlier. It doesn’t need to do anything special like cache the value, so it can simply be an extension function. Every time the variable delegated via `by map` is accessed, this function will be called. This pattern is used by a bunch of delegates within the Kotlin standard library, such as [`componentN` functions](https://kotlinlang.org/docs/reference/multi-declarations.html) for Lists, enabling list destructuring.

# Putting it all together

We’ve seen how we can use the `R` type parameter, and how we can use the property name for uniqueness. Let’s take a look at a problem which is usually quite tricky to solve in Android, but is made painless using property delegation.

## Aside: Config Changes and ViewModels

In Android, retaining software components through configuration change is a tricky thing to do. Before the introduction of Jetpack and Architecture Components, developers used various hacks to achieve this. `ViewModel`s made the problem a lot more simple, by wrapping up this retention logic into a simple package and giving us an easy way of retrieving references to these.

One problem some have encountered with `ViewModel`s is that they sometimes don’t play nicely with Dagger. Issues can arise when the `ViewModel` is retained, but the Dagger component used to inject dependencies into the `ViewModel` is not. This leads to objects becoming “stale”, causing issues which are often hard to debug. To solve this, we can try our hands at implementing a property delegate.

## The RetentionDelegate

By leveraging Android Architecture Components, we can build a simple retention mechanism that caches components in a retained map. We can use the delegates `thisRef` to get access to an instance of a `ViewModelProvider`. As this is scoped to the Android Fragment/Activity we’re using the delegate in, we can safely use the property’s name as a key for our retained map. All we need is some generic way of instantiating the value, and returning the correct type on subsequent calls. Let’s look at a slimmed down version, for retaining arbitrary objects within an Activity.

We’ll start by defining the signature of our function for getting an instance of our delegate:

    fun <T> AppCompatActivity.retained(    
    	initialiser: () -> T) = RetentionDelegate<T>(initialiser)

We’ll now define the `ViewModel` we’ll use to hold references to the objects we want to retain:

    class RetentionViewModel: ViewModel() {        
    	val values: MutableMap<String, Any> = hashMapOf()       
    }

Finally, let’s we’ll fill out `RetentionDelegate`

    class RetentionDelegate<T>(    
    	private val initialiser: () -> T
    ) {       
    	operator fun getValue(        
        	thisRef: AppCompatActivity,        
            property: KProperty<*>): T {        
            // Get an instance of our ViewModel        
            val viewModel = ViewModelProviders            
            	.of(thisRef)[RetentionViewModel::class]       
                
            val key = property.name        
            
            // Return early if the value is already persisted        
            if (viewModel.values.contains(key)) {            
            	return viewModel.values[key] as T        
            }        
            
            // Instantiate the value, store, and return it     
            viewModel.values[uniqueId] = initialiser() as Any        
            return value    
        }
    }

And that’s it! \~30 lines of code has allowed us to implement a generic retention mechanism to use in an Activity 🎉 Our new delegate can now be used within an Activity like this.

    class RetentionTestActivity: AppCompatActivity() {    
    	val component by retained { instantiateDaggerComponent() } 
    }

The first read of this value will call `instantiateDaggerComponent` and subsequent reads while the backing `ViewModel` still exists will return the cached value, even after configuration change!

This sample has been stripped down for demonstration purposes. A more fully-featured, generic implementation of the `RetentionDelegate` can be found in this [Gist](https://gist.github.com/jamiesanson/350910d5f4eba3e909ed99cc96741556).

# What’s next?

We now know everything there is to know about property delegation in Kotlin! We’ve explored all the different parameters which make up the getter and setter operator functions, and discovered how we can use them to create elegant and reusable solutions for an array of common problems.

In [**Part 3: Fast builds, fast runtime. Pick two**](/blog/handing-the-reins-to-kotlin-delegates-part-3-fast-builds-fast-runtime-pick-two/). we’ll look at how far delegates can be taken when used in systems like Android, where we can use components at the application scope to do more than just local logic. We’ll then compare property delegation to what we previously would have used in Java. We’ll look at how property delegation can provide an alternative solution to things like Annotation Processing + Code Generation and Proxy classes, and do a comparison.