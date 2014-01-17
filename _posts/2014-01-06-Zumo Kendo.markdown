---
layout: post
title: "Kendo DataSource and Azure Mobile Services."
date: 2014-01-07 -0800
comments: true
categories: [code]
---

From the topic of this and [my last post](http://haacked.com/archive/2014/01/04/duck-typing/), you would be excused if you think I have some weird fascination with ducks. In fact, I'm starting to question it myself.

![Is it a duck? - CC BY-ND 2.0 from http://www.flickr.com/photos/77043400@N00/224131630/](https://f.cloud.github.com/assets/19977/1845502/4d494752-758e-11e3-9c66-8fd6080662fe.jpg)

I just find the topic of duck typing (and structural typing) to be interesting and I'm not known to miss an opportunity to beat a dead horse (or duck for that matter).

Often, when we talk about duck typing, code examples only ask if the object quacks. But shouldn't it also ask if it quacks _like a duck_?

For example, here are some representative examples you might find if you Google the terms "duck typing". The first example is Python.

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

Here's an example in Ruby.

```ruby
def func(arg)
  if arg.respond_to?(:quack)
    arg.quack
  end
end
```

Does this miss half the [point of duck typing](https://groups.google.com/forum/?hl=en#!msg/comp.lang.python/CCs2oJdyuzc/NYjla5HKMOIJ)?

> In other words, don't check whether it IS-a duck: check whether it QUACKS-like-a duck, WALKS-like-a duck, etc, etc,...
