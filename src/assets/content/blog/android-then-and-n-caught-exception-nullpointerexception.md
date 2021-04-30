---
title: 'Android Then and N-; Caught exception: NullPointerException'
author: Jamie Sanson
date: 2021-04-30T00:00:00+12:00
hero_image: "/src/assets/content/images/question-mark-2492009_1920-1.jpg"

---
Welcome to the fourth instalment in this series of byte-sized blog posts, where we look at Android Then and Now. We'll go through bits and pieces of Android development that have changed, and look at why and how they are what they are today in less than five minutes of reading. We'll cover things like layouts, dependency management, background work, camera usage, navigation and much more!

This time, let’s talk `null`.

## Plain old Java

In the beginning, there was Java.

There are a bunch of reasons why Java was chosen as the programming language of choice for Android development, most of which stem from key benefits of virtual machines: isolation, and portability. The decision to go with Java was probably made early on in Android’s history, and the ramifications of that decision are still showing up to this day, [with later additions causing a case to escalate to the Supreme Court](https://en.wikipedia.org/wiki/Google_LLC_v._Oracle_America,_Inc.).

One of the ramifications we’ve had to manage as developers is the fact that in Java, fields, variables and parameters can be null. Null represents an absent value, and trying to operate on a value which isn’t there throws an exception. Doing the following is a bad idea, and will cause your app to crash:

```java
String myString = null;
myString.length(); // 💥 NullPointerException
```

Instead, good programming practice was (and still is) to program defensively. This means null checks:

```java
if (myString != null) {
    myString.length(); // ✅ No exception here!
}
```

In some cases, you could expect things to never be null. You might null check parameters before passing them into a method. If you have that at all the callsites of the method, you’re safe!

```java
if (myString != null) {
    computeBetterLength(myString); // myString can't be null
}

int computeBetterLength(String myString) {
	return myString.length(); // myString can't be null!
}
```

However, someone might come along and write new code which passes null in to your method anyway, so you might consider writing a null check within the method anyway to be defensive. This leads to duplication.

```java
if (myString != null) {
    computeBetterLength(myString); // myString can't be null
}

computeBetterLength(null); // -1 instead of NPE :shrug:

int computeBetterLength(String myString) {
	if (myString != null) { // We've already done this
		return string.length();
	} else {
		return -1;
	}
}
```

## Java + Nullness Annotations

NullPointerExceptions were so rampant and devastating in Java-based applications that annotations were introduced to warn the developer about potential issues while writing the code. This means more certainty around when you need to defensively null check.

```java
/**
 * Please don't pass in null. Consumers should do their own 
 * null checking, thanks.
 */
int computeBetterLength(@NonNull String myString) {
	return myString.length();
}
```

Lint can then infer nullness issues in and around this annotated code, and can generate warnings to hint at where unsafe code might exist. As time went on nullness annotations appeared in a lot of important SDKs, with the Android SDK itself gradually annotating its entire public API surface. Given the incremental approach, NullPointerExceptions slowly declined in prevalence in well annotated code. Until, that is, new contenders entered the ring.

## Other Languages

Instead of trying to duck tape a null-safety system together with lint, why not fix it in the type system? This is a common selling point of alternative JVM languages, such as the now-preffered Kotlin.

Java is implicit with its nullness. Things are null if you don’t do anything with them (besides primitives). Kotlin, and other JVM languages like Scala, approach nullness by being explicit. Instead, types are implicitly not-null unless otherwise specified. Null still exists, but it’s usage is modelled and enforced by the compiler, meaning _almost_ no NullPointerExceptions.

```kt
val myString: String = "" // Non-null
val myNullableString: String? = null // Nullable

// This is fine
val length = myString.length 

// 💥 Compilation error, IDE tells you to not do this
val nullableLength = myNullableString.length 

// What if I just sneak a null in there anyway?
// 💥 Compilation error, IDE tells you to not do this
val myString2: String = null 
```

And thus, as Kotlin became more widely adopted in Android development,  NullPointerExceptions became a thing of the past. Instead of crashing for consumers of our apps, our apps don’t compile - shifting errors left, and causing us to think more about how we model optionality.

***

I hope you enjoyed the second instalment of Android Then and Now! Next week: Navigation. If you want to see more, follow me on [Medium](https://medium.com/@jamiesanson). Alternatively, I’ll be cross posting to my [own personal blog](https://jamie.sanson.dev) and announcing each instalment on [Twitter](https://twitter.com/jamiesanson)