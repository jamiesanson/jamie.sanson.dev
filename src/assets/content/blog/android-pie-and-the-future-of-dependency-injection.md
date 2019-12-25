---
title: Android Pie and the Future of Dependency Injection
author: Jamie Sanson
date: 2019-01-14T00:00:00+13:00
hero_image: "/src/assets/content/images/di_header.jpeg"

---
Dependency Injection (DI) is a common pattern used in all forms of development for a number of reasons. Thanks to the Dagger project, it’s taken off as a pattern used in Android development. Recently changes in Android 9 Pie have meant that we now have more options when it comes to DI, specifically with the new [`AppComponentFactory`](https://developer.android.com/reference/android/app/AppComponentFactory) class.

***

Dependency Injection is huge when it comes to modern Android Development. It allows for less code overall when trying to get reference to services you share across classes, and decouples components nicely in general. In this article, we’ll be focussing on Dagger 2, the most widely adopted DI library used in Android Development. We’ll assume some basic knowledge of how it works, but you should be able to follow along without knowing too much. It’s worth noting this article is a bit of a moonshot. It’s interesting and all, but at time of writing, Android 9 Pie doesn’t even appear in the [platform dashboard](https://developer.android.com/about/dashboards/), therefore this probably won’t be relevant to day-by-day development for at least a few years.

# Android Dependency Injection Today

Put simply, we use dependency injection to help provide instances of “dependency” classes to our dependent classes, i.e the ones doing the work. Let’s assume we’re using the [Repository Pattern](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff649690(v=pandp.10)) for handling our data-related logic, and want to use our repository in an Activity for showing the user some data. We might want to use the same repository in a few places, so we use dependency injection to simplify sharing the same instance between a bunch of different classes.

To set this up, we need to provide our repository. We’ll define a `Provides` function in a module, allowing Dagger to know this is the instance we want injected. Note, our repository needs a context instance to do things with files and network. We’ll provide it the application context.

    @Module
    class AppModule(val appContext: Context) {
      
      @Provides
      @ApplicationScope
      fun provideApplicationContext(): Context = appContext
      
      @Provides
      @ApplicationScope
      fun provideRepository(context: Context): Repository = Repository(context)
    }

We now need to define a `Component` to handle injection of the classes we want our `Repository` to be used in.

    @ApplicationScope
    @Component(modules = [AppModule::class])
    interface ApplicationComponent {
        
      fun inject(activity: MainActivity)
    }

Finally, we can set up our `Activity` to make use of our repository. We’ll assume we’ve set up an instance of our `ApplicationComponent` somewhere else.

    class MainActivity: AppCompatActivity() {
     
      @Inject 
      lateinit var repository: Repository
      
      override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Inject our repository
        application.applicationComponent.inject(this) 
        
        // We can use our repository from here on
      }
    }

That’s it! We’ve just set up application-scoped dependency injection using Dagger. There’s a number of different ways that this could be done, however this seems the most simple approach.

# The Issue with Today’s Approach

What we saw in the above examples is two different types of injection — one more obvious than the other.

The first one you may have missed is known as **Constructor Injection**. This is the method of providing dependencies through the constructor of a class, meaning the class using the dependencies has no idea of where the instances came from. This is considered the cleanest form of dependency injection, as it nicely encapsulates our injection logic in our `Module` classes. We used Constructor Injection in our `AppModule` when providing our repository, as seen on line 10 of the first code sample. It needed a `Context` instance, which we provided it via its constructor through the `providesAppContext` function on line 6.

The second, more obvious one we saw is **Member Injection**. This was the method used in our `MainActivity` for injecting our repository. Here, we define fields as injection recipients using the `Inject` annotation. We then tell the `ApplicationComponent` we want our members injected, which it does in our `onCreate` function. This isn’t as clean as Constructor Injection, as we have explicit reference to our component, meaning we’re leaking the concept of injection into our dependent classes. This also has another downside in Android Framework classes, as we need to be sure the first thing we do is inject dependencies. If we do this at the wrong point in the lifecycle, we might accidentally try to use something which hasn’t been initialised.

Ideally, we’d do away with Member Injection entirely. It leaks information about injection in to classes which shouldn’t really know about it, and can potentially cause issues with lifecycles. We’ve seen attempts to make this better, and with `dagger-android` we can do this in an pretty robust way, but ultimately it’d be better if we could just use Constructor Injection. At the moment we can’t use Constructor Injection for a number of Framework classes, such as Activities, Services, Application classes etc. as they’re instantiated for us by the system. It seems we’re stuck with Member Injection for now. However, an interesting inclusion in Android 9 Pie may have just changed this entirely.

# Dependency Injection in Android 9 Pie

As mentioned at the start of the article, Android 9 Pie has included a class called `AppComponentFactory`. The documentation for this is pretty scarce, and is put simply on the developer site as such:

> _Interface used to control the instantiation of manifest elements._

This is intriguing. “Manifest elements” here refers to classes we list in our `AndroidManifest` file — things like Activities, Services, and our Application class. This lets us “control the instantiation” of these elements… so, wait, we can set out how we construct our Activities now? Sweet!

Let’s dive on in. We’ll start by extending [`AppComponentFactory`](https://developer.android.com/reference/android/app/AppComponentFactory) and overriding the `instantiateActivity` method.

    class InjectionComponentFactory: AppComponentFactory() {
      
      private val repository = NonContextRepository()
    
      override fun instantiateActivity(cl: ClassLoader, className: String, intent: Intent?): Activity {
        return when {
          className == MainActivity::class.java.name -> MainActivity(repository)
          else -> super.instantiateActivity(cl, className, intent)
        }
      }
    }

Now we need to declare our component factory in the manifest within the application tag.

    <application android:allowBackup="true"
                 android:label="@string/app_name"
                 android:icon="@mipmap/ic_launcher"
                 android:name=".InjectionApp"
                 android:appComponentFactory="nz.co.trademe.injectiontest.component.InjectionComponentFactory"
                 android:theme="@style/AppTheme"
                 tools:replace="android:appComponentFactory">

From here we can launch our app, and it’s worked! Our `NonContextRepository` is provided via the constructor of the MainActivity. Neat. Note, this does have some caveats. We can’t use `Context` instances here, as our instantiate functions are being called before context exists — confusing! We can go further to constructor inject our Application class, but lets move on to see how Dagger can make this even easier.

## Dagger Multi-Binds For The Win

I won’t go in to how Dagger multi-binding works under the hood, as that’ll take up far too much of the article. For the purpose of this article, all you need to know is that it provides a nice way of injecting things into the constructor of a class without having to manually call the constructor. We can use this to easily inject our framework classes in a scalable way. Let’s see how it all comes together.

Let’s set up our Activity first, to see where we’re headed.

    class MainActivity @Inject constructor(
      private val repository: NonContextRepository
    ): Activity() {
    
      override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // We can use our repository here
      }
    }

From this you can see immediately there’s almost _no_ mention of dependency injection. The only thing we see appear is `Inject` annotation on the constructor.

Our Dagger Component and Module now need to change.

    @Component(modules = [ApplicationModule::class])
    interface ApplicationComponent {
    
        fun inject(factory: InjectionComponentFactory)
    }

    @Module(includes = [ComponentModule::class])
    class ApplicationModule {
      
      @Provides
      fun provideRepository(): NonContextRepository = NonContextRepository()
    }

Nothing much has changed. We now only need to inject our component factory, but how do we instantiate our manifest elements? This is where `ComponentModule` comes in. Let’s take a look.

    @Module
    abstract class ComponentModule {
    
        @Binds
        @IntoMap
        @ComponentKey(MainActivity::class)
        abstract fun bindMainActivity(activity: MainActivity): Any
    
        @Binds
        abstract fun bindComponentHelper(componentHelper: ComponentHelper): ComponentInstanceHelper
    }
    
    @Target(AnnotationTarget.FUNCTION, AnnotationTarget.PROPERTY_GETTER, AnnotationTarget.PROPERTY_SETTER)
    @Retention(AnnotationRetention.RUNTIME)
    @MapKey
    internal annotation class ComponentKey(val clazz: KClass<out Any>

Woah, those are some annotations alright. What we’re doing here is binding our activity into a map, injecting that map into our `ComponentHelper` class, and providing that `ComponentHelper` — all in two `Binds` statements. Dagger knows how to create an instance of our `MainActivity` thanks to the `Inject` annotation so can “bind” a provider for this class, providing the dependencies we need to the constructor automatically. Our `ComponentHelper` looks like this.

    class ComponentHelper @Inject constructor(
      private val creators: Map<Class<out Any>, @JvmSuppressWildcards Provider<Any>>
    ): ComponentInstanceHelper {
    
      @Suppress("UNCHECKED_CAST")
      override fun <T> resolve(className: String): T? =
        creators
          .filter { it.key.name == className }
          .values
          .firstOrNull()
          ?.get() as? T
    }
    
    interface InstanceComponentHelper {
      fun <T> resolve(className: String): T?
    }

Put simply, we now have a map of classes to providers for those classes. When we try to resolve a class by name, we simply find the provider for that class (if we have it), call it to get a new instance of that class, and return it.

Finally, we need to make a change to our [`AppComponentFactory`](https://developer.android.com/reference/android/app/AppComponentFactory) to use our new helper class.

    class InjectionComponentFactory: AppComponentFactory() {
    
      @Inject
      lateinit var componentHelper: ComponentInstanceHelper
    
      init {
        DaggerApplicationComponent.create().inject(this)
      }
    
      override fun instantiateActivity(cl: ClassLoader, className: String, intent: Intent?): Activity {
        return componentHelper
            .resolve<Activity>(className)
            ?.apply { setIntent(intent) } ?: super.instantiateActivity(cl, className, intent)
      }
    }

Run the code again, and it all works! Neat.

## Constructor Injection has its problems

The heading here may be a bit lacklustre. Although we can inject most instances as normal through constructor injection, we have no obvious way of providing a context to our dependencies in standard ways. In Android, context is everything. You need it to access preferences, network, app configuration and more. Our dependencies are often things which use data related services, such as network and preferences. We can get around this by rewriting our dependencies to consist of pure functions, or initialising everything with context instances in our `Application` class, but it requires a lot more thought to determine the best way to do this.

Another drawback of this approach is scoping. In Dagger, one of the key concepts to implementing high-performance dependency injection with good separation of concern is to modularise your object graph and make use of scoping. Although this approach doesn’t inhibit our use of modules, it does inhibit our use of scoping. The [`AppComponentFactory`](https://developer.android.com/reference/android/app/AppComponentFactory) exists at an entirely different level of abstraction to our standard framework classes — we can’t get reference to it programmatically, therefore we have no way of instructing it to provide dependencies for an `Activity` at a different scope.

There are numerous ways to solve our scoping issues in practice, one such being the use of [`FragmentFactory`](https://developer.android.com/reference/androidx/fragment/app/FragmentFactory) for scoped constructor injection of our Fragments. I won’t go in to this, but turns out we have a method of controlling `Fragment` instantiation now, which not only allows us a lot more freedom in terms of scoping, and is also backwards compatible.

# Conclusion

Android 9 Pie introduced a method for us to use constructor injection to provide dependencies for our framework classes, such as Activities and Applications. We saw that with Dagger Multi-binding, we can provide application-scoped dependencies with ease.

Constructor injecting all our components is hugely appealing, and we can even do things to make it work properly with context instances. It’s a promising future, but only available from API 28. If you’re looking to target less than 0.5% of the market, this might be the method for you. Otherwise, we’ll wait and see if this becomes relevant in a few years time.