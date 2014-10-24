[Source](http://dotnetcodr.com/2013/03/04/claims-based-authentication-in-mvc4-with-net4-5-c-part-3-claims-based-authorisation/ "Permalink to Claims-based authentication in MVC4 with .NET4.5 C# part 3: claims based authorisation")

# Claims-based authentication in MVC4 with .NET4.5 C# part 3: claims based authorisation

In the [previous post][1] we discussed how to the save the authentication session so that we didn't need to perform the same auth logic on every page request. In this post we will look at how authorisation can be performed using claims in an MVC project.

**Introduction**

There are two main approaches to authorisation in an ASP.NET web application: pipeline authorisation and Intra-app authorisation.

**Pipeline auth** means performing coarse grained, URL-based authorisation. You may require the presence of a valid auth header in every request that comes to your server. Or the authenticated user must be in a certain Role in order to reach a certain protected URL. The advantage with this approach is that authorisation happens very early in the application lifecycle so you can reject a request very early on. In this scenario you will typically have little info about the user and what resource they are trying to access but these can be enough to reject a large number of users.

An example of pipeline auth in our simple MVC4 web we've been working on this series can be found in CustomClaimsTransformer.Authenticate. This is the stage where you can check the presence of a certain claim that your auth logic absolutely must have in order to make an early decision. If it's missing, then you may not care about what the user is trying to do, the request will be rejected.

Another example of pipeline auth comes from the good old 'location' elements in an ASP.NET web forms config where you could specify URL-based auth:



    <location path="customers">
        <system.web>
          <authorization>
            <allow roles="IT"/>
            <deny users="*"/>
          </authorization>
        </system.web>
      </location>


This is an acceptable approach in web-forms projects where the URL has a close affinity to the project file system, i.e. the value of the 'path' attribute represents an .aspx file. In MVC /Customers will of course not lead to an aspx page called Customers. In MVC urls and resources are unlikely to have a one-to-one match. You don't call physical files the same way as in a web-forms app. If the routing mechanism is changed then the path attribute will be meaningless. So all of a sudden people will have access to previously protected parts of your web app. Generally try to avoid this approach in an MVC application as it creates a tight coupling between the routing table and the project file structure.

Yet another example of pipeline auth is the ClaimsAuthorisationManager which can be registered in the web.config. This will sound familiar to you if you looked at the [post][2] on the very basics of claims. This is also a URL based approach, but it's based on Claims and not Roles.

**Intra-app auth** on the other hand means fine-grained checks within your code logic. The benefit is that you have the chance to collect as much information as possible about the user and the resources they are trying to use. Then you can tweak your authorisation logic on a wider information basis. In this scenario you will have more info on the user and make your reject/accept decision later in the app lifecycle than in the Pipeline auth scenario.

A definite advantage of this approach is that it is not URL based any more so it is independent of the routing tables. You will have more knowledge about the authorisation domain because you'll typically know exactly what claims the user holds and what they are trying to achieve on your site.

**PrincipalPermission and ClaimsPrincipalPermission**

You can follow a declarative approach using the ClaimsPrincipalPermission attribute or an imperative one within the method body. Either way you'll work with Claims and not Roles as in the 'old' days with the well-known 'Role="IT"' and .IsInRole("Admin") type of checks:



    [PrincipalPermission(SecurityAction.Demand, Role="IT")]


The old way of performing authorisation is not recommended now that we have access to claims in .NET4.5. Roles encouraged you to mix authorisation and business logic and they were limited to, well, Roles as the way of controlling access. However, you might have required more fine-grained control over your decision making. Then you ended up with specialised roles, like Admin, SuperAdmin, SuperSuperAdmin, MarketingOnThirdFloor etc. Decorating your methods with the PrincipalPermission attribute also disrupts unit testing as even the unit testing thread must have a User in the Role specified in the attribute. Also, if the current principal is not in the required group then an ugly security exception is thrown which you have to deal with somehow.

In [this][2] post we saw a detailed discussion on the ClaimsPrincipalPermission which replaces the PrincipalPermission. Here comes an example to refresh your memory:



    [ClaimsPrincipalPermission(SecurityAction.Demand, Operation="Show", Resource="Code")]


In short: we don't care which group or role the user is in any longer. This attribute describes the method it decorates. It involves a 'Show' operation on the 'Code' resource. If the current user wants to run this method then they better make sure that they have these claims. It will be the ClaimsAuthorizationManager that decides if the current principal is allowed to call the action 'Show' on the resource 'Code'. The principal still must have certain claims, just like they had to be in a certain Role before. However, the authorisation logic is now separated out to a different part of the application. You can even have that logic in a web service on a different machine so that the auth logic can be handled entirely separately from your application.

Another benefit is the following: what constitutes a certain Role can change over time. What is 'IT'? Who belongs to that group? So later on you may have to come back to every method with the attribute 'Role="IT"' and change it to e.g. "Geeks" because 'IT' has changed its definition at your company. On the other hand a method that has the function to 'Show' a resource called 'Code' will probably have that function over a long time, possible over the entire life time of the finalised production version of the application.

So, this attribute solves some of the problems with the PrincipalPermission. However, it does not solve all of them. It still gets in the way of unit testing and it still throws a SecurityException.

**The Authorize attribute**

The MVC 'equivalent' of the ClaimsPrincipal attribute is the Authorize attribute. It is still limited to roles:



    [Authorize]
    public ActionResult ShowMeTheCode()

    [Authorize(Roles="IT")]
    public ActionResult ShowMeTheCode()


It does not use the action/resource properties of the method and you still mix your auth logic with the 'real' application code leading to the same Separation of Concerns problem we mentioned above. However, this attribute is not invoked during unit testing and it does not throw Exceptions either. Instead, it returns a 404 which is a lot nicer way of dealing with unauthorised access.

We are only one step from the MVC4 claims-based authorisation nirvana. It would be great to have an Authorize attribute where you can specify the Resource and the Action just like in the case of ClaimsPrincipalPermission. You could derive from the existing Authorize attribute and implement this kind of logic there. The good news is that this has been done for you and it can be downloaded from NuGet. The NuGet package includes the imperative equivalent of the declarative attribute as well. So if you need to check if the user has access rights within a certain method, then there's a claims-enabled solution in MVC4. We'll use this attribute in the demo.

**Demo**

The initial steps of building the authorisation module have been outlined in [this][2] blog post. I will not repeat all of the details here again.

Open up the project where we left off in the previous blog post. If you remember then we included a CustomClaimsTransformer class to implement our own claims transformation logic. This is our claims based authentication module. We would like to extend the project to include authorisation as well.

First add a new class to the web project called CustomAuthorisationManager. It will need to derive from ClaimsAuthorizationManager in the System.Security.Claims namespace:



    public class CustomAuthorisationManager : ClaimsAuthorizationManager
        {
            public override bool CheckAccess(AuthorizationContext context)
            {
                return base.CheckAccess(context);
            }
        }


Recall that you can extract the Resource, the Action and the Principal from the AuthorizationContext object parameter.

Now let's say we want to make sure that only those with the name Andras who live in Sweden are allowed to view the code. I would do it as follows:



    public override bool CheckAccess(AuthorizationContext context)
            {
                string resource = context.Resource.First().Value;
                string action = context.Action.First().Value;

                if (action == "Show" && resource == "Code")
                {
                    bool livesInSweden = context.Principal.HasClaim(ClaimTypes.Country, "Sweden");
                    bool isAndras = context.Principal.HasClaim(ClaimTypes.GivenName, "Andras");
                    return isAndras && livesInSweden;
                }

                return false;
            }


Set a breakpoint at the first row of the method body, we'll need it later.

This should be straightforward: we extract the Action and the Resource – note that there can be multiple values, hence the 'First()' – and then check where the user lives and what their given name is. If those claims are missing or are not set to the required values then we return false.

Next we have to register this class in the web.config under the claimsAuthenticationManager we registered in the previous part:



    <system.identityModel>
        <identityConfiguration>
          <claimsAuthenticationManager type="ClaimsInMvc4.CustomClaimsTransformer,ClaimsInMvc4" />
          <claimsAuthorizationManager type="ClaimsInMvc4.CustomAuthorisationManager,ClaimsInMvc4"/>
        </identityConfiguration>
      </system.identityModel>


The type attribute is formatted as follows: [namespace.classname],[assembly].

Next we want to make sure that this logic is called when a protected action is called. We will try the claims-enabled version of the MVC4 Authorize attribute. Right-click 'References' and select 'Manage NuGet Packages…'. Search for 'Thinktecture' and install the below package:

![Thinktecture auth package NuGet][3]

This package will give you access to a new attribute called ClaimsAuthorize where you can pass in the Action and Resource parameters.

Imagine that our About page includes some highly sensitive data that can only be viewed by the ones specified in CustomAuthorisationManager.CheckAccess. So let's decorate the About action of the Home controller. Note that the attribute comes in two versions: one for MVC4 and one for WebAPI. If you haven't heard of Web API, then it is a technology to build RESTful web services whose structure is very much based on MVC. You can read more about it [here][4].

Reference the version for Mvc:

![Two versions of claims authorize][5]

…and decorate the About action as follows:



    [ClaimsAuthorize("Show", "Code")]
            public ActionResult About()
            {
                ViewBag.Message = "Your app description page.";

                return View();
            }


This is telling us that the About action will perform a 'Show' action on the resource called 'Code'.

Run the application now. Click on the 'About' link without logging in first. You should be redirected to the Log-in page. Enter the username and password and press the 'Log in' button. If everything went well then code execution should stop at our breakpoint within CustomAuthorisationManager.CheckAccess. Step through the method using F11 to see what happens. You can even inspect the AuthorizationContext object in the Locals window to see what it contains:

![AuthorizationContext object][6]

If the logged on user has the correct claims then you should be redirected to the About page. I will here again stress the point of getting away from the traditional Roles based authorisation of ASP.NET. We are not dealing with Roles any longer. We do not care who is in which group. Instead we describe using the Action and Resource parameters of the ClaimsAuthorize attribute what the logged on user is trying to achieve on our website. Based on that information we can make a better decision using the claims of the user whether to allow or deny access. The auth logic is separated away from the 'real' application in a class on its own which is called automatically if it is registered in web.config. The auth logic can even be 'outsourced' to a web service which can even be the basis of a separate user management application.

You can specify multiple Resource values in the attribute as follows:



    [ClaimsAuthorize("Show", "Code", "TvProgram", "Fireworks")]
            public ActionResult About()
            {
                ViewBag.Message = "Your app description page.";

                return View();
            }


…i.e. you just pass in the names of the Resources after the Action.

You can achieve the same imperatively within the method body as follows:



    public ActionResult About()
            {
                if (ClaimsAuthorization.CheckAccess("Show", "Code"))
                {
                    ViewBag.Message = "This is the secret code.";
                }
                else
                {
                    ViewBag.Message = "Too bad.";
                }

                return View();
            }


The CheckAccess method has an overloaded version which accepts an AuthorizationContext object, which gives the highest degree of freedom to specify all the resources and actions that are needed by the auth logic.

In case you wish to protect the entire controller, then it's possible as well:



    [ClaimsAuthorize("Show", "Everything")]
        public class HomeController : Controller


If you want to apply the attribute to the entire application you can do it by adding the attribute to the global filters in App_Data/FilterConfig as follows:



    public class FilterConfig
        {
            public static void RegisterGlobalFilters(GlobalFilterCollection filters)
            {
                filters.Add(new HandleErrorAttribute());
                filters.Add(new ClaimsAuthorizeAttribute());
            }
        }


This discussion should be enough for you to get started with Claims-based authentication and authorisation in an MVC4 internet application. In the next post we'll start looking at separating out the login mechanism entirely: Single SignOn and Single SignOut.

You can view the list of posts on Security and Cryptography [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/02/28/claims-based-authentication-in-mvc4-with-net4-5-c-part-2-storing-authentication-data-in-an-authentication-session/ "Claims-based authentication in MVC4 with .NET4.5 C# part 2: storing authentication data in an authentication session"
[2]: http://dotnetcodr.com/2013/02/21/introduction-to-claims-based-security-in-net4-5-with-c-part-4-authorisation-with-claims/ "Introduction to Claims based security in .NET4.5 with C# Part 4: authorisation with claims"
[3]: http://dotnetcodr.files.wordpress.com/2013/02/thinktectureidentitymodelnuget.png?w=300&h=46
[4]: http://www.asp.net/web-api "Web API"
[5]: http://dotnetcodr.files.wordpress.com/2013/02/twoversionsofclaimsauthorize.png?w=300&h=67
[6]: http://dotnetcodr.files.wordpress.com/2013/02/authorizationcontextobject.png?w=300&h=87
[7]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
