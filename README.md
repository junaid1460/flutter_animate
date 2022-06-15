Flutter Animate
================================================================================

A performant library that makes it simple to add almost any kind of animated 
effect in Flutter.

1. Pre-built effects, like fade, scale, slide, blur, shake, and shimmer
2. Easy custom effects
3. Simplified animated builders
4. Synchronized events

All via a simple, unified API without fussing with AnimationController and
StatefulWidget.

> **NOTE:** This library is currently in prerelease. Some aspects of the API
> will change as it is refined. Your feedback is welcome via Github issues.

![Example Image](https://raw.githubusercontent.com/gskinner/flutter_animate/assets/readme_example.gif)

_Above: An example showing some features of Flutter Animate in
[~15 simple lines of code](https://github.com/gskinner/flutter_animate/blob/922e9af5ac23e2df6574fa46c255ba270d9a9af3/example/lib/main.dart#L28)._


Duration extensions
----------------------------------------

This package includes extension methods for `num`, to make specifying durations
easier. For example: `2.seconds`, `0.1.minutes`, or `300.ms`.


Basics
================================================================================

Syntax
----------------------------------------

To apply effects, wrap the target widget in `Animate`, and specify a list of
effects:

``` dart
Animate(
  effects: [FadeEffect(), ScaleEffect()],
  child: Text("Hello World!"),
)
```

It also adds an `.animate()` extension method to all widgets, which wraps the
widget in `Animate()`. Each effect also adds a chainable extension method to
`Animate` to enable a shorthand syntax:

``` dart
Text("Hello World!").animate().fade().scale()
```

> **NOTE:** The shortform style is used in this README, but all functionality is
> available in either format.

Delay, duration, curve
----------------------------------------

Effects have optional `delay`, `duration`, and `curve` parameters. Effects run
in parallel, but you can use a `delay` to run them sequentially:

``` dart
Text("Hello").animate()
  .fade(duration: 500.ms)
  .scale(delay: 500.ms) // runs after fade.
```

Note that effects are "active" for the duration of the full animation, so for
example, two fade effects on the same target can have unexpected results
(`SwapEffect` detailed below, can help address this).

If not specified (or null), these values are inherited from the previous effect,
or from `Animate.defaultDuration` and `Animate.defaultCurve` if it is the first
effect:

``` dart
Text("Hello World!").animate()
  .fadeIn() // uses `Animate.defaultDuration`
  .scale() // inherits duration from fadeIn
  .move(delay: 300.ms, duration: 600.ms) // runs after the above w/new duration
  .blur(end: 8.0) // inherits the delay & duration from move
```

`Animate` also has its own `delay` parameter, which happens before the animation
runs. Unlike the delay on an `Effect`, it is only applied once if the animation
repeats.

``` dart
Text("Hello").animate(
    delay: 1000.ms, // this delay only happens once at the very start
    onInit: (controller) => controller.repeat(), // loop
  ).fadeIn(delay: 500.ms) // this delay happens at the start of each loop
```

Sequencing with ThenEffect
----------------------------------------
`ThenEffect` is a special "convenience" effect that simply sets its own
inheritable delay to the sum of the delay and duration of the previous effect,
and its own (optional) delay. This makes it easier to sequence effects.

In the following example, the slide would run immediately after the fade ended,
then the blur would run 200ms after the slide ended.

``` dart
Text("Hello").animate()
  .fadeIn(delay: 300.ms, duration: 500.ms)
  .then() // sets own delay to 800ms (300+500)
  .slide(duration: 400.ms) // inherits the 800ms delay
  .then(delay: 200.ms) // sets delay to 1400ms (800+400+200)
  .blur() // inherits the 1400ms delay
  // Explicitly setting delay overrides the inherited value.
  // This move effect will run BEFORE the initial fade:
  .move(delay: 0.ms)
```

Animating lists
----------------------------------------

The `AnimateList` class offers similar functionality for lists of widgets, with
the option to offset each child's animation by a specified `interval`:

``` dart
Column(children: AnimateList(
  interval: 400.ms,
  effects: [FadeEffect(duration: 300.ms)],
  children: [Text("Hello"), Text("World"),  Text("Goodbye")],
))

// or shorthand:
Column(
  children: [Text("Hello"), Text("World"),  Text("Goodbye")]
    .animate(interval: 400.ms).fade(duration: 300.ms),
)
```

Shared effects
----------------------------------------

Because `Effect` instances are immutable, they can be reused. This makes it easy
to create a global collection of effects that are used throughout your app and
updated in one place. This is also useful for design systems.

``` dart
MyGlobalEffects.transitionIn = <Effect>[
  FadeEffect(duration: 100.ms, curve: Curves.easeOut),
  ScaleEffect(begin: 0.8, curve: Curves.easeIn)
]

// then:
Text('Hello').animate(effects: MyGlobalEffects.transitionIn)
```


Custom effects & builders
================================================================================

It is easy to write new resuable effects by extending `Effect`, but you can also
easily create one-off custom effects by using `CustomEffect`, `ToggleEffect`,
and `SwapEffect`.

CustomEffect
----------------------------------------

`CustomEffect` lets you build custom animated effects. Simply specify a
`builder` function that accepts a `context`, `value`, and `child`. The child is
the target of the animation (which may already have been wrapped in other
effects).

For example, this would add a background behind the text and fade it from red to
blue:

``` dart
Text("Hello World").animate().custom(
  duration: 300.ms,
  builder: (context, value, child) => Container(
    color: Color.lerp(Colors.red, Colors.blue, value),
    padding: EdgeInsets.all(8),
    child: child, // child is the Text widget being animated
  )
)
```

By default it provides a `value` from `0-1` (though some curves can generate
values outside this range), based on the current time, duration, and curve. You
can also specify `begin` and `end` values as demonstrated in the example below.

`Animate` can be created without a child, so you use `CustomEffect` as a
simplified builder. For example, this would build text counting down from 10,
and fading out:

``` dart
Animate().custom(
  duration: 10.seconds,
  begin: 10,
  end: 0,
  builder: (_, value, __) => Text(value.round()),
).fadeOut()
```


ToggleEffect
----------------------------------------

`ToggleEffect` also provides builder functionality, but instead of a `double`,
it provides a boolean value equal to `true` before the end of the effect and
`false` after (ie. after its duration).

``` dart
Animate().toggle(
  duration: 2.seconds,
  builder: (_, value, __) => Text(value ? "Before" : "After"),
)
```

This can also be used to activate "Animated" widgets, like `AnimatedContainer`,
by toggling their values with a minimal delay:

``` dart
Animate().toggle(
  duration: 1.ms,
  builder: (_, value, __) => AnimatedContainer(
    duration: 1.second,
    color: value ? Colors.red : Colors.green,
  ),
)
```

SwapEffect
----------------------------------------

`SwapEffect` lets you swap out the whole target widget at a specified time:

``` dart
Text("Before").animate().swap(duration: 900.ms, builder: (_) => Text("After"))
```

This can also be useful for creating sequential effects, by swapping the target
widget back in, effectively wiping all previous effects:

``` dart
Widget text = Text("Hello World!");

// then:
text.animate().fadeOut(300.ms) // fade out & then...
  .swap(builder: (_) => text.fadeIn()) // swap in original widget & fade back in
```


Events & callbacks
================================================================================

There are `onInit` and `onComplete` callbacks on `Animate` that trigger when the
whole animation starts or ends. Use the provided `AnimationController` to
manipulate the animation (ex. repeat, reverse, etc).

``` dart
Text("Horrible Pulsing Text")
  .animate(onInit: (controller) => controller.repeat(reverse: true))
  .fadeOut(curve: Curves.easeInOut)
```

For more nuanced callbacks, use `CallbackEffect` or `ListenEffect`.

CallbackEffect
----------------------------------------

`CallbackEffect` lets you add a callback to an arbitrary postion in your
animations. For example, adding a callback halfway through a fade:

``` dart
Text("Hello").animate().fadeIn(duration: 600.ms)
  .callback(duration: 300.ms, callback: () => print('halfway'))
```

As with other effects, it will inherit the delay and duration of prior effects:

``` dart
Text("Hello").animate().scale(delay: 200.ms, duration: 400.ms)
  .callback(callback: () => print('scale is done'))
```

ListenEffect
----------------------------------------

`ListenEffect` lets you register a callback to receive the animation value (as a
`double`) for a given delay, duration, curve, begin, and end.

``` dart
Text("Hello").animate().fadeIn(curve: Curves.easeOutExpo)
  .listen(callback: (value) => print('current opacity: $value'))
```

The above example works, because the listen effect inherits duration and curve
from the fade, and both use `begin=0, end=1` by default.


Installation
================================================================================

Grab it from [pub.dev](https://pub.dev/packages/flutter_animate/install).