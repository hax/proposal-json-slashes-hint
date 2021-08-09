# JSON `\//` Hint

## Problems

```js
JSON.stringify({x: 1234567890n}) // throw!
```

You can custom it

```js
BigInt.prototype.toJSON = function () { return this.toString() }
JSON.stringify({x: 1234567890n}) // {"x": "1234567890"}
```

It works, but your application need to know `x` is a bigint to deserialize
it correctly. It's likely you will hard code your schema logic.

A much serious problem is, in big project, you or your workmate will introduce
many libraries directly or indirectly (node_modules black hole :-P ), some lib
may also want to add their "better" version to `BigInt.prototype`, which may
override yours, or yours override theirs :( , both can cause bugs, and such
bugs coud be very hard to discover and locate.

```js
// "better" version
BigInt.prototype.toJSON = function () {
	return {
		"type": "BigInt",
		data: convertBigIntToInt16Array(this),
	}
}
JSON.stringify({x: 1234567890n}) // {"x":{"type":"BigInt",data:[722,18838]}}
```

Even one day most of us all adopt this better version, there is still a question,
if you happened have a normal object has the same shape, how can we differatiate
them?

Some other similar cases:

```js
// schemaless string
JSON.stringify({x: new Date(0)}) // "1970-01-01T00:00:00.000Z"

// schemaless object
JSON.stringify(Buffer.from('hello')) // {"type":"Buffer","data":[104,101,108,108,111]}
```

One possible solution is use some special keys like `"@type"` instead of too
common `"type"`, and/or use escaped form for same shape. (TODO: add escaped example)

But `"@type"` already be used by libraries or even standards like JSON-LD.
`"%type"` seems much rarely used, but actually also already
[be used in some code base](https://github.com/search?q=%22%25TYPE%22+extension%3Ajson&type=Code), it's hard to guarantee a syntax is "special" enough and never be used in any
json file in the world.

And escaped form normally have bad readability for people, and have the risk
of inflating quickly in the mistaken of incompatible implementations combination.

## Constraints

- We can't change/extend the syntax of JSON! Even the simplest comments!
- Convention based solution is questionable, because the community of JS is very 
	large and divergent, the community of JSON is even much large and divergent.

## Proposal

```js
JSON.stringify({x: 1n}, magicOptionsToEnableBigIntAndOtherDatatypes, '\t')
```

generate:

```json
{
	"x": "\//@JS:42n"
}
```

and could be deserialize back to `{x: 1n}` by `JSON.parse` automatically
if parser understand the `\//` hint.

If parser don't understand the hint, it just deserialize it to `{x: "//@JS:42n"}`.

## `\//` hint

`\//` hint: an escaped `/` followed by a normal, unescaped `/`

JSON parsers (and people) could easily recognize this.

It's valid JSON string, normally tools don't escape `/`, or escape all `/`
(PHP `json_encode()` by default), or only escape `</` case for `</script>`.

Some tools use similar trick, eg. https://weblogs.asp.net/bleroy/dates-and-json.

As far as I can see, it's hopefully no implementation in any langauges or libraries would 
generate both escaped `/` and unescaped `/` in single JSON string value. 
So we can use it as a hint!

## Process mode

- A meta string is a string contains `\//` hint
- A meta key is a key in meta string

There are several possible options if parsers meet a meta string/key
and don't understand it:

- ignore it (deserialize it as normal strings/keys)
- report error
- drop the meta string or the meta key and corresponding value
- allow user provide custom subroutine (it could be a special hint hook, or
	a general mechnism like https://github.com/tc39/proposal-json-parse-with-source)
- combinations of above

## Possible proposals based on `\//` hint

```json5
// Not real valid JSON because JSON hate comments :)
{
	// unsuppported in JSON
	"undefined": "\//JS:undefined",
	"NaN": "\//JS:NaN",
	"-NaN": "\//JS:-NaN", // see signbit issue

	// would come from source 1e999, so allow serialize it accurately
	"+Infinity": "\//JS:Infinity",
	"-Infinity": "\//JS:-Infinity",
	"null": "\//JS:null",

	// common cases
	"bigint": "\//JS:42n",
	"legacyDate": "\//JS:Date!2020-01-01T00:00:00.0Z",
	"date": "\//JS:Temporal.Date!2020-01-01",
	"buffer": "\//JS:ArrayBuffer!SGVsbG8=",

	// Node.js Buffer style
	"buffer": {
		"\// type": "node:Buffer",
		"\// data": [1, 2, 3]
	},

	// key annotation style
	"int8array": [1, 2, 3],
	"int8array \// type": "JS:Int8Array",
	
	"tuple": [1, 2, 3],
	"tuple \// type": "JS:tuple" // #[1, 2, 3]
}
```

## Compliance requirements of JSON implementations

This part seems beyond the scope of TC39, maybe JSON RFC docucment?

1. Implementations should not generate a json string value with both unescaped
	`/` and escaped `/` except `\//` hint
0. Implementations should only generate one `\//` hint in one string value
0. `\//FOO:...` syntax is for standard bodys, for example `\//JS:` is for TC39
0. `\//x-FOO:...` syntax could be used for implementation-defined extensions
0. All other forms (like `\//type`, `\//data`) are reserved for cross-platform
	JSON standards
0. Spaces are allowed after `\//`, aka `"\// type` is just same as `"\//type"`
