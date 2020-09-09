# res-log

[![npm version](https://img.shields.io/npm/v/res-log.svg?style=flat-square)](https://www.npmjs.com/package/res-log)
[![license](https://img.shields.io/npm/l/res-log.svg?style=flat-square)](https://www.npmjs.com/package/res-log)

Logging implementation for [ReScript](https://rescript-lang.org).

![res-log](./.assets/example.png)

## Features
- Zero runtime in production builds.
- Multiple logging levels.
- Customizable verbosity.
- `[@log]` helper.
- `ReasonReact` integration.
- Custom loggers.
- Logging in libraries.

## Installation
Get the package:

```shell
# yarn
yarn add res-log
# or npm
npm install --save res-log
```

Then add it to `bsconfig.json`:

```json
"bs-dependencies": [
  "res-log"
],
"ppx-flags": ["res-log/ppx"]
```

## Usage
There are 5 log levels:
- `trace`
- `debug`
- `info`
- `warn`
- `error`

You can log message of specific level using appropriate macros:

```reason
// ReScript
%log.info("Info message")
%log.error("Error message")

// Reason
[%log.info "Info message"];
[%log.error "Error message"];
```

### Additional data
You can add data to log entry like this:

```reason
// ReScript
%log.info(
  "Info message"
  ("Foo", 42)
)
%log.info(
  "Info message"
  ("Foo", {x: 42})
  ("Bar", [1, 2, 3])
)

// Reason
[%log.info "Info message"; ("Foo", 42)];
[%log.info
  "Info message";
  ("Foo", {x: 42});
  ("Bar", [1, 2, 3]);
];
```

Currently, logger can accept up to 7 additional entries.

### Verbosity customization
Output verbosity can be customized by providing specific **log level** and/or **code locations**.

#### Log level
You can set maximum log level via environment variable `RES_LOG`.

Let's say you want to log only warnings and errors. To make it happen, run your build like this:

```shell
RES_LOG=warn bsb -clean-world -make-world
```

Available `RES_LOG` values:
- `*`: log everything
- `trace`: basically, the same as `*`
- `debug`: log everything except `trace` level messages
- `info`: log everything except `trace` & `debug` level messages
- `warn`: log `warn` & `error` messages only
- `error`: log `error` messages only
- `off`: don't log anything

If `RES_LOG` is set to `off`, nothing will be logged and none of the log entries will appear in your JS assets.

In case if `RES_LOG` environment variable is not set, log level `warn` will be used.

Also, see [Usage in libraries](#usage-in-libraries).

#### Code location
If you want to focus on logging from specific part(s) of your code, you can use `RES_LOG_ONLY` environment variable.

For example, if you want to see logs only from module `Test`, run the build as following:

```shell
RES_LOG_ONLY=Test bsb -clean-world -make-world
```

You can pass submodules and functions to it as well. If you want to log from multiple locations, separate them by `,`.

Consider the following source:

```reason
// Test.res
%log.warn("Top level message")

module Submodule1 = {
  %log.warn("Message from Submodule1")
}

module Submodule2 = {
  %log.warn("Message from Submodule2")

  let fn = () => %log.warn("Message from function within Submodule2")
  fn()
}
```

Here is what will be logged with different build configurations:

```
# build
RES_LOG_ONLY=Test bsb -clean-world -make-world

# output
WARNING  [Test] Top level message
WARNING  [Test.Submodule1] Message from Submodule1
WARNING  [Test.Submodule2] Message from Submodule2
WARNING  [Test.Submodule2.fn] Message from function within Submodule2

# build
RES_LOG_ONLY=Test.Submodule2 bsb -clean-world -make-world

# output
WARNING  [Test.Submodule2] Message from Submodule2
WARNING  [Test.Submodule2.fn] Message from function within Submodule2

# build
RES_LOG_ONLY=Test.Submodule1,Test.Submodule2.fn bsb -clean-world -make-world

# output
WARNING  [Test.Submodule1] Message from Submodule1
WARNING  [Test.Submodule2.fn] Message from function within Submodule2
```

### `[@log]` helper
This helper can be placed in front of any `switch` expression with constructor patterns and it will inject debug expressions into each branch.

```reason
// ReScript
let _ =
  x =>
    @log
    switch x {
    | A => "A"
    | B(b) => b
    }

// Reason
let _ =
  x =>
    [@log]
    switch (x) {
    | A => "A"
    | B(b) => b
    }
```

Without a `@log` helper, an equivalent would be:

```reason
// ReScript
let _ =
  x =>
    switch (x) {
    | A =>
      %log.debug("A")
      "A"
    | B(b) =>
      %log.debug("B with payload" ("b", b))
      b
    }

// Reason
let _ =
  x =>
    switch (x) {
    | A =>
      [%log.debug "A"];
      "A";
    | B(b) =>
      [%log.debug "B with payload"; ("b", b)];
      b;
    }
```

You can pass optional custom namespace to helper like this: `@log("MyNamespace")`.

`[@log]` helper works only for `switch` expressions with constructor patterns, for now. Let us know [in the issues](/issues) if you need to handle more cases.

### `ReasonReact` integration
Using `@log` helper, you can log dispatched actions in your components.

Annotate `reducer` function like this:

```reason
// ReScript
let reducer =
  (state, action) =>
    @log
    switch action {
      ...
    }

// Reason
let reducer =
  (state, action) =>
    [@log]
    switch (action) {
      ...
    }
```

These entries are logged on the `debug` level so none of it will appear in your production builds.

### Custom loggers
`res-log` ships with 2 loggers:
- `ResLog.Browser` (default)
- `ResLog.Node`

And you can easily plug your own.

For example, in development, you want to log everything to console using default logger, but in production, you want to disable console logging and send `error` level events to bug tracker.

To implement your own logger, you need to create a module (e.g. `BugTracker.re`) and set the following environment variables for production build.

```
RES_LOG=error
RES_LOGGER=BugTracker
```

Considering that you want to log only `error` level messages, you need to create functions only for errors logging.

```reason
// BugTracker.res

let error = (loc, msg) => BugTrackerSDK.notify(`${msg} in ${loc.rootModule}`)

let error1 =
  (
    loc,
    msg,
    (label, payload),
  ) =>
    BugTrackerSDK.notify(
      `${msg} in ${loc.rootModule}`,
      [|(label, payload)|],
    );

let error2 =
  (
    loc,
    msg,
    (label1, payload1),
    (label2, payload2),
  ) =>
    BugTrackerSDK.notify(
      `${msg} in ${loc.rootModule}`,
      [|
        (label1, payload1),
        (label2, payload2),
      |],
    );

// Up to 7
```

The first argument `loc` is a `ResLog.Location.t` record. It's passed by PPX and contains location data.

```reason
type t = {
  rootModule: string,
  subModulePath: list<string>,
  value: option<string>,
  fullPath: string,
  filePath: string,
}
```

If `Test.Submodule.fn` gets called, logger would receive the following location:

```reason
// Test.res
module Submodule = {
  let fn = () => %log.warn("Warn!")
}

// Location
{
  rootModule: "Test",
  subModulePath: list{"Submodule"},
  value: Some("fn"),
  fullPath: "Test.Submodule.fn",
  filePath: "/absolute/path/to/project/src/Test.res",
}
```

---
Note, you don't have to re-implement all functions from the default logger, only the ones you actually use. Don't worry to forget to implement something. If later on, you will attempt to use unimplemented method it will be compile time error.

### Usage in libraries
I you develop a library and want to use `res-log` during development process, you can do so without spamming output of consumers of your library.

`res-log/ppx` accepts `--lib` flag:

```json
"ppx-flags": [
  ["res-log/ppx", "--lib=my-lib"]
]
```

Once this flag is passed, you need to provide special value of `RES_LOG` to log your entries:

```shell
RES_LOG=my-lib=* bsb -make-world
```

If consumers of your lib would like to see log output from your lib, they can do so too by extending a value of `RES_LOG` variable:

```shell
RES_LOG=*,my-lib=error bsb -make-world
```

Few more examples to illustrate how it works:

```shell
# log everything from application code only
RES_LOG=* bsb -make-world

# log everything from application code
# log errors from `my-lib`
RES_LOG=*,my-lib=error bsb -make-world

# log everything from application code
# log errors from `my-lib-1`
# log warnings and errors from `my-lib-2`
RES_LOG=*,my-lib-1=error,my-lib-2=warn bsb -make-world
```

## Caveats
**Logging is disabled after file save**<br />
If you run `bsb` via editor integration, make sure editor picked up `RES_LOG` variable. E.g. if you use Atom run it like this:

```shell
RES_LOG=info atom .
```

If your editor is telling you, variables used in ppx are unused, you can either:
1. prefix such variables with `_`
2. or open editor with `RES_LOG` variable set to appropriate level.

**Changing value of `RES_LOG`/`RES_LOGGER`/`RES_LOG_ONLY` doesn't make any effect**<br />
When you change a value of environment variable, `-clean-world` before the next build.

## Developing
Repo consists of 2 parts:
- ReScript lib: dependencies are managed by `yarn`
- Dune PPX: dependencies are managed by `esy`

Clone repo and install deps:

```shell
esy install
yarn install
```

Build loggers and ppx:

```shell
esy build
cd lib && yarn run build
cd ../examples && yarn run build
```

### Auto-formatting
Note, this project doesn't use auto-formatting in Reason files (`*.re`), so if you're intended to contribute, please, turn auto-formatting in the editor off while editing such files.
