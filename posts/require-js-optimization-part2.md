---
title: Require.js optimizer
description: This is RequireJS optimizer tutorial, second one in the RequireJs series.
date: 2013-11-25
tags:
  - javascript
  - require-js
layout: layouts/post.njk
permalink: "require-js-optimization-part2/index.html"
---

This article covers the RequireJS optimizer (r.js) for bundling and minifying modules. For background on dependency management, see [Part 1](/require-js-dependency-management-part1/).

## Introduction

The RequireJS optimizer uses Node.js to bundle and minify your modules. Install Node.js from the [official website](http://nodejs.org/#download) before proceeding.
 
## Setting Up the Optimizer

Create a build configuration file at `webapp/build/build.js`:

```js
|-[wepapp]
|--- [build]
|----- r.js
|----- build.js
|----- build.single.js
|--- [js]
|------ [app]
|------ [lib]
|--------- jquery.js
|----- app.js
|----- app-built.js*
|-index.html
|-readme.md
```
 
Run the optimizer with:

```bash
node r.js -o build.js
```

**Example 1: Single Module Optimization**

To bundle `app.js` and all its dependencies into one file, configure the `modules` array in `build.js`:
 
```js
{
    baseUrl: "js/lib",
    appDir: "..",
    dir: "dist",

    modules: [
        { name: "app" }
    ],

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
Key options:
- **appDir** – root directory of your application (relative to build.js)
- **dir** – output directory for optimized files

Console output:
 
```js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/build/r.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/category.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/item.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/specialItem.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/jquery.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/require.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/text.js

js/app.js
----------------
js/lib/jquery.js
js/app/category/category.js
js/app/category/item.js
js/app/category/specialItem.js
js/app.js
```
 
All dependencies are minified and concatenated into a single file.

Note that individual modules retain their original dependencies. For example, `specialItem.js` still references `item.js` rather than including it:
 
```js
define(["./item"], function (Item) {
    var SpecialItem = function (itemName) {
            this.color = "Default color", this.weigth = "Default weigth", Item.call(this, itemName)
        };
    return SpecialItem.prototype = new Item, SpecialItem.prototype.constructor = SpecialItem, SpecialItem
})
```
 
**Example 2: Multiple Module Optimization**

Add `specialItem.js` to the modules array:
 
```js
{
    baseUrl: "js/lib",
    appDir: "..",
    dir: "dist",

    modules: [
        { name: "app" },
        { name: "app/category/specialItem" }
    ],

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
Console output:
 
```js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/build/r.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/category.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/item.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/specialItem.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/jquery.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/require.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/text.js

js/app.js
----------------
js/lib/jquery.js
js/app/category/category.js
js/app/category/item.js
js/app/category/specialItem.js
js/app.js

js/app/category/specialItem.js
----------------
js/app/category/item.js
js/app/category/specialItem.js
```
 
Now `item.js` is bundled with `specialItem.js`: 
 
```js
define("app/category/item", [], function () {
    var Item = function (itemName) {
            this.name = itemName
        };
    return Item.prototype.getItemName = function () {
        return this.name
    }, Item
}), define("app/category/specialItem", ["./item"], function (Item) {
    var SpecialItem = function (itemName) {
            this.color = "Default color", this.weigth = "Default weigth", Item.call(this, itemName)
        };
    return SpecialItem.prototype = new Item, SpecialItem.prototype.constructor = SpecialItem, SpecialItem
})
```
 
## Single File Build

To bundle everything into one file, create `build.single.js`:
 
```js
{
    baseUrl: "../js/lib",
    name: "../app",
    out: "app-built.js",

    paths: {
        app: '../app',
        jquery: 'jquery',
    }
}
```

Key options:
- **name** – entry module to bundle with dependencies
- **out** – output filename

Console output:
 
```js
 Tracing dependencies for: ../app
 Uglifying file: C:/vlada/practice/require/webapp/build/app-built.js
 
 C:/vlada/practice/require/webapp/build/app-built.js
 ----------------
 C:/vlada/practice/require/webapp/js/lib/jquery.js
 C:/vlada/practice/require/webapp/js/app/category/category.js
 C:/vlada/practice/require/webapp/js/app/category/item.js
 C:/vlada/practice/require/webapp/js/app/category/specialItem.js
 C:/vlada/practice/require/webapp/js/lib/../app.js
```
 
All modules are bundled into `webapp/build/app-built.js`.

## Complete Build Configurations
 
`build.js`
```js
{
    baseUrl: "js/lib",
    appDir: "..",
    dir: "dist",

    modules: [
        { name: "app" },
        { name: "app/category/specialItem" }
    ],

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
`build.single.js`
```js
{
    baseUrl: "../js/lib",
    name: "../app",
    out: "app-built.js",

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
## References

1. http://requirejs.org/docs/optimization.html
2. https://github.com/jrburke/r.js/blob/master/build/example.build.js
