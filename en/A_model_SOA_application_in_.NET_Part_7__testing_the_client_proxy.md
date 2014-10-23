[Source](http://dotnetcodr.com/2014/01/06/a-model-soa-application-in-net-part-6-testing-the-client-proxy/ "Permalink to A model SOA application in .NET Part 7: testing the client proxy")

# A model SOA application in .NET Part 7: testing the client proxy

**Introduction**

In the previous post we finished building a thin but functional SOA web service. Now it's time to test it. We could build just any type of consumer that's capable of issuing HTTP requests to a service but we'll stick to a simple Console app. The goal is to test the SOA service and not to win the next CSS contest.

**The tester**

Open the SOA application we've been working on. Set the WebProxy layer as the start up project and then press F5. Without setting the start up URL in Properties/Web you'll most likely get a funny looking error page from IIS that says you are not authorised to view the contents of the directory. That's fine. Extend the URI in the browser as follows:

<http://localhost:xxxx/reservation>

Refresh the browser and you should get a JSON message saying that there is no GET method for that resource. That's also fine as we only implemented a POST method. The same is true for the purchase controller:

<http://localhost:xxxx/purchase>

Open another Visual Studio instance and create a new Console application called SoaTester. Import the Json.NET package through NuGet. Add assembly references to System.Net and System.Net.Http. Add the following private variables to Program.cs:



    private static Uri _productReservationServiceUri = new Uri("http://localhost:49679/reservation");
    private static Uri _productPurchaseServiceUri = new Uri("http://localhost:49679/purchase");


Recall that the Post methods require Request objects to function correctly. The easiest way to ensure that we don't need to deal with JSON formatting and serialisation issues we can just copy the Request objects we already have the SoaIntroNet.Service layer. Insert the following two objects into the console app:



    public class ReserveProductRequest
    {
    	public string ProductId { get; set; }
    	public int ProductQuantity { get; set; }
    }




    public class PurchaseProductRequest
    {
    	public string ReservationId { get; set; }
    	public string ProductId { get; set; }
    }


Note that we dropped the correlation ID property from the purchase request. It's irrelevant for the actual user and is set within the Purchase.Post() action anyway before the purchase request is passed to the Service layer.

We'll need to read the JSON responses as well so insert the following three objects in the console app. They will all look familiar:



    public abstract class ServiceResponseBase
    {
    	public ServiceResponseBase()
    	{
    		this.Exception = null;
    	}

    	public Exception Exception { get; set; }
    }




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


We'll first call the Reservation operation. The below method calls the product reservation URI and returns a product reservation response:



    private static ProductReservationResponse ReserveProduct(ReserveProductRequest request)
    {
    	ProductReservationResponse response = new ProductReservationResponse();
    	try
    	{
    		HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Post, _productReservationServiceUri);
    		requestMessage.Headers.ExpectContinue = false;
    		String jsonArguments = JsonConvert.SerializeObject(request);
    		requestMessage.Content = new StringContent(jsonArguments, Encoding.UTF8, "application/json");
    		HttpClient httpClient = new HttpClient();
    		httpClient.Timeout = new TimeSpan(0, 10, 0);
    		Task<HttpResponseMessage> httpRequest = httpClient.SendAsync(requestMessage,
    			HttpCompletionOption.ResponseContentRead, CancellationToken.None);
    		HttpResponseMessage httpResponse = httpRequest.Result;
    		HttpStatusCode statusCode = httpResponse.StatusCode;
            	HttpContent responseContent = httpResponse.Content;
    		Task<String> stringContentsTask = responseContent.ReadAsStringAsync();
    		String stringContents = stringContentsTask.Result;
    		if (statusCode == HttpStatusCode.OK && responseContent != null)
    		{
    			response = JsonConvert.DeserializeObject<ProductReservationResponse>(stringContents);
    		}
    		else
    		{
    			response.Exception = new Exception(stringContents);
    		}
    	}
    	catch (Exception ex)
    	{
    		response.Exception = ex;
    	}
    	return response;
    }


We send a HTTP request to the service and read off the response. We use Json.NET to serialise and deserialise the objects. We set the HttpClient timeout to 10 minutes to make sure we don't get any timeout exceptions as we are testing the SOA application.

The below bit of code calls this method from Main:



    ReserveProductRequest reservationRequest = new ReserveProductRequest();
    reservationRequest.ProductId = "13a35876-ccf1-468a-88b1-0acc04422243";
    reservationRequest.ProductQuantity = 10;
    ProductReservationResponse reservationResponse = ReserveProduct(reservationRequest);

    Console.WriteLine("Reservation response received.");
    Console.WriteLine(string.Concat("Reservation success: ", (reservationResponse.Exception == null)));
    if (reservationResponse.Exception == null)
    {
    	Console.WriteLine("Reservation id: " + reservationResponse.ReservationId);
    }
    else
    {
    	Console.WriteLine(reservationResponse.Exception.Message);
    }

    Console.ReadKey();


Recall that we inserted a product with that ID in the in-memory database in the post discussing the repository layer.

Set two breakpoints in the SOA app: one within the ReservationController constructor and another within the ReservationController.Post method. Start the console application. You should see that the code execution stops within the controller constructor. Check the status of the incoming productService parameter. It is not null so StructureMap correctly resolved this dependency. The next stop is at the second breakpoint. Check the status of the reserveProductRequest parameter. It is not null and has been correctly populated by Web API and the Json serialiser. From this point on I encourage you to step through the execution by pressing F11 to see how each method is called. At the end the product should be reserved and a product reservation message will be sent back to the client. Back in the client the ProductReservationResponse object is populated by Json.NET based on the string contents from the web service. The results are then printed on the console window.

You can test the service response in the following way:

* Send a non-existing product id
* Send a malformatted GUID as the product id
* Send a quantity higher than the original allocation, e.g. 10000

You'll get the correct error messages in all three cases.

It's now time to purchase the product. The below method will send the purchase request to the web service:



    private static PurchaseProductResponse PurchaseProduct(PurchaseProductRequest request)
    {
    	PurchaseProductResponse response = new PurchaseProductResponse();
    	try
    	{
    		HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Post, _productPurchaseServiceUri);
    		requestMessage.Headers.ExpectContinue = false;
    		String jsonArguments = JsonConvert.SerializeObject(request);
    		requestMessage.Content = new StringContent(jsonArguments, Encoding.UTF8, "application/json");
    		HttpClient httpClient = new HttpClient();
    		httpClient.Timeout = new TimeSpan(0, 10, 0);
    		Task<HttpResponseMessage> httpRequest = httpClient.SendAsync(requestMessage,
    			HttpCompletionOption.ResponseContentRead, CancellationToken.None);
    		HttpResponseMessage httpResponse = httpRequest.Result;
    		HttpStatusCode statusCode = httpResponse.StatusCode;
    		HttpContent responseContent = httpResponse.Content;
    		Task<String> stringContentsTask = responseContent.ReadAsStringAsync();
    		String stringContents = stringContentsTask.Result;
    		if (statusCode == HttpStatusCode.OK && responseContent != null)
    		{
    			response = JsonConvert.DeserializeObject<PurchaseProductResponse>(stringContents);
    		}
    		else
    		{
    			response.Exception = new Exception(stringContents);
    		}
    	}
    	catch (Exception ex)
    	{
    		response.Exception = ex;
    	}
    	return response;
    }


I realise that this is almost identical to PurchaseProduct and a single method probably would have sufficed – you can do this improvement as homework.

Below the code line…



    Console.WriteLine("Reservation id: " + reservationResponse.ReservationId);


…add the following bit to complete the loop:



    PurchaseProductRequest purchaseRequest = new PurchaseProductRequest();
    purchaseRequest.ProductId = reservationResponse.ProductId;
    purchaseRequest.ReservationId = reservationResponse.ReservationId;
    PurchaseProductResponse purchaseResponse = PurchaseProduct(purchaseRequest);
    if (purchaseResponse.Exception == null)
    {
    	Console.WriteLine("Purchase confirmation id: " + purchaseResponse.PurchaseId);
    }
    else
    {
    	Console.WriteLine(purchaseResponse.Exception.Message);
    }


So we purchase the reserved product immediately after receiving the reservation response. Run the tester object without any break points. You should see the purchase ID in the console window.

You can test the SOA app in the following way:

* Set a breakpoint in the Purchase controller and step through the code slowly with F11 – make sure that the purchase expiry time of 1 minute is reached
* Alternatively use Thread.Sleep() in the tester app before the call to the Purchase controller
* Modify the reservation id to some non-existent GUID
* Set the reservation id to a malformed GUID

You will get the correct exception messages in all cases from the SOA app.

That's all folks about SOA basics in this series. I hope you have learned a lot of new concepts.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
