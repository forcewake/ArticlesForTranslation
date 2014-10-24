[Source](http://dotnetcodr.com/2013/11/28/an-encrypted-messaging-project-in-net-with-c-part-2-service-and-repository/ "Permalink to An encrypted messaging project in .NET with C# part 2: service and repository")

# An encrypted messaging project in .NET with C# part 2: service and repository

**Introduction**

In the previous post we started building the foundations of an encrypted messaging project. To be exact we built a couple of objects in the Infrastructure layer to support the creation of new RSA key pairs.

We'll now look at how the asymmetric key-pair can be stored, located and removed in a repository.

**Repository**

Insert a new class library called Receiver.Repository to the Receiver solution and add a reference to the Infrastructure layer. Add a new folder called "Cryptography". The exact way how a key-pair can be stored can vary a lot: in a database, in memory, in a file, on a separate machine, etc. You can read more about key storage [here][1]. Therefore it sounds like a good idea to hide the implementation behind an interface. Add the following interface to the folder:



    public interface IAsymmetricKeyRepository
    {
    	void Add(Guid messageId, AsymmetricKeyPairGenerationResult asymmetricKeyPair);
    	AsymmetricKeyPairGenerationResult FindBy(Guid messageId);
    	void Remove(Guid messageId);
    }


We saw the AsymmetricKeyPairGenerationResult object in the previous post. The message ID, as discussed before, will be used to check the validity of the key supplied by the sender. We also need to be able to locate a key-pair by the message ID and remove a message id after it's been used.

We'll go with a simple in-memory implementation to make things simple. The repository will be instantiated using the [singleton][2] design pattern. Again, as the implementation is hidden behind an interface you can easily change your key storage strategy later on.

Add the following class to the folder:



    public class InMemoryAsymmetricKeyRepository : IAsymmetricKeyRepository
    {
    	private Dictionary<Guid, AsymmetricKeyPairGenerationResult> _asymmetricKeyPairs;

    	public InMemoryAsymmetricKeyRepository()
    	{
    		_asymmetricKeyPairs = new Dictionary<Guid, AsymmetricKeyPairGenerationResult>();
    	}

    	public void Add(Guid messageId, AsymmetricKeyPairGenerationResult asymmetricKeyPair)
    	{
    		_asymmetricKeyPairs[messageId] = asymmetricKeyPair;
    	}

    	public AsymmetricKeyPairGenerationResult FindBy(Guid messageId)
    	{
    		if (_asymmetricKeyPairs.ContainsKey(messageId))
    		{
    			return _asymmetricKeyPairs[messageId];
    		}
    		throw new KeyNotFoundException("Invalid message ID.");
    	}

    	public void Remove(Guid messageId)
    	{
    		if (_asymmetricKeyPairs.ContainsKey(messageId))
    		{
    			_asymmetricKeyPairs.Remove(messageId);
    		}
    	}

            public static InMemoryAsymmetricKeyRepository Instance
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
    		internal static readonly InMemoryAsymmetricKeyRepository instance = new InMemoryAsymmetricKeyRepository();
    	}
    }


If you don't understand what the Nested class and the Instance property mean then make sure to check out the link provided above on the Singleton design pattern.

We store the message ID and its asymmetric key-pair in a Dictionary. I believe it's easy to follow the implementation: we add, find and remove elements in the dictionary.

We'll need a couple of extra classes in order to get an instance of the repository. Add the following classes into the Cryptography folder:



    public interface IAsymmetricKeyRepositoryFactory
    {
    	InMemoryAsymmetricKeyRepository Create();
    }




    public class LazySingletonAsymmetricKeyRepositoryFactory : IAsymmetricKeyRepositoryFactory
    {
    	public InMemoryAsymmetricKeyRepository Create()
    	{
    		return InMemoryAsymmetricKeyRepository.Instance;
    	}
    }


Check out [this][3] post to gain a full understanding on the purpose of the abstract factory and its implementation. We want to make sure that only a single instance of the InMemoryAsymmetricKeyRepository object is returned. We don't want different sets of the _asymmetricKeyPairs dictionary fly around every time we need an InMemoryAsymmetricKeyRepository.

We're actually done with the repository layer so let's move on.

**Application services**

If you've read through [this][4] series then you'll know what's the purpose of the application service layer. In short it's the thin connecting tissue between the consumer layer – MVC, web service, console and the like – and the backend parts. It should be void of any logic. It co-ordinates the tasks among its dependencies in order to respond to the consumer layer in a meaningful way. Make sure you read at least [this][5] post so that you become familiar with things like Request-Response and the ServiceResponseBase object. Those techniques will be reused here.

Add a new C# library called Receiver.ApplicationService to the solution. Add a folder called Interfaces. The first service we implement will need to provide a valid and unused pair of keys and a message ID to the consumer. Insert the following interface into the folder:



    public interface IAsymmetricCryptographyApplicationService
    {
    	GetAsymmetricPublicKeyResponse GetAsymmetricPublicKey();
    }


…where GetAsymmetricPublicKeyResponse is a simple wrapper. Add a new folder called Responses and in it add this object:



    public class GetAsymmetricPublicKeyResponse : ServiceResponseBase
    {
    	public Guid MessageId { get; set; }
    	public XDocument PublicKeyXml { get; set; }
    }


The ServiceResponseBase is located in the Responses folder as well:



    public abstract class ServiceResponseBase
    {
    	public ServiceResponseBase()
    	{
    		this.Exception = null;
    	}

    	/// <summary>
    	/// Save the exception thrown so that consumers can read it
    	/// </summary>
    	public Exception Exception { get; set; }
    }


Add a new folder called Implementations to the service layer. In it add the following implementation of the above interface:



    public class AsymmetricCryptographyApplicationService : IAsymmetricCryptographyApplicationService
    {
    	private readonly IAsymmetricCryptographyService _cryptographyInfrastructureService;
    	private readonly IAsymmetricKeyRepositoryFactory _asymmetricKeyRepositoryFactory;

    	public AsymmetricCryptographyApplicationService(IAsymmetricCryptographyService cryptographyInfrastructureService
    		, IAsymmetricKeyRepositoryFactory asymmetricKeyRepositoryFactory)
    	{
    		if (cryptographyInfrastructureService == null) throw new ArgumentNullException("IAsymmetricCryptographyService");
    		if (asymmetricKeyRepositoryFactory == null) throw new ArgumentNullException("IAsymmetricKeyRepositoryFactory");
    		_cryptographyInfrastructureService = cryptographyInfrastructureService;
    		_asymmetricKeyRepositoryFactory = asymmetricKeyRepositoryFactory;
    	}

    	public GetAsymmetricPublicKeyResponse GetAsymmetricPublicKey()
    	{
    		GetAsymmetricPublicKeyResponse publicKeyResponse = new GetAsymmetricPublicKeyResponse();
    		try
    		{
    			AsymmetricKeyPairGenerationResult asymmetricKeyPair = _cryptographyInfrastructureService.GenerateAsymmetricKeys();
    			Guid messageId = Guid.NewGuid();
    			publicKeyResponse.MessageId = messageId;
    			publicKeyResponse.PublicKeyXml = asymmetricKeyPair.PublicKeyOnlyXml;
    			_asymmetricKeyRepositoryFactory.Create().Add(messageId, asymmetricKeyPair);
    		}
    		catch (Exception ex)
    		{
    			publicKeyResponse.Exception = ex;
    		}
    		return publicKeyResponse;
    	}
    }


The implementation will need an IAsymmetricCryptographyService and an IAsymmetricKeyRepositoryFactory object to fulfil its job. It doesn't care which exact implementations are injected through its constructor. We set some guard clauses in the constructor so that the injected dependencies won't be null.

Take a look at the GetAsymmetricPublicKey() implementation. It first asks the crypto service in the infrastructure layer to generate the asymmetric keys. It then constructs a new GUID which will serve as the message ID, sets the appropriate properties of the GetAsymmetricPublicKeyResponse object and finally asks the repository to save the key pair and the message ID through the Create() method of the repository factory. We save any exceptions that may have been thrown along the way and return the response.

If you followed through the DDD project mentioned above, the next object will look familiar. Add a new folder called Exceptions to the project and insert the following custom Exception:



    public class ResourceNotFoundException : Exception
    {
    	public ResourceNotFoundException(string message)
    		: base(message)
    	{ }

    	public ResourceNotFoundException()
    		: base("The requested resource was not found.")
    	{ }
    }


This will be used in cases where the resource requested by the Sender is not found by the Receiver.

This completes our service layer for the time being. In the next post we'll start building the web service layer.

You can view the list of posts on Security and Cryptography [here][6].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/11/21/key-size-and-key-storage-in-net-cryptography/ "Key size and key storage in .NET cryptography"
[2]: http://dotnetcodr.com/2013/05/09/design-patterns-and-practices-in-net-the-singleton-pattern/ "Design patterns and practices in .NET: the Singleton pattern"
[3]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[4]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[5]: http://dotnetcodr.com/2013/10/03/a-model-net-web-service-based-on-domain-driven-design-part-7-the-abstract-service/ "A model .NET web service based on Domain Driven Design Part 7: the abstract Service"
[6]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
