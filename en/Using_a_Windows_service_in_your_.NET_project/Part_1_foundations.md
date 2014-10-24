[Source](http://dotnetcodr.com/2014/10/06/using-a-windows-service-in-your-net-project-part-1-foundations/ "Permalink to Using a Windows service in your .NET project part 1: foundations")

# Using a Windows service in your .NET project part 1: foundations

**Introduction**

As the title suggests we'll discuss Windows services in this new series and how they can be used in your .NET project. We'll take up the following topics:

* What is a Windows service?
* How can it be (un)installed using scripts?
* What's the use of a Windows Service?
* How can it be taken advantage of in a .NET project?
* Simplified layered architecture with OOP principles: we'll see a lot of abstractions and dependency injection

We'll also build a small demo application with the following components:

* A Windows service
* A Console-based consumer with a simplified layered architecture
* A MongoDb database

If you're not familiar with MongoDb, don't worry. You can read about it on this blog starting [here][1]. Just make sure you read at least the first 2 posts so that you know how to install MongoDb as a Windows service on your PC. Otherwise we won't see much MongoDb related code in the demo project.

A warning: it's not possible to create Windows services in the Express edition of Visual Studio. I'll be building the project in VS 2013 Professional.

**Windows services**

Windows services are executable assemblies that run in the background without the user's ability to interact with them directly. They are often used for long running or periodically recurring activities such as polling, monitoring, listening to network connections and the like. Windows services have no user interface and have their own user session. It's possible to start a Windows service without the user logging on to the computer.

An average Windows PC has a large number of pre-installed Windows services, like:

…and many more. You can view the services by opening the Services window:

![Windows services window][2]

We'll go through the different categories like "Name" and "Status" when we create the Windows service project.

There are some special considerations about Windows services:

* A Windows service project cannot be started directly in Visual Studio and step through its code with F11
* Instead the service must be installed and started in order to see it in action
* An installer must be attached to the Windows service so that it can registered with the Windows registry
* Windows services cannot communicate directly with the logged-on user. E.g. it cannot write to a Console that the user can read. Instead it must write messages to another source: a log file, the Windows event log, a web service, email etc.
* A Windows service runs under a user context different from that of the logged-on user. It's possible to give a different role to the Windows service than the role of the user. Normally you'll run the service with roles limited to the tasks of the service

**Demo project description**

In the demo project we'll simulate a service that analyses web sites for us: makes a HTTP call, takes note of the response time and also registers the contents of the web site. This operation could take a long time especially of the user is allowed to enter a range of URIs that should be carried out in succession. Think of a recorded session in [Selenium][3] where each step is carried out sequentially, like…

* Go to login page
* Log in
* Search for a product
* Reserve a product
* Pay
* Log out

It is not too wise to make these URL calls directly from a web page and present the results at the end. The HTTP communication may take far too much time to get the final status and the end user won't see any intermediate results, such as "initialising", "starting step 1″, "step 2 complete" etc. Websites are simply not designed to carry out long-running processes and wait for the response.

Instead it's better to just create a "job" in some data storage that a continuous process, like a Windows service, is monitoring. This service can then carry out the job and update the data storage with information. The web site receives a correlation ID for the job which it can use to query the data store periodically. The most basic solution for the web page is to have a timer which automatically refreshes the window and runs the code that retrieves the latest status of the job using the correlation ID. An alternative here is [web sockets][4].

You could even hide the data store and the Windows service setup behind a web service so that the web site only communicates with this web service using the correlation ID.

To keep this series short and concise I'll skip the web page. I'm not exactly a client-side wizard and it would only create unnecessary noise to build a web side with all the elements to enter the URLs and show the job updates. Instead, we'll replace it with a Console based consumer.

At the end we'll have a console app where you can enter a series of URLs which are then entered in the MongoDb data store as a job. The Console will receive a job correlation ID. The Windows Service will monitor this data store and act upon every new job. It will call the URLs in succession and take note of the response time and string content of each URL. It will also update the status message of the job. The console will in turn monitor the job status and print the messages so that the user can track the status. We'll also put in place a simple file-based logging system to see what the Windows service is actually doing.

**Start: infrastructure**

Open Visual Studio 2012/2013 Pro and create a blank solution called WindowsServiceDemo. Make sure to select .NET4.5 as the framework for all layers we create within this solution.

We'll start from the very end of the project and create an abstraction for the planned HTTP calls. We've discussed a lot on this blog why it's important to factor out dependencies such as file system operations, emailing, service calls etc. If you're not sure why this is a good idea then go through the series on SOLID starting [here][5], where especially [Dependency Injection][6] is most relevant.

We've also gone through a range of examples of cross cutting concerns in [this][7] series where we factored out the common concerns into an Infrastructure project. We'll do it here as well and describe the HTTP communication service in an interface first.

Insert a new C# class library called Demo.Infrastructure into the blank solution. Add a folder called HttpCommunication to it. Making a HTTP call can require a large amount of inputs: the URI, the HTTP verb, a HTTP body, the headers etc. Instead of creating a long range of overloaded methods we'll gather them in an object which will be used as the single input parameter. Let's keep this simple and insert the following interface into the folder:



    public interface IHttpService
    {
        Task<MakeHttpCallResponse> MakeHttpCallAsync(MakeHttpCallRequest request);
    }


The HTTP calls will be carried out asynchronously. If you're not familiar with asynchronous method calls in .NET4.5 then start [here][8]. MakeHttpCallResponse and MakeHttpCallRequest have the following form:



    public class MakeHttpCallResponse
    {
            public int HttpResponseCode { get; set; }
            public string HttpResponse { get; set; }
            public bool Success { get; set; }
            public string ExceptionMessage { get; set; }
    }


* HttpResponseCode: the HTTP response code from the server, such as 200, 302, 400 etc.
* HttpResponse: the string contents of the URL
* Success: whether the HTTP call was successful
* ExceptionMessage: any exception message in case Success was false



    public class MakeHttpCallRequest
    {
    	public Uri Uri { get; set; }
    	public HttpMethodType HttpMethod { get; set; }
    	public string PostPutPayload { get; set; }
    }


* Uri: the URI to be called
* HttpMethodType: the HTTP verb
* PostPutPayload: the string HTTP body for POST and PUT operations

HttpMethodType is an enumeration which you can also insert into HttpCommunication:



    public enum HttpMethodType
    {
    	Get
    	, Post
    	, Put
    	, Delete
    	, Head
    	, Options
    	, Trace
    }


That should be straightforward.

Both MakeHttpCallRequest and MakeHttpCallResponse are admittedly oversimplified in their current forms but they suffice for the demo.

For the implementation we'll use the [HttpClient][9] class in the System.Net library. Add the following .NET libraries to the Infrastructure layer:

* System.Net version 4.0.0.0
* System.Net.Http version 4.0.0.0

Insert a class called HttpClientService to the HttpCommunication folder, which will implement IHttpService as follows:



    public class HttpClientService : IHttpService
    {
    	public async Task<MakeHttpCallResponse> MakeHttpCallAsync(MakeHttpCallRequest request)
    	{
    		MakeHttpCallResponse response = new MakeHttpCallResponse();
    		using (HttpClient httpClient = new HttpClient())
    		{
    			httpClient.DefaultRequestHeaders.ExpectContinue = false;
    			HttpRequestMessage requestMessage = new HttpRequestMessage(Translate(request.HttpMethod), request.Uri);
    			if (!string.IsNullOrEmpty(request.PostPutPayload))
    			{
    				requestMessage.Content = new StringContent(request.PostPutPayload);
    			}
    			try
    			{
    				HttpResponseMessage responseMessage = await httpClient.SendAsync(requestMessage, HttpCompletionOption.ResponseContentRead, CancellationToken.None);
    				HttpStatusCode statusCode = responseMessage.StatusCode;
    				response.HttpResponseCode = (int)statusCode;
    				response.HttpResponse = await responseMessage.Content.ReadAsStringAsync();
    				response.Success = true;
    			}
    			catch (Exception ex)
    			{
    				Exception inner = ex.InnerException;
    				if (inner != null)
    				{
    					response.ExceptionMessage = inner.Message;
    				}
    				else
    				{
    					response.ExceptionMessage = ex.Message;
    				}
    			}
    		}
    		return response;
    	}

    	public HttpMethod Translate(HttpMethodType httpMethodType)
    	{
    		switch (httpMethodType)
    		{
    			case HttpMethodType.Delete:
    				return HttpMethod.Delete;
    			case HttpMethodType.Get:
    				return HttpMethod.Get;
    			case HttpMethodType.Head:
    				return HttpMethod.Head;
    			case HttpMethodType.Options:
    				return HttpMethod.Options;
    			case HttpMethodType.Post:
    				return HttpMethod.Post;
    			case HttpMethodType.Put:
    				return HttpMethod.Put;
    			case HttpMethodType.Trace:
    				return HttpMethod.Trace;
    		}
    		return HttpMethod.Get;
    	}
    }


So we simply call the URI using the HttpClient and HttpRequestMessage objects. We get the HttpResponseMessage by awaiting the asynchronous SendAsync method of HttpClient. We save the Http status code as an integer and the string content of the URI in the MakeHttpCallResponse object. We also store any exception message in this response object to indicate that the HTTP call failed.

In the [next part][10] of the series we'll build the domain and repository layer.

View the list of posts on Architecture and Patterns [here][11].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/07/28/mongodb-in-net-part-1-foundations/ "MongoDB in .NET part 1: foundations"
[2]: http://dotnetcodr.files.wordpress.com/2014/09/windows-services-window.png?w=300&h=147
[3]: http://www.seleniumhq.org/ "Selenium homepage"
[4]: http://dotnetcodr.com/2014/05/15/introduction-to-websockets-with-signalr-in-net-part-1-the-basics/ "Introduction to WebSockets with SignalR in .NET Part 1: the basics"
[5]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[6]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[7]: http://dotnetcodr.com/2014/09/01/externalising-dependencies-with-dependency-injection-in-net-part-1-setup/ "Externalising dependencies with Dependency Injection in .NET part 1: setup"
[8]: http://dotnetcodr.com/2012/12/31/await-and-async-in-net-4-5-with-c/ "Await and async in .NET 4.5 with C#"
[9]: http://msdn.microsoft.com/en-us/library/system.net.http.httpclient(v=vs.118).aspx "HttpClient on MSDN"
[10]: http://dotnetcodr.com/2014/10/09/using-a-windows-service-in-your-net-project-part-2-the-domain-and-repository/ "Using a Windows service in your .NET project part 2: the domain and repository"
[11]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
