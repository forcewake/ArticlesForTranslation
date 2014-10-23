[Source](http://dotnetcodr.com/2014/09/08/externalising-dependencies-with-dependency-injection-in-net-part-3-configuration/ "Permalink to Externalising dependencies with Dependency Injection in .NET part 3: configuration")

# Externalising dependencies with Dependency Injection in .NET part 3: configuration

**Introduction**

In the [previous post][1] of this series we looked at how to remove the caching logic from a class and hide it behind an interface. We'll follow in a similar fashion in this post which takes up reading settings from a configuration source.

We'll build upon the demo we've been working on so far so have it ready in Visual Studio.

**Configuration and settings**

Probably every even vaguely serious application out there has some configuration store. The configuration values contain the settings that are necessary so that the app can function normally. A good example from .NET is the web.config or app.config file with their appSettings section:



    <appSettings>
        <add key="Environment" value="Alpha" />
        <add key="MinimumLimit" value="1000" />
        <add key="Storage" value="mongo" />
    </appSettings>


In Java you'd put these settings into a .properties file. The configuration files will allow you to change the behaviour of your app without recompiling and redeploying them.

You can of course store your settings in other sources: a database, a web service or some other mechanism. Hence the motivations for hiding the concrete technology behind an abstraction are similar to those listed in the post on caching: testability, flexibility, SOLID.

**The abstraction**

Application settings are generally key-value pairs so constructing an interface for a settings storage is fairly simple and the below example is one option. The key is almost always a string and the value can be any primitive or a string. Add a new folder called Configuration to the infrastructure layer. Add the following interface to it:



    public interface IConfigurationRepository
    {
    	T GetConfigurationValue<T>(string key);
    	T GetConfigurationValue<T>(string key, T defaultValue);
    }


**The implementation**

One obvious implementation is to read the values from app.config/web.config using the ConfigurationManager class in the System.Configuration library. Add this dll to the references list of the infrastructure layer.

Then insert the following implementation of the interface to the Configuration folder:



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
    }


The console app has an app.config file by default. If not then you'll need to add an Application Configuration File as it's called in the Add New Item dialog. We'll simulate a test scenario where it's possible to return a default product instead of consulting the data store. Add the following appSettings section with the configuration tags of app.config:



    <appSettings>
    	<add key="returnDefaultProduct" value="true"/>
    	<add key="defaultProductId" value="99"/>
    	<add key="defaultProductName" value="DEFAULT"/>
    	<add key="defaultProductQuantity" value="1000"/>
    </appSettings>


Next we'll inject the dependency through the constructor of ProductService. Modify the ProductService private fields and constructor as follows:



    private readonly IProductRepository _productRepository;
    private readonly IConfigurationRepository _configurationRepository;

    public ProductService(IProductRepository productRepository, IConfigurationRepository configurationRepository)
    {
    	if (productRepository == null) throw new ArgumentNullException("ProductRepository");
    	if (configurationRepository == null) throw new ArgumentNullException("ConfigurationRepository");
    	_productRepository = productRepository;
    	_configurationRepository = configurationRepository;
    }


This breaks of course the existing code which constructs a ProductService. Modify it as follows:



    IProductService productService = new ProductService(new ProductRepository(), new ConfigFileConfigurationRepository());


The modify ProductService.GetProduct as follows to use the configuration repository:



    public GetProductResponse GetProduct(GetProductRequest getProductRequest)
    {
    	GetProductResponse response = new GetProductResponse();
    	try
    	{
    		bool returnDefault = _configurationRepository.GetConfigurationValue<bool>("returnDefaultProduct", false);
    		Product p = returnDefault ? BuildDefaultProduct() : _productRepository.FindBy(getProductRequest.Id);
    		response.Product = p;
    		if (p != null)
    		{
    			response.Success = true;
    		}
    		else
    		{
    			response.Exception = "No such product.";
    		}
    	}
    	catch (Exception ex)
    	{
    		response.Exception = ex.Message;
    	}
    	return response;
    }

    private Product BuildDefaultProduct()
    {
    	Product defaultProduct = new Product();
    	defaultProduct.Id = _configurationRepository.GetConfigurationValue<int>("defaultProductId");
    	defaultProduct.Name = _configurationRepository.GetConfigurationValue<string>("defaultProductName");
    	defaultProduct.OnStock = _configurationRepository.GetConfigurationValue<int>("defaultProductQuantity");
    	return defaultProduct;
    }


Run Main and you'll see that the configuration values are indeed correctly retrieved from app.config. A possible improvement here is to put all configuration keys into a separate container class or a resource file instead of putting them here directly. Then you could retrieve them by writing something like this:



    defaultProduct.Id = _configurationRepository.GetConfigurationValue<int>(ConfigurationKeys.DefaultProductId);


In the next post we'll look at logging.

View the list of posts on Architecture and Patterns [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/09/04/externalising-dependencies-with-dependency-injection-in-net-part-2-caching/ "Externalising dependencies with Dependency Injection in .NET part 2: caching"
[2]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
