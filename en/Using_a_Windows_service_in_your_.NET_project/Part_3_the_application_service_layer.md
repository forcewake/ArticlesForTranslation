[Source](http://dotnetcodr.com/2014/10/13/using-a-windows-service-in-your-net-project-part-3-the-application-service-layer/ "Permalink to Using a Windows service in your .NET project part 3: the application service layer")

# Using a Windows service in your .NET project part 3: the application service layer

**Introduction**

In the [previous post][1] we built the Domain and Repository layers of the demo app. We'll now build the application service layer that connects the two with the consumer layer which we'll build in the next post.

Normally the consumer layer, such as a Web UI should not directly contact the repository for its data. The application service is responsible for joining all necessary repositories and other services to provide the data that the consumer layer needs to show.

We'll build upon the demo app we've been working on so far in this series so have it ready in Visual Studio.

**Short return to Infrastructure**

Before we begin we'll need two new elements in the Infrastructure layer. You'll recall from the previous post that the abstract MongoDbRepository class will need an IDatabaseConnectionSettingsService interface to read the connection string and database name. We'll store those in the app.config file which we'll create in the next post. However, these settings could potentially be stored in various different places: a database, a web service, etc.

We saw in [this][2] post how reading settings can be hidden behind an interface. We'll take a similar approach here.

Add a new folder called ConfigurationService to the Infrastructure layer. Insert the following interface into it:



    public interface IConfigurationRepository
    {
    	T GetConfigurationValue<T>(string key);
    	T GetConfigurationValue<T>(string key, T defaultValue);
    	string GetConnectionString(string connectionStringName);
    }


Normally settings come in name-value pairs so we pass some key value and expect the corresponding setting value back.

For the implementation we'll take the ConfigurationManager class located in the System.Configuration namespace. You'll need to add this dll to the references list of the Infrastructure layer. Here's the implementing class:



    public class ConfigFileConfigurationRepository : IConfigurationRepository
    {
    	public T GetConfigurationValue<T>(string key)
    	{
    		string value = ConfigurationManager.AppSettings[key];
    		if (value == null)
    		{
    			throw new KeyNotFoundException("Key " + key + " not found.");
    		}
    		try
    		{
    			if (typeof(Enum).IsAssignableFrom(typeof(T)))
    				return (T)Enum.Parse(typeof(T), value);
    			return (T)Convert.ChangeType(value, typeof(T));
    		}
    		catch (Exception ex)
    		{
    			throw ex;
    		}
    	}

    	public T GetConfigurationValue<T>(string key, T defaultValue)
    	{
    		string value = ConfigurationManager.AppSettings[key];
    		if (value == null)
    		{
    			return defaultValue;
    		}
    		try
    		{
    			if (typeof(Enum).IsAssignableFrom(typeof(T)))
    				return (T)Enum.Parse(typeof(T), value);
    			return (T)Convert.ChangeType(value, typeof(T));
    		}
    		catch (Exception ex)
    		{
    			return defaultValue;
    		}
    	}

    	public string GetConnectionString(string connectionStringName)
    	{
    		return ConfigurationManager.ConnectionStrings[connectionStringName].ConnectionString;
    	}
    }


We'll see shortly how this abstraction is used.

**The service layer**

Add a new C# library called Demo.ApplicationService to the solution. Add a project reference to the Domain and Infrastructure layers. Insert 3 folders:

* Messaging
* JobServices
* DatabaseConnectionService

Let's get the database connection service out of the way first. The implementation will need a dependency of type IConfigurationRepository to read the connection string and database name values. Insert the following class into the DatabaseConnectionService folder:



    public class HttpJobDatabaseConnectionService : IDatabaseConnectionSettingsService
    {
    	private readonly IConfigurationRepository _configurationRepository;

    	public HttpJobDatabaseConnectionService(IConfigurationRepository configurationRepository)
    	{
    		if (configurationRepository == null) throw new ArgumentNullException("IConfigurationRepository");
    		_configurationRepository = configurationRepository;
    	}

    	public string GetDatabaseConnectionString()
    	{
    		return _configurationRepository.GetConnectionString(Constants.HttpJobsConnectionStringKeyName);
    	}

    	public string GetDatabaseName()
    	{
    		return _configurationRepository.GetConfigurationValue<string>(Constants.HttpJobsDatabaseNameSettingKeyName);
    	}
    }


This class reads the setting key names from another class called Constants:



    public static class Constants
    {
    	public static string HttpJobsConnectionStringKeyName = "HttpJobsConnectionString";
    	public static string HttpJobsDatabaseNameSettingKeyName = "HttpJobsDatabaseName";
    }


So we'll store the connection string with the key "HttpJobsConnectionString" and the database name with the key "HttpJobsDatabaseName". Remember, these settings could be stored anywhere hence the need to inject the service which actually reads the settings. We'll fill in the app settings file shortly.

**Service abstractions**

OK, let's create the abstractions for the Job related services. An IHttpJobService should be able to retrieve a specific job, retrieve all new jobs, insert a new job and update an existing job. Add the following interface to the JobServices folder:



    public interface IHttpJobService
    {
    	InsertHttpJobResponse InsertHttpJob(InsertHttpJobRequest request);
    	GetHttpJobResponse GetHttpJob(GetHttpJobRequest request);
    	UpdateHttpJobResponse UpdateHttpJob(UpdateHttpJobRequest request);
    	GetHttpJobsResponse GetNewHttpJobs();
    }


We have several new but simple RequestResponse objects here. They are used to relay all necessary inputs to the Service instead of creating many overloaded variants of the same methods. Insert the following classes to the Messaging folder:



    public abstract class BaseResponse
    {
    	public Exception OperationException { get; set; }
    }




    public class InsertHttpJobResponse : BaseResponse
    {
    	public Guid JobCorrelationId { get; set; }
    }




    public class InsertHttpJobRequest
    {
    	private List<Uri> _urisToRun;

    	public InsertHttpJobRequest(List<Uri> urisToRun)
    	{
    		if (urisToRun == null || !urisToRun.Any())
    		{
    			throw new ArgumentException("The URI list cannot be empty.");
    		}
    		_urisToRun = urisToRun;
    	}

    	public List<Uri> UrisToRun
    	{
    		get
    		{
    			return _urisToRun;
    		}
    	}
    }




    public class GetHttpJobResponse : BaseResponse
    {
    	public HttpJob Job { get; set; }
    }




    public class GetHttpJobRequest
    {
    	public Guid CorrelationId { get; set; }
    }




    public class UpdateHttpJobResponse : BaseResponse
    {}




    public class UpdateHttpJobRequest
    {
    	private HttpJob _updatedJob;
    	public UpdateHttpJobRequest(HttpJob updatedJob)
    	{
    		_updatedJob = updatedJob;
    	}

    	public HttpJob UpdatedHttpJob
    	{
    		get
    		{
    			return _updatedJob;
    		}
    	}
    }




    public class GetHttpJobsResponse : BaseResponse
    {
    	public IEnumerable<HttpJob> HttpJobs { get; set; }
    }


Those were the elements needed for the HttpJob service. We'll need a separate service to carry out a single JobUrl within a HttpJob object. Add the following interface to the JobServices folder:



    public interface IHttpJobUrlService
    {
    	Task<JobUrlProcessResponse> CarryOutSingleJobUrl(Uri uri);
    }


Carrying out a HTTP call can take some time so we'll go for an asynchronous model using the [Task library][3]. JobUrlProcessResponse is similar to the above Response objects. Insert the following class to the Messaging folder:



    public class JobUrlProcessResponse : BaseResponse
    {
    	public int HttpResponseCode { get; set; }
    	public string HttpContent { get; set; }
    	public TimeSpan TotalResponseTime { get; set; }
    }


You'll recognise the properties from the JobUrl domain object: we'll need to read the HTTP response code, the HTML content and the response time from a single URL call.

We need one final abstraction: an interface which co-ordinates the functions of IHttpJobService and IHttpJobUrlService. In other words we need a service that carries out a complete job. Insert the following interface into the JobServices folder:



    public interface IHttpJobExecutionService
    {
    	Task Execute(HttpJob httpJob);
    }


Again, it takes time to carry out a job so we'll go for asynchronous calls here as well.

If you don't understand why these methods are meant to be asynchronous and why they return Task objects then start [here][4].

**Concrete service classes**

OK, enough of abstractions, let's look at the implementations now. HttpJobService which implements IHttpJobService will work with the IJobRepository interface to carry out the repository-related actions such as insertions and updates. Insert the following class to the JobServices folder:



    public class HttpJobService : IHttpJobService
    {
    	private readonly IJobRepository _jobRepository;

    	public HttpJobService(IJobRepository jobRepository)
    	{
    		if (jobRepository == null) throw new ArgumentNullException("Job repository!!");
    		_jobRepository = jobRepository;
    	}

    	public InsertHttpJobResponse InsertHttpJob(InsertHttpJobRequest request)
    	{
    		InsertHttpJobResponse response = new InsertHttpJobResponse();
    		HttpJob job = new HttpJob();
    		job.StatusMessage = "Inserted";
    		List<JobUrl> jobUrls = new List<JobUrl>();
    		foreach (Uri uri in request.UrisToRun)
    		{
    			jobUrls.Add(new JobUrl() { Uri = uri });
    		}
    		job.UrlsToRun = jobUrls;
    		try
    		{
    			Guid correlationId = _jobRepository.InsertNewHttpJob(job);
    			response.JobCorrelationId = correlationId;
    		}
    		catch (Exception ex)
    		{
    			response.OperationException = ex;
    		}
    		return response;
    	}

    	public GetHttpJobResponse GetHttpJob(GetHttpJobRequest request)
    	{
    		GetHttpJobResponse response = new GetHttpJobResponse();

    		try
    		{
    			response.Job = _jobRepository.FindBy(request.CorrelationId);
    		}
    		catch (Exception ex)
    		{
    			response.OperationException = ex;
    		}

    		return response;
    	}

    	public UpdateHttpJobResponse UpdateHttpJob(UpdateHttpJobRequest request)
    	{
    		UpdateHttpJobResponse response = new UpdateHttpJobResponse();
    		try
    		{
    			_jobRepository.Update(request.UpdatedHttpJob);
    		}
    		catch (Exception ex)
    		{
    			response.OperationException = ex;
    		}
    		return response;
    	}

    	public GetHttpJobsResponse GetNewHttpJobs()
    	{
    		GetHttpJobsResponse response = new GetHttpJobsResponse();
    		try
    		{
    			response.HttpJobs = _jobRepository.GetUnhandledJobs();
    		}
    		catch (Exception ex)
    		{
    			response.OperationException = ex;
    		}
    		return response;
    	}
    }


I think the code is really simple and straightforward, there's nothing magic in it.

HttpJobUrlService which implements IHttpJobUrlService will need an IHttpService interface to carry out the HTTP calls. We saw the IHttpService interface in the [first post][5] of this series. Insert the following class into the JobServices folder:



    public class HttpJobUrlService : IHttpJobUrlService
    {
    	private readonly IHttpService _httpService;

    	public HttpJobUrlService(IHttpService httpService)
    	{
    		if (httpService == null) throw new ArgumentNullException("HttpService");
    		_httpService = httpService;
    	}

    	public async Task<JobUrlProcessResponse> CarryOutSingleJobUrl(Uri uri)
    	{
    		JobUrlProcessResponse response = new JobUrlProcessResponse();

    		try
    		{
    			MakeHttpCallRequest httpCallRequest = new MakeHttpCallRequest();
    			httpCallRequest.HttpMethod = HttpMethodType.Get;
    			httpCallRequest.Uri = uri;
    			DateTime start = DateTime.UtcNow;
    			MakeHttpCallResponse httpCallResponse = await httpService.MakeHttpCallAsync(httpCallRequest);
             		DateTime stop = DateTime.UtcNow;
    			TimeSpan diff = stop - start;
    			response.TotalResponseTime = diff;
    			if (!string.IsNullOrEmpty(httpCallResponse.ExceptionMessage))
    			{
    				response.HttpContent = httpCallResponse.ExceptionMessage;
    				response.HttpResponseCode = -1;
    			}
    			else
    			{
    				response.HttpContent = httpCallResponse.HttpResponse;
    				response.HttpResponseCode = httpCallResponse.HttpResponseCode;
    			}
    		}
    		catch (Exception ex)
    		{
    			response.OperationException = ex;
    		}

    		return response;
    	}
    }


We initiate a HTTP request through the injected IHttpService service and note the time between the start and end of the call. This might not be the most precise way to measure the response time of a URL but it's good enough for the demo. Then we populate the HttpContent and HttpResponseCode properties of the JobUrlProcessResponse object based on the result stored in MakeHttpCallResponse.

Finally, we have an implementation of the IHttpJobExecutionService interface. This implementation will use IHttpJobService and IHttpJobUrlService to carry out its work. Insert the following class into the JobServices folder:



    public class HttpJobExecutionService : IHttpJobExecutionService
    {
    	private readonly IHttpJobService _httpJobService;
    	private readonly IHttpJobUrlService _httpJobUrlService;

    	public HttpJobExecutionService(IHttpJobService httpJobService, IHttpJobUrlService httpJobUrlService)
    	{
    		if (httpJobService == null) throw new ArgumentNullException("HttpJobService");
    		if (httpJobUrlService == null) throw new ArgumentNullException("HttpJobUrlService");
    		_httpJobService = httpJobService;
    		_httpJobUrlService = httpJobUrlService;
    	}

    	public async Task Execute(HttpJob httpJob)
    	{
    		httpJob.Started = true;
    		httpJob.StatusMessage = string.Format("Starting job {0}.", httpJob.CorrelationId);
    		_httpJobService.UpdateHttpJob(new UpdateHttpJobRequest(httpJob));
    		TimeSpan totalTime = new TimeSpan(0, 0, 0);
    		foreach (JobUrl jobUrl in httpJob.UrlsToRun)
    		{
    			jobUrl.Started = true;
    			httpJob.StatusMessage = string.Concat("Starting url ", jobUrl.Uri);
    			_httpJobService.UpdateHttpJob(new UpdateHttpJobRequest(httpJob));
    			JobUrlProcessResponse jobUrlProcessResponse = await _httpJobUrlService.CarryOutSingleJobUrl(jobUrl.Uri);
    			jobUrl.Finished = true;
    			jobUrl.HttpContent = jobUrlProcessResponse.HttpContent.Length > 30 ? jobUrlProcessResponse.HttpContent.Substring(0, 30) : jobUrlProcessResponse.HttpContent;
    			jobUrl.HttpResponseCode = jobUrlProcessResponse.HttpResponseCode;
    			jobUrl.TotalResponseTime = jobUrlProcessResponse.TotalResponseTime;
    			httpJob.StatusMessage = string.Concat("Finished url ", jobUrl.Uri);
    			_httpJobService.UpdateHttpJob(new UpdateHttpJobRequest(httpJob));
    			totalTime += jobUrlProcessResponse.TotalResponseTime;
    		}
    		httpJob.Finished = true;
    		httpJob.TotalJobDuration = totalTime;
    		httpJob.StatusMessage = string.Format("Job {0} finished.", httpJob.CorrelationId);
    		_httpJobService.UpdateHttpJob(new UpdateHttpJobRequest(httpJob));
    	}
    }


In the Execute method we first update the job status to Started. Then we go through each URL in the UrlsToRun list of the HttpJob object. In the for-each loop we update the status of the URL to started and then save the HTTP response code and content. We also add the response time of the URL to the total response time of the job. Finally we update the job status to Finished and assign the total response time to the appropriate property.

That's it, we have the service layer ready to go.

In the [next post][6] we'll build the consumer layer which will be a simple Console app.

View the list of posts on Architecture and Patterns [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/10/09/using-a-windows-service-in-your-net-project-part-2-the-domain-and-repository/ "Using a Windows service in your .NET project part 2: the domain and repository"
[2]: http://dotnetcodr.com/2014/09/08/externalising-dependencies-with-dependency-injection-in-net-part-3-configuration/ "Externalising dependencies with Dependency Injection in .NET part 3: configuration"
[3]: http://dotnetcodr.com/task-parallel-library/ "Task Parallel Library"
[4]: http://dotnetcodr.com/2012/12/31/await-and-async-in-net-4-5-with-c/ "Await and async in .NET 4.5 with C#"
[5]: http://dotnetcodr.com/2014/10/06/using-a-windows-service-in-your-net-project-part-1-foundations/ "Using a Windows service in your .NET project part 1: foundations"
[6]: http://dotnetcodr.com/2014/10/16/using-a-windows-service-in-your-net-project-part-4-the-consumer-layer/ "Using a Windows service in your .NET project part 4: the consumer layer"
[7]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
