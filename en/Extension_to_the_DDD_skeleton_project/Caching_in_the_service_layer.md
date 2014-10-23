[Source](http://dotnetcodr.com/2014/03/24/extension-to-the-ddd-skeleton-project-caching-in-the-service-layer/ "Permalink to Extension to the DDD skeleton project: caching in the service layer")

# Extension to the DDD skeleton project: caching in the service layer

**Introduction**

This post is a direct continuation of the [DDD skeleton project][1]. In this extension we'll investigate how to incorporate caching into the solution.

Caching can mean several things in an application. It can be introduced at different levels: you can cache database call results, [MVC views][2], service call results, etc. You can save just about any object in one of the cache engines built into .NET.

In this post we'll concentrate on caching in the Service layer. We'll try to solve the requirement to cache the result of the following service call of ICustomerService for a period of time:



    GetCustomersResponse GetAllCustomers();


A naive but quick solution would be to cache the results directly in the implementing CustomerService class:



    public GetCustomersResponse GetAllCustomers()
    {
            if (cacheEngine.IncludesObject<GetCustomersResponse>())
            {
                  return cacheEngine.RetrieveObject<GetCustomersResponse>();
            }
            else
            {
    	     GetCustomersResponse getCustomersResponse = new GetCustomersResponse();
    	     IEnumerable<Customer> allCustomers = null;

    	     try
    	     {
    		    allCustomers = _customerRepository.FindAll();
    		    getCustomersResponse.Customers = allCustomers.ConvertToViewModels();
                        cacheEngine.SaveObject<GetCustomersResponse>(getCustomersResponse);
    	     }
    	     catch (Exception ex)
    	     {
    		    getCustomersResponse.Exception = ex;
    	     }
    	     return getCustomersResponse;
           }
    }


There are a couple of problems with this solution:

* The method violates the [Single Responsibility Principle][3]: it doesn't only look up all customers but caches them as well
* The method doesn't indicate to its consumers what's going in its method body. The signature states that it will retrieve all customers but there's nothing about caching. If it returns stale data then the programmer must inspect the method body to find the reason – and will get a surprise: hey, you never told me about caching!
* It's difficult to test the service call in isolation: the cache is empty on the first test run but then as the method returns the cached data you cannot test the actual customer retrieval easily. You'd need to clear the cache before every test run. So if the test fails then you won't know really why it fails: was it because the customer retrieval threw an exception or because the caching engine misbehaved?

Therefore a more thorough solution requires a different way of thinking.

If you are familiar with SOLID and especially the letter '[D][4]' then you'll know that all this is pointing towards injecting the caching strategy into CustomerService without having to add the caching operations directly within the GetAllCustomers() method body.

There are different ways to solve this problem that we discussed in [this][5] post on Interception with their pros and cons:

* The Decorator pattern
* AOP
* Dynamic interception with a DI container

In this post we'll go for the "purist" Decorator pattern: it is very object oriented and is probably the easiest to follow for a newcomer to this topic. I'm not a fan of AOP – for reasons read the reference above – and dynamic interception is too technical at this level. I don't want to make a very technical post concentrating on a specific DI container. We'll learn a lot of new things with the Decorator pattern. In case you're not at all familiar with the decorator pattern make sure to check out [this][6] reference. We'll also see the [Adapter pattern][7] in action to hide the concrete implementation of the caching engine.

The solution we're going to go through can be applied to a wide range of cross-cutting concerns, such as logging. I'll present a decorator that adds caching to a service. By the same token you can build a decorator for logging as well. You can then decorate the service class with a logger AND a caching mechanism as well.

**Infrastructure**

Caching is an example of a cross-cutting concern. Caching can be introduced in many places within a solution and the same caching strategy can be re-used across several applications. This is pointing us towards the Infrastructure layer of the DDD skeleton project.

Also, the technology you choose for caching can vary: use one of the built-in objects in .NET, such as the System.Runtime.Cache, you can use a third part component or write your own custom solution. The point is that the caching engine may change and you'll want to be able to easily change the concrete implementation. So it sounds like we have a variable dependency that may change over time, and therefore it needs to be hidden behind an abstraction.

Open the Infrastructure.Common layer of the DDD solution and add a new folder called Caching. Add the following interface:



    public interface ICacheStorage
    {
    	void Remove(string key);
    	void Store(string key, object data);
            void Store(string key, object data, TimeSpan slidingExpiration);
    	void Store(string key, object data, DateTime absoluteExpiration, TimeSpan slidingExpiration);
    	T Retrieve<T>(string key);
    }


All cache storage mechanisms will need to implement this interface. We'll go for the highly efficient, general-purpose caching engine of [System.Runtime.Caching][8] in the first implementation. Add the following class to the Caching folder:



    public class SystemRuntimeCacheStorage : ICacheStorage
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

            public void Store(string key, object data, TimeSpan slidingExpiration)
    	{
    		ObjectCache cache = MemoryCache.Default;
    		var policy = new CacheItemPolicy
    		{
    			SlidingExpiration = slidingExpiration
    		};

    		if (cache.Contains(key))
    		{
    			cache.Remove(key);
    		}

    		cache.Add(key, data, policy);
    	}
    }


The code may not compile at first as ObjectCache is located in an unreferenced library. Add a reference to the System.Runtime.Caching dll in the Infrastructure project.

**Decorator pattern in the ApplicationServices layer**

OK, now we have the caching elements in place using the Adapter pattern. Next we want to enrich the existing CustomerService implementation of the ICustomerService interface to include caching. In the following stub we'll delegate most of the methods to the inner customer service, which will be our "normal" CustomerService class – we'll look at the GetAllCustomers() method in a sec. Insert the following class into the Implementations folder:



    public class EnrichedCustomerService : ICustomerService
    {
    	private readonly ICustomerService _innerCustomerService;
    	private readonly ICacheStorage _cacheStorage;

    	public EnrichedCustomerService(ICustomerService innerCustomerService, ICacheStorage cacheStorage)
    	{
    		if (innerCustomerService == null) throw new ArgumentNullException("CustomerService");
    		if (cacheStorage == null) throw new ArgumentNullException("CacheStorage");
    		_innerCustomerService = innerCustomerService;
    		_cacheStorage = cacheStorage;
    	}

    	public GetCustomerResponse GetCustomer(GetCustomerRequest getCustomerRequest)
    	{
    		return _innerCustomerService.GetCustomer(getCustomerRequest);
    	}

    	public GetCustomersResponse GetAllCustomers()
    	{
    		throw new NotImplementedException();
    	}

    	public InsertCustomerResponse InsertCustomer(InsertCustomerRequest insertCustomerRequest)
    	{
    		return _innerCustomerService.InsertCustomer(insertCustomerRequest);
    	}

    	public UpdateCustomerResponse UpdateCustomer(UpdateCustomerRequest updateCustomerRequest)
    	{
    		return _innerCustomerService.UpdateCustomer(updateCustomerRequest);
    	}

    	public DeleteCustomerResponse DeleteCustomer(DeleteCustomerRequest deleteCustomerRequest)
    	{
    		return _innerCustomerService.DeleteCustomer(deleteCustomerRequest);
    	}
    }


Again, if you don't understand the structure of this class check out the post on the Decorator pattern. Here's the implementation of the GetAllCustomers() method:



    public GetCustomersResponse GetAllCustomers()
    {
    	string key = "GetAllCustomers";
    	GetCustomersResponse response = _cacheStorage.Retrieve<GetCustomersResponse>(key);
    	if (response == null)
    	{
    		response = _innerCustomerService.GetAllCustomers();
    		_cacheStorage.Store(key, response, TimeSpan.FromMinutes(1));
    	}
    	return response;
    }


We check in the cache if a GetCustomersResponse object is present and return it. Otherwise we ask the inner customer service to fetch all customers and put the result into the cache for 1 minute – we'll come back to this point at the end of the post.

We are done with the decorated CustomerService class. Now we'd like to make sure that the CustomerController receives the EnrichedCustomerService instead of CustomerService.

**StructureMap**

Recall that we're using StructureMap as our DI container. At present it follows the convention that whenever it sees an interface dependency whose name starts with "I" it will look for a concrete implementation with the same name without the "I": IProductService == ProcuctService, ICustomerRepository == CustomerRepository. We also saw examples of specifying a concrete class that StructureMap will insert:



    x.For<IUnitOfWork>().Use<InMemoryUnitOfWork>();
    x.For<IObjectContextFactory>().Use<LazySingletonObjectContextFactory>();


So we should be able to add the following code, right?



    x.For<ICustomerService>().Use<EnhancedCustomerService>();


Keep in mind that EnhancedCustomerService also has a dependency on ICustomerService so this code would inject another EnhancedCustomerService into EnhancedCustomerService which is not what we want. The goal is the following:

* The CustomerController class should get EnhancedCustomerService from StructureMap
* EnhancedCustomerService should get CustomerService from StructureMap

In fact this is such a common problem that it has been solved multiple times in StructureMap. All decent DI containers provide a solution for the Decorator pattern. It can be solved in at least 3 ways in StructureMap:

* Decorating with instance references
* Decorating with named instances
* Decorating with delegates

This is not a technical post about StructureMap so I won't go into any detail. We'll go for the first option. Locate the IoC.cs class in the WebService layer and add the the following piece of code the the ObjectFactory.Initialize block:



    x.For<ICacheStorage>().Use<SystemRuntimeCacheStorage>();
    var customerService = x.For<ICustomerService>().Use<CustomerService>();
    x.For<ICustomerService>().Use<EnrichedCustomerService>().Ctor<ICustomerService>().Is(customerService);


The first row is quite clear: we want to use the SystemRuntimeCacheStorage implementation of the ICacheStorage interface. The second row indicates that we want to use CustomerService where ICustomerService is used and retain a reference to the object returned by the Use method: a SmartInstance of T which is a StructureMap object. In the third row we modify this statement by saying that for ICustomerService we want an EnrichedCustomerService object and inject the retained customerService in its constructor. The other elements in the EnrichedCustomerService constructor will be resolved automatically by StructureMap. This looks a bit cryptic at first but that's how it's done, period.

Open the Properties window of the WebService layer, select the Web tag, click "Specific Page" and enter "customers" as the start-up page. Set a couple of breakpoints:

* One withing the Get() method of CustomersController
* Another one within GetAllCustomers() of EnrichedCustomerService

Start the web service and the start up URL in the browser should be something like <http://localhost:9985/customers>. The port number will probably be different in your case. Code execution should stop at the first breakpoint in the Get() method. Inspect the type of the injected ICustomerService in CustomersController.cs, it should be of type EnrichedCustomerService. Let the code continue and execution will stop again at GetAllCustomers in EnrichedCustomerService. Inspect the type of the inner customer service in that class: it should be of CustomerService. From here on I encourage you to step through the code with F11 to see how caching is used. Then refresh the page in the browser and you'll see that the GetCustomersResponse is fetched from the cache instead of asking the CustomerService instance.

That's about it. We have solved the caching problem in an object oriented, testable and SOLID way.

**Further enhancement**

There's still room for improvement in our cache solution. Right now the cache duration is hard coded here:



    _cacheStorage.Store(key, response, TimeSpan.FromMinutes(1));


It would be better with a more configurable solution where you can change the duration without coming back to this class every time. Imagine that you have caching in hundreds of classes – changing the duration in every single one will not make a happy programmer.

There are several ways to solve this problem. [The next post][9] will present a possible solution.

View the list of posts on Architecture and Patterns [here][10].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[2]: http://dotnetcodr.com/2013/02/07/caching-infrastructure-in-mvc4-with-c-caching-controller-actions/ "Caching infrastructure in MVC4 with C#: caching controller actions"
[3]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[4]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[5]: http://dotnetcodr.com/2013/09/05/solid-design-principles-in-net-the-dependency-inversion-principle-part-4-interception-and-conclusions/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 4, Interception and conclusions"
[6]: http://dotnetcodr.com/2013/05/13/design-patterns-and-practices-in-net-the-decorator-design-pattern/ "Design patterns and practices in .NET: the Decorator design pattern"
[7]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[8]: http://msdn.microsoft.com/en-us/library/system.runtime.caching(v=vs.110).aspx "System.Runtime.Caching on MSDN"
[9]: http://dotnetcodr.com/2014/03/27/extension-to-the-ddd-skeleton-project-using-an-external-settings-source/ "Extension to the DDD skeleton project: using an external settings source"
[10]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
