---
layout: post
title: VS Code Snippet Extensions
tags: Visual Studio Code Extension Snippet
---

Creating VS Code extensions is easier than one might think, navigating the docs is harder. Here is a short summary on how to provide snippets as a VS Code extensions.

## The Docs

VS Code documentation regarding extensions is good with many examples. But there are a lot of extension points. Providing snippets works via the [Declarative Language Features](https://code.visualstudio.com/api/language-extensions/overview#declarative-language-features). 

The extension point is called `contributes.snippets` and it is described [here](https://code.visualstudio.com/api/language-extensions/snippet-guide). An overview over all extension points can be found [here](https://code.visualstudio.com/api/references/contribution-points).

They even have an example for this use case on github: https://github.com/Microsoft/vscode-extension-samples/tree/master/snippet-sample

## The Implementation

You basically only need two files:

- `package.json`
- `snippets.json`

The `snippets.json` is a default snippet file that you can also use for local user defined snippets.

The `package.json` looks similar to this:

```json
{
	"name": "rfluxx-code-snippets",
	"displayName": "Rfluxx VS code snippets",
	"description": "This package provides VS code snippets that make creating rfluxx components easier",
	"version": "1.0.0",
	"publisher": "JochenGruen",
	"repository": {
		"type": "git",
		"url": "https://github.com/Useurmind/RFluXX.git"
	},
	"engines": {
		"vscode": "^1.37.0"
	},
	"categories": [
		"Snippets"
	],
	"contributes": {
		"snippets": [
			{
				"language": "typescript",
				"path": "./snippets.json"
			}
		]
	}
}
```

The `publisher` field needs to be configured to the id of the publisher you will create during setup of `vsce` (see below). The rest should be relative straight forward when you have read the documentation.

## Publishing 

Publishing the package is done via the `vsce` cli tool which can be installed via npm. You will need an Azure DevOps account to publish your extension to the marketplace.

The details are documented here: https://code.visualstudio.com/api/working-with-extensions/publishing-extension

Once registered and setup publishing is as simple as executing `vsce publish 1.0.0` in your extension folder.

