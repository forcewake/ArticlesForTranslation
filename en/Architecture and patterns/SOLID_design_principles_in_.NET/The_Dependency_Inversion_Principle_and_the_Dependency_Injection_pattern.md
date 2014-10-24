[Source](http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "Permalink to SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern")

# SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern

**Introduction**

The Dependency Inversion Principle (DIP) helps to decouple your code by ensuring that you depend on abstractions rather than concrete implementations. Dependency Injection (DI) is an implementation of this principle. In fact DI and DIP are often used to mean the same thing. A key feature of DIP is programming to abstractions so that consuming classes can depend on those abstractions rather than low-level implementations. DI is the act of supplying all classes that a service needs rather than letting the service obtain its concrete dependencies. The term Dependency Injection may first sound like some very advanced technology but in reality there are absolutely no complex techniques behind it. As long as you understand abstractions and constructors and methods that can accept parameters you'll understand DI as well.

Another related term is **Inversion of Control**, IoC. IoC is an older term than DI. Originally it meant a programming style where a framework or runtime controlled the programme flow. Sticking to this definition software that is developed with .NET uses IoC – in this case .NET is the controlling framework. You hook up to its events, lifetime management etc. You are in control of your methods and references but .NET provides the ultimate glue. Nowadays we're so used to working with frameworks that we don't care – we're actually happy that we don't need to worry about tedious infrastructure stuff. With time IoC drifted away to mean Inversion of Control Containers which are mechanisms that control dependencies. Martin Fowler was the one who came up with the term Dependency Injection to mean this particular flavour of IoC. Thus, DI is still the correct term to describe the control over dependencies although people often use IoC instead.

DIP is a large topic so it will span several posts to discuss it thoroughly. However, one important conclusion up front is the following:

**The frequency of the 'new' keyword in your code is a rough estimate of the degree of coupling in your object structure.**

A side note: in the demo I'll concentrate on interfaces but DI works equally well with abstract classes.

**What are dependencies?**

Dependencies can come in different forms.

A **framework** dependency is your choice of the development framework, such as .NET. You as a .NET developer are probably comfortable with that dependency as it's unlikely to change during the product's lifetime. Most often if you need the product to run on a different framework then it will be rewritten for that platform in a different language without discarding the original product. As this dependency is very unlikely to change and it's a very high level dependency we don't worry about it in this context.

**Third party libraries** introduce a lower level dependency that may well change over time. If a class depends on an external dll then that class may be difficult to test as we need that library to be in a consistent state when testing. However, some of those dependencies may never change. E.g. if I want to communicate with the Amazon cloud in code then it's best to download and reference the Amazon .NET SDK and use that for entire lifetime of the product. It's unlikely that I will write my own SDK to communicate with the Amazon web services.

**Databases** store your data and they can come in many different forms: SQL Server, MySQL, MongoDb, RavenDb, Oracle etc. You may think that an application will never change its storage mechanism, but you'll better be prepared and abstract it away behind a repository as we'll see in the demo.

Some other less obvious dependencies include the **File System**, **Emails**, **Web services** and other networking technologies.

**System resources** such as the Clock to get the current time. You may think that's unnecessary to abstract _that_ away. After all, why would you ever write your own time provider if the System Clock is available? Think of unit testing a method that depends on time: a price discount is given between 5pm and 6pm on every Friday. How would you unit test that logic? Do you really wait until 5pm on a Friday and hope that you can make the test pass before 6pm? That's not too clever, as your unit test can only run during that time. So there's a valid point in abstracting away system resources as well.

**Configuration** in general, such as the app settings you read from the config file.

The **new** keyword, as hinted at in the introduction, generally points to tighter coupling and an introduction of an extra dependency. As soon as you write var o = new MyObject() in your class MyClass then MyClass will be closely dependent on MyObject(). If MyObject() changes its behaviour then you have to be prepared for changes and unexpected behaviour in MyClass as well. Using static methods is no way out as your code will depend on the object where the static method is stored, such as MyFactory.GetObject(). You haven't newed up a MyFactory object but your code is still dependent on it.

Depending on **threading-related** objects such as Thread or Task can make your methods difficult to test as well. If there's a call to Thread.Sleep then even your unit test will need to wait.

Anytime you introduce a new **library reference** you take on an extra dependency. If you have a 4 layers in your solution, a UI, Services, Domains and Repository then you can glue them together by referencing them in Visual Studio: UI uses Services, Services use the Repo etc. The degree of coupling is closely related to how painful it is to replace those layers with new ones. Do you have to sit for days in front of your computer trying to reconnect all the broken links or does it go relatively fast as all you need to do is inject a different implementation of an abstraction?

Getting random numbers using the **Random** object also introduces an extra dependency that's hard to test. If your code depends on random numbers then you may have to run the unit test many times before you get to test all branches. Instead, you can provide a different implementation of generating random numbers where you control the outcome of this mechanism so that you can easily test your code.

I've provided a hint on how to abstract away built-in .NET objects in your code at the end of this post.

**Demo**

In the demo we'll use a simple Product domain whose price can be adjusted using a discount strategy. The Products will be retrieved using a ProductRepository. A ProductService class will depend on ProductRepository to communicate with the underlying data store. We'll first build the classes without DIP in mind and then we'll refactor the code. We'll keep all classes to a minimum without any real implementation so that we can concentrate on the issues at hand. Open Visual Studio and create a new console application. Add the following Product domain class:



    public class Product
    {
    	public void AdjustPrice(ProductDiscount productDiscount)
    	{
    	}
    }


…where ProductDiscount looks even more simple:



    public class ProductDiscount
    {
    }


ProductRepository will help us communicate with the data store:



    public class ProductRepository
    {
    	public IEnumerable<Product> FindAll()
    	{
    		return new List<Product>();
    	}
    }


We connect the above objects in the ProductService class:



    public class ProductService
    {
    	private ProductDiscount _productDiscount;
    	private ProductRepository _productRepository;

    	public ProductService()
    	{
    		_productDiscount = new ProductDiscount();
    		_productRepository = new ProductRepository();
    	}

    	public IEnumerable<Product> GetProducts()
    	{
    		IEnumerable<Product> productsFromDataStore = _productRepository.FindAll();
    		foreach (Product p in productsFromDataStore)
    		{
    			p.AdjustPrice(_productDiscount);
    		}
    		return productsFromDataStore;
    	}
    }


A lot of code is still written like that nowadays. This is the traditional approach in programming: high level modules call low level modules and instantiate their dependencies as they need them. Here ProductService calls ProductRepository, but before it can do that it needs to new one up using the 'new' keyword. The client, i.e. the ProductService class must fetch the dependencies it needs in order to carry out its tasks. Two dependencies are created in the constructor with the 'new' keyword. This breaks the [Single Responsibility Principle][1] as the class is forced to carry out work that's not really its concern.

The ProductService is thus **tightly coupled** to those two concrete classes. It is difficult to use different discount strategies: there may be different discounts at Christmas, Halloween, Easter, New Year's Day etc. Also, there may be different strategies to retrieve the data from the data store such as using an SQL database, a MySql database, a MongoDb database, file storage, memory storage etc. Whenever those strategies change you must update the ProductService class which breaks just about every principle we've seen so far in this series on SOLID.

It is also difficult to test the product service in isolation. The test must make sure that the ProductDiscount and ProductRepository objects are in a valid state and perform as they are expected so that the test result does not depend on them. If the ProductService sends an email then even the unit test call must send an email in order for the test to pass. If the emailing service is not available when the test runs then your test will fail regardless of the true business logic of the method under test.

All in all it would be easier if we could provide any kind of strategy to the ProductService class without having to change its implementation. This is where abstractions and DI enters the scene.

As we can have different strategies for price discounts and data store engines we'll need to introduce abstractions for them:



    public interface IProductDiscountStrategy
    {
    }




    public interface IProductRepository
    {
    	IEnumerable<Product> FindAll();
    }


Have the discount and repo classes implement these interfaces:



    public class ProductDiscount : IProductDiscountStrategy
    {
    }




    public class ProductRepository : IProductRepository
    {
    	public IEnumerable<Product> FindAll()
    	{
    		return new List<Product>();
    	}
    }


When we program against abstractions like that then we introduce a **seam** into the application. It comes from seams on cloths where they can be sewn together. Think of the teeth – or whatever they are called – on LEGO building blocks. You can mix and match those building blocks pretty much as you like due to the standard seams they have. In fact LEGO applied DIP and DI pretty well in their business idea.

Let's update the ProductService class step by step. The first step is to change the type of the private fields:



    private IProductDiscountStrategy _productDiscount;
    private IProductRepository _productRepository;


You'll see that the AdjustPrice method of Product now breaks so we'll update it too:



    public void AdjustPrice(IProductDiscountStrategy productDiscount)
    {
    }


Now the Product class can accept any type of product discount so we're on the right track. See that the AdjustPrice accepts a parameter of an abstract type? We'll look at the different flavours of DI later but that's actually an application of the pattern called **method injection**. We inject the dependency through a method parameter. We'll employ **constructor injection** to remove the hard dependency on the ProductRepository class within ProductService:



    public ProductService(IProductRepository productRepository)
    {
    	_productDiscount = new ProductDiscount();
    	_productRepository = productRepository;
    }


Here's the final version of the ProductService class:



    public class ProductService
    {
    	private IProductRepository _productRepository;

    	public ProductService(IProductRepository productRepository)
    	{
    		_productRepository = productRepository;
    	}

    	public IEnumerable<Product> GetProducts(IProductDiscountStrategy productDiscount)
    	{
    		IEnumerable<Product> productsFromDataStore = _productRepository.FindAll();
    		foreach (Product p in productsFromDataStore)
    		{
    			p.AdjustPrice(productDiscount);
    		}
    		return productsFromDataStore;
    	}
    }


Now clients of ProductService, possibly a ProductController in an MVC app, will need to provide these dependencies so that the ProductService class can concentrate on its job rather than having to new up dependencies. Obtaining the correct pricing and data store strategy should not be the responsibility of the ProductService. This relates well to the [Open-Closed Principle][2] in that you can create new pricing strategies without having to update the ProductService class.

It's important to note that the ProductService class is now **honest** and **transparent**. It is honest about its needs and doesn't try to hide the external objects it requires in order to fulfil its job, i.e. its dependencies are **explicit**. Clients using the ProductService class will know that it requires a discount strategy and a product repository. There are no hidden side effects and unexpected results.

The opposite case is a class having **implicit** dependencies such as the first version of the ProductService class. It's not obvious for the caller that ProductService will use external libraries and it does not have any control whatsoever over them. In the best case if you have access to the code then you can inspect it and maybe even refactor it. The first version of ProductService tells the client that it suffices to simply create a new ProductService object and then it will be able to fetch the products. However, what do we do if the database is not present? Or if the prices are not correct due to the wrong discount strategy? Then comes the part where ProductService says 'oh, sorry, I need database access before I can do anything, didn't you know that…?".

Imagine that you get a new job as a programmer. Normally all 'dependencies' that you need for your job are provided for you: a desk, a computer, the programming software etc. Now think how this would work without DI: you'll have to buy a desk, get a computer within some specified price range, install the software yourself etc. That is the opposite of DI and that is how ProduceService worked before the refactoring.

A related pattern to achieve DI is the [Adapter][3] pattern. It provides a simple mechanism to abstract away dependencies that you have no control over, such as the built-in .NET classes. E.g. if your class sends emails then your default choice could be to use the built-in .NET emailing objects, such as MailMessage and SmtpClient. You cannot easily abstract away that dependency as you have no access to the source code, you cannot make it implement a custom interface, such as IEmailingService. The Adapter patter will help you solve that problem. Also, unit testing code that sends emails is cumbersome as even the test call will need to send out an email. Ideally we don't test those external services so instead inject a mock object in place of the real one. How that is done? Start [here][4].

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[2]: http://dotnetcodr.com/2013/08/15/solid-design-principles-in-net-the-open-closed-principle/ "SOLID design principles in .NET: the Open-Closed Principle"
[3]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[4]: http://dotnetcodr.com/2013/03/25/test-driven-development-in-net-part-1-the-absolute-basics-of-red-green-refactor/ "Test Driven Development in .NET Part 1: the absolute basics of Red, Green, Refactor"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
