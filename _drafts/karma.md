---
layout: post
title: Karma TypeScript Unit Tests
tags: TypeScript Karma Test Unit Jasmine PhantomJS Webpack
---

Today we will dive into Unit Testing with TypeScript using Karma. Our special focus will be the ability to automate these tests with TFS. 

I will not specifically have a look at how to write good unit tests using jasmine because I currently only care about how to create a good setup that works on the dev and CI machine and is well integrated into the build pipeline.

Goals for the TypeScipt Tests
---

The main problem I faced when starting this setup was the integration with the default facilities for test results in TFS. TFS offers you a nice visual breakdown of your tests, if you provide it with the required information about the test results. The native format of TFS for these test results is the *.trx format (although there are more formats available xunit/nunit).

My goal is to find a test runner and setup with which you can:

- produce trx files
- run tests automatically and repeatedly on the developer machine
- run tests without having a browser installed (using a headless browser,  which I guess makes less problems in automation scenarios)

But lets get to work as the setup is quite involved.

Karma & Jasmine
---

The first decision is to find a good test runner to run your tests. I decided to try out the [Karma](https://karma-runner.github.io/2.0/index.html) test runner, which provides integration for 

- a lot of test frameworks 
- a lot of browsers
- a plugin that produces trx files
- runs repeatedly in the background

The default test framework for karma is jasmine. And because I currently do not want to spent a lot of time in the world of java script test frameworks I will continue to use it. Especially because Karma will make switching the framework in the same setup quite easy.

PhantomJS
---

Next was the question on how to run the tests on the build server. The problem is that on many build servers you will run in agents under service user accounts which are not allowed to open UI applications (in my job the customer has a lot of security related restrictions on their servers).

Therefore, we need a solution that runs the unit tests not in a standard browser but in headless mode.

[PhantomJS](http://phantomjs.org/) is such a browser and best is you can install it via npm. Although when you do not have Internet access from the build server you are forced to a little trick as the binaries for the browser are not actually included in the npm package. More on that later.

Project Structure
---

Usually I assume a structure that at least fulfills the default code styleguide of tools like tslint. That means all our classes are in separate source code files.

I also like to separate tests from source code. Finally I do not want to configure several tsconfig.jsons and would find it best to handle all the configuration in a single file.

That brings me to the following folder setup:

- Root
  - Scripts
    - ... (all source code)
  - Test
    - ... (all test code)
  - karma.conf.js
  - tsconfig.json
  - package.json

TypeScript Configuration
---

When configuring your TypeScript transpilation you are faced with a lot of possiblities and decisions. Personally, I have found the following settings to be very useful:

```json
{
    "compilerOptions": {
        "module": "umd",
        "moduleResolution": "node",
        "watch": false, 
        "noImplicitAny": true,
        "noImplicitReturns": true,
        "noImplicitThis": true,
        "sourceMap": true,
        "target": "es5",
        "importHelpers": false,
        "lib": [ "dom", "es6", "scripthost" ],
        "jsx": "react"
    },
    "compileOnSave": true,
    "exclude": [
        "node_modules"
    ]
}
```

My reasons for this setup are:

- ```moduleResolution: node```: works great in combination with npm and webpack
- ```watch: false```: execute just once, because we use this file on the build server too
- ```noImplicitXXX: true```: implicit stuff leads to errors, tell the compiler we don't want that
- ```sourceMap: true```: we want to havew source maps for development right
- ```target: es5```: IE11 is still only es5 compatible and things like classes (which typescript creates when targeting es6) can not be patched by shims
- ```importHelpers: false```: do not use [tslib](https://github.com/Microsoft/tslib) (actually using tslib makes sense depending on the setup, but not for our little test here)
- ```lib: [ "dom", "es6", "scripthost" ]```: we actually want to use es6 features like the default library stuff that can be patched by shims
- ```jsx: react```: we are using react in our projects and require this setting

Configure Karma & Webpack
---

Now we are faced with a lot TypeScript files that are converted into corresponding JavaScript files by the TypeScript compiler (tsc). To gather all of these together I prefer to use webpack which can handle my own code as well as libraries installed via npm quite well.

Thankfully, karma provides direct integration with webpack. But before we can configure karma we need to install quite a few npm packages:

```json
{
  "devDependencies": {
    "@types/jasmine": "^2.8.6",
    "fs": "0.0.1-security",
    "jasmine": "^3.1.0",
    "jasmine-core": "^3.1.0",
    "karma": "^2.0.0",
    "karma-chrome-launcher": "^2.2.0",
    "karma-jasmine": "^1.1.1",
    "karma-phantomjs-launcher": "^1.0.4",
    "karma-spec-reporter": "^0.0.32",
    "karma-trx-reporter": "^0.2.9",
    "karma-webpack": "^3.0.0",
    "phantomjs-prebuilt": "^2.1.16",
    "ts-loader": "^4.2.0",
    "typescript": "^2.8.1",
    "webpack": "^4.5.0",
    "webpack-cli": "^2.0.14"
  },
  "dependencies": {
    "es5-shim": "^4.5.10",
    "es6-shim": "^0.35.3"
  }
}
```

The package list tells you this:

- we use jasmine and therefore needs its typings (for tsc compilation) and the framework itself
- we use karma and integrate it with
  - chrome: for development
  - phantomjs: for build server
  - trx reporter: that generates the trx file for tfs,
  - webpack: to package our tests
- we use phantomjs as a prebuilt dependency that is automagically installed via npm
- we use ts-loader to automatically compile typescript inside the webpack process
- we need es5 and es6 shims because we want to be compatible to olde browsers (as of this writing ie11 and phantomjs do not support es6)

Now we can go on and finally configure karma and webpack (warning big file incoming):

```javascript
var webpack = require("webpack");

module.exports = function(config) {
  config.set({

    // base path that will be used to resolve all patterns (eg. files, exclude)
    basePath: '',


    // frameworks to use
    // available frameworks: https://npmjs.org/browse/keyword/karma-adapter
    frameworks: ['jasmine'],

    // add mime type for typescript so chrome will load it
    mime: {
      'text/x-typescript': ['ts','tsx']
    },

    // list of files / patterns to load in the browser
    files: [
      'Test/**/*Spec.ts'
    ],


    // list of files / patterns to exclude
    exclude: [
    ],


    // preprocess matching files before serving them to the browser
    // available preprocessors: https://npmjs.org/browse/keyword/karma-preprocessor
    preprocessors: {
      // add webpack as preprocessor
      'Test/**/*Spec.ts': [ 'webpack' ]
    },

    webpack: {
      // karma watches the test entry points
      // (you don't need to specify the entry option)
      // webpack watches dependencies

      // webpack configuration
      node: {
        fs: 'empty'
      },
      mode: "development",
      module: {
        rules: [
          {
            test: /\.tsx?$/,
            use: 'ts-loader',
            exclude: /node_modules/
          }
        ]
      },
      plugins: [
        // existing plugins go here
        new webpack.SourceMapDevToolPlugin({
          filename: null, // if no value is provided the sourcemap is inlined
          test: /\.(ts|js)($|\?)/i // process .js and .ts files only
        })
      ],
      resolve: {
        extensions: [ '.tsx', '.ts', '.js' ]
      },
      devtool: 'inline-source-map',
    },

    // test results reporter to use
    // possible values: 'dots', 'progress'
    // available reporters: https://npmjs.org/browse/keyword/karma-reporter
    reporters: ["spec", "trx"],

    trxReporter: { outputFile: 'typescript.trx', shortTestName: false },

    // web server port
    port: 9876,


    // enable / disable colors in the output (reporters and logs)
    colors: true,


    // level of logging
    // possible values: config.LOG_DISABLE || config.LOG_ERROR || config.LOG_WARN || config.LOG_INFO || config.LOG_DEBUG
    logLevel: config.LOG_INFO,


    // enable / disable watching file and executing tests whenever any file changes
    autoWatch: true,


    // start these browsers
    // available browser launchers: https://npmjs.org/browse/keyword/karma-launcher
    browsers: ["Chrome", "PhantomJS"],

    // you can define custom flags
    customLaunchers: {
      'PhantomJS_custom': {
        base: 'PhantomJS',
        options: {
          windowName: 'my-window',
          settings: {
            webSecurityEnabled: false
          },
        },
        flags: ['--load-images=true'],
        debug: true
      }
    },

    // Continuous Integration mode
    // if true, Karma captures browsers, runs the tests and exits
    singleRun: false,

    // Concurrency level
    // how many browser should be started simultaneous
    concurrency: Infinity
  })
}
```

This is quite huge, so let's break down the more out of the ordinary parts:

- Webpack will not change the name of the input which is a ts file. Chrome will reject a ts file because the content type is not correct. The following lets karma tell chrome that the ts file is actually a java script file
```javascript
mime: {
    'text/x-typescript': ['ts','tsx']
},
```
- Add the source map plugin to explicitely tell webpack to deliver source maps for our ts files (inline). If this is not set, source maps will not work correctly in karma despite having set devtool to inlinesourcemap
```javascript
plugins: [
    new webpack.SourceMapDevToolPlugin({
        filename: null, // if no value is provided the sourcemap is inlined
        test: /\.(ts|js)($|\?)/i // process .js and .ts files only
    })
],
```
- provide both command line output and output of a trx file named typescript.trx
```javascript
reporters: ["spec", "trx"],

trxReporter: { outputFile: 'typescript.trx', shortTestName: false },
```



PhantomJS File Share
---

TODO