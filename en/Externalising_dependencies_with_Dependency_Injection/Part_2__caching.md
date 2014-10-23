[Source](http://dotnetcodr.com/2014/09/04/externalising-dependencies-with-dependency-injection-in-net-part-2-caching/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 2: caching")

# Externalising dependencies with Dependency Injection in .NET part 2: caching

**Introduction**

In the [previous][1] post we set up the startup project for our real discussion. In this post we'll take a look at caching as a dependency.

**Adding caching**

Our glorious application is a success and grows by an incredible rate every month. It starts getting slow and the users keep complaining about the performance. The team decides to add caching to the service layer: cache the result of the "product by id" query to the repository for some time. The team decides to make this quick and modifies the GetProduct method as follows:



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


You'll need to set a reference to the System.Runtime.Caching dll to find the [ObjectCache][2] object.

So we cache the product search result for 5 minutes. In order to see this in action add a second call to the the product service just above Console.ReadKey() in Main:



    getProductResponse = productService.GetProduct(new GetProductRequest() { Id = 2 });


I encourage you to step through the code with F11. You'll see that the first productService.GetProduct retrieves the product from the repository and adds the item to the cache. The second call fetches the product from the cache.

**What's wrong with this?**

The code works, so where's the problem?

_Testability_

The method is difficult to test in isolation because of the dependency on the ObjectCache class. The actual purpose of the method is to get a product by ID. However, if the code also caches the result then it's difficult to test the result from GetProduct. If you want to get any reliable result from the test that tests the behaviour of this method you'll need to somehow flush the cache before every test so that you know that got a fresh result, not a cached one. Otherwise if the test fails, then why did it fail? Was it a genuine failure, meaning that the product was not retrieved? Or was it because the caching mechanism failed? It's the wrong approach making the test outcome dependent on such a dependency.

_Flexibility_

With this implementation we're stuck with the ObjectContext as our caching solution. What if we want to change over to a different one, such as Memcached or HttpContext? In that case we'd need to go in and manually replace the ObjectCache solution to a new one. Even worse, let's say all your service classes use ObjectCache for caching and you want to make the transition to another caching solution for all of them. You probably see how tedious, time consuming and error-prone this could be.

_Single responsibility_

The method also violates the [Single Responsibility Principle][3] as it performs caching in its method body. Strictly speaking it should not be doing this as it then introduces a hidden side effect. The caller of the method doesn't know that the result is cached and may be surprised to see an outdated product if it's e.g. updated directly in the database.

The solution will involve 2 patterns:

* [Adapter][4]: to factor out the caching strategy behind an abstraction. We used caching to demonstrate the Adapter pattern and much of it will be re-used here.
* [Decorator][5]: to completely break out the caching code from ProductService and put it in an encasing class

**Step 1: hide caching**

This is where we need to consider the operations expected from any caching engine. We should be able to store, retrieve and delete objects using any cache engine, right? Those operations should be common to all caching implementations.

As caching is a cross-cutting concern we'll start building our infrastructure layer. Add a new C# class library called MyCompany.Infrastructure.Common to the solution and insert a new folder called Caching. Add the following interface to the Caching folder:



    public interface ICacheStorage
    {
    	void Remove(string key);
    	void Store(string key, object data);
    	void Store(string key, object data, DateTime absoluteExpiration, TimeSpan slidingExpiration);
    	T Retrieve<T>(string key);
    }


Any decent caching engine should be able to fulfil this interface. Add the following implementation to the same folder. The implementation uses ObjectCache:



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
    		return cache.Contains(key) ? (T) cache[key] : default(T);
    	}
    }


You'll need to add a reference to the System.Runtime.Caching dll in the infrastructure layer as well. ObjectCache is an all-purpose cache which works with pretty much any .NET project type. However, say you'd like to go for an HttpContext cache then you can have the following implementation:



    public class HttpContextCacheStorage : ICacheStorage
    {
    	public void Remove(string key)
    	{
    		HttpContext.Current.Cache.Remove(key);
    	}

    	public void Store(string key, object data)
    	{
    		HttpContext.Current.Cache.Insert(key, data);
    	}

    	public void Store(string key, object data, DateTime absoluteExpiration, TimeSpan slidingExpiration)
    	{
    		HttpContext.Current.Cache.Insert(key, data, null, absoluteExpiration, slidingExpiration);
    	}

    	public T Retrieve<T>(string key)
    	{
    		T itemStored = (T)HttpContext.Current.Cache.Get(key);
    		if (itemStored == null)
    			itemStored = default(T);

    		return itemStored;
    	}
    }


We'll need to be able to inject our caching implementation to the product service and not let the product service determine which strategy to take. In that case the implementation is still hidden to the caller and we still have a hard dependency on one of the concrete implementations. The revised ProductService looks as follows:



    public class ProductService : IProductService
    {
    	private readonly IProductRepository _productRepository;
    	private readonly ICacheStorage _cacheStorage;

    	public ProductService(IProductRepository productRepository, ICacheStorage cacheStorage)
    	{
    		if (productRepository == null) throw new ArgumentNullException("ProductRepository");
    		if (cacheStorage == null) throw new ArgumentNullException("CacheStorage");
    		_productRepository = productRepository;
    		_cacheStorage = cacheStorage;
    	}

    	public GetProductResponse GetProduct(GetProductRequest getProductRequest)
    	{
    		GetProductResponse response = new GetProductResponse();
    		try
    		{
    			string storageKey = "GetProductById";
    			Product p = _cacheStorage.Retrieve<Product>(storageKey);
    			if (p == null)
    			{
    				p = _productRepository.FindBy(getProductRequest.Id);
    				_cacheStorage.Store(storageKey, p, DateTime.Now.AddMinutes(5), TimeSpan.Zero);

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
    }


You'll need to add a reference to the infrastructure layer from the console.

We've successfully got rid of the ObjectCache dependency. You can even remove the System.Runtime.Caching dll from the references list in the Console layer. The ProductService creation code in Main is modified as follows:



    IProductService productService = new ProductService(new ProductRepository(), new SystemRuntimeCacheStorage());


Run the code and you'll see that it still works. Now ProductService is oblivious of the concrete implementation of ICacheStorage and you're free to change it in the caller. You can then change the caching mechanism for all your caching needs by changing the concrete implementation of ICacheStorage.

**Step 2: removing caching altogether**

As it currently stands the GetProduct method still violates '[S][3]' in SOLID, i.e. the Single Responsibility Principle. The purpose of GetProduct is to retrieve a product from the injected repository, it has nothing to do with caching. The constructor signature at least indicates to the caller that there's caching going on – the caller must send an ICacheStorage implementation – but testing the true purpose of GetProduct is still not straightforward.

Luckily we have the Decorator pattern hinted at above to solve the problem. I'll not go into the details of the pattern here, you can read about it in great detail under the link provided. In short it helps to augment the functionality of a class in an object-oriented manner by building an encasing, "enriched" version of the class. That's exactly what we'd like to build: augment the plain ProductService class with caching. Let's see what this could look like.

The Decorator pattern has a couple of different implementations, here's one variant. Add the following implementation of IProductService into the Services folder:



    public class CachedProductService : IProductService
    {
    	private readonly IProductService _productService;
    	private readonly ICacheStorage _cacheStorage;

    	public CachedProductService(IProductService productService, ICacheStorage cacheStorage)
    	{
    		if (productService == null) throw new ArgumentNullException("ProductService");
    		if (cacheStorage == null) throw new ArgumentNullException("CacheStorage");
    		_cacheStorage = cacheStorage;
    		_productService = productService;
    	}

    	public GetProductResponse GetProduct(GetProductRequest getProductRequest)
    	{
    		GetProductResponse response = new GetProductResponse();
    		try
    		{
    			string storageKey = "GetProductById";
    			Product p = _cacheStorage.Retrieve<Product>(storageKey);
    			if (p == null)
    			{
    				response = _productService.GetProduct(getProductRequest);
    				_cacheStorage.Store(storageKey, response.Product, DateTime.Now.AddMinutes(5), TimeSpan.Zero);
    			}
                            else
    			{
    				response.Success = true;
    				response.Product = p;
    			}
    		}
    		catch (Exception ex)
    		{
    			response.Exception = ex.Message;
    		}
    		return response;
    	}
    }


We delegate both the product retrieval and the caching to the injected implementations of the abstractions.

ProductService.cs can be changed back to its original form:



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


The calling code in Main will look as follows:



    static void Main(string[] args)
    {
    	IProductService productService = new ProductService(new ProductRepository());
    	IProductService cachedProductService = new CachedProductService(productService, new SystemRuntimeCacheStorage());
    	GetProductResponse getProductResponse = cachedProductService.GetProduct(new GetProductRequest() { Id = 2 });
    	if (getProductResponse.Success)
    	{
    		Console.WriteLine(string.Concat("Product name: ", getProductResponse.Product.Name));
    	}
    	else
    	{
    		Console.WriteLine(getProductResponse.Exception);
    	}

    	getProductResponse = cachedProductService.GetProduct(new GetProductRequest() { Id = 2 });

    	Console.ReadKey();
    }


Note how we first create a ProductService which is then injected into the CachedProductService along with the selected caching strategy.

So now ProductService can be tested in isolation.

**Plan B: Null object caching**

You are not always in control of all parts of the source code. In other cases changing ProductService like that may cause a long delay in development time due to tightly coupled code. So imagine that you have to use a product service like we had after step 1:



    public ProductService(IProductRepository productRepository, ICacheStorage cacheStorage)


So you have to inject a caching strategy but still want to test the true purpose of GetMessage, i.e. bypass caching altogether. The [Null Object pattern][6] comes to the rescue. So we create a dummy implementation of ICacheStorage that doesn't do anything. Add the following implementation into the Caching folder of the infrastructure layer:



    public class NoCacheStorage : ICacheStorage
    {
    	public void Remove(string key)
    	{}

    	public void Store(string key, object data)
    	{}

    	public void Store(string key, object data, DateTime absoluteExpiration, TimeSpan slidingExpiration)
    	{}

    	public T Retrieve<T>(string key)
    	{
    		return default(T);
    	}
    }


You can inject this dummy implementation to ProductService to eliminate all caching:



    IProductService productService = new ProductService(new ProductRepository(), new NoCacheStorage());


In the [next post][7] we'll look at how to hide reading from a configuration file.

View the list of posts on Architecture and Patterns [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/01/externalising-dependencies-with-dependency-injection-in-net-part-1-setup/ "Externalising dependencies with Dependency Injection in .NET part 1: setup"
[2]: http://msdn.microsoft.com/en-us/library/system.runtime.caching.objectcache(v=vs.110).aspx "ObjectCache in MSDN"
[3]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[4]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[5]: http://dotnetcodr.com/2013/05/13/design-patterns-and-practices-in-net-the-decorator-design-pattern/ "Design patterns and practices in .NET: the Decorator design pattern"
[6]: http://dotnetcodr.com/2013/05/06/design-patterns-and-practices-in-net-the-null-object-pattern/ "Design patterns and practices in .NET: the Null Object pattern"
[7]: http://dotnetcodr.com/2014/09/08/externalising-dependencies-with-dependency-injection-in-net-part-3-configuration/ "Externalising dependencies with Dependency Injection in .NET part 3: configuration"
[8]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
