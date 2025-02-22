# Monaco YAML

[![ci workflow](https://github.com/remcohaszing/monaco-yaml/actions/workflows/ci.yaml/badge.svg)](https://github.com/remcohaszing/monaco-yaml/actions/workflows/ci.yaml)
[![npm version](https://img.shields.io/npm/v/monaco-yaml)](https://www.npmjs.com/package/monaco-yaml)
[![prettier code style](https://img.shields.io/badge/code_style-prettier-ff69b4.svg)](https://prettier.io)
[![demo](https://img.shields.io/badge/demo-monaco--yaml.js.org-61ffcf.svg)](https://monaco-yaml.js.org)
[![netlify status](https://api.netlify.com/api/v1/badges/20b08937-99d0-4882-b9a3-d5f09ddd29b7/deploy-status)](https://app.netlify.com/sites/monaco-yaml/deploys)

YAML language plugin for the Monaco Editor. It provides the following features when editing YAML
files:

- Code completion, based on JSON schemas or by looking at similar objects in the same file
- Hovers, based on JSON schemas
- Validation: Syntax errors and schema validation
- Formatting using Prettier
- Document Symbols
- Automatically load remote schema files (by enabling DiagnosticsOptions.enableSchemaRequest)
- Links from JSON references.
- Links and hover effects from YAML anchors.

Schemas can also be provided by configuration. See
[here](https://github.com/remcohaszing/monaco-yaml/blob/main/index.d.ts) for the API that the plugin
offers to configure the YAML language support.

## Installation

```sh
npm install monaco-yaml
```

## Usage

Import `monaco-yaml` and configure it before an editor instance is created.

```typescript
import { editor, Uri } from 'monaco-editor';
import { setDiagnosticsOptions } from 'monaco-yaml';

// The uri is used for the schema file match.
const modelUri = Uri.parse('a://b/foo.yaml');

setDiagnosticsOptions({
  enableSchemaRequest: true,
  hover: true,
  completion: true,
  validate: true,
  format: true,
  schemas: [
    {
      // Id of the first schema
      uri: 'http://myserver/foo-schema.json',
      // Associate with our model
      fileMatch: [String(modelUri)],
      schema: {
        type: 'object',
        properties: {
          p1: {
            enum: ['v1', 'v2'],
          },
          p2: {
            // Reference the second schema
            $ref: 'http://myserver/bar-schema.json',
          },
        },
      },
    },
    {
      // Id of the first schema
      uri: 'http://myserver/bar-schema.json',
      schema: {
        type: 'object',
        properties: {
          q1: {
            enum: ['x1', 'x2'],
          },
        },
      },
    },
  ],
});

editor.create(document.createElement('editor'), {
  // Monaco-yaml features should just work if the editor language is set to 'yaml'.
  language: 'yaml',
  model: editor.createModel('p1: \n', 'yaml', modelUri),
});
```

Also make sure to register the web worker. When using Webpack 5, this looks like the code below.
Other bundlers may use a different syntax, but the idea is the same. Languages you don’t used can be
omitted.

```js
window.MonacoEnvironment = {
  getWorker(moduleId, label) {
    switch (label) {
      case 'editorWorkerService':
        return new Worker(new URL('monaco-editor/esm/vs/editor/editor.worker', import.meta.url));
      case 'css':
      case 'less':
      case 'scss':
        return new Worker(new URL('monaco-editor/esm/vs/language/css/css.worker', import.meta.url));
      case 'handlebars':
      case 'html':
      case 'razor':
        return new Worker(
          new URL('monaco-editor/esm/vs/language/html/html.worker', import.meta.url),
        );
      case 'json':
        return new Worker(
          new URL('monaco-editor/esm/vs/language/json/json.worker', import.meta.url),
        );
      case 'javascript':
      case 'typescript':
        return new Worker(
          new URL('monaco-editor/esm/vs/language/typescript/ts.worker', import.meta.url),
        );
      case 'yaml':
        return new Worker(new URL('monaco-yaml/yaml.worker', import.meta.url));
      default:
        throw new Error(`Unknown label ${label}`);
    }
  },
};
```

## Examples

A demo is available on [monaco-yaml.js.org](https://monaco-yaml.js.org).

A running example: ![demo-image](test-demo.png)

Some usage examples can be found in the
[examples](https://github.com/remcohaszing/monaco-yaml/tree/main/examples) directory.

## FAQ

### Does this work with the Monaco UMD bundle?

No. Only ESM is supported.

### Does this work with Monaco Editor from a CDN?

No, because these use a UMD bundle, which isn’t supported.

### Does this work with `@monaco-editor/loader` or `@monaco-editor/react`?

No. These packages pull in the Monaco UMD bundle from a CDN. Because UMD isn’t supported, neither
are these packages.

### Is the web worker necessary?

Yes. The web worker provides the core functionality of `monaco-yaml`.

### Does it work without a bundler?

No. `monaco-yaml` uses dependencies from `node_modules`, so they can be deduped and your bundle size
is decreased. This comes at the cost of not being able to use it without a bundler.

### How do I integrate `monaco-yaml` with a framework? (Angular, React, Vue, etc.)

`monaco-yaml` only uses the Monaco Editor. It’s not tied to a framework, all that’s needed is a DOM
node to attach the Monaco Editor to. See the
[Monaco Editor examples](https://github.com/microsoft/monaco-editor/tree/main/monaco-editor-samples)
for examples on how to integrate Monaco Editor in your project, then configure `monaco-yaml` as
described above.

### Does `monaco-yaml` work with `create-react-app`?

Yes, but you’ll have to eject. See
[#92 (comment)](https://github.com/remcohaszing/monaco-yaml/issues/92#issuecomment-905836058) for
details.

### Why doesn’t it work with Vite?

Some users have experienced the following error when using Vite:

```
Uncaught (in promise) Error: Unexpected usage
  at EditorSimpleWorker.loadForeignModule (editorSimpleWorker.js)
  at webWorker.js
```

As a workaround, create a file named `yaml.worker.js` in your own project with the following
contents:

```js
import 'monaco-yaml/yaml.worker.js';
```

Then in your Monaco environment `getWorker` function, reference this file instead of referencing
`monaco-yaml/yaml.worker.js` directly:

```js
import YamlWorker from './yaml.worker.js?worker';

window.MonacoEnvironment = {
  getWorker(moduleId, label) {
    switch (label) {
      // Handle other cases
      case 'yaml':
        return new YamlWorker();
      default:
        throw new Error(`Unknown label ${label}`);
    }
  },
};
```

### Why isn’t `monaco-yaml` working? Official Monaco language extensions do work.

This is most likely due to the fact that `monaco-yaml` is using a different instance of the
`monaco-editor` package than you are. This is something you’ll want to avoid regardless of
`monaco-editor`, because it means your bundle is significantly larger than it needs to be. This is
likely caused by one of the following issues:

- A code splitting misconfiguration

  To solve this, try inspecting your bundle using for example
  [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer). If
  `monaco-editor` is in there twice, this is the issue. It’s up to you to solve this, as it’s
  project-specific.

- You’re using a package which imports `monaco-editor` for you, but it’s using a different version.

  You can find out why the `monaco-editor` is installed using `npm ls monaco-editor` or
  `yarn why monaco-editor`. It should exist only once, but it’s ok if it’s deduped.

  You may be able to solve this by deleting your `node_modules` folder and `package-lock.json` or
  `yarn.lock`, then running `npm install` or `yarn install` respectively.

## Contributing

Please see our [contributing guidelines](CONTRIBUTING.md)

## Credits

Originally [@kpdecker](https://github.com/kpdecker) forked this repository from
[`monaco-json`](https://github.com/microsoft/monaco-json) by
[@microsoft](https://github.com/microsoft) and rewrote it to work with
[`yaml-language-server`](https://github.com/redhat-developer/yaml-language-server) instead. Later
the repository maintenance was taken over by [@pengx17](https://github.com/pengx17). Eventually the
repository was tranferred to the account of [@remcohaszing](https://github.com/remcohaszing), who is
currently maintaining this repository with the help of [@fleon](https://github.com/fleon) and
[@yazaabed](https://github.com/yazaabed).

The heavy processing is done in
[`yaml-language-server`](https://github.com/redhat-developer/yaml-language-server), best known for
being the backbone for [`vscode-yaml`](https://github.com/redhat-developer/vscode-yaml). This
repository provides a thin layer to add functionality provided by `yaml-language-server` into
`monaco-editor`.

## License

[MIT](https://github.com/remcohaszing/monaco-yaml/blob/main/LICENSE.md)
