[Source](http://dotnetcodr.com/2014/03/31/extension-to-the-ddd-skeleton-project-per-session-object-context-in-repository-layer/ "Permalink to Extension to the DDD skeleton project: per session object context in Repository layer")

# Extension to the DDD skeleton project: per session object context in Repository layer

**Introduction**

If you followed through the [DDD skeleton project][1] blog series then you'll recall that the following interface was used to get hold of the DB object context in the concrete repositories:



    public interface IObjectContextFactory
    {
    	InMemoryDatabaseObjectContext Create();
    }


You'll also recall that we pulled a valid InMemoryDatabaseObjectContext instance using the singleton pattern:



    public class LazySingletonObjectContextFactory : IObjectContextFactory
    {
    	public InMemoryDatabaseObjectContext Create()
    	{
    		return InMemoryDatabaseObjectContext.Instance;
    	}
    }


In retrospect I think I should have gone through another implementation of the IObjectContextFactory interface, one that better suits a real life scenario with an ORM platform such as Entity Framework or NHibernate.

If you'd like to take on the singleton pattern to construct and retrieve an EF object context then you'll likely run into some difficulties. The singleton pattern was OK for this in-memory solution so that we could demonstrate insertions, deletions and updates in the repository. As the singleton always returns the same InMemoryDatabaseObjectContext instance we won't lose the new data from request to request as we saw in the [demo][2].

However, this is not desirable if you're working with an EF object context. If you always return the same object context for every request then even the data retrieved by that object context will be the same upon subsequent requests. If you make a change to an object then it won't be visible in another session as that new session retrieves the same object context that was present when it was created by the singleton, i.e. when it was first called:



    public static InMemoryDatabaseObjectContext Instance
    {
    	get
    	{
    		return Nested.instance;
    	}
    }

    private class Nested
    {
    	static Nested()
    	{
    	}
    	internal static readonly InMemoryDatabaseObjectContext instance = new InMemoryDatabaseObjectContext();
    }


If you update a Customer object in one session then the next session won't see the updates as it still retrieves the original, "unchanged" Customer that was present in the original instance of InMemoryDatabaseObjectContext. You'll need to call the Refresh method of the EF object context manually to make sure that you get the latest state of the retrieved object like this:



    Customer c = some query;
    if (c != null)
    {
    	EfObjectContextInstance.Refresh(RefreshMode.StoreWins, c);
    }


In short: my recommendation is that you don't use the singleton pattern to retrieve the EF object context.

**An EntityFramework friendly solution**

A better solution is to get an instance of the object context per HTTP session. This requires a different implementation of the IObjectContextFactory interface. The following implementation can do the job:



    public class HttpAwareOrmDataContextFactory : IObjectContextFactory
    {
    	private string _dataContextKey = "EfObjectContext";

    	public InMemoryDatabaseObjectContext Create()
    	{
    		InMemoryDatabaseObjectContext objectContext = null;
    		if (HttpContext.Current.Items.Contains(_dataContextKey))
    		{
    			objectContext = HttpContext.Current.Items[_dataContextKey] as InMemoryDatabaseObjectContext;
    		}
    		else
    		{
    			objectContext = new InMemoryDatabaseObjectContext();
    			Store(objectContext);
    		}
    		return objectContext;
    	}

    	private void Store(InMemoryDatabaseObjectContext objectContext)
    	{
    		if (HttpContext.Current.Items.Contains(_dataContextKey))
    		{
    			HttpContext.Current.Items[_dataContextKey] = objectContext;
    		}
    		else
    		{
    			HttpContext.Current.Items.Add(_dataContextKey, objectContext);
    		}
    	}
    }


We use the IDictionary object of HttpContext.Current.Items to store the InMemoryDatabaseObjectContext instance. We first check if an instance is available by the key _dataContextKey. If that's the case then we return that instance. Otherwise we create a new instance of the object context and store it in the dictionary. HttpContext.Current will of course always be different with each new HTTP request to the API which ensures that we get the fresh data from the data store as well. In case the InMemoryDatabaseObjectContext is requested more than once during the same session then we hand out the same instance.

You might ask why we don't construct a new InMemoryDatabaseObjectContext every single time one is needed like this:



    public InMemoryDatabaseObjectContext Create()
    {
         return new InMemoryDatabaseObjectContext();
    }


That wouldn't be a good idea as the object context will be called when making some changes to the underlying DB objects and then it's called again when the changes are to be persisted using two separate calls to the Unit of Work:



    void RegisterUpdate(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository);
    .
    .
    .
    void Commit();


The two calls will be very likely made during the same session otherwise the changes are lost. If you construct an InMemoryDatabaseObjectContext to register the changes and then construct another one to commit them then the second instance won't automatically track the same updates of course so it won't persist anything. That's logical as we're talking about two different instances: how would object context instance #2 know about the changes tracked by object context instance #1?

So now we have two implementations of the IObjectContextFactory interface. The beauty with our loosely coupled solution is that all it takes to change the concrete implementation injected into all Repository classes is to comment out the following line of code in IoC.cs:



    x.For<IObjectContextFactory>().Use<LazySingletonObjectContextFactory>();


…and insert the following instead:



    x.For<IObjectContextFactory>().Use<HttpAwareOrmDataContextFactory>();


You can run the project as it is now. Navigate to /customers and you should get the same dummy 3 customers we saw in the original demo.

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[2]: http://dotnetcodr.com/2013/10/14/a-model-net-web-service-based-on-domain-driven-design-part-10-tests-and-conclusions/ "A model .NET web service based on Domain Driven Design Part 10: tests and conclusions"
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
