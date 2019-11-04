---
layout: post
title: A Workable React Dev Stack (1) - Frameworks and Architecture
tags: React TypeScript Karma Webpack Test Unit Jasmine MaterialUI Rfluxx
---

Setting up a React app from scratch can be daunting task. Today I will present the stack that we are currently using in our projects and how all the pieces fit together.

Introduction
---

Today there are many helpers for setting up your react app. Take for example [create-react-app](https://create-react-app.dev) which sets up a complete react app with one command. However there are currently so many different possibile combinations of frameworks that it becomes hard to decide whether any possible generator fits your needs. Therefore, I want to describe our current stack and how all the different frameworks fit together. That should give you an overview of what is needed for a workable react stack.

Generate example app
---

Codifying things is good as they become repeatable and can be anaylized easier. The stack we will look at is codified in the [yeoman](https://yeoman.io/) generator [generator-rfluxx](https://www.npmjs.com/package/generator-rfluxx).

You can install both by typing:

    npm i -g yo generator-rfluxx

When both is installed create a folder for your app and type

    yo rfluxx

Answer the questions (just answer 'yes' for all of them) and after some time your app will be ready for use.
Check that it works by running

    npm run start

and go to `localhost:8080/Home`. You should see small welcome screen.

The components of the stack
---

The stack uses the following tools for doing its job

- __Visual Studio Code__: A great IDE for many use cases. And it is free. Very good for typescript development. Fast to install, start and update. Easy to extend. The list can probably go on for while. Only drawback is its slow intellisense in our project.
- __Npm__: When developing a serious project you arguably need a package manager that manages the hundreds or thousands of dependencies you need. Npm is one of the standard package managers for the Typescript community and our current choice.
- __Typescript (compiler)__: We write all our code in typescript, because the compilation process makes finding inconsistencies after refactorings/changes much, much easier.
- __Webpack__: Many other languages have a compiler and build system build in. In JS land you need to find a tool that gathers all the files/modules you have created including your assets and produces something you can deploy onto a server of your choice.
- __Webpack Dev Server__: Webpack provides a development server that serves our project locally on the fly with automatic recompilation.
- __Karma__: Karma is a unit test runner that integrates with many different browsers, test frameworks and packaging tools which in our case is webpack.

These tools already provide a good basis for developing, building and testing any javascript app. But you usually need more features like an implementation of the flux pattern so commonly used in react, an approach to routing inside your app, a framework to implement your unit tests, etc.

Therefore, in addition we use the following libraries:
- __jasmine__: This is the unit test framework we currently apply to write the actual unit tests.
- __react__: Naturally, it is a react stack that we describe here.
- __rfluxx__: This library implements the flux pattern so popular in react.
- __rfluxx-react__: React integration for the rfluxx library.
- __rfluxx-routing__: A routing library integrating with the rfluxx library.
- __rxjs__: This is the main dependency of the rfluxx library and a very flexible implementation of the Observer pattern.
- __material-ui__: Material-UI is not just a design approach for UI in apps, there are also implementations of it for different frameworks. material-ui is a UI component library for react that offers a good approach to styling (using JSS), modular components that can easily be adapted to your use case and a very strong theming support. It even targets development with typescript and offers docs/guides specifically for this usecase.  

Learning all of these things one by one is doable but finding a good combination of all of them really is a bigger task. Therefore, I do not want to go into each tool/lib separately but rather comment on the combination of the different tools.

IDE
---

### Npm & VS Code

VS Code comes with a very powerful task execution engine. Usually you also want to execute tasks on command lines e.g. when running your CI. Therefore, it is very nice that VS code can autodetect all npm scripts in your project (even recursively). Executing an npm script becomes easy as pie with this feature. You just have to open your task execution input and start typing `npm`.

Alternatively you can just call `npm run <scriptname>` on any command line to execute the task.

In our projects we therefore have npm scripts for all major tasks that we need like building with webpack, starting webpack dev server, starting karma test runner, linting with tslint etc.

Build Process
---

### Webpack Packaging

Webpack by itself is already great. Structure your code however you want in as many files as you want. Webpack will gather them as you are used to from other languages like C# or Java. It will produce a consistent output package that you can serve from any webserver. 

### Webpack Dev Server

The webpack dev server makes implementing changes very easy. You just code, the dev server gathers, compiles, and packages. After a short time you just open a webpage to check that your code does what you expect. No intervention necessary anymore.

### Webpack & Karma

And packaging your runtime app isn't even the whole story. In addition you need to package your unit tests for execution. With Karma you can just reuse the same webpack config you are already using for packaging your app.

### Webpack & Typescript

When using typescript there is the additional complexity of compiling your code before packaging it up for the webserver. And while developing you want a direct mapping of the running java script code to your original typescript source code. Here webpack also serves your need. With the ts-loader plugin you can compile and package your code in one go with webpack. The source map files will be generated as expected between input (typescript) and output (java script) and make your debugging experience much more pleasant.

### Webpack fonts/assets

The fonts and assets you need for your project can also be integrated into the webpack build process. All you need is one or two additional plugins that will load the required assets and integrate them into your output package.

### Incremental Builds

React Components
---

### Material-UI

When developing react components we stick to the [material-ui framework](https://material-ui.com). It offers catalogue of many basic components. But the more important reason to use it is its convincing styling library. It is based on the `jss` package that allows for styling in java script code. This once and for all time solves the problem of interacting global styles. There is also a typescript integration for `jss`. Other nice features of the framework are:

- all components are restylable on a granular level with one separate class per html element in the component. Very nice is also its
- theming support that offers centralized management of colors, fonts and spacing (available in the jss styles). Theming also allows to override the styles for all contained components.
- very good documentation with many code examples on par (or better) than some paid alternatives
- actively developed

### Flux Pattern

We currently use a home grown framework to implement the flux pattern. The library we use is called rfluxx. You can read more about the motivation in my other blog post ([RFluXX 1]({% post_url 2018-3-19-RFluXX-Flux-in-Typescript-with-Rx %}), [RFluXX 2]({% post_url 2018-3-24-RFluXX-Middleware %}). Usually we also apply the routing framework of RFluXX.

### Standardized component template

The following is a standardized template for functional typescript component with 

