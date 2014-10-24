[Source](http://dotnetcodr.com/2013/01/21/introduction-to-forms-based-authentication-in-net4-5-mvc4-with-c-part-1/ "Permalink to Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 1")

# Introduction to Forms based authentication in .NET4.5 MVC4 with C# Part 1

This blog post will give an introduction to Forms based authentication in an MVC4 internet application using .NET4.5 and C#.

**Introduction**

You can choose from 3 types of authentication when you create an MVC4 application: Windows, Forms and OAuth.

**Windows authentication** is best applied to intranet applications where all your users are registered in Active Directory and work within the boundaries of the company firewall. When your intranet application requires authentication it can use the claims available in the Active Directory and perform the login automatically. There's no need for the user to sign in separately to the application after logging on to their computer. Theoretically it's possible to have Windows auth on an internet site but the limitations are clear: it's not guaranteed that your users will be registered on your domain and that they use a device that supports Windows auth.

**OpenAuth** means that it is an external application that is responsible for authenticating your users. You must have seen pages where you were asked to log on using your Google/Microsoft/Facebook/Yahoo/etc. ID. In these cases it is the respective operator that will authenticate the user for you.

**Forms authentication** is probably the most common way of authenticating the user in an internet application. The user is presented with a login page with textboxes for all the details necessary for authentication: typically a combination of username/email and password. The credentials are validated against some user store, e.g. an SQL database. Upon successful validation an authentication cookie is issued which will be used by the application on subsequent pages that require authentication. Without the cookie the user would need to log in every time they visit a protected page even after the initial login.

**How it works**

Create a new MVC4 internet application with .NET4.5 as the underlying framework.

Initially the exact type of authentication is not specified in the application. .NET has no way of knowing for sure what kind of authentication you'd like to choose. Go to the Filters folder and in it you'll find a file called InitializeSimpleMembershipAttribute.cs. There you'll see some complicated code that has one main task: initialise Forms based auth in a lazy manner. If you are 100% sure that you'll go with forms auth then this piece of code is overkill. Locate the following bit of code within the file:



    WebSecurity.InitializeDatabaseConnection("DefaultConnection", "UserProfile", "UserId", "UserName", autoCreateTables: true);


This is the essence of this file: this call will set up the necessary authentication infrastructure. Let's inspect the components:

* "DefaultConnection": name of the connection string to the database where the user data is stored
* "UserProfile": name of the table that stores user information, i.e. the user profile
* "UserId": the column where the primary key of the user is located so that the authentication framework can look up a user using their primary key
* "UserName": the column that stores the user's username

You can have many more columns in the user table but you must at least give it a primary key and a user name column.
Since we know that we want to use forms auth we can let this initialisation run during application start-up. So cut this call and paste it in Global.asax.cs:



    protected void Application_Start()
            {
                WebSecurity.InitializeDatabaseConnection("DefaultConnection", "UserProfile", "UserId", "UserName", autoCreateTables: true);
                AreaRegistration.RegisterAllAreas();

                WebApiConfig.Register(GlobalConfiguration.Configuration);
                FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
                RouteConfig.RegisterRoutes(RouteTable.Routes);
                BundleConfig.RegisterBundles(BundleTable.Bundles);
                AuthConfig.RegisterAuth();
            }


Add the necessary using statement to include the WebMatrix.WebData library. There's no need for the InitializeSimpleMembershipAttribute.cs file so you can delete it from the project. You'll receive some build errors but before we solve them that there's one more step to take. Locate the AccountModels.cs file in the Models folder. In there you'll find a class called UsersContext:



    public class UsersContext : DbContext
        {
            public UsersContext()
                : base("DefaultConnection")
            {
            }

            public DbSet<UserProfile> UserProfiles { get; set; }
        }


We don't want to use the database context that the template provided for use. We would like to take control of the user store and use our own user context. We don't yet have a database so we'll need to create one. We'll follow the code-first approach meaning that we'll describe the structure of our tables with C# and let Entity Framework create our tables. Delete the above code from AccountModels.cs and insert a class called SecurityBasicsContext in the Models folder:



    public class SecurityBasicsContext : DbContext
        {
            public SecurityBasicsContext() : base("name=DefaultConnection")
            {}
        }


Note the 'name' parameter in the base constructor. We'll come back to this later.

In AccountModels.cs you'll also see a class called UserProfile:



    [Table("UserProfile")]
        public class UserProfile
        {
            [Key]
            [DatabaseGeneratedAttribute(DatabaseGeneratedOption.Identity)]
            public int UserId { get; set; }
            public string UserName { get; set; }
        }


We will need to replace this with our own code as well. We want to make sure that we own this code and we have the freedom to customise it the way we want and that we don't use some prepared code instead. Insert a class called UserProfile in the Models folder and cut and paste the above code to that. Import all the necessary namespaces into the new class so that you end up with legal C# code. This is where you can add custom properties such as favourite colour, but remember that a unique ID and a user name property are obligatory.

Now go back to SecurityBasicsContext.cs and just below the constructor add the following code:



    public DbSet<UserProfile> UserProfiles { get; set; }


DbSet describes a table of a certain type, in this case a table of user profiles. This will be translated into an SQL table called UserProfile with two columns: UserId and UserName as the UserProfile object has exactly those two properties right now.

We can now fix the errors that were created by our changes. It should be simple:

1\. Remove the references to InitializeSimpleMembershipAttribute. Example: the AccountController class declaration is decorated with this attribute, so erase it. Do the same everywhere this attribute is referenced.

2\. Note that you will also receive build errors when you removed the UsersContext object from AccountModels.cs. The AccountController will complain as it had a reference to that object in a using statement:



    using (UsersContext db = new UsersContext())


Now we have built our own database context we can replace this reference with our own implementation:



    using (SecurityBasicsContext db = new SecurityBasicsContext())


Now you should be able to build the project. The next step involves letting EntityFramework know of our database context class. The operation when the C# code describing our database tables is converted into tables in an SQL database is called a 'Migration'. We must activate this feature for our database context. Open the Package Manager Console and type 'PM> Enable-Migrations'. You should see something like this:

![Enable migrations command output][1]

**Aside: Migrations**

This operation added a folder called Migrations to your project. Locate that folder and in it you'll find a file called Configuration.cs. Right now it's quite empty: there is a setting to disable automatic migrations:



    public Configuration()
            {
                AutomaticMigrationsEnabled = false;
            }


In short: if you make a change to your context class and issue the Update-Database command in the Package Manager Console then the framework will detect these changes and update the underlying database accordingly. However, if the above property is set to false then there will be no such automatic migrations. You'll need to set it to true if you want to enable schema migrations. We'll need this feature later on so set it to true. You may be wondering why this feature should be disabled. In a production environment you may for example have strict rules regarding database changes where a DBA has the last word. In such a case it can be dangerous to let all changes be propagated to the database every time the context file is changed.

There is also an overridden method called Seed. This is used to populate the database with some data using C# code upon running the migration feature. This is a great tool if you have several tables in your database context class and want to insert some default data in it so that you never start with an empty database.

However, this post is not about EntityFramework and Migrations and these changes will suffice for our purposes. If you need more details on these topics then there are a lot of resources on the Internet.

**Where is the database located?**

You may be wondering where the database is located and what connection string will be used to save our users. Go to web.config and locate the connectionStrings section. You will find a connection sting similar to this:



    <connectionStrings>
        <add name="DefaultConnection" connectionString="Data Source=(LocalDb)v11.0;Initial Catalog=aspnet-SecurityBasics-20130112110929;Integrated Security=SSPI;AttachDBFilename=|DataDirectory|aspnet-SecurityBasics-20130112110929.mdf" providerName="System.Data.SqlClient" />
      </connectionStrings>


Entity framework will use a local SQL express with DB and .mdf names as above. The database name is not too sexy so let's remove the 'noisy' bits as follows:



    <connectionStrings>
        <add name="DefaultConnection" connectionString="Data Source=(LocalDb)v11.0;Initial Catalog=SecurityBasicsDb;Integrated Security=SSPI;AttachDBFilename=|DataDirectory|SecurityBasicsDb.mdf" providerName="System.Data.SqlClient" />
      </connectionStrings>


Note the name of the connection string: 'DefaultConnection'. If you go back to our SecurityBasicsContext object you'll find this name in the constructor. EntityFramework will use the connection string identified by this name in the database operations.

As we now have a table called UserProfile in our SecurityBasicsContex class we need to somehow create that table in the local SQL Express database. Go to the Package Manager Console and issue the following command: 'PM> Update-Database'. If all went well then the Console should tell you something like this:

Applying automatic migration: 201301121943084_AutomaticMigration.
Running Seed method.

Now open the Server Explorer in Visual Studio (Ctrl W, L) and then open the Data Connections node. You should see our new database with the UserProfile table:

![Database in Server Explorer][2]

If you right-click this table and select Open Table Definition you'll see the exact column types that the Migration process created. The UserName column is of type NVARCHAR(MAX) NULL which is probably not a good idea. Performance-wise it's better to limit the number of characters. Also, as this is definitely an obligatory field NULL values should not be permitted. Feel free to update the column type to VARCHAR(256) NOT NULL.

It's time to run the application. The MVC4 internet application template already includes the necessary Views, Controllers and Models to handle registration and logon/logoff scenarios. You can of course change the layout or add new fields to the registration page, but you get a lot of stuff for free which is nice.

On the Index page you'll see a link called Register in the top right corner:

![Register user link][3]

Click on it to go the default Register page prepared by the MVC4 internet application template. Enter a username and password and click Register. If all went as expected you should be redirected to the home page with a greeting that replaced the Register link:

![Successful registration][4]

You just managed to register on your own site! Where is the username extracted from? Open _Layout.cshtml in the Views/Shared folder and locate the following bit of code:



    <section id="login">
       @Html.Partial("_LoginPartial")
    </section>


So the Layout page uses a partial view called _LayoutPartial to show the login/logout section of the main layout. Locate _LayoutPartial.cshtml in the Views/Shared folder and you'll see the following bit of code in there:



    Hello, @Html.ActionLink(User.Identity.Name, "Manage", "Account", routeValues: null, htmlAttributes: new { @class = "username", title = "Manage" })!


The User object represents the currently logged on user. It is of type IPrincipal which probably sounds familiar and is found in the System.Security.Principal namespace.

Go back to Server Explorer in your VS project and refresh the Tables folder. You'll see some new tables that the WebSecurity framework created upon inserting the very first user:

![Asp.net membership tables][5]

You can read more about WebSecurity in part 2 of this introduction.

Right-click the UserProfile table and select 'Show table data' in the context menu. You'll see the user that you registered before. Take note of the ID, which should be '1'. Now check the contents of the webpages_Membership table; it in turn should also have a record with UserId 1 linking your user table with webpages_Membership. This is where the password is stored in an encrypted format. This is good news: the membership mechanism hashes the password for us so that it is not stored in clear text.

The webpages_OAuthMembership table will hold the information on external membership providers, such as Google, Microsoft, Yahoo etc. This table will be populated if your user enters his/her external membership details such as a FacebookID.

The webpages_Roles table comes into the picture when we talk about authorisation, i.e. checking if a user has the right to view a specific page in your application. Here you can store string values such as 'Sales', 'Purchasing' or 'Administrator'.

Finally, the webpages_UsersInRoles is a many-to-many table between webpages_Membership and webpages_Roles. It's obvious that a Role may have many users and a User can be a member of more than one Role.

Run the website again and try to log in using the 'Login' link just to the right of 'Register'. Enter your username and password and there you are, you have logged on! Click the link showing your username which will lead to a page where you can edit your profile. All of this functionality is provided out of the box when we created a new internet application.

You can view the list of posts on Security and Cryptography [here][6].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/01/enablemigrations1.png?w=300&h=56
[2]: http://dotnetcodr.files.wordpress.com/2013/01/databasecreatedinserverexplorer.png?w=630
[3]: http://dotnetcodr.files.wordpress.com/2013/01/registerlink.png?w=630
[4]: http://dotnetcodr.files.wordpress.com/2013/01/successfulregistration.png?w=630
[5]: http://dotnetcodr.files.wordpress.com/2013/01/aspnetmembershiptables.png?w=630
[6]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
