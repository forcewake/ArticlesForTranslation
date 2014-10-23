[Source](http://dotnetcodr.com/2013/09/26/a-model-net-web-service-based-on-domain-driven-design-part-5-the-concrete-repository/ "Permalink to A model .NET web service based on Domain Driven Design Part 5: the concrete Repository")

# A model .NET web service based on Domain Driven Design Part 5: the concrete Repository

**Introduction**

In the [previous][1] post we laid the foundation for our repository layer. It's now time to see how those elements can be implemented. The implemented data access layer will be a simple in-memory data storage solution. I could have selected something more technical, like EntityFramework but giving a detailed account on a data storage mechanism is not the target of these series. It would probably only sidetrack us too much. The things we take up in this post should suffice for you to implement an EF-based concrete repository.

This is quite a large topic and it's easy to get lost so this post will be devoted to laying the foundation of domain-specific repositories. In the next post we'll implement the Customer domain repository.

**The Customer repository**

We need to expose the operations that the consumer of the code is allowed to perform on the Customer object. Recall from the previous post that we have 2 interfaces: one for read-only entities and another for the ones where we allow all CRUD operations. We want to be able to insert, select, modify and delete Customer objects. The domain-specific repository interfaces must be declared in the Domain layer.

This point is important to keep in mind: **it is only the data access abstraction we define in the Domain layer. It is up to the implementing data access mechanism to hide the implementation details.** We do this in order to keep the Domain objects completely free of persistence logic so that they remain persistence-ignorant. You can persist the domains in the repository layer the way you want, it doesn't matter, as long as those details do not bubble up to the other layers in any way, shape or form. Once you start referencing e.g. the DB object context in the web layer you have committed to use EntityFramework and a switch to another technology will be all the more difficult.

Insert the following interface in the Domain/Customer folder:



    public interface ICustomerRepository : IRepository<Customer, int>
    {
    }


Right now it's an empty interface as we don't want to declare any new data access capability over and above the methods already defined in the IRepository interface. Recall that we included the FindAll() method in the IReadOnlyRepository interface which allows us to retrieve all entities from the data store. This may be dangerous to perform on an entity where we have millions of rows in the database. Therefore you may want to remove that method from the interface. However, if we only have a few customers then it may be OK and you could have the following customer repository:



    public interface ICustomerRepository : IRepository<Customer, int>
    {
    	IEnumerable<Customer> FindAll();
    }


**The Repository layer**

Insert a new C# class library called DDDSkeletonNET.Portal.Repository.Memory. Add a reference to the Infrastructure layer as we'll need access to the abstractions defined there. We'll first need to implement the Unit of Work. Add the following stub implementation to the Repository layer:



    public class InMemoryUnitOfWork : IUnitOfWork
    {
    	public void RegisterUpdate(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository)
    	{}

    	public void RegisterInsertion(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository)
    	{}

    	public void RegisterDeletion(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository)
    	{}

    	public void Commit()
    	{}
    }


Let's fill this in. As we don't have any built-in solution to track the changes to our entities we'll have to store the changes ourselves in in-memory collections. Add the following private fields to the class:



    private Dictionary<IAggregateRoot, IUnitOfWorkRepository> _insertedAggregates;
    private Dictionary<IAggregateRoot, IUnitOfWorkRepository> _updatedAggregates;
    private Dictionary<IAggregateRoot, IUnitOfWorkRepository> _deletedAggregates;


We keep track of the changes in these dictionaries. Recall the function of the unit of repository: it will perform the actual data persistence.

We'll initialise these objects in the unit of work constructor:



    public InMemoryUnitOfWork()
    {
    	_insertedAggregates = new Dictionary<IAggregateRoot, IUnitOfWorkRepository>();
    	_updatedAggregates = new Dictionary<IAggregateRoot, IUnitOfWorkRepository>();
    	_deletedAggregates = new Dictionary<IAggregateRoot, IUnitOfWorkRepository>();
    }


Next we implement the registration methods which in practice only means that we're filling up these dictionaries:



    public void RegisterUpdate(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository)
    {
    	if (!_updatedAggregates.ContainsKey(aggregateRoot))
    	{
    		_updatedAggregates.Add(aggregateRoot, repository);
    	}
    }

    public void RegisterInsertion(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository)
    {
    	if (!_insertedAggregates.ContainsKey(aggregateRoot))
    	{
    		_insertedAggregates.Add(aggregateRoot, repository);
    	}
    }

    public void RegisterDeletion(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository)
    {
    	if (!_deletedAggregates.ContainsKey(aggregateRoot))
    	{
    		_deletedAggregates.Add(aggregateRoot, repository);
    	}
    }


We only want to add those changes that haven't been added before hence the ContainsKey guard clause.

In the commit method we ask the unit of work repository to persist those changes:



    public void Commit()
    {
    	foreach (IAggregateRoot aggregateRoot in _insertedAggregates.Keys)
    	{
    		_insertedAggregates[aggregateRoot].PersistInsertion(aggregateRoot);
    	}

    	foreach (IAggregateRoot aggregateRoot in _updatedAggregates.Keys)
    	{
    		_updatedAggregates[aggregateRoot].PersistUpdate(aggregateRoot);
    	}

    	foreach (IAggregateRoot aggregateRoot in _deletedAggregates.Keys)
    	{
    		_deletedAggregates[aggregateRoot].PersistDeletion(aggregateRoot);
    	}
    }


At present we don't care about transactions and rollbacks as a fully optimised implementation is not the main goal here. We can however extend the Commit method to accommodate transactions:



    public void Commit()
    {
        using (TransactionScope scope = new TransactionScope())
        {
             //foreach loops...
             scope.Complete();
        }
    }


You'll need to import the System.Transactions library for this to work.

So what does a unit of work repository implementation look like? We'll define it as a base abstract class that each concrete repository must derive from. We'll build up an example step by step. Add the following stub to the Repository layer:



    public abstract class Repository<DomainType, IdType, DatabaseType> : IUnitOfWorkRepository where DomainType :  IAggregateRoot
    {
    	private readonly IUnitOfWork _unitOfWork;

    	public Repository(IUnitOfWork unitOfWork)
    	{
    		if (unitOfWork == null) throw new ArgumentNullException("Unit of work");
    		_unitOfWork = unitOfWork;
    	}

    	public void PersistInsertion(IAggregateRoot aggregateRoot)
    	{

    	}

    	public void PersistUpdate(IAggregateRoot aggregateRoot)
    	{

    	}

    	public void PersistDeletion(IAggregateRoot aggregateRoot)
    	{

    	}
    }


There's not much functionality in here yet, but it's an important first step. The Repository abstract class implements IUnitOfWorkRepository so it will be the persistence workhorse of the data storage. It has three type parameters: DomainType, IdType and DatabaseType.

TypeId probably sounds familiar by now so I won't dive into more details now. Then we distinguish between the domain type and the database type. The domain type will be the type of the domain class such as the Customer domain we've created. The database type will be the database representation of the same domain.

Why might we need a database type? It is not guaranteed that a domain object will have the same structure in the storage as in the domain layer. In the domain layer you have the freedom of adding, modifying and removing properties, rules etc. You may not have the same freedom in a relational database. If a data table is used by stored procedures and user-defined functions then you cannot just remove columns at will without breaking a lot of other DB logic. With the advent of NoSql databases, such as MongoDb or RavenDb, which allow a very loose and flexible data structure this requirement may change but at the time of writing this post relational databases are still the first choice for data storage. Usually as soon as a database is put into production and LIVE data starts to fill up the data tables with potentially hundreds of thousands of rows every day then the data table structure becomes a lot more rigid than your domain classes. Hence it can be a good idea to isolate the "domain" and "database" representations of a domain object. It is of course only the concrete repository layer that should know how a domain object is represented in the data storage.

Before we continue with the Repository class let's simulate the database representation of the Customer class. This can be likened to the objects that the EF or Linq to SQL automation tools generate for you. Insert a new folder called Database in the Repository.Memory layer. Insert the following class into it:



    public class DatabaseCustomer
    {
    	public int Id { get; set; }
    	public string CustomerName { get; set; }
    	public string Address { get; set; }
    	public string Country { get; set; }
    	public string City { get; set; }
    	public string Telephone { get; set; }
    }


This example shows an advantage with having separate domain and database representations. Recall that the Customer domain has an Address value object. In this example we imagine that there's no separate Address table, we didn't think of that when we designed the original database. As the database has been in use for some time it's probably even too late to create a separate Address table so that we don't interrupt the LIVE system.

However, we don't care because we've allowed for this possibility by the type parameters. We're free to construct and convert between domain and database objects in the concrete repository layer.

The next object we'll need in the database is an imitation of the DB object context in EF and Linq to SQL. I realise that I may be talking too much about these specific ORM technologies but it probably suits the vast majority of data-driven .NET projects out there. Insert the following class into the Database folder:



    public class InMemoryDatabaseObjectContext
    {
    	//simulation of database collections
    	public List<DatabaseCustomer> DatabaseCustomers { get; set; }

    	public InMemoryDatabaseObjectContext()
    	{
    		InitialiseDatabaseCustomers();
    	}

    	public void AddEntity<T>(T databaseEntity)
    	{
    		if (databaseEntity is DatabaseCustomer)
    		{
                            DatabaseCustomer databaseCustomer = databaseEntity as DatabaseCustomer;
    			databaseCustomer.Id = DatabaseCustomers.Count + 1;
    			DatabaseCustomers.Add(databaseEntity as DatabaseCustomer);
    		}
    	}

    	public void UpdateEntity<T>(T databaseEntity)
    	{
    		if (databaseEntity is DatabaseCustomer)
    		{
    			DatabaseCustomer dbCustomer = databaseEntity as DatabaseCustomer;
    			DatabaseCustomer dbCustomerToBeUpdated = (from c in DatabaseCustomers where c.Id == dbCustomer.Id select c).FirstOrDefault();
    			dbCustomerToBeUpdated.Address = dbCustomer.Address;
    			dbCustomerToBeUpdated.City = dbCustomer.City;
    			dbCustomerToBeUpdated.Country = dbCustomer.Country;
    			dbCustomerToBeUpdated.CustomerName = dbCustomer.CustomerName;
    			dbCustomerToBeUpdated.Telephone = dbCustomer.Telephone;
    		}
    	}

    	public void DeleteEntity<T>(T databaseEntity)
    	{
    		if (databaseEntity is DatabaseCustomer)
    		{
    			DatabaseCustomer dbCustomer = databaseEntity as DatabaseCustomer;
    			DatabaseCustomer dbCustomerToBeDeleted = (from c in DatabaseCustomers where c.Id == dbCustomer.Id select c).FirstOrDefault();
    			DatabaseCustomers.Remove(dbCustomerToBeDeleted);
    		}
    	}

    	private void InitialiseDatabaseCustomers()
    	{
    		DatabaseCustomers = new List<DatabaseCustomer>();
    		DatabaseCustomers.Add(new DatabaseCustomer(){Address = "Main street", City = "Birmingham", Country = "UK", CustomerName ="GreatCustomer", Id = 1, Telephone = "N/A"});
    		DatabaseCustomers.Add(new DatabaseCustomer() { Address = "Strandvägen", City = "Stockholm", Country = "Sweden", CustomerName = "BadCustomer", Id = 2, Telephone = "123345456" });
    		DatabaseCustomers.Add(new DatabaseCustomer() { Address = "Kis utca", City = "Budapest", Country = "Hungary", CustomerName = "FavouriteCustomer", Id = 3, Telephone = "987654312" });
    	}
    }


First we have a collection of DB customer objects. In the constructor we initialise the collection values so that we don't start with an empty one. Then we add insertion, update and delete methods for database type T to make it generic. These are somewhat analogous to the InsertOnSubmit, SubmitChanges and the DeleteOnSubmit methods from the Linq to SQL object context. The implementation itself is not too clever of course as we need to check the type but it's OK for now. Perfection is not the goal in this part of the code, the automation tools will prepare a lot more professional code for you from the database. In the AddEntity we assign an ID based on the number of elements in the database. This is a simulation of the ID auto-assign feature of relational databases.

We'll need access to this InMemoryDatabaseObjectContext in the repository layer. In a real system this object will be used intensively therefore access to it should be regulated. Creating a new object context for every single DB operation is not too clever so we'll hide the access behind a thread-safe lazy singleton class. If you don't know what these term mean, check out my post on the [singleton design pattern][2]. I won't repeat the stuff that's written there.

Insert the following [abstract factory][3] interface in the Database folder:



    public interface IObjectContextFactory
    {
    	InMemoryDatabaseObjectContext Create();
    }


The interface will be implemented by the following class:



    public class LazySingletonObjectContextFactory : IObjectContextFactory
    {
    	public InMemoryDatabaseObjectContext Create()
    	{
    		return InMemoryDatabaseObjectContext.Instance;
    	}
    }


The InMemoryDatabaseObjectContext object does not have any Instance property yet, so add the following code to it:



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


If you don't understand what this code does then make sure you read through the post on the singleton pattern.

We'll need a reference to the abstract factory in the Repository object, so modify the private field declarations and the constructor as follows:



    private readonly IUnitOfWork _unitOfWork;
    private readonly IObjectContextFactory _objectContextFactory;

    public Repository(IUnitOfWork unitOfWork, IObjectContextFactory objectContextFactory)
    {
    	if (unitOfWork == null) throw new ArgumentNullException("Unit of work");
    	if (objectContextFactory == null) throw new ArgumentNullException("Object context factory");
    	_unitOfWork = unitOfWork;
    	_objectContextFactory = objectContextFactory;
    }


At this point it's also clear that we'll need to be able to convert a domain type to a database type. This is best implemented in the concrete repository classes so it's enough to add an abstract method in the Repository class:



    public abstract DatabaseType ConvertToDatabaseType(DomainType domainType);


We can implement the Persist methods as follows:



    public void PersistInsertion(IAggregateRoot aggregateRoot)
    {
    	DatabaseType databaseType = RetrieveDatabaseTypeFrom(aggregateRoot);
    	_objectContextFactory.Create().AddEntity<DatabaseType>(databaseType);
    }

    public void PersistUpdate(IAggregateRoot aggregateRoot)
    {
    	DatabaseType databaseType = RetrieveDatabaseTypeFrom(aggregateRoot);
    	_objectContextFactory.Create().UpdateEntity<DatabaseType>(databaseType);
    }

    public void PersistDeletion(IAggregateRoot aggregateRoot)
    {
    	DatabaseType databaseType = RetrieveDatabaseTypeFrom(aggregateRoot);
    	_objectContextFactory.Create().DeleteEntity<DatabaseType>(databaseType);
    }

    private DatabaseType RetrieveDatabaseTypeFrom(IAggregateRoot aggregateRoot)
    {
    	DomainType domainType = (DomainType)aggregateRoot;
    	DatabaseType databaseType = ConvertToDatabaseType(domainType);
    	return databaseType;
    }


We need to convert the incoming IAggregateRoot object to the database type as it is the database type that the DB understands. The DB type is its own representation of the domain so we need to talk to it that way.

So these are the persistence methods but we need to register the changes first. Add the following methods to Repository.cs:



    public void Update(DomainType aggregate)
    {
    	_unitOfWork.RegisterUpdate(aggregate, this);
    }

    public void Insert(DomainType aggregate)
    {
    	_unitOfWork.RegisterInsertion(aggregate, this);
    }

    public void Delete(DomainType aggregate)
    {
    	_unitOfWork.RegisterDeletion(aggregate, this);
    }


These operations are so common and repetitive that we can put them in this abstract base class instead of letting the domain-specific repositories implement them over and over again. All they do is adding the operations to the queue of the unit of work to be performed when Commit() is called. Upon Commit() the unit of work repository, i.e. the abstract Repository object will persist the changes using the in memory object context.

This model is relatively straightforward to change:

* If you need a plain file-based storage mechanism then you might create a FileObjectContext class where you read to and from a file.
* For EntityFramework and Linq to SQL you'll use the built-in object context classes to keep track of and persist the changes
* File-based NoSql solutions generally also have drivers for .NET – they can be used to implement a NoSql solution

So the possibilities are endless in fact. You can hide the implementation behind abstractions such as the IUnitOfWork interface or the Repository abstract class. It may well be that you need to add methods to the Repository class depending on the data storage technology you use but that's perfectly acceptable. The concrete repository layer is…, well, concrete. You can dedicate it to a specific technology, just like the one we're building here is dedicated to an in-memory storage mechanism. You can have several different implementations of the IUnitOfWork and IUnitOfWorkRepository interfaces and test them before you let your application go public. You can even mix and match the implementations:

* Register the changes in a temporary file and commit them in NoSql
* Register the changes in memory and commit them to the cache
* Register the changes in cache and commit them to SQL Azure

Of course these combinations are very exotic but it shows you the flexibility behind all these abstractions. Don't assume that the Repository class is a solution for ALL types of concrete unit of work. You'll certainly have to modify it depending on the concrete data storage mechanism you work with. However, note the following points:

* **The rest of the application will not be concerned with the concrete data access implementation**
* **The implementation details are well hidden behind this layer without them bubbling up to and permeating the other layers**

So as long as the concrete implementations are hidden in the data access layer you'll be fine.

There's one last abstract method we'll add to Repository.cs is the ubiquitous find-by-id method which we'll delegate to the implementing classes:



    public abstract DomainType FindBy(IdType id);


We'll implement the CustomerRepository class in the next post.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/23/a-model-net-web-service-based-on-domain-driven-design-part-4-the-abstract-repository/ "A model .NET web service based on Domain Driven Design Part 4: the abstract Repository"
[2]: http://dotnetcodr.com/2013/05/09/design-patterns-and-practices-in-net-the-singleton-pattern/ "Design patterns and practices in .NET: the Singleton pattern"
[3]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
