---
layout: post
title: A Workable React Dev Stack (1) - Tooling and Build Process
tags: React TypeScript Karma Webpack Test Unit Jasmine MaterialUI Rfluxx
---

Setting up a React app from scratch can be a daunting task. Today I will present the tooling and build process of the stack that we are currently using in our projects and how all the pieces fit together.

## Introduction

Today there are many helpers for setting up your react app. Take for example [create-react-app](https://create-react-app.dev) which sets up a complete react app with one command. However there are currently so many different possibile combinations of frameworks that it becomes hard to decide whether any possible generator fits your needs. Therefore, I want to describe our current stack and how all the different frameworks fit together. That should give you an overview of what is needed for a workable react stack.

## Generate example app

Codifying things is good as they become repeatable and can be anaylized easier. The stack we will look at is codified in the [yeoman](https://yeoman.io/) generator [generator-rfluxx](https://www.npmjs.com/package/generator-rfluxx).

You can install both by typing:

    npm i -g yo generator-rfluxx

When both is installed create a folder for your app and type

    yo rfluxx

Answer the questions (just answer 'yes' for all of them) and after some time your app will be ready for use.
Check that it works by running

    npm run start

and go to `localhost:8080/Home`. You should see a small welcome screen.

## The components of the stack

The stack uses the following tools during development:

- __Visual Studio Code__: A great IDE for many use cases. And it is free. Very good for typescript development. Fast to install, start and update. Easy to extend. The list can probably go on for while. Only drawback is its slow intellisense in our project.
- __Npm__: When developing a serious project you arguably need a package manager that manages the hundreds or thousands of dependencies you need. Npm is one of the standard package managers for the Typescript community and our current choice.
- __Typescript (compiler)__: We write all our code in typescript, because the compilation process makes finding inconsistencies after refactorings/changes much, much easier.
- __Webpack__: Many other languages have a compiler and build system build in. In JS land you need to find a tool that gathers all the files/modules you have created including your assets and produces something you can deploy onto a server of your choice.
- __Webpack Dev Server__: Webpack provides a development server that serves our project locally on the fly with automatic recompilation.
- __Karma__: Karma is a unit test runner that integrates with many different browsers, test frameworks and packaging tools which in our case is webpack.

These tools provide a good basis for developing, building and testing any javascript app. Setting all of these tools up to play together hand-in-hand is a complex task. Therefore I will explain the setup in depth during the course of this post.

## Npm & VS Code

![VS Code & NPM]({{ "/assets/img/ATypescriptStack_VSC_NPM.svg" | absolute_url }})

### NPM core functions

NPM is the package manager and meta information management system for our apps/packages. Inside its `package.json` it stores not only dependencies (packages our apps depend on) but can also store information about the name/version and other information regarding the created package. The most important function is installing all required packages into your project. During installation npm computes all required packages and downloads them into the `node_modules` folder. This is the de facto standard directory that is searched by most tools in the JS world for dependencies/packages. There is even a [default search algorithm](https://nodejs.org/api/modules.html#modules_all_together) to find packages inside these folders which is compatible with most setups and tooling.

### NPM Package Lock

Direct dependencies which are packages that you install via `npm install` are stored in the `package.json`. When packaging your app the packaging system also needs to retrieve code for the dependencies of your direct dependencies (recursively). Computation of this full version tree is time consuming. Worst of all the version tree can differ over time if packages are not installed with exact versions but with [caret ranges](https://docs.npmjs.com/misc/semver#caret-ranges-123-025-004). The full version tree is stored in the `package-lock.json`. You can use this full version tree during install in continuous integration to speed up installation (for us it reduced installation time from 70s to 35s) and to ensure that the same version tree is used as during development.

### NPM Scripts and VS Code

In npm's `package.json` you can store a set of scripts that have meaning inside your project. Script names could e.g. be build, test, and start. You start npm scripts from the command line by typing `npm run <scriptname>`.

On the other hand, VS Code comes with a very handy task execution engine. VS code can autodetect all npm scripts in your project (even recursively). Executing an npm script becomes easy as pie with this feature. You just have to open your task execution input and start typing `npm` or click the task in the NPM SCRIPTS window. Even more important is the feedback you get from those task executions which can be parsed using [VS Code problem matchers](https://code.visualstudio.com/docs/editor/tasks#_defining-a-problem-matcher). The results are then shown in the problems window of VS Code.

Being able to run scripts from the IDE as well as the command line  is important for e.g. running stuff on your build server. By defaulting to npm scripts and integrating them in the developer workflow you have killed two birds with one stone.

In our projects we therefore have npm scripts for all major tasks that we need like building with webpack, starting webpack dev server, starting karma test runner, linting with tslint etc.

## VS Code & Typescript

![VS Code & Typescript]({{ "/assets/img/ATypescriptStack_VSC_TS.svg" | absolute_url }})

Intellisense is probably one of the hottest topics during development. It saves typing effort through code completion, and indicates errors very early and continuously. It just saves time and effort and makes your life as a developer so much more pleasant. If it is accurate and fast that is. Slow or wrong itellisense can actually make you slower. Either because you are waiting for the completion that will only come after enough time to type the text three times. Or because of iritiation and search for errors that are not really errors.

VS Code integrates intellisense for typescript and linting via extensions. The extensions will look for the configuration files of their tools in the project directory. For typescript itself this is the `tsconfig.json`. For e.g. tslint which we use to check code style the configuration file is named `tslint.json`. Usually the config files should be placed in the root folder of your project. But they also work if placed somewhere else. The extensions usually search for config files in the parent folders of the file that is treated.

## Webpack Configuration

### Webpack Core Functions

Webpack by itself is already great. Structure your code however you want in as many files as you want. Webpack will gather them as you are used to from other languages like C# or Java. It will produce a consistent output package that you can serve from any webserver. To enable this for a vast bandwith of possible configurations it allows for customizable plugin & file loader pipelines. These pipelines are what makes webpack so flexible.

### Webpack & Typescript

![Webpack & Typescript]({{ "/assets/img/ATypescriptStack_webpack_TS.svg" | absolute_url }})

When using typescript there is the additional complexity of compiling your code before packaging it up for the webserver. There is a whole list of requirements here:

- typescript files should be loaded and transpiled to java script
- typescript files should be compiled and type checked
- transpiled java script code should be packaged into the output package
- while developing you want a direct mapping of the running java script code to your original typescript source code via source maps
- incremental compilation should be possible to enable fast feedback during development

To achieve all of these goals you have several options. At the least you need to install the `ts-loader` and apply it to all typescript files. It will transpile the code to javascript and provide the source mappings necessary for debugging (only works if tsconfig specifies `sourceMap = true`). 

By default `ts-loader` will also compile and type check the code. This can become quite slow for larger projects. Therefore, we changed to a setup containing `fork-ts-checker-webpack-plugin`. This plugin will do the compilation asynchronously and apply the incremental build capability of the typescript compiler. In our case usage of this plugin reduced incremental build times from 20s to just below 2s.

The `fork-ts-checker-webpack-plugin` also checks for any linting errors. It supports tslint and eslint and will reuse their respective config files.

### Webpack Config reused

![Webpack & Typescript]({{ "/assets/img/ATypescriptStack_webpack_config_reused.svg" | absolute_url }})

The webpack configuration for our projects is reused in several places. We use the `webpack-cli` to run the webpack build from the command line, e.g. on our build server. The `webpack-dev-server` is used during development to host the app that is created and continuously provide the latest compiled source bundle for rapid development cycles. Finally, we use Karma to run unit tests against our code. The javascript specs are compiled and packaged using the same webpack config.

Both the webpack dev server as well as karma work in watch mode and profit from the improved transpilation/compilation time described above.

### Webpack & VS Code

![Webpack & Typescript]({{ "/assets/img/ATypescriptStack_webpack_VSC.svg" | absolute_url }})

Remember that we are using npm scripts to execute common commands as tasks in VS code. This is also true for running `webpack-cli`, `webpack-dev-server` or karma. It would be nice to see the feedback and problems from those tools directly inside VS code. By defining a proper problem matcher you can integrate the error/warning messages resulting from the compilation in webpack. As all of the tools deliver the same output resulting from the webpack typescript compilation you can use the same problem matcher for all tasks.

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
```

We assume here the scripts `build` and `run` have been configured in the `package.json` similar to this:
```json
"scripts": {
    "build": "webpack-cli --config webpack.Debug.js",
    "start": "webpack-dev-server --https --config webpack.Debug.js"
    // ... more scripts
}
```

### Webpack fonts/assets

The fonts and assets you need for your project can also be integrated into the webpack build process. All you need is one or two additional plugins that will load the required assets and integrate them into your output package.

## Summary

In this post we discussed how to setup a complete development environment with VS Code, npm, webpack and typescript. The setup integrates dev tasks via npm scripts and VS code tasks into the editor. It keeps our builds fast by dividing the typescript compilation in two asynchronously running steps namely transpilation and type checking.

Have a look at the complete thing [here]({{ "/assets/img/ATypescriptStack.svg" | absolute_url }}).