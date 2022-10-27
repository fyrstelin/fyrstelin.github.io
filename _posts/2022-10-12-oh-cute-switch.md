---
tags:
  - JavaScript
  - NPM
---

# Oh Cute - Switch
Through out the last couple of years I've often been missing a switch expression in JavaScript. Sometimes object literals can solve the problem, but having a default handler is a bit subtle and might add some noise to the code.

Inspired by [ShinySwitch](https://github.com/asgerhallas/ShinySwitch), I decided to implement something similar for NodeJS.

## Where is my const?
The general use case is something like:

```javascript
// could be inside a react component
let color
switch (job.status) {
  case 'pending':
    color = 'gray'
    break
  case 'inProgress':
    color = 'blue'
    break
  case 'done':
    color = 'green'
    break
  case 'failed':
    color = 'red'
    break
  default:
    color = 'black'
    break
}
```

Due to simplicity of this example, color can be initialized as black, and that way the default case is redudent, but that is rarely the case (and may even confuses the reader).

Some might want to replace the `let` with a `const`, which can be achieved by using a function:


```javascript
const color = (() => {
  switch (job.status) {
    case 'pending':
      return 'gray'
    case 'inProgress':
      return 'blue'
    case 'done':
      return 'green'
    case 'failed':
      return 'red'
    default:
      return 'black'
  }
})()
```

It is a bit more dense, and this is a pattern I've often been using. It is still a bit bloated though.


## Objects to the rescue
A simple alternative to a switch statement, is simply to use an object:

```javascript
const color = {
  pending: 'gray'
  inProgress: 'blue'
  done: 'green'
  failed: 'red'
}[job.status] ?? 'black'
```

This is very compact and fairly easy to understand. One downside is that you have to find your target in the middle of the switch construct. If you also have some calculations (or side effects) for some of the cases, you need to use functions, to ensure they only apply for the right case:


```javascript
const color = ({
  pending: () => 'gray'
  inProgress: () => 'blue'
  done: () => 'green'
  failed: () => {
    // some calculation or side effect
    return 'red'
  }
}[job.status] ?? (() => 'black'))()
```

Now it becomes less readable due to too many parentheses.

## A cute abstraction

To make switching more elegant, I've created [Cute Switch](https://www.npmjs.com/package/cute-switch):

```javascript
const color = Switch(job.status)
  .case('pending', () => 'gray')
  .case('inProgress', () => 'blue')
  .case('done', () => 'green')
  .case('failed', () => {
    // some calculation or side effect
    return 'red'
  })
  .defaultTo(() => 'black')
```

It it quite close to using a object litteral, but the target have been moved to the top and the `defaultTo` (or `orThrow()`) is the last thing you call - just like the plain `switch`.

Another benefit (due to advanced use of types in TypeScript), is that CuteSwitch tells you which cases you are missing through intellisense - again, just like a normal switch statement.


## Other use cases.
Cute switch also supports switching on classes or discriminated unions - for examples, goto [Cute Switch](https://www.npmjs.com/package/cute-switch)


## Thoughts about the future
Right now Cute Switch is only available as a CommonJS package. At some point it should also be available as an ESM package, but right now their is no need driving this.

For simpler implementation, I've made a `Switch` and a `SwitchOn` function for handling "normal switching" and "class switching" respectively. One day the implementation might merge to form a simpler API.
