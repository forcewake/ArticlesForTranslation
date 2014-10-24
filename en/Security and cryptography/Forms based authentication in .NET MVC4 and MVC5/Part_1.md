[Source](http://dotnetcodr.com/2014/06/23/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-1/ "Permalink to Introduction to forms based authentication in ASP.NET MVC5 Part 1")

# Introduction to forms based authentication in ASP.NET MVC5 Part 1

**Introduction**

ASP.NET MVC5 comes with a number of new elements regarding user management and security. When you create a new MVC 5 web app you'll be able to choose between 4 default authentication types:

* No authentication, i.e. anonymous users can access your site
* Individual user accounts: the traditional way to log onto the site using a login form. The user store is backed by SQL identity tables. You can also enable some well known auth providers: Twitter, Google, Facebook and Microsoft where you don't need to worry about the password
* Organisational accounts: Active Directory Federation Services (ADFS), used within organisations that manage their users in ADFS, typically coupled with Single SignOn for the applications within the organisation. You can enable Azure-based ADFS as well
* Windows auth: Windows Active Directory authentication for intranet apps. Your Windows login credentials will be used to access the internal applications of your company. This is somewhat like a stripped down version of the organisational accounts option

In this blog series we'll look at the new identity features of MVC5.

**Forms based authentication**

Fire up Visual Studio 2013 and select the ASP.NET Web Application template in the New Project window. Give the project some name and click OK. A new window will open where you can select additional templates. Pick MVC. Press the 'Change authentication' button and make sure that 'Individual user accounts' is selected:

![Individual user accounts option][1]

Visual Studio will set up a forms-enabled MVC app for you without any extra effort. Run the web app and you'll be directed to the default home page. You'll see that it comes with the Register and Log in links:

![Register and log in links in MVC 5 web app][2]

If you used [Forms based auth in MVC 4][3] then this is of course no surprise to you. Click Register and create a user. If everything goes well you'll be automatically logged in and you'll see your username instead of "Register":

![User registered and logged in][4]

You can click on "Hello (your name)" to manage save your password if you want. Click Log off and then log in again with your credentials to check if it works fine. It should.

**The Layout view**

The top menu of the MVC 5 template is controlled by _Layout.cshtml in the Views/Shared folder. Open that file. You'll see the links for Home, About and Contact. Below those links you'll have a partial view called _LoginPartial. _LoginPartial is located in the same folder. Open it and let's see what it contains.

There's an if-else statement which tweaks what the user sees based on the Request.IsAuthenticated property. This property is set by the framework depending on whether the current user has logged on or not.

The user name is extracted using the User.Identity.GetUserName() method. User is an IPrincipal object and represents the currently logged-on user. Identity is the IIdentity belonging to the user which contains a small set of information about the user such as the user name or the authentication type. User.Identity is also set by the framework just like with the IsAuthenticated property.

You can read the User object anywhere within the controllers and views, i.e. where a valid HTTP session is available. Open Controllers/HomeController.cs and add the following code to Index() just above the return statement:



    string userName = User.Identity.Name;


Set a breakpoint within Index and run the application. You'll see that the username can be easily extracted this way.

**Restrict access**

There's little point in authenticating users if you don't limit the access to certain parts of your website to authenticated users only.

Right-click the Controllers folder and select Add, Controller. In the Add Scaffold window select the topmost option, i.e. Empty. Call it CustomersController. Add the following two methods to the controller:



    public string SecretName()
    {
    	return "This is the secret customer name.";
    }

    public string PublicName()
    {
    	return "This is the public customer name.";
    }


The goal is to protect access to the SecretName action and only let anonymous users read the public name.

Run the application and log off if you're still logged in. Then navigate to the above actions:

* localhost:xxxx/Customers/publicname
* localhost:xxxx/Customers/secretname

Not surprisingly you can access both methods without having to log in first.

If your only requirement is to restrict access to a controller action to authenticated users only then you can use the Authorize attribute like this:



    [Authorize]
    public string SecretName()
    {
    	return "This is the secret customer name.";
    }


Re-run the app and navigate to the secretname action. You should see that you're redirected to the login page. Log in and you'll see the secret. Note the URL of the Login page: Account/Login. It is defined in a Katana component in Startup.Auth.cs in the App_Start folder:



    app.UseCookieAuthentication(new CookieAuthenticationOptions
    {
              AuthenticationType = DefaultAuthenticationTypes.ApplicationCookie,
              LoginPath = new PathString("/Account/Login")
    });


If you don't know what OWIN and Katana mean then you can start [here][5].

The ReturnUrl query string in the URL will store which controller and action you've tried to access. It will be fed to the POST Login action of AccountController:



    public async Task<ActionResult> Login(LoginViewModel model, string returnUrl)


Now log off and navigate to /Customers/publicname. It should still be available to anonymous users.

What if you want to restrict access to all actions within the controller? You can apply the Authorize attribute on the controller level:



    [Authorize]
    public class CustomersController : Controller


Run the app again and navigate to /Customers/publicname. It is now also a restricted site so you'll be redirected to the login page.

You can override the controller level Authorize attribute by decorated the individual action(s) with the AllowAnonymous attribute:



    [AllowAnonymous]
    public string PublicName()
    {
    	return "This is the public customer name.";
    }


Run the app and verify that /Customers/publicname is publicly available again.

The Authorize attribute accepts a couple of parameters to further refine the access filter. Examples:



    [Authorize(Users="andras,admin")]
    [Authorize(Roles="admin,poweruser")]


You can probably guess what they mean: allow users with specific user names – andras and admin – or only allow users who have either admin and power user role to access an action.

You can test this as follows. Add the following attribute to SecretName:



    [Authorize(Users = "elvis,bob")]
    public string SecretName()
    {
    	return "This is the secret customer name.";
    }


Run the app and navigate to the secret name action. Log in with your user. You should see that you're immediately redirected back to the Login page – unless you selected 'elvis' or 'bob' as the user name in the sign up process. In that case the great secret will be revealed to you.

We've now seen the basics of forms based authentication in MVC5. We'll dig much deeper in the coming blog posts.

Read the next post in this series [here][6].

You can view the list of posts on Security and Cryptography [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2014/05/individual-user-accounts-option.png?w=630&h=439
[2]: http://dotnetcodr.files.wordpress.com/2014/05/register-and-log-in-links-in-mvc-5-web-app.png?w=630&h=132
[3]: http://dotnetcodr.com/2013/01/21/introduction-to-forms-based-authentication-in-net4-5-mvc4-with-c-part-1/ "Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 1"
[4]: http://dotnetcodr.files.wordpress.com/2014/05/user-registered-and-logged-in.png?w=630
[5]: http://dotnetcodr.com/2014/04/14/owin-and-katana-part-1-the-basics/ "OWIN and Katana part 1: the basics"
[6]: http://dotnetcodr.com/2014/06/26/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-2/ "Introduction to forms based authentication in ASP.NET MVC5 Part 2"
[7]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
