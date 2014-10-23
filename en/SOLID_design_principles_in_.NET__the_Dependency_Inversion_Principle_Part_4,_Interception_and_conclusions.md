[Source](http://dotnetcodr.com/2013/09/05/solid-design-principles-in-net-the-dependency-inversion-principle-part-4-interception-and-conclusions/ "Permalink to SOLID design principles in .NET: the Dependency Inversion Principle Part 4, Interception and conclusions")

# SOLID design principles in .NET: the Dependency Inversion Principle Part 4, Interception and conclusions

**Introduction**

I briefly mentioned the concept of Interception in the [this][1] post. It is a technique that can help you implement cross-cutting concerns such as logging, tracing, caching and other similar activities. Cross-cutting concerns include actions that are not strictly related to a specific domain but can potentially be called from many different objects. E.g. you may want to cache certain method results pretty much anywhere in your application so potentially you'll need an ICacheService dependency in many places. In the post mentioned above I went through a possible DI pattern – ambient context – to implement such actions with all its pitfalls.

If you're completely new to these concepts make sure you read through all the previous posts on DI in this series. I won't repeat what was already explained before.

The idea behind Interception is quite simple. When a consumer calls a service you may wish to intercept that call and execute some action before and/or after the actual service is invoked.

It happens occasionally that I do the shopping on my way home from work. This is a real life example of interception: the true purpose of my action is to get home but I "intercept" that action with another one, namely buying some food. I can also do the shopping when I pick up my daughter from the kindergarten or when I want to go for a walk. So I intercept the main actions PickUpFromKindergarten() and GoForAWalk() with the shopping action because it is convenient to do so. The Shopping action can be injected into several other actions so in this case it may be considered as a Cross-Cutting Concern. Of course the shopping activity can be performed in itself as the main action, just like you can call a CacheService directly to cache something, in which case it too can be considered as the main action.

**The problem**

Say you have a service that looks up an object with an ID:



    public interface IProductService
    {
    	Product GetProduct(int productId);
    }




    public class DefaultProductService : IProductService
    {
    	public Product GetProduct(int productId)
    	{
    		return new Product();
    	}
    }


Say you don't want to look up this product every time so you decide to cache the result for 10 minutes.

**Possible solutions**

_Total lack of DI_

The first "solution" is to directly implement caching within the GetProduct method. Here I'm using the ObjectCache object located in the System.Runtime.Caching namespace:



    public Product GetProduct(int productId)
    {
    	ObjectCache cache = MemoryCache.Default;
    	string key = "product|" + productId;
    	Product p = null;
    	if (cache.Contains(key))
    	{
    		p = (Product)cache[key];
    	}
    	else
    	{
    		p = new Product();
    		CacheItemPolicy policy = new CacheItemPolicy();
    		DateTimeOffset dof = DateTimeOffset.Now.AddMinutes(10);
    		policy.AbsoluteExpiration = dof;
    		cache.Add(key, p, policy);
    	}
    	return p;
    }


We check the cache using the cache key and retrieve the Product object if it's available. Otherwise we simulate a database lookup and put the Product object in the cache with an absolute expiration of 10 minutes.

If you've read through the posts on DI and SOLID then you should know that this type of code has numerous pitfalls:

* It is tightly coupled to the ObjectCache class
* You cannot easily specify a different caching strategy – if you want to increase the caching time to 20 minutes then you'll have to come back here and modify the method
* The method signature does not tell anything to the caller about caching, so it violates the idea of an Intention Revealing Interface mentioned before
* Therefore the caller will need to intimately know the internals of the GetProduct method
* The method is difficult to test as it's impossible to abstract away the caching logic. The test result will depend on the caching mechanism within the code so it will be inconclusive

Nevertheless you have probably encountered this style of coding quite often. There is nothing stopping you from writing code like that. It's quick, it's dirty, but it certainly works.

As an attempt to remedy the situation you can factor out the caching logic to a service:



    public class SystemRuntimeCacheStorage
    {
    	public void Remove(string key)
    	{
    		ObjectCache cache = MemoryCache.Default;
    		cache.Remove(key);
    	}

    	public void Store(string key, object data)
    	{
    		ObjectCache cache = MemoryCache.Default;
    		cache.Add(key, data, null);
    	}

    	public void Store(string key, object data, DateTime absoluteExpiration, TimeSpan slidingExpiration)
    	{
    		ObjectCache cache = MemoryCache.Default;
    		var policy = new CacheItemPolicy
    		{
    			AbsoluteExpiration = absoluteExpiration,
    			SlidingExpiration = slidingExpiration
    		};

    		if (cache.Contains(key))
    		{
    			cache.Remove(key);
    		}
    		cache.Add(key, data, policy);
    	}

    	public T Retrieve<T>(string key)
    	{
    		ObjectCache cache = MemoryCache.Default;
    		return cache.Contains(key) ? (T)cache[key] : default(T);
    	}
    }


This is a generic class to store, remove and retrieve objects of type T. As the next step you want to call this service from the DefaultProductService class as follows:



    public class DefaultProductService : IProductService
    {
    	private SystemRuntimeCacheStorage _cacheStorage;

    	public DefaultProductService()
    	{
    		_cacheStorage = new SystemRuntimeCacheStorage();
    	}

    	public Product GetProduct(int productId)
    	{
    		string key = "product|" + productId;
    		Product p = _cacheStorage.Retrieve<Product>(key);
    		if (p == null)
    		{
            		p = new Product();
    			_cacheStorage.Store(key, p);
    		}
    		return p;
    	}
    }


We've seen a similar example in the previous post where the consuming class constructs its own dependency. This "solution" has the same errors as the one above – it's only the stacktrace that has changed. You'll get the same faulty design with a factory as well. However, this was a step towards a loosely coupled solution.

_Dependency injection_

As you know by now abstractions are the way to go to reach loose coupling. We can factor out the caching logic into an interface:



    public interface ICacheStorage
    {
    	void Remove(string key);
    	void Store(string key, object data);
    	void Store(string key, object data, DateTime absoluteExpiration, TimeSpan slidingExpiration);
    	T Retrieve<T>(string key);
    }


Then using constructor injection we can inject the caching mechanism as follows:



    public class DefaultProductService : IProductService
    {
    	private readonly ICacheStorage _cacheStorage;

    	public DefaultProductService(ICacheStorage cacheStorage)
    	{
    		_cacheStorage = cacheStorage;
    	}

    	public Product GetProduct(int productId)
    	{
    		string key = "product|" + productId;
    		Product p = _cacheStorage.Retrieve<Product>(key);
    		if (p == null)
    		{
    			p = new Product();
    			_cacheStorage.Store(key, p);
    		}
    		return p;
    	}
    }


Now we can inject any type of concrete caching solution which implements the ICacheStorage interface. As far as tests are concerned you can inject an empty caching solution using the [Null object pattern][2] so that the test can concentrate on the true logic of the GetProduct method.

This is certainly a loosely coupled solution but you may need to inject similar interfaces to a potentially large number of services:



    public ProductService(ICacheStorage cacheStorage, ILoggingService loggingService, IPerformanceService performanceService)
    public CustomerService(ICacheStorage cacheStorage, ILoggingService loggingService, IPerformanceService performanceService)
    public OrderService(ICacheStorage cacheStorage, ILoggingService loggingService, IPerformanceService performanceService)


These services will permeate your class structure. Also, you may create a base service for all services like this:



    public ServiceBase(ICacheStorage cacheStorage, ILoggingService loggingService, IPerformanceService performanceService)


If all services must inherit this base class then they will start their lives with 3 abstract dependencies that they may not even need. Also, these dependencies don't represent the true purpose of the services, they are only "sidekicks".

_Ambient context_

For a discussion on this type of DI and when and why (not) to use it consult [this][1] post.

_Interception using the Decorator pattern_

The [Decorator][3] design pattern can be used as a do-it-yourself interception. The product service class can be reduced to its true purpose:



    public class DefaultProductService : IProductService
    	{
    		public Product GetProduct(int productId)
    		{
    			return new Product();
    		}
    	}


A cached product service might look as follows:



    public class CachedProductService : IProductService
    {
    	private readonly IProductService _innerProductService;
    	private readonly ICacheStorage _cacheStorage;

    	public CachedProductService(IProductService innerProductService, ICacheStorage cacheStorage)
    	{
    		if (innerProductService == null) throw new ArgumentNullException("ProductService");
    		if (cacheStorage == null) throw new ArgumentNullException("CacheStorage");
    		_cacheStorage = cacheStorage;
    		_innerProductService = innerProductService;
    	}

    	public Product GetProduct(int productId)
    	{
    		string key = "Product|" + productId;
    		Product p = _cacheStorage.Retrieve<Product>(key);
    		if (p == null)
    		{
    			p = _innerProductService.GetProduct(productId);
    			_cacheStorage.Store(key, p);
    		}

    		return p;
    	}
    }


The cached product service itself implements IProductService and accepts another IProductService in its constructor. The injected product service will be used to retrieve the product in case the injected cache service cannot find it.

The consumer can actively use the cached implementation of the IProductService in place of the DefaultProductService class to deliberately call for caching. Here the call to retrieve a product is intercepted by caching. The cached service can concentrate on its task using the injected ICacheStorage object and delegates the actual product retrieval to the injected IProductService class.

You can imagine that it's possible to write a logging decorator, a performance decorator etc., i.e. a decorator for any type of cross-cutting concern. You can even decorate the decorator to include logging AND caching. Here you see several applications of SOLID. You keep the product service clean so that it adheres to the Single Responsibility Principle. You extend its functionality through the cached product service decorator which is an application of the Open-Closed principle. And obviously injecting the dependencies through abstractions is an example of the Dependency Inversion Principle.

The Decorator is a well-tested pattern to implement interception in a highly flexible object-oriented way. You can implement a lot of decorators for different purposes and you will adhere to SOLID pretty well. However, imagine that in a large business application with hundreds of domains and hundreds of services you may potentially have to write hundreds of decorators. As each decorator executes one thing only to adhere to SRP you may need to implement 3-4 decorators for each service.

That's a lot of code to write… This is actually a practical limitation of solely using this pattern in a large application to achieve interception: it's extremely repetitive and time consuming.

_Aspect oriented programming (AOP)_

The idea behind AOP is strongly related to attributes in .NET. An example of an attribute in .NET is the following:



    [PrincipalPermission(SecurityAction.Demand, Role = "Administrator")]
    protected void Page_Load(object sender, EventArgs e)
    {
    }


This is also an example of interception. The PrincipalPermission attribute checks the role of the current principal before the decorated method can continue. In this case the ASP.NET page won't load unless the principal has the Administrator role. I.e. the call to Page_Load is intercepted by this Security attribute.

The decorator pattern we saw above is an example of imperative coding. The attributes are an example of declarative interception. Applying attributes to declare aspects is a common technique in AOP. Imagine that instead of writing all those decorators by hand you could simply decorate your objects as follows:



    [Cached]
    [Logged]
    [PerformanceChecked]
    public class DefaultProductService : IProductService
    {
    	public Product GetProduct(int productId)
    	{
    		return new Product();
    	}
    }


It looks attractive, right? Well, let's see.

The PrincipalPermission attribute is special as it's built into the .NET base class library (BCL) along with some other attributes. .NET understands this attribute and knows how to act upon it. However, there are no built-in attributes for caching and other cross-cutting concerns. So you'd need to implement your own aspects. That's not too difficult; your custom attribute will need to derive from the System.Attribute base class. You can then decorate your classes with your custom attribute but .NET won't understand how to act upon it. The code behind your implemented attribute won't run just like that.

There are commercial products, like [PostSharp][4], that enable you to write attributes that are acted upon. PostSharp carries out its job by modifying your code in the post-compilation step. The "normal" compilation runs first, e.g. by csc.exe and then PostSharp adds its post-compilation step by taking the code behind your custom attribute(s) and injecting it into the code compiled by csc.exe in the correct places.

This sounds enticing. At least it sounded to me like heaven when we tested AOP with PostSharp: we wanted to measure the execution time and save several values about the caller of some very important methods of a service. So we implemented our custom attributes and very extremely proud of ourselves. Well, until someone else on the team started using PostSharp in his own assembly. When I referenced his project in mine I suddenly kept getting these funny notices that I have to activate my PostSharp account… So what's wrong with those aspects?

* The code you write will be different from what will be executed as new code will be injected into the compiled one in the post-compilation step. This may be tricky in a debugging session
* The vendors will be happy to provide helper tools for debugging which may or may not be included in the base price and push you towards an anti-pattern where you depend on certain external vendors – also a form of tight coupling
* All attributes must have default parameterless constructors – it's not easy to consume dependencies from within an attribute. Your best bet is using ambient context – or abandon DI and go with default implementations of the dependencies
* It can be difficult to fine-grain the rules when to apply an aspect. You may want to go with a convention-based applicability such as "apply the aspect on all objects whose name ends with '_log'"
* The aspect itself is not an abstraction; it's not straightforward to inject different implementations of an aspect – therefore if you decide to go with the System.Runtime.Cache in your attribute implementation then you cannot change your mind afterwards. You cannot implement a factory or any other mechanism to inject a certain aspect in place of some abstract aspect as there's no such thing

This last point is probably the most serious one. It pulls you towards the dreaded tight-coupling scenario where you cannot easily redistribute a class or a module due to the concrete dependency introduced by an aspect. If you consume such an external library, like in the example I gave you, then you're stuck with one implementation – and you better make sure you have access to the correct credentials to use that unwanted dependency…

_Dynamic interception with a DI container_

We briefly mentioned DI containers, or IoC containers in this series. You may be familiar with some of them, such as StructureMap and CastleWindsor. I won't get into any details regarding those tools. There are numerous tutorials available on the net to get you started. As you get more and more exposed to SOLID in your projects then eventually you'll most likely become familiar with at least one of them.

Dynamic interception makes use of the ability of .NET to dynamically emit types. Some DI containers enable you to automate the generation of decorators to be emitted straight into a running process.

This approach is fully object-oriented and helps you avoid the shortcomings of AOP attributes listed above. You can register your own decorators with the IoC container, you don't need to rely on a default one.

If you are new to DI containers then make sure you understand the basics before you go down the dynamic interception route. I won't show you any code here on how to implement this technique as it depends on the type of IoC container of your choosing. The key steps as far as CastleWindsor is concerned are as follows:

* Implement the IInterceptor interface for your decorator
* Register the interceptor with the container
* Activate the interceptor by implementing the IModelInterceptorsSelector interface – this is the step where you declare when and where the interceptors will be invoked
* Register the class that implements the IModelInterceptorsSelector interface with the container

Carefully following these steps will ensure that you can implement dynamic interception without the need for attributes. Note that not all IoC containers come with the feature of dynamic interception.

**Conclusions**

In this mini-series on DI within the series about SOLID I hope to have explained the basics of the Dependency Inversion Principle. This last constituent of SOLID is probably the one that has caused the most controversy and misunderstanding of the 5. Ask 10 developers on the purposes of DIP and you'll get 11 different answers. You may absolutely have come across ideas in these posts that you disagree with – feel free to comment in that case.

However, I think there are several myths and misunderstandings about DI and DIP that were successfully dismissed:

* DI is the same as IoC containers: no, IoC containers can automate DI but you can by any means apply DI in your code without a tool like that
* DI can be solved with factories: look at the post on DI anti-patterns and you'll laugh at this idea
* DI requires an IoC containers: see the first point, this is absolutely false
* DI is only necessary if you want to enable unit testing: no, DI has several advantages as we saw, effective unit testing being only one of them
* Interception is best done with AOP: no, see above
* Using an IoC container will automatically result in DI: no, you have to prepare your code according to the DI patterns otherwise an IoC container will have nothing to inject

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[2]: http://dotnetcodr.com/2013/05/06/design-patterns-and-practices-in-net-the-null-object-pattern/ "Design patterns and practices in .NET: the Null Object pattern"
[3]: http://dotnetcodr.com/2013/05/13/design-patterns-and-practices-in-net-the-decorator-design-pattern/ "Design patterns and practices in .NET: the Decorator design pattern"
[4]: http://www.postsharp.net/ "PostSharp"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
