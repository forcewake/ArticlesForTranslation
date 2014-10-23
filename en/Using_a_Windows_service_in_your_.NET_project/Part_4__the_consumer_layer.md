[Source](http://dotnetcodr.com/2014/10/16/using-a-windows-service-in-your-net-project-part-4-the-consumer-layer/ "Permalink to Using a Windows service in your .NET project part 4: the consumer layer")

# Using a Windows service in your .NET project part 4: the consumer layer

**Introduction**

In the [previous][1] post of this series we built the application service of the demo application. We're now ready to build the consumer layer.

For demo purposes we'll only build a Console-based consumer where the user can insert the URLs to run and monitor the progress. In reality the consumer layer would most likely be an MVC application which can then be refreshed periodically to get the latest status. Alternatively you can turn to [web sockets with SignalR][2] to push the updates to the screen. In our example, however, a console app will suffice to reach the goal of using a Windows service.

We'll build upon the demo app we've been using so far so have it ready in Visual Studio

**The consumer layer**

Add a new Console application to the demo solution called Demo.DemoConsumer. Add a reference to all other layers in the solution:

* Repository
* ApplicationService
* Domain
* Infrastructure

First we'll need a method that builds an IHttpJobService that the Console can work with:



    private static IHttpJobService BuildHttpJobService()
    {
    	IConfigurationRepository configurationRepository = new ConfigFileConfigurationRepository();
    	IDatabaseConnectionSettingsService dbConnectionSettingsService = new HttpJobDatabaseConnectionService(configurationRepository);
    	IJobRepository jobRepository = new JobRepository(dbConnectionSettingsService);
    	IHttpJobService httpJobService = new HttpJobService(jobRepository);
    	return httpJobService;
    }


There's nothing magic here I hope. We simply build an IHttpJobService from various other ingredients according to their dependencies.

Then we'll build a simple console UI to type in the URLs to run. Insert the following method to Program.cs where the user can enter URLs and break the loop with an empty string:



    private static List<Uri> EnterUrlJobs()
    {
    	HttpJob httpJob = new HttpJob();
    	List<Uri> uris = new List<Uri>();
    	Console.WriteLine("Enter a range of URLs. Leave empty and press ENTER when done.");
    	Console.Write("Url 1: ");
    	string url = Console.ReadLine();
    	uris.Add(new Uri(url));
    	int urlCounter = 2;
    	while (!string.IsNullOrEmpty(url))
    	{
    		Console.Write("Url {0}: ", urlCounter);
    		url = Console.ReadLine();
    		if (!string.IsNullOrEmpty(url))
    		{
    			uris.Add(new Uri(url));
    			urlCounter++;
    		}
    	}

    	return uris;
    }


The following method will insert a new HttpJob and return the insertion response:



    private static InsertHttpJobResponse InsertHttpJob(List<Uri> uris, IHttpJobService httpJobService)
    {
    	InsertHttpJobRequest insertRequest = new InsertHttpJobRequest(uris);
    	InsertHttpJobResponse insertResponse = httpJobService.InsertHttpJob(insertRequest);
    	return insertResponse;
    }


Finally we need a method to monitor the job and print the job status on the screen:



    private static void MonitorJob(Guid correlationId, IHttpJobService httpJobService)
    {
    	GetHttpJobRequest getJobRequest = new GetHttpJobRequest() { CorrelationId = correlationId };
    	GetHttpJobResponse getjobResponse = httpJobService.GetHttpJob(getJobRequest);
    	bool jobFinished = getjobResponse.Job.Finished;
    	while (!jobFinished)
    	{
    		Console.WriteLine(getjobResponse.Job.ToString());
    		getjobResponse = httpJobService.GetHttpJob(getJobRequest);
    		jobFinished = getjobResponse.Job.Finished;
    		if (!jobFinished)
    		{
    			Thread.Sleep(2000);
    			Console.WriteLine();
    		}
    	}
    	getjobResponse = httpJobService.GetHttpJob(getJobRequest);
    	Console.WriteLine(getjobResponse.Job.ToString());
    }


We keep extracting the updated HttpJob until it has reached the Finished status. We wait for 2 seconds between each iteration. After the loop we want to print the final job status.

The methods can be called as following from Main:



    static void Main(string[] args)
    {
    	List<Uri> uris = EnterUrlJobs();
    	IHttpJobService httpJobService = BuildHttpJobService();
    	InsertHttpJobResponse insertResponse = InsertHttpJob(uris, httpJobService);
    	MonitorJob(insertResponse.JobCorrelationId, httpJobService);

    	Console.WriteLine("Main finishing, press any key to exit...");
    	Console.ReadKey();
    }


Run this code to see it in action. You'll be able enter a number of URLs first. Note that there's no extra validation so be precise and enter the full address, like "[https://www.google.com&#8221][3];. When you're done just press enter without entering a URL. The monitoring phase will begin:

![Initial Program output with no runner yet][4]

There's of course nothing that carries out the job yet so the while loop never exits in MonitorJob.

We'll start building the Windows service in the [next post][5].

View the list of posts on Architecture and Patterns [here][6].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/10/13/using-a-windows-service-in-your-net-project-part-3-the-application-service-layer/ "Using a Windows service in your .NET project part 3: the application service layer"
[2]: http://dotnetcodr.com/2014/05/15/introduction-to-websockets-with-signalr-in-net-part-1-the-basics/ "Introduction to WebSockets with SignalR in .NET Part 1: the basics"
[3]: https://www.google.com&#8221
[4]: http://dotnetcodr.files.wordpress.com/2014/09/initial-program-output-with-no-runner-yet.png?w=300&h=260
[5]: http://dotnetcodr.com/2014/10/20/using-a-windows-service-in-your-net-project-part-5-the-windows-service-project-basics/ "Using a Windows service in your .NET project part 5: the Windows Service project basics"
[6]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
