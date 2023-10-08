![npm](https://img.shields.io/npm/v/ts-poet)
[![CircleCI](https://circleci.com/gh/stephenh/ts-poet.svg?style=svg)](https://circleci.com/gh/stephenh/ts-poet)

Overview
========

ts-poet is a TypeScript code generator that is a small wrapper around [template literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals).

It lets you generate code basically as "just strings" (no need to re-create a low-level AST), but then also:

1. "Auto imports" only the types/symbols that are actually used in the output

   I.e. you use `const Foo = imp("Foo@foo")` to define the imports you need in your generated code, and ts-poet will create the import stanza at the top of the file.

   This can seem minor, but lets you break up/decompose your code generation logic, so that you can have multiple levels of helper methods/etc. that can return `code` template literals that embed both the code itself and the necessary type imports.

   And when the final file is generated, ts-poet will collect and emit the necessary imports.

2. Includes any other conditional output (see later), as/if needed.

3. Formats the output with [dprint-node](https://github.com/devongovett/dprint-node), an extremely fast formatter with "basically prettier-ish" output.

   ts-poet originally used prettier directly, but it became the bottleneck for multiple projects that use ts-poet for their code generation, even with caching to only format actually-changed output, so we switched to dprint and saw dramatic speedups.

Example
=======

Here's some example `HelloWorld` output generated by ts-poet:

```typescript
import { Observable } from "rxjs/Observable";

export class Greeter {
  private name: string;

  constructor(private name: string) {}

  greet(): Observable<string> {
    return Observable.from(`Hello $name`);
  }
}
```

And this is the code to generate it with ts-poet:

```typescript
import { code, imp } from "ts-poet";

// Use `imp` to declare an import that will conditionally auto-imported
const Observable = imp("@rxjs/Observable");

// Optionally create helper consts/methods to incrementally create output 
const greet = code`
  greet(): ${Observable}<string> {
    return ${Observable}.from(\`Hello $name\`);
  }
`;

// Combine all of the output (note no imports are at the top, they'll be auto-added)
const greeter = code`
export class Greeter {

  private name: string;

  constructor(private name: string) {
  }

  ${greet}
}
`;

// Generate the full output, with imports
const output = greeter.toString();
```


Import Specs
============

Given the primary goal of ts-poet is managing imports, there are several ways of specifying imports via the `imp` function:

* `imp("Observable@rxjs")` --> `import { Observable } from "rxjs"` (named import)
* `imp("Observable@./Api")` --> `import { Observable } from "./Api"`
* `imp("Observable:Obs@rxjs")` --> `import { Observable as Obs } from "rxjs"` (renamed import)
* `imp("t:Observable@rxjs")` --> `import type { Observable } from "rxjs"` (type import)
* `imp("t:Observable:Obs@rxjs")` --> `import type { Observable as Obs } from "rxjs"`
* `imp("api*./Api")` --> `import * as api from "./Api"` (namespace import)
* `imp("api=./Api")` --> `import api from "./Api"` (default import)
* `imp("describe+mocha")` --> `import "mocha"` (implicit import)

### Avoiding Import Conflicts

Sometimes code generation output may declare a symbol that conflicts with an imported type (usually for generic names like `Error`).

ts-poet will automatically detect and avoid conflicts if you tell it which symbols you're declaring, i.e.:

```typescript
const bar = imp('Bar@./bar');
const output = code`
  class ${def("Bar")} extends ${bar} {
     ...
  }
`;
```

Will result in the imported `Bar` symbol being remapped to `Bar1` in the output:

```typescript
import { Bar as Bar1 } from "./bar";
class Bar extends Bar1 {}
```

This is an admittedly contrived example for documentation purposes, but can be really useful when generating code against arbitrary / user-defined input (i.e. a schema that happens to uses a really common term).

# saveFiles

If you're generating multiple files, ts-poet provides a `saveFiles` command that can help with conditional output, i.e.:

```typescript
const author = {
  name: "Author.ts",
  contents: "class Author extends AuthorCodegen {}",
  overwrite: false,
};
const authorCodegen = {
   name: "AuthorCodegen.ts",
   contents: "class AuthorCodegen {}",
   overwrite: true,
};
await saveFiles({
 directory: "./src/entities",
 files: [author, authorCodegen]
});

```
# Conditional Output

Sometimes when generating larger, intricate output, you want to conditionally include helper methods. I.e. have a `convertTimestamps` function declared at the top of your module, but only actually include that function if some other part of the output actually uses timestamps (which might depend on the specific input/schema you're generating code against).

ts-poet supports this with a `conditionalOutput` method:

```typescript
const convertTimestamps = conditionalOutput(
  // The string to output at the usage site
  "convertTimestamps",
  // The code to conditionally output if convertTimestamps is used
  code`function convertTimestamps() { ...impl... }`,
);

const output = code`
  ${someSchema.map(f => {
    if (f.type === "timestamp") {
      // Using the convertTimestamps const marks it as used in our output
      return code`${convertTimestamps}(f)`;
    }
  })}
  // The .ifUsed result will be empty unless `convertTimestamps` has been marked has used
  ${convertTimestamps.ifUsed}
`;
```

And your output will have the `convertTimestamps` declaration only if one of the schema fields had a `timestamp` type.

This helps cut down on unnecessary output in the code, and compiler/IDE warnings like unused functions.

# Literals

If you want to add a literal value, you can use `literalOf` and `arrayOf`:

| code                              | output                    |
| --------------------------------- | ------------------------- |
| `let a = ${literalOf('foo')}`     | `let a = 'foo';`          |
| `let a = ${arrayOf(1, 2, 3)}`     | `let a = [1, 2, 3];`      |
| `let a = ${{foo: 'bar'}}`         | `let a = { foo: 'bar' };` |
| `` let a = ${{foo: code`bar`}} `` | `let a = { foo: bar };`   |

# esModuleInterop Support

Unfortunately some dependencies need different imports based on your project's `esModuleInterop` setting.

For example, with protobufjs, the `Reader` symbol is imported differently:

```typescript
// With esModuleInterop: true, need to use default import
// import m1 from "protobufjs"
// let r1: m1.Reader = ...
const r1 = imp("Reader@protobufjs")

// With esModuleInterop: false, need to use module star import
// import * as m1 from "protobufjs"
// let r1: m1.Reader = ...
const r2 = imp("Reader@protobufjs")
```

For these scenarios, you can use `forceDefaultImport` or `forceModuleImport`:

```ts
const Reader = imp("Reader@protobufjs")
const c = code`let r1: ${Reader} = ...`
const esModuleInterop = fromYourConfig();
console.log(c.toString(
  esModuleInterop
    ? { forceDefaultImport: ["protobufjs"] }
    : { forceModuleImport: ["protobufjs"] }
));
```

This is most useful for frameworks that generate code, and have to support downstream projects that might have either `esModuleInterop` setting.

Similarly, you can force the [import require](https://www.typescriptlang.org/docs/handbook/modules.html#export--and-import--require) syntax, to support projects that use the pre-default exports `export =` typing syntax, to export a single symbol:

```ts
const Long = imp("Long=long")
console.log(c.toString({ forceRequireImport: ["long"] }));
// outputs import Long = require("long")
```

History
=======

ts-poet was originally inspired by Square's [JavaPoet](https://github.com/square/javapoet) code generation DSL, which has a very "Java-esque" builder API of `addFunction`/`addProperty`/etc. that ts-poet copied in it's original v1/v2 releases.

JavaPoet's approach worked very well for the Java ecosystem, as it was providing three features:
 
1. nice formatting (historically code generation output has looked terrible; bad formatting, bad indentation, etc.)
2. nice multi-line string support, via `appendLine(...).appendLine(...)` style methods.
3. "auto organize imports", of collecting imported symbols across the entire compilation unit of output, and organizing/formatting them at the top of the output file.

However, in the JavaScript/TypeScript world we have prettier for formatting, and nice multi-line string support via template literals, so really the only value add that ts-poet needs to provide is the "auto organize imports", which is what the post-v2/3.0 API has been rewritten (and dramatically simplified as a result) to provide.

