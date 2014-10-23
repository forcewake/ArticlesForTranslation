[Source](http://dotnetcodr.com/2013/06/13/design-patterns-and-practices-in-net-the-facade-pattern/ "Permalink to Design patterns and practices in .NET: the Facade pattern")

# Design patterns and practices in .NET: the Facade pattern

**Introduction**

Even if you've just started learning about patterns chances are the you used the Facade pattern before. You just didn't know that it had a name.

The main intention of the pattern is to hide a large, complex and possibly poorly written body of code behind a purpose-built interface. The poorly written code obviously wasn't written by you but by those other baaaad programmers you inherited the code from.

The pattern is often used in conjunction with legacy code – if you want to shield the consumers from the complexities of some old-style spaghetti code you will want to hide its methods behind a simplified interface with a couple of methods. In other words you put a facade in front of the complex code. The interface doesn't necessary cover all the functionality of the complex code, only the parts that are the most interesting and useful for a consumer. Thus the client code doesn't need to contact the complex code directly, it will communicate with it through the facade interface.

Another typical scenario is when you reference a large external library with hundreds of methods of which you only need a handful. Instead of making the other developers go through the entire library you can extract the most important functions into an interface that all calling code can use. The fact that a lot larger library sits behind the interface is not important to the caller.

It is perfectly fine to create multiple facades to factor out different chunks of functionality from a large API. The facade will also need to be extended and updated if you wish to expose more of the underlying API.

**Demo**

We'll simulate an application which looks up the temperature of our current location using several services. We want to show the temperature in Fahrenheit and Celsius as well.

Start Visual Studio and create a new Console application. We start with the simplest service which is the one that converts Fahrenheit to Celsius. Call this class MetricConverterService:



    public class MetricConverterService
    {
    	public double FarenheitToCelcius(double degrees)
    	{
    		return ((degrees - 32) / 9.0) * 5.0;
    	}
    }


Next we'll need a service that looks up our location based on a zip code:



    public class GeoLocService
    {
    	public Coordinates GetCoordinatesForZipCode(string zipCode)
    	{
    		return new Coordinates()
    		{
    			Latitude = 10,
    			Longitude = 20
    		};
    	}

    	public string GetCityForZipCode(string zipCode)
    	{
    		return "Seattle";
    	}

    	public string GetStateForZipCode(string zipCode)
    	{
    		return "Washington";
    	}
    }


I don't actually know the coordinates of Seattle, but building a true geoloc service is way beyond the scope and true purpose of this post.

The Coordinates class is very simple:



    public class Coordinates
    {
    	public double Latitude { get; set; }
    	public double Longitude { get; set; }
    }


The WeatherService is also very basic:



    public class WeatherService
    {
    	public double GetTempFarenheit(double latitude, double longitude)
    	{
    		return 86.5;
    	}
    }


We return the temperature in F based on the coordinates of the location. We of course ignore the true implementation of such a service.

The first implementation of the calling code in the Main method may look as follows:



    static void Main(string[] args)
    {
    	const string zipCode = "SeattleZipCode";

    	GeoLocService geoLookupService = new GeoLocService();

    	string city = geoLookupService.GetCityForZipCode(zipCode);
    	string state = geoLookupService.GetStateForZipCode(zipCode);
    	Coordinates coords = geoLookupService.GetCoordinatesForZipCode(zipCode);

    	WeatherService weatherService = new WeatherService();
    	double farenheit = weatherService.GetTempFarenheit(coords.Latitude, coords.Longitude);

    	MetricConverterService metricConverter = new MetricConverterService();
    	double celcius = metricConverter.FarenheitToCelcius(farenheit);

    	Console.WriteLine("The current temperature is {0}F/{1}C. in {2}, {3}",
    		farenheit.ToString("F1"),
    		celcius.ToString("F1"),
    		city,
            	state);
            Console.ReadKey();
    }


The Main method will use the 3 services we created before to perform its work. We first get back some information from the geoloc service based on the zip code. Then we ask the weather service and the metric converter service to get the temperature at that zip code in both F and C.

Run the application and you'll see some temperature info in the console.

The Main method has to do a lot of things. Getting the zip code in the beginning and writing out the information at the end are trivial tasks, we don't need to worry about them. However, the method talks to 3 potentially complicated API:s in between. The services may be dlls we downloaded from NuGet or external web services. The Main method will need to know about all three of these services/libraries to carry out its work. It will also need to know how they work, what parameters they need, in what order they must be called etc. Whereas all we really want is to take a ZIP code and get the temperature with the city and state information. It would be beneficial to hide this complexity behind an easy-to-use class or interface.

Let's insert a new interface:



    public interface ITemperatureService
    {
    	LocalTemperature GetTemperature(string zipCode);
    }


…where the LocalTemperature class looks as follows:



    public class LocalTemperature
    {
    	public double Celcius { get; set; }
    	public double Farenheit { get; set; }
    	public string City { get; set; }
    	public string State { get; set; }
    }


The interface represents the ideal way to get all information needed by the Main method. LocalTemperature encapsulates the individual bits of information.

Let's implement the interface as follows:



    public class TemperatureService : ITemperatureService
    {
    	readonly WeatherService _weatherService;
    	readonly GeoLocService _geoLocService;
    	readonly MetricConverterService _converter;

            public TemperatureService() : this(new WeatherService(), new GeoLocService(), new MetricConverterService())
    	{}

    	public TemperatureService(WeatherService weatherService, GeoLocService geoLocService, MetricConverterService converter)
    	{
    		_weatherService = weatherService;
    		_converter = converter;
    		_geoLocService = geoLocService;
    	}

    	public LocalTemperature GetTemperature(string zipCode)
    	{
    		Coordinates coords = _geoLocService.GetCoordinatesForZipCode(zipCode);
    		string city = _geoLocService.GetCityForZipCode(zipCode);
    		string state = _geoLocService.GetStateForZipCode(zipCode);

    		double farenheit = _weatherService.GetTempFarenheit(coords.Latitude, coords.Longitude);
    		double celcius = _converter.FarenheitToCelcius(farenheit);

    		LocalTemperature localTemperature = new LocalTemperature()
    		{
    			Farenheit = farenheit,
    			Celcius = celcius,
    			City = city,
    			State = state
    		};

    		return localTemperature;
    	}
    }


This is really nothing else than a refactored version of the API calls in the Main method. The dependencies on the external services have been moved to this temperature service implementation. In a more advanced solution those dependencies would be hidden behind interfaces to avoid the tight coupling between them and the Temperature Service and injected via constructor injection. Note that this class structure is not specific to the Facade pattern, so don't feel obliged to introduce an empty constructor and an overloaded one etc. The goal is to simplify the usage of those external components from the caller's point of view.

The revised Main method looks as follows:



    static void Main(string[] args)
    {
    	const string zipCode = "SeattleZipCode";

    	ITemperatureService temperatureService = new TemperatureService();
    	LocalTemperature localTemp = temperatureService.GetTemperature(zipCode);

    	Console.WriteLine("The current temperature is {0}F/{1}C. in {2}, {3}",
    						localTemp.Farenheit.ToString("F1"),
    						localTemp.Celcius.ToString("F1"),
    						localTemp.City,
    						localTemp.State);

    	Console.ReadKey();
    }


I think you agree that this is a lot more streamlined solution. So as you see the facade pattern in this case is equal to some sound refactoring of code. Run the application and you'll see the same output as before we had the facade in place.

Examples from .NET include File I/O operations such as File.ReadAllText(string filename) and the data access types such as Linq to SQL and the Entity Framework. The tedious operations of opening and closing files and database connections are hidden behind simple methods.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
