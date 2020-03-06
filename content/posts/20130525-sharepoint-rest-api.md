---
title: "Using the REST api in a SharePoint-hosted App"
date: 2013-05-25T16:00:41+01:00
draft: false
categories: ["SharePoint"]
tags: ["rest", "sharepoint-2013", "sharepoint-apps"]
---

This week-end, I’ve written a small “static class” in JavaScript (it works a bit like a List Generic Dao) to help me make CRUD operations on my lists in Sharepoint-hosted Apps.

<!--more-->

`$ cat generic-list.js`

```js
'use strict';

var gobbe = window.gobbe || {};

gobbe.GenericList = function() {
    var createItem = function(list, data, callback) {
            $.ajax({
                url: _spPageContextInfo.webServerRelativeUrl +
                    "/_api/web/lists/getByTitle('" + list + "')/items",
                type: "POST",
                contentType: "application/json;odata=verbose",
                data: data,
                headers: {
                    "accept": "application/json;odata=verbose",
                    "X-RequestDigest": $("#__REQUESTDIGEST").val()
                },
                success: function() {
                    if (callback !== undefined)
                        callback();
                },
                error: function(err) {
                    console.log(JSON.stringify(err));
                }
            });
        },
        getItems = function(list, callback) {
            $.ajax({
                url: _spPageContextInfo.webServerRelativeUrl +
                    "/_api/web/lists/getByTitle('" + list + "')/items/",
                type: "GET",
                headers: {
                    "accept": "application/json;odata=verbose",
                },
                success: function(data) {
                    if (callback !== undefined)
                        callback(data.d.results);
                },
                error: function(err) {
                    console.log(JSON.stringify(err));
                }
            });
        },
        getItem = function(list, id, callback) {
            $.ajax({
                url: _spPageContextInfo.webServerRelativeUrl +
                    "/_api/web/lists/getByTitle('" + list + "')/items('" + id + "')",
                type: "GET",
                headers: {
                    "accept": "application/json;odata=verbose",
                },
                success: function(data) {
                    if (callback !== undefined)
                        callback(data.d);
                },
                error: function(err) {
                    console.log(JSON.stringify(err));
                }
            });
        },
        updateItem = function(list, id, data, callback) {
            $.ajax({
                url: _spPageContextInfo.webServerRelativeUrl +
                    "/_api/web/lists/getByTitle('" + list + "')/items('" + id + "')",
                type: "POST",
                contentType: "application/json;odata=verbose",
                data: data,
                headers: {
                    "accept": "application/json;odata=verbose",
                    "X-RequestDigest": $("#__REQUESTDIGEST").val(),
                    "IF-MATCH": "*",
                    "X-Http-Method": "PATCH"
                },
                success: function() {
                    if (callback !== undefined)
                        callback();
                },
                error: function(err) {
                    console.log(JSON.stringify(err));
                }
            });
        },
        removeItem = function(list, id, callback) {
            $.ajax({
                url: _spPageContextInfo.webServerRelativeUrl +
                    "/_api/web/lists/getByTitle('" + list + "')/items('" + id + "')",
                type: "DELETE",
                headers: {
                    "accept": "application/json;odata=verbose",
                    "X-RequestDigest": $("#__REQUESTDIGEST").val(),
                    "IF-MATCH": "*"
                },
                success: function() {
                    if (callback !== undefined)
                        callback();
                },
                error: function(err) {
                    console.log(JSON.stringify(err));
                }
            });
        }

    return {
        createItem: createItem,
        getItems: getItems,
        getItem: getItem,
        updateItem: updateItem,
        removeItem: removeItem,
    }
}();
```

Using the code above, it is now easier for me to create/read/update/delete items in any lists.

You’ll find a few samples below :

`$ cat generic-list-examples.js`

```js
// List name
var list = "Students";

// Create 
gobbe.GenericList.createItem(list,
    JSON.stringify({
        '__metadata': {
            'type': 'SP.Data.' + list + 'ListItem'
        },
        'Name': 'John',
        'Surname': 'Doe',
        'School': 'Microsoft Academic'
    }),
    function() {
        alert("Student created");
    }
);

// Read all
gobbe.GenericList.getItems(list, function(results) {
    for (var i = 0; i < results.length; i++) {
        alert(results[i].ID + " : " + results[i].Title);
    }
});

// Read one
gobbe.GenericList.getItem(list, 1, function(result) {
    alert(result.ID + " : " + result.Title);
});

// Update
gobbe.GenericList.updateItem(list, 1,
    JSON.stringify({
        '__metadata': {
            'type': 'SP.Data.' + list + 'ListItem'
        },
        'Name': 'John',
        'Surname': 'Doe',
        'School': 'Microsoft'
    }),
    function() {
        alert("Student updated");
    }
);

// Delete
gobbe.GenericList.removeItem(list, 1, function() {
    alert("Item deleted.");
});
```

I know there is still place for improvement but it’s a start !