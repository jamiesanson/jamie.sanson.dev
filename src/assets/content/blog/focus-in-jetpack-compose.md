---
title: Focus in Jetpack Compose
author: Jamie Sanson
date: 2021-06-17T00:00:00+12:00
hero_image: "/src/assets/content/images/stefan-cosma-0go3-b-5m80-unsplash.jpg"

---
It takes a mindset shift to really get going with Compose. When you start thinking in the right way, working with declarative UI is incredibly fast and rewarding. It takes time to train yourself out of old habits, relearning how to do things. Focus management is a little different in Compose versus what we’re used to in plain old Android, so let’s take a look!

## Setting the scene

For a short while I’ve been working on a [todo list app](https://github.com/jamiesanson/tick) as a side project. This app is set to do many fancy things, with a UI built entirely in Compose. Disregard all the fancy things for the time being, and let’s _focus_ on the core experience of using this app. Here’s a snippet of code describing a basic todo list.

```kt
LazyColumn {
    items(todos) { todo -> 
        TodoRow(
            text = todo.text,
            isDone = todo.isDone, 
			callbacks = TodoCallbacks(/* For things a row can do */)
        )
    } 
}
```

I set out this morning to improve the navigability of this list. It should be quick and easy to add and remove todos using nothing but the keyboard. This gif shows the kind of behaviour I was after, taken from Google Keep.

![](/src/assets/content/images/ezgif-com-gif-maker.gif)

In a nutshell, if I’m typing out a thing that needs doing and press enter, I expect my cursor to move to a new line with a new todo item. If I backspace out an item entirely, I expect the item to be deleted and my cursor to be moved to the line above. It’s clear we need to manage focus, so let’s look at our options.

## FocusManager

Compose UI includes a `FocusManager` type which allows you to push focus around in one-shot calls, perfect for simple content traversal.

For example, you might have a simple column of `TextField`s in a form which you want to navigate through when the user presses the “next” button on their keyboard. `FocusManager` will traverse your focus hierarchy for you, and find the next thing to focus on in the direction you ask for. Here’s some code, with `FocusDirection.Down` causing the focus to move to the next field in the list.

```kt
Column {
    val focusManager = LocalFocusManager.current

    for (i in 1..4) {
        TextField(
            // ...
            keyboardActions = KeyboardActions(
                onNext = {
                     focusManager.moveFocus(FocusDirection.Down)
                }    
            )
        )
    }	
}
```

What if you want `Down` to mean something other than the next element on the list? Or you want your custom composable to be focusable? Well you’re in luck, as focus modifiers underpin `FocusManager`, and can be used for any manner of things.

## Focus Modifiers

You can interact with the focus mechanisms in Compose through a few different modifiers

* `Modifier.focusTarget()` - This allows you to make the component focusable
* `Modifier.focusOrder()` - In combination with `FocusRequester`s, this allows you to change the focus order
* `Modifier.focusRequester()` - Add a custom `FocusRequester`, allowing you to observe focus state, and request focus for individual components
* `Modifier.onFocusEvent()`, `Modifier.onFocusChanged()` - Easier way to observe internal, or actual changes to focus state

To understand how these work, let’s look at some examples.

### Observing focus state with `onFocusChanged`

`onFocusChanged` allows you to react to focus changes affecting your composable, or it’s children:

```kt
TextField(
    value = "My text field",
	onValueChange = { },
	modifier = Modifier.onFocusChanged { focusState -> 
        when {
            focusState.isFocused -> 
                println("I'm focused!")
            focusState.hasFocus -> 
                println("A child of mine has focus!")
        }
    }
)
```

You can get finer-grained changes via the `onFocusEvent` modifier, which will emit a new `FocusState` whenever the internal focus state is written to.

### Implement focussing for a custom Layout

When implementing a custom composable, you may it to be focusable for interactivity. In this case, `focusTarget()` is your friend!

```kt
@Composable
fun CoolFocusableGraph(modifier: Modifier = Modifier) {
    // Make ensure our laid out component is focusable, and 
    // observe focus events to make it interactive
    val customComponentModifier = modifier
            .focusTarget() // Now focusable!
            .onFocusEvent { TODO("React to events") }
            .drawBehind { TODO("Draw something cool") }
 
    Layout(
        content = {},
        modifier = customComponentModifier,
        measurePolicy = TODO()
    )
}
```

### Changing the focus order

There might be cases where you want the focus order to differ from the default. In most cases this is inadvisable, but if you need it, here’s how to do it. One bad example could be skipping over fields when they’re populated with valid data.

```kt
// First, get a reference to two focus requesters
val (first, second) = FocusRequester.createRefs()

Column {
    // Down should take us to the third component
    TextField(
        ...
        modifier = Modifier.focusOrder(first) { down = second }
    )

    // Skip this one when moving in the "down" direction
    TextField(...)

    // Set the requester to tie them together
    TextField(
        ...
        modifier = Modifier.focusOrder(second)
    )
}
```

### Requesting focus of a specific component programmatically

Even more inadvisably, you can opt to manage focus all by yourself! This can be a bad idea if you’re not careful enough, as it’s easy to miss subtleties in how focus traversal is expected to work. If you are careful enough, you end up re-implementing focus traversal logic without using focus internals - not fun. In general, it’s worth trying to piggyback off `FocusManager` as much as possible.

If this is absolutely something you need to do though, you can use `FocusRequester` with the `focusRequester()` modifier to programmatically request focus for specific components.

```kt
// Focus could be part of your state
data class InputField(val text: String, val isFocused: Boolean)

@Composable 
fun InputRow(item: InputField) {
    val requester = FocusRequester()

    TextField(
        ...
        modifier = Modifier.focusRequester(requester)
    )

    // Request focus as a SideEffect (after the composition)
    SideEffect {
        if (item.isFocused) {
            requester.requestFocus()
        }
    }
}
```

## Traversing a todo list

Now that we’ve seen all there is to see with focus, let’s try and apply it to our todo list problem.

My desired solution was `FocusManager`, where I attempted to move focus around in either the up or down direction when adding or remove items. This worked nicely for removing items, as the next focus target already existed. When it came to moving the focus onto newly added items, things started to get a little more complicated. I was attempting to move focus down to a component which did not currently exist, leaving me on the line I was on to start with.

The fix for this, I first thought, was to try and run the `focusManager.moveFocus` call in a `SideEffect` - a piece of code which runs after every composition. This made sense as I’m effectively publishing state to something external, but it made things trickier still as I’d only want to call this once per item added to the list. Really what I wanted was to shift the focus after a successful composition where the list of items has changed. Eventually I landed on a combination of `LaunchedEffect`, and `focusManager.moveFocus` calls, to move the focus only when the list of items changes. The full code for this can be seem below.

```kt
@Composable
fun ListScreen(lists: List<TodoList>) {
    // Get a reference to the current FocusManager
    val focusManager = LocalFocusManager.current
    var focusDirectionToMove by remember { mutableStateOf<FocusDirection?>(null) }
    
    // Redux dispatch - stay tuned for a blog about this
    val dispatch = LocalDispatch.current

    // When add or remove events are dispatched, move the focus
    val wrappedDispatch: (Any) -> Any = { action ->
        when (action) {
            is Action.AddTodo, 
            is Action.AddTodoAsSibling -> focusDirectionToMove = FocusDirection.Down
            is Action.DeleteTodo -> focusDirectionToMove = FocusDirection.Up
        }

        dispatch(action)
    }

    // My list of items
    TodoListColumn(lists, wrappedDispatch)

    // If we've previously asked to move the focus, do it when
    // the lists parameter changes
    LaunchedEffect(lists) {
        focusDirectionToMove?.let(focusManager::moveFocus)
        focusDirectionToMove = null
    }
} 
```

***

Focus is a tricky subject, and one which could negatively impact your users if not done correctly. The focus APIs in Compose let you configure focus as much or as little as you like - hopefully this gives you an idea of what’s possible!

If you want to see more, follow me on [Medium](https://medium.com/@jamiesanson). Alternatively, I’ll be cross posting to my [own personal blog](https://jamie.sanson.dev), and I occasionally tweeting about things on [Twitter](https://twitter.com/jamiesanson).