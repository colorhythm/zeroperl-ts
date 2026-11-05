> **Fork notice & thanks** — this is Colorhythm's fork of
> [**zeroperl-ts** by Andrew Sampson (6over3)](https://github.com/6over3/zeroperl-ts).
> The TypeScript embedding of Perl-in-WebAssembly is his work; our
> changes serve the `@colorhythm/exiftool-wasm` toolchain. Thank you,
> Andrew.

# zeroperl-ts

Perl 5 compiled to WebAssembly. Run Perl scripts in the browser or other JavaScript environments without installing Perl.

Built on [zeroperl](https://github.com/colorhythm/zeroperl).

## Role in the Colorhythm forks

This repository provides the standalone TypeScript embedding and a checked-in
development WASM binary. It is not the release controller for ExifTool.
[@colorhythm/exiftool-wasm](https://www.npmjs.com/package/@colorhythm/exiftool-wasm)
vendors its adapter, including separate Worker-loading and byte-safe output
fixes, and pairs it with a runtime built from a recorded zeroperl revision.
Updating this checkout's binary does not update a published ExifTool package.

## Features

- Perl 5 with loader branches for browser Window, Node.js, Deno, and Bun
- Virtual filesystem for script and data files
- Bidirectional data exchange between JavaScript and Perl
- Register JavaScript functions callable from Perl
- Call Perl functions from JavaScript
- Environment variable support
- Output capture (stdout/stderr)
- TypeScript type definitions included

## Installation

The declared package name is `@colorhythm/zeroperl-ts`, but this fork is not
currently published under that name on the public npm registry. npm install and
public CDN URLs for that name do not work. For metadata extraction, install the
published `@colorhythm/exiftool-wasm` package instead.

For standalone development, build and pack this checkout:

```bash
git clone https://github.com/colorhythm/zeroperl-ts.git
cd zeroperl-ts
npm install
bun test
bun run build
npm pack --ignore-scripts
```

Install the resulting `.tgz` by its local path in a consuming app. The examples
below assume that local package is installed. The repository currently does not
track a dependency lockfile; these development installs resolve the declared
dependency ranges.

## Quick Start

```typescript
import { ZeroPerl } from '@colorhythm/zeroperl-ts';

const perl = await ZeroPerl.create({
    stdout: (data) => console.log(typeof data === 'string' ? data : new TextDecoder().decode(data)),
    stderr: (data) => console.error(typeof data === 'string' ? data : new TextDecoder().decode(data)),
});
try {
    const result = await perl.eval('print "Hello, World!\\n"');
    perl.flush();
    if (!result.success) throw new Error(result.error);
} finally {
    perl.dispose();
}
```

## Output Buffering

Without stdout/stderr callbacks, this wrapper discards that output. Other
snippets focus on individual APIs; add the callbacks above when output should
be visible. Perl also buffers output. Choose one approach:

**Option 1: Call `flush()` after printing**
```typescript
await perl.eval('print "Hello!\\n"');
perl.flush();
```

**Option 2: Enable autoflush in Perl**
```typescript
await perl.eval(`
  $| = 1;  # Enable autoflush
  print "Hello!\\n";
`);
```

## Basic Usage

### Evaluating Perl Code

```typescript
import { ZeroPerl } from '@colorhythm/zeroperl-ts';

const perl = await ZeroPerl.create();

const result = await perl.eval(`
  $| = 1;
  my $x = 42;
  print "The answer is $x\\n";
`);

if (!result.success) {
  console.error('Error:', result.error);
}

perl.dispose();
```

### Capturing Output

```typescript
let output = '';

const perl = await ZeroPerl.create({
  stdout: (data) => {
    output += typeof data === 'string' ? data : new TextDecoder().decode(data);
  }
});

await perl.eval(`
  $| = 1;
  print "Line 1\\n";
  print "Line 2\\n";
`);

console.log(output);
// Output:
// Line 1
// Line 2

perl.dispose();
```

### Exchanging Data

```typescript
const perl = await ZeroPerl.create();

// JavaScript to Perl
perl.setVariable('name', 'Alice');
perl.setVariable('age', 30);

await perl.eval(`
  $| = 1;
  print "Name: $name\\n";
  print "Age: $age\\n";
`);

// Perl to JavaScript
await perl.eval('$result = 2 + 2');
const result = perl.getVariable('result');
console.log(result.toInt()); // 4

result.dispose();
perl.dispose();
```

### Working with Arrays and Hashes

```typescript
const perl = await ZeroPerl.create();

// Create array
const arr = perl.createArray([1, 2, 3, 'hello']);
perl.setVariable('myarray', arr.toValue());

// Create hash
const hash = perl.createHash({
  name: 'Alice',
  age: 30,
  active: true
});
perl.setVariable('user', hash.toValue());

await perl.eval(`
  $| = 1;
  print "Array length: ", scalar(@$myarray), "\\n";
  print "User: $user->{name}, Age: $user->{age}\\n";
`);

// Convert back to JavaScript
const jsArray = arr.project(); // [1, 2, 3, 'hello']
const jsObject = hash.project(); // { name: 'Alice', age: 30, active: true }

arr.dispose();
hash.dispose();
perl.dispose();
```

### Command-Line Arguments

```typescript
const perl = await ZeroPerl.create();

await perl.eval(`
  $| = 1;
  print "Arguments: @ARGV\\n";
  foreach my $arg (@ARGV) {
    print "  $arg\\n";
  }
`, ['foo', 'bar', 'baz']);

perl.dispose();
```

## Working with Files

### Creating a Virtual Filesystem

```typescript
import { ZeroPerl, MemoryFileSystem } from '@colorhythm/zeroperl-ts';

const fs = new MemoryFileSystem({ "/": "" });

fs.addFile("/data.txt", "Hello from a file!");
fs.addFile("/script.pl", `
  $| = 1;
  open my $fh, '<', '/data.txt' or die $!;
  while (my $line = <$fh>) {
    print "Read: $line";
  }
  close $fh;
`);

const perl = await ZeroPerl.create({ fileSystem: fs });

await perl.runFile('/script.pl');

perl.dispose();
```

### Running Scripts with Arguments

```typescript
const fs = new MemoryFileSystem({ "/": "" });
fs.addFile("/greet.pl", `
  $| = 1;
  my ($name, $greeting) = @ARGV;
  print "$greeting, $name!\\n";
`);

const perl = await ZeroPerl.create({ fileSystem: fs });

await perl.runFile('/greet.pl', ['Alice', 'Hello']);
// Output: Hello, Alice!

perl.dispose();
```

### Reading and Writing Files

```typescript
const fs = new MemoryFileSystem({ "/": "" });

const perl = await ZeroPerl.create({ fileSystem: fs });

// Write from Perl
await perl.eval(`
  $| = 1;
  open my $fh, '>', '/output.txt' or die $!;
  print $fh "Generated content\\n";
  close $fh;
  print "File written!\\n";
`);

// Read from JavaScript
const outputFile = fs.lookup('/output.txt');
if (outputFile?.type === 'file') {
    const bytes = outputFile.content instanceof Blob
        ? new Uint8Array(await outputFile.content.arrayBuffer())
        : outputFile.content;
    console.log(new TextDecoder().decode(bytes)); // "Generated content\n"
}

perl.dispose();
```

## Advanced Usage

### Registering JavaScript Functions

Register JavaScript functions that can be called from Perl:

```typescript
const perl = await ZeroPerl.create();

perl.registerFunction('add', (a, b) => {
  const x = a.toInt();
  const y = b.toInt();
  return perl.createInt(x + y);
});

await perl.eval(`
  $| = 1;
  my $sum = add(10, 32);
  print "Sum: $sum\\n";
`);

perl.dispose();
```

### Registering JavaScript Methods

```typescript
const perl = await ZeroPerl.create();

perl.registerMethod('Math', 'square', (x) => {
  const num = x.toInt();
  return perl.createInt(num * num);
});

await perl.eval(`
  $| = 1;
  my $result = Math::square(7);
  print "Square: $result\\n";
`);

perl.dispose();
```

### Calling Perl Functions from JavaScript

```typescript
const perl = await ZeroPerl.create();

await perl.eval(`
  sub greet {
    my ($name) = @_;
    return "Hello, $name!";
  }

  sub get_values {
    return (1, 2, 3);
  }
`);

// Scalar context (single return value)
const arg = perl.createString("Alice");
const greeting = await perl.call("greet", [arg], "scalar");
console.log(greeting?.toString()); // "Hello, Alice!"

// List context (multiple return values)
const values = await perl.call("get_values", [], "list");
console.log(values.map(v => v.toInt())); // [1, 2, 3]

// Void context (no return value)
await perl.call("some_sub", [], "void");

arg.dispose();
greeting?.dispose();
for (const v of values) v.dispose();
perl.dispose();
```

### Error Handling

```typescript
const perl = await ZeroPerl.create();

const result = await perl.eval(`
  die "Something went wrong!";
`);

if (!result.success) {
  console.log('Exit code:', result.exitCode);
  console.log('Error:', result.error);
}

// Get error directly
const error = perl.getLastError();
console.log(error); // "Something went wrong! at ..."

// Clear error
perl.clearError();

perl.dispose();
```

### Environment Variables

```typescript
const perl = await ZeroPerl.create({
  env: {
    API_KEY: 'secret123',
    DEBUG: 'true'
  }
});

await perl.eval(`
  $| = 1;
  print "API Key: $ENV{API_KEY}\\n";
  print "Debug: $ENV{DEBUG}\\n";
`);

perl.dispose();
```

### Resetting State

```typescript
const perl = await ZeroPerl.create();

await perl.eval('$counter = 1');
const val1 = perl.getVariable('counter');
console.log(val1?.toInt()); // 1

await perl.reset();

const val2 = perl.getVariable('counter');
console.log(val2); // null

val1?.dispose();
perl.dispose();
```

### Progressive Output

```typescript
const perl = await ZeroPerl.create({
  stdout: (data) => process.stdout.write(data)
});

for (let i = 0; i < 5; i++) {
  await perl.eval('print "."');
  perl.flush();
  await new Promise(r => setTimeout(r, 500));
}

perl.dispose();
```

## Browser Usage

Serve the WASM file from your app instead of a public CDN for this unpublished
fork. After installing your locally built tarball, copy its ESM runtime asset:

```bash
mkdir -p public
cp node_modules/@colorhythm/zeroperl-ts/dist/esm/zeroperl.wasm public/zeroperl.wasm
```

Bundle the JavaScript entry normally, and supply the URL where your app serves
that matching WASM file. For example, when it is served next to the HTML page:

```typescript
import { ZeroPerl } from '@colorhythm/zeroperl-ts';

const wasmUrl = new URL('./zeroperl.wasm', document.baseURI);
const perl = await ZeroPerl.create({
    fetch: () => fetch(wasmUrl),
    stdout: (data) => console.log(data),
});
try {
    const result = await perl.eval('$| = 1; print "Hello from Perl!";');
    if (!result.success) throw new Error(result.error);
} finally {
    perl.dispose();
}
```

Adjust the URL for your app's base path. Keep the JavaScript and WASM from the
same build; copying a different runtime is not an adapter compatibility check.

The current standalone loader identifies a browser through both `window` and
`document`. A browser Worker falls into its filesystem branch, and the custom
fetch callback does not override that branch. Worker loading requires an adapter
fix; do not assume the Worker support in exiftool-wasm is present here.

The standalone stdout/stderr path is text-oriented and does not expose an option
to enable the WASI layer's raw-buffer output. A callback type that includes
Uint8Array does not establish byte-safe binary output. Use exiftool-wasm's
`binary: true` API for its supported binary metadata extraction.

## API Reference

### `ZeroPerl.create(options?)`

Create a new Perl interpreter instance.

**Options:**
- `env` - Environment variables (Record<string, string>)
- `fileSystem` - Virtual filesystem (MemoryFileSystem)
- `stdout` - stdout callback ((data: string | Uint8Array) => void)
- `stderr` - stderr callback ((data: string | Uint8Array) => void)
- `fetch` - Custom fetch for WASM loading in browser Window contexts

```typescript
const perl = await ZeroPerl.create({
  env: { KEY: 'value' },
  fileSystem: fs,
  stdout: (data) => console.log(data),
  stderr: (data) => console.error(data)
});
```

### `perl.eval(code, args?)`

Evaluate Perl code. Returns `{ success: boolean, error?: string, exitCode: number }`.

```typescript
const result = await perl.eval('print "Hello\\n"', ['arg1', 'arg2']);
```

### `perl.runFile(path, args?)`

Run a Perl script from the virtual filesystem.

```typescript
await perl.runFile('/script.pl', ['arg1', 'arg2']);
```

### `perl.createInt(value)`, `perl.createDouble(value)`, `perl.createString(value)`, `perl.createBool(value)`, `perl.createUndef()`

Create Perl values. Returns `PerlValue`.

```typescript
const num = perl.createInt(42);
const str = perl.createString("hello");
const bool = perl.createBool(true);
```

### `perl.createArray(values?)`, `perl.createHash(object?)`

Create Perl arrays and hashes. Returns `PerlArray` or `PerlHash`.

```typescript
const arr = perl.createArray([1, 2, 3]);
const hash = perl.createHash({ key: 'value' });
```

### `perl.toPerlValue(value)`

Convert JavaScript value to Perl. Handles primitives, arrays, and objects.

```typescript
const perlVal = perl.toPerlValue({ name: 'Alice', age: 30 });
```

### `perl.setVariable(name, value)`, `perl.getVariable(name)`

Set and get scalar variables. Variable names should not include the `$` prefix.

```typescript
perl.setVariable('x', 42);
const x = perl.getVariable('x');
console.log(x?.toInt()); // 42
```

### `perl.getArrayVariable(name)`, `perl.getHashVariable(name)`

Get array and hash variables. Returns `PerlArray` or `PerlHash`.

```typescript
const arr = perl.getArrayVariable('myarray');
const hash = perl.getHashVariable('myhash');
```

### `perl.registerFunction(name, fn)`

Register a JavaScript function callable from Perl.

```typescript
perl.registerFunction('add', (a, b) => {
  const x = a.toInt();
  const y = b.toInt();
  return perl.createInt(x + y);
});
```

### `perl.registerMethod(packageName, methodName, fn)`

Register a JavaScript method callable from Perl.

```typescript
perl.registerMethod('Math', 'square', (x) => {
  const num = x.toInt();
  return perl.createInt(num * num);
});
```

### `perl.call(name, args, context?)`

Call a Perl function. Context can be `"void"`, `"scalar"`, or `"list"`.

```typescript
const result = await perl.call('my_sub', [arg1, arg2], 'scalar');
const results = await perl.call('my_sub', [], 'list');
await perl.call('my_sub', [], 'void');
```

### `perl.flush()`

Flush output buffers. Required if autoflush (`$| = 1`) is not set.

```typescript
await perl.eval('print "text"');
perl.flush();
```

### `perl.reset()`

Reset interpreter to clean state. Clears all variables.

```typescript
await perl.reset();
```

### `perl.getLastError()`, `perl.clearError()`

Get and clear the Perl error state (`$@`).

```typescript
const error = perl.getLastError();
perl.clearError();
```

### `perl.isInitialized()`, `perl.canEvaluate()`

Check interpreter state.

```typescript
const ready = perl.isInitialized() && perl.canEvaluate();
```

### `perl.dispose()`, `perl.shutdown()`

Free resources. Use `dispose()` for normal cleanup, `shutdown()` for complete termination.

```typescript
perl.dispose();
// or
perl.shutdown();
```

### PerlValue Methods

- `toInt()` - Convert to 32-bit integer
- `toDouble()` - Convert to double-precision float
- `toString()` - Convert to UTF-8 string
- `toBoolean()` - Convert to boolean (Perl truth test)
- `isUndef()` - Check if undefined
- `isRef()` - Check if reference
- `getType()` - Get Perl type
- `project()` - Convert to JavaScript primitive
- `createRef()` - Create reference
- `deref()` - Dereference value
- `dispose()` - Free memory

### PerlArray Methods

- `push(value)` - Add to end
- `pop()` - Remove from end
- `get(index)` - Get value at index
- `set(index, value)` - Set value at index
- `getLength()` - Get array length
- `clear()` - Remove all elements
- `toValue()` - Convert to PerlValue (array reference)
- `project()` - Convert to JavaScript array
- `[Symbol.iterator]()` - Iterate over values
- `dispose()` - Free memory

### PerlHash Methods

- `set(key, value)` - Set key-value pair
- `get(key)` - Get value by key
- `has(key)` - Check if key exists
- `delete(key)` - Delete key
- `clear()` - Remove all entries
- `toValue()` - Convert to PerlValue (hash reference)
- `project()` - Convert to JavaScript object
- `entries()` - Iterate over key-value pairs
- `keys()` - Iterate over keys
- `values()` - Iterate over values
- `dispose()` - Free memory

## Examples

### Processing JSON

```typescript
import { ZeroPerl, MemoryFileSystem } from '@colorhythm/zeroperl-ts';

const fs = new MemoryFileSystem({ "/": "" });
fs.addFile("/data.json", JSON.stringify({ users: ['Alice', 'Bob'] }));
fs.addFile("/process.pl", `
  $| = 1;
  use strict;
  use warnings;
  
  open my $fh, '<', '/data.json' or die $!;
  my $json = do { local $/; <$fh> };
  close $fh;
  
  print "Processing: $json\\n";
`);

const perl = await ZeroPerl.create({ fileSystem: fs });
await perl.runFile('/process.pl');
perl.dispose();
```

### Interactive REPL

```typescript
const perl = await ZeroPerl.create({
  stdout: (data) => console.log(data)
});

await perl.eval('$| = 1');

await perl.eval('$x = 10');
await perl.eval('print "$x\\n"');
await perl.eval('$x *= 2');
await perl.eval('print "$x\\n"');

perl.dispose();
```

### Complex Data Structures

```typescript
const perl = await ZeroPerl.create();

perl.setVariable('config', {
  server: {
    host: 'localhost',
    port: 8080
  },
  features: ['auth', 'logging']
});

await perl.eval(`
  $| = 1;
  print "Host: $config->{server}{host}\\n";
  print "Port: $config->{server}{port}\\n";
  print "Features: @{$config->{features}}\\n";
`);

perl.dispose();
```

## Development

```bash
npm install
bun test
bun run build
```

The companion tests execute under Bun. Some cases are explicitly skipped, so a
green run does not establish those capabilities or a Node/Deno/browser test
matrix. The separate exiftool-wasm release gate tests its vendored adapter in
Node, Chrome, and Playwright WebKit; that coverage does not transfer wholesale
to this standalone package.

## License

Apache-2.0

## About

ZeroPerl compiles Perl 5 to WebAssembly using a WASI-compliant implementation. This package provides a TypeScript/JavaScript API for the [ZeroPerl](https://github.com/colorhythm/zeroperl) WASM module.
