---
layout: post
title: "Kendo DataSource and Azure Mobile Services."
date: 2014-01-07 -0800
comments: true
categories: [code]
---

The [DataSource](http://demos.kendoui.com/web/datasource/index.html) is one of the most powerful framework components of Kendo UI. It is the best way to get data in and out of the many widgets that the Kendo suite offers.
The Kendo team has done a great job at explaining how to [get started](http://docs.kendoui.com/getting-started/framework/datasource/overview) using it, [deep diving](http://www.kendoui.com/blogs/teamblog/posts/13-01-24/learning_kendo_data_datasource.aspx) into it and [integrating it with Web Api and EntityFramework](http://www.kendoui.com/blogs/teamblog/posts/12-10-25/using_kendo_ui_with_mvc4_webapi_odata_and_ef.aspx), [Backbone JS](http://www.kendoui.com/blogs/teamblog/posts/13-02-07/wrapping_a_backbone_collection_in_a_kendo_data_datasource.aspx), [Breeze JS](http://www.kendoui.com/blogs/teamblog/posts/13-02-21/breeze_js_and_the_kendo_ui_datasource.aspx), among others.

Microsoft recently released the [javascript SDK](http://msdn.microsoft.com/en-us/library/windowsazure/jj554207.aspx) for its BaaS, [Windows Azure Mobile Services (ZUMO)](http://www.windowsazure.com/en-us/develop/mobile/)

On this blog post I will explain how to use this SDK in conjunction with the Kendo DataSource.
I will use a Kendo UI Grid as example, but theoretically the DataSource can be used with many different Kendo components.

Lets start by setting up a simple Video Games Kendo Grid that supports pagination.

```javascript
$("#grid").kendoGrid(
            {
                pageable: true,
                dataSource: dataSource,
                columns: [
                            "name",
                            "developer",
                            {
                                command: [
                                  { name: "edit", text: "Edit" },
                                  { name: "destroy", text: "Delete" }
                                ]
                            }],
                toolbar: [
                    { name: "create" }
                ],
                editable: "inline"
            }
        );
```

The Kendo Grid itself is very easy to set-up. Most of the configuration necessary to work with the back-end will reside on the dataSource property.
if we want to write as little code as possible to get this working we simply need to make use of the ZUMO SDK and override the transport methods on the Kendo DataSource


```javascript
		var client = new WindowsAzure.MobileServiceClient("https:///*your ZUMO service name here*/.azure-mobile.net/", "/*Your API KEY here*/");
        var table = client.getTable("games");
    	var dataSource = new kendo.data.DataSource({
            transport: {
                read: function (options) {
                    table.includeTotalCount()
                         .read()
                         .done(options.success);
                },
                update: function (options) {
                    table.update(options.data)
                         .done(options.success);
                },
                create: function (options) {
                    var item = options.data;
                    delete item.id;
                    table.insert(item)
                         .done(options.success);
                },
                destroy: function (options) {
                    table.del(options.data)
                         .done(options.success);
                }
            },
            pageSize: 4,
            schema: {
                total: "totalCount",
                model: {
                    id: "id",
                    fields: {
                        id: { type: "number" },
                        name: { type: "string" },
                        developer: { type: "string" },
                    }
                }
            }
        });
```

That's it, we now have a Video Games Grid that loads data from our ZUMO back-end.

One caveat with this approach, though, is that the paging will be done client-side. This means that all data will be loaded on one HTTP request, which might not be ideal.
If we want to add server side paging we will have to pass the page and pageSize to the server so that it can just send the limited result set back down to the client.


```javascript
         var dataSource = new kendo.data.DataSource({
            transport: {
                read: function (options) {
                    table.skip(options.data.skip)
                         .take(options.data.take)
                         .includeTotalCount()
                         .read()
                         .done(options.success);
                },
                update: function (options) {
                    table.update(options.data)
                         .done(options.success);
                },
                create: function (options) {
                    var item = options.data;
                    delete item.id;
                    table.insert(item)
                         .done(options.success);
                },
                destroy: function (options) {
                    table.del(options.data)
                         .done(options.success);
                }
            },
            serverPaging: true,
            pageSize: 4,
            schema: {
                total: "totalCount",
                model: {
                    id: "id",
                    fields: {
                        id: { type: "number" },
                        name: { type: "string" },
                        developer: { type: "string" },
                    }
                }
            }
        });
```