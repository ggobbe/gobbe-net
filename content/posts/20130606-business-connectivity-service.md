---
title: "Database tables as SharePoint lists with Business Connectivity Service (BCS)"
date: 2013-06-06T22:04:01+01:00
draft: false
categories: ["SharePoint"]
tags: ["bcs", "odata", "sharepoint-2013", "sharepoint-apps"]
---

Let’s suppose that you want to use an external database table in a SharePoint-hosted App, here is how to achieve this !

This might seem a long process but the truth is that is quite easy.

We’ll have to create a WCF Data Service ([OData Protocol](http://en.wikipedia.org/wiki/Open_Data_Protocol)) to expose our database. Then the next step is to deploy this Data Service somewhere to be accessible to the SharePoint-hosted App (e.g. on [Microsoft Azure](https://azure.microsoft.com)). Once that is done, you’ll be able to add an External Content Type (based on the Data Service previously deployed) in the SharePoint-hosted App (be careful, you can’t add an External Content Type to a Cloud-hosted App).

Let’s start by creating our Data Service (I’ll assume that you already have a database deployed somewhere).

First of all, create a new **ASP.NET MVC 4 Web Application** project (or any project that can hold an Entity Data Model and expose a WCF Data Service).

![screenshot](/images/20130606-business-connectivity-service/20130605-234302-new-project.png)

As the ASP.NET MVC 4 Application will be useless except for the Data Service, I’ll thus select the **Empty** template in the next dialog window.

One the project is created, let’s add an Entity Data Model to it. Right click on the ODataService project, then select **Add**, then hit the **New Item** entry.

![screenshot](/images/20130606-business-connectivity-service/20130605-235038-add-new-item.png)

There, you’ll search for **entity** on the top right corner and select **ADO.NET Entity Data Model**.

![screenshot](/images/20130606-business-connectivity-service/20130606-162029-add-new-item-odataservice.png)

In the next window, select **Generate from database**, then create a new connection or use an existing one as below (I’ll use one hosted on Microsoft Azure for the purpose of this article).

![screenshot](/images/20130606-business-connectivity-service/20130606-163351-entity-data-model-wizard.png)

You’ll arrive on a window allowing you to select which tables you want to include in your model. Just select thoses you want.

![screenshot](/images/20130606-business-connectivity-service/20130606-163807-entity-data-model-wizard.png)

You’ll only have to hit the finish button to get your Data Model created (mine is below).

![screenshot](/images/20130606-business-connectivity-service/20130606-164102-gobbe-microsoft-visual-studio-300x159.png)

Now that we have our Data Model, we’ll add a new Data Service by right clicking on the ODataService project, then select **Add**, then hit the **New Item** entry (just as we did to add a new Entity Data Model previously).

![screenshot](/images/20130606-business-connectivity-service/20130605-235506-add-new-item-odataservice.png)

You should normally have a new DataService created which looks like this:

```csharp
public class GobbeDataService : DataService< /* TODO: put your data source class name here */ >
{
    // This method is called only once to initialize service-wide policies.
    public static void InitializeService(DataServiceConfiguration config)
    {
        // TODO: set rules to indicate which entity sets and service operations are visible, updatable, etc.
        // Examples:
        // config.SetEntitySetAccessRule("MyEntityset", EntitySetRights.AllRead);
        // config.SetServiceOperationAccessRule("MyServiceOperation", ServiceOperationRights.All);
        config.DataServiceBehavior.MaxProtocolVersion = DataServiceProtocolVersion.V3;
    }
}
```

Let’s edit the TODO on the top of it to put our data source class name there. Mine is *GobbeEntities*.

`public class GobbeDataService : DataService`

And finally, let’s add some access rules on our entities with the `config.SetEntitySetAccessRule()` method. I’ll put all the rights on the entity Eleves, and only Read rights on the entity Ecoles

```csharp
public class GobbeDataService : DataService<GobbeEntities>
{
    // This method is called only once to initialize service-wide policies.
    public static void InitializeService(DataServiceConfiguration config)
    {
        config.SetEntitySetAccessRule("Eleves", EntitySetRights.All);
        config.SetEntitySetAccessRule("Ecoles", EntitySetRights.AllRead);

        config.DataServiceBehavior.MaxProtocolVersion = DataServiceProtocolVersion.V3;
    }
}
```

Now if you do a right click on the GobbeDataService.svc then you select **View in Browser**, it should open a window in your browser showing you a 404 error. That’s normal because ASP.NET MVC have routes and we’ll have to tell him to ignore routes finishing by .svc to make our Data Service available.

Open the file **App_Start\RouteConfig.cs** in your application and add the line 6 from the code below at the same place (before the `MapRoute()` call).

```csharp
public static void RegisterRoutes(RouteCollection routes)
{
    routes.IgnoreRoute("{resource}.axd/{*pathInfo}");

    // Let's ignore ignore routes regarding SVC services
    routes.IgnoreRoute("{*allsvc}", new { allsvc = @".*\.svc(/.*)?" });

    routes.MapRoute(
        name: "Default",
        url: "{controller}/{action}/{id}",
        defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional }
    );
}
```

Now you can right click on your DataService (mine is GobbeDataService.svc) and select **View in Browser**, it should open a new page in your browser with XML content just like this:

```xml
<service xml:base="http://localhost:51193/GobbeDataService.svc/" xmlns="http://www.w3.org/2007/app" xmlns:atom="http://www.w3.org/2005/Atom">
	<workspace>
		<atom:title>Default</atom:title>
		<collection href="Ecoles">
			<atom:title>Ecoles</atom:title>
		</collection>
		<collection href="Eleves">
			<atom:title>Eleves</atom:title>
		</collection>
	</workspace>
</service>
```

If it works, you can now publish it by right clicking on the ODataService project, then select **Publish**. Here I’ll publish it on a free Windows Azure Website. Once it is published, test it in your browser by typing the website URL suffixed by the Data Service file name (e.g. http://gobbe.azurewebsites.net/GobbeDataService.svc/ ). Here is the result expected.

![screenshot](/images/20130606-business-connectivity-service/20130606-182015-dataservice-svc.png)

If everything works fine, your database tables are now exposed through an OData Service which is accessible from everywhere on the Internet (maybe you should put some authentication if you put write rights as I do… but for this article we won’t do it), we can now start using it in our SharePoint-hosted App.

To achieve this, right click on the solution then go in the **Add** menu and select **New Project…**

In the **New Project** window, select the **App for SharePoint 2013** template and name it (don’t forget to install the [Microsoft Office Developer Tools](http://www.microsoft.com/visualstudio/eng/office-dev-tools-for-visual-studio) otherwise you won’t be able to create an App for SharePoint 2013).

![screenshot](/images/20130606-business-connectivity-service/20130606-182658-add-new-project.png)

Then fill the next window like below (be sure to choose a **SharePoint-hosted App**) on the bottom.

![screenshot](/images/20130606-business-connectivity-service/20130606-182928-new-app-for-sharepoint-1.png)

Wait a few moments and your SharePoint-hosted App should be created ! Next step is to right click on the SharePoint-hosted App project, then select the **Add** menu, and hit **Content Types for an External Data Source**.

In the newly opened window, enter the address of the Data Service we just hosted!

![screenshot](/images/20130606-business-connectivity-service/20130606-183840-sharepoint-customization-wizard-1.png)

In the next window, just select the tables for which you want to generate an external content type.

![screenshot](/images/20130606-business-connectivity-service/20130606-184111-sharepoint-customization-wizard-2.png)

And it is done, you are now able to access your lists (which are actually tables from your database) from within your App (with JSOM or through the REST API, you can use the List.getByTitle access point and give your list name to it).

Just one more point, by default, there is a **Limit** filter on your External Content Types which limits the result set of queries made on the list to 100 results maximum. To remove it, open the **<ListName>.ect** file in your **SharePoint-hosted App project** and in the filters list (on the bottom of the **.ect** file opened), click on the **Limit** filter, then right click on the green arrow on the left of the filter and hit the **Delete** action in the context menu.

![screenshot](/images/20130606-business-connectivity-service/20130606-184839-gobbe-microsoft-visual-studio.png)

I hope everything worked fine on your side, if you meet any problem during the set up of something, just leave a comment on this post and I’ll try to help you!