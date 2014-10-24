[Source](http://dotnetcodr.com/2013/05/06/design-patterns-and-practices-in-net-the-null-object-pattern/ "Permalink to Design patterns and practices in .NET: the Null Object pattern")

# Design patterns and practices in .NET: the Null Object pattern

**Introduction**

Null references are a fact of life for a programmer. Some would actually call it a curse. We have to start thinking about null references even in the case of the simplest Console application in .NET.:



    static void Main(string[] args)


If there are no arguments passed in to the Main method and you access the args array with args[0] then you'll get a NullReferenceException because the array has not been initialised. You have to check for args == null already at this stage.

This problem is so pervasive that if your class or method has some dependency then the first thing you need to check is if somebody has tried to pass in a null in a guard clause:



    if (dependency == null) throw new ArgumentNullException("Dependency name");


Also, if you call a method that returns an object and you intend to use that object in some way then you may need to include the following check:



    if (object == null) return;


…or throw an exception, it doesn't matter. The point is that your code may be littered with those checks disrupting the flow. It would be a lot more efficient to be able to assume that the return value has been instantiated so that it is a 'valid' object that we can use without checking for null values first.

This is exactly the goal of the Null Object pattern: to be able to provide an 'empty' object instead of 'null' so that we don't need to check for null values all the time. The Null Object will be sort of a zero-implementation of the returned object type where the object does not perform anything meaningful.

Also, there may be times where you just don't want to make use of a dependency. You cannot pass in null as that would throw an exception – instead you can pass in a valid object that does not perform anything useful. All method calls on the Null Object will be valid, meaning you don't need to worry about null references. This may occur often in testing scenarios where the test may not care about the behaviour of the dependency as it wants to test the true logic of the system under test instead.

Of course it's not possible to get rid null checks 100%. There will still be places where you need to perform them.

This pattern is also known by other names: Stub, Active Nothing, Active Null.

**Demo**

Open Visual Studio and create a new Console application. We'll simulate that a method expects a caching strategy to cache some object. The example is similar to and builds on the example available [here][1] under the discussion on the Adapter Pattern. Insert the following interface:



    public interface ICacheStorage
    	{
    		void Remove(string key);
    		void Store(string key, object data);
    		T Retrieve<T>(string key);
    	}


Insert the following concrete type that implements the HttpContext.Current.Cache type of solution:



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


You'll need to add a reference to System.Web. It's of course not too wise to rely on the HttpContext in a Console application but that's beside the point right now.

Add the following private method to Program.cs:



    private static void PerformWork(ICacheStorage cacheStorage)
    {
    	string key = "key";
    	object o = cacheStorage.Retrieve<object>(key);
    	if (o == null)
    	{
    		//simulate database lookup
    		o = new object();
    		cacheStorage.Store(key, o);
    	}
    	//perform some work on object o...
    }


We first check whether object 'o' is available in the cache provided by the injected ICacheStorage object. If not then we fetch it from some source, like a DB and then cache it.

What if the caller doesn't want to cache the object? They might intentionally force a database lookup. If they pass in a null then they'll get a NullReferenceException. Also, if we want to test this method using TDD then we may not be interested in caching. The test may probably want to test the true logic of the code i.e. the 'perform some work on object o' bit, where the caching strategy is irrelevant.

The solution is a caching strategy that doesn't do any work:



    public class NullObjectCache : ICacheStorage
    	{
    		public void Remove(string key)
    		{

    		}

    		public void Store(string key, object data)
    		{

    		}

    		public T Retrieve<T>(string key)
    		{
    			return default(T);
    		}
    	}


If you pass this implementation to PerformWork then the object will never be cached and the Retrieve method will always return null. This forces PerformWork to look up the object in the storage. Also, you can pass this implementation from a unit test so that the caching dependency is effectively ignored.

**Another example**

Check out my post on the [factory][2] patterns. You will find an example of the Null Object Pattern there in the form of the UnknownMachine class. Instead of CreateInstance of the MachineFactory class returning null in case a concrete type was not found it returns this empty object which doesn't perform anything.

**Consequences**

Using this pattern wisely will result in fewer checks for null values: your code will be cleaner and more concise. Also, the need for code branching may decrease.

The caller must obviously know that a Null Object is returned instead of a null, otherwise they may still check for nulls. You can help them by commenting your methods and classes properly. Also, there are certainly cases where an empty NullObject, such as UnknownMachine mentioned above may be confusing for the caller. They will call the TurnOn() method but will not see anything happening. You can extend the NullObject implementation with messages indicating this status, e.g. "Cannot turn on an empty machine." or something similar.

Null objects are quite often implemented as singletons – the subject of the next post: as all NullObjects implementations of an abstraction are identical, i.e. have the same properties and states they can be shared across the application. This may become cumbersome in large applications where team members may not agree on what a NullObject representation should look like. Should it be empty? Should it have some minimal implementation? Then it's wiser to allow for multiple representations of the Null Object.

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[2]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
