[Source](http://dotnetcodr.com/2014/09/11/externalising-dependencies-with-dependency-injection-in-net-part-4-logging-part-1/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 4: logging part 1")

# Externalising dependencies with Dependency Injection in .NET part 4: logging part 1

**Introduction**

In [this][1] post of this series we looked at ways to remove caching logic from our ProductService class. In the [previous][2] post we saw how to hide reading from a configuration file. We'll follow in a similar fashion in this post which takes up logging. A lot of the techniques and motivations will be re-used and not explained again.

We'll build upon the demo app we started on so have it ready in Visual Studio.

**Logging**

Logging is a wide area with several questions you need to consider. What needs to be logged? Where should we save the logs? How can the log messages be correlated? We'll definitely not answer those questions here. Instead, we'll look at techniques to call the logging implementation.

I'd like to go through 3 different ways to put logging into your application. For the demo we'll first send the log messages to the Console window for an easy start. After that, we'll propose a way of implementing logging with log4net. The material is too much for a single post so I've decided to divide this topic into 2 posts.

**Starting point**

If you ever had to log something in a professional .NET project then chances are that you used [Log4Net][3]. It's a mature and widely used logging framework. However, it's by far not the only choice. There's e.g. [NLog][4] and probably many more.

If you recall the first solution for the caching problem we put the caching logic directly inside ProductService.GetProduct:



    public GetProductResponse GetProduct(GetProductRequest getProductRequest)
    {
    	GetProductResponse response = new GetProductResponse();
    	try
    	{
    		string storageKey = "GetProductById";
    		ObjectCache cache = MemoryCache.Default;
    		Product p = cache.Contains(storageKey) ? (Product)cache[storageKey] : null;
    		if (p == null)
    		{
    			p = _productRepository.FindBy(getProductRequest.Id);
    			CacheItemPolicy policy = new CacheItemPolicy() { AbsoluteExpiration = DateTime.Now.AddMinutes(5) };
    			cache.Add(storageKey, p, policy);
    		}
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


We then also discussed the disadvantages of such a solution, like diminished testability and flexibility. We could follow a similar solution for the logging with log4net with direct calls from ProductService like this:



    var logger = logSource.GetLogger();
    var loggingEvent = new LoggingEvent(logSource.GetType(), logger.Logger.Repository, logger.Logger.Name, Level.Info, "Code starting", null);
    logger.Logger.Log(loggingEvent);


…which would probably be factored out to a different method but the strong dependency on and tight coupling to log4net won't go away. We could have similar code for NLog and other logging technologies out there. Then some day the project leader says that the selected technology must change because the one they used on a different project is much more powerful. Then the developer tasked with changing the implementation at every possible place in the code won't have a good day out.

**The logging interface**

As we saw in the previous post the key step towards an OOP solution is an abstraction which hides the concrete implementation. This is not always a straightforward step. In textbooks it always looks easy to create these interfaces because they always give you a variety of concrete implementations beforehand. Then you'll know what's common to them and you can put those common methods in an interface. In reality, however, you'll always need to think carefully about the features that can be common across a variety of platforms: what should every decent logging framework be able to do at a minimum? What should a caching technology be able to do? Etc., you can ask similar questions about every dependency in your code that you're trying to factor out. The interface should find the common features and not lean towards one specific implementation so that it becomes as future-proof as possible.

Normally any logging framework should be able to log messages, message levels – e.g. information or fatal – and possibly exceptions. Insert a new folder called Logging to the Infrastructure layer and add the following interface into it:



    public interface ILoggingService
    {
    	void LogInfo(object logSource, string message, Exception exception = null);
    	void LogWarning(object logSource, string message, Exception exception = null);
    	void LogError(object logSource, string message, Exception exception = null);
    	void LogFatal(object logSource, string message, Exception exception = null);
    }


This should be enough for starters: the source where the logging happens, the message and an exception. The level is represented by the method names.

**Logging to the console**

As promised above, we'll take it easy first and send the log messages directly to the console. So add the following implementation to the Logging folder:



    public class ConsoleLoggingService : ILoggingService
    {
    	private ConsoleColor _defaultColor = ConsoleColor.Gray;

    	public void LogInfo(object logSource, string message, Exception exception = null)
    	{
    		Console.ForegroundColor = ConsoleColor.Green;
    		Console.WriteLine(string.Concat("Info from ", logSource.ToString(), ": ", message));
    		PrintException(exception);
    		ResetConsoleColor();
    	}

    	public void LogWarning(object logSource, string message, Exception exception = null)
    	{
    		Console.ForegroundColor = ConsoleColor.Yellow;
    		Console.WriteLine(string.Concat("Warning from ", logSource.ToString(), ": ", message));
    		PrintException(exception);
    		ResetConsoleColor();
    	}

    	public void LogError(object logSource, string message, Exception exception = null)
    	{
    		Console.ForegroundColor = ConsoleColor.DarkMagenta;
    		Console.WriteLine(string.Concat("Error from ", logSource.ToString(), ": ", message));
    		PrintException(exception);
    		ResetConsoleColor();
    	}

    	public void LogFatal(object logSource, string message, Exception exception = null)
    	{
    		Console.ForegroundColor = ConsoleColor.Red;
    		Console.WriteLine(string.Concat("Fatal from ", logSource.ToString(), ": ", message));
    		PrintException(exception);
    		ResetConsoleColor();
    	}

    	private void ResetConsoleColor()
    	{
    		Console.ForegroundColor = _defaultColor;
    	}

    	private void PrintException(Exception exception)
    	{
    		if (exception != null)
    		{
    			Console.WriteLine(string.Concat("Exception logged: ", exception.Message));
    		}
    	}
    }


That should be quite straightforward I believe.

**Solution 1: decorator**

We saw an example of the [Decorator pattern][5] in the post on caching referred to in the intro and we'll build on that. We'll extend the original ProductService implementation to include both caching and logging. Add the following decorator to the Services folder of the Console app layer:



    public class LoggedProductService : IProductService
    {
    	private readonly IProductService _productService;
    	private readonly ILoggingService _loggingService;

    	public LoggedProductService(IProductService productService, ILoggingService loggingService)
    	{
    		if (productService == null) throw new ArgumentNullException("ProductService");
    		if (loggingService == null) throw new ArgumentNullException("LoggingService");
    		_productService = productService;
    		_loggingService = loggingService;
    	}

    	public GetProductResponse GetProduct(GetProductRequest getProductRequest)
    	{
    		GetProductResponse response = new GetProductResponse();
    		_loggingService.LogInfo(this, "Starting GetProduct method");
    		try
    		{
    			response = _productService.GetProduct(getProductRequest);
    			if (response.Success)
    			{
    				_loggingService.LogInfo(this, "GetProduct success!!!");
    			}
    			else
    			{
    				_loggingService.LogError(this, "GetProduct failure...", new Exception(response.Exception));
    			}
    		}
    		catch (Exception ex)
    		{
    			response.Exception = ex.Message;
    			_loggingService.LogError(this, "Exception in GetProduct!!!", ex);
    		}
    		return response;
    	}
    }


You'll recall the structure from the Caching decorator. We hide both the product and logging service behind interfaces. We delegate the product retrieval to the product service and logging to the logging service. So if you'd like to add logging to the original ProductService class then you can have the following code in Main:



    IProductService productService = new ProductService(new ProductRepository());
    IProductService loggedProductService = new LoggedProductService(productService, new ConsoleLoggingService());
    GetProductResponse getProductResponse = loggedProductService.GetProduct(new GetProductRequest() { Id = 2 });
    if (getProductResponse.Success)
    {
    	Console.WriteLine(string.Concat("Product name: ", getProductResponse.Product.Name));
    }
    else
    {
    	Console.WriteLine(getProductResponse.Exception);
    }


If you run this code then you'll see an output similar to this:

![Info messages from logging service][6]

If you run the code with a non-existent product ID then you'll see an exception as well:

![Exception from logging service][7]

If you'd then like to add both caching and logging to the plain ProductService class then you can have the following test code:



    IProductService productService = new ProductService(new ProductRepository());
    IProductService cachedProductService = new CachedProductService(productService, new SystemRuntimeCacheStorage());
    IProductService loggedCachedProductService = new LoggedProductService(cachedProductService, new ConsoleLoggingService());
    GetProductResponse getProductResponse = loggedCachedProductService.GetProduct(new GetProductRequest() { Id = 2 });
    if (getProductResponse.Success)
    {
    	Console.WriteLine(string.Concat("Product name: ", getProductResponse.Product.Name));
    }
    else
    {
    	Console.WriteLine(getProductResponse.Exception);
    }

    getProductResponse = loggedCachedProductService.GetProduct(new GetProductRequest() { Id = 2 });

    Console.ReadKey();


Notice how we build up the compound decorator loggedCachedProductService from productService and cachedProductService in the beginning of the code. You can step through the code and you'll see how we print the log messages, check the cache and retrieve the product from ProductService if necessary.

**Solution 2: ambient context**

I think the above solution with dependency injection, interfaces and decorators follows SOLID principles quite well. We can build upon the ProductService class using the decorators, pass different implementations of the abstractions and test each component independently.

As discussed in the post on [Interception][8] – check the section called Interception using the Decorator pattern – this solution can have some practical limitations, such as writing a large number of tedious code. While you may not want to do caching in every single layer, logging is different. You may want to log from just about any layer of the application: UI, services, controllers, repositories. So you may need to write a LoggedController, LoggedRepository, LoggedService etc. for any component where you want to introduce logging.

In the post on [patterns in dependency injection][9] we discussed a technique called **ambient context**. I suggest you read that section if you don't understand the term otherwise you may not understand the purpose of the code below. Also, you'll find its advantages and disadvantages there as well.

Insert the following class into the Logging folder of the infrastructure project:



    public abstract class LogProviderContext
    {
    	private static readonly string _nameDataSlot = "LogProvider";

    	public static ILoggingService Current
    	{
    		get
    		{
    			ILoggingService logProviderContext = Thread.GetData(Thread.GetNamedDataSlot(_nameDataSlot)) as ILoggingService;
    			if (logProviderContext == null)
    			{
    				logProviderContext = LogProviderContext.DefaultLogProviderContext;
    				Thread.SetData(Thread.GetNamedDataSlot(_nameDataSlot), logProviderContext);
    			}
    			return logProviderContext;
    		}
    		set
    		{
    			Thread.SetData(Thread.GetNamedDataSlot(_nameDataSlot), value);
    		}
    	}

    	public static ILoggingService DefaultLogProviderContext = new ConsoleLoggingService();
    }


You can call this code from any other layer which has a reference to the infrastructure layer. E.g. you can have the following code directly in ProductService.GetProducts:



    LogProviderContext.Current.LogInfo(this, "Log message from the contextual log provider");


**Solution 3: injecting logging into ProductService**

Another solution which lies somewhere between ambient context and the decorator is having ProductService depend upon an ILoggingService:



    public ProductService(IProductRepository productRepository, ILoggingService loggingService)


You can then use the injected ILoggingService to log you messages. We saw a similar example in the post on caching and also in the post on Interception referred to above. Read the section called Dependency injection in the post on Interception to read more why this approach might be good or bad.

However, if you're faced with such a constructor and still want to test ProductService in isolation you can use a Null Object version of ILoggingService. Add the following class to the Logging folder:



    public class NoLogService : ILoggingService
    {
    	public void LogInfo(object logSource, string message, Exception exception = null)
    	{}

    	public void LogWarning(object logSource, string message, Exception exception = null)
    	{}

    	public void LogError(object logSource, string message, Exception exception = null)
    	{}

    	public void LogFatal(object logSource, string message, Exception exception = null)
    	{}
    }


In the [next post][10] we'll continue with logging to show a more realistic implementation of ILoggingService with log4net.

View the list of posts on Architecture and Patterns [here][11].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/04/externalising-dependencies-with-dependency-injection-in-net-part-2-caching/ "Externalising dependencies with Dependency Injection in .NET part 2: caching"
[2]: http://dotnetcodr.com/2014/09/08/externalising-dependencies-with-dependency-injection-in-net-part-3-configuration/ "Externalising dependencies with Dependency Injection in .NET part 3: configuration"
[3]: http://logging.apache.org/log4net/ "Log4net on apache"
[4]: http://nlog-project.org/ "NLog homepage"
[5]: http://dotnetcodr.com/2013/05/13/design-patterns-and-practices-in-net-the-decorator-design-pattern/ "Design patterns and practices in .NET: the Decorator design pattern"
[6]: http://dotnetcodr.files.wordpress.com/2014/08/info-messages-from-logging-service.png?w=300&h=32
[7]: http://dotnetcodr.files.wordpress.com/2014/08/exception-from-logging-service.png?w=300&h=31
[8]: http://dotnetcodr.com/2013/09/05/solid-design-principles-in-net-the-dependency-inversion-principle-part-4-interception-and-conclusions/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 4, Interception and conclusions"
[9]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[10]: http://dotnetcodr.com/2014/09/15/externalising-dependencies-with-dependency-injection-in-net-part-5-logging-with-log4net/ "Externalising dependencies with Dependency Injection in .NET part 5: logging with log4net"
[11]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
