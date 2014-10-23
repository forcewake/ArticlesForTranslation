[Source](http://dotnetcodr.com/2013/10/14/a-model-net-web-service-based-on-domain-driven-design-part-10-tests-and-conclusions/ "Permalink to A model .NET web service based on Domain Driven Design Part 10: tests and conclusions")

# A model .NET web service based on Domain Driven Design Part 10: tests and conclusions

**Introduction**

In the [previous][1] post we've come as far as implementing all Get, Post, Put and Delete methods. We also tested the Get methods as the results could be viewed in the browser alone. In order to test the Post, Put and Delete methods we'll need to do some more work.

**Demo**

There are various tools out there that can generate any type of HTTP calls for you where you can specify the JSON inputs, HTTP headers etc. However, we're programmers, right? We don't need no tools, we can write a simple one ourselves! Don't worry, I only mean a GUI-less throw-away application that consists of a few lines of code, not a complete Fiddler.

Fire up Visual Studio and create a new Console application. Add a reference to the System.Net.Http library. Also, add a NuGet package reference to Json.NET:

![Newtonsoft Json.NET in NuGet][2]

System.Net.Http includes all objects necessary for creating Http messages. Json.NET will help us package the input parameters in the message body.

_POST_

Let's start with testing the insertion method. Recall from the previous post that the Post method in CustomersController is expecting an InsertCustomerRequest object:



    public HttpResponseMessage Post(CustomerPropertiesViewModel insertCustomerViewModel)


We'll make this easy for us and create an identical CustomerPropertiesViewModel in the tester app so that the Json translation will be as easy as possible. So insert a class called CustomerPropertiesViewModel into the tester app with the following properties:



    public string Name { get; set; }
    public string AddressLine1 { get; set; }
    public string AddressLine2 { get; set; }
    public string City { get; set; }
    public string PostalCode { get; set; }


Insert the following method in order to test the addition of a new customer:



    private static void RunPostOperation()
    {
    	HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Post, _serviceUri);
    	requestMessage.Headers.ExpectContinue = false;
    	CustomerPropertiesViewModel newCustomer = new CustomerPropertiesViewModel()
    	{
    		AddressLine1 = "New address"
    		, AddressLine2 = string.Empty
    		, City = "Moscow"
    		, Name = "Awesome customer"
    		, PostalCode = "123987"
    	};
    	string jsonInputs = JsonConvert.SerializeObject(newCustomer);
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


…where _serviceUri is a private Uri:



    private static Uri _serviceUri = new Uri("http://localhost:9985/customers");


This is the URL that's displayed in the browser when you start the web API project. The port number may of course differ from yours, so adjust it accordingly.

This is of course not a very optimal method as it carries out a lot of things, but that's beside the point right now. We only a need a simple tester, that's all.

If you don't know the HttpClient and the related objects in this method, then don't worry, you've just learned something new. We set the web method to POST and assign the string contents to the JSONified version of the CustomerPropertiesViewModel object. We also set the request timeout to 10 minutes so that you don't get a timeout exception as you slowly jump through the code in the web service in a bit. The application type is set to JSON so that the web service understands which media type converter to take. We then send the request to the web service using the SendAsynch method and wait for a reply.

Make sure to insert a call to this method from within Main.

Open CustomersController in the DDD skeleton project and set a breakpoint within the Post method here:



    InsertCustomerResponse insertCustomerResponse = _customerService.InsertCustomer(new InsertCustomerRequest() { CustomerProperties = insertCustomerViewModel });


Start the skeleton project. Then run the tester. If everything went well then execution should stop at the breakpoint. Inspect the contents of the incoming insertCustomerViewModel parameter. You'll see that the parameter properties have been correctly assigned by the JSON formatter.

From this point on I encourage you to step through the web service call with F11. You'll see how the domain object is created and validated – including the Address property, how it is converted into the corresponding database type, how the IUnitOfWork implementation – InMemoryUnitOfWork – registers the insertion and how it is persisted. You'll also see that all abstract dependencies have been correctly instantiated by StructureMap, we haven't got any exceptions along the way which may point to some null dependency problem.

When the web service call returns the you should see that it returned an Exception == null property to the caller, i.e. the tester app. Now refresh the browser and you'll see the new customer just inserted into memory.

Let's try something else: assign an empty string to the Name property of the CustomerPropertiesViewModel object in the tester app. We know that a customer must have a name so we're expecting some trouble. Run the tester app and you should see an exception being thrown at this code line:



    throw new Exception(brokenRulesBuilder.ToString());


…within CustomerService.cs. This is because the BrokenRules list of the Customer domain contains one broken rule. Let the code execute and you'll see in the tester app that we received the correct exception message:

![Validation exception from web service][3]

Now assign an empty string to the City property as we know that an Address must have a city. You'll see a similar response:

![Address must have a city validation exception][4]

_PUT_

We'll follow the same analogy as above in order to update a customer. Insert the following object into the tester app:



    public class UpdateCustomerViewModel : CustomerPropertiesViewModel
    {
    	public int Id { get; set; }
    }


Insert the following method into Program.cs:



    private static void RunPutOperation()
    {
    	HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Put, _serviceUri);
    	requestMessage.Headers.ExpectContinue = false;
    	UpdateCustomerViewModel updatedCustomer = new UpdateCustomerViewModel()
    	{
    		Id = 2
    		, AddressLine1 = "Updated address line 1"
    		, AddressLine2 = string.Empty
    		, City = "Updated city"
    		, Name = "Updated customer name"
    		, PostalCode = "0988765"
    	};

    	string jsonInputs = JsonConvert.SerializeObject(updatedCustomer);
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


This is almost identical to the Post operation method except that we set the http verb to PUT and we send a JSONified UpdateCustomerViewModel object to the service. You'll notice that we want to update the customer with id = 2. Comment out the call to RunPostOperation() in Main and add a new call to RunPutOperation() instead. Set a breakpoint within the Put method in CustomersController of the skeleton web service. Jump through the code execution with F11 as before and follow along how the customer is found and updated. Refresh the browser to see the updated values of Customer with id = 2. Run the same test as above: set the customer name to an empty string and try to update the resource. The request should fail in the validation phase.

_DELETE_

The delete method is a lot simpler as we only need to send an id to the web Delete method of the web service:



    private static void RunDeleteOperation()
    {
    	HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Delete, string.Concat(_serviceUri, "/3"));
    	requestMessage.Headers.ExpectContinue = false;
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


You'll see that we set the http method to DELETE and extended to service Uri to show that we're talking about customer #3.

Set a breakpoint within the Delete method in CustomersController.cs and as before step through the code execution line by line to see how the deletion is registered and persisted by IUnitOfWork.

**Analysis and conclusions**

That actually completes the first version of the skeleton project. Let's see if we're doing any better than the tightly coupled solution of the first installment of this series.

_Dependency graph_

If you recall from the first part of this series then the greatest criticism against the technology-driven layered application was that all layers depended on the repository layer and that EF-related objects permeated the rest of the application.

We'll now look at an updated dependency graph. Before we do that there's a special group of dependency in the skeleton app that slightly distorts the picture: the web layer references all other layers for the sake of StructureMap. StructureMap needs some hints as to where to look for implementations so we had to set a library reference to all other layers. This is not part of any business logic so a true dependency graph should, I think, not consider this set of links. If you don't like this coupling then it's perfectly reasonable to create a separate very thin layer for StructureMap and let that layer reference infra, repo, domains and services.

If we start from the top then we see that the web layer talks to the service layer through the ICustomerService interface. The ICustomerService interface uses RequestResponse objects to communicate with the outside world. Any implementation of ICustomerService will communicate through those objects so their use within CustomersController is acceptable as well. The Service doesn't spit out Customer domain objects directly. Instead it wraps CustomerViewModels that are wrapped within the corresponding GetCustomersResponse object. Therefore I think we can conclude that the Web layer only depends on the service layer and that this coupling is fairly loose.

The application service layer has a reference to the Infrastructure layer – through the IUnitOfWork interface – and the Domain layer. In a full-blown project there will be more links to the Infrastructure layer – logging, caching, authentication etc. – but as longs as you hide those concerns behind abstractions you'll be fine. The Customer repository is only propagated in the form of an interface – ICustomerRepository. Otherwise the domain objects are allowed to bubble up to the Service layer as they are the central elements of the application.

The Domain layer has a dependency on the Infrastructure layer through abstractions such as EntityBase, IAggregateRoot and IRepository. That's all fine and good. The only coupling that's tighter than these is the BusinessRule object where we construct a new BusinessRule in the CustomerBusinessRule class. In retrospect it may have been a better idea to have an abstract BusinessRule class in the infrastructure layer with concrete implementations of it in the domain layer.

The repository layer has a reference to the Infrastructure layer – again through abstractions such as IAggregateRoot and IUnitOfWorkRepository – and the Domain layer. Notice that we changed the direction of the dependency compared to the how-not-to-do mini-project of the first post in this series: the domain layer does not depend on the repository but the repository depends on the domain layer.

Here's the updated dependency graph:

![DDD improved dependency graph][5]

I think the most significant change compared to where we started is that **no single layer is directly or indirectly dependent on the concrete repository layer**. You can test and unload the project from the solution – right-click, select Unload Project. There will be a broken reference in the Web layer that only exists for the sake of StructureMap, but otherwise the solution survives this "amputation". We have successfully hidden the most technology-driven layer behind an abstraction that can be replaced at ease. You need to implement the IUnitOfWork and IUnitOfWorkRepository interfaces and you should be fine. You can then instruct StructureMap to use the new implementation instead. You can even switch between two or more different implementations to test how the different technologies work before you go for a specific one in your project. Of course writing those implementations may not be a trivial task but switching from one technology to another will certainly be.

Another important change is that the domain layer is now truly the central one in the solution. The services and data access layers directly reference it. The UI layer depends on it indirectly through the service layer.

That's all folks. I hope you have learned new things and can use this skeleton solution in some way in your own project.

The project has been extended. Read the first extension [here][6].

View the list of posts on Architecture and Patterns [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/10/10/a-model-net-web-service-based-on-domain-driven-design-part-9-the-web-layer/ "A model .NET web service based on Domain Driven Design Part 9: the Web layer"
[2]: http://dotnetcodr.files.wordpress.com/2013/09/newtonsoftinnuget.png?w=630&h=123
[3]: http://dotnetcodr.files.wordpress.com/2013/09/acustomermusthaveaname.png?w=630&h=281
[4]: http://dotnetcodr.files.wordpress.com/2013/09/addressmusthaveacity.png?w=630&h=137
[5]: http://dotnetcodr.files.wordpress.com/2013/09/dddcorrectdependencygraph.png?w=630
[6]: http://dotnetcodr.com/2014/03/17/extension-to-the-ddd-skeleton-project-domain-specific-rules-part-1/ "Extension to the DDD skeleton project: domain specific rules Part 1"
[7]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
