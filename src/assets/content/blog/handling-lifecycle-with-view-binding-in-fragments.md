---
title: Handling Lifecycle with View Binding in Fragments
author: Jamie Sanson
date: 2020-01-15T15:30:00+13:00
hero_image: "/src/assets/content/images/close-up-environment-flora-ground-1151418.jpg"

---
[View Binding](https://developer.android.com/topic/libraries/view-binding "View Binding Documentation") is an upcoming feature in Android, available in Android Studio 3.6 Canary 11+ which allows you to more easily interact with Views. It's quick and easy to enable, and allows for type-safe view access. We're likely all going to be using it in Fragments. Let's explore how we can use it, in a safe and easy way!

# View Binding in Fragments

Let's take a look at the example from the [View Binding Documentation](https://developer.android.com/topic/libraries/view-binding#usage). First we define some layout file:

**result_profile.xml**

    <LinearLayout ... >   
        <TextView android:id="@+id/name" />    
        <ImageView android:cropToPadding="true" />    
        <Button android:id="@+id/button"        
                android:background="@drawable/rounded_button" />
    </LinearLayout>

This then generates a "binding" class, `ResultProfileBinding`. This class contains two fields, `name` and `button`, which refer to the views in our layout file. Nice!

Using this binding in a Fragment takes one more step - inflating it in `onCreateView`, and returning the root view. For example:

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        _binding = ResultProfileBinding.inflate(inflater, container, false)
        val view = binding.root
        return view
    }

On line 6 we're storing our binding class in a property so that we can access it later. As it turns out, due to the lifecycle of Fragment Views, this isn't all we have to do.

# Keeping Track of Lifecycle

When our view is destroyed we need to remember to clear our property, otherwise we'll end up with a [memory leak](https://en.wikipedia.org/wiki/Memory_leak) at best, and crashes at worst! The documentation recommends you do the following in your Fragments:

    private var _binding: ResultProfileBinding? = null
    
    // This property is only valid between onCreateView and
    // onDestroyView.
    private val binding get() = _binding!!
    
    override fun onDestroyView() {
        _binding = null
    }

This method works, but you can see how adding this to several different Fragment classes could get repetitive, and start to feel like boilerplate. Luckily for us, we can shorten this considerably in our Kotlin Fragments!

    private var binding: ResultProfileBinding by viewLifecycle()
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        binding = ResultProfileBinding.inflate(inflater, container, false)
        val view = binding.root
        return view
    }

We now no longer have to override `onDestroyView`, and we've decreased the number of properties we have to write! But, `viewLifecycle()` isn't in AndroidX? How did we do that? By writing our own [Property Delegate](https://jamie.sanson.dev/blog/handing-the-reins-to-kotlin-delegates-part-1-what-and-why/)!

Let's look at a trimmed down definition of `viewLifecycle()`:

    fun <T> Fragment.viewLifecycle(): ReadWriteProperty<Fragment, T> =
            object: ReadWriteProperty<Fragment, T>, LifecycleObserver {
    
                // A backing property to hold our value
                private var binding: T? = null
                
                // The current LifecycleOwner
                private var viewLifecycleOwner: LifecycleOwner? = null
                
                init {
                    // Observe the View LifecycleOwner, then observe the new lifecycle
                    this@viewLifecycle
                        .viewLifecycleOwnerLiveData
                        .observe(this@viewLifecycle, Observer { newLifecycleOwner -> 
                            viewLifecycleOwner?.lifecycle?.removeObserver(this)
                            viewLifecycleOwner = newLifecycleOwner.also { 
                                it.lifecycle.addObserver(this)
                            }
                    })
                }
                
                @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
                fun onDestroy() {
                    // Clear out backing property just before onDestroyView
                    binding = null
                }
                
                override fun getValue(
                    thisRef: Fragment,
                    property: KProperty<*>
                ): T {
                    // Return the backing property if it's set
                    return this.binding!!
                }
                override fun setValue(
                    thisRef: Fragment,
                    property: KProperty<*>,
                    value: T
                ) {
                    // Set the backing property
                    this.binding = value
                }
            }

Wow, there's a lot going on there. Let's break it down.

* `viewLifecycle()` is an extension function of `Fragment`, meaning we can use `Fragment`-related properties.
* `viewLifecycle()` returns a `ReadWriteProperty<Fragment, T>`, an implementation of a property of a Fragment, which is of the generic type `T`.
* We construct an anonymous class which implements `ReadWriteProperty` and `LifecycleObserver`, allowing us to listen to Lifecycle Events.
* In the `init` block, we observe the Fragments `viewLifecycleOwner`. This is due to the fact that the Fragments view may go through multiple different lifecycles, being destroyed and re-created multiple times.
* Finally, when the View's Lifecycle Owner changes, we observe the new Lifecycle. On the `ON_DESTROY` event sent when `onDestroyView` is about to be called, we null out our backing property.

This gives us the same behaviour as in the View Binding Documentation, but with much less code to cart around in our Fragments! You can find a full example in this [Gist](https://gist.github.com/jamiesanson/d1a3ed0910cd605e928572ce245bafc4). For more information around Kotlin Delegated Properties, have a read of my ["Handing the Reins to Kotlin Delegates"]() blog series!