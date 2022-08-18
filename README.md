# Refactor of import-related host hooks

ECMA-262 currently exposes two hooks related to modules loading: [`HostResolveImportedModule`](https://tc39.es/ecma262/#sec-hostresolveimportedmodule) and `HostImportModuleDynamically`.

`HostResolveImportedModule(referencingScriptOrModule, specifier)` synchronously resolves an imported module and returns the corresponding module record. While module resolution and loading is _usually_ asynchronous, this was a good enough abstraction for the ES2015 modules specification: before evaluating a module, hosts could pre-build the module graph before evaluating a module and asynchronously load all its dependencies. This asynchronous step was not observable from ECMA-262, whose algorithms where only run once all the dependencies where synchronously available.

When we introduced dynamic imports in ES2020, this abstraction leaked: the asynchronous loading part needed to run _during_ the execution of other ECMAScript code, so we had to introduce the new host hook [`HostImportModuleDynamically(referencingScriptOrModule, specifier, promiseCapability)`](https://tc39.es/ecma262/#sec-hostimportmoduledynamically) to give hosts the opportunity to asynchronously prepare for the synchronous `HostResolveImportedModule` calls.

When loading and evaluating modules, either using host-defined mehanisms such as `<script>` tags or when using `HostImportModuleDynamically` via dynamic import, the complete algorithm is divided between the host and ECMA-262:
1. **(host, potentially async)** Load the module graph:
   1. **(host, potentially async)** Load the Module Record.
   1. **(host)** Get all its static dependency specifiers.
   1. **(host)** For each dependency, do 1.i.
1. **(host)** Call `.Link()` on the top-level Module Record:
   1. **(ECMA-262)** Get all its static dependency specifiers.
   1. **(ECMA-262)** For each dependency:
      1. **(ECMA-262)** Call `HostResolveImportedModule(specifer, referencingModule)`.
      1. **(host)** Get the pre-loaded module curresponding to the `(specifer, referencingModule)` pair.
      1. **(ECMA-262)** Validate that the imported bindings are actually exported.
      1. **(ECMA-262)** Do 2.i for the resulting module.
1. **(host, potentially async)** Call `.Evaluate()` on the top-level Module Record.

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

TODO

## Is this normative or editorial?

This proposal changes the number of spec-defined promise ticks when successfully importing a module with `import("foo")`.

| Has `"foo"` already been imported? | Is `"foo"` a Cyclic Module Record? | Old number of ticks         | New number of ticks         |
| :--------------------------------- | :--------------------------------- | :-------------------------- | :-------------------------- |
| Yes, from the same module          | Yes                                | (host-defined) + 2          | 2                           |
| Yes, from a somewhere else         | Yes                                | (host-defined) + 2          | (host-defined) + 2          |
| No                                 | Yes                                | (host-defined) + 2 + (Eval) | (host-defined) + 2 + (Eval) |
| Yes, from the same module          | No                                 | (host-defined) + 2          | 1 + (Eval)                  |
| Yes, from a somewhere else         | No                                 | (host-defined) + 2          | (host-defined) + 1 + (Eval) |
| No                                 | No                                 | (host-defined) + 2 + (Eval) | (host-defined) + 1 + (Eval) |

- (host-defined) represents the number of ticks that the host needs to load the module and its dependencies. With the old behavior it's the time needed by the host to call `FinishDynamicImport` and to resolve its `promise` argument; with the new behavior it's the time taken by all the `HostLoadImportedModule` executions to call `FinishLoadImportedModule`. It's always at least 1, because in both cases it results in a promise that needs to be awaited.
- (Eval) represents the number of promise ticks used by the `.Evaluate()` method of module records, which returns a promise. With the old behavior hosts could inspect the promise and synchronously continue if it was already resolved, so it could be 0 ticks; with the new behavior it always takes at least 1 tick because the resulting promise is always awaited.
