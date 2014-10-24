[Source](http://dotnetcodr.com/2013/02/11/introduction-to-claims-based-security-in-net4-5-with-c-part-1/ "Permalink to Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics")

# Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics

The well-known built-in Identity objects, such as GenericPrincipal and WindowsPrincipal have been available for more than 10 years now in .NET. Claims were introduced in .NET4.5 to build Claims based authentication into the framework in the form of ClaimsIdentity and ClaimsPrincipal in the System.Security.Claims namespace.

Before we see some action in terms of code let's look at a general description of claims. If you already know what they are and want to see the code then you can safely jump this section.

**What is a claim?**

A claim in the world of authentication and authorisation can be defined as a statement about an entity, typically a user. A claim can be very fine grained:

* Tom is an administrator
* Tom's email address is tom@yahoo.com
* Tom lives in Los Angeles
* Tom's allowed to view sales figures between 2009 and 2012
* Tom's allowed to wipe out the universe

If you are familiar with Roles in the ASP.NET Membership framework then you'll see that Roles are also claim types. Claims come in key-value pairs:

* Role: administrator
* Email: tom@yahoo.com
* Address: Los Angeles
* Viewable sales period: 2009-2012

These statements are a lot more expressive than just putting a user in a specific role, such as marketing, sales, IT etc. If you want to create a more fine-grained authorisation process then you'll need to create specialised roles, like the inevitable SuperAdmin and the just as inevitable SuperSuperAdmin.

Claims originate from a trusted entity other than the one that is being described. This means that it is not enough that Tom says that he is an administrator. If your company's intranet runs on Windows then Tom will most likely figure in Active Directory/Email server/Sales authorisation system and it will be these systems that will hold these claims about Tom. **The idea is that if a trusted entity such as AD tells us things about Tom then we believe them a lot more than if Tom himself comes with the same claims**. This external trusted entity is called an **Issuer**. The KEY in the key-value pairs is called a **Type** and the VALUE in the key-value pair is called the **Value**. Example:

* Type: role
* Value: marketing
* Issuer: Active Directory

Trust is very important: normally we trust AD and our own Exchange Server, but it might not be as straightforward with other external systems. In such cases there must be something else, e.g. a certificate that authenticates the issuer.

An advantage of claims is that they can be handled by disparate authentication systems which can have different implementations of claims: groups, roles, permissions, capabilities etc. If two such systems need to communicate through authorisation data then Claims will be a good common ground to start with.

**How do we attach these claims to the Principals?**

A Claim is represented in .NET4.5 by an object called… …Claim! It is found in the System.Security.Claims namespace.

There is a new implementation of the well-known IIdentity interface called ClaimsIdentity. This implementation holds an IEnumerable of Claim objects. This means that this type of identity is described by an arbitrary number of statements.

Not surprisingly we also have a new implementation of IPrincipal called ClaimsPrincipal which holds a read-only collection of ClaimsIdentity objects. Normally this collection of identities will only hold one ClaimsIdentity element. In some special scenarios with disparate systems a user can have different types of identities if they can identify themselves in multiple ways. The ClaimsPrincipal implementation accommodates this eventuality.

The advantage with having these new objects deriving from IPrincipal and IIdentity is compatibility. You can start introducing Claims in your project where you work with IPrincipals and IIdentities without breaking your code.

**It's been enough talk, let's see some action**

Start Visual Studio 2012 and create a new Console application with .NET4.5 as the underlying framework.

The simplest way to create a new Claim is by providing a Type and a Value in the Claim constructor:



    static void Main(string[] args)
            {
                Claim claim = new Claim("Name", "Andras");
            }


You can use strings to describe types as above but we all know the disadvantages with such hard-coded strings. There is an enumeration called ClaimTypes that stores the most common claim types.



    static void Main(string[] args)
            {
                Claim claim = new Claim("Name", "Andras");
                Claim newClaim = new Claim(ClaimTypes.Country, "Sweden");
            }


You can check out the values available in ClaimTypes using IntelliSense. It is safer to use the enumeration where-ever possible rather than the straight string values as chances are the system you're trying to communicate Claims with will also understand the values behind the enumerated one. If you don't find any suitable one there you can revert back to string-based Types and even define a formal namespace to the claim as follows:



    new Claim("http://www.mycompany.com/building/floor", "Two")


Place the cursor on "Country" in 'new Claim(ClaimTypes.Country, "Sweden");' and press F12. You'll see a long list of namespaces, such as the following:



    public const string Country = "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/country";


These namespaces will uniquely describe the claim type.

It is very rare that there's only one claim about a person. Instead, claims come in collections, so let's create one:



    static void Main(string[] args)
            {
                IList<Claim> claimCollection = new List<Claim>
                {
                    new Claim(ClaimTypes.Name, "Andras")
                    , new Claim(ClaimTypes.Country, "Sweden")
                    , new Claim(ClaimTypes.Gender, "M")
                    , new Claim(ClaimTypes.Surname, "Nemes")
                    , new Claim(ClaimTypes.Email, "hello@me.com")
                    , new Claim(ClaimTypes.Role, "IT")
                };
            }


We can easily build an object of type IIDentity out of a collection of claims, in this case a ClaimsIdentity as mentioned above:



    ClaimsIdentity claimsIdentity = new ClaimsIdentity(claimCollection);


We now test if this identity is authenticated or not:



    Console.WriteLine(claimsIdentity.IsAuthenticated);


Run the Console app and test for yourself; IsAuthenticated will return false. Claims can be attached to anonymous users which is different from the standard principal types in .NET4 and earlier. It used to be enough for the user to have a name for them to be authenticated. With Claims that is not enough. Why would you attach claims to an unauthenticated user? It is possible that you gather information about a user on your site which will be used to save their preferences when they are turned into members of your site.

If you want to turn this IIdentity into an authenticated one you need to provide the authentication type which is a string descriptor:



    ClaimsIdentity claimsIdentity = new ClaimsIdentity(claimCollection, "My e-commerce website");


Run the application again and you'll see that the user is now authenticated.

Using our identity object we can also create an IPrincipal:



    ClaimsPrincipal principal = new ClaimsPrincipal(claimsIdentity);


As ClaimsPrincipal implements IPrincipal we can assign the ClaimsPrincipal to the current Thread as the current principal:



    Thread.CurrentPrincipal = principal;


From this point on the principal is available on the current thread in the rest of the application.

Create a new method called Setup() and copy over all our code as follows:



    static void Main(string[] args)
            {
                Setup();

                Console.ReadLine();
            }

            private static void Setup()
            {
                IList<Claim> claimCollection = new List<Claim>
                {
                    new Claim(ClaimTypes.Name, "Andras")
                    , new Claim(ClaimTypes.Country, "Sweden")
                    , new Claim(ClaimTypes.Gender, "M")
                    , new Claim(ClaimTypes.Surname, "Nemes")
                    , new Claim(ClaimTypes.Email, "hello@me.com")
                    , new Claim(ClaimTypes.Role, "IT")
                };

                ClaimsIdentity claimsIdentity = new ClaimsIdentity(claimCollection, "My e-commerce website");

                Console.WriteLine(claimsIdentity.IsAuthenticated);

                ClaimsPrincipal principal = new ClaimsPrincipal(claimsIdentity);
                Thread.CurrentPrincipal = principal;
            }


Let's see if the new Claims-related objects can be used in .NET4 where Claims are not available. Create a new method as follows:



    private static void CheckCompatibility()
            {

            }


Call this method from Main as follows:



    static void Main(string[] args)
            {
                Setup();
                CheckCompatibility();

                Console.ReadLine();
            }


Add the following to the CheckCompatibility method:



    IPrincipal currentPrincipal = Thread.CurrentPrincipal;
    Console.WriteLine(currentPrincipal.Identity.Name);


Run the application and the name you entered in the claims collection will be shown on the output window. The call for the Name property will look through the claims in the ClaimsIdentity object and extracts the value of Name claim type.

In some cases it is not straightforward what a 'Name' type means. Is it the unique identifier of the user? Is it the email address? Or is it the display name? You can easily specify how a claim type is defined. You can pass two extra values to the ClaimsIdentity constructor: which claim type constitutes the 'Name' and the 'Role' claim types. So if you want the email address to be the 'Name' claim type you would do as follows:



    ClaimsIdentity claimsIdentity = new ClaimsIdentity(claimCollection, "My e-commerce website", ClaimTypes.Email, ClaimTypes.Role);


Run the application and you will see that the email is returned as the name from the claim collection.

You can check if a user is in a specific role as before:



    private static void CheckCompatibility()
            {
                IPrincipal currentPrincipal = Thread.CurrentPrincipal;
                Console.WriteLine(currentPrincipal.Identity.Name);
                Console.WriteLine(currentPrincipal.IsInRole("IT"));
            }


…which will yield true.

So we now know that 'old' authentication code will still be able to handle the Claims implementation. We'll now turn to 'new' code. Create a new method called CheckNewClaimsUsage() and call it from Main just after CheckCompatibility().
The following will retrieve the current claims principal from the current thread:



    private static void CheckNewClaimsUsage()
            {
                ClaimsPrincipal currentClaimsPrincipal = Thread.CurrentPrincipal as ClaimsPrincipal;
            }


Now you'll have access to all the extra methods and properties built into ClaimsPrincipal. You can enumerate through the claims collection in the principal object, find a specific claim using Linq etc. Example:



    private static void CheckNewClaimsUsage()
            {
                ClaimsPrincipal currentClaimsPrincipal = Thread.CurrentPrincipal as ClaimsPrincipal;
                Claim nameClaim = currentClaimsPrincipal.FindFirst(ClaimTypes.Name);
                Console.WriteLine(nameClaim.Value);
            }


You can use .HasClaim to check if the claims collection includes a specific claim you're looking for.

You can also query the identities of the ClaimsPrincipal:



    foreach (ClaimsIdentity ci in currentClaimsPrincipal.Identities)
                {
                    Console.WriteLine(ci.Name);
                }


The line…



    ClaimsPrincipal currentClaimsPrincipal = Thread.CurrentPrincipal as ClaimsPrincipal;


…becomes so common that there's a built-in way to achieve the same thing:



    ClaimsPrincipal currentClaimsPrincipal = ClaimsPrincipal.Current;


This will throw an exception if the concrete IPrincipal type is not ClaimsPrincipal for whatever reason.

We have looked at the very basics of Claims in .NET4.5. The next post will look at the new IIdentity and IPrincipal inheritance model.

You can view the list of posts on Security and Cryptography [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
