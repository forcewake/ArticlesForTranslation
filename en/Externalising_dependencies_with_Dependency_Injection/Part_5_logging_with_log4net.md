[Source](http://dotnetcodr.com/2014/09/15/externalising-dependencies-with-dependency-injection-in-net-part-5-logging-with-log4net/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 5: logging with log4net")

# Externalising dependencies with Dependency Injection in .NET part 5: logging with log4net

**Introduction**

In the [previous][1] post we looked at how to hide the concrete implementation of the logging technology behind an abstraction. In fact we reached the original goal of showing how to rid the code of logging logic or at least how to make the code less dependent on it.

In the posts on [caching][2] and [configuration][3] we also looked at some real implementations of the abstractions that you can readily use in your project. However, with logging we only provided a simple Console based logging which is far from realistic in any non-trivial application. Therefore I've decided to extend the discussion on logging with a real powerhouse: [log4net][4] by Apache.

Log4net is a well-established, general purpose and widely used logging framework for .NET. You can set it up to send logging messages to multiple targets: console, file, database, a web service etc. In this post we'll look at how to log to a file using log4net.

In a real-life large web-based application you would likely log to at least 2 sources: a file or a database and another, more advanced tool which helps you search among the messages in an efficient way. An example of such a tool is [GrayLog][5], a web-based application where you can set up your channels and make very quick and efficient searches to track your messages.

The primary source of investigation in case of exception tracking will be this advanced tool. However, as in the case of GrayLog it may be down in which case the log messages are lost. As a backup you can then read the log messages from the log file. As mentioned above, we'll be looking into file-based logging but if you're looking for a more professional tool then I can recommend GrayLog.

NB: I'm not going to go through log4net specific details too much so be prepared to do your own search in some areas. A full description and demo of log4net would deserve its own series which is out of bounds in this case. However, the goal is to provide code that you can readily use in your project without much modification.

We'll build upon the CommonInfrastructureDemo project we've been working with so far so have it open in Visual Studio.

**Some basics**

Let's go through some preparations first. The log4net library is available through NuGet. Add the following NuGet package to the Infrastructure.Common project:

![log4net NuGet][6]

By the time you read this post the version may be higher but hopefully it won't have any breaking changes.

As mentioned above, log4net can be configured to send the log messages to a variety of sources. Log4net will have one or more so-called **appenders** that will "append" the message to the defined source. Log4net can be configured in code or via a separate configuration file. The advantage of a configuration file is that you can modify the values on the deploy server without re-deploying the application. There are numerous examples on the internet showing snippets of log4net configurations. A very good starting point is the documentation on the log4net homepage available [here][7].

In our case we'll go for the RollingFileAppender. If the log file exceeds a certain limit then the oldest messages are erased to make room for the new ones. Add an Application Configuration File file called log4net.config to the root of the Console app, i.e. to the same level as the app.config file. Erase any default content in log4net.config and instead add the following XML content:



    <?xml version="1.0"?>
    <log4net>
    	<appender name="RollingFileAppender" type="log4net.Appender.RollingFileAppender">
    		<file value="log.xml"/>
    		<threshold value="INFO" />
    		<appendToFile value="true" />
    		<rollingStyle value="Size" />
    		<maxSizeRollBackups value="30" />
    		<maximumFileSize value="30MB" />
    		<staticLogFileName value="true" />
    		<lockingModel type="log4net.Appender.FileAppender+MinimalLock" />
    		<layout type="log4net.Layout.XMLLayout" />
    	</appender>

    	<root>
    		<level value="ALL" />
    		<!-- Value of priority may be ALL, DEBUG, INFO, WARN, ERROR, FATAL, OFF -->
    		<appender-ref ref="RollingFileAppender"/>
    	</root>
    </log4net>


You can find the full documentation of the rolling file appender [here][8] and [here][9]. The main thing to note that we want to log to a file called log.xml. In Solution Explorer right-click log4net.config and select Properties. Locate the "Copy to Output Directory" in the Properties window and select "Copy always". This will put the config file to the bin folder when the application is compiled.

We'll need to store the name of the log4net config file name in the application configuration file which we added in the post on configuration referred to above. Add the following app setting to app.config:



    <add key="Log4NetSettingsFile" value="log4net.config"/>


The log4net implementation of ILoggingService will therefore need an IConfigurationRepository we saw before. It's good that we have an implementation of IConfigurationRepository that reads from the app.config file so we'll be able to use it.

However, we need something more. Whenever you're trying to track down what exactly went wrong in the application based on a series of log messages you'll need all sorts of contextual information: the user, the session, the user agent, the referrer, the exact version of the browser, the requested URL etc. Add a new folder to Infrastructure.Common called ContextProvider and insert an interface called IContextService into it:



    public interface IContextService
    {
    	string GetContextualFullFilePath(string fileName);
    	string GetUserName();
    	ContextProperties GetContextProperties();
    }


GetContextualFullFilePath will help us find the full path to a physical file after the deployment of the application. In our case we want to be able to find log4net.config. GetUserName is probably self-explanatory. All other context properties will be stored in the ContextProperties object. Add the following class to the ContextProvider folder:



    public class ContextProperties
    {
    	private string _notAvailable = "N/A";

    	public ContextProperties()
    	{
    		UserAgent = _notAvailable;
    		RemoteHost = _notAvailable;
    		Path = _notAvailable;
    		Query = _notAvailable;
    		Referrer = _notAvailable;
    		RequestId = _notAvailable;
    		SessionId = _notAvailable;
    	}

    	public string UserAgent { get; set; }
    	public string RemoteHost { get; set; }
    	public string Path { get; set; }
    	public string Query { get; set; }
    	public string Referrer { get; set; }
    	public string RequestId { get; set; }
    	public string SessionId { get; set; }
    	public string Method { get; set; }
    }


In a web-based application with a valid HttpContext object we can have the following implementation. Add the following class to the ContextProvider folder:



    public class HttpContextService : IContextService
    {
    	public HttpContextService()
    	{
    		if (HttpContext.Current == null)
    		{
    			throw new ArgumentException("There's no available Http context.");
    		}
    	}

    	public string GetContextualFullFilePath(string fileName)
    	{
    		return HttpContext.Current.Server.MapPath(string.Concat("~/", fileName));
    	}

    	public string GetUserName()
    	{
    		string userName = "<null>";
    		try
    		{
    			if (HttpContext.Current != null && HttpContext.Current.User != null)
    			{
    				userName = (HttpContext.Current.User.Identity.IsAuthenticated
    								? HttpContext.Current.User.Identity.Name
    								: "<null>");
    			}
    		}
    		catch
    		{
    		}
    		return userName;
    	}

    	public ContextProperties GetContextProperties()
    	{
    		ContextProperties props = new ContextProperties();
    		if (HttpContext.Current != null)
    		{
    			HttpRequest request = null;
    			try
    			{
    				request = HttpContext.Current.Request;
    			}
    			catch (HttpException)
    			{
    			}
             		if (request != null)
    			{
    				props.UserAgent = request.Browser == null ? "" : request.Browser.Browser;
    				props.RemoteHost = request.ServerVariables == null ? "" : request.ServerVariables["REMOTE_HOST"];
    				props.Path = request.Url == null ? "" : request.Url.AbsolutePath;
    				props.Query = request.Url == null ? "" : request.Url.Query;
    				props.Referrer = request.UrlReferrer == null ? "" : request.UrlReferrer.ToString();
    				props.Method = request.HttpMethod;
    			}

    			IDictionary items = HttpContext.Current.Items;
    			if (items != null)
    			{
    				var requestId = items["RequestId"];
    				if (requestId != null)
    				{
    					props.RequestId = items["RequestId"].ToString();
    				}
    			}

    			var session = HttpContext.Current.Session;
    			if (session != null)
    			{
    				var sessionId = session["SessionId"];
    				if (sessionId != null)
    				{
    					props.SessionId = session["SessionId"].ToString();
    				}
    			}
    		}

    		return props;
    	}
    }


Most of this code is about extracting various data from the HTTP request/context.

In a non-HTTP based application we'll go for a simpler implementation. Add a class called ThreadContextService to the ContextProvider folder:



    public class ThreadContextService : IContextService
    {
    	public string GetContextualFullFilePath(string fileName)
    	{
    		string dir = Directory.GetCurrentDirectory();
    		FileInfo resourceFileInfo = new FileInfo(Path.Combine(dir, fileName));
    		return resourceFileInfo.FullName;
    	}

    	public string GetUserName()
    	{
    		string userName = "<null>";
    		try
    		{
    			if (Thread.CurrentPrincipal != null)
    			{
    				userName = (Thread.CurrentPrincipal.Identity.IsAuthenticated
    								? Thread.CurrentPrincipal.Identity.Name
    								: "<null>");
    			}
    		}
    		catch
    		{
    		}
    		return userName;
    	}

    	public ContextProperties GetContextProperties()
    	{
    		return new ContextProperties();
    	}
    }


Now we have all the ingredients for the log4net implementation if ILoggingService. Add the following class called Log4NetLoggingService to the Logging folder:



    public class Log4NetLoggingService : ILoggingService
    {
    	private readonly IConfigurationRepository _configurationRepository;
    	private readonly IContextService _contextService;
    	private string _log4netConfigFileName;

    	public Log4NetLoggingService(IConfigurationRepository configurationRepository, IContextService contextService)
    	{
    		if (configurationRepository == null) throw new ArgumentNullException("ConfigurationRepository");
    		if (contextService == null) throw new ArgumentNullException("ContextService");
    		_configurationRepository = configurationRepository;
    		_contextService = contextService;
    		_log4netConfigFileName = _configurationRepository.GetConfigurationValue<string>("Log4NetSettingsFile");
    		if (string.IsNullOrEmpty(_log4netConfigFileName))
    		{
    			throw new ApplicationException("Log4net settings file missing from the configuration source.");
    		}
    		SetupLogger();
    	}

    	private void SetupLogger()
    	{
    		FileInfo log4netSettingsFileInfo = new FileInfo(_contextService.GetContextualFullFilePath(_log4netConfigFileName));
    		if (!log4netSettingsFileInfo.Exists)
    		{
    			throw new ApplicationException(string.Concat("Log4net settings file ", _log4netConfigFileName, " not found."));
    		}
    		log4net.Config.XmlConfigurator
    			.ConfigureAndWatch(log4netSettingsFileInfo);
    	}

    	public void LogInfo(object logSource, string message, Exception exception = null)
    	{
    		LogMessageWithProperties(logSource, message, Level.Info, exception);
    	}

    	public void LogWarning(object logSource, string message, Exception exception = null)
    	{
    		LogMessageWithProperties(logSource, message, Level.Warn, exception);
    	}

    	public void LogError(object logSource, string message, Exception exception = null)
    	{
    		LogMessageWithProperties(logSource, message, Level.Error, exception);
    	}

    	public void LogFatal(object logSource, string message, Exception exception = null)
    	{
    		LogMessageWithProperties(logSource, message, Level.Fatal, exception);
    	}

    	private void LogMessageWithProperties(object logSource, string message, Level level, Exception exception)
    	{
    		var logger = LogManager.GetLogger(logSource.GetType());

    		var loggingEvent = new LoggingEvent(logSource.GetType(), logger.Logger.Repository, logger.Logger.Name, level, message, null);
    		AddProperties(logSource, exception, loggingEvent);
    		try
    		{
    			logger.Logger.Log(loggingEvent);
    		}
    		catch (AggregateException ae)
    		{
    			ae.Handle(x => { return true; });
    		}
    		catch (Exception) { }
    	}

    	private string GetUserName()
    	{
    		return _contextService.GetUserName();
    	}

    	private void AddProperties(object logSource, Exception exception, LoggingEvent loggingEvent)
    	{
    		loggingEvent.Properties["UserName"] = GetUserName();
    		try
    		{
    			ContextProperties contextProperties = _contextService.GetContextProperties();
    			if (contextProperties != null)
    			{
    				try
    				{
    					loggingEvent.Properties["UserAgent"] = contextProperties.UserAgent;
    					loggingEvent.Properties["RemoteHost"] = contextProperties.RemoteHost;
    					loggingEvent.Properties["Path"] = contextProperties.Path;
    					loggingEvent.Properties["Query"] = contextProperties.Query;
    					loggingEvent.Properties["RefererUrl"] = contextProperties.Referrer;
    					loggingEvent.Properties["RequestId"] = contextProperties.RequestId;
    					loggingEvent.Properties["SessionId"] = contextProperties.SessionId;
    				}
    				catch (Exception)
    				{
    				}
    			}

    			loggingEvent.Properties["ExceptionType"] = exception == null ? "" : exception.GetType().ToString();
    			loggingEvent.Properties["ExceptionMessage"] = exception == null ? "" : exception.Message;
    			loggingEvent.Properties["ExceptionStackTrace"] = exception == null ? "" : exception.StackTrace;
    			loggingEvent.Properties["LogSource"] = logSource.GetType().ToString();
    		}
    		catch (Exception ex)
    		{
    			var type = typeof(Log4NetLoggingService);
    			var logger = LogManager.GetLogger(type);
    			logger.Logger.Log(type, Level.Fatal, "Exception when extracting properties: " + ex.Message, ex);
    		}
    	}
    }


We inject two dependencies by way of constructor injection: IConfigurationRepository and IContextService. IConfigurationRepository will help us locate the name of the log4net configuration file. IContextService will provide contextual data to the log message. Log4net is set up in SetupLogger(). We check for the existence of the config file. We then call log4net.Config.XmlConfigurator.ConfigureAndWatch to let log4net read the configuration settings from an XML file and watch for any changes.

The implemented LogX messages all call upon LogMessageWithProperties where we simply wrap the log message in a LoggingEvent object along with all contextual data.

Originally we had the below code to instantiate a logged and cached ProductService to log to the console:



    IProductService productService = new ProductService(new ProductRepository(), new ConfigFileConfigurationRepository());
    IProductService cachedProductService = new CachedProductService(productService, new SystemRuntimeCacheStorage());
    IProductService loggedCachedProductService = new LoggedProductService(cachedProductService, new ConsoleLoggingService());


We can easily change the logging mechanism by simply injecting the log4net implementation of ILoggingService into LoggedProductService:



    IConfigurationRepository configurationRepository = new ConfigFileConfigurationRepository();
    IProductService productService = new ProductService(new ProductRepository(), configurationRepository);
    IProductService cachedProductService = new CachedProductService(productService, new SystemRuntimeCacheStorage());
    ILoggingService loggingService = new Log4NetLoggingService(configurationRepository, new ThreadContextService());
    IProductService loggedCachedProductService = new LoggedProductService(cachedProductService, loggingService);


Run Program.cs and if all went well then you'll have a new file called log.xml in the …MyCompany.ProductConsole/bin/Debug folder with some XML entries. Here comes an excerpt:



    <log4net:event logger="MyCompany.ProductConsole.Services.LoggedProductService" timestamp="2014-08-23T22:44:52.7591938+02:00" level="INFO" thread="9" domain="MyCompany.ProductConsole.vshost.exe" username="andras.nemes"><log4net:message>Starting GetProduct method</log4net:message><log4net:properties><log4net:data name="log4net:HostName" value="andras1" /><log4net:data name="Path" value="N/A" /><log4net:data name="SessionId" value="N/A" /><log4net:data name="log4net:UserName" value="andras.nemes" /><log4net:data name="Query" value="N/A" /><log4net:data name="ExceptionMessage" value="" /><log4net:data name="UserName" value="&lt;null&gt;" /><log4net:data name="RefererUrl" value="N/A" /><log4net:data name="LogSource" value="MyCompany.ProductConsole.Services.LoggedProductService" /><log4net:data name="RemoteHost" value="N/A" /><log4net:data name="ExceptionStackTrace" value="" /><log4net:data name="UserAgent" value="N/A" /><log4net:data name="ExceptionType" value="" /><log4net:data name="RequestId" value="N/A" /><log4net:data name="log4net:Identity" value="" /></log4net:properties></log4net:event>


So it took some time to implement the log4net version of ILoggingService but to switch from the console-based implementation was a breeze.

However, you may still see one log message logged to the console. That's from the code line in ProductService.GetProduct:



    LogProviderContext.Current.LogInfo(this, "Log message from the contextual log provider");


…where LogProviderContext returns a ConsoleLoggingService by default. However, it also allows us to set the implementation of ILoggingService. Insert the following code in Main…



    LogProviderContext.Current = loggingService;


…just below…



    ILoggingService loggingService = new Log4NetLoggingService(configurationRepository, new ThreadContextService());


Then you'll see that even the previously mentioned log message ends up in log.xml:



    <log4net:event logger="MyCompany.ProductConsole.Services.ProductService" timestamp="2014-08-23T22:56:19.411468+02:00" level="INFO" thread="8" domain="MyCompany.ProductConsole.vshost.exe" username="andras.nemes"><log4net:message>Log message from the contextual log provider</log4net:message><log4net:properties><log4net:data name="log4net:HostName" value="andras1" /><log4net:data name="Path" value="N/A" /><log4net:data name="SessionId" value="N/A" /><log4net:data name="log4net:UserName" value="andras.nemes" /><log4net:data name="Query" value="N/A" /><log4net:data name="ExceptionMessage" value="" /><log4net:data name="UserName" value="&lt;null&gt;" /><log4net:data name="RefererUrl" value="N/A" /><log4net:data name="LogSource" value="MyCompany.ProductConsole.Services.ProductService" /><log4net:data name="RemoteHost" value="N/A" /><log4net:data name="ExceptionStackTrace" value="" /><log4net:data name="UserAgent" value="N/A" /><log4net:data name="ExceptionType" value="" /><log4net:data name="RequestId" value="N/A" /><log4net:data name="log4net:Identity" value="" /></log4net:properties></log4net:event>


So that's it about logging. We'll continue with the file system in the [next part][10].

View the list of posts on Architecture and Patterns [here][11].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/11/externalising-dependencies-with-dependency-injection-in-net-part-4-logging-part-1/ "Externalising dependencies with Dependency Injection in .NET part 4: logging part 1"
[2]: http://dotnetcodr.com/2014/09/04/externalising-dependencies-with-dependency-injection-in-net-part-2-caching/ "Externalising dependencies with Dependency Injection in .NET part 2: caching"
[3]: http://dotnetcodr.com/2014/09/08/externalising-dependencies-with-dependency-injection-in-net-part-3-configuration/ "Externalising dependencies with Dependency Injection in .NET part 3: configuration"
[4]: http://logging.apache.org/log4net/ "Apache log4net homepage"
[5]: http://graylog2.org/ "GrayLog homepage"
[6]: http://dotnetcodr.files.wordpress.com/2014/08/log4net-nuget.png?w=300&h=57
[7]: http://logging.apache.org/log4net/release/config-examples.html "log4net configuration examples"
[8]: http://logging.apache.org/log4net/release/sdk/log4net.Appender.RollingFileAppender.html "Rolling file appender on Apache"
[9]: http://logging.apache.org/log4net/release/sdk/log4net.Appender.RollingFileAppenderMembers.html "Rolling file appender members on Apache"
[10]: http://dotnetcodr.com/2014/09/18/externalising-dependencies-with-dependency-injection-in-net-part-6-file-system/ "Externalising dependencies with Dependency Injection in .NET part 6: file system"
[11]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
