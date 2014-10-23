[Source](http://dotnetcodr.com/2014/10/09/using-a-windows-service-in-your-net-project-part-2-the-domain-and-repository/ "Permalink to Using a Windows service in your .NET project part 2: the domain and repository")

# Using a Windows service in your .NET project part 2: the domain and repository

**Introduction**

In the [previous][1] post we went through the goals of this series and also started building a demo app. At present we have an Infrastructure layer in the solution to abstract away the service that makes the HTTP calls. As we said the demo project will carry out HTTP calls and measure the response times.

In this post we'll build the domain and repository layers. We'll build upon the demo project so have it ready in Visual Studio.

**The domain**

The domain will be quite small with only two domain objects and one abstract repository. Add a C# library to the solution called Demo.Domain. Add the following object to it:



    public class JobUrl
    {
    	public Uri Uri { get; set; }
    	public int HttpResponseCode { get; set; }
    	public string HttpContent { get; set; }
    	public bool Started { get; set; }
    	public bool Finished { get; set; }
    	public TimeSpan TotalResponseTime { get; set; }
    }


JobUrl has the following properties:

* Uri: the Uri to call
* HttpResponseCode: the HTTP response from the call such as 200, 403 etc.
* HttpContent: the pure HTML content of the URI
* Started: whether the HTTP call to the URI has been initiated
* Finished: whether the HTTP call has been processed
* TotalResponseTime: the response time of the URL

We want to be able to carry out multiple HTTP calls within a single job so we'll need the following "wrapper" object:



    public class HttpJob
    {
    	public List<JobUrl> UrlsToRun { get; set; }
    	public string StatusMessage { get; set; }
    	public bool Started { get; set; }
    	public bool Finished { get; set; }
    	public TimeSpan TotalJobDuration { get; set; }
    	public Guid CorrelationId { get; set; }

    	public override string ToString()
    	{
    		string NL = Environment.NewLine;
    		StringBuilder sb = new StringBuilder();
    		sb.Append("Status report of job with correlation id: ").Append(CorrelationId).Append(NL);
    		sb.Append("------------------------------------------").Append(NL);
    		sb.Append("Started: ").Append(Started).Append(NL);
    		sb.Append("Finished: ").Append(Finished).Append(NL);
    		sb.Append("Status: ").Append(StatusMessage).Append(NL).Append(NL);

    		foreach (JobUrl jobUrl in UrlsToRun)
    		{
    			sb.Append(NL);
    			sb.Append("Url: ").Append(jobUrl.Uri).Append(NL);
    			sb.Append("Started: ").Append(jobUrl.Started).Append(NL);
    			sb.Append("Finished: ").Append(jobUrl.Finished).Append(NL);
    			if (jobUrl.Finished)
    			{
    				sb.Append("Http response code: ").Append(jobUrl.HttpResponseCode).Append(NL);
    				sb.Append("Partial content: ").Append(jobUrl.HttpContent).Append(NL);
    				sb.Append("Response time: ").Append(Convert.ToInt32(jobUrl.TotalResponseTime.TotalMilliseconds)).Append(" ms").Append(NL);
    			}
    		}

    		if (Finished)
    		{
    			sb.Append("Total job duration: ").Append(Convert.ToInt32(TotalJobDuration.TotalMilliseconds)).Append(" ms. ").Append(NL);
    		}
    		sb.Append("------------------------------------------").Append(NL).Append(NL);

    		return sb.ToString();
    	}
    }


HttpJob has the following properties:

* UrlsToRun: the URIs to be visited in the job
* StatusMessage: a status message such as "starting" or "in progress"
* Started: whether the job has been started
* Finished: whether the job has been finished
* TotalJobDuration: the sum of the response times of the individual URLs
* CorrelationId: in essence the job ID which can be used to retrieve the job from the data store

Then we have an overridden ToString method which builds a long string to show the current status of the job. We'll see it in action towards the end of the series.

We'll also need an [abstract repository][2] where we declare the capabilities of the job repository. This abstraction must be then implemented by a concrete repository such as Entity Framework or NHibernate. Add the following interface to the Domain layer:



    public interface IJobRepository
    {
    	Guid InsertNewHttpJob(HttpJob httpJob);
    	HttpJob FindBy(Guid correlationId);
    	void Update(HttpJob httpJob);
    	IEnumerable<HttpJob> GetUnhandledJobs();
    }


So in this interface we declare that any concrete repository implementation should be able to do the following:

* Insert a new HttpJob and return a correlation ID
* Find a job based on the correlation ID
* Update a job
* Find all "unhandled" jobs i.e. the ones that have not been started yet

This concludes the Domain layer.

**The repository layer**

As mentioned in the previous post we'll use the file-based MongoDb as the data storage mechanism. Don't worry if you've never used it, it's very easy to get going and you'll learn something new. We won't see much of MongoDb in action here anyway. I have a series devoted to MongoDb that you can go through if you wish, the list of posts is available [here][3].

In order to move on though you'll need to install MongoDb. If you want to keep it short then go through the section called "Setup on Windows" in [part two][4] of the MongoDb series. In case you want to read the foundations then here's [part one][5] for you.

At the end of the setup process you should have MongoDb up and running as a Windows service:

![MongoDb as Windows Service][6]

Add a new C# library called Demo.Repository.MongoDb to the solution. Add a project reference to the Domain and Infrastructure layers. We'll also need to work with MongoDb in a programmatic way so add the following Nuget package:

![MongoDb C# driver][7]

Add a new folder called DatabaseObjects and insert a class called DbHttpJobs into that folder:



    public class DbHttpJob : HttpJob
    {
    	public ObjectId Id { get; set; }
    }


**Let's stop for a second**

**What on Earth is this???** We can save just about any object in MongoDb on one condition: the object must have a property called Id which will be used as the primary key for the object. The most straightforward implementation for the Id is the MongoDb ObjectId type which is similar to a GUID. We won't work with it at all but it's a must for MongoDb so we simply add it to the DbHttpJob object which inherits everything else from HttpJob.

**Why not use the correlation ID as the ID in the DB???** We could certainly do that by decorating the CorrelationId property of the HttpJob object with the BsonId attribute. However, that would require us to add a reference to the MongoDb C# driver in the Domain layer and add that MongoDb-specific attribute to our POCO model(s).

**And so what???** If you've gone through [the series on Domain Driven Design][8] then you'll understand the concept of POCO classes that are oblivious of the underlying storage mechanism. This is called **persistence ignorance**. A POCO should be as POCO as possible. As soon as we add data storage specific elements to a POCO then it's not really POCO any more I think. There will be traces of the concrete repository in the POCO, so at best it can be called a contaminated POCO. The domain layer will have a tight coupling to the underlying concrete repository which could be avoided. In case you'd like to share the class with someone else who also needs to work with the domain then that person will also inherit MongoDb from you whereas they might work with an entirely different data storage mechanism.

If we go down that path and introduce MongoDb at the domain level then it can be difficult to change the storage mechanism later. Say that you want to store the HttpJob objects in Sql Server with Entity Framework or in Amazon S3 later on. Then you'll first have to decouple the Domain layer from MongoDb and replace it with another technology, right? Well, hopefully you won't make the same mistake again and keep the technology-specific code in the technology-specific layer.

**OK, let's move on**

After some theory we'll move on with the practical discussion. A MongoDb repository will need two things in order to find the database: the connection string and the database name. These elements will most likely be stored in a settings file like app.config or web.config but they could come from any other store: a database, a web service etc. This sounds like we'll need to abstract away the service that finds the connection-related elements.

Let's return briefly to the Infrastructure layer. Add a new folder called DatabaseConnectionSettingsService and insert the following interface:



    public interface IDatabaseConnectionSettingsService
    {
    	string GetDatabaseConnectionString();
    	string GetDatabaseName();
    }


Back in the repository layer we can build the abstract base class for all repositories. Insert the following class to the project:



    public abstract class MongoDbRepository
    {
    	private readonly IDatabaseConnectionSettingsService _databaseConnectionSettingsService;
    	private MongoClient _mongoClient;
    	private MongoServer _mongoServer;
    	private MongoDatabase _mongoDatabase;

    	public MongoDbRepository(IDatabaseConnectionSettingsService databaseConnectionSettingsService)
    	{
    		if (databaseConnectionSettingsService == null) throw new ArgumentNullException();
    		_databaseConnectionSettingsService = databaseConnectionSettingsService;
    		_mongoClient = new MongoClient(_databaseConnectionSettingsService.GetDatabaseConnectionString());
    		_mongoServer = _mongoClient.GetServer();
    		_mongoDatabase = _mongoServer.GetDatabase(_databaseConnectionSettingsService.GetDatabaseName());
    	}

    	public MongoDatabase HttpJobsDatabase
    	{
    		get
    		{
    			return _mongoDatabase;
    		}
    	}

    	public MongoCollection<DbHttpJob> HttpJobs
    	{
    		get
    		{
    			return HttpJobsDatabase.GetCollection<DbHttpJob>("httpjobs");
    		}
    	}
    }


You'll recognise the IDatabaseConnectionSettingsService interface which is injected to the MongoDbRepository constructor. I won't go into the MongoDb-specific details here, just keep in mind the following:

* A "collection" in MongoDb is similar to a "table" in MS SQL
* The collection name "httpjobs" could be anything, like "mickeymouse", it doesn't matter really – the type variable declares the type of objects stored in the collection
* There's no need to check for the presence of a database or a collection – they will be created by MongoDb upon the first data entry
* The HttpJobs property is similar to DbContext.Cars in EntityFramework i.e. it gets a reference to the collection of DbHttpJobs in the database

We're now ready to implement the IJobRepository interface. Add the following class to the project:



    public class JobRepository : MongoDbRepository, IJobRepository
    {
    	public JobRepository(IDatabaseConnectionSettingsService databaseConnectionSettingsService)
    		: base(databaseConnectionSettingsService)
    	{}

    	public Guid InsertNewHttpJob(HttpJob httpJob)
    	{
    		Guid correlationId = Guid.NewGuid();
    		httpJob.CorrelationId = correlationId;
    		DbHttpJob dbHttpJob = httpJob.ConvertToInsertDbObject();
    		HttpJobs.Insert(dbHttpJob);
    		return correlationId;
    	}

    	public HttpJob FindBy(Guid correlationId)
    	{
    		return FindInDb(correlationId);
    	}

    	public void Update(HttpJob httpJob)
    	{
    		DbHttpJob existing = FindInDb(httpJob.CorrelationId);
    		existing.Finished = httpJob.Finished;
    		existing.Started = httpJob.Started;
    		existing.StatusMessage = httpJob.StatusMessage;
    		existing.TotalJobDuration = httpJob.TotalJobDuration;
    		existing.UrlsToRun = httpJob.UrlsToRun;
    		HttpJobs.Save(existing);
    	}

    	public IEnumerable<HttpJob> GetUnhandledJobs()
    	{
    		IMongoQuery query = Query<DbHttpJob>.EQ(j => j.Started, false);
    		return HttpJobs.Find(query);
    	}

    	private DbHttpJob FindInDb(Guid correlationId)
    	{
    		IMongoQuery query = Query<DbHttpJob>.EQ(j => j.CorrelationId, correlationId);
    		DbHttpJob firstJob = HttpJobs.FindOne(query);
    		return firstJob;
    	}
    }


Again, I won't go into the MongoDb specific code. It is quite easy to follow anyway as it's only about retrieving, inserting and updating a HttpJob object.

There's one [extension method][9] called ConvertToInsertDbObject() which looks as follows:



    namespace Demo.Repository.MongoDb
    {
    	public static class ModelExtensions
    	{
    		public static DbHttpJob ConvertToInsertDbObject(this HttpJob domain)
    		{
    			return new DbHttpJob()
    			{
    				CorrelationId = domain.CorrelationId
    				, Finished = domain.Finished
    				, Started = domain.Started
    				, StatusMessage = domain.StatusMessage
    				, TotalJobDuration = domain.TotalJobDuration
    				, UrlsToRun = domain.UrlsToRun
    				, Id = ObjectId.GenerateNewId()
    			};
    		}
    	}
    }


You can put this anywhere in the MongoDb layer. The method simply constructs a new DbHttpJob object with a new ObjectId ready for insertion. The generation of the Id is not really necessary as MongoDb will create one for us in case it's null but I prefer to be specific.

We're done with the repository layer. In the [next post][10] we'll look at the Service layer.

View the list of posts on Architecture and Patterns [here][11].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/10/06/using-a-windows-service-in-your-net-project-part-1-foundations/ "Using a Windows service in your .NET project part 1: foundations"
[2]: http://dotnetcodr.com/2013/09/23/a-model-net-web-service-based-on-domain-driven-design-part-4-the-abstract-repository/ "A model .NET web service based on Domain Driven Design Part 4: the abstract Repository"
[3]: http://dotnetcodr.com/data-storage/ "Data storage"
[4]: http://dotnetcodr.com/2014/07/31/mongodb-in-net-part-2-setup/ "MongoDB in .NET part 2: setup"
[5]: http://dotnetcodr.com/2014/07/28/mongodb-in-net-part-1-foundations/ "MongoDB in .NET part 1: foundations"
[6]: http://dotnetcodr.files.wordpress.com/2014/09/mongodb-as-windows-service.png?w=300&h=23
[7]: http://dotnetcodr.files.wordpress.com/2014/09/mongodb-c-driver.png?w=300&h=72
[8]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[9]: http://dotnetcodr.com/2014/03/06/extension-methods-part-1-the-basics/ "Extension methods in C# .NET part 1: the basics"
[10]: http://dotnetcodr.com/2014/10/13/using-a-windows-service-in-your-net-project-part-3-the-application-service-layer/ "Using a Windows service in your .NET project part 3: the application service layer"
[11]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
