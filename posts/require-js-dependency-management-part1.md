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

This is an introductory RequireJS tutorial in the series (<a target="_blank" href="/require-js-dependency-management-part1/">part1</a>, <a target="_blank"  href="/require-js-optimization-part2/">part2</a>).
 
## Table of contents:
1. <a href="#t0">Introduction</a>
2. <a href="#t1">Basic structure</a>
3. <a href="#t2">Prototypal inheritance and Require.js</a>
 
### <a name="t0" id="t0">Introduction</a>

Before or after reading this article I strongly recommend reading the documentation on [RequireJS website](http://requirejs.org/docs/start.html). James Burke did a great job here.
 
Require.js is a dependency management and async script loading tool(AMD library). What that means? AMD stands for Asynchronous Module Definition.
 
RequireJS is asynchronous which means that you can do non-blocking and parallel fetch of your javascript files.
 
RequireJS is built around the Module pattern. Modern web applications tend to have fairly complex frontends. Module pattern should improve the maintainability of our bloated javascript code. Remember those giant javascript files and svn merge conflict hell on your last project? Javascript code should consist of smaller components enforcing separation of concerns and avoiding globals since modules are wrapped by closures. 
 
RequireJS is well defined and standardized. While we wait for ES-Harmony to knock on our doors, libraries like RequireJS are giving us hint on how we should structure our applications. Require.js works in current browsers.
 
### <a name="t1" id="t1">Basic structure</a>
 
**STEP 1 - How to structure RequireJS project**

This is our sample RequireJS project structure.

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

Build scripts and RequireJS optimization tool r.js are inside `[build]` directory.
 
Application code reside in `[webapp/js/app]` directory. This is place where you should place all of your /Model/View/Router/Template code.
 
Libraries like jQuery, Backbone, Handlebars and others are inside `[webapp/js/lib]` directory.
 
Application config and main entry point is `[webapp/js/app.js]`
 
#### STEP 2 - How to include RequireJS

Your `[webapp/index.html]` should look like this:
 
```js
<html>
<head></head>
<body>
	<div id="test">Some sample content</div>
 	<script data-main="js/app.js" src="js/lib/require.js"></script>
</body>
</html>
```
 
Attribute `[data-main="js/app.js"]` is entry point of our application. This means that RequireJS will first load `[js/app.js]` after initialization.
 
**STEP 3 - How to configure RequireJS**

Open your `[webapp/js/app.js]` file.
 
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
 
`[baseUrl: 'js/lib']` - RequireJS loads code realative to directory specified in baseUrl. By default baseUrl is set to the same directory as data-main. If you dont specify data-main attribute and baseUrl property is not present in RequireJS config, than default path is directory that contains html page which include RequireJS library. 
[base url](http://requirejs.org/docs/api.html#config-baseUrl)
 
Attribute `[paths]` [paths config](http://requirejs.org/docs/api.html#config-paths)
 
**STEP 4 - How to use requre() and define() in RequireJS**

require() and define() are most important concepts in RequireJS.
 
**define()** - Used for module definition.
Consists of module wrapper, dependency list and definition function.
 
```js
define(['dependency1', ['dependency2', ['dependency3'], function(dependency1, dependency2, dependency3) {
 	var Module = function(value) {
 		this.property = value;
 	}
 	return (Module);
 });
```
  
**require()** - Used for dependency loading
Consists of public api, dependency list, callback function
 
```js
 requirejs(['dependency1', ['dependency2', ['dependency3'], function(dependency1, dependency2, dependency3) {
 	// You can use your imported module here
 	var instance = dependency1;
 	var constructorFunction = new dependency2();
 });
```
 
### <a name="t2" id="t2">Prototypal inheritance and Require.js</a>
 
Skip to <a href="#source">complete source code</a>.
 
With RequireJS you can organize your modules in separate files with ease.
 
Now we are going to add 3 object definitions to our project: Category, Item and Specialized Item.

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

With RequireJS you can provide function or object in return statement. We are going to use this feature to retrieve constructor functions for our object.
 
With RequireJS you can import and use external dependencies inside your module. We are going to use this feature to achieve prototypal inheritance in RequireJS.
 
Category is collection of our Items: `[webapp/js/app/category/category.js]`.
 
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
  
Item is base object: `[webapp/js/app/category/item.js]`.
 
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
 
SpecialItem is object that extends Item. In order to extend Item object, we must import Item as a dependency<strong>['./item'].</strong>
Now Item object is available for use in our new SpecialItem module: `[webapp/js/app/category/specialItem.js]`.
 
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
 
 <h1><a name="source" id="source">Source code listing</a></h1>
 
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
 
In next part of Require.js series I will talk about r.js RequireJS optimization and build tool.
 
[Follow me on Twitter](https://twitter.com/#!/svlada)
 
**References**

1. http://addyosmani.com/writing-modular-js/
2. http://msdn.microsoft.com/en-us/magazine/hh227261.aspx
3. http://tomdale.net/2012/01/amd-is-not-the-answer/
4. http://tagneto.blogspot.com/2012/01/reply-to-tom-on-amd.html