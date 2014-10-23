[Source](http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Permalink to Design patterns and practices in .NET: the Adapter Pattern")

# Design patterns and practices in .NET: the Adapter Pattern

**Introduction**

The adapter pattern is definitely one of the most utilised designed patterns in software development. It can be used in the following situation:

Say you have a class – Class A – with a dependency on some interface type. On the other hand you have another class – Class B – that would be ideal to inject into Class A as the dependency, but Class B does not implement the necessary interface.

Another situation is when you want to factor out some tightly coupled dependency. A common scenario would be a direct usage of some .NET class, say HttpRuntime in a method that you want to test. You must have a valid HttpRuntime while running the test otherwise the test may fail and that is of course not acceptable. The solution is to let the method be dependent on some abstraction that is injected to it – the question is how to extract HttpRuntime and make it implement some interface instead? After all, it's unlikely that you'll have write access to the HttpRuntime class, right?

The solution is to write an adapter that sits between Class A and Class B that wraps Class B's functionality. An example from the real world: if you travel from Britain to Sweden and try to plug in your PC in your hotel room you'll fail as the electric sockets in Sweden are different from those in the UK. Class A is your PC's power plug and Class B is the socket in the wall. The two objects clearly don't fit but there's a solution: you can insert a specially made adapter into the socket which makes the "conversion" between the plug and the socket enabling you to pair them up. In summary: the adapter pattern allows classes to work together that couldn't otherwise due to incompatible interfaces. The adapter will take the form of an interface which has the additional benefit of extensibility: you can write many implementations of this adapter interface making sure your method is not tightly coupled to one single concrete implementation.

**Demo**

Open Visual Studio and create a new blank solution. Add a new class library called Domain and delete Class1.cs. Add a new class to this project called Customer. We'll leave it void of any properties, that is not the point here. We want to expose the repository operations by an interface, so insert an interface called ICustomerRepository to the Domain layer:



    public interface ICustomerRepository
    	{
    		IList<Customer> GetCustomers();
    	}


Add another class library called Repository and a class called CustomerRepository which implements ICustomerRepository:



    public class CustomerRepository : ICustomerRepository
    	{
    		public IList<Customer> GetCustomers()
    		{
    			//simulate database operation
    			return new List<Customer>();
    		}
    	}


Add a new class library project called Service and a class called CustomerService to it:



    public class CustomerService
    	{
    		private readonly ICustomerRepository _customerReposutory;

    		public CustomerService(ICustomerRepository customerRepository)
    		{
    			_customerReposutory = customerRepository;
    		}

    		public IList<Customer> GetAllProducts()
    		{
    			IList<Customer> customers;
    			string storageKey = "GetAllProducts";
    			customers = (List<Customer>)HttpContext.Current.Cache.Get(storageKey);
    			if (customers == null)
    			{
    				customers = _customerReposutory.GetCustomers();
    				HttpContext.Current.Cache.Insert(storageKey, customers);
    			}

    			return customers;
    		}
    	}


This bit of code should be straightforward: we want to return a list of customers using the abstract dependency ICustomerRepository. We check if the list is available in the HttpContext cache. If that's the case then convert the cached value and return the customers. Otherwise fetch the list from the repository and put the value to the cache.

So what's wrong with the GetAllProducts method?

_Testability_

The method is difficult to test because of the dependency on the HttpContext class. If you want to get any reliable result from the test that tests the behaviour of this method you'll need to somehow provide a valid HttpContext object. Otherwise if the test fails, then why did it fail? Was it a genuine failure, meaning that the customer list was not retrieved? Or was it because there was no HttpContext available? It's the wrong approach making the test outcome dependent on such a volatile object.

_Flexibility_

With this implementation we're stuck with the HttpContext as our caching solution. What if we want to change over to a different one, such as Memcached or System.Runtime? In that case we'd need to go in and manually replace the HttpContext solution to a new one. Even worse, let's say all your service classes use HttpContext for caching and you want to make the transition to another caching solution for all of them. You probably see how tedious, time consuming and error-prone this could be.

On a different note: the method also violates the Single Responsibility Principle as it performs caching in its body. Strictly speaking it should not be doing this as it then introduces a hidden side effect. The solution to that problem is provided by the Decorator pattern, which we'll look at in the next blog post.

**Solution**

It's clear that we have to factor out the HttpContext.Current.Cache object and let the consumer of CustomerService inject it instead – a simple design principle known as Dependency Injection. As usual, the most optimal option is to write an abstraction that encapsulates the functions of a cache solution. Insert the following interface to the Service layer:



    public interface ICacheStorage
    	{
    		void Remove(string key);
    		void Store(string key, object data);
    		T Retrieve<T>(string key);
    	}


I believe this is quite straightforward. Next we want to update CustomerService to depend on this interface:



    public class CustomerService
    	{
    		private readonly ICustomerRepository _customerReposutory;
    		private readonly ICacheStorage _cacheStorage;

    		public CustomerService(ICustomerRepository customerRepository, ICacheStorage cacheStorage)
    		{
    			_customerReposutory = customerRepository;
    			_cacheStorage = cacheStorage;
    		}

    		public IList<Customer> GetAllProducts()
    		{
    			IList<Customer> customers;
    			string storageKey = "GetAllProducts";
    			customers = _cacheStorage.Retrieve<List<Customer>>(storageKey);
    			if (customers == null)
    			{
    				customers = _customerReposutory.GetCustomers();
    				_cacheStorage.Store(storageKey, customers);
    			}

    			return customers;
    		}
    	}


We've got rid of the HttpContext object, so the next task is to inject it somehow using the ICacheStorage interface. This is the essence of the Adapter pattern: write an adapter class that will resolve the incompatibility of our home-made ICacheStorage interface and HttpContext.Current.Cache. The solution is really simple. Add a new class to the Service layer called HttpContextCacheStorage:



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

    		public T Retrieve<T>(string key)
    		{
    			T itemsStored = (T)HttpContext.Current.Cache.Get(key);
    			if (itemsStored == null)
    			{
    				itemsStored = default(T);
    			}
    			return itemsStored;
    		}
    	}


This is the adapter class that encapsulates the HttpContext caching object. You can now inject this concrete class into CustomerService. In the future if you want to make use of a different caching solution then all you need to do is to write another adapter for that: MemCached, Velocity, System.Runtime.Cache, you name it.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
