# Refactor of import-related host hooks

- [Refactor of import-related host hooks](#refactor-of-import-related-host-hooks)
  - [Proposed changes](#proposed-changes)
  - [Motivation](#motivation)
  - [Constraints](#constraints)
  - [Hosts integration](#hosts-integration)
  - [Is this normative or editorial?](#is-this-normative-or-editorial)
  - [Open questions](#open-questions)

## Proposed changes

ECMA-262 currently exposes two hooks related to modules loading: [`HostResolveImportedModule`](https://tc39.es/ecma262/#sec-hostresolveimportedmodule) and [`HostImportModuleDynamically`](https://tc39.es/ecma262/#sec-hostimportmoduledynamically).

`HostResolveImportedModule(referencingScriptOrModule, specifier)` synchronously resolves an imported module and returns the corresponding module record. While module resolution and loading is _usually_ asynchronous, this was a good enough abstraction for the ES2015 modules specification: before evaluating a module, hosts could pre-build the module graph before evaluating a module and asynchronously load all its dependencies. This asynchronous step was not observable from ECMA-262, whose algorithms where only run once all the dependencies where synchronously available.

When we introduced dynamic imports in ES2020, this abstraction leaked: the asynchronous loading part needed to run _during_ the execution of other ECMAScript code, so we had to introduce the new host hook [`HostImportModuleDynamically(referencingScriptOrModule, specifier, promiseCapability)`](https://tc39.es/ecma262/#sec-hostimportmoduledynamically) to give hosts the opportunity to asynchronously prepare for the synchronous `HostResolveImportedModule` calls.

When loading and evaluating modules, either using host-defined mehanisms such as `<script>` tags or when using `HostImportModuleDynamically` via dynamic import, the complete algorithm is divided between the host and ECMA-262:

1. **(host, potentially async)** Load the module graph:
   1. **(host, potentially async)** Load the Module Record.
   2. **(host)** Get all its static dependency specifiers.
   3. **(host)** For each dependency, do 1.i.
2. **(host)** Call `.Link()` on the top-level Module Record:
   1. **(ECMA-262)** Get all its static dependency specifiers.
   2. **(ECMA-262)** For each dependency:
      1. **(ECMA-262)** Call `HostResolveImportedModule(specifer, referencingModule)`.
      2. **(host)** Get the pre-loaded module curresponding to the `(specifer, referencingModule)` pair.
      3. **(ECMA-262)** Validate that the imported bindings are actually exported.
      4. **(ECMA-262)** Do 2.i for the resulting module.
3. **(host, potentially async)** Call `.Evaluate()` on the top-level Module Record.

This refactor proposal aims to revisit the layering decision made by the dynamic import proposal: rather than introducing a new hook to permit async host steps for `import()` calls, it replaces `HostResolveImportedModule` with an equivalent but async-compatible `HostLoadImportedModule` hook: it loads a single module, and ECMA-262 iterates through its dependencies asking to the host to load them. The updated algorithm is:

1. **(host or ECMA-262, potentially async)** Load the module graph:
   1. **(ECMA-262, potentially async)** Call `HostLoadImportedModule(specifier, referencingModule)`.
   1. **(ECMA-262)** Get all its static dependency specifiers.
   1. **(ECMA-262)** For each dependency, do 1.i.
1. **(host or ECMA-262)** Call `.Link()` on the top-level Module Record:
   1. **(ECMA-262)** Get all its static dependencies.
   1. **(ECMA-262)** For each dependency:
      1. **(ECMA-262)** Validate that the imported bindings are actually exported.
      1. **(ECMA-262)** Do 2.i for the resulting module.
1. **(host or ECMA-262, potentially async)** Call `.Evaluate()` on the top-level Module Record.

where **host or ECMA-262** means "host if the algorithm is run by an host-defined mechanism such as `<script>` tags, ECMA-262 if it's run by dynamic import".

## Motivation

This refactor has two benefits on its own: it reduces the amount of behavior delegated to the host, by taking ownership of the loading steps shared across all the hosts that use asynchronous loading.

However, it's most useful for some current proposals that introduce the concept of a "module whose dependencies have not been loaded yet":

- [Module Blocks](https://github.com/tc39/proposal-js-module-blocks) and [Compartments](https://github.com/tc39/proposal-compartments) need a new host hook to "load the dependencies of an already loaded module" ([`HostImportModuleRecordDynamically`](https://tc39.es/proposal-compartments/0-module-and-module-source.html#sec-hostimportmodulerecorddynamically));
- [Import Reflection](https://github.com/tc39/proposal-import-reflection) needs a new host hook to "load a module without loading its dependencies" ([`HostResolveModuleReflection`](https://tc39.es/proposal-import-reflection/#sec-hostresolvemodulereflection)).

`HostLoadImpotedModule` is low-level enough that it already satisfies those use cases:

- Module Blocks and Compartments can recursively call `HostLoadImportedModule` on the dependencies of the unlinked module;
- Import Reflection can use `HostLoadImportedModule` to load a single module without recursively loading its dependencies.

This refactor reduces the number of loading-related host hooks from 2 to 1, and prevents it from growing to 4 in the future.

## Constraints

This refactor should not force module loading to be asynchronous:

- For hosts that currently load modules synchronously, forcing a promise tick for each recursive dependency might cause a big performance regression.
- Some hosts (for example, [Bun](https://github.com/oven-sh/bun/blob/47a91e745781bd97b1747e2b11234244cd46b29d/src/bun.js/builtins/js/ImportMetaObject.js#L67)) allow synchronously importing modules that don't use top-level await, by relying on the fact that `module.Evaluate()` returns a resolved promise and synchronously inspecting is value.

With this refactor both are still possible: `HostLoadImportedModule` can synchronously give control back to ECMA-262 (reusing the same logic they had in `HostResolveImportedModule`) to synchronously continue the loading process. The new `module.LoadRequestedModules()` will then return a resolved promise.

Hosts can still implement synchronous import of modules:

1. Load the module.
1. Call `module.LoadRequestedModules()`, which returns a `loadPromise`.
1. If `loadPromise.[[Status]]` is `rejected`, throw; otherwise it's `fulfilled`.
1. Call `module.Link()`.
1. Call `module.Evaluate()`, which returns a `evalPromise`.
1. If `evalPromise.[[Status]]` is `rejected`, throw.
1. If `evalPromise.[[Status]]` is `pending`, throw (it's using top-level await).
1. Return `GetModuleNamespace(module)`.

## Hosts integration

Hosts can use these new ECMA-262 algorithms in two ways.

This is the most straigthforward integration is to keeps the loading algorithms used for `HostResolveImportedModule`/`HostImportModuleDynamically`.

Assuming that the old hooks are implemented as follows:

- _HostResolveImportedModule_(`referencingScriptOrModule`, `specifier`):
  1. Let `fullSpecifier` be the result of resolving `specifier` from `referencingScriptOrModule` (for example, via URLs).
  2. Return the already loaded Module Record corresponding to `fullSpecifier`.
- _HostImportModuleDynamically_(`referencingScriptOrModule`, `specifier`, `promiseCapability`):
  1. Let `fullSpecifier` be the result of resolving `specifier` from `referencingScriptOrModule` (for example, via URLs).
  2. (async) Let `module` be the result of loading and parsing the source code corresponding to `fullSpecifier`.
  3. (async) Recursively load and parse the dependencies of `module`.
  4. Call `module`._Link_().
  5. Let `evaluationPromise` be all `module`._Evaluate_().
  6. Call _FinishDynamicImport_(`referencingScriptOrModule`, `specifier`, `promiseCapability`, `evaluationPromise`).

The new host hook would be implemented as follows:

- _HostLoadImportedModule_(`referrer`, `specifier`):
  1. Let `fullSpecifier` be the result of resolving `specifier` from `referencingScriptOrModule` (for example, via URLs).
  2. If a Module Record corresponding to `fullSpecifier` has already been loaded, then:
     1. Let `module` be such Module Record.
     2. Return `module`.
  3. (async) Let `module` be the result of loading and parsing the source code corresponding to `fullSpecifier`.
  4. (async) Recursively load and parse the dependencies of `module`.
  5. Call _FinishLoadImportedModule_(`referrer`, `specifier`, `module`).

A more advanced refactor would avoid step 3. of the above _HostLoadImportedModule_ implementation, and fully delegate the dependencies discovery algorithm to ECMA-262. Hosts should carefully consider the differences between the ECMA-262 algorithm and their own before doing so.

## Is this normative or editorial?

This proposal changes the number of spec-defined promise ticks when successfully importing a module with `import("foo")`.

| Has `"foo"` already been imported? | Is `"foo"` a Cyclic Module Record? | Old number of ticks | New number of ticks         |
| :--------------------------------- | :--------------------------------- | :------------------ | :-------------------------- |
| Yes, from the same module          | Yes                                | (host-defined) + 1  | 2                           |
| Yes, from a somewhere else         | Yes                                | (host-defined) + 1  | (host-defined) + 2          |
| No                                 | Yes                                | (host-defined) + 1  | (host-defined) + 1 + (Eval) |
| Yes, from the same module          | No                                 | (host-defined) + 1  | 1 + (Eval)                  |
| Yes, from a somewhere else         | No                                 | (host-defined) + 1  | (host-defined) + 1 + (Eval) |
| No                                 | No                                 | (host-defined) + 1  | (host-defined) + 1 + (Eval) |

- (host-defined) represents the number of ticks that the host needs to load the module and its dependencies. With the old behavior it's the time needed by the host to call `FinishDynamicImport` and to resolve its `promise` argument; with the new behavior it's the time taken by all the `HostLoadImportedModule` executions to call `FinishLoadImportedModule`.
- (Eval) represents the number of promise ticks used by the `.Evaluate()` method of module records, which returns a promise and thus takes at least 1 tick to be awaited.

## Open questions

**Is it possibe to use a single method that does both _module.LoadRequestedModules()_ and _module.Link()_?**

_module.Link()_ uses [Tarjan's algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm) to find <abbr title="Strongly Connected Components">SCCs</abbr> in the modules graph and transition their elements' status from `linking` to `linked` at the same time. This algorithm tracks SCCs using a mutable stack where it pushes/pops Module Records while traversing the graph.

This approach doesn't work with _module.LoadRequestedModules()_, because it visits multiple paths of the graph concurrently: a mutable stack would cause race conditions in the detection of different SCCs. For this reason, _module.LoadRequestedModules()_ transitions the modules status from `new` to `unlinked` after loading the whole graph.

1. Is it important that _module.Link()_ transition the status from `linking` to `linked` as soon as possible?
2. Is there an efficient alternative to Trajan's algorithm that works with concurrent traversals?
