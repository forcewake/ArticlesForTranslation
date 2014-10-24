[Source](http://dotnetcodr.com/2013/03/07/claims-based-authentication-in-net4-5-mvc4-with-c-external-authentication-with-ws-federation-part-1/ "Permalink to Claims-based authentication in .NET4.5 MVC4 with C#: External authentication with WS-Federation Part 1")

# Claims-based authentication in .NET4.5 MVC4 with C#: External authentication with WS-Federation Part 1

Our model MVC4 internet applications in this series had one important feature in common: they all provided the authentication logic internally.

This is the traditional approach to logging in on a web page: there is a login page within the application which provides the gateway to the protected parts of the website. Upon providing the login credentials and pressing the 'Log in' button the web application will check the the validity of those credentials against some data store and accept or reject the request.

We will build on the demo application from the [previous post][1] on claims in MVC4. If you are new to Claims in .NET4.5 then I'd recommend that you start from the [beginning][2].

**External authentication: introduction**

There are several reasons why the internal auth approach might not be the most suitable one:

* This is not a trivial exercise: logging in and out must happen in a secure way
* Your attention therefore may be diverted from the 'true' purpose of your application, i.e. the very reason for its existence
* You may not like programming in Security-related topics which holds you back from writing the 'real' application logic of your app
* Multiple authentication types are often problematic to implement: you can typically only provide one specific type of authentication on your site and it's usually a Forms-based one
* As the auth logic is internal to your app it is difficult to re-use in other apps that need the same type of login: the result is a copy-paste type of horror

Thus it would be nice to somehow factor out the authentication logic in a separate project/application which can perform the authentication for your web app and for any other apps that also need authentication against the same user store. The benefits of such a scenario are the following:

* Multiple applications can share the login logic
* Keep the authentication logic in one place and avoid the copy-paste scenario: if the logic changes it will be automatically propagated in all consuming applications, also called the **RELYING PARTIES**
* It's possible to re-use the auth session across several applications so that the user does not need to log in on multiple sites: this is called Single SignOn
* The external apps, i.e. the relying parties can get rid of their internal auth data allowing developers to concentrate on the 'real stuff'
* The responsibilities are more clearly divided: the relying party carries out the business logic and the auth app takes care of the authentication
* The relying parties can establish a trust relationship with the auth app using **Federation**: this is important as the external apps should not blindly accept a authentication result as it may come from an unreliable source
* The team of developers can be divided up more efficiently: domain experts who work on the real business logic and security experts that work on the authentication and user store part
* You can put the external auth app anywhere: on a different physical server, in the cloud, behind some web service, etc.
* Your web app can be set up to accept claims from multiple authentication services: as long as the claims are coming from a trusted source your web app will not care which one they are coming from

What would such a scenario look like? First I'll try to describe the scenario in words.

The external authentication app we have been talking about is called a Security Token Service, or an STS in short. It is also called an Identity Provider. The STS is a normal website with its own login page sitting on some web server.

Imagine the following:

* You have a web page that relies on external authentication
* Thus it will be void of all types of auth logic and it will have no Login page either
* A client wishes to reach a protected page within your web app
* The client will then be redirected to the **LOGIN PAGE OF THE STS**
* The STS performs the authentication and issues a security token to the client upon successful login
* This token, which we'll talk more about later, probably does not include too many claims: user ID, user name, email
* This token will also include an identifier that identifies the issuer of the token in a reliable way
* The token is sent back to the client which is then redirected to the external application where the user originally wanted to log in
* The relying party inspects the token, checks the issuer, maybe transforms the claims and can reject or accept the user depending on the validity of the token and the claims within the token
* Example: if the issuer of the token is not coming from a trusted auth service, the signature in the token has been tampered with or an important claim is missing or is malformed then you can still reject the request in your web app very early on
* If everything is fine with the token then the relying web app will establish a ClaimsPrincipal the same way as we saw before in related blog posts

The flow can be shown graphically as follows:

![An STS model][3]

The security token is meaningless for the client. As mentioned above, it will be used by your web app to check its validity, transform the claims etc. Also, just to stress the point, it is not important any more where the STS is located.

**Security Assertion Markup Language: SAML**

You may be wondering what the security token issued by the STS looks like. There are some standard and certainly lots of company-specific formats out there. The default in .NET4.5 follows the SAML format, which is sort of a specialised XML. Here comes a portion of such a token from [Wikipedia][4]:

![SAML example][5]

You'll see the Issuer, the X509 cert data, i.e. the digital signature and the NameID in the picture. The signature will be used to see if the token has been tampered with after it left the STS and if the issuer is a trusted one. There's typically not much else shown in a SAML token. It is up to the STS what kind of data it will include in the SAML token. The STS may provide a different set of initial claims depending on the type of application wishing to be authenticated. The good news is that you will not have to work with SAML directly; .NET will translate the XML into Claims automatically. It is also important to note that if you have complete control over the STS then it is up you what you include in the SAML: anything from UserId to EyeColour and FavouriteBand can be sent along.

**WS-Federation**

The protocol that makes this trust relationship and token communication possible is called WS-Federation. It is a standard and is now available in .NET4.5. The flow of communication in words is as follows:

* The client tries to access a protected page on your Claims-enabled site by sending a HTTP GET request
* .NET will see that the request is void of any security token so it will be redirected to the Login page of the STS by another HTTP 302 request
* The URL of the redirect will include a special query string that may look something like this: wsfed?wa=wsignin1.0&wtrealm=[ID of relying party]
* The query string says that we want to sign in to a certain Realm, which is the identifier of the relying party, usually its URL
* Upon successful login the STS somehow needs to send the SAML token to the relying party, so let's stop here for a second…

The STS will send back a form with method = "POST" which will be redirected from the client to the relying party. This form might look like the following:



    <form method="post" action="address of relying party">
        <input name="wresult" value="<saml:assertion..." />
        <script>
            window.setTimeout('document.forms[0].submit()', 0);
        </script>
    </form>


The STS attaches the SAML to the value attribute of the input field within the form. The form is then submitted using a very simple piece of embedded JavaScript. Let's continue with the flow:

* The form is POSTed back to the relying party from the client
* The relying party will validate the token and its contents and turn it into an Identity

It's important to stress that this is not some Microsoft specific framework targeting .NET applications only. WS-Federation is part of the larger WS* family of web service specifications. It can happen that you have an STS built with .NET and a Ruby on Rails web app that you would like to connect to the STS. The fact that the STS was implemented using .NET is an unimportant detail in the bigger picture as the communication is based on a widely accepted standard. If you are in this situation then you need to check if Ruby has built-in support for WS-Federation, which I'm pretty sure it does although I know precious little about that framework.

**Security Token Service**

What does an actual STS look like then? There are several commercial products out there. Examples:

.NET4.5 includes base classes that allow you to build your own STS. Beware though that this is not a trivial exercise. You must be very knowledgeable and experienced in programming Security.

There's an open source STS available on GitHub: [Thinktecture IdentityServer][6] which we'll take a closer look at in the next blog post.

For now you won't need any of the real STS solutions out there while developing your solution. You can download an extension to Visual Studio which enables you to use a Development STS with pre-set claims. We will use this in the demo.

**Demo**

You will need to download and install the Identity and Access Tool extension from [here][7] for the demo.

This is a great tool for development purposes; you won't need a real STS but you can still write your code that accepts the security token as if it comes from a real STS. Then when you're done you simply replace the tool with the STS of your choice.

Open the MVC4 application from the previous post. As it currently stands this application still uses Forms-based authentication and we'll try to convert it to a Claims-based one.

Before we change anything let's note some important identity-related aspects of web.config:

1\. We have our system.identityModel section where we registered the custom authentication and custom authorisation managers:



    <system.identityModel>
        <identityConfiguration>
          <claimsAuthenticationManager type="ClaimsInMvc4.CustomClaimsTransformer,ClaimsInMvc4" />
          <claimsAuthorizationManager type="ClaimsInMvc4.CustomAuthorisationManager,ClaimsInMvc4" />
        </identityConfiguration>
      </system.identityModel>


2\. We let users log in by their usernames and passwords on our login page:



    <authentication mode="Forms">
          <forms loginUrl="~/Account/Login" timeout="2880" />
        </authentication>


3\. We registered a session authentication module under the modules node:



    <modules>
          <add name="SessionAuthenticationModule" type="System.IdentityModel.Services.SessionAuthenticationModule, System.IdentityModel.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089"></add>
    </modules>


4\. There is no authorization element, meaning we let anonymous users view the unprotected pages of the website.

Upon successful installation of the Identity and Access Tool you should see a new menu point when you right-click the project:

![Identity tool menu point][8]

Click on the underlined menu point which will open up the Identity and Access window:

![Identity and access window][9]

You have here a number of options to add an STS to your project:

* Local Development STS is the one you'll want to use for development purposes if you don't have a real STS available
* A business identity provider, like the ones listed above, e.g. Oracle
* An Azure cloud based STS

Select the first option. You can then select the 'Local Development STS' tab:

![Local development STS tab][10]

You will see a list of test claims that the web application will receive, such as the name 'Terry'. Again, keep in mind that there's no way to directly log on to a fully claims-based web app; here we pretend that an external STS is sending these claims to your application after a user has successfully signed in on the login page of the STS. You can configure this list according to the needs of your token validation and authorisation logic.

Change the value of the name claim, i.e. the very first one to the name of the user you created in the previous blog posts, so I've changed mine to 'Andras'.

You can select the SAML version: either 1.1 or 2.0. This depends on the available versions of the STS of your choice. In our case it doesn't make any difference, so leave option 1.1 selected.

Click OK and let's see what happens. At first you won't see any changes. Let's inspect web.config though:

1\. The system.identityModel has been extended to include claims-related elements:



    <system.identityModel>
        <identityConfiguration>
          <claimsAuthenticationManager type="ClaimsMvc.CustomClaimsTransformer,ClaimsMvc" />
          <claimsAuthorizationManager type="ClaimsMvc.CustomAuthorisationManager,ClaimsMvc" />
          <audienceUris>
            <add value="http://localhost:2533/" />
          </audienceUris>
          <issuerNameRegistry type="System.IdentityModel.Tokens.ValidatingIssuerNameRegistry, System.IdentityModel.Tokens.ValidatingIssuerNameRegistry">
            <authority name="LocalSTS">
              <keys>
                <add thumbprint="9B74CB2F320F7AAFC156E1252270B1DC01EF40D0" />
              </keys>
              <validIssuers>
                <add name="LocalSTS" />
              </validIssuers>
            </authority>
          </issuerNameRegistry>
          <certificateValidation certificateValidationMode="None" />
        </identityConfiguration>
      </system.identityModel>


We will discuss these elements in more detail in the next blog post. Note the following: the Identity and Access Tool is periodically updated and can be downloaded from within Visual Studio. Select Extensions and Updates… in the Tools menu. Make sure you check if there are any updates available under the Updates menu point:

![Tools updates in Visual Studio][11]

When I published the first version of this post – some time in March 2013 – the above XML was slightly different. I updated the Identity and Access Tool on 12 May 2013 which yielded the above system.identityModel node. It is possible that when you read this post the Access Tool will again yield something different. Let me know in the comments section if you notice a change and I'll update this post accordingly.

2\. Forms-based login is gone:



    <authentication mode="None" />


3\. The modules element has been extended with WS-Federation:



    <modules>
          <add name="SessionAuthenticationModule" type="System.IdentityModel.Services.SessionAuthenticationModule, System.IdentityModel.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089">
          </add>
          <remove name="FormsAuthentication" />
          <add name="WSFederationAuthenticationModule" type="System.IdentityModel.Services.WSFederationAuthenticationModule, System.IdentityModel.Services, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" preCondition="managedHandler" />
        </modules>


4\. By default we'll deny access to anonymous users:



    <authorization>
          <deny users="?" />
        </authorization>


Run the application and you may be greeted with the following error message:

![Must have admin rights to local STS][12]

If you started VS in admin mode then you shouldn't see this, I'll just restart mine.

Watch the browser bar carefully while the page is loading. At some point there should be a URL similar to this:

[http://localhost:12175/wsFederationSTS/Issue/?wa=wsignin1.0&wtrealm=http%3a%2f%2flocalhost%3a2533%2f&wctx=rm%3d0%26id%3dpassive%26ru%3d%252f&wct=2013-05-12T12%3a22%3a58Z][13]

This is the external 'login page', but there's of course no external login page of the model STS. This is what's happening:

* Web.config has been changed by the identity tool to deny access to all anonymous users
* When you run the application you will initially be an anonymous user
* Your request is redirected to the model STS page on localhost:12175. Remember that this was the port number that we selected in the Identity and Access window. Don't worry if yours has a different port number, it doesn't make any difference
* You will probably recognise the format of the URL with '?wa=wsignin1.0&wtrealm=' followed by the URL of the MVC4 website
* The local STS returns the list of claims we specified in the Identity and Access window
* The request is redirected to our web page and the user is logged in
* The request is redirected by the forms-based mechanism we discussed above where the form containing the SAML value of the authentication token was submitted by JavaScript

Recall that we protected the About page with the ClaimsAuthorize attribute:



    [ClaimsAuthorize("Show", "Code")]
            public ActionResult About()


…which will activate our custom authorisation logic in CustomAuthorisationManager.cs:



    public class CustomAuthorisationManager : ClaimsAuthorizationManager
        {
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
        }


Add two breakpoints to the application: one within CustomClaimsTransformer.Authenticate and one within CustomAuthorisationManager.CheckAccess. Re-run the application. If the code execution hasn't stopped then click the Log off link to force a new 'login' via the local STS. Code execution should stop at CustomClaimsTransformer.Authenticate. This is good news as our custom auth manager still kicks in and dresses up the Principal with our custom claims…:



    private ClaimsPrincipal DressUpPrincipal(String userName)
            {
                List<Claim> claims = new List<Claim>();

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

                return new ClaimsPrincipal(new ClaimsIdentity(claims, "Custom"));
            }


…and also establishes the authentication session as per the CreateSession method. Now click the About link on the front page. As this is a protected page code execution will stop within CustomAuthorisationManager.CheckAccess which shows that even this custom manager class works as it should. Upon successful authorisation the About page should load as expected.

So our previous investments are still worth the effort. The external login doesn't invalidate our claims authentication and claims transformation logic.

In the next post we'll look at the changes in web.config in more details and hook up our MVC4 with a real STS.

You can view the list of posts on Security and Cryptography [here][14].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/03/04/claims-based-authentication-in-mvc4-with-net4-5-c-part-3-claims-based-authorisation/ "Claims-based authentication in MVC4 with .NET4.5 C# part 3: claims based authorisation"
[2]: http://dotnetcodr.com/2013/02/11/introduction-to-claims-based-security-in-net4-5-with-c-part-1/ "Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics"
[3]: http://dotnetcodr.files.wordpress.com/2013/02/stspicture1.png?w=630&h=471
[4]: http://en.wikipedia.org/wiki/SAML_2.0 "SAML on Wikipedia"
[5]: http://dotnetcodr.files.wordpress.com/2013/02/samlexample.png?w=630&h=569
[6]: http://thinktecture.github.com/Thinktecture.IdentityServer.v2/ "ThinkTecture identity server"
[7]: http://goo.gl/evBj7 "Identity and access tool"
[8]: http://dotnetcodr.files.wordpress.com/2013/02/identitytoolmenupoint.png?w=630
[9]: http://dotnetcodr.files.wordpress.com/2013/02/identityandaccesswindow.png?w=630&h=232
[10]: http://dotnetcodr.files.wordpress.com/2013/02/localdevelopmentststab.png?w=630&h=465
[11]: http://dotnetcodr.files.wordpress.com/2013/03/toolsupdatesinvisualstudio.png?w=630
[12]: http://dotnetcodr.files.wordpress.com/2013/02/noaccesstosts.png?w=630
[13]: http://localhost:12175/wsFederationSTS/Issue/?wa=wsignin1.0&wtrealm=http%3a%2f%2flocalhost%3a2533%2f&wctx=rm%3d0%26id%3dpassive%26ru%3d%252f&wct=2013-05-12T12%3a22%3a58Z
[14]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
