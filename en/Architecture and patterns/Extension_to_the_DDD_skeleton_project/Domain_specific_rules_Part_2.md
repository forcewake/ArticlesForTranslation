[Source](http://dotnetcodr.com/2014/03/20/extension-to-the-ddd-skeleton-project-domain-specific-rules-part-2/ "Permalink to Extension to the DDD skeleton project: domain specific rules Part 2")

# Extension to the DDD skeleton project: domain specific rules Part 2

**Introduction**

In the [first part][1] of this DDD skeleton project extension we completed the domain and repository layers. The repository implementation was only a stub as we want to concentrate on validating domains with differing regional rules.

In this post we'll finish off the communication chain in the skeleton web service.

Let's get started! Open the DDD project and we'll continue with the application service layer.

**The Application services layer**

We'll concentrate on inserting new CountrySpecificCustomer objects in the data store. We won't support the entire GET/POST/PUT/DELETE spectrum. You can do all that based on how we did it for the simple Customer object. Locate the Messaging folder and insert a new sub-folder called EnhancedCustomers. Insert the following messaging objects in this new folder:



    public class CountrySpecificCustomerViewModel
    {
    	public string FirstName { get; set; }
    	public int Age { get; set; }
    	public string NickName { get; set; }
    	public string Email { get; set; }
    	public string CountryCode { get; set; }
    }

    public class InsertCountrySpecificCustomerRequest : ServiceRequestBase
    {
    	public CountrySpecificCustomerViewModel CountrySpecificCustomer { get; set; }
    }

    public class InsertCountrySpecificCustomerResponse : ServiceResponseBase
    {}


Nothing fancy here I hope. These objects follow the same structure as their original Customer counterparts.

Add the following interface in the Interfaces folder:



    public interface ICountrySpecificCustomerService
    {
    	InsertCountrySpecificCustomerResponse InsertCountrySpecificCustomer(InsertCountrySpecificCustomerRequest insertCustomerRequest);
    }


Recall that the constructor of the CountrySpecificCustomer object is expecting an instance of Country. We already have a factory in the Domain layer that returns the correct country based on the country code. So we *could* use that factory here as well since the CountrySpecificCustomerViewModel allows for the specification of a country code. However, it would be unfortunate to use that concrete factory in the Services layer as it would only increase the degree of coupling between the Services and the Domain layer. So we'll hide it behind an abstraction.

Locate the folder where you saved the CountryFactory object in the Domain layer. Insert the following interface:



    public interface ICountryFactory
    {
    	Country CreateCountry(string countryCode);
    }


The existing CountryFactory can implement the interface as follows:



    public class CountryFactory : ICountryFactory
    {
    	private static IEnumerable<Country> AllCountries()
    	{
    		return new List<Country>() { new Hungary(), new Germany(), new Sweden() };
    	}

    	public static Country Create(string countryCode)
    	{
    		return (from c in AllCountries() where c.CountryCode.ToLower() == countryCode.ToLower() select c).FirstOrDefault();
    	}

    	public Country CreateCountry(string countryCode)
    	{
    		return Create(countryCode);
    	}
    }


Back in the services layer we can implement the ICountrySpecificCustomerService interface. Just to refresh our minds open the CustomerService class in the Implementations folder. You'll recall that its constructor requires an IUnitOfWork object. The implementation of the ICountrySpecificCustomerService will also need one. This calls for a base class which both implementations can derive from. Insert the following abstract class in the Implementations folder:



    public abstract class ApplicationServiceBase
    {
    	private readonly IUnitOfWork _unitOfWork;

    	public ApplicationServiceBase(IUnitOfWork unitOfWork)
    	{
    		if (unitOfWork == null) throw new ArgumentNullException("UnitOfWork");
    		_unitOfWork = unitOfWork;
    	}

    	public IUnitOfWork UnitOfWork
    	{
    		get
    		{
    			return _unitOfWork;
    		}
    	}
    }


Next modify the declaration and constructor of the CustomerService object as follows:



    public class CustomerService : ApplicationServiceBase, ICustomerService
    {
    	private readonly ICustomerRepository _customerRepository;

    	public CustomerService(ICustomerRepository customerRepository, IUnitOfWork unitOfWork)
    		: base(unitOfWork)
    	{
    		if (customerRepository == null) throw new ArgumentNullException("Customer repo");
    		_customerRepository = customerRepository;
    	}
    .//implementations omitted
    .
    .
    }


After this change you'll get some errors saying that the _unitOfWork variable is not accessible, e.g. here:



    _unitOfWork.Commit();


Replace those calls by referring to the public accessor of the base class:



    UnitOfWork.Commit();


Add a new class to the Implementations folder called CountrySpecificCustomerService. Insert the following implementation stub:



    public class CountrySpecificCustomerService : ApplicationServiceBase, ICountrySpecificCustomerService
    {
    	private readonly ICountrySpecificCustomerRepository _repository;
    	private readonly ICountryFactory _countryFactory;

    	public CountrySpecificCustomerService(IUnitOfWork unitOfWork, ICountrySpecificCustomerRepository repository
    		, ICountryFactory countryFactory) : base(unitOfWork)
    	{
    		if (repository == null) throw new ArgumentNullException("CountrySpecificCustomerRepository");
    		if (countryFactory == null) throw new ArgumentNullException("ICountryFactory");
    		_repository = repository;
    		_countryFactory = countryFactory;
    	}

    	public InsertCountrySpecificCustomerResponse InsertCountrySpecificCustomer(InsertCountrySpecificCustomerRequest insertCustomerRequest)
    	{
    		throw new NotImplementedException();
    	}
    }


In the InsertCountrySpecificCustomer implementation we'll follow the same structure as in CustomerService.InsertCustomer: construct the domain object, validate it and then insert it. Insert the following implementation:



    public InsertCountrySpecificCustomerResponse InsertCountrySpecificCustomer(InsertCountrySpecificCustomerRequest insertCustomerRequest)
    {
    	CountrySpecificCustomer newCustomer = BuildCountrySpecificCustomer(insertCustomerRequest.CountrySpecificCustomer);
    	ThrowExceptionIfCustomerIsInvalid(newCustomer);
    	try
    	{
    		_repository.Insert(newCustomer);
    		UnitOfWork.Commit();
    		return new InsertCountrySpecificCustomerResponse();
    	}
    	catch (Exception ex)
    	{
    		return new InsertCountrySpecificCustomerResponse() { Exception = ex };
    	}
    }


BuildCountrySpecificCustomer and ThrowExceptionIfCustomerIsInvalid look as follows:



    private CountrySpecificCustomer BuildCountrySpecificCustomer(CountrySpecificCustomerViewModel viewModel)
    {
    	return new CountrySpecificCustomer(_countryFactory.CreateCountry(viewModel.CountryCode))
    	{
    		Age = viewModel.Age
    		, Email = viewModel.Email
    		, FirstName = viewModel.FirstName
    		, NickName = viewModel.NickName
    	};
    }

    private void ThrowExceptionIfCustomerIsInvalid(CountrySpecificCustomer newCustomer)
    {
    	IEnumerable<BusinessRule> brokenRules = newCustomer.GetBrokenRules();
    	if (brokenRules.Count() > 0)
    	{
    		StringBuilder brokenRulesBuilder = new StringBuilder();
    		brokenRulesBuilder.AppendLine("There were problems saving the LoadtestPortalCustomer object:");
    		foreach (BusinessRule businessRule in brokenRules)
    		{
    			brokenRulesBuilder.AppendLine(businessRule.RuleDescription);
    		}

    		throw new Exception(brokenRulesBuilder.ToString());
    	}
    }


**New elements in the Web layer**

We're now ready to construct our new Controller. Locate the Controllers folder in the WebService project of the solution. Add a new empty ApiController called CountrySpecificCustomersController. Add the following private field and constructor:



    private readonly ICountrySpecificCustomerService _countrySpecificCustomerService;

    public CountrySpecificCustomersController(ICountrySpecificCustomerService countrySpecificCustomerService)
    {
    	if (countrySpecificCustomerService == null) throw new ArgumentNullException("CountrySpecificCustomerService");
    	_countrySpecificCustomerService = countrySpecificCustomerService;
    }


We'll only support the POST operation so that we can concentrate on validation. Add the following Post method to the controller:



    public HttpResponseMessage Post(CountrySpecificCustomerViewModel viewModel)
    {
    	ServiceResponseBase response = _countrySpecificCustomerService.InsertCountrySpecificCustomer(new InsertCountrySpecificCustomerRequest() { CountrySpecificCustomer = viewModel });
    	return Request.BuildResponse(response);
    }


That's it actually, we can now test the new parts using the simple Console tool we built in the [last post of the DDD series][2]. Start the DDD web service and take note of the localhost URL in the web browser. Insert a breakpoint within the Post method of the CountrySpecificCustomersController class so that you can go through the code step by step.

Open that little test application and add the following class to the main project:



    public class CountrySpecificCustomerViewModel
    {
    	public string FirstName { get; set; }
    	public int Age { get; set; }
    	public string NickName { get; set; }
    	public string Email { get; set; }
    	public string CountryCode { get; set; }
    }


Insert two methods that will build a valid and an invalid country specific customer:



    private static CountrySpecificCustomerViewModel BuildFailingCustomer()
    {
    	return new CountrySpecificCustomerViewModel()
    	{
    		FirstName = "Elvis"
    		, Age = 17
    		, CountryCode = "SWE"
    	};
    }

    private static CountrySpecificCustomerViewModel BuildPassingCustomer()
    {
    	return new CountrySpecificCustomerViewModel()
    	{
    		FirstName = "John"
    		, Age = 19
    		, CountryCode = "SWE"
    	};
    }


Recall that Swedish customers must be over 18, so the insertion of the first object should fail. Insert the following test method to Program.cs:



    private static void TestCountrySpecificCustomerValidation()
    {
    	HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Post, _serviceUriTwo);
    	requestMessage.Headers.ExpectContinue = false;
    	CountrySpecificCustomerViewModel vm = BuildFailingCustomer();
    	string jsonInputs = JsonConvert.SerializeObject(vm);
    	requestMessage.Content = new StringContent(jsonInputs, Encoding.UTF8, "application/json");
    	HttpClient httpClient = new HttpClient();
    	httpClient.Timeout = new TimeSpan(0, 10, 0);
    	Task<HttpResponseMessage> httpRequest = httpClient.SendAsync(requestMessage,
    			HttpCompletionOption.ResponseContentRead, CancellationToken.None);
    	HttpResponseMessage httpResponse = httpRequest.Result;
    	HttpStatusCode statusCode = httpResponse.StatusCode;
    	HttpContent responseContent = httpResponse.Content;
    	if (responseContent != null)
    	{
    		Task<String> stringContentsTask = responseContent.ReadAsStringAsync();
    		String stringContents = stringContentsTask.Result;
    		Console.WriteLine("Response from service: " + stringContents);
    	}
    	Console.ReadKey();
    }


…where _serviceUriTwo is the URL of the controller. It should be something like this:



    private static Uri _serviceUriTwo = new Uri("http://localhost:9985/countryspecificcustomers");


So we'll be testing the failing customer first. Insert a call to this test method from Main and press F5. If you entered the breakpoint mentioned above then the code execution should stop there. Step through the code and when it hits the validation part in the application service layer – the ThrowExceptionIfCustomerIsInvalid method – you should see a validation error: There were problems saving the LoadtestPortalCustomer object: Swedish customers must be at least 18.

Next replace the following…



    CountrySpecificCustomerViewModel vm = BuildFailingCustomer();


…with…



    CountrySpecificCustomerViewModel vm = BuildPassingCustomer();


…within the TestCountrySpecificCustomerValidation() method and run the test again. Validation should succeed without any problems.

Feel free to test the code with the other cases we set up in the original specifications by playing with the property values of the CountrySpecificCustomerViewModel object we send to the service.

That's all folks. We'll look at caching in the [next extension of the DDD project][3].

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/03/17/extension-to-the-ddd-skeleton-project-domain-specific-rules-part-1/ "Extension to the DDD skeleton project: domain specific rules Part 1"
[2]: http://dotnetcodr.com/2013/10/14/a-model-net-web-service-based-on-domain-driven-design-part-10-tests-and-conclusions/ "A model .NET web service based on Domain Driven Design Part 10: tests and conclusions"
[3]: http://dotnetcodr.com/2014/03/24/extension-to-the-ddd-skeleton-project-caching-in-the-service-layer/ "Extension to the DDD skeleton project: caching in the service layer"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
