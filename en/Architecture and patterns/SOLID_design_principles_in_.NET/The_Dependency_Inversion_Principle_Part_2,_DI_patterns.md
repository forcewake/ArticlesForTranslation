[Source](http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "Permalink to SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns")

# SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns

In the [previous][1] post we went through the basics of DIP and DI. We'll continue the discussion with the following questions: how to implement DI?

**Flavours of DI**

I hinted at the different forms of DI in the previous post. By far the most common form is called **Constructor Injection**:



    public ProductService(IProductRepository productRepository)
    {
    	_productRepository = productRepository;
    }


Constructor injection is the perfect way to ensure that the necessary dependency is always available to the class. We force the clients to supply some kind of implementation of the interface. All you need is a public constructor that accepts the dependency as a parameter. In the constructor we save the incoming concrete class for later use, i.e. assign to a private variable.

We can introduce a second level of security by introducing a guard clause:



    public ProductService(IProductRepository productRepository)
    {
           if (productRepository == null) throw new ArgumentNullException("ProductRepo");
    	_productRepository = productRepository;
    }


It is good practice to mark the private field readonly, in this case the _productRepository, as it guarantees that once the initialisation logic of the constructor has executed it cannot be modified by any other code:



    private readonly IProductRepository _productRepository;


This is not required for DI to work but it protects you against mistakes such as setting the value to null someplace else in your code.

Constructor injection is a good way to document your class to the clients. The clients will see that ProductService will need an IProductRepository to perform its job. There's no attempt to hide this fact.

When the constructor is finished then the object is guaranteed to have a valid dependency, i.e. the object is in a consistent state. The following method will not choke on a null value where _productRepository is called:



    public IEnumerable<Product> GetProducts()
    {
    	IEnumerable<Product> productsFromDataStore = _productRepository.FindAll();
    	return productsFromDataStore;
    }


We don't need to test for null within the GetProducts method as we know it is guaranteed to be in a consistent state.

Constructor injection should be your default DI technique in case your class has a dependency and no reasonable local defaults exist. A **local default** is an acceptable local implementation of an abstract dependency in case none is injected by the client. We'll talk more about this a bit later. Also, try to avoid overloaded constructors because then you'll need to rely on those local defaults if an empty constructor is used. In addition, having just one constructor greatly simplifies the usage of automated _DI containers_, also called **IoC containers**. If you don't know what they are, then don't worry about them for now. They are not a prerequisite for DIP and DI and they cannot magically turn tightly coupled code into a loosely coupled one. Make sure that you first understand how to do DI without such tools.

You can read briefly about them [here][2]. In short they are 'magic' tools that can initialise dependencies. In the above example you can configure such a container, such as [StructureMap][3] to magically inject a MyDefaultProductRepository when it sees that an IProductRepository is needed.

The next DI type in line is called **Property injection**. Property injection is used when your class has a good local default for a dependency but still want to enable clients to override that default. It is also called Setter injection:



    public IProductRepository ProductRepository { get; set; }


In this case you obviously cannot mark the backing field readonly.

This implementation is fragile as by default all object types are set to null so ProductRepository will also be null. You'll need to extend the property setter as follows:



    public IProductRepository ProductRepository
    {
    	get
    	{
    		return _productRepository;
    	}
    	set
    	{
    		if (value == null) throw new ArgumentNullException("ProductRepo");
    		_productRepository = value;
    	}
    }


We're still not done. There's nothing forcing the client to call this setter so the GetProducts method will throw an exception. At some point we must initialise the dependency, maybe in the constructor:



    public ProductService()
    {
    	_productRepository = new DefaultProductRepository();
    }


By now we know that initialising an object with the 'new' keyword increases coupling between two objects so whenever possible accept the dependency in form of an abstraction instead.

Alternatively you can take the Lazy Initialisation approach in the property getter:



    public IProductRepository ProductRepository
    {
    	get
    	{
    		if (_productRepository == null)
    		{
    			_productRepository = new ProductRepository();
    		}
    		return _productRepository;
    	}
    	set
    	{
    		if (value == null) throw new ArgumentNullException("ProductRepo");
    		_productRepository = value;
    	}
    }


We can go even further. If you want that the client should only set the dependency once without the possibility to alter it during the object's lifetime then the following approach can work:



    public IProductRepository ProductRepository
    {
    	get
    	{
    		if (_productRepository == null)
    		{
    			_productRepository = new ProductRepository();
    		}
    		return _productRepository;
    	}
    	set
    	{
    		if (value == null) throw new ArgumentNullException("ProductRepo");
    		if (_productRepository != null) throw new InvalidOperationException("You are not allowed to set this dependency more than once.");
    		_productRepository = value;
    	}
    }


BTW this doesn't mean that from now on you must never again use the new keyword to initialise objects. At some point you'll HAVE TO use it as you can't connect the bits and pieces using abstractions only. E.g. this is invalid code:



    ProductService ps = new ProductService(new IProductRepository());


No, unless you have some IoC Container in place you'll need to go with what's called the **poor man's dependency injection**:



    ProductService ps = new ProductService(new DefaultProductRepository());


Where DefaultProductRepository implements IProductRepository.

Coming back to Property injection, you can use it whenever there's a well functioning default implementation of a dependency but you still want to let your users provide their own implementation. In other words the dependency is optional. However, this choice must be a conscious one. As there's nothing forcing the clients to call the property setter your code shouldn't complain if the dependency is null when it's needed. That's a sign saying that you need to turn to constructor injection instead. In case you don't really need a local default then the [Null Object pattern][4] can come handy. If the local default is part of the .NET base class library then it may be an acceptable approach to use it.

As you see the last version of the property getter-setter is considerably more complex than taking the constructor injection approach. It looks easy at first, just insert a get;set; type of property but you soon notice that it's a fragile structure.

A third way of doing DI is called **method injection**. It is used when we want to ensure that we can inject a different implementation every time the dependency is used:



    public IEnumerable<Product> GetProducts(IProductDiscountStrategy productDiscount)
    {
    	IEnumerable<Product> productsFromDataStore = _productRepository.FindAll();
    	foreach (Product p in productsFromDataStore)
    	{
    		p.AdjustPrice(productDiscount);
    	}
    	return productsFromDataStore;
    }


Here we apply a discount strategy on each product in the iteration. The actual strategy may change a lot, it can depend on the season, the loyalty scheme, the weather, the time of the day etc. As usual you can include a guard clause:



    public IEnumerable<Product> GetProducts(IProductDiscountStrategy productDiscount)
    {
    	if (productDiscount == null) throw new ArgumentNullException("Discount strategy");
    	IEnumerable<Product> productsFromDataStore = _productRepository.FindAll();
    	foreach (Product p in productsFromDataStore)
    	{
    		p.AdjustPrice(productDiscount);
    	}
    	return productsFromDataStore;
    }


In this approach it's easy to vary the concrete discount strategy every time we call the GetProducts method. If you had to inject this into the constructor then you'd need to create a new ProductService class every time you want to apply some pricing strategy. In case the method doesn't use the injected dependency you won't need a guard clause. This may sound strange at first; why have a dependency in the signature if it is not used in the method body? Occasionally you're forced to implement an interface where the interface method defines the dependency – more on that in the post about the [Interface Segregation Principle][5].

Patterns related to method injection are [Factory][6] and [Strategy][7]. Choosing the proper implementation to be injected into the GetProducts method will almost certainly depend on other inputs such as the choices the user makes on the UI. These patterns will help you solve that problem.

An example of method injection from .NET is the Contains extension method from LINQ:



    bool IEnumerable<T>.Contains(T object, IEqualityComparer<T> comparer);


You can then provide your own version of the equality comparer.

The last type of DI we'll discuss is making dependencies available through a **static accessor**. It is also called injection through the **ambient context**. It is used when implementing cross-cutting concerns.

What are cross-cutting concerns? Operations that may be performed in many unrelated classes and that are not closely related to those classes. A classic example is logging. You may want to log exceptions, performance data etc. I may want to log the time it takes the GetProducts method to return in the ProductService class. You could inject an ILogger interface to every class that needs logging but it introduces a large amount of pollution across your objects. Also, an ILogger interface attached to a constructor is not truly necessary for the dependent object to perform its real job. Logging is usually not part of the core job of services and repositories.

If you're a web developer then you must have come across the HttpContext object at some point. It is an application of the ambient context pattern. You can always try and access that object and get hold of the current HTTP context which may or may not be available right there and then by its Current static property. You can construct your own Context object where you retrieve the current context using threads. A time provider context may look as follows:



    public abstract class TimeProviderContext
    {
    	public static TimeProviderContext Current
    	{
    		get
    		{
    			TimeProviderContext timeProviderContext = Thread.GetData(Thread.GetNamedDataSlot("TimeProvider")) as TimeProviderContext;
    			if (timeProviderContext == null)
    			{
    				timeProviderContext = TimeProviderContext.DefaultTimeProviderContext;
    				Thread.SetData(Thread.GetNamedDataSlot("TimeProvider"), timeProviderContext);
    			}
    			return timeProviderContext;
    		}
    		set
    		{
    			Thread.SetData(Thread.GetNamedDataSlot("TimeProvider"), value);
    		}
    	}
    	public static TimeProviderContext DefaultTimeProviderContext = new DotNetTimeProvider();

    	public abstract DateTime GetTime { get; }
    }


…where DotNetTimeProvider looks as follows:



    public class DotNetTimeProvider : TimeProviderContext
    {
    	public override DateTime GetTime
    	{
    		get { return DateTime.Now; }
    	}
    }


The TimeProviderContext is abstract and has a static Current property to get hold of the current context. That's the classic setup of the ambient context. Using the Thread object like that will ensure that each thread has its own context. There's a default implementation which is the standard DateTime class from .NET. It's important to note that the Current property must be writable so that clients can assign their own time providers by deriving from the TimeProviderContext object. This can be helpful in unit testing where ideally you want to control the time instead of waiting for some specific date. The local default makes sure that the client doesn't get a null reference exception when calling the Current property.

For simplicity's sake I only put a single abstract method in this class to get the date but ambient context classes can provide as many properties as you need.

You can then use this context in client classes as follows:



    public DateTime TestTime()
    {
    	return TimeProviderContext.Current.GetTime;
    }


Ambient context should be used with care. Use it only if you want to make sure that a cross-cutting concern is available anywhere throughout your application. In that case it may be futile to force objects to take on dependencies they may not need now but might do so at some point in the future:



    public IProductRepository ProductRepository
    {
    	get
    	{
    		if (_productRepository == null)
    		{
    			_productRepository = new ProductRepository(TimeProviderContext.Current);
    		}
    		return _productRepository;
    	}
    }


…where ProductRepository looks as follows:



    public class ProductRepository : IProductRepository
    {

    	public ProductRepository(TimeProviderContext timeProviderContext)
    	{
    		//do nothing with the time provider context right now but it may be needed later
    	}

    	public IEnumerable<Product> FindAll()
    	{
    		return new List<Product>();
    	}
    }


That only increases the coupling between objects and pollutes your object structure.

A disadvantage with the ambient context approach is that the consuming class carries an implicit dependency. In other words it hides from the clients that it needs a time provider in order to perform its job. If you put the TestTime() method shown above in the ProductService class then there's no way for the client to tell just by looking at the interface that ProductService uses this dependency. Also, callers of the TestTime method will get different results depending on the actual context and it may not be transparent to them why this happens without looking at the source code.

There's actually a technical term to describe this "openness" that Eric Evans came up with: **Intention-revealing interfaces**. An API should communicate what it does by its public interface alone. A class using the ambient context does exactly the opposite. Clients may know of the existence of a TimeProviderContext class but will probably not know that it is used in ProductService.

There are cross cutting concerns that only perform some task without returning an answer: logging exceptions and performance data are such cases. If you have that type of scenario then a technique called **Interception** is something to consider. We'll look at interception briefly it the blog post after the next one.

**Conclusion**

Of the four techniques discussed your default choice should always be Constructor Injection in case there's a dependency within your class. If the dependency varies from operation to operation then Method Injection is good candidate.

Then you can ask the question if the dependency represents a cross-cutting concern. If not and a good local default exists then you can go down the Property Injection path. Otherwise if you need some return value from the cross-cutting concern dependency and you have a good local default in case Context.Current is null then Ambient Context can help. Else if this dependency only has void methods then take a look at Interception.

When in doubt, especially if you are trying to pick a strategy between Constructor Injection and another variant then pick Constructor injection. Things can never go fatally wrong with that option.

View the list of posts on Architecture and Patterns [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[2]: http://www.hanselman.com/blog/ListOfNETDependencyInjectionContainersIOC.aspx "IoC containers"
[3]: http://docs.structuremap.net/ "StructureMap homepage"
[4]: http://dotnetcodr.com/2013/05/06/design-patterns-and-practices-in-net-the-null-object-pattern/ "Design patterns and practices in .NET: the Null Object pattern"
[5]: http://dotnetcodr.com/2013/08/22/solid-design-principles-in-net-the-interface-segregation-principle/ "SOLID design principles in .NET: the Interface Segregation Principle"
[6]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[7]: http://dotnetcodr.com/2013/04/29/design-patterns-and-practices-in-net-the-strategy-pattern/ "Design patterns and practices in .NET: the Strategy Pattern"
[8]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
