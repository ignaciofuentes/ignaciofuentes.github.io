---
layout: post
title: "Kendo DataSource and Azure Mobile Services."
date: 2014-01-07 -0800
comments: true
categories: [code]
---

The DataSource is one of the most powerful framework components of Kendo UI. It is the best way to get data in and out of the many widgets that the Kendo suite offers.
The Kendo team has done a great job at explaining the many different back-end technologies you can use with your DataSource.

Microsoft recently released the javascript SDK for its BaaS, Windows Azure Mobile Services.

On this blog post I will explain how to use this SDK in conjunction with the Kendo DataSource.
I will use a Kendo UI Grid as example, but theoretically the DataSource can be used with many different Kendo components.





```javascript
$("#grid").kendoGrid(
            {
                pageable: true,
                sortable: true,
                filterable:true,
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