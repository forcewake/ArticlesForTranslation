[Source](http://dotnetcodr.com/2013/09/30/a-model-net-web-service-based-on-domain-driven-design-part-6-the-concrete-repository-continued/ "Permalink to A model .NET web service based on Domain Driven Design Part 6: the concrete Repository continued")

# A model .NET web service based on Domain Driven Design Part 6: the concrete Repository continued

**Introduction**

In the [previous][1] post we laid the foundation for the concrete domain-specific repositories. We implemented the IUnitOfWork and IUnitOfWorkRepository interfaces and created an abstract Repository class. It's time to see how we can implement the Customer repository. We'll continue where we left off in the previous post.

**The concrete Customer repository**

Add a new folder called Repositories in the Repository.Memory data access layer. Add a new class called CustomerRepository with the following stub:



    public class CustomerRepository : Repository<Customer, int, DatabaseCustomer>, ICustomerRepository
    {
             public CustomerRepository(IUnitOfWork unitOfWork, IObjectContextFactory objectContextFactory) :  base(unitOfWork, objectContextFactory)
    	{}

    	public override Customer FindBy(int id)
    	{
    		throw new NotImplementedException();
    	}

    	public override DatabaseCustomer ConvertToDatabaseType(Customer domainType)
    	{
    		throw new NotImplementedException();
    	}

    	public IEnumerable<Customer> FindAll()
    	{
    		throw new NotImplementedException();
    	}
    }


You'll need to add a reference to the Domain layer for this to compile.

We derive from the abstract Repository class and declare that the object is represented by the Customer class in the domain layer and by the DatabaseCustomer in the data storage layer and has an id type int. We inherit the FindBy, the ConvertToDatabaseType and the FindAll methods. FindAll() is coming indirectly from the IReadOnlyRepository interface which is implemented by IRepository which in turn is implemented by ICustomerRepository. And where are the three methods of IRepository? Remember, that the Update, Insert and Delete methods have already been implemented in the Repository class so we don't need to worry about them. Any time we create a new domain-specific repository, those methods have been taken care of.

Let's implement the methods one by one. In the FindBy(int id) method we'll need to consult the DatabaseCustomers collection first and get the DatabaseCustomer object with the incoming id. Then we'll populate the Customer domain object from its database representation. The DatabaseCustomers collection can be retrieved using the IObjectContextFactory dependency of the base class. However, its backing field is private, so let's add the following read-only property to Repository.cs:



    public IObjectContextFactory ObjectContextFactory
    {
    	get
    	{
    		return _objectContextFactory;
    	}
    }


The FindBy(int id) method can be implemented as follows:



    public override Customer FindBy(int id)
    {
    	DatabaseCustomer databaseCustomer = (from dc in ObjectContextFactory.Create().DatabaseCustomers
    										 where dc.Id == id
    										 select dc).FirstOrDefault();
    	Customer customer = new Customer()
    	{
    		Id = databaseCustomer.Id
    		,Name = databaseCustomer.CustomerName
    		,CustomerAddress = new Address()
    		{
    			AddressLine1 = databaseCustomer.Address
    			,AddressLine2 = string.Empty
    			,City = databaseCustomer.City
    			,PostalCode = "N/A"
    		}
    	};
    	return customer;
    }


So we populate the Customer object based on the properties of the DatabaseCustomer object. We don't store the postal code or address line 2 in the DB so they are not populated. In a real life scenario we'd probably need to rectify this by extending the DatabaseCustomer table, but for now we don't care. This example shows again the advantage of having complete freedom over the database and domain representations of a domain. You have the freedom of populating the domain object from its underlying database representation. You can even consult other database objects if needed as the domain object properties may be dispersed across 2-3 or even more database tables. The domain object won't be concerned with such details. The domain and the database are completely detached so you don't have to worry about changing either the DB or the domain representation.

Let's factor out the population process to a separate method as it will be needed later:



    private Customer ConvertToDomain(DatabaseCustomer databaseCustomer)
    {
    	Customer customer = new Customer()
    	{
    		Id = databaseCustomer.Id
    		,Name = databaseCustomer.CustomerName
    		,CustomerAddress = new Address()
    		{
    			AddressLine1 = databaseCustomer.Address
    			,AddressLine2 = string.Empty
    			,City = databaseCustomer.City
    			,PostalCode = "N/A"
    		}
    	};
    	return customer;
    }


The update version of FindBy(int id) looks as follows:



    public override Customer FindBy(int id)
    {
    	DatabaseCustomer databaseCustomer = (from dc in ObjectContextFactory.Create().DatabaseCustomers
    										 where dc.Id == id
    										 select dc).FirstOrDefault();
    	if (databaseCustomer != null)
    	{
    		return ConvertToDomain(databaseCustomer);
    	}
    	return null;
    }


The ConvertToDatabaseType(Customer domainType) method will be used when inserting and modifying domain objects:



    public override DatabaseCustomer ConvertToDatabaseType(Customer domainType)
    {
    	return new DatabaseCustomer()
    	{
    		Address = domainType.CustomerAddress.AddressLine1
    		,City = domainType.CustomerAddress.City
    		,Country = "N/A"
    		,CustomerName = domainType.Name
    		,Id = domainType.Id
    		,Telephone = "N/A"
    	};
    }


Nothing fancy here I suppose.

Finally FindAll simply retrieves all customers from the database and converts each to a domain type:



    public IEnumerable<Customer> FindAll()
    {
    	List<Customer> allCustomers = new List<Customer>();
    	List<DatabaseCustomer> allDatabaseCustomers = (from dc in ObjectContextFactory.Create().DatabaseCustomers
    						   select dc).ToList();
    	foreach (DatabaseCustomer dc in allDatabaseCustomers)
    	{
    		allCustomers.Add(ConvertToDomain(dc));
    	}
    	return allCustomers;
    }


That's it, there's at present nothing more to add the CustomerRepository class. If you add any specialised queries in the ICustomerRepository interface they will need to be implemented here.

I'll finish this post with a couple of remarks on EntityFramework.

**Entity Framework**

There is a way to implement the Unit of Work pattern in conjunction with the EntityFramework in a different, more simplified way. The database type is completely omitted from the structure leading to the following Repository abstract class declaration:



    public abstract class Repository<DomainType, IdType> : IUnitOfWorkRepository where DomainType : IAggregateRoot


So there's no trace of how the domain is represented in the database. There is a way in the EF designer to map the columns of a table to the properties of an object. You can specify the namespace of the objects created by EF – the .edmx file – like this:

1\. Remove the code generation tool from the edmx file properties:

![Remove EF code generation tool][2]

2\. Change the namespace of the project by right-clicking anywhere on the EF diagram and selecting Properties:

![Change namespace in entity framework][3]

Change that value to the namespace of your domain objects.

3\. If the ID of a domain must be auto-generated by the database then you'll need to set the StoreGeneratedPattern to Identity on that field:

![EF identity generation][4]

You can fine-tune the mapping by opening the .edmx file in a text editor. It is standard XML so you can edit it as long as you know what you are doing.

This way we don't need to declare the domain type and the database type, we can work with only one type because it will be common to EF and the domain model. It is reasonable to follow down this path as it simplifies certain things, e.g. you don't need the conversions between DB and domain types. However, there are some things to keep in mind I believe.

We inherently couple the domain object to its database counterpart. It may work with simple domain objects, but not with more complex objects where the database representation can be spread out in different tables. We may even face a different scenario: say we made a design mistake in the database structuring phase and built a table that represents more than one domain object. We may not have the possibility of rebuilding that table as it contains millions of rows and is deployed in a production environment. How do we then map 2 or more domain objects to the same database table in the EF designer? It won't work, at least I don't know any way how this can be solved. Also, mapping stored procedures to domain objects or domain object rules can be problematic. What's more, you'll need to mark those properties of your domains virtual where you want to allow lazy loading by the ORM framework like this:



    public class Book
    {
        public int Id { get; set; }
        public virtual Title Title { get; set; }
    }


I think this breaks the persistence ignorance (PI) feature of proper POCO classes. **We're modifying the domain to give way for a persistence technology**.

However, you may be OK with these limitations. Your philosophy may well differ from my rigid PI and POCO approach. It is certainly desirable that the database tables and domains are as close as possible, but you don't always have this luxury with large legacy databases. If you start off with a completely empty database with no tables then EF and the code-first approach – where the data tables will be created for you based on your self-written object context implementation – can simplify the repository development process. However, keep in mind that database tables are a lot more rigid and resistant to change than your loosely coupled and layered code. Once the database has been put into production and you want to change the structure of a domain object you may run into difficulties as the corresponding database table may not be modified that easily.

In the next post we'll discuss the application services layer.

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/26/a-model-net-web-service-based-on-domain-driven-design-part-5-the-concrete-repository/ "A model .NET web service based on Domain Driven Design Part 5: the concrete Repository"
[2]: http://dotnetcodr.files.wordpress.com/2013/09/efremovecodegenerationtool.png?w=630&h=161
[3]: http://dotnetcodr.files.wordpress.com/2013/09/efchangenamespace.png?w=630&h=145
[4]: http://dotnetcodr.files.wordpress.com/2013/09/efidentity.png?w=630&h=158
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
