[Source](http://dotnetcodr.com/2013/09/23/a-model-net-web-service-based-on-domain-driven-design-part-4-the-abstract-repository/ "Permalink to A model .NET web service based on Domain Driven Design Part 4: the abstract Repository")

# A model .NET web service based on Domain Driven Design Part 4: the abstract Repository

**Introduction**

In the [previous][1] post we created a very basic domain layer with the first domain object: Customer. We'll now see what the data access layer could look like in an abstract form. This is where we must be careful not to commit the same mistakes as in the technology-driven example of the introductory post of this series.

We must abstract away the implementation details of the data access technology we use so that we can easily switch strategies later if necessary. We cannot let any technology-specific implementation bubble up from the data access layer. These details include the object context in EntityFramework, MongoClient and MongoServer in MongoDb .NET, the objects related to the file system in a purely file based data access solution etc., you probably get the idea. We must therefore make sure that no other layer will be dependent on the concrete data access solution.

We'll first lay the foundations for abstracting away any type of data access technology and you may find it a "heavy" process with a steep learning curve. As these technologies come in many different shapes this is not the most trivial task to achieve. We can have ORM technologies like EntityFramework, file-based data storage like MongoDb, key-value style storage such as Azure storage, and it's not easy to find a common abstraction that fits all of them. We'll need to accommodate these technologies without hurting the [ISP][2] principle too much. We'll follow a couple of well-established patterns and see how they can be implemented.

**Aggregate root**

We discussed aggregates and aggregate roots in [this][3] post. Recall that aggregates are handled as one unit where the aggregate root is the entry point, the "boss" of the aggregate. This implies that the data access layer should only handle aggregate roots. It should not accept objects that lie somewhere within the aggregate. We haven't yet implemented any code regarding aggregate roots, but we can take a very simple approach. Insert the following empty interface in the Infrastructure.Common/Domain folder:



    public interface IAggregateRoot
    {
    }


Conceptually it would probably be better to create a base class for aggregate roots, but we already have one for entities. As you know an object cannot derive from two base classes so we'll indicate aggregate roots with this interface instead. At present it is simply an indicator interface with no methods, such as the ISerializable interface in .NET. We could add common properties or methods here but I cannot think of any right now.

After consulting the domain expert we decide that the Customer domain is an aggregate root: the root of its own Customer aggregate. Right now the Customer aggregate has only one member, the Customer entity, but that's perfectly acceptable. Aggregate roots don't necessarily consist of at least 2 objects. Let's change the Customer domain object declaration as follows:



    public class Customer : EntityBase<int>, IAggregateRoot


The rest of the post will look at how to abstract away the data access technology and some patterns related to that.

**Unit of work**

The first important concept within data access and persistence is the Unit of Work. The purpose of a Unit of Work is to maintain a list of objects that have been modified by a transaction. These modifications include insertions, updates and deletions. The Unit of Work co-ordinates the persistence of these changes and also checks for concurrency problems, such as the same object being modified by different threads.

This may sound a bit cryptic but if you're familiar with EntityFramework or Linq to SQL then the object context in each technology is a good example for an implementation of a unit of work:



    DbContext.AddObject("Customers", new Customer());
    DbContext.SaveChanges();




    DbContext.Customers.InsertOnSubmit(new Customer());
    DbContext.SubmitChanges();


In both cases the DbContext, which takes the role of the Unit of Work, first registers the changes. The changes are not persisted until the SaveChanges() – EntityFramework – or the SubmitChanges() – Linq to SQL – method is called. Note that the DB object context is responsible for registering AND persisting the changes.

The Unit of Work pattern can be represented by the following interface:



    public interface IUnitOfWork
    {
    	void RegisterUpdate(IAggregateRoot aggregateRoot);
    	void RegisterInsertion(IAggregateRoot aggregateRoot);
    	void RegisterDeletion(IAggregateRoot aggregateRoot);
    	void Commit();
    }


Notice that we have separated the registration and commit actions. In the case of EntityFramework and similar ORM technologies the object that registers and persists the changes will often be the same: the object context or a similar object.

However, this is not always the case. It is perfectly reasonable that you are not fond of these automation tools and want to use something more basic where you have the freedom of specifying how you track changes and how you persist them: you may register the changes in memory and persist them in a file. Or you may still have a lot of legacy ADO.NET where you want to move to a more modern layered architecture. ADO.NET lacks a DbContext object so you may have to solve the registration of changes in a different way.

For those scenarios we need to introduce another abstraction. Insert the following interface in a new folder called UnitOfWork in the Infrastructure layer:



    public interface IUnitOfWorkRepository
    {
    	void PersistInsertion(IAggregateRoot aggregateRoot);
    	void PersistUpdate(IAggregateRoot aggregateRoot);
    	void PersistDeletion(IAggregateRoot aggregateRoot);
    }


Insert a new interface called IUnitOfWork in the same folder:



    public interface IUnitOfWork
    {
    	void RegisterUpdate(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository);
    	void RegisterInsertion(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository);
    	void RegisterDeletion(IAggregateRoot aggregateRoot, IUnitOfWorkRepository repository);
    	void Commit();
    }


The above Unit of Work pattern abstraction can be extended as above to give way for the total separation between the registration and persistence of aggregate roots. Later on, as we see these elements at work and as you get more acquainted with the structure of the solution, you may decide that this is overkill and you want to go with the more simple solution, it's up to you.

**Repository pattern**

The repository pattern is used to declare the possible data access related actions you want to expose for your aggregate roots. The most basic of those are CRUD operations: insert, update, delete, select. You can have other methods such as insert many, select by ID, select by id and date etc., but the CRUD operations are probably the most common across all aggregate roots.

It may be beneficial to divide those operations into two groups: read and write operations. You may have a read-only policy for some aggregate roots so you don't need to expose all of those operations. For read-only aggregates you can have a read-only repository. Insert the following interface in the Domain folder of the Infrastructure project:



    public interface IReadOnlyRepository<AggregateType, IdType> where AggregateType : IAggregateRoot
    {
    	AggregateType FindBy(IdType id);
    	IEnumerable<AggregateType> FindAll();
    }


Here again we only allow aggregate roots to be selected. Finding an object by its ID is such a basic operation that it just has to be included here. FindAll can be a bit more sensitive issue. If you have a data table with millions of rows then you may not want to expose this method as it will bog down your data server, so use it with care. It's perfectly OK to omit this operation from this interface. You will be able to include this method in domain-specific repositories as we'll see in the next post. E.g. if you want to enable the retrieval of all Customers from the database then you can expose this method in the CustomerRepository class.

The other data access operations can be exposed in an "actionable" interface. Insert the following interface in the folder of the Infrastructure project:



    public interface IRepository<AggregateType, IdType>
    		: IReadOnlyRepository<AggregateType, IdType> where AggregateType
    		: IAggregateRoot
    {
    	void Update(AggregateType aggregate);
    	void Insert(AggregateType aggregate);
    	void Delete(AggregateType aggregate);
    }


Keep in mind that these interfaces only expose the most common actions that are applicable across all aggregate roots. You'll be able to specify domain-specific actions in domain-specific implementations of these repositories.

We'll look at how these elements can be implemented for the Customer object in the next post.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/19/a-model-net-web-service-based-on-domain-driven-design-part-3-the-domain/ "A model .NET web service based on Domain Driven Design Part 3: the Domain"
[2]: http://dotnetcodr.com/2013/08/22/solid-design-principles-in-net-the-interface-segregation-principle/ "SOLID design principles in .NET: the Interface Segregation Principle"
[3]: http://dotnetcodr.com/2013/09/16/a-model-net-web-service-based-on-domain-driven-design-part-2-ddd-basics/ "A model .NET web service based on Domain Driven Design Part 2: DDD basics"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
