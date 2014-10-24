[Source](http://dotnetcodr.com/2014/06/26/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-2/ "Permalink to Introduction to forms based authentication in ASP.NET MVC5 Part 2")

# Introduction to forms based authentication in ASP.NET MVC5 Part 2

**Introduction**

In the [previous][1] part of this series we looked at the absolute basics of Forms Based Authentication in MVC5. Most of what we've seen is familiar from MVC4.

It's time to dive into what's behind the scenes so that we gain a more in-depth understanding of the topic.

We'll start with the database part: where are users stored by default and in what form? We created a user in the previous post so let's see where it had ended up.

**Demo**

Open the project we started building previously.

By default if you have nothing else specified then MVC will create a database for you in the project when you created your user – we'll see how in the next series devoted to EntityFramework. The database is not visible at first within the project. Click on the Show All Files icon in the solution explorer…:

![Show all files icon in solution explorer][2]

…and you'll see an .mdf file appear in the App_Data folder:

![Database file in App_Data folder][3]

The exact name will differ in your case of course. Double-click that file. The contents will open in the Server Explorer:

![Membership tables in MVC 5][4]

Some of these tables might look at least vaguely familiar from the default ASP.NET Membership tables in MVC4. However, you'll see that there are a lot fewer tables now so that data is stored in a more compact format. A short summary of each table – we'll look at some of them in more detail later:

* _MigrationHistory: used by EntityFramework when migrating users – migrations to be discussed in the next series
* AspNetRoles: where the roles are stored. We have no roles defined yet so it's empty
* AspNetUserClaims: where the user's claims are stored with claim type and claim value. New to claims? Start [here][5].
* AspNetUserLogins: used by external authentication providers, such as Twitter or Google
* AspNetUserRoles: the many-to-many mapping table to connect users and roles
* AspNetUsers: this is where all site users are stored with their usernames and hashed passwords

As you can see the membership tables have been streamlined a lot compared to what they looked like in previous versions. They have a lot fewer columns and as we'll see later they are very much customisable with EntityFramework.

Right-click the AspNetUsers table and select Show Table Data. You'll see the user you created before along with the hashed password.

**Database details**

Go back to the Solution explorer and open up web.config. Locate the connectionStrings section. That's where the default database connection string is stored with that incredibly sexy and easy-to-remember name. So the identity components of MVC5 will use DefaultConnection to begin with. We can see from the connection string that a local DB will be used with no extra login and password.

You can in fact change the connection string to match the real requirements of your app of course. The SQL file name is defined by the AttachDbFilename parameter. The Initial Catalog parameter denotes the database name as it appears in the SQL management studio. Change both to AwesomeDatabase:

AttachDbFilename=|DataDirectory|AwesomeDatabase.mdf;Initial Catalog=AwesomeDatabase;

Run the application. Now two things can happen:

* If you continued straight from the previous post of the series then you may still be logged on – you might wonder if the app is still using the old database, but it's not the case. The application has picked up the auth cookie available in the HTTP request. In this case press Log Off to remove that cookie.
* Otherwise you're not logged in and you'll see the Register and Log in links

Try to log in, it should fail. This is because the new database doesn't have any users in it yet and the existing users in the old database haven't been magically transported. Stop the application, press Refresh in Solution Explorer and you'll see the new database file:

![New database created automatically][6]

**Communication with the database**

We've seen where the database is created and how to control the connection string. Let's go a layer up and see which components communicate with the database. Identity data is primarily managed by the Microsoft.AspNet.Identity.Core NuGet library. It is referenced by the any MVC5 app where you run forms based authentication. It contains – among others – abstractions for the following identity-related elements:

* IUser with ID and username
* IRole with ID and role name
* IUserStore: interface to abstract away various basic actions around a user, such as creating, deleting, finding and updating a user
* IUserPasswordStore which implements IUserStore: interface to abstract away password related actions such as getting and setting the hashed password of the user and determining – on top all functions of an IUserStore

There's also a corresponding abstraction for storing the user roles and claims and they all derive from IUserStore. IUserStore is the mother interface for a lot of elements around user management in the new Identity library.

There are some concrete classes in the Identity.Core library, such as UserManager and RoleManager. You can see the UserManager in action in various methods of AccountController.cs:



    var result = await UserManager.CreateAsync(user, model.Password);
    IdentityResult result = await UserManager.ChangePasswordAsync(User.Identity.GetUserId(), model.OldPassword, model.NewPassword);
    IdentityResult result = await UserManager.AddPasswordAsync(User.Identity.GetUserId(), model.NewPassword);


The default UserManager is set in the constructor of the AccountController object:



    public AccountController()
               : this(new UserManager<ApplicationUser>(new UserStore<ApplicationUser>(new ApplicationDbContext())))
    {
    }

    public AccountController(UserManager<ApplicationUser> userManager)
    {
         UserManager = userManager;
    }

    public UserManager<ApplicationUser> UserManager { get; private set; }


You see that we supply a UserStore object which implements a whole range of interfaces:

* IUserLoginStore
* IUserClaimStore
* IUserRoleStore
* IUserPasswordStore
* IUserSecurityStampStore
* IUserStore

So the default built-in UserManager object will be able to handle a lot of aspects around user management: passwords, claims, logins etc. As a starting point the UserManager will provide all domain logic around user management, such as validation, password hashing etc.

In case you want to have your custom solution to any of these components then define your solution so that it implements the appropriate interface and then you can plug it into the UserManager class. E.g. if you want to store your users in MongoDb then implement IUserStore, define your logic there and pass it in as the IUserStore parameter to the UserManager object. It's a good idea to implement as many sub-interfaces such as IUserClaimsStore and IUserRoleStore as possible so that your custom UserStore that you pass into UserManager will be very "clever": it will be able to handle a lot of aspects around user management. And then when you call upon e.g. UserManager.CreateAsync then UserManager will pick up your custom solution to create a user.

However, if you're happy with an SQL server solution governed by EntityFramework then you may consider the default setup and implementations inserted by the MVC5 template. We'll investigate those in the [next post][7].

You can view the list of posts on Security and Cryptography [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/06/23/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-1/ "Introduction to forms based authentication in ASP.NET MVC5 Part 1"
[2]: http://dotnetcodr.files.wordpress.com/2014/05/show-all-files-icon-in-solution-explorer.png?w=630
[3]: http://dotnetcodr.files.wordpress.com/2014/05/database-file-in-app_data-folder.png?w=630
[4]: http://dotnetcodr.files.wordpress.com/2014/05/membership-tables-in-mvc-5.png?w=630
[5]: http://dotnetcodr.com/2013/02/11/introduction-to-claims-based-security-in-net4-5-with-c-part-1/ "Introduction to Claims based security in .NET4.5 with C# Part 1: the absolute basics"
[6]: http://dotnetcodr.files.wordpress.com/2014/05/new-database-created-automatically.png?w=630
[7]: http://dotnetcodr.com/2014/06/30/introduction-to-forms-based-authentication-in-asp-net-mvc5-part-3/ "Introduction to forms based authentication in ASP.NET MVC5 Part 3"
[8]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
