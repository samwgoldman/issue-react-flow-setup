Starting with React 0.14, getting started with Flow requires an extra configuration step, which is a barrier for new people to get started using Flow. React is a excellent "gateway drug" for Flow, so I want to optimize the DX here to encourage more people to use Flow.

Dependencies: `npm` and `flow`

To reproduce:

1. `npm install` to install React 0.14 locally
1. `flow check` to run the tests

Expected results:

`No errors!`

Actual results:

```
/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/Deferred.js:39:15,15: return
Missing annotation

/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/PromiseMap.js:15:16,34: Deferred
Required module not found

/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/PromiseMap.js:17:17,36: invariant
Required module not found

/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/fetchWithRetries.js:16:28,58: ExecutionEnvironment
Required module not found

/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/fetchWithRetries.js:19:15,32: sprintf
Required module not found

/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/fetchWithRetries.js:20:13,28: fetch
Required module not found

/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/fetchWithRetries.js:21:15,32: warning
Required module not found
```

What's going on:

React 0.14 depends on fbjs. The npm package for fbjs includes some Flow-annotated source files in `flow/include`, but those files have `require` calls referencing modules that can't be found. See more in [Issue #44](https://github.com/facebook/fbjs/issues/44) in the fbjs repo.

Workaround:

You can work around these errors by telling Flow to ignore the fbjs files, which are pulled in indirectly via React. In `.flowconfig`, you want the following:

```
[ignore]
.*/react/node_modules/.*
```

Note on module systems:

Facebook internally uses a separate module system called "haste" which Flow understands. You can instruct Flow to use the haste module system instead of the (default) node module system with the following in your `.flowconfig`:

```
[options]
module.system=haste
```

In our case, that doesn't help, because fbjs includes duplicate module providers, so you see errors like this:

```
/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/lib/CSSCore.js: CSSCore
Duplicate module provider
/path/to/issue-react-flow-setup/node_modules/react/node_modules/fbjs/flow/include/CSSCore.js: current provider
```

Solution:

I think there are a few options to improve the "getting started" problem with React 0.14/Flow:

* Just document the required config options. This is the easiest way forward, but I really want to fix this in a way that requires zero configuration from the end user. It's bad if every project that wants to use React/Flow together needs to share this configuration.
* Implement features in Flow that allow the configuration to be inserted by React. There is some discussion about adding a feature to Flow which would recognize a `.flowconfig` inside of a dependency, and respect that config in addition to the project's local config. Hopefully, this would make it possible for React to provide a `.flowconfig` with an `[ignore]` entry for its own fbjs dependency. However, this feature in Flow isn't exactly on the horizon. React 0.14 will most likely be released well in advance of that feature.
* Just nuke the `flow/include` directory from the npm package for fbjs. This is my preferred solution. The existing vendored code doesn't actually help npm-based end users take advantage of the annotations therein. This code only serves to confuse Flow in the npm/OSS use case.
* Fix the npm-packaged `flow/include` files to use a non-haste module system. That is, the `require` calls in those files should use relative paths to eachother. This exact transformation is already applied to the fbjs code when creating the javascript in the `lib` directory of that project.
