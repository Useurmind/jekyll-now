---
layout: post
title: A Workable React Dev Stack (1) - Tooling and Build Process
tags: React TypeScript Karma Webpack Test Unit Jasmine MaterialUI Rfluxx
---

Setting up a React app from scratch can be a daunting task. Today I will present the tooling and build process of the stack that we are currently using in our projects and how all the pieces fit together.

## Introduction

Today there are many helpers for setting up your react app. Take for example [create-react-app](https://create-react-app.dev) which sets up a complete react app with one command. However there are currently so many different possibile combinations of frameworks that it becomes hard to decide whether any possible generator fits your needs. Therefore, I want to describe our current stack and how all the different tools in the build process fit together. That will  give you an overview of what is needed for a workable react stack.

## The Example App

Through this series of posts we will look at an example app to explain how the stack works. In the following we will set it up and then start looking at part of its content.

### Generating the Example App

Codifying things is good as they become repeatable and can be anaylized easier. I liked the idea of [create-react-app](https://create-react-app.dev) and generated a code generator for our setup. The stack we will look at is codified in the [yeoman](https://yeoman.io/) generator [generator-rfluxx](https://www.npmjs.com/package/generator-rfluxx).

You can install both by typing:

    npm i -g yo generator-rfluxx

When both is installed create a folder for your app and type

    yo rfluxx

Answer the questions (just answer 'yes' for all of them) and after some time your app will be ready for use.
Check that it works by running

    npm run start

and go to `localhost:8080/Home`. You should see a small welcome screen.

### The Example App explained

Let's have a look at the contents of the example app:

```bash
root:
│   karma.conf.js
│   package-lock.json
│   package.json
│   tsconfig.json
│   tslint.json
│   webpack.config.js
│   webpack.debug.js
│   webpack.release.js
│
├───.vscode
│       tasks.json
│
├───spec
│       ...
│
└───src
        ...
```

As we will only discuss the build process and dev setup in this post, the part of the files to focus on now is the content of the root and `.vscode` directory. They contain all the configuration files for the tools we will be using.

- `karma.conf.js`: Configuration file for the karma test runner.
- `package-lock.json`: Configuration file for npm package manager.
- `package.json`: Configuration file for npm package manager.
- `tsconfig.json`: Configuration file for the typescript compiler.
- `tslint.json`: Configuration file for tslint code style checker.
- `webpack.config.js`: Configuration file for webpack with common configuration.
- `webpack.debug.js`: Configuration file for webpack with development configuration.
- `webpack.release.js`: Configuration file for webpack with production configuration.
- `.vscode/tasks.json`: Configuration for visual studio code tasks.

## The components of the stack

We are using the following tools during development:

- __[Visual Studio Code](https://code.visualstudio.com/)__: A great IDE for many use cases. And it is free. Very good for typescript development. Fast to install, start and update. Easy to extend. The list can probably go on for while. Only drawback is its slow intellisense in our project.
- __[Npm](https://github.com/npm/cli)__: When developing a serious project you arguably need a package manager that manages the hundreds or thousands of dependencies you need. Npm is one of the standard package managers for the Typescript community and our current choice.
- __[Typescript](https://www.typescriptlang.org/) (compiler)__: We write all our code in typescript, because the compilation process makes finding inconsistencies after refactorings/changes much, much easier.
- __[Webpack](https://webpack.js.org/)__: Many other languages have a compiler and build system build in. In JS land you need to find a tool that gathers all the files/modules you have created including your assets and produces something you can deploy onto a server of your choice.
- __[Webpack Dev Server](https://webpack.js.org/configuration/dev-server/)__: Webpack provides a development server that serves the project locally on the fly with automatic recompilation.
- __[Karma](https://karma-runner.github.io/)__: Karma is a unit test runner that integrates with many different browsers, test frameworks and packaging tools which in our case is webpack.
- __[Tslint](https://palantir.github.io/tslint/)__: Tslint is a code style checker specifically designed for typescript. An alternative tool would be [eslint](https://eslint.org/) which seems to be the future choice according to [Typescript Roadmap](https://github.com/Microsoft/TypeScript/issues/29288) and a [post from palantir](https://medium.com/palantir/tslint-in-2019-1a144c2317a9) the creators of tslint.

These tools provide a good basis for developing, building and testing any javascript app. Setting all of these tools up to play together hand-in-hand is arguably a complex task. Therefore we will have a look at the setup in depth during the course of this post.

## Npm & VS Code

NPM is the package/dependency manager for our apps/packages. Inside its `package.json` it stores not only dependencies (packages our apps depend on) but can also store information about the name/version and other information regarding the created package. 

![VS Code & NPM]({{ "/assets/img/ATypescriptStack_VSC_NPM.svg" | absolute_url }})

The most important function is installing all required packages into your project. During installation npm computes all required packages and downloads them into the `node_modules` folder. This is the de facto standard directory that is searched by most tools in the JS world for dependencies/packages. There is even a [default search algorithm](https://nodejs.org/api/modules.html#modules_all_together) to find packages inside these folders which is compatible with most setups and tooling.

__`package.json`:__
```json
{
  "name": "<whatever you choose as a name>",
  // ...
  "scripts": {
    "lint": "tslint -p .\\tsconfig.json",
    "lintfix": "tslint -p .\\tsconfig.json --fix",
    "test": "karma start",
    "testonce": "karma start --single-run",
    "build": "webpack-cli --config webpack.debug.js",
    "build_production": "webpack-cli --config webpack.release.js",
    "start": "webpack-dev-server --https --config webpack.debug.js",
    "start_production": "webpack-dev-server --https --config webpack.release.js"
  },
  // ...
  "dependencies": {
    "@material-ui/core": "^4.6.0",
    "@material-ui/icons": "^4.5.1",
    "@material-ui/styles": "^4.6.0",
    "classnames": "^2.2.6",
    // ...
  },
  "devDependencies": {
    "@types/classnames": "^2.2.9",
    "@types/jasmine": "^3.4.6",
    "@types/react": "^16.9.11",
    "@types/react-dom": "^16.9.4",
    "clean-webpack-plugin": "^3.0.0",
    "fork-ts-checker-webpack-plugin": "^3.1.0",
    // ...
  }
}
```

The `package.json` contains all this information. Above you can see the most crucial parts for our work.

- `name`: The name of the app.
- `scripts`: Tasks that can be executed with `npm run` or [`npx`](https://www.npmjs.com/package/npx)
- `dependencies`: The packages your app depends on, e.g. react.
- `devDependencies`: The packages you need to develop your app, e.g. webpack, typescript.

### NPM Package Lock

Direct dependencies which are packages that you install via `npm install` are remembered in the `package.json`. When packaging your app the packaging system also needs to retrieve code for the dependencies of your direct dependencies (recursively). Computation of this full version tree is time consuming. Worst of all the version tree can differ over time if packages are not installed with exact versions but with [caret ranges](https://docs.npmjs.com/misc/semver#caret-ranges-123-025-004). The full version tree with exact versions is stored in the `package-lock.json`. You can use this full version tree during install in continuous integration to speed up installation (for us it reduced installation time from 70s to 35s) and to ensure that the same version tree is used as during development.

### NPM Scripts and VS Code

In npm's `package.json` you can store a set of scripts that have meaning inside your project. Script names could e.g. be build, test, and start. You start npm scripts from the command line by typing `npm run <scriptname>` or by using the [npx tool](https://www.npmjs.com/package/npx) with `npx <scriptname>`.

On the other hand, VS Code comes with a very handy task execution engine. VS code can autodetect all npm scripts in your project (even recursively). Executing an npm script becomes easy as pie with this feature. You just have to open your task execution input and start typing `npm` or click the task in the NPM SCRIPTS window. Even more important is the feedback you get from those task executions which can be parsed using [VS Code problem matchers](https://code.visualstudio.com/docs/editor/tasks#_defining-a-problem-matcher). The results are then shown in the problems window of VS Code.

Being able to run scripts from the IDE as well as the command line is important for e.g. running stuff on your build server. By defaulting to npm scripts and integrating them in the developer workflow you have killed two birds with one stone.

In our projects we therefore have npm scripts for all major tasks that we need like building with webpack, starting webpack dev server, starting karma test runner, linting with tslint etc.

## VS Code & Typescript

Intellisense is probably one of the hottest topics during development. It saves typing effort through code completion, and indicates errors very early and continuously. It just saves time and effort and makes your life as a developer so much more pleasant. If it is accurate and fast that is. Slow or wrong itellisense can actually make you slower. Either because you are waiting for the completion that will only come after enough time to type the text three times. Or because of iritiation and search for errors that are not really errors.

![VS Code & Typescript]({{ "/assets/img/ATypescriptStack_VSC_TS.svg" | absolute_url }})

VS Code integrates intellisense for typescript and linting via extensions. The extensions will look for the configuration files of their tools in the project directory. For typescript itself this is the `tsconfig.json`. VS Code runs tsserver in the background to compile your code configuring it using the config file. For e.g. tslint which we use to check code style the configuration file is named `tslint.json`. Usually the config files should be placed in the root folder of your project. But they also work if placed somewhere else. The extensions usually search for config files in the parent folders of the file that is treated.

## Webpack Configuration

[Webpack](https://webpack.js.org/) by itself is already great. Structure your code however you want in as many files as you want. Webpack will gather them as you are used to from other languages like C# or Java. It will produce a consistent output package that you can serve from any webserver. To enable this for a vast bandwith of possible configurations it allows for customizable plugin & file loader pipelines. These pipelines are what makes webpack so flexible.

![Webpack & Typescript]({{ "/assets/img/ATypescriptStack_webpack_TS.svg" | absolute_url }})

In general you need a configuration file for webpack that tells it where to start looking for your code. In any code files webpack will resolve import and require statements. It does so by

- loading relative imports from the relative path stated in the import (relative to the source file that contains the import)
- loading absolute imports by following the [node default search algorithm](https://nodejs.org/api/modules.html#modules_all_together) (that is loading them from the next `node_modules` folder available in the parent directories of the source file that contains the import)

You can even require assets that are not code from code (a specific loader is required for this). 

Webpack will gather everything and put it into a single big javascript file. If required you can also split the javascript bundle into smaller units. The rest of the assets will be put into the output folder beside the javascript bundle(s). Assets can be comprised of e.g. images and fonts that were found while analyzing the code or e.g. sourcemaps that were created by loaders.

### Webpack & Typescript

When using typescript there is the additional complexity of compiling your code before packaging it up for the webserver. There is a whole list of requirements here:

- typescript files should be loaded and transpiled to java script
- typescript files should be compiled and type checked
- transpiled java script code should be packaged into the output package
- while developing you want a direct mapping of the running java script code to your original typescript source code via source maps
- incremental compilation should be possible to enable fast feedback during development

To achieve all of these goals you have several options. At the least you need to install the `ts-loader` and apply it to all typescript files. It will transpile the code to javascript and provide the source mappings necessary for debugging (only works if tsconfig specifies `sourceMap = true`). 

By default `ts-loader` will also compile and type check the code. This can become quite slow for larger projects. Therefore, we changed to a setup containing `fork-ts-checker-webpack-plugin`. This plugin will do the compilation asynchronously and apply the incremental build capability of the typescript compiler. In our case usage of this plugin reduced incremental build times from 20s to just below 2s.

The `fork-ts-checker-webpack-plugin` also checks for any linting errors. It supports tslint and eslint and will reuse their respective config files.

Here are the relevant parts of the webpack config file in `webpack.config.js`:

```js
module.exports = (params) => ({
    // ...
    module: {
        rules: [
            // all files with a `.ts` or `.tsx` extension
            // will be handled by `ts-loader`
            { 
                test: /\.tsx?$/, 
                loader: "ts-loader",
                exclude: /node_modules/,
                options: {
                    // just transpile (very fast), no code checking
                    transpileOnly: true,
                    experimentalWatchApi: true
                }
            }
        ]
    },
    plugins: [
        // checks typescript code for errors
        // asynchronously with incremental recompilation
        new ForkTsCheckerWebpackPlugin({
            useTypescriptIncrementalApi: true
            // tslint: true
            // eslint: true
        }),
        // more plugins ...
    ],
    // ...
})
```

### Webpack Config reused

The webpack configuration is quite a lot of code. It makes sense to only manage it once and reuse it for all scenarios where compiling/packaging is necessary. In our projects the configuration is reused in several places. 

![Webpack & Typescript]({{ "/assets/img/ATypescriptStack_webpack_config_reused.svg" | absolute_url }})

We use the `webpack-cli` to run the webpack build from the command line, e.g. on our build server. The `webpack-dev-server` is used during development to host the app that is created and continuously provide the latest compiled source bundle for rapid development cycles. Finally, we use [Karma](https://karma-runner.github.io/) to run unit tests against our code. The javascript specs are compiled and packaged using the same webpack config.

The `karma.conf.js` contains the following code that reuses the webpack config:
```js
// reuse webpack config
var webpackConfig = require("./webpack.debug");

// karma determines entry and output itself
webpackConfig.entry = undefined;
webpackConfig.output = undefined;

module.exports = function(config) {
  config.set({
    // ...

    // reuse webpack config
    webpack: webpackConfig,

    // preprocess matching files before serving them to the browser
    // available preprocessors: https://npmjs.org/browse/keyword/karma-preprocessor
    preprocessors: {
      // add webpack as preprocessor
      'spec/**/*Spec.ts': [ 'webpack' ]
    },

    // ...
  })
```

Both the webpack dev server as well as karma work in watch mode and profit from the improved transpilation/compilation time described above.

### Webpack & VS Code

Remember that we are using npm scripts to execute common commands as tasks in VS code. This is also true for running `webpack-cli`, `webpack-dev-server` or karma. It would be nice to see the feedback and problems from those tools directly inside VS code.

![Webpack & Typescript]({{ "/assets/img/ATypescriptStack_webpack_VSC.svg" | absolute_url }})

By defining a proper problem matcher you can integrate the error/warning messages resulting from the compilation in webpack. As all of the tools deliver the same output resulting from the webpack typescript compilation you can use the same problem matcher for all tasks.

There are already predefined problem matchers for this configuration. You only need to install this VS Code extension to make them available: [TypeScript + Webpack Problem Matchers](https://marketplace.visualstudio.com/items?itemName=eamodio.tsl-problem-matcher).

You can then define e.g. the following tasks in your `tasks.json` to build with `webpack-cli` or to run the `webpack-dev-server` respectively:
```json
{
    // run webpack cli
    "type": "npm",
    "script": "build",
    "isBackground": false,
    // problem matchers from extension
    "problemMatcher": [
        "$ts-webpack",
        "$tslint-webpack",
        "$ts-checker-webpack"
    ]
},
{
    // run webpack dev server
    "type": "npm",
    "script": "start",
    "isBackground": true,
    // problem matchers from extension
    "problemMatcher": [
        "$ts-webpack-watch",
        "$tslint-webpack-watch",
        "$ts-checker-webpack-watch"
    ]
},
{
    // run webpack dev server
    "type": "npm",
    "script": "test",
    "isBackground": true,
    // problem matchers from extension
    "problemMatcher": [
        "$ts-webpack-watch",
        "$tslint-webpack-watch",
        "$ts-checker-webpack-watch"
    ]
},
```

We assume here the scripts `build`, `run`, and `test` have been configured in the `package.json` similar to this:
```json
"scripts": {
    "build": "webpack-cli --config webpack.Debug.js",
    "start": "webpack-dev-server --https --config webpack.Debug.js",
    "test": "karma start",
    // ... more scripts
}
```

## Summary

In this post we discussed how to setup a complete development environment with VS Code, npm, webpack and typescript. The setup integrates dev tasks via npm scripts and VS code tasks into the editor. It keeps our builds fast by dividing the typescript compilation in two asynchronously running steps namely transpilation and type checking.

Have a look at the complete thing [here]({{ "/assets/img/ATypescriptStack.svg" | absolute_url }}).