# Relationship to Other Proposals

The F# Pipeline is intended to solve a small, specific problem. Other proposals can be used with pipeline to greater effect than any of them individually, but it doesn't require them.

## Impact on Lifted Pipeline Operator

The [lifted pipeline operator](https://github.com/isiahmeadows/lifted-pipeline-strawman) would not be blocked or adversely impacted. None of the its syntax is used by the F# pipeline. The examples in the current README work fine:

```js
const toSlug = string =>
  string
    |> _ => _.split(" ")
    :> word => word.toLowerCase()
    |> _ => _.join("-")
    |> encodeURIComponent;
```

This would enable pipelines to dispatch based on type, using a well-known Symbol, `Symbol.lift`. Check out [the proposal](https://github.com/isiahmeadows/lifted-pipeline-strawman) for more information.

## Usage with `?` partial application syntax

If the [partial application proposal](https://github.com/tc39/proposal-partial-application) (currently a stage 1 proposal) gets accepted, the pipeline operator would be even easier to use. We would then be able to rewrite the previous example like so:

```js
const person = { score: 25 };

const newScore = person.score
  |> double
  |> add(7, ?)
  |> boundScore(0, 100, ?);
```

Arrow functions currently provide this capability with more syntax. If it was included, code using arrow functions could be simplified to use partial application, improving the overall syntax of the pipeline.

### Usage with `this` bound partial application

If partial application adopts the [`?this` placeholder](https://github.com/tc39/proposal-partial-application/issues/23), then the usage with prototypes example can be simplified thus:

```js
import Lazy from 'lazy.js'

getAllPlayers()
  .filter(p => p.score > 100)
  .sort()
  |> Lazy
  |> ?this.map(p => p.name)
  |> ?this.take(5)
  |> renderLeaderboard('#my-div', ?);
```

While this was previously wrapped in an arrow function, its inclusion turns the entire chain into a flat pipeline.
