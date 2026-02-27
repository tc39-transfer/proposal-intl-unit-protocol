# TC39 Intl Unit Protocol

Status: Stage 1

Champion: Shane F Carr (Google i18n)

## Background

The i18n community has converged on the principle that a number ought to be annotated with the quantity it is measuring. Unlike the localized decimal separators, numbering systems for digits, and grouping strategy, the unit is a core part of the data model to be formatted, not a style that gets applied by the formatter.

For example, when formatting messages, this allows the measurement to be localized with locale unit preferences.

```javascript
const message = "You are {$distance :unit usage=road} from your destination";
let formatter = new MessageFormat("en", message);
formatter.format({
  distance: /* what goes here? */
})
```

A number annotated with a unit is the only data type required for MessageFormat 2.0 that does not have an analog in JavaScript. See: [List of functions in MessageFormat 2.0](https://github.com/unicode-org/message-format-wg/blob/main/spec/functions/README.md).

Adding a new primordial, such as [Amount](https://github.com/tc39/proposal-amount), would solve this problem and multiple others impacting i18n. A key question is how Amount interfaces with Intl and how third-party libraries can implement an Amount-like object. The champions believe that this is a sufficiently different problem space that they are pursuing Intl Unit Protocol as a standalone proposal.

## Proposed Solution

Add a protocol to `Intl.NumberFormat.prototype.format` that accepts a numeric type paired with a unit. The protocol should also be accepted by `Intl.PluralRules.prototype.select`.

Given these inputs:

```javascript
let locale = /* an Intl.Locale, a string, or a list of these */;
let unit = /* a string */;
let value = /* a Number, a BigInt, or a string */;
```

The user can currently write:

```javascript
let formatter = new Intl.NumberFormat(locale, {
    style: "unit",
    unit,
});
let result = formatter.format(value);
```

With a protocol, the user can instead write:

```javascript
let formatter = new Intl.NumberFormat(locale, {
    style: "unit",
});
let result = formatter.format({
    value,
    unit,
});
```

### Construction vs Formatting

Intl has long allowed constructors and formatting functions to be separate. This achieves two ends:

1. The formatter can encapsulate the locale and options to specify the style and context of the value being formatted, such as when initializing a templating engine.
2. Locale data can be initialized ahead of time, allowing increased efficiency when formatting multiple items in a loop.

By moving `unit` from the constructor bucket to the formatting bucket, we advance goal 1.

The proposal comes at some cost to goal 2, since loading the unit display name has some implementation cost that must now be deferred. We will work with ICU[4X] to minimize this cost.

### Currencies

The protocol would allow currency units to be specified in a similar way.

```javascript
let locale = /* an Intl.Locale, a string, or a list of these */;
let currency = /* a string consisting of 3 upper-case ASCII letters */;
let number = /* a Number, a BigInt, or a string */;

// Today:
let formatter = new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
});
let result = formatter.format(number);

// With the protocol:
let formatter = new Intl.NumberFormat(locale, {
    style: "currency",
});
let result = formatter.format({
    number,
    unit: currency,
});
```

### Conflicting Units

If the constructor has a unit and the unit is not the same as the one in the protocol, an exception will be thrown, since this is a programmer error.

```javascript
new Intl.NumberFormat("en", {
    style: "unit",
    unit: "meter",
}).format({
    number: 1234,
    unit: "kilometer",
}) // throws a RangeError
```

When the Amount proposal advances, this can be changed to automatically convert the input unit to the formatter unit.

## Integration with Future Proposals

The Amount proposal seeks to add a primordial encapsulating a numeric type with a unit. It will implement the Intl protocol specified here.

The Decimal proposal seeks to add a primordial that represents a decimal number in a form designed for correct and efficient arithmetic. It will likely be supported as another number type for the `number` field in the protocol.

Other fields could be added to this protocol in the future, such as alternative ways of expressing precision or an override to the number of significant digits specified in the constructor.
