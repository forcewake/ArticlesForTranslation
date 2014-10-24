[Source](http://dotnetcodr.com/2013/02/28/claims-based-authentication-in-mvc4-with-net4-5-c-part-2-storing-authentication-data-in-an-authentication-session/ "Permalink to Claims-based authentication in MVC4 with .NET4.5 C# part 2: storing authentication data in an authentication session")

# Claims-based authentication in MVC4 with .NET4.5 C# part 2: storing authentication data in an authentication session

In the [previous post][1] we built a simple claims-aware MVC4 internet application. We saw that calling the Authenticate method in CustomClaimsTransformer.cs with every page refresh might not be desirable. In this post we'll look at caching possibilities so that we don't need to look up the claims of the user in the DB every time they request a page in our website.

**Basics**

The claims transformation pipeline will look as follows with auth sessions:

Upon the first page request:

1. Authentication
2. Claims transformation
3. Cache the ClaimsPrincipal
4. Produce the requested page

Upon subsequent page requests:

1. Authentication
2. Load cached ClaimsPrincipal
3. Produce the requested page

You can immediately see the benefit: we skip the claims transformation step after the first page request so we save the potentially expensive DB lookups.

By default the authentication session is saved in a cookie. However, this is customisable and there are some advanced scenarios you can do with the auth session.

The authentication session is represented by an object called SessionSecurityToken. It is a wrapper around a ClaimsPrincipal object and can be read and written to using a SessionSecurityTokenHandler.

**Demo**

Open the MVC4 project we started building in the previous post. In order to introduce auth session caching we need to start with our web.config, so open that file.

We need to define some config sections for System.identityModel and system.identityModel.services. Add the following sections within the configSection element in web.config:



    <section name="system.identityModel" type="System.IdentityModel.Configuration.SystemIdentityModelSection, System.IdentityModel, Version=4.0.0.0, Culture=neutral, PublicKeyToken=B77A5C561934E089" />
        <section name="system.identityModel.services" type="System.IdentityModel.Services.Configuration.SystemIdentityModelServicesSection, System.IdentityModel.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=B77A5C561934E089" />


Build the application so that even IntelliSense will be aware of the new config sections when you modify web.config later on.

By default the authentication session feature will only work through SSL. This is well and good but may be an overkill for a local demo app. To disable it let's add the following bit of XML somewhere within the configuration element in web.config:



    <system.identityModel.services>
        <federationConfiguration>
          <cookieHandler requireSsl="false" />
        </federationConfiguration>
      </system.identityModel.services>


Remember to turn it on again for the production environment and install your X509 certificate.

The next step in the web.config is to register the module that will handle the auth sessions. Add the following module within the system.webServer element:



    <modules>
          <add name="SessionAuthenticationModule" type="System.IdentityModel.Services.SessionAuthenticationModule, System.IdentityModel.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"></add>
        </modules>


This module will be activated at the post-authentication stage. It will attempt to acquire the auth cookie and turn it to a ClaimsPrincipal.

The last step in web.config is to register our custom claims transformation class:



    <system.identityModel>
        <identityConfiguration>
          <claimsAuthenticationManager type="ClaimsInMvc4.CustomClaimsTransformer,ClaimsInMvc4"/>
        </identityConfiguration>
      </system.identityModel>


The type value is built up as follows: [namespace.class],[assembly]. You can find the assembly name under the project properties.

We can now register the session in our code. Go to CustomClaimsTransformer.cs and update the Authenticate method as follows:



    public override ClaimsPrincipal Authenticate(string resourceName, ClaimsPrincipal incomingPrincipal)
            {
                if (!incomingPrincipal.Identity.IsAuthenticated)
                {
                    return base.Authenticate(resourceName, incomingPrincipal);
                }

                ClaimsPrincipal transformedPrincipal = DressUpPrincipal(incomingPrincipal.Identity.Name);

                CreateSession(transformedPrincipal);

                return transformedPrincipal;
            }


…where CreateSession look as follows:



    private void CreateSession(ClaimsPrincipal transformedPrincipal)
            {
                SessionSecurityToken sessionSecurityToken = new SessionSecurityToken(transformedPrincipal, TimeSpan.FromHours(8));
                FederatedAuthentication.SessionAuthenticationModule.WriteSessionTokenToCookie(sessionSecurityToken);
            }


We create a SessionSecurityToken object and pass in the transformed principal and an expiration. By default the auth session mechanism works with absolute expiration, we'll see later how to implement sliding expiration. Then we write that token to a cookie. From this point on we don't need to run the transformation logic any longer.

Go to Global.asax and comment out the Application_PostAuthenticateRequest() method. We don't want to run the auth logic upon every page request, so this is redundant code.

Instead we need to change our Login page. Go to Controllers/AccountController.cs and locate the following HTTP POST action:



    public ActionResult Login(LoginModel model, string returnUrl)


In there you'll see the following code bit:



    if (ModelState.IsValid && WebSecurity.Login(model.UserName, model.Password, persistCookie: model.RememberMe))
                {
                    return RedirectToLocal(returnUrl);
                }


It is here we'll call our auth session manager to set the auth cookie as follows:



    if (ModelState.IsValid && WebSecurity.Login(model.UserName, model.Password, persistCookie: model.RememberMe))
                {
                    List<Claim> initialClaims = new List<Claim>();
                    initialClaims.Add(new Claim(ClaimTypes.Name, model.UserName));
                    ClaimsPrincipal claimsPrincipal = new ClaimsPrincipal(new ClaimsIdentity(initialClaims, "Forms"));
                    ClaimsAuthenticationManager authManager = FederatedAuthentication.FederationConfiguration.IdentityConfiguration.ClaimsAuthenticationManager;
                    authManager.Authenticate(string.Empty, claimsPrincipal);
                    return RedirectToLocal(returnUrl);
                }


We first create an initial set of claims. We only add in the Name claim as it is sufficient for our demo purposes but you'll need to pass in everything that's needed by the claims transformation logic in the custom Authenticate method. Then we create a new ClaimsPrincipal object and pass it into our transformation logic which in turn will fetch all the necessary claims about the user and set up the auth session. Note that we new up a ClaimsAuthenticationManager by a long chain of calls, but what that does is that it extracts the registered auth manager from the web.config, which is CustomClaimsTransformer.cs. We finally call the Authenticate method on the auth manager.

This code is slightly more complicated than in pure Forms based auth scenarios where a single call to FormsAuthentication.SetAuthCookie would have sufficed. However, you get a lot more flexibility with claims; you can pass in a whole range of input claims, can call your claims transformation logic and you also get a cookie that serialises the entire claim set.

Set a breakpoint at the first row of CustomClaimsTransformer.Authenticate. Run the application now. There should be no visible difference in the behaviour of the web app, but the underlying authentication plumbing has been changed. One difference in code execution is that the Authenticate method is not called with every page request.

Now log on to the site. Code execution should stop at the break point. Insect the incoming ClaimsPrincipal and you'll see that the name claim we assigned in the Login action above is readily available:

![Name claim available after auth session][2]

Step through the code in CustomClaimsTransformer.cs and you'll see that the SessionSecurityToken has been established and written to a cookie.

**Where is that cookie?**

With the web app running in IE press F12 to open the developer tools. Click the Network tab and press Start capturing:

![Start capturing developer tools][3]

Click on the About link on the web page. You'll see that the URL list of the Developer Tool fills up with some items that the page loaded, e.g. site.css. You'll see a button with the caption 'Go to detailed view' to the right of 'Stop capturing'. This will open a new section with some new tabs. Select the 'Cookies' tab. You may see something like this:

![Authentication session cookie][4]

I highlighted 'FedAuth' which is the default name given the authentication session cookie. Its value is quite big as we're serialising a whole claim set. Don't worry much about the size of this long string. If the serialised value becomes too big, then the part that does not fit into FedAuth will be added to another cookie called FedAuth1 and so on. Cookies are partitioned into chunks of 2KB.

You may be wondering what the .ASPXAUTH cookie is doing there, which is the traditional auth cookie set in a forms-based authentication scenario. If you check the code in the Login action again you'll see a call to WebSecurity.Login. It is here that the .ASPXAUTH cookie will be set. You can read more about WebSecurity in my previous blog posts on Forms auth [here][5] and [here][6].

**Logging out**

This should be easy, right? Just press the Logout link and you're done. Well, try it! You'll see that you cannot log out; it will still say Hello, [username]. Recall that the auth session was persisted to a cookie and we specified an absolute expiration of 8 hours. Go to AccountController.cs and locate the Logoff action. You'll see a call to WebSecurity.Logout() there which only removes the .ASPXAUTH cookie. Check the cookies collection in the Developer Tools:

![web security removes aspxauth cookie][7]

However, we still have our FedAuth cookie. So even if you log out, the FedAuth cookie will be sent along every subsequent request and from the point of view of the application you are still authenticated and logged in. Add the following code the the LogOff action in order to remove the FedAuth cookie as well:



    FederatedAuthentication.SessionAuthenticationModule.SignOut();


Try again to log on and off, you should succeed.

**Events**

You can attach events to SessionAuthenticationModule:

* SessionSecurityTokenReceived: modify the token as you wish, or even cancel it
* SessionSecurityTokenCreated: modify the session details
* SignedIn/SignedOut: raise event when the user signs in or out
* SignOutError: raise event if there is an error when signing out

You can hook up the events in CustomClaimsTransformer.CreateSession as follows:



    private void CreateSession(ClaimsPrincipal transformedPrincipal)
            {
                SessionSecurityToken sessionSecurityToken = new SessionSecurityToken(transformedPrincipal, TimeSpan.FromHours(8));
                FederatedAuthentication.SessionAuthenticationModule.WriteSessionTokenToCookie(sessionSecurityToken);
                FederatedAuthentication.SessionAuthenticationModule.SessionSecurityTokenCreated += SessionAuthenticationModule_SessionSecurityTokenCreated;
            }

            void SessionAuthenticationModule_SessionSecurityTokenCreated(object sender, SessionSecurityTokenCreatedEventArgs e)
            {
                throw new NotImplementedException();
            }


**Sliding expiration**

SessionSecurityTokenReceived event is useful if you want to set a sliding expiration to the auth session. It's generally a bad idea to set a sliding expiration to a cookie; a cookie can be stolen and with sliding expiration in place it can be used forever if the expiry date is renewed over and over again. So normally the default setting of absolute expiration is what you should implement.

You can reissue the cookie in the SessionSecurityTokenReceived event as follows:



    private void CreateSession(ClaimsPrincipal transformedPrincipal)
            {
                SessionSecurityToken sessionSecurityToken = new SessionSecurityToken(transformedPrincipal, TimeSpan.FromHours(8));
                FederatedAuthentication.SessionAuthenticationModule.WriteSessionTokenToCookie(sessionSecurityToken);
                FederatedAuthentication.SessionAuthenticationModule.SessionSecurityTokenReceived += SessionAuthenticationModule_SessionSecurityTokenReceived;
            }

            void SessionAuthenticationModule_SessionSecurityTokenReceived(object sender, SessionSecurityTokenReceivedEventArgs e)
            {
                SessionAuthenticationModule sam = sender as SessionAuthenticationModule;
                e.SessionToken = sam.CreateSessionSecurityToken(...);
                e.ReissueCookie = true;
            }


You will have access to the current session token in the incoming SessionSecurityTokenReceivedEventArgs parameter. Here you have the chance to inspect the token as you wish. Here we set the SessionToken property to a new token – I omitted the constructor parameters for brevity. You can obviously use the same session creating logic as in the CreateSession method, the CreateSessionSecurityToken is just another way to achieve the same goal.

Set a break point at…



    SessionAuthenticationModule sam = sender as SessionAuthenticationModule;


…and run the application. Code execution will stop at the break point when logging in. The event will be then raised on every subsequent page request and the session security token will be reissued over and over again.

**Cookie handling**

It is a good idea to protect the auth session cookie so that it cannot easily be read. Fortunately the session token is by default protected by the token handler using DPAPI. The key that's used to encrypt the token is local to the server. This means that you will run into problems in a web farm scenario where each server has a different machine key. In this case you'll need a shared key.

You can build a new class that derives from SessionSecurityTokenHandler and set your encryption logic as you wish.

Alternatively starting with .NET4.5 there's a built-in token handler that uses the ASP.NET machine key to protect the cookie. You can find more details on the MachineKeySessionSecurityTokenHandler class [on MSDN.][8] The shared key material will be used to protect the cookie. It won't make any difference which server the browser connects to.

**Server side caching**

In case you need a large amount of claims to get the job done the auth session cookie size may grow considerably large. This may become an issue in mobile networking where the mobile device can have a slow connection. The good news is that session tokens can be cached on the server as well. It is only the session token identifier that will be sent back and forth between the browser and the server. The auth framework will find the claims collection on the server based on the identifier; you don't need to write code to find it yourself. You can achieve this in code as follows:



    SessionSecurityToken sessionSecurityToken = new SessionSecurityToken(transformedPrincipal, TimeSpan.FromHours(8));
                sessionSecurityToken.IsReferenceMode = true;


The downside is that the cookie is linked to server, i.e. we have to deal with server-affinity; if the app pool is refreshed, e.g. when you deploy your web app, or reset IIS, then the claims will be lost. Also, this solution does not work out of the box in the case of web farms. A way to solve this is to introduce your own implementation of SessionSecurityTokenCache ([Read on MSDN][9]) to connect to another server dedicated to caching, such as AppFabric caching.

This post discussed the properties of the authentication session. In the next post we'll look at claims-based authorisation in MVC4.

You can view the list of posts on Security and Cryptography [here][10].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/02/25/claims-based-authentication-in-mvc4-with-net4-5-c-part-1-claims-transformation/ "Claims-based authentication in MVC4 with .NET4.5 C# part 1: Claims transformation"
[2]: http://dotnetcodr.files.wordpress.com/2013/01/nameclaimwithauthsession.png?w=300&h=105
[3]: http://dotnetcodr.files.wordpress.com/2013/01/startcapturing.png?w=300&h=113
[4]: http://dotnetcodr.files.wordpress.com/2013/01/authsessioncookie.png?w=300&h=86
[5]: http://dotnetcodr.com/2013/01/21/introduction-to-forms-based-authentication-in-net4-5-mvc4-with-c-part-1/ "Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 1"
[6]: http://dotnetcodr.com/2013/01/24/introduction-to-forms-based-authentication-in-net4-5-mvc4-with-c-part-2/ "Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 2"
[7]: http://dotnetcodr.files.wordpress.com/2013/01/aspxauthremoved.png?w=300&h=69
[8]: http://msdn.microsoft.com/en-us/library/system.identitymodel.services.tokens.machinekeysessionsecuritytokenhandler.aspx "MSDN"
[9]: http://msdn.microsoft.com/en-us/library/system.identitymodel.tokens.sessionsecuritytokencache.aspx "MSDN"
[10]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
