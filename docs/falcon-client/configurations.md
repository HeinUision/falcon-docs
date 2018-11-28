---
title: Configurations
---

## API contract

Application needs to have `index.js` file, and the following optional configuration `bootstrap.js` and `falcon-client.build.config.js` files. Each of them should be placed into the application root directory.

### `index.js` (required)

This is an application entry point which needs to export valid React element `React.ReactElement<any>` as default:

```js
import App from './src/App';

export default App;
```

In order to use your custom Apollo Schema you need to export it via `clientApolloSchema`:

```js
import clientApolloSchema from './src/clientState';

export { clientApolloSchema };
```

For more information see [this](falcon-client/state-management.md)

### `bootstrap.js`

This is an optional runtime configuration file.

```js
const config = require('config');

export default {
  config: { ...config },
  onServerCreated: server => {},
  onServerInitialized: server => {},
  onServerStarted: server => {}
};
```

#### config

This is configuration object used to setup `@deity/falcon-client`

`config: object`

- `logLevel: string` - (default: `'error'`) [@deity/falcon-logger](https://github.com/deity-io/falcon/tree/master/packages/falcon-logger) logger level
- `serverSideRendering: boolean` - (default `true`) switch to control whether the [SSR](/docs/falcon-client/basics#server-side-rendering) is enabled
- `googleTagManager: object` - Google Tag Manager configuration, [see the details](/docs/falcon-client/basics#google-tag-manager)
- `googleAnalytics`:
  - `trackerID` - Google Analytics tracking code
- `i18n: object` - internationalization configuration, [see the details](/docs/falcon-client/internationalization)
- `menus: object` - menus configuration [TODO]
- `apolloClient` - ApolloClient configuration object
  - `apolloClient.httpLink` - configuration object that will be passed to the
    `apollo-link-http` method which defines HTTP link the remote Falcon Server
    ([available options](https://www.apollographql.com/docs/link/links/http.html#options))
  - `apolloClient.config` - configuration object that will be passed to the
    main `ApolloClient` constructor
    ([available options](https://www.apollographql.com/docs/react/api/apollo-client.html#apollo-client))

All configuration passed by `config` is accessible via `ApolloClient`, which mean you can access any of its property via graphQL query.

In order to retrieve `logLevel` you can run following query:

```graphql
gql`
  query LogLevel {
    config @client {
        logLevel
      }
    }
  }
`
```

It is also possible to extend `config` object about your custom properties.

#### Runtime hooks

Falcon Client exposes set of hooks to which you can attache custom logic:

- `onServerCreated(server: Koa)` - handler invoked immediately after koa server creation
- `onServerInitialized(server: Koa)` - handler invoked immediately after koa server setup (when middlewares like handling errors, serving static files and routes were set up)
- `onServerStarted(server: Koa)` - handler invoked when koa server started with no errors

### `falcon-client.build.config.js`

This is an optional build-time configuration file which is used to set up the entire build process. Bellow, there is an example of `falcon-client.build.config.js` file content with defaults:

```js
module.exports = {
  clearConsole: true,
  useWebmanifest: false,
  i18n: {},
  envToBuildIn: [],
  plugins: []
};
```

- `clearConsole: boolean` - (default: `true`) determines whether console should be cleared when starting script
- `useWebmanifest: boolean` - (default: `false`) determines whether [Web App Manifest](/docs/falcon-client/basics#webmanifest) should be processed via webpack and included in output bundle
- `i18n: object` - (default: `{}`) internationalization configuration, [see the details](/docs/falcon-client/internationalization#configuration)
- `envToBuildIn` - (default: `[]`) an array of environment variable names which should be build in into bundle, [see the details](#environment-variables)
- `plugins` - (default: `[]`) an array of plugins which can modify underlying [webpack configuration](#webpack).

Falcon Client provides you much more build configuration options. You can find all of them described in [Build process configuration](#build-process-configuration) section.

## Build process configuration

Falcon Client comes with all necessary features and development tools turned on by default:

- Universal [HMR](https://webpack.js.org/concepts/hot-module-replacement/) - page auto-reload if you make edits (on both backend and frontend)
- Latest JavaScript achieved via babel 7 compiler, [see the details](#babel).
- ESLint with [Prettier](https://github.com/prettier/prettier) - to keep your code base clean and consistent, [see the details](#eslint)
- Jest test runner setup with sensible defaults [see the details](#jest).

However, you can still modify Falcon Client defaults

### Environment Variables

<!-- ### Build-time Variables -->

Falcon client uses set of environment variables. All of them can be accessed via `process.env` anywhere in your application.

- `process.env.NODE_ENV` - `development` or `production`
- `process.env.BABEL_ENV`- `development` or `production`
- `process.env.PORT`- (default: `3000`), builded in only when `development`
- `process.env.HOST`- default is `0.0.0.0`
- `process.env.BUILD_TARGET` - `client` or `server`
- `process.env.ASSETS_MANIFEST` - path to webpack assets manifest,
- `process.env.PUBLIC_DIR`: directory with your static assets

You can create your own custom build-time environment variables. They must be listed on `envToBuildIn` in `falcon-client.build.config.js` file. Any other variables except the ones listed above will be ignored to avoid accidentally exposing a private key on the machine that could have the same name. Changing any environment variables will require you to restart the development server if it is running.

For example, bellow configuration expose environment variable named `SECRET_CODE` as `process.env.SECRET_CODE`:

```js
module.exports = {
  envToBuildIn: ['SECRET_CODE']
};
```

### Webpack

Falcon Client gives you the possibility to extend the underlying webpack configuration. You can do it via exposed plugins API (not webpack plugins). It is worth to mention that Falcon Client plugins API is [razzle](https://github.com/jaredpalmer/razzle#plugins) compatible.

To use Falcon Client (or Razzle) plugin, you need to install it in your project, and add it to `plugins` array in `falcon-client.build.config.js` file:

```js
module.exports = {
  plugins: ['plugin-name']
};
```

Plugins are simply functions that modify and return Falcon Client's webpack config. Each plugin function will receive the following arguments:

- `config: object` - webpack configuration object,
- `env.target: 'web' | 'node'` - webpack build target,
- `env.dev: boolean` - `true` when `process.env.NODE_ENV` is `development`, otherwise `false`,
- `webpack: Webpack` - webpack reference,

```js
module.exports = function myFalconClientPlugin(config, env, webpack) {
  const { target, dev } = env;

  if (target === 'web') {
    // client only
  }
  if (target === 'server') {
    // server only
  }
  if (dev) {
    // development only
  } else {
    // production only
  }

  // your webpack config modifications

  return webpackConfig;
};
```

`falcon-client.build.config.js` file accepts also `modify` setting which is an escape hatch function, which can be used for quick webpack configuration modifications. Basically, it works same as plugins, but can be specified inline. Below you can find an example of extending webpack configuration about `MyWebpackPlugin` only in `development`:

```js
module.exports = {
  modify: (config, { target, dev }, webpack) => {
    if (dev) {
      config.plugins = [...config.plugins, new MyWebpackPlugin()];
    }

    // your webpack config modifications
    return config;
  }
};
```

### Babel

Falcon Client gives you Ecma Script 6 compiled via Babel 7. However, if you want to add your own babel transformations, you can override defaults by adding the `.babelrc` file into the root of your project. Please note that it is necessary to at the very minimum the default `@deity/babel-preset-falcon-client` preset:

```js
{
  "presets": [
    "@deity/babel-preset-falcon-client", // needed
  ],
  "plugins": [
    // additional plugins
  ]
}
```

### ESLint

Falcon Client comes with ESLint with [Prettier](https://github.com/prettier/prettier) rules - to keep your code base clean and consistent, [see presets](https://github.com/deity-io/falcon/tree/master/packages/falcon-dev-tools/eslint-config-falcon).
You can override (or extend) defaults by adding the `.eslintrc` file into the root of your project:

```JSON
{
  "extends": ["@deity/eslint-config-falcon"],
  "rules": {
    "foo/bar": "error",
  }
}
```

### Jest

Falcon Client comes with configured [Jest](https://jestjs.io/) test runner. However it is possible to override it by adding `jest` node into `package.json`. Below example configures `setupTestFrameworkScriptFile` file:

```JSON
// package.json
{
 "jest": {
   "setupTestFrameworkScriptFile": "./setupTests.js"
 }
}
```