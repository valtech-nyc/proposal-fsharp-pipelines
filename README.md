# ESNext Proposal: F# Pipeline Operator

## Introduction

This proposal introduces a new operator `|>` similar to
  [F#](https://en.wikibooks.org/wiki/F_Sharp_Programming/Higher_Order_Functions#The_.7C.3E_Operator),
  [OCaml](http://caml.inria.fr/pub/docs/manual-ocaml/libref/Pervasives.html#VAL%28|%3E%29),
  [Elixir](https://www.safaribooksonline.com/library/view/programming-elixir/9781680500530/f_0057.html),
  [Elm](https://edmz.org/design/2015/07/29/elm-lang-notes.html),
  [Julia](http://docs.julialang.org/en/release-0.4/stdlib/base/?highlight=|%3E#Base.|%3E),
  [Hack](https://docs.hhvm.com/hack/operators/pipe-operator),
  and [LiveScript](http://livescript.net/#piping),
  as well as UNIX pipes. It's a backwards-compatible way of streamlining chained function calls in a functional manner, and provides a practical alternative to extending object prototypes for methods.

### The Problem

At its worst, a series of data transformations can become a deeply-nested and hard-to-read mess of non-linear data flow. Take a look at the following example:

```js
console.log(
  await stream.write(
    new User.Message(
      capitalize(
        doubledSay(
          await promise,
          ', '
        )
      ) + '!'
    )
  )
);
```

Notice how your eyes have to move back and forth, up and down, in order to follow the flow through the code. The inclusion of `await` increases the complexity, as now you have to consider both the data flow as well the impact on the event loop.

This may appear to be a contrived example, but application code often does this kind of data manipulation. In the real world, you might break this out onto separate lines, leading to overly verbose code and unnecessary intermediate variables.

### The Solution

The pipeline operator simplifies the process of chaining several functions together. We can flatten the above invocation to this:

```js
promise
  |> await
  |> x => doubleSay(x, ', ')
  |> capitalize
  |> x => x + '!'
  |> x => new User.Message(x)
  |> x => stream.write(x)
  |> await
  |> console.log;
```

Now the flow reads top to bottom, left to right, without requiring your eyes to jump around the code to follow it. The points at which async control flow comes into play is clear. Steps can be easily added or removed from the flow, or collapsed into each other, without requiring the addition or removal of lots of parentheses or indentation, enabling git diffs to be clearer as well.

## Overview

The pipeline operator provides syntactic sugar over a function call with a single argument. In other words, `64 |> sqrt` would desugar to `sqrt(64)`. This application can be chained, producing a clear sequence of operations to perform on an input.

## Basic Example

Given the following functions:

```js
const doubleSay = str => str + ", " + str;
const capitalize = str => str[0].toUpperCase() + str.substring(1);
const exclaim = str => str + '!';
```

...the following invocations are equivalent:

```js
const result = exclaim(capitalize(doubleSay("hello")));
result //=> "Hello, hello!"

const result = "hello"
  |> doubleSay
  |> capitalize
  |> exclaim;

result //=> "Hello, hello!"
```

## Use with N-ary Functions

The pipeline operator does not need any special rules for functions with multiple arguments; JavaScript can already handle such cases. This keeps the syntactic overhead of the new operator to a minimum while enabling other future syntax to complement it.

For example, given the following functions:

```js
const double = x => x + x;
const add = (x, y) => x + y;

const boundScore = (min, max, score) =>
  Math.max(min, Math.min(max, score));
```

...you can use an arrow function to handle multi-argument functions:

```js
const person = { score: 25 };

const newScore = person.score
  |> double
  |> n => add(7, n)
  |> n => boundScore(0, 100, n);

newScore //=> 57

// As opposed to:
let newScore = boundScore(0, 100, add(7, double(person.score)));
```

As you can see, because the pipeline operator always pipes a single result value, it plays very nicely with the single-argument arrow function syntax. Because the pipeline operator's semantics are pure and simple, JavaScript engines can optimize away the arrow function, as the [Babel plugin](babel) currently does.

### Impact on Precedence

Arrow functions do not require parentheses, allowing developers to use the pipeline operator to build up a composition chain. This means that this:

```js
const a = x => x |> a |> b
```

...will parse as:

```js
const a = x => (x |> a |> b)
```

...with the body of the arrow function a single pipeline. Once inside a pipeline, arrow functions are terminated at the first operator, meaning this:

```js
const updateScore = score =>
  score
    |> double
    |> n => add(7, n)
    |> n => boundScore(0, 100, n);
```

...will parse as:

```js
const updateScore = score => (
  score
    |> double
    |> (n => add(7, n))
    |> (n => boundScore(0, 100, n));
)
```

...enabling an ergonomic means for function composition

TODO: Add this behavior to the spec.

## Use with Methods

When a pipeline is applied to method, the receiver of the method is bound to its current value. That is:

```js
x |> a.f;
```

...will desugar as:

```js
a.f(x);
```

...ensuring the method `f` is called with the correct `this` (`a`).

## Use with `await`

The pipeline operator treats `await` similar to a unary function. `await` can appear in the pipeline like so:

```js
const result = promise |> await;
```

which desugars to:

```js
const result = await promise;
```

This enables awaiting the previous value in the pipeline. That means the following:

```js
const user = url
  |> api.get
  |> await
  |> r => r.json()
  |> await
  |> j => j.data.user;
```

desugars roughly as follows:

```js
const _temp1 = api.get(url);
const _temp2 = await _temp1;
const _temp3 = _temp2.json();
const _temp4 = await _temp3;
const user = _temp4.data.json;
```

Attempting to pipe to `x |> await f` is a Syntax Error. Parentheses would be required (`x |> (await f)`) or it needs to be piped through `f` (`x |> f |> await`), depending on your intention.

## Motivating Examples

### Object Decorators

Mixins via `Object.assign` are great, but sometimes you need something more advanced. A **decorator function** is a function that receives an existing object, adds to it (mutative or not), and then returns the result.

Decorator functions are useful when you want to share behavior across multiple kinds of objects. For example, given the following decorators:

```js
const greets = person => {
  person.greet = () => `${person.name} says hi!`;
  return person;
};
const ages = age => person => {
  person.age = age;
  person.birthday = () => { person.age += 1; };
  return person;
};
const programs = favLang => person => {
  person.favLang = favLang;
  person.program = () => `${person.name} starts to write ${person.favLang}!`;
  return person;
};
```

...you can create multiple "classes" that share one or more behaviors:

```js
function Person (name, age) {
  return { name } |> greets |> ages(age);
}
function Programmer (name, age) {
  return { name }
    |> greets
    |> ages(age)
    |> programs('javascript');
}
```

### Validation

Validation is a great use case for pipelining functions. For example, given the following validators:

```js
const bounded = (prop, min, max) => obj => {
  if (obj[prop] < min || obj[prop] > max) throw Error('out of bounds');
  return obj;
};
const format = (prop, regex) => obj => {
  if (!regex.test(obj[prop])) throw Error('invalid format');
  return obj;
};
```

...we can use the pipeline operator to validate objects quite pleasantly:

```js
const createPerson = attrs =>
  attrs
    |> bounded('age', 1, 100)
    |> format('name', /^[a-z]$/i)
    |> Person.insertIntoDatabase;
```

### Usage with Prototypes

Although the pipeline operator operates well with functions that don't use `this`, it can still integrate nicely into current workflows:

```js
import Lazy from 'lazy.js'

getAllPlayers()
  .filter(p => p.score > 100)
  .sort()
  |> _ => Lazy(_)
    .map(p => p.name)
    .take(5)
  |> _ => renderLeaderboard('#my-div', _);
```

Arrow functions are able to easily handle this use case.

### Importable Methods

In the above example, `lazy.js` could update its API to use Importable Methods instead of attaching them to the prototype. This allows the importing application to only include the methods they actually use and tree-shake the code the don't use.

Developers can use it in essentially the same way they do now with a fluent interface, except they use `|>` instead of `.`:

```js
import Lazy, { map, take } from 'lazy.js';
import { filter, sort } from './array-utils';

getAllPlayers()
  |> filter(p => p.score > 100)
  |> sort()
  |> Lazy
  |> map(p => p.name)
  |> take(5)
  |> _ => renderLeaderboard('#my-div', _);
```

Now, `lazy.js` can have all its unused methods pruned from the final application bundle.

### Mixins

["Real" Mixins](http://justinfagnani.com/2015/12/21/real-mixins-with-javascript-classes/) have some syntax problems, but the pipeline operator cleans them up quite nicely. For example, given the following classes and mixins:

```js
class Model {
  // ...
}
let Editable = superclass => class extends superclass {
  // ...
};
let Sharable = superclass => class extends superclass {
  // ...
};
```

... we can use the pipeline operator to create a new class that extends `Model` and mixes `Editable` and `Sharable`, with a more readable syntax:

```js
// Before:
class Comment extends Sharable(Editable(Model)) {
  // ...
}
// After:
class Comment extends Model |> Editable |> Sharable {
  // ...
}
```

## Real-world Use Cases

Check out the [Example Use Cases](https://github.com/tc39/proposal-pipeline-operator/wiki/Example-Use-Cases) wiki page to see more possibilities.

## Implementation

Check out [@babel/plugin-proposal-pipeline-operator](https://github.com/babel/babel/tree/master/packages/babel-plugin-proposal-pipeline-operator).
