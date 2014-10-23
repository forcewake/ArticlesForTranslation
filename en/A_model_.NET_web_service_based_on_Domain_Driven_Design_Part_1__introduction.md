[Source](http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "Permalink to A model .NET web service based on Domain Driven Design Part 1: introduction")

# A model .NET web service based on Domain Driven Design Part 1: introduction

**Introduction**

Layered architecture has been the norm in enterprise projects for quite some time now. We layer our solutions into groups of responsibilities: UI, services, data access, infrastructure and others. We can connect these layers by setting references to them from the consuming layer. There are various ways to set these layers and references. The choices you make here depend on the project type – Web, WPF, WCF, games etc. – you're working on and the philosophy you follow.

On the one hand you may be very technology driven and base your choices primarily on the technologies available to build your project: Entity Framework, Ajax, MVC etc. Most programmers are like that and that's natural. It's a safe bet that you've become a developer because you're interested in computers and computer-related technologies like video games and the Internet. It's exciting to explore and test new technologies as they emerge: .NET, ORM, MVC, WebAPI, various JavaScript libraries, WebSockets, HighCharts, you name it. In fact, you find programming so enthralling that you would keep doing it for free, right? Just don't tell your boss about it in the next salary review… With this approach it can happen in a layered architecture that the objects in your repository layer, i.e. the data access layer, such as an SQL Server database, will receive the top focus in your application. This is due to the popular ORM technologies available in .NET: EntityFramework and Linq to SQL. They generate classes based on some database representation of your domain objects. All other layers will depend on the data access layer. We'll see an example of this below.

On the other hand you may put a different part of the application to the forefront: the domain. What is the domain? The domain describes your business. It is a collection of all concepts and logic that drive your enterprise. If you follow this type of philosophy, which is the essence of Domain Driven Design (DDD), then you give the domain layer the top priority. It will be the most important ingredient of the application. All other layers will depend on the domain layer. The domain layer will be an entirely independent one that can function on its own. The most important questions won't be "How do we solve this with WebSockets?" or "How does it work with AngularJs?" but rather like these ones:

* How does the Customer fit into our domain?
* Have we modelled the Product domain correctly?
* Is this logic part of the Product or the OrderItem domain?
* Where do we put the pricing logic?

Technology-related questions will suddenly become implementation details where the exact implementation type can vary. We don't really care which graph package we use: it's an implementation detail. We don't really care which ORM technology we use: it's an implementation detail.

Of course technology-related concerns will be important aspects of the planning phase, but not as important as how to correctly represent our business in code.

In a technology-driven approach it can easily happen that the domain is given low priority and the technology choices will directly affect the domain: "We have to change the Customer domain because its structure doesn't fit Linq to SQL". In DDD the direction is reversed: "An important part of our Customer domain structure has changed so we need to update our Linq to SQL classes accordingly.". A change in the technology should never force a change in your domain. Your domain is an independent entity that only changes if your business rules change or you discover that your domain logic doesn't represent the true state of things.

Keep in mind that it is not the number of layers that makes your solution "modern". You can easily layer your application in a way that it becomes a tightly coupled nightmare to work with and – almost – impossible to extend. If you do it correctly then you will get an application that is fun and easy to work with and is open to all sorts of extensions, even unanticipated ones. This last aspect is important to keep in mind. Nowadays customers' expectations and wishlists change very often. You must have noticed that the days of waterfall graphs, where the the first version of the application may have rolled out months or even years after the initial specs were written, are over. Today we have a product release every week almost where we work closely with the customer on every increment we build in. Based on these product increments the customer can come with new demands and you better be prepared to react fast instead of having to rewrite large chunks of your code.

**Goals of this series**

I'll start with stating what's definitely not the goal: give a detailed account on all details and aspects of DDD. You can read all about DDD by its inventor Eric Evans in [this][1] book. It's a book that discusses all areas of DDD you can think of. It's no use regurgitating all of that in these posts. Also, building a model application which uses all concepts from DDD may easily grow into full-fledged enterprise application, such as SAP.

Instead, I'll try to build the skeleton of a .NET solution that takes the most important ideas from DDD. The result of this skeleton will hopefully be a solution that you can learn from and even tweak it and use in your own project. Don't feel constrained by this specific implementation – feel free to change it in a way that fits your philosophy.

However, if you're completely new to DDD then you should still benefit from these posts as I'll provide explanations for those key concepts. I'll try to avoid giving you too much theory and rather concentrate on code but some background must be given in order to explain the key ideas. We'll need to look at some key terms before providing any code. I'll also refer to ideas from SOLID here and there – if you don't understand this term then start [here][2].

Also, it would be great to build the example application with TDD, but that would add a lot of noise to the main discussion. If you don't know what TDD means, start [here][3].

The ultimate goal is to end up with a skeleton solution with the following layers:

* Infrastructure: infrastructure services to accommodate cross-cutting concerns
* Data access (repository): data access and persistence technology layer, such as EntityFramework
* Domain: the domain layer with our business entities and logic, the centre of the application
* Application services: thin layer to provide actions for the consumer(s) such as a web or desktop app
* Web: the ultimate consumer of the application whose only purpose is to show data and accept requests from the user

…where the layers communicate with each other in a loosely coupled manner through abstractions. It should be easy to replace an implementation with another. Of course writing a new version of – say – the repository layer is not a trivial task and layering won't make it easier. However, the switch to the new implementation should go with as little pain as possible **without the need to modify any of the other layers**.

The Web layer implementation will actually be a web service layer using the Web API technology so that we don't need to waste time on CSS and HTML. If you don't know what Web API is about, make sure you understand the basics from [this][4] post.

UPDATE: the skeleton application is available for download on GitHub [here][5].

**Doing it wrong**

Let's take an application that is layered using the technology-driven approach. Imagine that you've been tasked with creating a web application that lists the customers of the company you work for. You know that layered architectures are the norm so you decide to build these 3 layers:

* User Interface
* Domain logic
* Data access

You know more or less what the Customer domain looks like so you create the following table in SQL Server:

![Customer table in SQL Server][6]

Then you want to use the ORM capabilities of Visual Studio to create the Customer class for you using the Entity Framework Data Model Wizard. Alternatively you could go for a Linq to SQL project type, it doesn't make any difference to our discussion. At this point you're done with the first layer in your project, the data access one:

![Entity Framework model context][7]

EntityFramework has created a DbContext and the Customer class for us.

Next, you know that there's logic attached to the Customer domain. In this example we want to refine the logic of what price reduction a customer receives when purchasing a product. Therefore you implement the Domain Logic Layer, a C# class library. To handle Customer-related queries you create a CustomerService class:



    public class CustomerService
    {
    	private readonly ModelContext _objectContext;

    	public CustomerService()
    	{
    		_objectContext = new ModelContext();
    	}

    	public IEnumerable<Customer> GetCustomers(decimal totalAmountSpent)
    	{
    		decimal priceDiscount = totalAmountSpent > 1000M ? .9m : 1m;
    		List<Customer> allCustomers = (from c in _objectContext.Customers select c).ToList();
    		return from c in allCustomers select new Customer()
    		{
    			Address = c.Address
    			, DiscountGiven = c.DiscountGiven * priceDiscount
    			, Id = c.Id
    			, IsPreferred = c.IsPreferred
    			, Name = c.Name
    		};
    	}
    }


As you can see the final price discount depends also on the amount of money spent by the customer, not just the value currently stored in the database. So, that's some extra logic that is now embedded in this CustomerService class. If you're familiar with SOLID then the '_objectContext = new ModelContext();' part immediately raises a warning flag for you. If not, then make sure to read about the [Dependency Inversion Principle][8].

In order to make this work I had to add a library reference from the domain logic to the data access layer. I also had to add a reference to the EntityFramework library due to the following exception:

**The type 'System.Data.Entity.DbContext' is defined in an assembly that is not referenced. You must add a reference to assembly 'EntityFramework, Version=4.4.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089′.**

So I import the EntityFramework library into the DomainLogic layer to make the project compile.

At this point I have the following projects in the solution:

![Data access and domain logic layer][9]

The last layer to add is the UI layer, we'll go with an MVC solution. The Index() method which lists all customers might look like the following:



    public class HomeController : Controller
    {
    	public ActionResult Index()
    	{
    		decimal totalAmountSpent = 1000; //to be fetched with a different service, ignored here
    		CustomerService customerService = new CustomerService();
    		IEnumerable<Customer> customers = customerService.GetCustomers(totalAmountSpent);
    		return View(customers);
    	}
    }


In order to make this work I had to reference both the DomainLogic AND the DataAccess projects from the UI layer. I now have 3 layers in the solution:

![DDD three layers][10]

So I press F5 and… …get an exception:

"No connection string named 'ModelContext' could be found in the application config file."

This exception is thrown at the following line within CustomerService.GetCustomers:



    List<Customer> allCustomers = (from c in _objectContext.Customers select c).ToList();


The ModelContext object implicitly expects that a connection string called ModelContext is available in the web.config file. It was originally inserted into app.config of the DataAccess layer by the EntityFramework code generation tool. So a quick fix is to copy the connection string from this app.config to web.config of the web UI layer:



    <connectionStrings>
        <add name="ModelContext" connectionString="metadata=res://*/Model.csdl|res://*/Model.ssdl|res://*/Model.msl;provider=System.Data.SqlClient;provider connection string=&quot;data source=mycomputer;initial catalog=Test;integrated security=True;MultipleActiveResultSets=True;App=EntityFramework&quot;" providerName="System.Data.EntityClient" />
    </connectionStrings>


**Evaluation**

Have we managed to build a "modern" layered application? Well, we certainly have layers, but they are tightly coupled. Both the UI and the Logic layer are strongly dependent on the DataAccess layer. Examples of tight coupling:

* The HomeController/Index method constructs a CustomerService object
* The HomeController/Index directly uses the Customer object from the DataAccess layer
* The CustomerService constructs a new ModelContext object
* The UI layer is forced to include an EntityFramework-specific connection string in its config file

At present we have the following dependency graph:

![DDD wrong dependency graph][11]

It's immediately obvious that the UI layer can easily bypass any logic and consult the data access layer directly. It can read all customers from the database without applying the extra logic to determine the price discount.

Is it possible to replace the MVC layer with another UI type, say WPF? It certainly is, but it does not change the dependency graph at all. The WPF layer would also need to reference both the Logic and the DataAccess layers.

Is it possible to replace the data access layer? Say you're done with object-relational databases and want to switch to file-based NoSql solutions such as MongoDb. Or you might want to move to a cloud and go with an Azure key-value type of storage. We don't need to go as far as entirely changing the data storage mechanism. What if you only want to upgrade your technology Linq to SQL to EntityFramework? It can be done, but it will be a difficult process. Well, maybe not in this small example application, but imagine a real-world enterprise app with hundreds of domains where the layers are coupled to such a degree with all those references that removing the data access layer would cause the application to break down immediately.

The ultimate problem is that the entire domain model is defined in the data access layer. We've let the technology – EntityFramework – take over and it generated our domain objects based on some database representation of our business. It is an entirely acceptable solution to let an ORM technology help us with programming against a data storage mechanism in a strongly-typed object-oriented fashion. Who wants to work with DataReaders and SQL commands in a plain string format nowadays? However, letting some data access automation service take over our core business and permeate the rest of the application is far from desirable. The CustomerService constructor tightly couples the DomainLogic to EntityFramework. Since we need to reference the data access layer from within our UI even the MVC layer is coupled to EntityFramework. In case we need to change the storage mechanism or the domain logic layers we potentially have a considerable amount of painful rework to be done where you have to manually overwrite e.g. the Linq to SQL object context references to EntityFramework object context ones.

We have failed miserably in building a loosely coupled, composable application whose layers have a single responsibility. The domain layer, as stated in the introduction, should be the single most important layer in your application which is independent of all other layers – possibly with the exception of infrastructure services which we'll look at in a later post in this series. Instead, now it is a technological data access layer which rules and where the domain logic layer is relegated to a second-class citizen which can be circumvented entirely.

The domain is at the heart of our business. It exists regardless of any technological details. If we sell cars, then we sell cars even if the car objects are stored in an XML file or in the cloud. It is a technological detail how our business is represented in the data storage. It is something that the Dev department needs to be concerned with. The business rules and domains have a lot higher priority than that: it is known to all departments of the company.

Another project type where you can easily confuse the roles of each layer is ASP.NET WebForms with its code behind. It's very easy to put a lot of logic, database calls, validation etc. into the code-behind file. You'll soon end up with the **Smart UI anti-pattern** where the UI is responsible for all business aspects of your application. Such a web application is difficult to compose, unit test and decouple. This doesn't mean that you must forget WebForms entirely but use it wisely. Look into the Model-View-Presenter pattern, which is sort of MVC for WebForms, and then you'll be fine.

So how can we better structure our layers? That will be the main topic of the series. However, we'll need to lay the foundation for our understanding with some theory and terms which we'll do in the next post.

View the list of posts on Architecture and Patterns [here][12].

### Like this:

Like Loading...

### _Related_

[1]: http://www.amazon.co.uk/Domain-driven-Design-Tackling-Complexity-Software/dp/0321125215/ref=sr_1_1?ie=UTF8&qid=1377594203&sr=8-1&keywords=eric+evans "DDD by Eric Evans on Amazon"
[2]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[3]: http://dotnetcodr.com/2013/03/25/test-driven-development-in-net-part-1-the-absolute-basics-of-red-green-refactor/ "Test Driven Development in .NET Part 1: the absolute basics of Red, Green, Refactor"
[4]: http://www.asp.net/web-api "Web API on MSDN"
[5]: https://github.com/andras-nemes/DDDSkeletonNet "DDD skeleton application on GitHub"
[6]: http://dotnetcodr.files.wordpress.com/2013/08/dddcustomertable1.png?w=630
[7]: http://dotnetcodr.files.wordpress.com/2013/08/dddmodelcontext1.png?w=630
[8]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[9]: http://dotnetcodr.files.wordpress.com/2013/08/dddwrongtwolayers.png?w=630
[10]: http://dotnetcodr.files.wordpress.com/2013/08/dddwrongthreelayers.png?w=630
[11]: http://dotnetcodr.files.wordpress.com/2013/08/dddwrongdependencygraph.png?w=630
[12]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
