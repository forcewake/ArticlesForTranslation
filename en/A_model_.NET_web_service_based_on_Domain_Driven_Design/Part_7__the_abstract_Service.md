[Source](http://dotnetcodr.com/2013/10/03/a-model-net-web-service-based-on-domain-driven-design-part-7-the-abstract-service/ "Permalink to A model .NET web service based on Domain Driven Design Part 7: the abstract Service")

# A model .NET web service based on Domain Driven Design Part 7: the abstract Service

**Introduction**

Services can come in several different forms in an application. Services are not associated with things, but rather actions and activities. Services are the "doers", the manager classes and may span multiple domain objects. They define what they can DO for a client. They are unlikely to have attributes as Entities or Value objects do and will absolutely lack any form of identity. This post is the direct continuation of the [previous][1] post.

We can put services into three different groups:

_Infrastructure services_

These are all the cross-cutting services that you put into the Infrastructure layer: logging, emailing, caching etc. We've built a thin infrastructure layer in this skeleton.

_Domain services_

Normally logic pertaining to a single domain should be encapsulated within the body of that domain. If the domain object needs other domain objects to perform some logic then you can send in those as parameters. E.g. to determine whether a Person domain can loan a certain book you can have the following method in the Person class:



    private bool CanLoanBook(Book book)
    {
         if (!this.IsLateWithOtherBooks && !book.OnLoan) return true;
         return false;
    }


This is a very small and uncomplicated piece of code.

However, this is not always the case. It may be so that this logic becomes so involved that putting it in one domain object would make that domain bloated with logic that spans multiple other domains. Example: a money transfer may involve domain objects such as Bank, Account, Customer, Transfer and possibly more. The money may only be transferred if there's enough deposit on one Account, if the Customer has preferred status, if the Bank is not located in a money-laundering paradise etc. Where should this validation logic go? Probably the best solution is to build a specially designated IMoneyTransferService interface with a method called MakeTransfer(params).

Domain services help you avoid bloating the logic of one domain object. Just like other service types, they come in the form of interfaces and implementations of those interfaces. They will have operations that do not fit the domain object it is acting on. However, be careful not to transfer all business rules of the domain to a service. These services will be part of the domain layer.

_Application services_

Application services normally only carry out co-ordination tasks and are void of any extra logic themselves. They consult repositories and possibly other services to carry out tasks on behalf of the caller. The caller is typically some front end layer: an MVC controller in a web project, a Web API controller in a Web API project or a XAML view-model in an MVVM WPF project. Application services provide therefore the connecting tissue between the backend and the front end of the application.

It is this type of service that we'll implement in this post. Before we do that let's look at a couple of new terms.

**The Request-Response pattern**

The Request-Response pattern is a common pattern used within messaging in SOA – Service-Oriented Architecture. Consider the following interface of a service:



    IEnumerable<Product> GetProducts(string region);
    IEnumerable<Product> GetProducts(string region, string salesPoint);
    IEnumerable<Product> GetProducts(string region, string salesPoint, DateTime fromDate);
    IEnumerable<Product> GetProducts(string region, string salesPoint, DateTime fromDate, DateTime toDate);


The consumer can communicate with this interface in 4 different ways to retrieve a set of products. You can imagine that this list can grow on and on making the maintenance of the service difficult. It's easier to assign an object to hold all these criteria:



    public class GetProductsRequest
    {
        public string Region {get; set;}
        public string SalesPoint {get; set;}
        public DateTime FromDate {get; set;}
        public DateTime ToDate {get; set;}
    }


You can then simplify the service as follows:



    GetProductsResponse GetProducts(GetProductsRequest request);


…where GetProductsResponse may look as simple as the following:



    public class GetProductsResponse
    {
        public IEnumerable<Product> Products {get; set;}
    }


All Response objects can derive from a base response to show e.g. if the search operation was successful, if there are any specific error messages etc. The Amazon SDK for .NET uses this pattern heavily. You can check out a couple of examples [here][2] and [here][3].

I'll use this pattern in the service layer. You may think it is overkill in some cases but it's up to you how you take ideas from here and how you apply them.

**View models**

View models are objects that show the **view** representation of a domain. E.g. a domain may have a decimal price property with no specific formatting. You may want to show that price in your visitor's native format based on their culture settings. That will be the view representation of the price. Normally domain objects are not concerned with view details. That task is delegated to view models. Also, as we said before, domains should be void of technology-related things, such as MVC attributes. This is where a view model comes in handy as you are free to do to it what you want. Add attributes, format strings, add extra properties that the UI is expecting etc.

In an MVC UI layer it is quite customary to have view models injected into the views instead of the plain domains otherwise the Razor code may need to perform extra formatting which is not what it's supposed to do.

View models are therefore a technique to decouple the domain from its view-specific representation(s). A domain can have multiple different views depending on e.g. who is requesting the data: an admin, a simple user, Elvis Presley etc.

In this example app we'll employ a Customer view model in the service layer that will be served to the Web layer which we'll look at later. Note, that this is not a must in order to build a layered solution. Some readers may even find it excessive as it represents an extra round of conversion – I'm showing you how it can be done and I'll let you decide whether it's something for you or not.

Also note that the type of view-model I'm talking about now is not the same as view-models in the Model-View-ViewModel (MVVM) pattern used extensively in XAML-based technologies, such as Silverlight, WPF Windows 8+ store apps. In MVVM a view-model is a lot richer than the DTO variants I've described above. In MVVM the view-models can have logic, event handlers, property change notifications etc., which will not fit "our" view-models.

**Application service layer**

Open the project where we left off in the previous post and add a new C# class library called DDDSkeletonNET.Portal.ApplicationServices. Remove Class1.cs.

Add a folder called ViewModels and insert a class called CustomerViewModel in it:



    public class CustomerViewModel
    {
    	public string Name { get; set; }
    	public string AddressLine1 { get; set; }
    	public string AddressLine2 { get; set; }
    	public string City { get; set; }
    	public string PostalCode { get; set; }
    	public int Id { get; set; }
    }


Add another folder called ModelConversions with the following extension methods:



    public static class ConversionHelper
    {
    	public static CustomerViewModel ConvertToViewModel(this Customer customer)
    	{
    		return new CustomerViewModel()
    		{
    			AddressLine1 = customer.CustomerAddress.AddressLine1
    			,AddressLine2 = customer.CustomerAddress.AddressLine2
    			,City = customer.CustomerAddress.City
    			,Id = customer.Id
    			,Name = customer.Name
    			,PostalCode = customer.CustomerAddress.PostalCode
    		};
    	}

    	public static IEnumerable<CustomerViewModel> ConvertToViewModels(this IEnumerable<Customer> customers)
    	{
    		List<CustomerViewModel> customerViewModels = new List<CustomerViewModel>();
    		foreach (Customer customer in customers)
    		{
    			customerViewModels.Add(customer.ConvertToViewModel());
    		}
    		return customerViewModels;
    	}
    }


Make sure to set the namespace to DDDSkeletonNET.Portal so that the extension methods will be available without setting any extra using statements elsewhere in the project.

Add a new folder called Messaging and insert the following base classes for the service requests and responses:



    public abstract class ServiceRequestBase
    {
    }




    public abstract class ServiceResponseBase
    {
    	public ServiceResponseBase()
    	{
    		this.Exception = null;
    	}

    	public Exception Exception { get; set; }
    }


The base request is empty at this point but we could add any properties common to all requests later. We want to preserve any caught exception in the responses which is why we have the Exception property.

Many request types will involve an ID: search, update, delete. Let's create a base request class for those with integer IDs in the Messaging folder:



    public abstract class IntegerIdRequest : ServiceRequestBase
    {
    	private int _id;

    	public IntegerIdRequest(int id)
    	{
    		if (id < 1)
    		{
    			throw new ArgumentException("ID must be positive.");
    		}
    		_id = id;
    	}

    	public int Id { get { return _id; } }
    }


We'll expose the service calls in an interface. Add a new folder called Interfaces and insert the following ICustomerService interface:



    public interface ICustomerService
    {
    	GetCustomerResponse GetCustomer(GetCustomerRequest getCustomerRequest);
    	GetCustomersResponse GetAllCustomers();
    	InsertCustomerResponse InsertCustomer(InsertCustomerRequest insertCustomerRequest);
    	UpdateCustomerResponse UpdateCustomer(UpdateCustomerRequest updateCustomerRequest);
    	DeleteCustomerResponse DeleteCustomer(DeleteCustomerRequest deleteCustomerRequest);
    }


Insert a new folder called Customers within the Messaging folder. The request response objects look as follows:



    public class DeleteCustomerRequest : IntegerIdRequest
    {
    	public DeleteCustomerRequest(int customerId) : base(customerId)
    	{}
    }




    public class DeleteCustomerResponse : ServiceResponseBase
    {}




    public class GetCustomerRequest : IntegerIdRequest
    {
    	public GetCustomerRequest(int customerId) : base(customerId)
    	{}
    }




    public class GetCustomerResponse : ServiceResponseBase
    {
    	public CustomerViewModel Customer { get; set; }
    }




    public class GetCustomersResponse : ServiceResponseBase
    {
    	public IEnumerable<CustomerViewModel> Customers { get; set; }
    }




    public class InsertCustomerRequest : ServiceRequestBase
    {
    	public CustomerPropertiesViewModel CustomerProperties { get; set; }
    }


…where CustomerPropertiesViewModel looks as follows:



    public class CustomerPropertiesViewModel
    {
    	public string Name { get; set; }
    	public string AddressLine1 { get; set; }
    	public string AddressLine2 { get; set; }
    	public string City { get; set; }
    	public string PostalCode { get; set; }
    }




    public class InsertCustomerResponse : ServiceResponseBase
    {}




    public class UpdateCustomerRequest : IntegerIdRequest
    {
    	public UpdateCustomerRequest(int customerId) : base(customerId)
    	{}

    	public CustomerPropertiesViewModel CustomerProperties { get; set; }
    }




    public class UpdateCustomerResponse : ServiceResponseBase
    {}


You'll need to add references to the Infrastructure and Domain layers for this to compile.

We have now the basis for the first concrete service. The next post will show how that can be implemented.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/30/a-model-net-web-service-based-on-domain-driven-design-part-6-the-concrete-repository-continued/ "A model .NET web service based on Domain Driven Design Part 6: the concrete Repository continued"
[2]: http://dotnetcodr.com/2013/07/15/how-to-manage-amazon-machine-images-with-the-net-amazon-sdk-part-1-starting-an-image-instance/ "How to manage Amazon Machine Images with the .NET Amazon SDK Part 1: starting an image instance"
[3]: http://dotnetcodr.com/2013/07/18/how-to-manage-amazon-machine-images-with-the-net-amazon-sdk-part-2-monitoring-and-terminating-ami-instances-managing-security-groups/ "How to manage Amazon Machine Images with the .NET Amazon SDK Part 2: monitoring and terminating AMI instances, managing Security Groups"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
