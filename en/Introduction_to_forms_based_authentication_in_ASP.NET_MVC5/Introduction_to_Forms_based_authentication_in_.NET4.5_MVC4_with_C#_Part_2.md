[Source](http://dotnetcodr.com/2013/01/24/introduction-to-forms-based-authentication-in-net4-5-mvc4-with-c-part-2/ "Permalink to Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 2")

# Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 2

This post will build upon the basics of Forms Based Authentication in an MVC4 we discussed in the previous blog post.

**What happen upon registration?**

The layout of the registration page is provided by Register.cshtml in the Views/Account folder. Look at the first row in that file to locate the Model the View is based on:



    @model SecurityBasics.Models.RegisterModel


Note that the view is not based directly on our UserProfile class but on a special "intermediate" object that we can call a ViewModel. This is a common technique to avoid Mass Assignment attacks where the attacker deliberately tries to pass in values for properties that are not meant to be populated by an external call. Suppose that our UserProfile had a property called AccountBalance. The attacker could then add 'accountbalance=1000′ to the collection of form values in an HTTP POST request to populate that value and save it in the database. To avoid the possibility of such an action we construct a ViewModel that represents all the properties that we want our users to populate.

Another purpose of ViewModels is to keep our business logic and view logic separate. Note that SecurityBasics.Models.RegisterModel has a property called ConfirmPassword. If we based the Register view directly on our UserProfile class then we'd have to include a property called ConfirmPassword. Is that really part of our user business logic? Its only purpose is to check that the new user has typed the same password. In addition, the corresponding database table will also get a new column called RegisterPassword when we migrate the structure described in SecurityBasicsContext.cs. Also, check the structure of the RegisterModel class: it's filled with view-related attributes, such as "Display" or "StringLength". These attributes are important for data validation that do not belong in a true domain object which should be as clean of such notations as possible.

So, when the user presses the 'Register' button then the RegisterModel is populated with the values entered in the text boxes on that page:



    <ol>
                <li>
                    @Html.LabelFor(m => m.UserName)
                    @Html.TextBoxFor(m => m.UserName)
                </li>
                <li>
                    @Html.LabelFor(m => m.Password)
                    @Html.PasswordFor(m => m.Password)
                </li>
                <li>
                    @Html.LabelFor(m => m.ConfirmPassword)
                    @Html.PasswordFor(m => m.ConfirmPassword)
                </li>
            </ol>


The next step in the chain is the AccountController, more specifically the Register method:



    [HttpPost]
    [AllowAnonymous]
    [ValidateAntiForgeryToken]
    public ActionResult Register(RegisterModel model)


Within this action method you'll find the following lines of code:



    WebSecurity.CreateUserAndAccount(model.UserName, model.Password);
    WebSecurity.Login(model.UserName, model.Password);
    return RedirectToAction("Index", "Home");


WebSecurity is a wrapper class around Membership functionality. It takes care of things like access and cryptography. The CreateUserAndAccount call does exactly what it says it will do: create the user in the database that we created in the previous post. WebSecurity.Login will try to let the user log on immediately after a successful registration. You can see what this action leads to in the AccountController.cs file in the below method:



            [HttpPost]
            [AllowAnonymous]
            [ValidateAntiForgeryToken]
            public ActionResult Login(LoginModel model, string returnUrl)
            {
                if (ModelState.IsValid && WebSecurity.Login(model.UserName, model.Password, persistCookie: model.RememberMe))
                {
                    return RedirectToLocal(returnUrl);
                }

                // If we got this far, something failed, redisplay form
                ModelState.AddModelError("", "The user name or password provided is incorrect.");
                return View(model);
            }


WebSecurity will compare the hashed version of the password entered on the login page and the hashed password in the database for the user with the provided username. If the two values match then an authentication cookie is issued that the website can use on subsequent requests to authenticate the user.

**How to protect a certain page**

Of course the whole purpose of user registration and logging in is to protect specific parts of your website. You may typically have pages that are only meant for authenticated users. The easiest way to demand authentication in MVC4 is through the Authorize attribute.

Let's say that we want to restrict access to our About page. Locate the About action in the HomeController. As in MVC we don't have 'pages' in the same sense as we have .aspx files in a traditional ASP.NET web app, we restrict access to actions instead. You can decorate the About action as follows:



    [Authorize]
    public ActionResult About()


This attribute will automatically intercept the incoming HTTP request to inspect if an authentication cookie is present and what it contains. If the current user is anonymous then they will be redirected to the Login page. Run the web application and click the About link. You will be presented with the Login page as you are an anonymous user. Before you log on check the URL in the browser. It should look something like this:

/Account/Login?ReturnUrl=%2fHome%2fAbout

The ReturnUrl stores the information about which Controller and which Action the user wanted to access, i.e. the About action within the Home controller. This value will be used by the Login action shown above as the returnUrl parameter. This provides convenience for the user. Upon successful login they will be redirected to the About page. Otherwise the user would have to locate the page that they tried to access before logging in.

Now try to log in. You should be redirected to the About page.

**How to add custom properties to the users upon registration**

In this section we will revisit some of the classes presented above in this blog post.

Suppose you'd like to record the user's favourite programming language when they register with your site. Open UserProfile.cs and extend the code as follows:



    [Table("UserProfile")]
        public class UserProfile
        {
            [Key]
            [DatabaseGeneratedAttribute(DatabaseGeneratedOption.Identity)]
            public int UserId { get; set; }
            public string UserName { get; set; }
            public string FavouriteLanguage { get; set; }
        }


As you recall from the previous post this class is used in our database context class SecurityBasicsContext.cs. The new property will be translated into a new column in the UserProfile database table upon schema migration. Let's update the database by typing the following command in the Package Manager Console: PM> Update-Database. You should see an output similar to this:

Applying automatic migration: 201301141950249_AutomaticMigration.
Running Seed method.

Open Server Explorer from the View menu and locate the DefaultConnection node. Open the UserProfile table within the Tables folder. You'll see the new column called FavouriteLanguage.

The next step is to add some value to that column when a new user registers with the site. Recall that the Register action in the AccountController.cs file is not based on our UserProfile object but on a ViewModel called RegisterModel. Locate that object in the AccountModels.cs file. Let's include our new property which we will make compulsory with a max length of 10 characters. Add the following property below 'ConfirmPassword':



    [Display(Name = "Favourite programming language")]
    [Required]
    [StringLength(10)]
    public string FavouriteLanguage { get; set; }


We will want to show a textbox on Register.cshtml in order to populate this property. Open that file and add another text box to the ordered list as follows:



    <li>
       @Html.LabelFor(m => m.FavouriteLanguage)
       @Html.TextBoxFor(m => m.FavouriteLanguage)
    </li>


The last step is to modify the Register action in the Account controller to save this property along with the user name and password. Remember the following bit of code?



     WebSecurity.CreateUserAndAccount(model.UserName, model.Password);


This method has an overloaded version that accepts an array of anonymous properties which you can populate with your custom values. So to include the favourite language we'll do as follows:



    WebSecurity.CreateUserAndAccount(model.UserName, model.Password, new { FavouriteLanguage = model.FavouriteLanguage });


We're done. Run the application and register a new user. You should see the textbox to enter your favourite language. Press register and go to DefaultConnection in the Server Explorer. Check the contents of the UserProfile table. It may look something like this:

![Registration with a custom property][1]

**Where is the security cookie located?**

Log out from the application and press F12 in IE to open the developer tools. Click on the 'About' page to get authenticated. Press 'Start capturing' in the developer tools window:

![Start capturing web traffic web developer tool][2]

Fill out your login data, select the Remember me check box and press Log in. You will see an output similar to this in the developer tools window:

![Web traffic upon login][3]

Look at the first entry: you posted your username and password to Account/Login. The original page you wanted to reach is saved in the returnUrl query string as discussed before. Upon successful login you are redirected to Home/About, check the 302 Redirect as the result of the first POST.

Double-click on the first entry in this list to check the response headers:

![Set ASPNET auth header][4]

Check the set-cookie header. This is how we tell the browser to accept a cookie. You'll see a cookie named .ASPXAUTH. The decrypted value of the cookie will tell ASP.NET that the user was successfully authenticated and there's no need to request authentication again before the user logs off. The browser will send this cookie along with every subsequent request. Click the 'Back to summary view' button in the developer tools and double-click on the second item in the list: localhost:xxxx/Home/About. Select the Request header to inspect what the browser sent to the server along with the request. In the 'Key' column locate the Cookie section:

![Cookie section in Http request header][5]

Double-click that value to inspect all cookies. In the bottom of the list you should see our .ASPXAUTH cookie:

![AspxAuth cookie in http request header][6]

This cookie value will be included in every single subsequent Http request at least as long as the session lasts. Possibly longer if you select the Remember me check box upon logging in.

The last thing I want to show you calls for the need to tighten the security around your website. Again, click 'Back to summary view', double-click the first entry in the list and select the Request body tab:

![Request body with username and password][7]

Inspect the value of the RequestVerificationToken. Scroll all the way to the right and you'll see the following:

__RequestVerificationToken=…6QyPsNsLRPbwvuRvxq9lMZ-Bk1&UserName=andras&Password=[passwordincleartext]&RememberMe=true&RememberMe=false

That's right, the username and password are visible in clear text. I obviously won't show my password here, but it is definitely shown in clear text, you can check this on your own screen. These values can be read by any hacker sniffing the web traffic between you and the server. This is of course not secure. So if your website is going to handle some data that needs some serious protection then SSL should be used to tighten security. The login process will be forced to go through HTTPS. The easiest way to enforce HTTPS on an Action is to add the [RequireHttps] attribute to it as follows:



            [HttpPost]
            [AllowAnonymous]
            [ValidateAntiForgeryToken]
            [RequireHttps]
            public ActionResult Register(RegisterModel model)


Of course you'll need to install a certificate on the web server to make this work but that is beyond the scope of this blog post.

You can view the list of posts on Security and Cryptography [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/01/registrationwithcustomproperty.png?w=300&h=90
[2]: http://dotnetcodr.files.wordpress.com/2013/01/developertoolsstartcapturing.png?w=300&h=101
[3]: http://dotnetcodr.files.wordpress.com/2013/01/startcapturinglogin.png?w=300&h=163
[4]: http://dotnetcodr.files.wordpress.com/2013/01/setauthheaderuponauthentication.png?w=300&h=126
[5]: http://dotnetcodr.files.wordpress.com/2013/01/cookiesectioninhttprequestheader.png?w=300&h=123
[6]: http://dotnetcodr.files.wordpress.com/2013/01/aspxauthcookieinrequest.png?w=300&h=61
[7]: http://dotnetcodr.files.wordpress.com/2013/01/requestbodyofhttpcall.png?w=300&h=53
[8]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
