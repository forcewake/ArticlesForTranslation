[Source](http://dotnetcodr.com/2013/10/07/a-model-net-web-service-based-on-domain-driven-design-part-8-the-concrete-service/ "Permalink to A model .NET web service based on Domain Driven Design Part 8: the concrete Service")

# A model .NET web service based on Domain Driven Design Part 8: the concrete Service

We'll continue where we left off in the [previous][1] post. It's time to implement the first concrete service in the skeleton application: the CustomerService.

Open the project we've been working on in this series. Locate the ApplicationServices layer and add a new folder called Implementations. Add a new class called CustomerService which implements the ICustomerService interface we inserted in the previous post. The initial skeleton will look like this:



    public class CustomerService : ICustomerService
    {
    	public GetCustomerResponse GetCustomer(GetCustomerRequest getCustomerRequest)
    	{
    		throw new NotImplementedException();
    	}

    	public GetCustomersResponse GetAllCustomers()
    	{
    		throw new NotImplementedException();
    	}

    	public InsertCustomerResponse InsertCustomer(InsertCustomerRequest insertCustomerRequest)
    	{
    		throw new NotImplementedException();
    	}

    	public UpdateCustomerResponse UpdateCustomer(UpdateCustomerRequest updateCustomerRequest)
    	{
    		throw new NotImplementedException();
    	}

    	public DeleteCustomerResponse DeleteCustomer(DeleteCustomerRequest deleteCustomerRequest)
    	{
    		throw new NotImplementedException();
    	}
    }


We know that the service will need some repository to retrieve the requested records. Which repository is it? The abstract one of course: ICustomerRepository. It represents all operations that the consumer is allowed to do in the customer repository. The service layer doesn't care about the exact implementation of this interface.

We'll also need a reference to the unit of work which will maintain and persist the changes we make. Again, we'll take the abstract IUnitOfWork object.

These abstractions must be injected into the customer service class through its constructor. You can read about constructor injection and the other types of dependency injection [here][2].

Add the following backing fields and the constructor to CustomerService.cs:



    private readonly ICustomerRepository _customerRepository;
    private readonly IUnitOfWork _unitOfWork;

    public CustomerService(ICustomerRepository customerRepository, IUnitOfWork unitOfWork)
    {
    	if (customerRepository == null) throw new ArgumentNullException("Customer repo");
    	if (unitOfWork == null) throw new ArgumentNullException("Unit of work");
    	_customerRepository = customerRepository;
    	_unitOfWork = unitOfWork;
    }


Let's implement the GetCustomer method first:



    public GetCustomerResponse GetCustomer(GetCustomerRequest getCustomerRequest)
    {
    	GetCustomerResponse getCustomerResponse = new GetCustomerResponse();
    	Customer customer = null;
    	try
    	{
    		customer = _customerRepository.FindBy(getCustomerRequest.Id);
    		if (customer == null)
    		{
    			getCustomerResponse.Exception = GetStandardCustomerNotFoundException();
    		}
                    else
    		{
    			getCustomerResponse.Customer = customer.ConvertToViewModel();
    		}
    	}
    	catch (Exception ex)
    	{
    		getCustomerResponse.Exception = ex;
    	}
    	return getCustomerResponse;
    }


…where GetStandardCustomerNotFoundException() looks like this:



    private ResourceNotFoundException GetStandardCustomerNotFoundException()
    {
    	return new ResourceNotFoundException("The requested customer was not found.");
    }


…where ResourceNotFoundException looks like the following:



    public class ResourceNotFoundException : Exception
    {
    	public ResourceNotFoundException(string message)
    		: base(message)
    	{}

    	public ResourceNotFoundException()
    		: base("The requested resource was not found.")
    	{}
    }


There's nothing too complicated in the GetCustomer method I hope. Note that we use the extension method ConvertToViewModel() we implemented in the previous post to return a customer view model. We call upon the FindBy method of the repository to locate the resource and save any exception thrown along the way.

GetAllCustomers is equally simple:



    public GetCustomersResponse GetAllCustomers()
    {
    	GetCustomersResponse getCustomersResponse = new GetCustomersResponse();
    	IEnumerable<Customer> allCustomers = null;

    	try
    	{
    		allCustomers = _customerRepository.FindAll();
                    getCustomersResponse.Customers = allCustomers.ConvertToViewModels();
    	}
    	catch (Exception ex)
    	{
    		getCustomersResponse.Exception = ex;
    	}
    	return getCustomersResponse;
    }


In the InsertCustomer method we create a new Customer domain object, validate it, call the repository to insert the object and finally call the unit of work to commit the changes:



    public InsertCustomerResponse InsertCustomer(InsertCustomerRequest insertCustomerRequest)
    {
    	Customer newCustomer = AssignAvailablePropertiesToDomain(insertCustomerRequest.CustomerProperties);
    	ThrowExceptionIfCustomerIsInvalid(newCustomer);
    	try
    	{
    		_customerRepository.Insert(newCustomer);
    		_unitOfWork.Commit();
    		return new InsertCustomerResponse();
    	}
    	catch (Exception ex)
    	{
    		return new InsertCustomerResponse() { Exception = ex };
    	}
    }


…where AssignAvailablePropertiesToDomain looks like this:



    private Customer AssignAvailablePropertiesToDomain(CustomerPropertiesViewModel customerProperties)
    {
    	Customer customer = new Customer();
    	customer.Name = customerProperties.Name;
    	Address address = new Address();
    	address.AddressLine1 = customerProperties.AddressLine1;
    	address.AddressLine2 = customerProperties.AddressLine2;
    	address.City = customerProperties.City;
    	address.PostalCode = customerProperties.PostalCode;
    	customer.CustomerAddress = address;
    	return customer;
    }


So we simply dress up a new Customer domain object based on the properties of the incoming CustomerPropertiesViewModel object. In the ThrowExceptionIfCustomerIsInvalid method we validate the Customer domain:



    private void ThrowExceptionIfCustomerIsInvalid(Customer newCustomer)
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


Revisit the post on EntityBase and the Domain layer if you've forgotten what the BusinessRule object and the GetBrokenRules() method are about.

In the UpdateCustomer method we first check if the requested Customer object exists. Then we change its properties based on the incoming UpdateCustomerRequest object. The process after that is the same as in the case of InsertCustomer:



    public UpdateCustomerResponse UpdateCustomer(UpdateCustomerRequest updateCustomerRequest)
    {
    	try
    	{
    		Customer existingCustomer = _customerRepository.FindBy(updateCustomerRequest.Id);
    		if (existingCustomer != null)
    		{
    			Customer assignableProperties = AssignAvailablePropertiesToDomain(updateCustomerRequest.CustomerProperties);
    			existingCustomer.CustomerAddress = assignableProperties.CustomerAddress;
    			existingCustomer.Name = assignableProperties.Name;
    			ThrowExceptionIfCustomerIsInvalid(existingCustomer);
    			_customerRepository.Update(existingCustomer);
    			_unitOfWork.Commit();
    			return new UpdateCustomerResponse();
    		}
    		else
    		{
    			return new UpdateCustomerResponse() { Exception = GetStandardCustomerNotFoundException() };
    		}
    	}
    	catch (Exception ex)
    	{
    		return new UpdateCustomerResponse() { Exception = ex };
    	}
    }


In the DeleteCustomer method we again first retrieve the object to see if it exists:



    public DeleteCustomerResponse DeleteCustomer(DeleteCustomerRequest deleteCustomerRequest)
    {
    	try
    	{
    		Customer customer = _customerRepository.FindBy(deleteCustomerRequest.Id);
    		if (customer != null)
    		{
    			_customerRepository.Delete(customer);
    			_unitOfWork.Commit();
    			return new DeleteCustomerResponse();
    		}
    		else
    		{
    			return new DeleteCustomerResponse() { Exception = GetStandardCustomerNotFoundException() };
    		}
    	}
    	catch (Exception ex)
    	{
    		return new DeleteCustomerResponse() { Exception = ex };
    	}
    }


That should be it really, this is the implementation of the CustomerService class.

In the next post we'll start building the ultimate consumer of the application: the web layer which in this case will be a Web API web service. However, it could equally be a Console app, a WPF desktop app, a Silverlight app or an MVC web app, etc. It's up to you what type of interface you build upon the backend skeleton.

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/10/03/a-model-net-web-service-based-on-domain-driven-design-part-7-the-abstract-service/ "A model .NET web service based on Domain Driven Design Part 7: the abstract Service"
[2]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
