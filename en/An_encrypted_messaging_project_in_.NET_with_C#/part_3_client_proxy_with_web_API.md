[Source](http://dotnetcodr.com/2013/12/02/an-encrypted-messaging-project-in-net-with-c-part-3-client-proxy-with-web-api/ "Permalink to An encrypted messaging project in .NET with C# part 3: client proxy with web API")

# An encrypted messaging project in .NET with C# part 3: client proxy with web API

**Introduction**

In the [previous][1] post we finished building the backend part for the first step in the conversation between the sender and the receiver. The sender needs to get a valid one-time public key and a message ID from the receiver. The receiver will communicate with the receiver through a [web API][2] based web service.

**The Web layer**

In this section I'll refer a lot to [this][3] post in the series on the DDD skeleton project. Make sure to get familiar with it as it contains a lot of information that's relevant to this post. In that post I show you how to add a web API project and transform it so that it only contains the web service relevant parts. We don't want to deal with views, JS files, images etc. in a web service project. I also go through the basics of how to install and use the IoC called StructureMap in an MVC-based project. It will be responsible for resolving dependencies such as the ones in this constructor:



    public AsymmetricCryptographyApplicationService(IAsymmetricCryptographyService cryptographyInfrastructureService
    			, IAsymmetricKeyRepositoryFactory asymmetricKeyRepositoryFactory)


Add a new MVC4 web application called Receiver.Web to the solution. Make sure to select the Web API template in the MVC4 Project Template window. Add a reference to all other layers – this is necessary for StructureMap as we'll see in a bit.

Next get rid of the standard MVC4 web application components. Again, consult the above link to see how it can be done. Install StructureMap from NuGet. Locate the generated IoC.cs file in the DependencyResolution folder. Make sure it looks as follows:



    public static IContainer Initialize()
    {
    	ObjectFactory.Initialize(x =>
    		{
    			x.Scan(scan =>
    			{
    				scan.TheCallingAssembly();
    				scan.AssemblyContainingType<IAsymmetricCryptographyApplicationService>();
    				scan.AssemblyContainingType<IAsymmetricCryptographyService>();
    				scan.AssemblyContainingType<IAsymmetricKeyRepository>();
    				scan.WithDefaultConventions();
    			});
    			x.For<IAsymmetricCryptographyService>().Use<RsaCryptographyService>();
    			x.For<IAsymmetricKeyRepositoryFactory>().Use<LazySingletonAsymmetricKeyRepositoryFactory>();
                            x.For<ISymmetricEncryptionService>().Use<RijndaelSymmetricCryptographyService>();
    		});
    	ObjectFactory.AssertConfigurationIsValid();
            return ObjectFactory.Container;
    }


Again, consult the above link if you don't know what this code means. We tell StructureMap to look for concrete implementations of abstractions in the other layers of the application. We also tell it to pick the RSA implementation of the IAsymmetricCryptographyService interface and the InMemoryAsymmetricKeyRepositoryFactory implementation of the IAsymmetricKeyRepositoryFactory interface.

Add the following section to the Register method of WebApiConfig.cs in the App_Start folder to make sure that we respond with JSON. Web API seems to have an issue with serialising XML:



    var json = config.Formatters.JsonFormatter;
    json.SerializerSettings.PreserveReferencesHandling = Newtonsoft.Json.PreserveReferencesHandling.Objects;
    config.Formatters.Remove(config.Formatters.XmlFormatter);


In the same method you'll see that the API calls are routed to api/controller/id by default. Remove the 'api' bit as follows:



    config.Routes.MapHttpRoute(
    	name: "DefaultApi",
    	routeTemplate: "{controller}/{id}",
    	defaults: new { id = RouteParameter.Optional }
    	);


Add a folder called Helpers to the web layer. Add the following two classes with extension methods:



    public static class ExceptionDictionary
    {
    	public static HttpStatusCode ConvertToHttpStatusCode(this Exception exception)
    	{
    		Dictionary<Type, HttpStatusCode> dict = GetExceptionDictionary();
    		if (dict.ContainsKey(exception.GetType()))
    		{
    			return dict[exception.GetType()];
    		}
    		return dict[typeof(Exception)];
    	}

    	private static Dictionary<Type, HttpStatusCode> GetExceptionDictionary()
    	{
    		Dictionary<Type, HttpStatusCode> dict = new Dictionary<Type, HttpStatusCode>();
    		dict[typeof(ResourceNotFoundException)] = HttpStatusCode.NotFound;
    		dict[typeof(Exception)] = HttpStatusCode.InternalServerError;
    		return dict;
    	}
    }




    public static class HttpResponseBuilder
    {
    	public static HttpResponseMessage BuildResponse(this HttpRequestMessage requestMessage, ServiceResponseBase baseResponse)
    	{
    		HttpStatusCode statusCode = HttpStatusCode.OK;
    		if (baseResponse.Exception != null)
    		{
    			statusCode = baseResponse.Exception.ConvertToHttpStatusCode();
    			HttpResponseMessage message = new HttpResponseMessage(statusCode);
    			message.Content = new StringContent(baseResponse.Exception.Message);
    			throw new HttpResponseException(message);
    		}
    		return requestMessage.CreateResponse<ServiceResponseBase>(statusCode, baseResponse);
    	}
    }


Set the namespace of both classes to Receiver.Web so that they are available from everywhere in the Web project without having to set using statements.

Add a new Web API controller in the Controllers folder. Name it PublicKeyController and make sure to select the Empty Api controller template in the Add Controller window. You'll see that the controller derives from ApiController. The public key controller will depend on the IAsymmetricCryptographyApplicationService interface, so insert the following private field and constructor:



    private readonly IAsymmetricCryptographyApplicationService _asymmetricCryptoApplicationService;

    public PublicKeyController(IAsymmetricCryptographyApplicationService asymmetricCryptoApplicationService)
    {
    	if (asymmetricCryptoApplicationService == null) throw new ArgumentNullException("IAsymmetricCryptographyApplicationService");
    	_asymmetricCryptoApplicationService = asymmetricCryptoApplicationService;
    }


The client will call this controller to get a new public key and a message ID. Therefore we need a GET method that returns just these elements. The implementation is very simple:



    public HttpResponseMessage Get()
    {
    	ServiceResponseBase response = _asymmetricCryptoApplicationService.GetAsymmetricPublicKey();
    	return Request.BuildResponse(response);
    }


We ask the application service to create the necessary elements and build a JSON response out of them. Set a breakpoint within AsymmetricCryptographyApplicationService.GetAsymmetricPublicKey(). Set the start page to PublicKey in properties of the web layer:

![Set PublicKey as start page in web properties][4]

Press F5 to start the application. Execution should stop at the breakpoint you set previously. Follow the code execution with F11 and see how the parts are connected. The new asymmetric key pair is generated and saved in the dictionary of the repository. At the end of the process the message ID and the public key should appear in the web browser:

![RSA key printed in web browser][5]

Take note of the URL, it will be the web service URL the sender will communicate with in the next step.

Now let's see how this value can be read by a sender.

**The sender**

The sender will be a Console application that sends and receives HTTP messages to and from the web service. I won't care too much about software practices here in order to save time.

Create a new Console application in Visual Studio and call it Sender. Add the following library references:

* System.Net
* System.Net.Http

Add a reference to Json.NET – previously known as NewtonSoft – through NuGet. We have contacted the Receiver before and we know that they now employ a standard RSA asymmetric algorithm for their encryption purposes. Synchronisation is vital in encryption scenarios otherwise the two parties will not be able to read each other's messages.

Add the following service URI to Program.cs:



    private static Uri _publicKeyServiceUri = new Uri("http://localhost:7695/PublicKey");


The port will almost certainly differ in your case so please modify it.

We'll process the Json coming from the Receiver in the following object:



    public class PublicKeyResponse
    {
    	public Guid MessageId { get; set; }
    	public RSACryptoServiceProvider RsaPublicKey { get; set; }
    }


From [this][6] post you'll know that RSACryptoServiceProvider is the .NET implementation of the RSA asymmetric encryption.

Enter the following method to Program.cs:



    private static PublicKeyResponse GetPublicKeyMessageId()
    {
    	PublicKeyResponse resp = new PublicKeyResponse();
    	try
    	{
    		HttpRequestMessage requestMessage = new HttpRequestMessage(HttpMethod.Get, _publicKeyServiceUri);
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
    			String stringContents = stringContentsTask.Result.Replace("$", string.Empty);
    			XmlDocument doc = (XmlDocument)JsonConvert.DeserializeXmlNode(stringContents, "root");
    			XmlElement root = doc.DocumentElement;
    			XmlNode messageIdNode = root.SelectSingleNode("MessageId");
    			resp.MessageId = Guid.Parse(doc.DocumentElement.SelectSingleNode("MessageId").InnerText);
    					RSACryptoServiceProvider rsaProvider = new RSACryptoServiceProvider();
    			rsaProvider.FromXmlString(doc.DocumentElement.SelectSingleNode("PublicKeyXml").InnerXml);
    			resp.RsaPublicKey = rsaProvider;
    		}
    	}
    	catch (Exception ex)
    	{
    		Console.WriteLine(ex.Message);
    	}
    	return resp;
    }


Let's see what's going on here. We want to return a PublicKeyResponse object. We initiate a Http GET request using the HttpClient and HttpRequestMessage objects which are built-in components of the System.Net library. We send the request by calling the SendAsync method. We read the Http status code and the response content. We then extract the string message coming from the service, which is the JSON you saw in the web browser.

We want to transform the JSON into XML to take advantage of the FromXmlString method of the RSACryptoServiceProvider object. However, the JSON response includes an element '$id' which invalidates the XML, the '$' character is illegal. Hence we need to get rid of it. The JSON now looks as follows:

"id": "1",
"MessageId": "157a0890-be7d-4a25-bcf8-78829c99f13a",
"PublicKeyXml": {
"RSAKeyValue": {
"Modulus": "uB6pzihs7A70Su2J33SegPMkMXdWayRoqE57X94eyOPAeiPW+lMhy+pFxHNZ6+1+l35vRzcGZ8xL6AFKFiiriq1Dgu5WSqluBylkkWGTZfA+k7cpAh4CEEy0jg4BvPlxg/f7ufzLhy9iAGKARISKLBbCLIEeBpjqgyXEcW0tzrk=",
"Exponent": "AQAB"
}
},
"Exception": null

The exact values will certainly differ from what you see but the format should be the same.

There's a very handy method in the Json.NET library which allows to transform a JSON string into an XmlDocument in the System.Xml namespace. This is done by the DeserializeXmlNode method. You'll notice the "root" parameter. The things is that the current JSON format makes it difficult for the JSON converter to know what the root node is. As you know there's no valid XML without a root node. Without this parameter you'll get an exception:

JSON root object has multiple properties. The root object must have a single property in order to create a valid XML document. Consider specifying a DeserializeRootElementName.

After the conversion our XML should look like this:



    <root>
    	<id>1</id>
    	<MessageId>4d38ee45-8c8a-4531-9513-765c5569c942</MessageId>
    	<PublicKeyXml>
    		<RSAKeyValue>
    			<Modulus>kdl1gUhLj6RNk6Ppm5LC6mol1CcIXKft9O34naJLbAOy2720a1HshFfhjkvW4A6ibqhLJCo2N2IGfIBGNDaOeQfyzxvltC6Mvw3Zykxe4Jkmbv7cbzxpnZ20GMW1wCzmfGoqB36movrDqAn1c2ftiznhu0IbDY8GJf3bkB0SwFc=</Modulus>
    			<Exponent>AQAB</Exponent>
    		</RSAKeyValue>
    	</PublicKeyXml>
    	<Exception />
    </root>


Then it's only a matter of reading out the relevant nodes from the XML and populate the PublicKeyResponse object.

You can call this method from Main as follows:



    PublicKeyResponse publicKeyResponse = GetPublicKeyMessageId();
    if (publicKeyResponse.MessageId != null &amp;&amp; publicKeyResponse.RsaPublicKey != null)
    {
    	Console.Write("Public key request successful. Enter your message: ");
    	string message = Console.ReadLine();
    }
    else
    {
    	Console.WriteLine("Public key request failed.");
    }
    Console.ReadKey();


We check if the public key request was successful and then ask the user to enter a message.

Make sure that the web API project is running and then start the Sender app. You can set a breakpoint within the Get method of the PublicKeyController of the Receiver to see the request come in. The Receiver will respond and the Sender will process the Json message. If everything goes well then you should have a populated PublicKeyResponse with a message id and an RSA public key.

So what happens with the message entered by the user? We'll find out in the next post.

You can view the list of posts on Security and Cryptography [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/11/28/an-encrypted-messaging-project-in-net-with-c-part-2-service-and-repository/ "An encrypted messaging project in .NET with C# part 2: service and repository"
[2]: http://www.asp.net/web-api "Web API official homepage"
[3]: http://dotnetcodr.com/2013/10/10/a-model-net-web-service-based-on-domain-driven-design-part-9-the-web-layer/ "A model .NET web service based on Domain Driven Design Part 9: the Web layer"
[4]: http://dotnetcodr.files.wordpress.com/2013/10/setpublickeyasstartpage.png?w=630
[5]: http://dotnetcodr.files.wordpress.com/2013/10/rsakeyinwebbrowser.png?w=630&h=103
[6]: http://dotnetcodr.com/2013/11/11/introduction-to-asymmetric-encryption-in-net-cryptography/ "Introduction to asymmetric encryption in .NET cryptography"
[7]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
