---
layout: post
title: "Kendo UI Grid now supports SignalR!"
date: 2014-04-07 -0800
comments: true
categories: [code]
---

The Kendo UI Grid Widget is a very powerful tool that can help you leverage a simple UI paradigm to provide full CRUD functionality that spans from your web page to your server. What makes this control so versatile is the fact that it can work with a varied set of backend technologies thanks to the [DataSource.](http://demos.kendoui.com/web/datasource/index.html)

Up until now, though, regardless of what backend technology you were using to connect with your database, the datasource was the one responsible to request the data from the server. This means that, after the grid and its data has been loaded on to the page, the grid has had no way of knowing if there has been any modification on the server.

What if we want our grid to hold a real-time connection to the server so that it can be immediately notified of modifications made by another client?

## Introducing real-time connections with ASP.NET SignalR.

[SignalR](http://signalr.net/) is an ASP.NET Library that provides a framework for clients to persisently connect to servers by using the best possible technology that the server and the client support. This means that SignalR will automatically choose if the persistence connection will be done via WebSockets, Server-Sent Events, Long-Polling or regular polling for changes. After the technology has been chosen and the real-time connection intitialized, SignalR allows the connected clients to [call methods on the server](http://www.asp.net/signalr/overview/signalr-20/hubs-api/hubs-api-guide-server#hubmethods) and the server to [call methods on the connected clients.](http://www.asp.net/signalr/overview/signalr-20/hubs-api/hubs-api-guide-server#callfromhub)

Q1 2014 version of Kendo UI brings [support for SignalR.](http://demos.telerik.com/kendo-ui/web/grid/signalr.html) A Kendo Grid can now be notified of any changes made on the server and react accordingly.

## How you can do it.

The easiest way to install SignalR to your app is through the ASP.NET SignalR nuget package. This package will place the necessary dlls in your references and add the SignalR Javascript library to your Scripts folder.

After this you will want to add a Startup.cs class to your project

```c#
public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            app.MapSignalR();
        }
    }
```
The MapSignalR method generates the "/signalr" route on the asp.net project that clients will connect to for real-time communications.
The final bit of boilerplate configuration we need to do is simply add the necessary javascript libraries to our page.

```html
<script src="http://code.jquery.com/jquery-1.9.1.min.js"></script>
<script src="~/Scripts/jquery.signalR-2.0.3.min.js"></script>
<script src="~/signalr/hubs"></script>
<script src="http://cdn.kendostatic.com/2014.1.318/js/kendo.all.min.js"></script>
```

With all of this in place we can build our Kendo Grid with real-time communications. 

For this demonstration I will build a Games Grid with two columns (Name, Developer) and full CRUD functionality and real-time reactions in the case of modifications to the data made by other clients.

## Server-side.

We will need to create a SignalR Hub. This is the class that will have the capacity to call javascript methods on the connected clients as well as expose methods that connected clients can remotely invoke. For data access I will use Entity Framework, but this is something that could work with any ORM or Data Access technology you want to use, as long as you follow the same contract that the grid expects.

```c#
public class GridHub : Hub
    {
        private GamesDbEntities db; //Entity Framework context class
        public GridHub()
        {
            db = new GamesDbEntities();
        }

        public IEnumerable<GameViewModel> Read()
        {
            return from game in db.Games
                   select new GameViewModel { Id = game.Id, Name = game.Name, Developer = game.Developer };
        }

        public void Update(GameViewModel gameVm)
        {
            var g = db.Games.Where(game => game.Id == gameVm.Id).SingleOrDefault();
            if (g != null)
            {
                g.Name = gameVm.Name;
                g.Developer = gameVm.Developer;
                db.SaveChanges();
                Clients.Others.update(gameVm);
            }

        }

        public void Destroy(GameViewModel gameVm)
        {
            var g = db.Games.Where(game => game.Id == gameVm.Id).SingleOrDefault();
            if (g != null)
            {
                db.Games.Remove(g);
                db.SaveChanges();
                Clients.Others.destroy(gameVm);
            }
        }

        public GameViewModel Create(GameViewModel gameVm)
        {
            var g = new Game { Name = gameVm.Name, Developer = gameVm.Developer };
            db.Games.Add(g);
            db.SaveChanges();
            gameVm.Id = g.Id;
            Clients.Others.create(gameVm);
            return gameVm;
        }
    }
```
These four methods are exposed via javascript to web clients that connect to it. In our case, the grid's datasource.


## Client-side

As usual, the grid itself is trivial to set up.

```javascript
$("#grid").kendoGrid({
    height: 300,
    toolbar: ["create"],
    editable: true,
    columns: [
        "Name",
        "Developer",
        {
            command: [
              { name: "destroy", text: "Delete" }
            ]
        }
    ],
    dataSource: signalRDataSource
});
```

The meat of the logic will reside on our signalRDataSource property, which will make us of our GridHub.

```javascript
var connection = $.connection;
var hub = connection.gridHub;
var hubStart = connection.hub.start();
var signalRDataSource = new kendo.data.DataSource({
    type: "signalr",
    autoSync: true,
    schema: {
        model: {
            id: "Id",
            fields: {
                "Id": { editable: false, type: "Number" },
                "Name": { type: "string" },
                "Developer": { type: "string" }
            }
        }
    },
    transport: {
        signalr: {
            promise: hubStart,
            hub: hub,
            server: {
                read: "read",
                update: "update",
                destroy: "destroy",
                create: "create"
            },
            client: {
                read: "read",
                update: "update",
                destroy: "destroy",
                create: "create"
            }
        }
    }
});
```
And that's it. 

 We can now make modifications to our data and other connected clients can be notified immediately!

 ![real-time](https://raw2.github.com/ignaciofuentes/ignaciofuentes.github.io/master/images/signalr.gif)


You can find the full source code for this demo in my [GitHub repository.](https://github.com/ignaciofuentes/KendoSignalR)
Hopefully this has helped you understand the basics of SignalR and how you can leverage its capabilites to provide real-time communications to a Kendo Grid.