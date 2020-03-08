---
title: "SharePoint List Dao with TypeScript"
date: 2014-07-12T21:12:07+01:00
draft: false
categories: ["SharePoint"]
tags: ["sharepoint-2013", "sharepoint-apps", "typescript"]
---

As I am now using TypeScript, I thought that it would be good to rewrite the [GenericList script](/posts/20130525-sharepoint-rest-api/) that I had provided a year ago.

Before to start, make sure that you’ve added TypeScript support to your Sharepoint solution, in order to do that I would suggest the reading of the following MSDN article by *Marat Bakirov*: [Creating SharePoint solutions with TypeScript](https://web.archive.org/web/20140819083450/http://blogs.msdn.com/b/mbakirov/archive/2014/06/06/10531701.aspx)

Once that is done, you can find below the TypeScript code of the updated ListDao (previously GenericList).

```typescript
'use strict';

module Gobbe {
    export class ListDao {
        constructor(public listName: string) {
        }

        createItem(data: any, callback: Function) {
            $.ajax({
                url: _spPageContextInfo.webServerRelativeUrl +
                    "/_api/web/lists/getByTitle('" + this.listName + "')/items",
                type: "POST",
                contentType: "application/json;odata=verbose",
                data: data,
                headers: {
                    "accept": "application/json;odata=verbose",
                    "X-RequestDigest": $("#__REQUESTDIGEST").val()
                },
                success: () => {
                    if (callback !== undefined)
                        callback();
                },
                error: (err) => {
                    console.log(JSON.stringify(err));
                }
            });
        }

        getItems(callback: Function) {
            $.ajax(
            {
                url: _spPageContextInfo.webServerRelativeUrl +
                    "/_api/web/lists/getByTitle('" + this.listName + "')/items/",
                type: "GET",
                headers: {
                    "accept": "application/json;odata=verbose",
                },
                success: (data) => {
                    if (callback !== undefined)
                        callback(data.d.results);
                },
                error: (err) => {
                    console.log(JSON.stringify(err));
                }
            });
        }

        getItem(id: number, callback: Function) {
            $.ajax(
                {
                    url: _spPageContextInfo.webServerRelativeUrl +
                        "/_api/web/lists/getByTitle('" + this.listName + "')/items('" + id + "')",
                    type: "GET",
                    headers: {
                        "accept": "application/json;odata=verbose",
                    },
                    success: (data) => {
                        if (callback !== undefined)
                            callback(data.d);
                    },
                    error: (err) => {
                        console.log(JSON.stringify(err));
                    }
                }
            );
        }

        updateItem(id: number, data: any, callback: Function) {
            $.ajax(
                {
                    url: _spPageContextInfo.webServerRelativeUrl +
                        "/_api/web/lists/getByTitle('" + this.listName + "')/items('" + id + "')",
                    type: "POST",
                    contentType: "application/json;odata=verbose",
                    data: data,
                    headers: {
                        "accept": "application/json;odata=verbose",
                        "X-RequestDigest": $("#__REQUESTDIGEST").val(),
                        "IF-MATCH": "*",
                        "X-Http-Method": "PATCH"
                    },
                    success: () => {
                        if (callback !== undefined)
                            callback();
                    },
                    error: (err) => {
                        console.log(JSON.stringify(err));
                    }
                }
            );
        }

        removeItem(id: number, callback: Function) {
            $.ajax(
                {
                    url: _spPageContextInfo.webServerRelativeUrl +
                        "/_api/web/lists/getByTitle('" + this.listName + "')/items('" + id + "')",
                    type: "DELETE",
                    headers: {
                        "accept": "application/json;odata=verbose",
                        "X-RequestDigest": $("#__REQUESTDIGEST").val(),
                        "IF-MATCH": "*"
                    },
                    success: () => {
                        if (callback !== undefined)
                            callback();
                    },
                    error: (err) => {
                        console.log(JSON.stringify(err));
                    }
                }
            );
        }
    }
}
```

All what is left to do in order to retrieve all the items of the **Funds** list (e.g.) is:

```typescript
var fundsDao = new Gobbe.ListDao('Funds');
console.dir(fundsDao.getItems());
```