[Source](http://dotnetcodr.com/2014/07/03/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-4/ "Permalink to Introduction to forms based authentication in ASP.NET MVC5 Part 4")

# Introduction to forms based authentication in ASP.NET MVC5 Part 4

**Introduction**

In the [previous part][1] of this series we looked at some key objects around the new Identity packages in .NET. We also identified the EntityFramework database context object which can be extended with your own entities. By default it only contains User-related entities. Then we tried to extend the default implementation of IUser, i.e. ApplicationUser which derives from the EF entity IdentityUser. We failed at first as there's no mechanism that will update our database on the fly if we make a change to the underlying User model.

We'll continue working on the same demo project as before so have it ready in Visual Studio 2013.

**Database migrations and seeding**

As hinted at in the previous post we'll look at EntityFramework 6 and DB migration in the series after this one but we'll need to solve the current issue somehow.

Open the Package Manager Console in Visual Studio: View, Other Windows, Package Manager Console. Run the following command:

Enable-Migrations

This is an EntityFramework specific command. After some console output you should see the following success message:

Code First Migrations enabled for project [your project name]

Also, a couple of new items will be created for you in VS. You'll see a migration script in a new folder called Migrations. The migration script will have a name with a date stamp followed by _InitialCreate.cs. Also in the same folder you'll see a file called Configuration.cs. An interesting property is called AutomaticMigrationsEnabled. It is set to false by default. If it's set to true then whenever you change the structure of your entities then EF will reflect those in the database automatically. In an alpha testing environment this is probably a good idea but for the production environment you'll want to set it to false. Set it to true for this exercise.

Another interesting element in Configuration.cs is the Seed method. The code is commented out but the purpose of the method is to make sure that there's some data in the database when the database is updated. E.g. if want to run integration tests with real data in the database then you can use this method to populate the DB tables with some real data.

There are at least two strategies you can follow to populate the User database within the Seed method. The traditional EF context approach looks like this:



    PasswordHasher passwordHasher = new PasswordHasher();
    context.Users.AddOrUpdate(user => user.UserName
    	, new ApplicationUser() { UserName = "andras", PasswordHash = passwordHasher.HashPassword("hello") });
    context.SaveChanges();


We use PasswordHasher class built into the identity library to hash a password. The AddOrUpdate method takes a property selector where we define which property should be used for equality. If there's already a user with the username "andras" then do an update. Otherwise insert the new user.

Another approach is to use the UserManager object in the Identity.EntityFramework assembly. Insert the following code into the Seed method:



    if (!context.Users.Any(user => user.UserName == "andras"))
    {
    	UserStore<ApplicationUser> userStore = new UserStore<ApplicationUser>(context);
    	UserManager<ApplicationUser> userManager = new UserManager<ApplicationUser>(userStore);
    	ApplicationUser applicationUser = new ApplicationUser() { UserName = "andras" };
    	userManager.Create<ApplicationUser>(applicationUser, "password");
    }


We first check for the presence of any user with the username "andras" with the help of the Any extension method. If there's none then we build up the UserManager object in a way that's now familiar from the AccountController constructor. We then call the Create method of the UserManager and let it take care of the user creation behind the scenes.

Go back to the Package Manager Console and issue the following command:

Update-Database

The console output should say – among other things – the following:

Running Seed method.

Open the database file in the App_Data folder and then check the contents of the AspNetUsers table. The new user should be available and the table also includes the FavouriteProgrammingLanguage column:

![User data migration success][2]

Run the application and try to log in with the user you've just created in the Seed method. It should go fine. Then log off and register a new user and provide some programming language in the appropriate field. Then refresh the database in the Server Explorer and check the contents of AspNetUsers. You'll see the new user with there with their favourite language.

If you'd like to create roles and add users to roles in the seed method then it's a similar process:



    string roleName = "IT";
    RoleStore<IdentityRole> roleStore = new RoleStore<IdentityRole>(context);
    RoleManager<IdentityRole> roleManager = new RoleManager<IdentityRole>(roleStore);
    roleManager.Create(new IdentityRole() { Name = roleName });
    userManager.AddToRole(applicationUser.Id, roleName);


**Third party identity providers**

We looked at Start.Auth.cs briefly in a previous part of this series. By default most of the code is commented out in that file and only the traditional login form is activated. However, if you look at the inactive code bits then you'll see that you can quite quickly enable Microsoft, Twitter, Facebook and Google authentication.

You can take advantage of these external providers so that you don't have to take care of storing your users' passwords and all actions that come with it such as updating passwords. Instead, a proven and reliable service will tell your application that the person trying to log in is indeed an authenticated one. Also, your users won't have to remember another set of username of password.

In order to use these platforms you'll need to register your application with them with one exception: Google. E.g. for Facebook you'll need to go to developers.facebook.com, sign in and register an application with a return URL. In return you'll get an application ID and a secret:

![Facebook application ID and secret][3]

The Azure mobile URL is the callback URL for an application I registered before on each of those 4 providers. I hid the application ID and application secret values.

For Twitter you'll need to go to dev.twitter.com and register your application in a similar fashion. You'll get an API key and an API secret:

![Twitter application keys][4]

These providers will use [OAuth2][5] and [OpenId Connect][6] to perform the login but the complex details are hidden behind the [Katana extensions][7], like app.UseMicrosoftAccountAuthentication.

As you activate the external providers there will be new buttons on the Log in page for them:

![External providers login buttons][8]

Upon successful login your web app will receive an authentication token from the provider and that token will be used in all subsequent communication with your website. The token will tell your website that the user has authenticated herself along with details such as the expiry date of the token and maybe some user details depending on what your web app has requested.

As mentioned before there's one exception to the client ID / client secret data requirement: Google. So comment out…



    app.UseGoogleAuthentication();


…and run the application. Navigate to the Log in page and press the Google button. You'll be directed to the standard Google login page. If you're already logged on with Google then this step is skipped. Next you'll be shown a consent page where you can read what data "localhost" will be able to read from you:

![Google consent screen][9]

If you implement the other providers at a later point then they'll follow the same process, i.e. show the consent screen. Normally you can configure the kind of data your application will require from the user on the individual developer websites mentioned above.

Click Accept and then you can complete the registration in the last step:

![Confirm registration with Google][10]

This is where you create a local account which will be stored in the AspNetUserLogins table. Press Register and there you are, you've registered using Google.

Check the contents of AspNetUserLogins:

![Google login data in DB][11]

Also, the user is stored in the AspNetUsers table:

![Google user][12]

You'll see that the password is not stored anywhere which is expected. The credentials are stored in the provider's database.

Read the last part in this series [here][13].

You can view the list of posts on Security and Cryptography [here][14].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/06/30/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-3/ "Introduction to forms based authentication in ASP.NET MVC5 Part 3"
[2]: http://dotnetcodr.files.wordpress.com/2014/05/user-data-migration-success.png?w=630&h=68
[3]: http://dotnetcodr.files.wordpress.com/2014/05/facebook-application-id-and-secret.png?w=630&h=311
[4]: http://dotnetcodr.files.wordpress.com/2014/05/twitter-application-keys.png?w=630
[5]: http://dotnetcodr.com/2014/01/20/introduction-to-oauth2-json-web-tokens/ "Introduction to OAuth2: Json Web Tokens"
[6]: http://dotnetcodr.com/2014/02/06/introduction-to-oauth2-part-6-openid-connect-basics/ "Introduction to OAuth2 part 6: OpenID Connect basics"
[7]: http://dotnetcodr.com/2014/04/14/owin-and-katana-part-1-the-basics/ "OWIN and Katana part 1: the basics"
[8]: http://dotnetcodr.files.wordpress.com/2014/05/external-providers-login-buttons.png?w=630
[9]: http://dotnetcodr.files.wordpress.com/2014/05/google-consent-screen.png?w=630
[10]: http://dotnetcodr.files.wordpress.com/2014/05/confirm-registration-with-google.png?w=630&h=196
[11]: http://dotnetcodr.files.wordpress.com/2014/05/google-login-data-in-db.png?w=630&h=65
[12]: http://dotnetcodr.files.wordpress.com/2014/05/google-user.png?w=630
[13]: http://dotnetcodr.com/2014/07/07/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-5-claims/ "Introduction to forms based authentication in ASP.NET MVC5 Part 5: Claims"
[14]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
