[Source](http://dotnetcodr.com/2013/02/18/introduction-to-claims-based-security-in-net4-5-with-c-part-3-claims-transformation/ "Permalink to Introduction to Claims based security in .NET4.5 with C# Part 3: claims transformation")

# Introduction to Claims based security in .NET4.5 with C# Part 3: claims transformation

An important feature of ClaimsPrincipal in .NET4.5 is the unification of different credential formats. 'Unification' means that all credential types can be handled with the help of Claims. The application doesn't need to know exactly how the Claims were issued, i.e. which authentication type was used to issue the claims. The only thing we care about is the claims collection.

There's a pluggable infrastructure in .NET4.5 which understands these protocols and knows how to turn them into Claims. There's built-in support for the most common protocols, such as Windows, Forms, SSL, SAML, which all issue their credentials in different formats. They can be turned into claims using an object called the SecurityTokenHandler. One of the functions of this object is to translate one format into another.

This object has methods for:

* Reading tokens
* Validating tokens
* Writing tokens
* Determining token types

The first step is usually to read the incoming tokens. The various overloaded ReadToken methods return a SecurityToken object. The ValidateToken method accepts a SecurityToken and returns a collection of ClaimsIdentity objects. You can use this object to create your own mechanism to create tokens, a so called Security Token Service (STS), in which case methods, such as WriteToken and CreateToken will be useful. There are ready-made STSs, such as the Active Directory Federation Service (ADFS) that have the function of authenticating a user and issuing claims that describe the user. .NET4.5 includes several derived types of the SecurityTokenHandler object, such as:

* KerberosSecurityTokenHandler
* RsaSecurityTokenHandler
* SamlSecurityTokenHandler
* UserNameSecurityTokenHandler

…and many more. If the built-in types do not suit your needs then you can always create your own class by deriving from the SecurityTokenHandler object and implement the methods that you need: ReadToken and ValidateToken if you only want to read and validate tokens. Indeed, you may not even have to implement your own type this way. If all you need is an X509 certificate handler that's slightly different from the built-in X509SecurityTokenHandler then derive from this object instead of the SecurityTokenHandler object. This approach will save you a lot of time.

The processing pipeline of claims is as follows:

1. Your application receives the token in some format: XML, Binary or text
2. The token handling mechanism, i.e. the SecurityTokenHandler kicks in and deserialises the token using the ReadToken method
3. The same object will also validate the token using the ValidateToken method – this step results in a ClaimsPrincipal object
4. The resulting ClaimsPrincipal is transformed according to your needs

Why do we need the last step? Actually it is an optional step. If the ClaimsPrincipal from step 3 perfectly meets your authentication and authorisation needs then no more transformation is necessary. If the token was issued by ADFS then it will include all the claims that are usually available in Windows: the Name claim and the group SIDs. In a common web-based log-in scenario the only claim available may be the user name. If you need to check other claims during your authorisation process then you'll need to 'dress up' the ClaimsPrincipal from step 3 using some claims transformation technique.

It is also possible that the claims collection you receive from step 3 is too large. Then you may even have to remove the unnecessary claims before passing them on to your authorisation process. It is recommended to include only those claims that are absolutely necessary for your application – not more, not less.

**Claims transformation demo**

Let's take a look at how the claims can be transformed in C#.

Start Visual Studio 2012 and create a new Console application with .NET4.5 as the underlying framework. We will pretend that our application receives a WindowsPrincipal object whose claims collection needs to be transformed.

Create the following skeleton in Main.cs:



    static void Main(string[] args)
            {
                SetCurrentPrincipal();
                UseCurrentPrincipal();
            }

            private static void UseCurrentPrincipal()
            {

            }

            private static void SetCurrentPrincipal()
            {
                Thread.CurrentPrincipal = new WindowsPrincipal(WindowsIdentity.GetCurrent());
            }


In [Part 2][1] of the blog posts on Claims we saw that WindowsPrincipal includes a lot of group SID claims. It is unlikely that your application will need them all. In order to transform the claims to fit our needs we need a class that derives from an object called ClaimsAuthenticationManager in the System.Security.Claims namespace. Add a class called CustomClaimsTransformer and add a reference to System.IdentityModel.dll, otherwise the ClaimsAuthenticationManager object will not be found:



    public class CustomClaimsTransformer : ClaimsAuthenticationManager
        {
        }


The method that we'll need to override is called Authenticate:



    public class CustomClaimsTransformer : ClaimsAuthenticationManager
        {
            public override ClaimsPrincipal Authenticate(string resourceName, ClaimsPrincipal incomingPrincipal)
            {
                return base.Authenticate(resourceName, incomingPrincipal);
            }
        }


The first parameter, i.e. 'resourceName' gives us the opportunity to specify some type of context. E.g. in an ASP.NET application we can pass in the URL the user tries to access. The transformation logic may depend on the resource type.

Note that the method accepts a ClaimsPrincipal and also returns a ClaimsPrincipal. You can inspect the claims collection of the 'incomingPrincipal' object, add/remove claims directly in it and return the transformed object. The recommended way, however, is to build a brand new ClaimsPrincipal object and fill its claims collection with the necessary claim types and values. You can remove the call to the base.Authenticate method. The application will not compile as the method doesn't return any objects but we'll fix that soon.

The first step in our custom implementation is to validate the claims. In this step you can check whether all necessary **initial** claim types have come in, they are in the required format and were issued by a trusted issuer. Here we'll only check if the Name claim is null or empty:



                //validate name claim
                string nameClaimValue = incomingPrincipal.Identity.Name;

                if (string.IsNullOrEmpty(nameClaimValue))
                {
                    throw new SecurityException("A user with no name???");
                }


The next step is to build the ClaimsPrincipal and this is entirely up to you and your authentication/authorisation needs. Example: you can look at the SIDs in the WindowsPrincipal and convert them to your custom permissions. Here we'll simulate a database lookup based on the name claim. Add the following return statement to the Authenticate method:



    return CreatePrincipal(nameClaimValue);


…where the CreatePrincipal method looks as follows:



            private ClaimsPrincipal CreatePrincipal(string userName)
            {
                bool likesJavaToo = false;

                if (userName.IndexOf("andras", StringComparison.InvariantCultureIgnoreCase) > -1)
                {
                    likesJavaToo = true;
                }

                List<Claim> claimsCollection = new List<Claim>()
                {
                    new Claim(ClaimTypes.Name, userName)
                    , new Claim("http://www.mysite.com/likesjavatoo", likesJavaToo.ToString())
                };

                return new ClaimsPrincipal(new ClaimsIdentity(claimsCollection, "Custom"));
            }


We're only interested in two claims: the name claim and whether the user likes writing Java as well. Instead of the userName.IndexOf call we would of course have some database lookup to see who actually likes Java. Then we build our claims collection and pass it in the ClaimsPrincipal through the ClaimsIdentity object. "Custom" is the authentication type description, can be whatever value, it's up to you. Recall from [Part 1][2] of the blog posts on Claims that you need to set the authentication type to some value in order to turn this Identity into an authenticated one. It's not enough for the user to have a name, you need to provide the authentication type as well.

You need to register your custom claims transformer in app.config. Before you do that add a reference to the System.identitymodel.services.dll. In app.config insert a new config section as follows:



    <configSections>
        <section name="system.identityModel" type="System.IdentityModel.Configuration.SystemIdentityModelSection, System.IdentityModel, Version=4.0.0.0, Culture=neutral, PublicKeyToken=B77A5C561934E089"/>
      </configSections>


You need to register the custom claims transformer as follows:



      <system.identityModel>
        <identityConfiguration>
          <claimsAuthenticationManager type="ClaimsTransformationDemo.CustomClaimsTransformer,ClaimsTransformationDemo"/>
        </identityConfiguration>
      </system.identityModel>


The type value may differ in your case depending on the name you gave your project. You'll need to give the fully qualified name of the class, i.e. namespace and the class name and then the assembly name after the comma.

In order to invoke the transformation logic we don't set the incoming WindowsPrincipal object to the current principal of the thread as we currently do it in SetCurrentPrincipal(). We need to get hold of the incoming principal, pass it in our custom Authenticate method, get the transformed ClaimsPrincipal object and set it to the current principal of the thread. This can be done in 2 lines of code but the transformer invocation call is quite long:



    private static void SetCurrentPrincipal()
            {
                WindowsPrincipal incomingPrincipal = new WindowsPrincipal(WindowsIdentity.GetCurrent());
                Thread.CurrentPrincipal = FederatedAuthentication.FederationConfiguration.IdentityConfiguration
                    .ClaimsAuthenticationManager.Authenticate("none", incomingPrincipal);
            }


You will need to import the System.IdentityModel.Services namespace where the FederatedAuthentication object is located. We don't care about the resource name so we pass in a "none". Set a breakpoint at the Authenticate method of the CustomClaimsTransformer class and run the application. You may receive an exception saying that you'll need a reference to the System.Web assembly, so do exactly as it says. So, run the application with the breakpoint at Authenticate and see if it is invoked.

If all went well then code execution is stopped at the breakpoint meaning that your custom claims transformation logic has successfully been registered and invoked. Step through the code with F11 and check that each line executes as expected. Inspect the contents of Thread.CurrentPrincipal and you'll see that it has two claims; the name claim and our custom 'likesjavatoo' claim:

![Claims transformation success][3]

From this point on our application will not see all the 'noise' in the claims collection, i.e. all the unnecessary Windows-related claims. It will only see the ones that are needed.

We discussed how to populate an incoming Claims collection to one that fits our authentication and authorisation needs further down in the code. In the next post we'll see how to use the claims to authorise access to certain parts of the application.

You can view the list of posts on Security and Cryptography [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/02/14/introduction-to-claims-based-security-in-net4-5-with-c-part-2-the-new-inheritance-model/ "Introduction to Claims based security in .NET4.5 with C# Part 2: the new inheritance model"
[2]: http://dotnetcodr.com/2013/02/11/introduction-to-claims-based-security-in-net4-5-with-c-part-1/ "Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics"
[3]: http://dotnetcodr.files.wordpress.com/2013/01/claimstransformation.png?w=300&h=115
[4]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
