[Source](http://dotnetcodr.com/2013/02/25/claims-based-authentication-in-mvc4-with-net4-5-c-part-1-claims-transformation/ "Permalink to Claims-based authentication in MVC4 with .NET4.5 C# part 1: Claims transformation")

# Claims-based authentication in MVC4 with .NET4.5 C# part 1: Claims transformation

This post will look into how Claims can be introduced in an MVC4 internet application.

We will build on the basics of claims we discussed in previous posts:

I will make references to those posts and if you have absolutely no experience with Claims-based auth in .NET4.5 then I suggest you take a look at them as well.

Start Visual Studio and create a new MVC4 internet application with .NET4.5 as the underlying framework.

The template will set up a good starting point for an MVC web app including forms based authentication. Run the application and click the 'Register' link in the top right hand corner of the screen. You'll be redirected to a page where you can fill out the user name and password:

![Register link in basic MVC4 web app][1]

Fill in the textboxes, press the 'Register' button and there you are, you've created a user. You'll see that you were also automatically logged on:

![User automatically logged in upon registration][2]

**A short aside on forms-based auth in MVC4**

The default structure of creating and authenticating users in the basic template MVC4 app is far from ideal. I wrote two posts on forms-based auth in MVC4 [here][3] and [here][4] where you can learn more on this topic and how you can transform the template to something that's a more suitable starting point.

For now just accept that the user you created in the previous step is saved in an attached local SQL Express database that was created for you. You can view this database by looking at the Server Explorer (Ctrl W, L). There you should see a database called DefaultConnection:

![Default connection database][5]

Open that node and you'll see several tables that .NET created for you when the first user was registered. You'll find a detailed discussion on what they are and how they were created in the posts on forms-based auth I mentioned above. For now it's enough to say that the user was saved in two different tables. One's called webpages_Membership, where the password is stored in a hashed form. The table UserProfile stores the user name. Right-click that table and select 'Show table data' from the context menu. There you should see the user with user id = 1:

![Registered user in DB][6]

Open the membership table as well to view how the password is stored.

In case you're wondering about what connection string was used you can see it in web.config:



    <connectionStrings>
        <add name="DefaultConnection" connectionString="Data Source=(LocalDb)v11.0;Initial Catalog=aspnet-ClaimsInMvc4-20130130200447;Integrated Security=SSPI;AttachDBFilename=|DataDirectory|aspnet-ClaimsInMvc4-20130130200447.mdf" providerName="System.Data.SqlClient"/>
     </connectionStrings>


**Claims transformation in MVC4**

If you recall from [part 3][7] we registered a claims transformation logic in our simple Console application. Our goal is to do the same in our web app.

Right-click the web project in the Solution Explorer and add a new class file called CustomClaimsTransformer. The class must derive from ClaimsAuthenticationManager in the System.Security.Claims namespace. You will have add two references to your project: System.IdentityModel and System.IdentityModel.Services. The skeleton of the class will look like this:



    public class CustomClaimsTransformer : ClaimsAuthenticationManager
        {
        }


We will override the Authenticate method:



    public class CustomClaimsTransformer : ClaimsAuthenticationManager
        {
            public override ClaimsPrincipal Authenticate(string resourceName, ClaimsPrincipal incomingPrincipal)
            {
                return base.Authenticate(resourceName, incomingPrincipal);
            }
        }


This structure should look familiar from Part 3 of Claims basics. The resource name is an optional variable to describe the resource the user is trying to access. The incoming ClaimsPrincipal object is the outcome of the authentication; it represents the User that has just been authenticated on the login page. Now we want to 'dress up' the claims collection of that user.

Our first check is to see if the ClaimsPrincipal has been authenticated.



    public override ClaimsPrincipal Authenticate(string resourceName, ClaimsPrincipal incomingPrincipal)
            {
                if (!incomingPrincipal.Identity.IsAuthenticated)
                {
                    return base.Authenticate(resourceName, incomingPrincipal);
                }
            }


If the user is anonymous then we let the base class handle the call. The Authenticate method in the base class will simply return the incoming principal so an unauthenticated user will not gain access to any further claims and will stay anonymous.

You will typically not have access to a lot of claims right after authentication. You can count with the bare minimum of the UserName and often that's all you get in a forms-based login scenario. So we will simulate a database lookup based on the user name. The user's claims are probably stored somewhere in some storage. We'll start with a skeleton:



    public override ClaimsPrincipal Authenticate(string resourceName, ClaimsPrincipal incomingPrincipal)
            {
                if (!incomingPrincipal.Identity.IsAuthenticated)
                {
                    return base.Authenticate(resourceName, incomingPrincipal);
                }

                return DressUpPrincipal(incomingPrincipal.Identity.Name);
            }

            private ClaimsPrincipal DressUpPrincipal(String userName)
            {

            }


However, in case the application is expecting more Claims than the name claim at this stage then this is the time to check the claims collection. If you need access to an "age" claim then you can check its presence through the Claims property of the ClaimsPrincipal:



    incomingPrincipal.Claims.Where(c => c.Type == "age").FirstOrDefault();


Then if the age claim is missing then you can throw an exception.

Fill in the DressUpPrincipal method as follows:



    private ClaimsPrincipal DressUpPrincipal(String userName)
            {
                List<Claim> claims = new List<Claim>();

                //simulate database lookup
                if (userName.IndexOf("andras", StringComparison.InvariantCultureIgnoreCase) > -1)
                {
                    claims.Add(new Claim(ClaimTypes.Country, "Sweden"));
                    claims.Add(new Claim(ClaimTypes.GivenName, "Andras"));
                    claims.Add(new Claim(ClaimTypes.Name, "Andras"));
                    claims.Add(new Claim(ClaimTypes.Role, "IT"));
                }
                else
                {
                    claims.Add(new Claim(ClaimTypes.GivenName, userName));
                    claims.Add(new Claim(ClaimTypes.Name, userName));
                }

                return new ClaimsPrincipal(new ClaimsIdentity(claims, "Custom"));
            }


So we add some claims based on the user name. Here you would obviously do some check in your data storage. If you have Roles in your web app, which is probably true for 99% of all .NET web apps with .NET4 and lower, then you can still use them by the 'Role' property of ClaimTypes. You can run the good old Roles.GetRolesForUser(string userName) method of ASP.NET Membership and assign the resulting string array to the Claims.

Finally we return a new ClaimsPrincipal which includes the result of our datastore lookup. Keep in mind that you need to specify the authentication type with a string to make the ClaimsPrincipal authenticated. Here we assigned "Custom" to that value, but it's up to you.

Now we need to invoke our custom auth logic. We will do that at application start-up in Global.asax. The method that we want to implement is called PostAuthenticationRequest.

First we need to get hold of the current principal, i.e. the one that has just logged in:



    protected void Application_PostAuthenticateRequest()
            {
                ClaimsPrincipal currentPrincipal = ClaimsPrincipal.Current;
            }


We will then reference our custom auth manager, pass in the current principal, retrieve the transformed principal and set the transformed principal to the current thread and current user:



    protected void Application_PostAuthenticateRequest()
            {
                ClaimsPrincipal currentPrincipal = ClaimsPrincipal.Current;
                CustomClaimsTransformer customClaimsTransformer = new CustomClaimsTransformer();
                ClaimsPrincipal tranformedClaimsPrincipal = customClaimsTransformer.Authenticate(string.Empty, currentPrincipal);
                Thread.CurrentPrincipal = tranformedClaimsPrincipal;
                HttpContext.Current.User = tranformedClaimsPrincipal;
            }


Set a breakpoint at the first row within Application_PostAuthenticateRequest and run the application. Code execution should stop at the breakpoint. Step through the code using F11. As the user is anonymous base.Authenticate will be invoked within CustomClaimsTransformer.Authenticate. The anonymous user will be set as the current principal of the thread with no claims attached.

Remove the breakpoint from within the Application_PostAuthenticateRequest(), we don't need that. Instead set a new breakpoint within CustomClaimsTransformer.Authenticate where you call the DressUpPrincipal method. We only want to stop code execution if something interesting happens.

Now use the user created before to log in. Code execution should stop within the Authenticate method as the user is not anonymous any more. Use F11 to step through the code line by line. You should see that the user receives the assigned claims and is set as the current user of the current thread and HttpContext.

Let the rest of the code run and… …you may be greeted with a mystical InvalidOperationException:

A claim of type '[http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier&#8217][8]; or '[http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider&#8217][9]; was not present on the provided ClaimsIdentity. To enable anti-forgery token support with claims-based authentication, please verify that the configured claims provider is providing both of these claims on the ClaimsIdentity instances it generates. If the configured claims provider instead uses a different claim type as a unique identifier, it can be configured by setting the static property AntiForgeryConfig.UniqueClaimTypeIdentifier.

It turns out the AntiForgeryToken is expecting BOTH the NameIdentifier and IdentityProvider claim even if the exception message says OR. If you recall our implementation of DressUpPrincipal then we omitted both. So let's go back and include the NameIdentifier:



    //simulate database lookup
                if (userName.IndexOf("andras", StringComparison.InvariantCultureIgnoreCase) > -1)
                {
                    claims.Add(new Claim(ClaimTypes.Country, "Sweden"));
                    claims.Add(new Claim(ClaimTypes.GivenName, "Andras"));
                    claims.Add(new Claim(ClaimTypes.Name, "Andras"));
                    claims.Add(new Claim(ClaimTypes.NameIdentifier, "Andras"));
                    claims.Add(new Claim(ClaimTypes.Role, "IT"));
                }
                else
                {
                    claims.Add(new Claim(ClaimTypes.GivenName, userName));
                    claims.Add(new Claim(ClaimTypes.Name, userName));
                    claims.Add(new Claim(ClaimTypes.NameIdentifier, userName));
                }


Note the ClaimTypes.NameIdentifier in the claims collection. If you run the application now you should still get the same exception which proves the point that by default BOTH claims are needed for the AntiForgeryToken. You can avoid adding both claims by adding the following bit of code in Application_Start within Global.asax:



    AntiForgeryConfig.UniqueClaimTypeIdentifier = ClaimTypes.NameIdentifier;


Re-run the application and you should be able to log in without getting the exception.

You'll notice that the text where it says 'Register' changes to Hello, [username] upon successful login. Where is that value coming from? Navigate to Views/Shared/_Layout.cshtml in the Solution Explorer and locate the following element:



    <section id="login">


This element includes a reference to a partial view "_LoginPartial". You'll find _LayoutPartial.cshtml in the same folder. At the top of that file you'll see the following code:



    @if (Request.IsAuthenticated) {
        <text>
            Hello, @Html.ActionLink(User.Identity.Name, "Manage", "Account", routeValues: null, htmlAttributes: new { @class = "username", title = "Manage" })!
            @using (Html.BeginForm("LogOff", "Account", FormMethod.Post, new { id = "logoutForm" })) {
                @Html.AntiForgeryToken()
                <a href="javascript:document.getElementById('logoutForm').submit()">Log off</a>
            }
        </text>
    }


Note the call to User.Identity.Name after 'Hello'. This is a conservative approach as the MVC4 template has no way of knowing in advance whether you're planning to make you application claims-aware. User is an IPrincipal and Identity is an IIDentity so it will work with pre-.NET4.5 claims auth identity types as well. However, we know that we use claims, so let's replace User.Identity.Name with:



    System.Security.Claims.ClaimsPrincipal.Current.FindFirst(System.IdentityModel.Claims.ClaimTypes.GivenName).Value


Run the web app, log in and you should see the GivenName claim instead of the generic IPrincipal.Name.

Since the web app is now claims-aware you can use the claims to show the email claim after login or the user's favourite colour or what have you, as long as that particular Claim is present. This was not so easily available with the good old GenericPrincipal and WindowsPrincipal objects.

Note an important thing: do you recall that we hooked up our custom auth manager in Global.asax by overriding Application_PostAuthenticateRequest()? In fact what happens is that our custom Authenticate method in CustomClaimsTransformer.cs runs with every single page request. Set a breakpoint within the method and code execution will stop every time you navigate to a page in the web app. The real implementation of the Authenticate method will most likely involve some expensive DB lookup which then also runs with every page request. That's of course not maintainable.

How can we solve that problem? By caching the outcome of the Authenticate method in a built-in authentication session.
That's what we'll look at in the next blog post.

You can view the list of posts on Security and Cryptography [here][10].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/01/registerlinkonbasicmvc4app.png?w=300&h=130
[2]: http://dotnetcodr.files.wordpress.com/2013/01/registereduserloggedon.png?w=630
[3]: http://dotnetcodr.com/2013/01/21/introduction-to-forms-based-authentication-in-net4-5-mvc4-with-c-part-1/ "Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 1"
[4]: http://dotnetcodr.com/2013/01/24/introduction-to-forms-based-authentication-in-net4-5-mvc4-with-c-part-2/ "Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 2"
[5]: http://dotnetcodr.files.wordpress.com/2013/01/defaultconnectiondb.png?w=630
[6]: http://dotnetcodr.files.wordpress.com/2013/01/firstuserinuserprofile.png?w=300&h=131
[7]: http://dotnetcodr.com/2013/02/18/introduction-to-claims-based-security-in-net4-5-with-c-part-3-claims-transformation/ "Introduction to Claims based security in .NET4.5 with C# Part 3: claims transformation"
[8]: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier&#8217
[9]: http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider&#8217
[10]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
