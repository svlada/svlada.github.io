---
title: Require.js dependency management
description: This is introductory RequireJS tutorial in RequireJs series.
date: 2013-11-23
tags:
  - javascript
  - require-js
layout: layouts/post.njk
permalink: "require-js-dependency-management-part1/index.html"
---

This article is the first part of a two-part series on RequireJS dependency management. In the second part, we'll take a closer look at the [RequireJS optimizer tool](/require-js-optimization-part2/).

## Introduction

Before (or after) reading this article, I strongly recommend checking out the official [RequireJS documentation](http://requirejs.org/docs/start.html). James Burke has done an excellent job there.

RequireJS is a dependency management and asynchronous script loading tool—an AMD (Asynchronous Module Definition) library.

What does that mean? In practice, AMD allows you to load JavaScript files asynchronously, so they don't block the page, and dependencies are fetched in parallel.

RequireJS builds on the Module pattern, which is especially useful for modern web applications with complex frontends. Instead of dealing with giant, monolithic JavaScript files and merge conflict nightmares, you can break code into smaller, focused modules. This enforces separation of concerns and avoids polluting the global namespace, since modules are wrapped in closures.

RequireJS is well-defined and standardized. While we wait for ES Harmony (ES6 modules) to become fully mainstream, libraries like RequireJS give us an early hint of how to structure applications more cleanly and maintainably.

## Basic structure
 
**Step 1 – Structuring a RequireJS Project**

A typical RequireJS project might look like this:

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

- **build/** – contains build scripts and the RequireJS optimization tool (r.js).
- **webapp/js/app/** – holds the application code. Place all of your Model, View, Router, and Template files here.
- **webapp/js/lib/** – contains third-party libraries such as jQuery, Backbone, Handlebars, and others.
- **webapp/js/app.js** – the application configuration and main entry point.
 
**Step 2 – Including RequireJS**

Your `webapp/index.html` should include RequireJS and point to your application's entry point (`app.js`):
 
```js
<html>
<head></head>
<body>
	<div id="test">Some sample content</div>
 	<script data-main="js/app.js" src="js/lib/require.js"></script>
</body>
</html>
```
 
- **src="js/lib/require.js"** – loads the RequireJS library.
- **data-main="js/app"** – tells RequireJS to load `app.js` (without the `.js` extension) as the entry point.
 
**Step 3 – Configuring RequireJS**

Open your `webapp/js/app.js` file and define the RequireJS configuration:
 
```js
 requirejs.config({
 	baseUrl: 'js/lib',
 	paths: {
 		app: '../app',
 		jquery: 'jquery'
 	}
 
 });
 
 requirejs(['jquery'], function($) {
 	console.log($('#test'));
 });
```
 
- **baseUrl** – sets the base directory for your modules. RequireJS loads code relative to this directory. [Read more](http://requirejs.org/docs/api.html#config-baseUrl)
- **paths** – maps library names to their file locations. [Read more](http://requirejs.org/docs/api.html#config-paths)
- **require([...])** – loads the main application module and starts the app.
 
**Step 4 – Using require() and define() in RequireJS**

The two most important concepts in RequireJS are `define()` and `require()`.

- **define()** – used to define a module. A module definition includes:
  1. A list of dependencies
  2. A definition function that receives those dependencies as arguments

Example:
 
```js
define(['dependency1', ['dependency2', ['dependency3'], function(dependency1, dependency2, dependency3) {
 	var Module = function(value) {
 		this.property = value;
 	}
 	return (Module);
 });
```
  
- **require()** – Used for Dependency Loading

The `require()` function loads dependencies and executes a callback once they're available. It consists of:
  1. A list of dependencies
  2. A callback function that receives them as arguments

Example:

```js
requirejs(['dependency1', 'dependency2', 'dependency3'], function(dependency1, dependency2, dependency3) {
	// You can use your imported modules here
	var instance = dependency1;
	var constructorFunction = new dependency2();
});
```
 
## Prototypal Inheritance with RequireJS

It's recommended to keep each module in a separate file on disk. Each module should return a function or an object that can be used for constructing instances.

In this example, we'll add three object definitions to our project:
• Category
• Item
• SpecializedItem

```js
 |-[wepapp]
 |--- [build]
 |----- r.js
 |----- build.js
 |----- build.single.js
 |--- [js]
 |------ [app]
 |--------- [category]
 |------------ category.js
 |------------ item.js
 |------------ specializedItem.js
 |------ [lib]
 |--------- jquery.js
 |----- app.js
 |----- app-built.js*
 |-index.html
 |-readme.md
```
 
Here's a clear RequireJS module definition for the Category:

```js
 "use strict";

 define(function() {

 	var Category = function() {
 		this.name = 'Default name';
 		this.category = 'Default category';
 		this.items = [];
 	}

 	Category.prototype.showItems = function() {
 		for (var i = 0, len = this.items.length; i < len; i++) {
 			console.log(this.items[i]);
 		}
 	}

 	Category.prototype.addItem = function(item) {
 		this.items.push(item);
 	}

 	return (Category);

 });
```
  
Here's a clean RequireJS module definition for the Item base object:

```js
 "use strict";

 define(function() {

 	var Item = function(itemName) {
 		this.name = itemName;
 	}

 	Item.prototype.getItemName = function() {
 		return this.name;
 	}

 	return (Item);

 });
```
 
Here's the SpecialItem module defined with RequireJS, extending Item via prototypal inheritance:

```js
 "use strict";

 define(['./item'], function(Item) {

 	var SpecialItem = function(itemName) {
 		this.color = 'Default color';
 		this.weigth = 'Default weigth';
 		Item.call(this, itemName);
 	}

 	SpecialItem.prototype = new Item;
 	SpecialItem.prototype.constructor = SpecialItem;

 	return (SpecialItem);

 });
```
 
## Source code listing
 
 `[webapp/js/app.js]`
 
```js
 requirejs.config({
 	baseUrl: 'js/lib',
 	paths: {
 		app: '../app',
 		jquery: 'jquery'
 	}
 });
 
 requirejs(['jquery', 'app/category/category', 'app/category/item', 'app/category/specialItem'],
 function($, Category, Item, SpecialItem) {
 	var c1 = new Category();
 	c1.items.push(new Item('RequireJS in Action'));
 	c1.items.push(new Item('Javascript Good parts'));
 	c1.items.push(new Item('Some book'));
 	c1.items.push(new SpecialItem('Special promotion book'));
 	c1.showItems();
 	console.log(c1);
 });
```
 
 `[webapp/js/app/category/category.js]`
 
```js
 "use strict";
 
 define(function() {
 	
 	var Category = function() {
 		this.name = 'Default name';
 		this.category = 'Default category';
 		this.items = [];		
 	}
 
 	Category.prototype.showItems = function() {
 		for (var i = 0, len = this.items.length; i < len; i++) {
 			console.log(this.items[i]);
 		}
 	}
 
 	Category.prototype.addItem = function(item) {
 		this.items.push(item);
 	}
 
 	return (Category);
 
 });
```
 
`[webapp/js/app/category/item.js]`
 
```js
 "use strict";
 
 define(function() {
 
 	var Item = function(itemName) {
 		this.name = itemName;
 	}
 
 	Item.prototype.getItemName = function() {
 		return this.name;
 	}
 
 	return (Item);
 
 });
```
 
`[webapp/js/app/category/specialItem.js]`
 
```js
 "use strict";
 
 define(['./item'], function(Item) {
 
 	var SpecialItem = function(itemName) {
 		this.color = 'Default color';
 		this.weigth = 'Default weigth';
 		Item.call(this, itemName);
 	}
 
 	SpecialItem.prototype = new Item;
 	SpecialItem.prototype.constructor = SpecialItem;
 
 	return (SpecialItem);
 
 });
```

The next part of this RequireJS series will dive deeper into the r.js optimization tool.

**References**

1. http://addyosmani.com/writing-modular-js/
2. http://msdn.microsoft.com/en-us/magazine/hh227261.aspx
3. http://tomdale.net/2012/01/amd-is-not-the-answer/
4. http://tagneto.blogspot.com/2012/01/reply-to-tom-on-amd.html