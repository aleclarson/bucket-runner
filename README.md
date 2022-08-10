bucket-runner [![Build Status](https://travis-ci.org/spotify/bucket-runner.svg?branch=master)](https://travis-ci.org/spotify/bucket-runner) [![Coverage Status](https://coveralls.io/repos/github/spotify/bucket-runner/badge.svg?branch=master)](https://coveralls.io/github/spotify/bucket-runner?branch=master)
=============

Run a command in parallel, distributing the input files to each process and buffer the output to prevent interleaving. Kind of like xargs but with control over output.

A common example would be to easily run [mocha](https://mochajs.org/) tests in parallel without any code changes:

```
$ bucket-runner --partition-size 5 tests/* -- mocha
```

Typically mocha runs tests serially, but if your tests are in separate files you can run each file at the same time!

Output is buffered (and untouched), so it will still be linear and whole when tests finish.

An example using the fixtures in this repo:

```sh
# using mocha directly
$ time node_modules/.bin/mocha fixtures/tests/**/*.js
...
real  0m4.200s
user  0m0.163s
sys   0m0.035s
```

```sh
# using bucket-runner to execute mocha
$ time bin/cli.js fixtures/tests/**/*.js --partition-size 1 -- node_modules/.bin/mocha
...
real  0m1.403s
user  0m0.936s
sys   0m0.198s
```

Getting Started
---------------

Global utility:

```sh
$ npm install -g bucket-runner
$ bucket-runner --help
```

Local utility:

```sh
$ npm install bucket-runner
$ $(npm bin)/bucket-runner --help
```

Can also be `require`ed for [script usage](#api-usage):

```sh
$ npm install bucket-runner
$ node
> var runner = require('bucket-runner');
```


CLI Usage
---------

```sh
$ bucket-runner [options] [files|globs...] -- [cmd] [{files}, {partition}]
```

`files|globs` can either be many files globbed through shell expansion, or a quoted glob. If quoted, the glob is passed into [glob](https://www.npmjs.com/package/glob) to expand into a list of files.

If the explicit token `{files}` is contained within the `[cmd]`, then the resolved globbed files will be placed at that point in the command string.

Example:

```sh
$ ls tests
page1.spec.js page2.spec.js page3.spec.js lib1.spec.js lib2.spec.js

$ bucket-runner tests -- echo {files} are the files
page1.spec.js page2.spec.js are the files
page3.spec.js lib1.spec.js are the files
lib2.spec.js are the files

$ bucket-runner tests -- echo are the files
are the files page1.spec.js page2.spec.js
are the files page3.spec.js lib1.spec.js
are the files lib2.spec.js
```

Options
-------

### `--concurrency [count]` (default cpus * 4)

The number of simultaneous processes to use when executing.

### `--partition-size [size]` (default: 1)

Use `[size]` as the batch size for grouping files. Some examples:

```sh
# will use one echo command
$ bucket-runner --partition-size 5 f1.js f2.js f3.js f4.js f5.js -- echo
f1.js f2.js f3.js f4.js f5.js
```

```sh
# will use five echo commands (the default behavior)
$ bucket-runner --partition-size 1 f1.js f2.js f3.js f4.js f5.js -- echo
f4.js
f3.js
f2.js
f1.js
f5.js
```

### `--partition-regex [regex]`

Use `[regex]` to group the list of files into processes. The regex is matched using the absolute path to the file. If a capture group is specified, it can be accessed via a special command substitution token `{partition}` (including the `{}`). An example scenario: you want to create coverage reports, but your coverage framework needs a unique name for each process creating output:

```
$ ls tests
page1.spec.js page2.spec.js page3.spec.js lib1.spec.js lib2.spec.js
$ bucket-runner --partition-regex '(page|lib)\d' tests/* -- istanbul cover _mocha --dir coverage/{partition} --
```

In the above example the coverage destinations would be named `coverage/page` and `coverage/lib` since that was the result of the first-defined capture group in the regex.

NOTE: If `--partition-regex` is used, `partition-size` is ignored as the regex will potentially create imbalanced partitions.

### `--resolve-files`

Enable file existence checking.

By default, bucket-runner runs your command without any path validation, so even directories are passed through. This option lets you only run the command for actual files.

### `--continue-on-error`

Continue processing commands, even if one of the parallel processes emits an error. Default is to halt and exit if an error is emitted.

### `--stream-output`

Do not buffer output, and instead stream directly to stdio.

One of bucket-runner's benefits is that it buffers output to ensure that output from multiple processes is not interwoven. This enables file redirection without having to use temporary files. But sometimes the output might be too large to buffer, or the speed hit too great.


API Usage
---------

```js
var runner = require('bucket-runner');

runner(['file1.js', 'glob1/*/**.js'], '_mocha', {
  concurrency: 2,
  'partition-size': 2
}, function (err) {
  if (err) throw err;
});
```

Contributing
------------

Integration and unit tests are executed using:

```sh
$ npm run test
```

The integration tests use bash, and are untested on Windows. Contributing a Windows version of the test would be greatly appreciated!

If adding a new option, be sure to add descriptions to both this README and [bin/usage.txt](bin/usage.txt) for command-line help.

This project adheres to the [Open Code of Conduct][code-of-conduct]. By participating, you are expected to honor this code.

[code-of-conduct]: https://github.com/spotify/code-of-conduct/blob/master/code-of-conduct.md

Releasing
---------

```sh
$ npm version [major|minor|patch]
$ git push origin HEAD ---tags
$ npm publish
```

License
-------

[Apache 2.0](LICENSE)
