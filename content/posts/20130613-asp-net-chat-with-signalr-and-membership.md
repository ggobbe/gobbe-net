---
title: "Build a chat with SignalR and ASP.NET Membership"
date: 2013-06-13T07:45:39+01:00
draft: false
categories: ["ASP.NET"]
tags: ["aspnet-mvc", "signalr", "aspnet-membership"]
---

Today, I had a look at a simple tutorial about SignalR which made me create a simple chat using SignalR.

The chat ask the user to give a pseudonym that will be used on the chat, then allows the user to enter messages and to receive other users messages.

After I completed the tutorial, I wished to use the already existing **ASP.NET Membership** from the **Internet ASP.NET Application** template which was used to create the chat in the tutorial.

I’ll explain the few steps I’ve done to achieve this.

First of all, follow this tutorial to get the basic chat:
[Tutorial: Getting Started with SignalR and MVC 4 (C#)](https://web.archive.org/web/20130513154625/http://www.asp.net:80/signalr/overview/getting-started/tutorial-getting-started-with-signalr-and-mvc-4)

Once it works, let’s add our changes to use the ASP.NET Membership instead of asking the user to choose his pseudonym.

I opened the **HomeController** to add a few membership annotations so that the user will still be able to access the basics actions methods without having to be identified, but the **Chat** action method will require him to be identified.

Add these annotations to the HomeController class:

```csharp
[Authorize]
[InitializeSimpleMembership]
```

Then add this annotation to the Index, About and Contact action methods: `[AllowAnonymous]`

Your HomeController class should look like this:

```csharp
using SignalRChat.Filters;
using SignalRChat.Helpers;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Security.Cryptography;
using System.Web;
using System.Web.Mvc;
using System.Web.Security;

namespace SignalRChat.Controllers
{
    [Authorize]
    [InitializeSimpleMembership]
    public class HomeController : Controller
    {
        [AllowAnonymous]
        public ActionResult Index()
        {
            ViewBag.Message = "Modify this template to jump-start your ASP.NET MVC application.";

            return View();
        }

        [AllowAnonymous]
        public ActionResult About()
        {
            ViewBag.Message = "Your app description page.";

            return View();
        }

        [AllowAnonymous]
        public ActionResult Contact()
        {
            ViewBag.Message = "Your contact page.";

            return View();
        }

        public ActionResult Chat()
        {
            ViewBag.Token = MD5Helper.GetMd5Hash(TokenHelper.GetToken(User.Identity.Name));

            return View();
        }
    }
}
```

When it is done, you could try to access the **/Home/Chat** Url in your browser, it will ask you to register or to login (except if you are already registered) but the Chat is still asking you for a pseudonym to use on it.

So let’s edit the **Chat.cshtml** view to use the name of the logged user instead of asking him to give one. We will then modify the value of the **displayname** hidden field like this:

```html
<input id="displayname" type="hidden" value="@User.Identity.Name" />
```

Then in the JavaScript code below (in Chat.cshtml) we’ll comment the line where the user pseudonym was asked:

```js
// We do not ask for a pseudonym as we are using the name of the user
// $('#displayname').val(prompt('Enter your name:', ''));
```

If you launch the application, the chat won’t ask you for a pseudonym and will correctly use the name of the logged user. That’s great but yet it is quite easy to edit the value of the **displayname** hidden field to use another pseudonym (e.g. to speak in the name of somebody else).

Don’t worry about it, we’ll fix it!

To achieve this, I’ve choosen to create some kind of token verification to forbid the user to change his name in the **displayname** hidden field.

To do this, I’ve created two new classes to help me work with tokens and md5 hashes. So let’s add a new **Helpers** folder in our projet and add these two classes in it:

`$ cat MD5Helper.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using System.Web;

namespace SignalRChat.Helpers
{
    public class MD5Helper
    {
        public static string GetMd5Hash(string input)
        {
            using (MD5 md5Hash = MD5.Create())
            {
                // Convert the input string to a byte array and compute the hash. 
                byte[] data = md5Hash.ComputeHash(Encoding.UTF8.GetBytes(input));

                // Create a new Stringbuilder to collect the bytes 
                // and create a string.
                StringBuilder sBuilder = new StringBuilder();

                // Loop through each byte of the hashed data  
                // and format each one as a hexadecimal string. 
                for (int i = 0; i < data.Length; i++)
                {
                    sBuilder.Append(data[i].ToString("x2"));
                }

                // Return the hexadecimal string. 
                return sBuilder.ToString();
            }
        }

        // Verify a hash against a string. 
        public static bool VerifyMd5Hash(string input, string hash)
        {
            using (MD5 md5Hash = MD5.Create())
            {
                // Hash the input. 
                string hashOfInput = GetMd5Hash(input);

                // Create a StringComparer an compare the hashes.
                StringComparer comparer = StringComparer.OrdinalIgnoreCase;

                // If you prefer you can do this instead of what's following below :
                // return !Convert.ToBoolean(comparer.Compare(hashOfInput, hash));
                if (0 == comparer.Compare(hashOfInput, hash))
                {
                    return true;
                }
                else
                {
                    return false;
                }
            }
        }
    }
}
```

`$ cat TokenHelper.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Web;

namespace SignalRChat.Helpers
{
    public class TokenHelper
    {
        public static string GetToken(string name)
        {
            // Edit this to add your own salt !
            return string.Format("{0}&{1}={2}", name, (name.Length * 97), new string(name.Reverse().ToArray()));
        }

        public static bool VerifyToken(string name, string token)
        {
            return MD5Helper.VerifyMd5Hash(TokenHelper.GetToken(name), token);
        }
    }
}
```

So basically, the MD5Helper class will help us to hash a string or to verify if a string matches a given hash. And the TokenHelper class will help us to generate an hash for a user’s name and to verify if a given hash is correct for a given name.

Don’t forget to edit the **GetToken** method in **TokenHelper.cs** to use some kind of randomness of your own so that the malicious user won’t know what you are using to generate the hash (otherwise he will be able to generate a new hash for any user’s name that he wants to use.

Now that we can generate a hash for a user name, let’s add this hash to the dynamic variable **ViewBag.Token** in the **Chat** action method in **HomeController.cs**

```csharp
public ActionResult Chat()
{
    ViewBag.Token = MD5Helper.GetMd5Hash(TokenHelper.GetToken(User.Identity.Name));
    return View();
}
```

And to allow JavaScript to give this hash to the server method **Send**, we’ll add a new hidden field with the id token and edit the JavaScript of the Chat.cshtml view to give this hash to the **Send** server method.

Add an hidden field which will contain the hash to the **Chat.cshtml** view (e.g. after the **displayname** hidden field) :

```html
<input type="hidden" id="token" value="@ViewBag.Token" />
```

And edit the JavaScript code of the **Chat.cshtml** view to give a third parameter (the hash) to the **Send** server method:

```js
// Call the Send method on the hub. 
chat.server.send(
    $('#displayname').val(),
    $('#message').val(),
    $('#token').val()
);
```

Don’t forget to edit the **ChatHub.cs Send** method signature to accept this new hash parameter:

```csharp
public void Send(string name, string message, string token)
{
    // ...
}
```

Test if the chat is still working, if yes, there is one last step to do: verify if the received hash in the **Send** method of the **ChatHub.cs** matches the expected hash for the received the name.

Just add theses lines on the top of the **Send** method in **ChatHub.cs**:

```csharp
// Verify if token is valid
if (!TokenHelper.VerifyToken(name, token))
{
    Clients.Caller.addNewMessageToPage("Error", "Token is not valid !");
    return;
}
```

Great, with this small code, if the received hashed token don’t match the expected hash for the received name, we will print an error message in the user’s chat (only the caller) and we won’t send his message to other users.

Test your chat, everything should be working fine, try to edit the **displayname** hidden field (hit F12 in your favourite browser to edit the DOM) and you’ll see that you receive the error message when you try to send a message if you have modified the **displayname** hidden field.

*Update: There is a post on the MSDN Blogs that speaks about User Identity and SignalR, follow the link [SignalR and user identity (authentication and authorization)](https://web.archive.org/web/20131217183316/http://blogs.msdn.com/b/webdev/archive/2013/10/11/signalr-and-user-identity-authentication-and-authorization.aspx)*