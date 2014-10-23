[Source](http://dotnetcodr.com/2013/12/26/a-model-soa-application-in-net-part-3-the-service-layer-part-1/ "Permalink to A model SOA application in .NET Part 4: the service layer part 1")

# A model SOA application in .NET Part 4: the service layer part 1

**Introduction**

Now that we're done with the domain and repository layers it's time to build the service layer. If you are not familiar with the purposes of the service layer then make sure to read [this][1] post. I'll employ the same Request-Response pattern here.

**The service layer**

Open the application we've been working on and add a new C# class library called SoaIntroNet.Service. Remove Class1.cs and make a reference to both the Domain and the Repository layers. Insert a new folder called Responses and in it a base class for all service responses:



    public abstract class ServiceResponseBase
    {
    	public ServiceResponseBase()
    	{
    		this.Exception = null;
    	}

            public Exception Exception { get; set; }
    }


The clients will be able to purchase and reserve products. We put the result of the purchasing and reservation processes into the following two Response objects:



    public class PurchaseProductResponse : ServiceResponseBase
    {
    	public string PurchaseId { get; set; }
    	public string ProductName { get; set; }
    	public string ProductId { get; set; }
    	public int ProductQuantity { get; set; }
    }




    public class ProductReservationResponse : ServiceResponseBase
    {
    	public string ReservationId { get; set; }
    	public DateTime Expiration { get; set; }
    	public string ProductId { get; set; }
    	public string ProductName { get; set; }
    	public int ProductQuantity { get; set; }
    }


Nothing really fancy up to this point I hope. In order to make a purchase or reservation the corresponding Request objects must be provided by the client. Add a new folder called Requests and in it the following two requests:



    public class PurchaseProductRequest
    {
    	public string CorrelationId { get; set; }
    	public string ReservationId { get; set; }
    	public string ProductId { get; set; }
    }




    public class ReserveProductRequest
    {
    	public string ProductId { get; set; }
    	public int ProductQuantity { get; set; }
    }


The only somewhat unexpected property is the correlation id. Recall its purpose from the [opening post][2] in this series: ensure that a state-changing operation is not carried out more than once, e.g. purchasing the same tickets twice.

Add a folder called Exceptions and insert the following two custom exceptions:



    public class ResourceNotFoundException : Exception
    {
    	public ResourceNotFoundException(string message)
    		: base(message)
    	{ }

    	public ResourceNotFoundException()
    		: base(&quot;The requested resource was not found.&quot;)
    	{ }
    }




    public class LimitedAvailabilityException : Exception
    {
    	public LimitedAvailabilityException(string message)
    		: base(message)
    	{}

    	public LimitedAvailabilityException()
    		: base(&quot;There are not enough products left to fulfil your request.&quot;)
    	{}
    }


As the names imply we'll throw these exception if some requested resource could not be located or that the required amount in the request could not be fulfilled.

Add the following service interface to the Service layer:



    public interface IProductService
    {
    	ProductReservationResponse ReserveProduct(ReserveProductRequest productReservationRequest);
    	PurchaseProductResponse PurchaseProduct(PurchaseProductRequest productPurchaseRequest);
    }


This represents our service contract. We state that our service will be able to handle product purchases and reservations and they need the necessary Request objects in order to fulfil their functions. The concrete implementation – which we'll look at in a bit – is an unimportant implementation detail from the client's point of view.

However, before we do that we'll need to revisit the repository layer and make room for saving and retrieving the messaging history based on the correlation ID. These messages are not part of our domain so we won't bother with setting up a separate Message object.

Open SoaIntroNet.Repository and add a new folder called MessagingHistory. Insert the following interface which describes the operations a message repository must be able to handle:



    public interface IMessageRepository
    {
    	bool IsUniqueRequest(string correlationId);
    	void SaveResponse&lt;T&gt;(string correlationId, T response);
    	T RetrieveResponseFor&lt;T&gt;(string correlationId);
    }


The purpose of IsUniqueRequest is to show whether the message with that correlation ID has been saved before. You can probably guess the purpose of the other two methods.

Here comes an implementation of the repository with the same lazy loading static initialiser we saw in the ProductRepository example:



    public class MessageRepository : IMessageRepository
    {
    	private Dictionary&lt;string, object&gt; _responseHistory;

    	public MessageRepository()
    	{
    		_responseHistory = new Dictionary&lt;string, object&gt;();
    	}

    	public bool IsUniqueRequest(string correlationId)
    	{
    		return !_responseHistory.ContainsKey(correlationId);
    	}

    	public void SaveResponse&lt;T&gt;(string correlationId, T response)
    	{
    		_responseHistory[correlationId] = response;
    	}

    	public T RetrieveResponseFor&lt;T&gt;(string correlationId)
    	{
    		if (_responseHistory.ContainsKey(correlationId))
    		{
    			return (T)_responseHistory[correlationId];
    		};
    		return default(T);
    	}

    	public static MessageRepository Instance
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
    		internal static readonly MessageRepository instance = new MessageRepository();
    	}
    }


We store the responses in a Dictionary. In a real case scenario we can store these in any type of storage mechanism: a database, a web service, cache, you name it. An in-memory solution is the easiest in this case.

**A side note**: it's not necessary to go with the lazy singleton pattern for all repositories. However, in this example the pattern will make sure that a single object is created every time one is required and the in-memory objects won't be re-instantiated. We don't want to lose the Dictionary object every time somebody sends a new purchase request. In the series on DDD, especially in [this][3] post and the one after that, I show how it suffices to lazily instantiate a Unit of Work instead of all implemented repositories.

Just like for the ProductRepository we'll need the corresponding factory:



    public interface IMessageRepositoryFactory
    {
    	MessageRepository Create();
    }




    public class LazySingletonMessageRepositoryFactory : IMessageRepositoryFactory
    {
    	public MessageRepository Create()
    	{
    		return MessageRepository.Instance;
    	}
    }


We can now turn our attention to the Service layer again. Add a class called ProductService with the following stub:



    public class ProductService : IProductService
    {
    	public ProductReservationResponse ReserveProduct(ReserveProductRequest productReservationRequest)
    	{
    		throw new NotImplementedException();
    	}

    	public PurchaseProductResponse PurchaseProduct(PurchaseProductRequest productPurchaseRequest)
    	{
    		throw new NotImplementedException();
    	}
    }


The ProductService will need some help from the Repository layer to complete these tasks. It will need a product repository and a message repository. These objects are created by our lazy singleton factories. We know from the discussion on [SOLID][4] and especially the [Dependency inversion principle][5] that an object – in this case the ProductService – should not create its own dependencies. Instead, the caller should inject the proper dependencies.

The most obvious technique is [constructor injection][6]. Add the following private fields and a constructor to ProductService:



    private readonly IMessageRepositoryFactory _messageRepositoryFactory;
    private readonly IProductRepositoryFactory _productRepositoryFactory;
    private readonly IMessageRepository _messageRepository;
    private readonly IProductRepository _productRepository;

    public ProductService(IMessageRepositoryFactory messageRepositoryFactory, IProductRepositoryFactory productRepositoryFactory)
    {
    	if (messageRepositoryFactory == null) throw new ArgumentNullException(&quot;MessageRepositoryFactory&quot;);
    	if (productRepositoryFactory == null) throw new ArgumentNullException(&quot;ProductRepositoryFactory&quot;);
    	_messageRepositoryFactory = messageRepositoryFactory;
    	_productRepositoryFactory = productRepositoryFactory;
    	_messageRepository = _messageRepositoryFactory.Create();
    	_productRepository = _productRepositoryFactory.Create();
    }


We let the factories be injected through the constructor. This follows the good habit of programming against abstractions. We then ask the factories to create the repositories for us – note that they are abstractions as well.

In a real application you wouldn't rely on the memory to store objects for you. A more "normal" constructor would look like this:



    private readonly IMessageRepository _messageRepository;
    private readonly IProductRepository _productRepository;

    public ProductService(IMessageRepository messageRepository, IProductRepository productRepository)
    {
    	if (messageRepository == null) throw new ArgumentNullException(&quot;MessageRepository&quot;);
    	if (productRepository == null) throw new ArgumentNullException(&quot;ProductRepository&quot;);
    	_messageRepository = messageRepository;
    	_productRepository = productRepository;
    }


However, for the reasons outlined above we go with the factory solution in this demo. If we followed this simplified version then our in-memory data would be re-instantiated after every subsequent request making a demo meaningless.

We'll see in the next post how the reserve ticket and purchase ticket methods are implemented.

View the list of posts on Architecture and Patterns [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/10/03/a-model-net-web-service-based-on-domain-driven-design-part-7-the-abstract-service/ "A model .NET web service based on Domain Driven Design Part 7: the abstract Service"
[2]: http://dotnetcodr.com/2013/12/16/a-model-soa-application-in-net-part-1-the-fundamentals/ "A model SOA application in .NET Part 1: the fundamentals"
[3]: http://dotnetcodr.com/2013/09/26/a-model-net-web-service-based-on-domain-driven-design-part-5-the-concrete-repository/ "A model .NET web service based on Domain Driven Design Part 5: the concrete Repository"
[4]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[5]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[6]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[7]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
