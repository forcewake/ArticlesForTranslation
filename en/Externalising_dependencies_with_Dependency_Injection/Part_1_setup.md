[Source](http://dotnetcodr.com/2014/09/01/externalising-dependencies-with-dependency-injection-in-net-part-1-setup/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 1: setup")

# Externalising dependencies with Dependency Injection in .NET part 1: setup

**Introduction**

Earlier on this blog we talked a lot about concepts like [SOLID][1], patterns and layered software architecture using [Domain Driven Design][2].

This new series of 10 posts will draw a lot on those ideas. Here are the main goals:

* Show how hard dependencies of an object can be made external by hiding them behind abstractions
* Show the reasons why you should care about those dependencies
* Build a common infrastructure layer that you can use in your own projects

What is an infrastructure layer? We saw an example of that in [this][3] post. It is a special layer in the software architecture that holds the code which is common to all other layers in the project. This is a collection of cross-cutting concerns, like caching, security, emailing, logging etc. Logging is seldom part of any company domain but you may want to log trace messages from the "main" layers of the project: the service, the UI and the repository layer. It's not too clever to replicate the logging code in each layer. Instead, break it out to a separate infrastructure layer which is typically a C# library project.

Furthermore, it's possible that you'll want to follow the same logging technique and strategy across all projects of your company. Then you can have one central infrastructure layer which all projects refer to from within Visual Studio. Then if the developers decide to change the logging, caching or other strategy then it will change across all projects by modifying this one central infrastructure layer.

In this post we'll set up a very simple starting point that lays the foundations for the upcoming parts in the series.

**Starting point**

Open Visual Studio 2012/2013 and create a new Blank Solution, call it CommonInfrastructureDemo. Add a new Console Application to it called MyCompany.ProductConsole. Add 3 new folders to the ProductConsole app called Domains, Services, Repositories.

Normally we'd put these elements into separate C# libraries but it's not the main focus of this series. If you're interested in layered software architecture then you can find a proposed way of achieving it in the DDD series referred to above.

We'll have only one domain: Product. Add a C# class called Product to the Domains folder:



    public class Product
    {
    	public int Id { get; set; }
    	public string Name { get; set; }
    	public int OnStock { get; set; }
    }


Also, add the following repository interface to the Domains folder:



    public interface IProductRepository
    {
    	Product FindBy(int id);
    }


Add the following implementation of the repository to the Repositories folder:



    public class ProductRepository : IProductRepository
    {
    	public Product FindBy(int id)
    	{
    		return (from p in AllProducts() where p.Id == id select p).FirstOrDefault();
    	}

    	private IEnumerable<Product> AllProducts()
    	{
    		return new List<Product>()
    		{
    			new Product(){Id = 1, Name = "Chair", OnStock = 10}
    			, new Product(){Id = 2, Name = "Desk", OnStock = 20}
    			, new Product(){Id = 3, Name = "Cupboard", OnStock = 15}
    		};
    	}
    }


There's nothing really magical here I guess.

Normally the ultimate consumer layer, such as an MVC or Web API layer with a number of controllers, will not talk directly to the repositories. It will only consult the services for any data retrieval. Add the following interface to the Services folder:



    public interface IProductService
    {
    	GetProductResponse GetProduct(GetProductRequest getProductRequest);
    }


…where GetProductResponse and GetProductRequest also reside in the Services folder and look as follows:



    public class GetProductRequest
    {
    	public int Id { get; set; }
    }

    public class GetProductResponse
    {
    	public bool Success { get; set; }
    	public string Exception { get; set; }
    	public Product Product { get; set; }
    }


The implementation of IProductService will need an IProductRepository. We'll let this dependency to be injected into the product service using [constructor injection][4]. Insert the following ProductService class into the Services folder:



    public class ProductService : IProductService
    {
    	private readonly IProductRepository _productRepository;

    	public ProductService(IProductRepository productRepository)
    	{
    		if (productRepository == null) throw new ArgumentNullException("ProductRepository");
    		_productRepository = productRepository;
    	}

    	public GetProductResponse GetProduct(GetProductRequest getProductRequest)
    	{
    		GetProductResponse response = new GetProductResponse();
    		try
    		{
    			Product p = _productRepository.FindBy(getProductRequest.Id);
    			response.Product = p;
    			if (p != null)
    			{
    				response.Success = true;
    			}
    			else
    			{
    				response.Exception = "No such product.";
    			}
    		}
    		catch (Exception ex)
    		{
    			response.Exception = ex.Message;
    		}
    		return response;
    	}
    }


In the implemented GetProduct method we ask the repository to return a Product. If none's found or if the operation throws an exception then we let the Success property be false and populate the exception message accordingly.

We can now tie together all elements in the consumer, i.e. the Main method. We won't employ any Inversion of Control containers to inject our dependencies so that we don't get distracted by technical details. Instead we'll do it the old way, i.e. by poor man's dependency injection. We define the concrete implementations in Main ourselves. However, it's an acceptable solution as this definition happens right at the entry point of the application. If you don't know what I mean I suggest you go through the series on SOLID referred to above with special attention to the letter '[D][5]'.

With the above remarks in mind add the following code into Main in Program.cs:



    static void Main(string[] args)
    {
    	IProductService productService = new ProductService(new ProductRepository());
    	GetProductResponse getProductResponse = productService.GetProduct(new GetProductRequest() { Id = 2 });
    	if (getProductResponse.Success)
    	{
    		Console.WriteLine(string.Concat("Product name: ", getProductResponse.Product.Name));
    	}
    	else
    	{
    		Console.WriteLine(getProductResponse.Exception);
    	}

    	Console.ReadKey();
    }


Run the app and you'll see that the Console prints "Desk" for us. Test the code with Id = 4 and you'll see the appropriate exception message: "No such product".

This finishes the project setup phase. In the [next post][6] we'll start discussing the true purpose of this series with a look at caching in the service layer.

View the list of posts on Architecture and Patterns [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[2]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[3]: http://dotnetcodr.com/2013/09/19/a-model-net-web-service-based-on-domain-driven-design-part-3-the-domain/ "A model .NET web service based on Domain Driven Design Part 3: the Domain"
[4]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[5]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[6]: http://dotnetcodr.com/2014/09/04/externalising-dependencies-with-dependency-injection-in-net-part-2-caching/ "Externalising dependencies with Dependency Injection in .NET part 2: caching"
[7]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
