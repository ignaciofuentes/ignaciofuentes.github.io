---
layout: post
title: "Performance Considerations when using Kendo Grids in BatchEdit Mode."
date: 2014-09-10 -0800
comments: true
categories: [code]
---

One of the most attractive features of the Kendo Grid is that you can use it in [BatchEdit Mode](http://demos.telerik.com/kendo-ui/grid/editing). This means that you can make all the changes you like to the data currently being displayed and then save them with a button click, all at the same time. This saves you several trips to the server as it all just happens in one HTTP request (per type of operation) that has all the grid items that were just added/deleted/modified. 

This does not mean you shouldn't strive to make sure that the code that runs to support these actions is as performant as possible, though, as it is possible that in these methods you are doing a lot more work than needed. This is particularly true if you are using an ORM, like Entity Framework, that abstracts database interaction so much that sometimes it's difficult to tell what SQL queries were made and why.

Let's take a look at a simple Grid in Batch Edit Mode, the implementation of the Controller that supports it, and how it could be significantly improved for performance.

I will be using [MVC MiniProfiler](https://www.nuget.org/packages/MiniProfiler/) and [Miniprofile.EF6](https://www.nuget.org/packages/MiniProfiler.EF6/) to monitor all the SQL queries that Entity Framework sends to the database to determine if there is any room for improvement on this.

##Ground Work

The goal is to create a Grid of "Cars", each car will have a "Category" that will be displayed as a Kendo ComboBox so the the user can either pick a pre-existing Category or simply type a new category to be added to the database when creating or editing a car in the grid. This means that we will need a Car class, a Category class, and for design principles, a CarViewModel class that will be used for the grid.

```csharp
public class Car
{   
    public int Id { get; set; }
    [Required]
    [StringLength(50)]
    public string Name { get; set; }
    public int CategoryId { get; set; }
    public virtual Category Category { get; set; }

}

public class Category {
    public int Id { get; set; }
    public string Name { get; set; }
    public virtual List<Car> Cars { get; set; }    
}

public class CarViewModel
{
    public int Id { get; set; }

    public string Name { get; set; }

    [UIHint("ClientCategory")]
    [Required]
    public string Category
    {
        get;
        set;
    }
}
```


The UI Hint on the Category property of the CarViewModel tells the MVC framework where to look for the EditorTemplate it will use for the creation/updating of the Category of our cars.
This gives us all the pieces we need to start working on creating our Grid and the Controller to support it.

## The Grid and EditorTemplate

For this demo, I will use the UI For ASP.NET MVC Kendo Wrappers to generate a simple Kendo Grid of "CarViewModel" but this would be very similar using the JavaScript API for Kendo UI.


```csharp
//Index.cshtml
@(Html.Kendo().Grid<TelerikMvcApp4.Controllers.CarViewModel>()
.Name("grid")
.Columns(columns =>
{
    columns.Bound(p => p.Id);
    columns.Bound(p => p.Name);
    columns.Bound(p => p.Category);
    columns.Command(command => { command.Destroy(); }).Width(172);
})
.ToolBar(toolbar =>
{
    toolbar.Create();
    toolbar.Save();
})
.Editable(e => e.Mode(GridEditMode.InCell))
.Pageable()
.Sortable()
.Scrollable(e => e.Height(340))
.DataSource(dataSource => dataSource
    .Ajax()
    .Sort(sort => sort.Add("Id").Descending())
    .Batch(true)
    .Model(model =>
    {
        model.Id(p => p.Id);
        model.Field(p => p.Id).Editable(false);
        model.Field(p => p.Category);
    })
    .PageSize(10)
    .Create(c => c.Action("Cars_Create", "Home"))
    .Read(read => read.Action("Cars_Read", "Home").Type(HttpVerbs.Get))
    .Update(update => update.Action("Cars_Update", "Home"))
    .Destroy(update => update.Action("Cars_Destroy", "Home"))
)
)
```
```csharp
//ClientCategory.cshtml
@(Html.Kendo().ComboBoxFor(m => m)
    .Placeholder("Select a Category")
    .HighlightFirst(false)
    .DataSource(ds =>
    {
        ds.Read(r => r.Action("Categories", "Home"));
    })
)
```


## Initial Controller Set-Up

For the sake of brevity, I am going to limit this post to examining the performance of the Update Action of the Controller but the same principles apply to the Create Action of the Controller.

```csharp
[HttpPost]
public ActionResult Cars_Update([DataSourceRequest] DataSourceRequest request, [Bind(Prefix = "models")]IEnumerable<CarViewModel> cars)
{
    foreach (var car in cars)
    {
        if (car != null && ModelState.IsValid)
        {
            var dbCar = db.Cars.Where(c => c.Id == car.Id).SingleOrDefault();
            if (dbCar != null)
            {                        
                var category = db.Categories.Where(cat => cat.Name == car.Category).SingleOrDefault();
                if (category == null)
                {
                    category = new Category { Name = car.Category };
                }
                dbCar.Name = car.Name;
                dbCar.Category = category;
            }

        }
    }
    db.SaveChanges();
    return Json(cars.ToDataSourceResult(request, ModelState));
}
```

Now when I make a few edits on the page and hit the Save Changes button all of my changes are synced, but when I look at the output from MiniProfile I see that a lot of SQL queries (up to 40 in this sample!!) were made even though I only have ten items per page on my Grid.

![naive implementation](https://raw2.github.com/ignaciofuentes/ignaciofuentes.github.io/master/images/optimization/naive.gif)


## What is wrong with this


At first glance there doesn't seem to be anything evidently wrong with this method. It's a simple iteration over each car being updated, with a straight-forward update to the values on each car, followed by a standard SaveChanges call on the EntityFramework DbContext.

The problem, though, is that before updating the Car record, it is being retrieved. That's already a possibility of 10 SQL Select queries before even getting to updating any of the records. The same is happening with the related Category record, which also needs to be retrieved by using the Category (string) of the incoming CarViewModel.
The fact that these operations happen for each car (bacause this is a Grid in BatchEdit Mode) is what aggravates this easy-to-overlook problem to a much larger scale.

## How this can be fixed

Luckily for us, [EntityFramework allows for entities to be Attached to the DbContext](http://msdn.microsoft.com/en-us/data/jj592676.aspx) even though they were not retrieved from the database initially. This means we can write code that will tell Entity Framework to Edit a record without having to manually retrieve it first.


That fixes the problem of retrieving the Car record before updating its values. But, the Category record needs to be handled differently, as we are not updating it at all. We are simply assigning a CategoryId to the Car record. In the case where the Category does not exist we are creating it first and then assigning its Id it to the CategoryId of the car record being updated.

To improve this we can change our implementation a little bit so that the Grid actually sends either a CategoryId (if the Category already exists) or a CategoryId with a null value and a string value for CategoryName so that it gets created before updating the car. We will need to change the ViewModel so that the editor applies now to the CategoryName. The CategoryId gets set with javascript (to either a numeric value or null) on the [Change Event of our Kendo ComboBox.](http://docs.telerik.com/kendo-ui/api/javascript/ui/combobox#events-change)


```csharp
public class CarViewModel
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int? CategoryId { get; set; }
    [UIHint("ClientCategory")]
    public string CategoryName { get; set; }
}

//ClientCategory.cshtml
@(Html.Kendo().ComboBoxFor(m => m)
.Placeholder("Select a Category")
.DataTextField("Name")
.DataValueField("Name")
.DataSource(ds =>
{

    ds.Read(r => r.Action("Categories", "Home"));

})
.Events(d => d.Change("selectionChanged"))
)
```


```javascript
function selectionChanged(e) {
    var tr = this.element.closest("tr");
    var cbBoxItem = this.dataItem();
    var grid = $("#grid").data("kendoGrid");
    var item = grid.dataItem(tr);
    item.CategoryId = cbBoxItem ? cbBoxItem.Id : null;
}
``` 

With these changes in place the Controller looks not too differently but is certainly much more performant when put to the test.

```csharp
[HttpPost]
public ActionResult Cars_Update([DataSourceRequest] DataSourceRequest request, [Bind(Prefix = "models")]IEnumerable<CarViewModel> cars)
{
    foreach (var car in cars)
    {
        if (car != null && ModelState.IsValid)
        {
            var dbCar = new Car { Id = car.Id, Name = car.Name };
            if (car.CategoryId.HasValue)
            {
                dbCar.CategoryId = car.CategoryId.Value;
            }
            else
            {
                var category = new Category { Name = car.CategoryName };
                db.Entry(category).State = EntityState.Added;
                dbCar.Category = category;
            }

            db.Cars.Attach(dbCar);
            db.Entry(dbCar).State = EntityState.Modified;

        }
    }
    await db.SaveChanges();
    return Json(cars.ToDataSourceResult(request, ModelState));
}
```


![performant implementation](https://raw2.github.com/ignaciofuentes/ignaciofuentes.github.io/master/images/optimization/performant.gif)

Ahh. Much better :)

You can find the full source-code for the [original application](https://github.com/ignaciofuentes/BatchEditDemoNaive) and the [improved one](https://github.com/ignaciofuentes/BatchEditDemoOptimized) in my [GitHub](https://github.com/ignaciofuentes).