+++
date = '2026-01-21T12:42:06+02:00'
draft = false
title = 'Extending Parser Combinators'
description = 'This article is part of a series. An exploration of advanced parser combinators with some practical examples in TypeScript.'
series = ['Parser Combinators with Typescript']
tags = ['TypeScript', 'Parsing', 'Parser Combinators', 'Functional Programming']
series_order = 2
+++

In the previous article in the series, we explored the basics of parser combinators and created some simple parsers. In this article, we'll extend the parser combinator library to handle more complex scenarios, including self-referential (recursive) structures.

## Number literal parsing

An exercise left for the reader in the previous article was to create a parser for number literals:

> What other combinators do you think would be useful to implement? Can you think of a way to implement a parser that matches a positive number? Example: `42`, `3.14`, `0.001`

Let's implement a parser for positive number literals, including both integers and floating-point numbers. We have most of the building blocks we need already implemented, but we need to create an additional combinator to handle an arbitrary number of repetitions, until the condition is not met anymore or the input is exhausted. You can think of this as the `*` or `+` operators in regular expressions (we'll implement both variants).

```typescript
class P<T> {
    many(): P<T[]> {
		return new P<T[]>((input) => {
			const values: T[] = [];
			let currentInput = input;

			while (true) {
				const result = this.run(currentInput);
				if (!result.success) {
					break;
				}
				values.push(result.value);
				currentInput = result.remaining;
			}

			return { success: true, value: values, remaining: currentInput };
		});
	}

	many1(): P<T[]> {
		return new P<T[]>((input) => {
			const result = this.run(input);
			if (!result.success) {
				return { success: false };
			}

			// P::many parsers always succeed
			const manyResult = this.many().run(result.remaining) as ParseResult<
				T[]
			> & { success: true };
			return {
				success: true,
				value: [result.value, ...manyResult.value],
				remaining: manyResult.remaining,
			};
		});
	}
}
```

It would be helpful to also add a building block for character groups, to avoid long `or` chains:

```typescript
class P<T> {
    static dictionary(str: string): P<string> {
		const set = new Set([...str]);
		return new P<string>((input) => {
			for (const word of set) {
				if (input.text.slice(input.index, input.index + word.length) === word) {
					return {
						success: true,
						value: word,
						remaining: { text: input.text, index: input.index + word.length },
					};
				}
			}
			return { success: false };
		});
	}

	static digit(): P<string> {
		return P.dictionary("0123456789");
	}
}
```

This leaves us with everything needed to implement a number literal parser:

```typescript
const zero = P.char("0");
const nonZero = P.dictionary("123456789");
const integer = P.seq([nonZero, P.digit().many()]).or(zero);
const float = P.seq([integer, P.char("."), P.digit().many1()]);
const numberParser = float.or(integer);
```

Pretty self-documenting, right? We could test our parser as it is, but the output as is would be a bit messy. By hovering over the type of `numberParser`, we can see it has the following output type:

```typescript
ParseResult<"0" | [string, string[]] | ["0" | [string, string[]], ".", string[]]>
```

It would be practical to join back the segments into a single string. Furthermore, it would be helpful to add metadata to indicate whether the parsed number is an integer or a float. Imagine that you're implementing a full programming language parser; having this information would be useful for allocating the right type of variable or performing type checking. We'll achieve this by using the `map` combinator we implemented in the previous article:

```typescript
const zero = P.char("0");
const nonZero = P.dictionary("123456789");
const integer = P.seq([nonZero, P.digit().many()])
	.map(([first, rest]) => first + rest.join(""))
	.or(zero);
const integerToken = integer.map((value) => ({
	type: "IntLiteral" as const,
	value: value,
}));
const float = P.seq([integer, P.char("."), P.digit().many1()]).map(
	([intPart, dot, fracPart]) => intPart + dot + fracPart.join(""),
);
const floatToken = float.map((value) => ({
	type: "FloatLiteral" as const,
	value: value,
}));
const numberParser = floatToken.or(integerToken);
```

Now, when we parse a number literal, we get a nicely structured output:

```typescript
const result = numberParser.parse("123.0014");
if (result.success) {
  console.log(result.value);
  // Output: { type: 'FloatLiteral', value: '123.0014' }
}
```

Up for a small challenge? Try extending the parser to handle signed numbers, with an optional `+` or `-` prefix!

## Nested structures

It's extremely common for programming languages to have nested structures. For example, in javascript, we could have nested array literals like this:

```javascript
const arr = [1, 2, [3, 4], [5, [6, 7]]];
```

To parse such structures, the parser needs to be self-referential - every element inside the array can itself be an array. Let's implement a parser for this, but to keep things simple, we'll only handle nested arrays of integers (we already defined a parser for integers above).

## Sub-problem: Non-nested integer array

We will need an additional combinator, similar to `many`, but that expects its elements to be separated by a specific delimiter (in this case, a comma). Let's call it `sepBy`:

```typescript
class P<T> {
    sepBy<U>(separator: P<U>): P<T[]> {
		return new P<T[]>((input) => {
			const firstResult = this.run(input);
			if (!firstResult.success) {
				return { success: true, value: [], remaining: input };
			}

			const othersParser = P.seq([separator, this])
				.map(([, value]) => value)
				.many();
			const othersResult = othersParser.run(
				firstResult.remaining,
			) as ParseResult<T[]> & { success: true };

			return {
				success: true,
				value: [firstResult.value, ...othersResult.value],
				remaining: othersResult.remaining,
			};
		});
	}
}
```

One more thing we have to handle for the sub-problem is whitespaces between elements. Those should be ignored. We can create a parser for whitespace characters and use it to wrap our element parsers:

```typescript
class P<T> {
    static whitespace(): P<string> {
        return P.dictionary(" \t\n\r");
    }

    between(boundary: P<unknown>): P<T>
    between(left: P<unknown>, right: P<unknown>): P<T>
    between(left: P<unknown>, right?: P<unknown>): P<T> {
        const rightBoundary = right ?? left;
        return P.seq([left, this, rightBoundary]).map(([, value]) => value);
    }
}
```

With these additions, we're ready to implement the non-nested array parser. To wrap up the requirements for the sub-problem, we need:
- The elements to be surrounded by `[` and `]`
- Elements to be separated by `,`
- Each element to be an unsigned integer
- Whitespaces to be ignored inside the array definition

```typescript
const intArrayElement = integer.map(Number.parseInt);
const arraySeparator = P.char(",").between(P.whitespace().many());

const intArrayParser = intArrayElement
	.sepBy(arraySeparator)
    .between(P.whitespace().many())
	.between(P.char("["), P.char("]"));
```

We can give it a try:

```typescript
const arrayResult = intArrayParser.parse("[1, 22 ,  3, 4,5]");
if (arrayResult.success) {
	console.log(arrayResult.value);
}
```

And that produces the expected output:

```
[ 1, 22, 3, 4, 5 ]
```

### Self-referential parser for nested integer arrays

Now that we have a parser for non-nested integer arrays, we need to extend it. Instead of `intArrayParser` parsing elements defined by the `intArrayElement` parser, it should parse either `intArrayElement` or itself.

Does that sound confusing or easy? The naive approach would be to write something like this:

```typescript
const intArrayParser = intArrayElement.or(intArrayParser)
	.sepBy(P.char(","))
	.between(P.char("["), P.char("]"));
```

But then, we'd get the following error from TypeScript:

```
Variable `intArrayParser` is used before being assigned.
```

The actual solution is not too far from that, but we need to introduce a new combinator, `lazy`, which allows us to define parsers that refer to themselves. As the name implies, we'll use a *lazily* initiated parser, that will be evaluated only after the initial variable initialization is complete:

```typescript
class P<T> {
    static lazy<T>(thunk: () => P<T>): P<T> {
		return new P<T>((input) => {
			return thunk().run(input);
		});
	}
}
```

With this combinator, we can now define our self-referential parser:

```typescript
const intArrayElement = integer.map(Number.parseInt);
const arraySeparator = P.char(",").between(P.whitespace().many());

type NestedElement = number | NestedElement[];
const intArrayParser: P<NestedElement[]> = intArrayElement
	.or(P.lazy(() => intArrayParser))
	.sepBy(arraySeparator)
	.between(P.whitespace().many())
	.between(P.char("["), P.char("]"));
```

Notice that we had to explicitly define the output type of `intArrayParser` to be `P<NestedElement[]>`, where `NestedElement` is a recursive type that can either be a number or an array of `NestedElement`s. This is necessary because with only type inference, intArrayParser depends on the type of `.or(P.lazy())`, which in turn depends on `intArrayParser` itself, leading to an unresolved type.

## Conclusion

In this article we extended the parser combinator library to handle more complex parsing scenarios. As you might have observed, it's easy to add new combinators to the library, many of which reuse existing combinators as buildings blocks. With more combinators, parser definitions become more expressive and easier to read.

The full code for this article can be found in [this gist](https://gist.github.com/prenaissance/6775fe1b52b4324fb1677d6185e0769a).