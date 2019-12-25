---
title: Covert — Swipe Actions made easy
author: Jamie Sanson
date: 2018-09-10T00:00:00+12:00
hero_image: "/src/assets/content/images/iker-urteaga-GIXUMw8wsoc-unsplash.jpg"

---
Have you ever wanted to implement swiping on your RecyclerView items, only to find that you’ll have to draw graphics on a canvas to reveal anything behind the view? The [Material Design Documentation website](https://material.io/design/) is full of beautiful mocks of interesting user experiences — more often than not however, these are just mocks. Covert takes the [Material Swipe Actions](https://material.io/design/interaction/gestures.html#types-of-gestures) mock, and builds it into an easily configurable library.

![](/src/assets/content/images/covert.gif)

## Introducing [Covert](https://github.com/TradeMe/Covert)

Covert was created to fill the gap in support for swipe actions. It allows you to add support for swipe actions in just a few lines of code, and is easily configurable for more advanced uses.

## Matching the Guidelines

Applying Covert to your RecyclerView takes just a few lines. Here’s an example of applying Covert to make the interaction shown above.

First off, you need to define your config — this tells the library how to present the swipe interactions.

With your config set up, you can now build Covert and apply it to your RecyclerView.

Step 2. Build Covert and apply it to your RecyclerView

Finally, pass through your Covert instance to your adapter to draw corner flags on your ViewHolders. This is used when initially binding the ViewHolder, to show flags on items which are initially checked.

Step 3. Let covert draw your corner flags

That’s it!

## More Advanced Applications

Covert can be applied using more complex configuration to enable unique swipe actions. For our [Trade Me application](https://play.google.com/store/apps/details?id=nz.co.trademe.trademe), we went with different icons, different colours and a triangular corner flag

For more information on more advanced Covert configurations, check out the [README in the Github Repo](https://github.com/TradeMe/Covert/blob/master/README.md)!

# Give it a whirl!

Covert tries to make implementing Swipe Actions as easy as possible, while still opening up loads of different ways for you to make each interaction unique.

Check out [Covert on Github](https://github.com/TradeMe/Covert) — it’s under the MIT License, so go for gold!