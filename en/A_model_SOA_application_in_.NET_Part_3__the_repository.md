[Source](http://dotnetcodr.com/2013/12/23/a-model-soa-application-in-net-part-3-the-repository/ "Permalink to A model SOA application in .NET Part 3: the repository")

# A model SOA application in .NET Part 3: the repository

In the previous post we built the thin domain layer of the model application. Now it's time to build the repository.

**The abstract repository**

Open the SoaIntroNet application we started building previously. Add a new interface in the SoaIntroNet.Domain.ProductDomain folder:



    public interface IProductRepository
    {
    	Product FindBy(Guid productId);
    	void Save(Product product);
    }


This is a very simplified repository interface but it suffices for our demo purposes. In case you're wondering what the purpose of this interface is and what it is doing in the domain layer then make sure to at least skim through the series on Domain-Driven-Design [here][1]. Alternatively you can go through the solution structure of the series about the cryptography project starting [here][2]. This latter project follows a simplified repository pattern that we'll employ here.

**The concrete repository**

The repository will be a simple in-memory repository so that we don't need to spend our time on installing external storage mechanisms.

Let's build the infrastructure around the concrete repository. Add a new C# library project called SoaIntroNet.Repository to the solution. Add a new folder called ProductRepository. Add the following stub to the class:



    public class InMemoryProductRepository : IProductRepository
    {
    	public Product FindBy(Guid productId)
    	{
    		throw new NotImplementedException();
    	}

    	public void Save(Product product)
    	{
    		throw new NotImplementedException();
    	}
    }


We'll use the thread-safe [singleton design pattern][3] to build an instance of this object. Add the following code to the class:



    public static InMemoryProductRepository Instance
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
    	internal static readonly InMemoryProductRepository instance = new InMemoryProductRepository();
    }


If you don't understand what this means then make sure to check out the post on the singleton pattern. It ensures that we always return the same instance of the concrete repository.

Before we get rid of the NotImplementException bits we'll take care of the rest of the initialisation elements. Add the following interface to the folder:



    public interface IProductRepositoryFactory
    {
    	InMemoryProductRepository Create();
    }


The following class implements the above interface:



    public class LazySingletonProductRepositoryFactory : IProductRepositoryFactory
    {
    	public InMemoryProductRepository Create()
    	{
    		return InMemoryProductRepository.Instance;
    	}
    }


We don't want to start with an empty "database" so add the following fields, properties, a constructor and an initialisation method to the class:



    private int standardReservationTimeoutMinutes = 1;
    public List&lt;Product&gt; DatabaseProducts { get; set; }
    public List&lt;ProductPurchase&gt; DatabaseProductPurchases { get; set; }
    public List&lt;ProductReservation&gt; DatabaseProductReservations { get; set; }

    public InMemoryProductRepository()
    {
    	InitialiseDatabase();
    }

    private void InitialiseDatabase()
    {
    	DatabaseProducts = new List&lt;Product&gt;();
    	Product firstProduct = new Product()
    	{
    		Allocation = 200,
    		Description = &quot;Product A&quot;,
    		ID = Guid.Parse(&quot;13a35876-ccf1-468a-88b1-0acc04422243&quot;),
    		Name = &quot;A&quot;
    	};
    	Product secondProduct = new Product()
    	{
    		Allocation = 500,
    		Description = &quot;Product B&quot;,
    		ID = Guid.Parse(&quot;f5efdfe0-7933-4efc-a290-03d20014703e&quot;),
    		Name = &quot;B&quot;
    	};
            DatabaseProducts.Add(firstProduct);
    	DatabaseProducts.Add(secondProduct);

    	DatabaseProductPurchases = new List&lt;ProductPurchase&gt;();
    	DatabaseProductPurchases.Add(new ProductPurchase(firstProduct, 10) { Id = Guid.Parse(&quot;0ede40e0-5a52-48b1-8578-de1891c5a7f0&quot;) });
    	DatabaseProductPurchases.Add(new ProductPurchase(firstProduct, 20) { Id = Guid.Parse(&quot;5868144e-e04d-4c1f-81d7-fc671bfc52dd&quot;) });
    	DatabaseProductPurchases.Add(new ProductPurchase(secondProduct, 12) { Id = Guid.Parse(&quot;8e6195ac-d448-4e28-9064-b3b1b792895e&quot;) });
    	DatabaseProductPurchases.Add(new ProductPurchase(secondProduct, 32) { Id = Guid.Parse(&quot;f66844e5-594b-44b8-a0ef-2a2064ec2f43&quot;) });
    	DatabaseProductPurchases.Add(new ProductPurchase(secondProduct, 1) { Id = Guid.Parse(&quot;0e73c8b3-f7fa-455d-ba7f-7d3f1bc2e469&quot;) });
    	DatabaseProductPurchases.Add(new ProductPurchase(secondProduct, 4) { Id = Guid.Parse(&quot;e28a3cb5-1d3e-40a1-be7e-e0fa12b0c763&quot;) });

    	DatabaseProductReservations = new List&lt;ProductReservation&gt;();
    	DatabaseProductReservations.Add(new ProductReservation(firstProduct, standardReservationTimeoutMinutes, 10) { HasBeenConfirmed = true, Id = Guid.Parse(&quot;a2c2a6db-763c-4492-9974-62ab192201fe&quot;) });
    	DatabaseProductReservations.Add(new ProductReservation(firstProduct, standardReservationTimeoutMinutes, 5) { HasBeenConfirmed = false, Id = Guid.Parse(&quot;37f2e5ac-bbe0-48b0-a3cd-9c0b47842fa1&quot;) });
    	DatabaseProductReservations.Add(new ProductReservation(firstProduct, standardReservationTimeoutMinutes, 13) { HasBeenConfirmed = true, Id = Guid.Parse(&quot;b9393ea4-6257-4dea-a8cb-b78a0c040255&quot;) });
    	DatabaseProductReservations.Add(new ProductReservation(firstProduct, standardReservationTimeoutMinutes, 3) { HasBeenConfirmed = false, Id = Guid.Parse(&quot;a70ef898-5da9-4ac1-953c-a6420d37b295&quot;) });
    	DatabaseProductReservations.Add(new ProductReservation(secondProduct, standardReservationTimeoutMinutes, 17) { Id = Guid.Parse(&quot;85eaebfa-4be4-407b-87cc-9a9ea46d547b&quot;) });
    	DatabaseProductReservations.Add(new ProductReservation(secondProduct, standardReservationTimeoutMinutes, 3) { Id = Guid.Parse(&quot;39d4278e-5643-4c43-841c-214c1c3892b0&quot;) });
    	DatabaseProductReservations.Add(new ProductReservation(secondProduct, standardReservationTimeoutMinutes, 9) { Id = Guid.Parse(&quot;86fff675-e5e3-4e0e-bcce-36332c4de165&quot;) });

    	firstProduct.PurchasedProducts = (from p in DatabaseProductPurchases where p.Product.ID == firstProduct.ID select p).ToList();
    	firstProduct.ReservedProducts = (from p in DatabaseProductReservations where p.Product.ID == firstProduct.ID select p).ToList();

    	secondProduct.PurchasedProducts = (from p in DatabaseProductPurchases where p.Product.ID == secondProduct.ID select p).ToList();
    	secondProduct.ReservedProducts = (from p in DatabaseProductReservations where p.Product.ID == secondProduct.ID select p).ToList();
    }


We have 2 products with a couple of product purchases and reservations in our data store.

The lazy singleton implementation will make sure that the initialisation code only runs once so we'll have access to the initial set of data every time we need a concrete repository.

The implementation of FindBy is very simple:



    public Product FindBy(Guid productId)
    {
    	return (from p in DatabaseProducts where p.ID == productId select p).FirstOrDefault();
    }


The implementation of Save is not much harder either:



    public void Save(Product product)
    {
    	ClearPurchasedAndReservedProducts(product);
    	InsertPurchasedProducts(product);
    	InsertReservedProducts(product);
    }


…which calls the following three private methods:



    private void ClearPurchasedAndReservedProducts(Product product)
    {
    	DatabaseProductPurchases.RemoveAll(p =&gt; p.Id == product.ID);
    	DatabaseProductReservations.RemoveAll(p =&gt; p.Id == product.ID);
    }

    private void InsertReservedProducts(Product product)
    {
    	DatabaseProductReservations.AddRange(product.ReservedProducts);
    }

    private void InsertPurchasedProducts(Product product)
    {
    	DatabaseProductPurchases.AddRange(product.PurchasedProducts);
    }


In the Save method we want to concentrate on updating the product reservations and purchases and ignore the changes in the Product domain itself. Updating product reservations and purchases line by line is a cumbersome task. It's easier to remove all existing purchases and reservations first and insert the old ones along with the new ones. The existing product purchases and reservations are always populated correctly in the InitialiseDatabase() method. Therefore a call to FindBy will product a product with the existing purchases and reservations.

So we now have the Domain and the Repository layer ready – we'll build the service layer in the next solution.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[2]: http://dotnetcodr.com/2013/11/25/an-encrypted-messaging-project-in-net-with-c-part-1-foundations/ "An encrypted messaging project in .NET with C# part 1: foundations"
[3]: http://dotnetcodr.com/2013/05/09/design-patterns-and-practices-in-net-the-singleton-pattern/ "Design patterns and practices in .NET: the Singleton pattern"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
