---
title: "ClientContext Helper for SharePoint cloud-hosted Apps"
date: 2013-05-26T11:32:13+01:00
draft: false
categories: ["SharePoint"]
tags: ["csom", "sharepoint-2013", "sharepoint-apps"]
---

When I created my first Autohosted-App in SharePoint 2013, it contained only one page. Then I tried to add multiple pages and it was such a pain to work with the ClientContext through all the pages on my website. That’s because you receive the standard tokens only on the start page of your remote web.

<!--more-->

To allow me working with the ClientContext on any page of my website, I created a small Helper that will manage the Client Context for me. Thus I don’t have to give the standard tokens in the Url of any links on my remote web.

First, be sure to have the TokenHelper.cs class in your remote web project, then you might add a new class named ContextHelper.cs

```csharp
using Microsoft.SharePoint.Client;
using System;
 
namespace Gobbe.Helpers
{
    public class ContextHelper
    {
        public static ClientContext GetClientContext(Uri url)
        {
            string contextToken = TokenHelper.GetContextTokenFromRequest(System.Web.HttpContext.Current.Request);
            string hostWeb = System.Web.HttpContext.Current.Request["SPHostUrl"];
 
            if (!string.IsNullOrEmpty(contextToken))
                System.Web.HttpContext.Current.Session["ctx"] = contextToken;
            else
                contextToken = (string)System.Web.HttpContext.Current.Session["ctx"];
 
            if (!string.IsNullOrEmpty(hostWeb))
                System.Web.HttpContext.Current.Session["host"] = hostWeb;
            else
                hostWeb = (string)System.Web.HttpContext.Current.Session["host"];
 
            return TokenHelper.GetClientContextWithContextToken(hostWeb, contextToken, url.Authority);
        }
    }
}
```

In my case the remote web is an ASP.NET MVC4 website, with this ContextHelper, I’m now able to use the ClientContext in every Action method of any Controller and I don’t have to give the Standard Tokens to every ActionLink (SPHostUrl, …) to be able to get the ClientContext.

```csharp
namespace Gobbe.Controllers
{
    public class HomeController : Controller
    {
        public ActionResult Index()
        {
            using (var clientContext = ContextHelper.GetClientContext(Request.Url))
            {
                Web web = clientContext.Web;
                List list = web.Lists.GetByTitle("MyCalendar");

                string camlString = String.Format("{0}\n{1}\n{2}\n{3}\n{4}\n{5}",
                    "<view><viewfields>",
                        "<fieldref name='Title'></fieldref>",
                        "<fieldref name='EventDate'></fieldref>",
                        "<fieldref name='Category'></fieldref>",
                        "<fieldref name='Location'></fieldref>",
                    "</viewfields></view>");

                CamlQuery camlQuery = new Microsoft.SharePoint.Client.CamlQuery() { ViewXml = camlString };

                ListItemCollection allItems = list.GetItems(camlQuery);

                clientContext.Load(allItems);
                clientContext.ExecuteQuery();

                return View(allItems.ToList<listitem>());
            }
        }

        public ActionResult Another()
        {
            using (var clientContext = ContextHelper.GetClientContext(Request.Url))
            {
                // Do some stuff with the client context

                return View();
            }
        }
    }
}
```

Having this will store the context token and the host web in the session if the standard tokens are given in the Url. And when you’ll go to another action without giving the standard tokens in the Url, the context token and the host web will just be recovered from the session.

NB : You’ll have to call at least one time the GetClientContext method from the ContextHelper class in your start page Action, otherwise the standard tokens will not be saved in the session.