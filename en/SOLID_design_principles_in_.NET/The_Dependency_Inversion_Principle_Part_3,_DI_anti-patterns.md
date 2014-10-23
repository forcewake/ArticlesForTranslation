[Source](http://dotnetcodr.com/2013/09/02/solid-design-principles-in-net-the-dependency-inversion-principle-part-3-di-anti-patterns/ "Permalink to SOLID design principles in .NET: the Dependency Inversion Principle Part 3, DI anti-patterns")

# SOLID design principles in .NET: the Dependency Inversion Principle Part 3, DI anti-patterns

In the [previous][1] post we discussed the various techniques how to implement Dependency Injection. Now it's time to show how NOT to do DI. Therefore we'll look at a couple of anti-patterns in this post.

**Lack of DI**

The most obvious DI anti-patterns is the total absence of it where the class controls it dependencies. It is the most common anti-pattern in DI to see code like this:



    public class ProductService
    {
    	private ProductRepository _productRepository;

    	public ProductService()
    	{
    		_productRepository = new ProductRepository();
    	}
    }


Here the consumer class, i.e. ProductService, creates an instance of the ProductRepository class with the 'new' keyword. Thereby it directly controls the lifetime of the dependency. There's no attempt to introduce an abstraction and the client has no way of introducing another type – implementation – for the dependency.

.NET languages, and certainly other similar platforms as well, make this extremely easy for the programmer. There is no automatic SOLID checking in Visual Studio, and why would there be such a mechanism? We want to give as much freedom to the programmer as possible, so they can pick – or even mix – C#, VB, F#, C++ etc., write in an object-oriented way or follow some other style, so there's a high degree of control given to them within the framework. So it feels natural to new up objects as much as we like: if I need something I'll need to go and get it. If the ProductService needs a product repository then it will need to fetch one. So even experienced programmers who know SOLID and DI inside out can fall into this trap from time to time, simply because it's easy and programming against abstractions means more work and a higher degree of complexity.

The first step towards salvation may be to declare the private field as an abstract type and mark it as readonly:



    public class ProductService
    {
    	private readonly IProductRepository _productRepository;

    	public ProductService()
    	{
    		_productRepository = new ProductRepository();
    	}
    }


However, not much is gained here yet, as at runtime _productRepository will always be a new ProductRepository.

A common, but very wrong way of trying to resolve the dependency is by using some [Factory][2]. Factories are an extremely popular pattern and they seem to be the solution to just about anything – besides copy/paste of course. I'm half expecting Bruce Willis to save the world in Die Hard 27 by applying the static factory pattern. So no wonder people are trying to solve DI with it too. You can see from the post on factories that they come in 3 forms: abstract, static and concrete.

The "solution" with the concrete factory may look like this:



    public class ProductRepositoryFactory
    {
    	public ProductRepository Create()
    	{
    		return new ProductRepository();
    	}
    }




    public ProductService()
    {
    	ProductRepositoryFactory factory = new ProductRepositoryFactory();
    	_productRepository = factory.Create();
    }


Great, we're now depending directly on ProductRepositoryFactory and ProductRepository is still hard-coded within the factory. So instead of just one hard dependency we now have two, well done!

What about a static factory?



    public class ProductRepositoryFactory
    {
    	public static ProductRepository Create()
    	{
    		return new ProductRepository();
    	}
    }




    public ProductService()
    {
    	_productRepository = ProductRepositoryFactory.Create();
    }


We've got rid of the 'new', yaaay! Well, no, if you recall from the [introduction][3] using static methods still creates a hard dependency, so ProductService still depends directly on ProductRepositoryFactory and indirectly on ProductRepository.

To make matters worse the static factory can be misused to produce a sense of freedom to the client to control the type of dependency as follows:



    public class ProductRepositoryFactory
    {
    	public static IProductRepository Create(string repositoryTypeDescription)
    	{
    		switch (repositoryTypeDescription)
    		{
    			case "default":
    				return new ProductRepository();
    			case "test":
    				return new TestProductRepository();
    			default:
    				throw new NotImplementedException();
    		}
    	}
    }




    public ProductService()
    {
    	_productRepository = ProductRepositoryFactory.Create("default");
    }


Oh, brother, this is a real mess. ProductService still depends on the ProductRespositoryFactory class, so we haven't eliminated that. It is now indirectly dependent on the two concrete repo types returned by factory. Also, we now have magic strings flying around. If we ever introduce a third type of repository then we'll need to revisit the factory and inform all actors that there's a new magic string. This model is very difficult to extend. The ability to configure in code, or possibly in a config file, gives a false sense of security to the developer.

You can create a mock Product repository for a unit test scenario, extend the switch-case statement in the factory and maybe introduce a new magic string "mock" only for testing purposes. Then you can put "mock" in the Create method, recompile and run your test just for unit testing. Then you forget to put it back to "sql" or whatever and deploy the solution… Realising this you may want to overload the ProductService constructor like this:



    public ProductService(string productRepoDescription)
    {
    	_productRepository = ProductRepositoryFactory.Create(productRepoDescription);
    }


That only moves the magic string problem up the stacktrace but does nothing to solve the dependency problems outlined above.

Let's look at an abstract factory:



    public interface IProductRepositoryFactory
    {
    	IProductRepository Create(string repoDescription);
    }


ProductRepositoryFactory can implement this interface:



    public class ProductRepositoryFactory : IProductRepositoryFactory
    {
    	public IProductRepository Create(string repositoryTypeDescription)
    	{
    		switch (repositoryTypeDescription)
    		{
    			case "default":
    				return new ProductRepository();
    			case "test":
    				return new TestProductRepository();
    			default:
    				throw new NotImplementedException();
    		}
    	}
    }


You can use it from ProductService as follows:



    private IProductRepositoryFactory _productRepositoryFactory;
    private IProductRepository _productRepository;

    public ProductService(string productRepoDescription)
    {
    	_productRepositoryFactory = new ProductRepositoryFactory();
    	_productRepository = _productRepositoryFactory.Create(productRepoDescription);
    }


We need to new up a ProductRepositoryFactory, so we're back at square one. However, abstract factory is still the least harmful of the factory patterns when trying to solve DI as we can refactor this code in the following way:



    private readonly IProductRepository _productRepository;

    public ProductService(string productRepoDescription, IProductRepositoryFactory productRepositoryFactory)
    {
    	_productRepository = productRepositoryFactory.Create(productRepoDescription);
    }


This is not THAT bad. We can provide any type of factory now but we still need to provide a magic string. A way of getting rid of that string would be to create specialised methods within the factory as follows:



    public interface IProductRepositoryFactory
    {
    	IProductRepository Create(string repoDescription);
    	IProductRepository CreateTestRepository();
    	IProductRepository CreateSqlRepository();
    	IProductRepository CreateMongoDbRepository();
    }


…with ProductService using it like this:



    private readonly IProductRepository _productRepository;

    public ProductService(IProductRepositoryFactory productRepositoryFactory)
    {
    	_productRepository = productRepositoryFactory.CreateMongoDbRepository();
    }


This is starting to look like proper DI. We can inject our own version of the factory and then pick the method that returns the necessary repository type. However, this is a false positive impression again. The ProductService still controls the type of repository returned by the factory. If we wanted to test the SQL repo then we have to revisit the product service – and all other services that need a repository – and select CreateSqlRepository() instead. Same goes for unit testing. You can certainly create a mock repository factory but you'll need to make sure that the mock implementation returns mock objects for all repository types the factory returns. That breaks [ISP][4] in SOLID.

No, even in the above case the caller cannot control type of IProductRepository used within ProductRepository. You can certainly control the concrete implementation of IProductRepositoryFactory, but that's not enough.

Conclusion: factories are great for purposes other than DI. Use one of the strategies outlined in the previous post.

Lack of DI creates tightly coupled classes where one cannot exist without the other. You cannot redistribute the ProductService class without the concrete ProductRepository so it diminishes re-usability.

**Overloaded constructors**

Consider the following code:



    private readonly IProductRepository _productRepository;

    public ProductService() : this(new ProductRepository())
    {}

    public ProductService(IProductRepository productRepository)
    {
            _productRepository = productRepository;
    }


It's great that we have can inject an IProductRepository but what's the default constructor doing there? It simply calls the overloaded one with a concrete implementation of the repository interface. There you are, we've just introduced a completely unnecessary coupling. Matters get worse if the default implementation comes from an external source such as a factory seen above:



    private readonly IProductRepository _productRepository;

    public ProductService() : this(new ProductRepositoryFactory().Create("sql"))
    {}

    public ProductService(IProductRepository productRepository)
    {
    	_productRepository = productRepository;
    }


By now you know why factories are not suitable for solving DI so I won't repeat myself. Even if clients always call the overloaded constructor the class still cannot exist without either the ProductRepository or the ProductRepositoryFactory class.

The solution is easy: get rid of the default constructor and force clients to provide their own implementations.

**Service locator**

I've already written a post on this available [here][5], so I won't repeat the whole post. In short: a Service Locator resembles a proper IoC container such as StructureMap but it introduces easily avoidable couplings between objects.

**Conclusion**

We've discussed some of the ways how not to do DI. There may certainly be more of those but these are probably the most frequent ones. The most important variant to get rid of is the lack of DI which is exacerbated by the use of factories. That's also the easiest to spot – look for the 'new' keyword in conjunction with dependencies.

The use of static method and properties can also be indicators of DIP violation:



    DateTime.Now.ToString();
    DataAccess.SaveCustomer(customer);
    ProductRepositoryFactory.Create("sql");


We've seen especially in the case of factories that static methods and factories only place the dependencies one step down the execution ladder. Static methods are acceptable if they don't themselves have concrete dependencies but only use the parameters already used by the object that's calling the static method. However, if those static methods new up other dependencies which in turn may instantiate their own dependencies then that will quickly become a tightly coupled nightmare.

View the list of posts on Architecture and Patterns [here][6].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[2]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[3]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[4]: http://dotnetcodr.com/2013/08/22/solid-design-principles-in-net-the-interface-segregation-principle/ "SOLID design principles in .NET: the Interface Segregation Principle"
[5]: http://dotnetcodr.com/2013/08/08/design-patterns-and-practices-in-net-the-service-locator-anti-pattern/ "Design patterns and practices in .NET: the Service Locator anti-pattern"
[6]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
