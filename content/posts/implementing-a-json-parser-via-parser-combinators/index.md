+++
date = '2026-01-31T20:40:48+01:00'
draft = false
title = 'Implementing a Json Parser via Parser Combinators'
series = ['Parser Combinators with Typescript']
series_order = 3
+++

In the previous articles in the series we explored the basics of parser combinators and created a decently sized parser combinator library, capable of parsing various constructs of intermediate complexity. In this article, we'll combine all the pieces we've built so far to create a JSON parser.

## JSON Overview

Why JSON? Javascript already has `JSON.parse`!

JSON (JavaScript Object Notation) is a lightweight data-interchange format, with pretty simple rules. It's a great exercise for building a parser with a real-world application, but which is less complex than a full programming language.

The JSON format is so simple that its entire specification fits in a single page: [https://www.json.org/json-en.html](https://www.json.org/json-en.html). We'll use the handy diagrams on that page as a reference for our parser implementation.

## Building the JSON Parser

We'll try to adhere to the naming conventions used in the JSON specification. We'll start off with JSONs accepted "whitespace" characters, which are: space, tab, line feed, and carriage return, as shown in the diagram below:

{{<figure
    src="whitespace.png"
    alt="JSON Whitespace characters"
    caption="JSON Whitespace characters. [Source](https://www.json.org/json-en.html)"
    class="bg-white p-1"
>}}

The previously defined `P.whitespace()` parser already handles the characters defined as whitespace in JSON, so we can use to to create an "any amount of whitespace" parser:

```typescript
const ws = P.whitespace().many();
```

The following parser for number literals will be the most verbose one, although the rules are quite intuitive:

{{<figure
    src="number.png"
    alt="JSON Number literal"
    caption="JSON Number literal. [Source](https://www.json.org/json-en.html)"
    class="bg-white p-1"
>}}

Here, although not necessarily needed, it would be helpful to implement a `P.optional` combinator, for implementing the optional "-" prefix for negative numbers:

```typescript
class P<T> {
    optional(): P<T | null> {
		return new P<T | null>((input) => {
			const result = this.run(input);
			if (result.success) {
				return result;
			}
			return { success: true, value: null, remaining: input };
		});
	}
}
```

### Number Literal Parser

Next up comes the integer part of the number literal. We have already implemented a parser for this in the previous article, but let's try a slightly different approach here that's closer to the notation used in the JSON specification. Let's also map the resulting string of digits to a number right away:

```typescript
const onenine = P.dictionary("123456789");
const digit = P.char("0").or(onenine);
const digits = digit.many().map((chars) => chars.join(""));
const integer = P.seq([onenine, digits])
	.map(([first, rest]) => first + rest)
	.or(P.char("0"))
	.map(Number.parseInt);
```

Next comes the fractional part, which is optional:

```typescript
const fraction = P.seq([P.char("."), digit.many1()])
	.map(([, fracDigits]) => `0.${fracDigits.join("")}`)
	.map(Number.parseFloat)
	.optional();
```

Finally, the exponent part, which is also optional:

```typescript
const sign = P.char("-").or(P.char("+")).optional();
const exponent = P.seq([P.char("e").or(P.char("E")), sign, digit.many1()])
	.map(([, sign, digits]) => {
		const isPositive = sign !== "-";
		const exponentValue = Number.parseInt(digits.join(""), 10);
		return isPositive ? exponentValue : -exponentValue;
	})
	.optional();
```

This all comes together to form the final number parser:

```typescript
const number = P.seq([integer, fraction, exponent]).map(
	([intPart, fracPart, expPart]) => {
		let value = intPart + (fracPart ?? 0);
		if (expPart !== null) {
			value *= 10 ** expPart;
		}
		return value;
	},
);
```

You can give it a try with tricky inputs such as `-0.123e-45`.

### String Literal Parser

This part would be easy if it weren't for the escape sequences, such as `\"`, or if you wanted to encode an emoji you'd use something like `\ud83d\ude00`. Well, at least in lower level language that work with bytes instead of graphemes. Let's start with the easy part: parsing unescaped characters. According to the JSON specification, these are all characters except for `"` and `\`, and control characters (U+0000 through U+001F).

{{<figure
    src="string.png"
    alt="JSON String literal"
    caption="JSON String literal. [Source](https://www.json.org/json-en.html)"
    class="bg-white p-1"
>}}

We can implement this in two ways:
1. Adding a negative lookahead parser to exclude the unwanted characters.
2. Adding an unicode range parser to include only the wanted character ranges.

Here, we'll go with the first approach, implementing an `any` parser that accepts any single character, and an `except` combinator that excludes the given parser:

```typescript
class P<T> {
    static any(): P<string> {
		return new P<string>((input) => {
			if (input.index < input.text.length) {
				const char = input.text[input.index];
				return {
					success: true,
					value: char,
					remaining: { text: input.text, index: input.index + 1 },
				};
			}
			return { success: false };
		});
	}

    except<U>(other: P<U>): P<T> {
		return new P<T>((input) => {
			const otherResult = other.run(input);
			if (otherResult.success) {
				return { success: false };
			}
			return this.run(input);
		});
	}
}
```