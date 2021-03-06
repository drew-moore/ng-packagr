Design Doc: "ng-packagr"
========================

> Packaging TypeScript libraries in Angular Package Format


## Library authoring

#### Public API entry

Angular core libraries make heavy use of this pattern.
They provide a `public_api.ts` file as entry point to their library.
The file and all sources referenced from with this file are transpiled and bundled up in the library's public-facing API.
`ng-packagr` **WILL** support a single entry file to a library's public API.


#### Package definition and library metadata

`ng-packagr` **WILL** generate a `package.json` for library users.
The package definition is provided in that `package.json` file, including `peerDependencies`, `name`, `version`, and other information needed to distribute and publish the library.
Most important, the build artefacts (see below) are referenced in `package.json`.
In this way, library users and build tools will pick-up the correct build artefact of the library for compiling their applications.

`ng-packagr` **WILL NOT** implement a publishing workflow.
Steps in a publishing workflow are: verify or control version number (version bumping), generating a changelog, tracking public API changes, tagging a release, publishing distributables to a registry.
NONE of these will be implemented by `ng-packagr`.
There are other tools for these kind of purposes.
If neccessay, it's recommended that these tools pick up the `package.json` generated by `ng-packagr` and modify it (e.g. increase version number).


#### Build artefacts

`ng-packagr` **WILL** generate a TypeScript library in the Angular Package Format.
There need to be the following build artefacts:

* FESM2015 Bundle: `@<prefix>/<name>.js` in ES2015 syntax and ES2015 module format, a so-called Flat ECMAScript Module. It will be referenced in the es2015 property of `package.json`.
* FESM5 Bundle: `@<prefix>/<name>.es5.js` in ES5 syntax and ES2015 module format, a so-called Flat ECMAScript Module (ESM, or FESM5, or FESM2014). It will be referenced in the module property of `package.json`.
* UMD Bundle: `@<prefix>/<name>.umd.js` in ES5 syntax and UMD module format, a so-called universal module definition format. It will be referenced in the main property of `package.json`.
* Type definitions and AoT metadata: an `index.d.ts` file and a `index.metadata.json` file will be generated to support TypeScript debugging as well as AoT compilation. The `*.metadata.json` file MUST have templates and stylesheets inlined.
* A `package.json` file: it describes the structure of the library and serves as the entry point for library users, when resolving TypeScript `import { .. } from '@<prefix>/<name>'` statements.


---


## Tools and implementation details

Internally, `ng-packagr` is going to use several other tools to do the desired transformations from TypeScript sources (+ HTML templates and stylesheets) to Angular package format.
Here is a trade-off decision:

As first option, `ng-packagr` allows users to provide a full custom configuration for tools such as `ngc`, `rollup`, and so on.
This forces useer to write a configuration file for these tools and deal with configuration options.

Alternatively, `ng-packagr` will hide configuration and internals of tools such as `ngc`, `rollup`, and so on.
In this case, the configuration of `ng-packagr` will only allow to configure a limited set of options that will be passed through to the tools.


#### NGC: tsconfig.json

Right now, `@angular/tsc-wrapped` does not support the `"extends"` property of `tsconfig.json`.
Because of that, `ng-packagr` needs to support self-contained JSON configuration files for ngc.
If auto-generating a tsconfig, `ng-packagr` would need to read its default values, merge that with the custom user tsconfig and copy the result to its working directory.

The most important setting here is the `"files": []` property, which must contain exactly one file since `"flatModuleId"` and `"flatModuleOutFile"` options will also be used for flattended library indexes.
The value for `"flatModuleId"` could be inferred by the library's name as given in `package.json`, `"flatModuleOutFile"` could be statically set to `"index"`.

Other configuration properties like `"target"` or `"module"` cannot be set by users since the order of transformations relies on certain settings.
For example, `ngc` will need to compile to `"target": "es2015"` and `"module": "es2015"` in order to allow subsequent steps to happen.

~The path to `tsconfig.json` will be given `ngc.tsconfig` JSON configuration property.~
~A default configuration file should be provided with the tool, so that users can copy&paste.~


#### Rollup Config

For generating the bundled versions of the library, rollup will be used.
Rollup requires a configuration with a symbol mapping table.

Reasonable default values should be shipped with `ng-packagr` without forcing users to write special configuration.
The default configuration should try to support `@angular/*` packages as well as `rxjs`, which is a transitive dependency in most cases and also requires special configuration in Rollup.

Other configuration properties like `"entry"` or `"format"` cannot be set by users since their values depend on the order of transformations being applied.
For example, the transformation to UMD requires an FESM5 input file.
The FESM5 input file got created prior in the build process, thus `ng-packagr` will pass both the `"entry"` and `"format"` property to `rollup` without users being able to custimize.

~If required, users should be able to provide a custom rollup configuration to `ng-packagr` by settings the `rollup.config` JSON configuation property.~

---


## Configuration and customization

#### Config file "ng-package.json"

`ng-packagr` **WILL** support configuration of the tool through a JSON configuration file similar to a `tsconfig.json` or `package.json` per project.
It can be passed as a CLI argument with `ng-packagr -p myconfig.json`.
The default file name is `ng-package.json`.
The default config file settings are:

```json
{
  "src": ".",
  "dest": "dist",
  "workingDirectory": ".ng_build",
  "ngc": {
    "tsconfig": "tsconfig.lib.json"
  },
  "rollup": {
    "config": "rollup-config.js"
  }
}
```


#### Default file layout

By default, a source folder of a library should look like this:

```
src
| - public_api.ts
| - ..
ng-package.json
package.json
rollup-config.js
tsconfig.lib.json
```

In consequence, the distributable folder according to [Angular Package Format](https://docs.google.com/document/d/1CZC2rcpxffTDfRDs6p1cfbmKNLA6x5O-NtkJglDaBVs/preview) is:

```
@<prefix>
|- <name>.js
|- <name>.js.map
|- <name>.es5.js
|- <name>.es5.js.map
bundles
|- <name>.umd.js
|- <name>.umd.js.map
src
|- index.d.ts
|- index.metadata.json
package.json
README.md
LICENSE
```


#### Custom file layout

It's possible to cusomize the folder structure, as demonstrated by the official sample:

```
sample
| - src
    | - public_api.ts
    | - ..
| - package.json
| - rollup-config.js
| - tsconfig.lib.json
ng-package.json
```

The sample library is built with the following config:

```json
{
  "src": "sample",
  "dest": "dist",
  "workingDirectory": ".ng_build",
  "ngc": {
    "tsconfig": "tsconfig.lib.json"
  },
  "rollup": {
    "config": "rollup-config.js"
  }
}
```
