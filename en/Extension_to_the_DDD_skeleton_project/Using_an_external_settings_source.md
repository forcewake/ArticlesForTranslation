[Source](http://dotnetcodr.com/2014/03/27/extension-to-the-ddd-skeleton-project-using-an-external-settings-source/ "Permalink to Extension to the DDD skeleton project: using an external settings source")

# Extension to the DDD skeleton project: using an external settings source

**Introduction**

In [this][1] post we saw a way how to introduce caching in the DDD project. We also mentioned that hardcoding the caching duration directly in code may not be desirable for testing and maintenance purposes.

One way to externalise the caching duration is the following setup:

* We can have a settings file where we set up the caching profiles, such as LongCache = 1 minute, MediumCache = 30sec, ShortCache = 10sec.
* These profiles can be saved in different places: web.config or app.config, database, customised XML/JSON file, even an external service
* This is telling us that retrieving the correct setting should be abstracted away

**Infrastructure**

Open the Infrastructure.Common layer and insert a new folder called Configuration. Insert the following interface:



    public interface IConfigurationRepository
    {
            T GetConfigurationValue<T>(string key);
            string GetConfigurationValue(string key);
            T GetConfigurationValue<T>(string key, T defaultValue);
            string GetConfigurationValue(string key, string defaultValue);
    }


We'll go for the most obvious implementation: the appSettings section in web.config. Insert the following class into the folder:



    public class AppSettingsConfigurationRepository : IConfigurationRepository
    {
            public T GetConfigurationValue<T>(string key)
            {
                return GetConfigurationValue(key, default(T), true);
            }

            public string GetConfigurationValue(string key)
            {
                return GetConfigurationValue<string>(key);
            }

            public T GetConfigurationValue<T>(string key, T defaultValue)
            {
                return GetConfigurationValue(key, defaultValue, false);
            }

            public string GetConfigurationValue(string key, string defaultValue)
            {
                return GetConfigurationValue<string>(key, defaultValue);
            }

            private T GetConfigurationValue<T>(string key, T defaultValue, bool throwException)
            {
                var value = ConfigurationManager.AppSettings[key];
                if (value == null)
                {
                    if(throwException)
                        throw new KeyNotFoundException("Key "+key+ " not found.");
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
                    if (throwException)
                        throw ex;
                    return defaultValue;
                }
            }
    }


You'll need to set a reference to the System.Configuration library otherwise ConfigurationManager will not be found.

This is an application of the [Adapter pattern][2] which we used in the previous post.

**Application service**

Locate the EnrichedCustomerService class from the previous post on Service layer caching. Add the following private field:



    private readonly IConfigurationRepository _configurationRepository;


Modify the constructor as follows:



    public EnrichedCustomerService(ICustomerService innerCustomerService, ICacheStorage cacheStorage, IConfigurationRepository configurationRepository)
    {
    	if (innerCustomerService == null) throw new ArgumentNullException("CustomerService");
    	if (cacheStorage == null) throw new ArgumentNullException("CacheStorage");
    	if (configurationRepository == null) throw new ArgumentNullException("ConfigurationRepository");
    	_innerCustomerService = innerCustomerService;
    	_cacheStorage = cacheStorage;
    	_configurationRepository = configurationRepository;
    }


We want to read the cache duration setting in GetAllCustomers. Modify the method as follows:



    public GetCustomersResponse GetAllCustomers()
    {
    	string key = "GetAllCustomers";
    	GetCustomersResponse response = _cacheStorage.Retrieve<GetCustomersResponse>(key);
    	if (response == null)
    	{
    		int cacheDurationSeconds = _configurationRepository.GetConfigurationValue<int>("ShortCacheDuration");
    		response = _innerCustomerService.GetAllCustomers();
    		_cacheStorage.Store(key, response, TimeSpan.FromSeconds(cacheDurationSeconds));
    	}
    	return response;
    }


It would be better to have some external container for string values such as ShortCacheDuration, but this will be enough for now.

**Web layer**

Open the web.config file and enter the following appSetting:



    <add key="ShortCacheDuration" value="60"/>


We also need to instruct StructureMap to select AppSettingsConfigurationRepository for any IConfigurationRepository it encounters. Enter the following line into the ObjectFactory.Initialize block of IoC.cs:



    x.For<IConfigurationRepository>().Use<AppSettingsConfigurationRepository>();


Set a breakpoint within GetAllCustomers() of EnrichedCustomerService.cs. Set the start page to "customers": open the Properties of the WebService layer, select Web, Specific Page, "customers". Start the application.

Code execution will stop at the break point. Step through the code by pressing F11. You'll see how the settings are extracted from the config file and used with the cache storage mechanism.

Now every time you want to change the cache duration you just update the appropriate settings value. There's nothing stopping you from inserting different cache duration profiles in your configuration source as mentioned above: ShortCache, LongCache, MediumCache etc. Also, we solved yet another dependency problem in an OOP and SOLID way.

Read the next extension to the DDD project [here][3].

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/03/24/extension-to-the-ddd-skeleton-project-caching-in-the-service-layer/ "Extension to the DDD skeleton project: caching in the service layer"
[2]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[3]: http://dotnetcodr.com/2014/03/31/extension-to-the-ddd-skeleton-project-per-session-object-context-in-repository-layer/ "Extension to the DDD skeleton project: per session object context in Repository layer"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
