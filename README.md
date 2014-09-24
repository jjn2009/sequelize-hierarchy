# sequelize-hierarchy.js

# Nested hierarchies for Sequelize

## What's it for?

Relational databases aren't very good at dealing with nested hierarchies.

Examples of hierarchies are:

* Nested folders where each folder has many subfolders, those subfolders themselves have subfolders, and so on
* Categories and sub-categories e.g. for a newspaper with sections for different sports, Sports category splits into Track Sports and Water Sports, Water Sports into Swimming and Diving, Diving into High Board, Middle Board and Low Board etc
* Tree structures

To store a hierarchy in a database, the usual method is to give each record a ParentID field which says which is the record one level above it.

Fetching the parent or children of any record is easy, but if you want to retrieve an entire tree/hierarchy structure from the database, it requires multiple queries, recursively getting each level of the hierarchy. For a big tree structure, this is a lengthy process, and annoying to code.

This plugin for [Sequelize](http://sequelizejs.com/) solves this problem.

## Current status

API is stable and works with MySQL.
Testing and re-coding to work with other DB dialects supported by Sequelize (Postgres, SQLite etc) would be a welcome contribution.

Requires recent version of Sequelize v2.0.0 development branch (after 23 Sept 2014, when universal hooks were introduced).

## Usage

Example:

	var Sequelize = require('sequelize');
	require('sequelize-hierarchy')(Sequelize);
	
	var sequelize = new Sequelize('database', 'user', 'password');
	
	var folder = sequelize.define('folder', name: { type: Sequelize.STRING });
	folder.isHierarchy();

`folder.isHierarchy()` does the following:

* Adds a column `parentId` to Folder model
* Adds a column `hierarchyLevel` to Folder model (which should not be updated directly)
* Creates a new table `foldersAncestors` which contains the ancestry information
* Creates hooks into standard Sequelize methods (create, update, destroy etc) to automatically update the ancestry table as details in the folder table change
* Creates hooks into Sequelize's `Model#find()` and `Model#findAll()` methods so that hierarchies can be returned as javascript object tree structures

The column and table names etc can be modified by passing options to `.isHierarchy()`. See `modelExtends.js` in the code for details.

Examples of getting a hierarchy structure:

	// get entire hierarchy as a flat list
	folder.findAll().then(function(results) {
		// results = [
		//	{ id: 1, parentId: null, name: 'a' },
		//	{ id: 2, parentId: 1, name: 'ab' },
		//	{ id: 3, parentId: 2, name: 'abc' }
		// ]
	})

	// get entire hierarchy as a nested tree
	folder.findAll({ hierarchy: true }).then(function(results) {
		// results = [
		//	{ id: 1, parentId: null, name: 'a', children: [
		//		{ id: 2, parentId: 1, name: 'ab', children: [
		//			{ id: 3, parentId: 2, name: 'abc' }
		//		] }
		//	] }
		// ]
	})
	
	// get all the descendents of a particular item
	folder.find({ where: { name: 'a' }, include: { model: folder, as: 'descendents', hierarchy: true } }).then(function(result) {
		// result =
		// { id: 1, parentId: null, name: 'a', children: [
		//		{ id: 2, parentId: 1, name: 'ab', children: [
		//			{ id: 3, parentId: 2, name: 'abc' }
		//		] }
		// ] }
	})
	
	// get all the ancestors (i.e. parent and parent's parent and so on)
	folder.find({ where: { name: 'abc' }, order: [ [ 'hierarchyLevel' ] ] }).then(function(result) {
		// results = [
		//	{ id: 1, parentId: null, name: 'a' },
		//	{ id: 2, parentId: 1, name: 'ab' }
		// ]
	})

The forms with `{ hierarchy: true }` are equivalent to using `folder.findAll({ include: { model: folder, as: 'children' } })` except that the include is recursed however deeply the tree structure goes.

Accessors are also supported:

	thisFolder.getParent()
	thisFolder.getChildren()
	thisFolder.getAncestors()
	thisFolder.getDescendents()

To build the hierarchy data on an existing table, or if hierarchy data gets corrupted in some way (e.g. by changes to parentId being made directly in the database not through Sequelize), you can rebuild it with:

	folder.rebuildHierarchy()

## Tests

Use `npm test` to run the tests.
Requires a database called 'sequelize_test' and a db user 'sequelize_test', password 'sequelize_test'.

## Changelog

0.0.1

* Initial release

0.0.2

* Implemented with hooks

0.0.3

* Removed unused dependency sequelize-transaction-promises
* Check for illegal parent ID in updates
* `Model#rebuildHierarchy()` function
* Bug fix for defining through table
* Hooks for `Model.find()` and `Model.findAll()` to convert flat representations of hierarchies into tree structures

0.0.4

* Transactionalised if operations to alter tables are called within a transaction
* Do not pass results back from hooks (not needed by Sequelize)
* Replaced usage of Promise.resolve().then() with Promise.try()
* Changed uses of Utils._.str.capitalize() to Utils.uppercaseFirst() to reflect removal of underscore.string dependency from sequelize
* Adjusted capitalization to reflect that model names and tables names are no longer capitalized
* Changed 'childs' to 'children' as pluralization now performed through Inflection library which plururalizes "child" correctly

0.0.5 (First working version ready for use)

* bulkCreate and bulkUpdate use hooks instead of shimming
* Dependency on shimming module removed
* Added tests for main functions
* Bug fixes

0.0.6

* `Model#find()` hooks made universal to allow e.g. `Person.findAll({ include: { model: Department, include: { model: Department, as: 'descendents', hierarchy: true } } })`
* Tests for find and accessors (`Model#getDescendents()` etc)

## TODO

* Change behaviour of `Model#find()` with option `hierarchy` to include descendents automatically.
* Add other creation methods (e.g. createChild, createParent etc)
* Create more efficient function for bulkCreate (+ alter sequelize bulkCreate to do single multi-row insertion?)

## Known issues

* beforeUpdate hook function assumes that item has not been updated since it was originally retrieved from DB
* All hooks should be within transactions
