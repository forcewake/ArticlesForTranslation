[Source](http://dotnetcodr.com/2013/05/13/design-patterns-and-practices-in-net-the-decorator-design-pattern/ "Permalink to Design patterns and practices in .NET: the Decorator design pattern")

# Design patterns and practices in .NET: the Decorator design pattern

**Introduction**

The Decorator pattern aims to extend the functionality of objects at runtime. It achieves this goal by wrapping the object in a decorator class leaving the original object intact. The result is an enhanced implementation of the object which does not break any client code that's using the object.

It is an alternative solution to subclassing: you can always create subclasses to an object to extend and modify its functionality but that can result in class-explosion. Class-explosion means that you create a large number of concrete classes from a base class to allow for the combination of different possible behaviour types. The result is a bloated class structure and a hard-to-maintain application.

The pattern supports the Open/Closed principle of SOLID software design: you don't change the implementation of an existing object. Instead, you modify the object using a wrapper decorator class. All this happens dynamically at runtime, i.e. this is not a design time implementation. This yields another benefit: flexible design. Our application will be flexible enough to take on new functionality to meet changing requirements.

When is this pattern most applicable?

* Legacy systems: changing legacy code can be nasty. It's easier to modify existing code with decorators
* Add functionality to controls in Windows Forms / WPF
* Sealed classes: the pattern lets you change these classes despite they're sealed

The key to understanding this pattern is wrapping: the decorator class behaves just like the object you're trying to change by behaving like a wrapper of that object. The decorator will therefore have a reference to that object in its class structure. The pattern is also called the Wrapper pattern for exactly this reason. You can even have a chain of decorators by wrapping the decorator of the decorator of the decorator etc.

**Demo with a Base Class**

Fire up Visual Studio 2010 or 2012 and create a new Console application with the name you want. We'll simulate a travel agency that offers 3 specific types of vacations: Recreation, Beach, Activity. Insert a base abstract class called Vacation:



    public abstract class Vacation
    	{
    		public abstract string Description { get; }
    		public abstract int Price { get; }
    	}


Insert the following concrete classes:



    public class Beach : Vacation
    	{
    		public override string Description
    		{
    			get { return "Beach"; }
    		}

    		public override int Price
    		{
    			get
    			{
    				return 2;
    			}
    		}
    	}




    public class Activity : Vacation
    	{
    		public override string Description
    		{
    			get { return "Activity"; }
    		}

    		public override int Price
    		{
    			get { return 4; }
    		}
    	}




    public class Recreation : Vacation
    	{
    		public override string Description
    		{
    			get { return "Recreation"; }
    		}

    		public override int Price
    		{
    			get { return 3; }
    		}
    	}


The Main method in Program.cs is very simple:



    static void Main(string[] args)
    		{
    			Beach beach = new Beach();

    			Console.WriteLine(beach.Description);
    			Console.WriteLine("Price: {0}", beach.Price);
    			Console.ReadKey();
    		}


I believe all is clear so far. Run the programme to see the output.

As time goes by customers want to add extras to their vacations: private pool, all-inclusive, massage etc. All of these can be added to each vacation type. So you start creating concrete classes such as BeachWithPool, BeachWithPoolAndMassage, ActivityWithPoolAndMassageAndAllInclusive, right?

Well, no, not really. This would be result in the aforementioned class-explosion. The amount of concrete classes in the application would grow exponentially as we add more and more extras and vacation types. This clearly cannot be maintained. Instead we'll use the decorator pattern to decorate the concrete types with the extras.

Go ahead and insert the following decorator class:



    public class VacationDecorator : Vacation
    	{
    		private Vacation _vacation;

    		public VacationDecorator(Vacation vacation)
    		{
    			_vacation = vacation;
    		}

    		public Vacation Vacation
    		{
    			get { return _vacation; }
    		}

    		public override string Description
    		{
    			get { return _vacation.Description; }
    		}

    		public override int Price
    		{
    			get { return _vacation.Price; }
    		}
    	}


Note that the decorator also derives from the Vacation abstract class. The decorator delegates the overridden implemented getters to its wrapped Vacation object. The wrapped Vacation object is the one that's going to be decorated. This will be the base class for the concrete decorators. So let's start with the PrivatePool decorator:



    public class PrivatePool : VacationDecorator
    	{
    		public PrivatePool(Vacation vacation) : base(vacation){}

    		public override string Description
    		{
    			get
    			{
    				return string.Concat(base.Vacation.Description, ", Private pool.");
    			}
    		}

    		public override int Price
    		{
    			get
    			{
    				return base.Vacation.Price + 2;
    			}
    		}
    	}


This concrete decorator derives from the VacationDecorator class. It overrides the Description and Price property getters by extending those of the wrapped Vacation object. Let's try to add a private pool to our beach holiday in Program.cs:



    static void Main(string[] args)
    		{
    			Vacation beach = new Beach();
    			beach = new PrivatePool(beach);

    			Console.WriteLine(beach.Description);
    			Console.WriteLine("Price: {0}", beach.Price);
    			Console.ReadKey();
    		}


Note that we first create a new Beach object, but declare its type as Vacation. Then we take the PrivatePool decorator and use it as a wrapper for the Beach. The rest of the Main method has remained unchanged. Run the programme and you'll see that the description and total price include the Beach and PrivatePool descriptions and prices. Note that when we wrap the Beach object in the PrivatePool decorator it becomes of type PrivatePool instead of the original Beach.

Let's create two more decorators:



    public class Massage : VacationDecorator
    	{
    		public Massage(Vacation vacation) : base(vacation){}

    		public override string Description
    		{
    			get
    			{
    				return string.Concat(base.Vacation.Description, ", Massage.");
    			}
    		}

    		public override int Price
    		{
    			get
    			{
    				return base.Vacation.Price + 1;
    			}
    		}
    	}




    public class AllInclusive : VacationDecorator
    	{
    		public AllInclusive(Vacation vacation) : base(vacation)
    		{}

    		public override string Description
    		{
    			get
    			{
    				return string.Concat(base.Vacation.Description, ", All inclusive.");
    			}
    		}

    		public override int Price
    		{
    			get
    			{
    				return base.Vacation.Price + 3;
    			}
    		}
    	}


These two decorators have the same structure as PrivatePool, but the description and price getters have been modified of course. We're now ready to add the all-inclusive and massage extras to our Beach vacation:



    static void Main(string[] args)
    		{
    			Vacation beach = new Beach();
    			beach = new PrivatePool(beach);
    			beach = new AllInclusive(beach);
    			beach = new Massage(beach);

    			Console.WriteLine(beach.Description);
    			Console.WriteLine("Price: {0}", beach.Price);
    			Console.ReadKey();
    		}


We keep wrapping the decorators until the original Beach object has been wrapped in a PrivatePool, an AllInclusive and a Massage decorator. The final type is Massage. The call to the Description and Price property getters will simply keep going "upwards" in the decorator chain. Run the programme and you'll see that the description and price getters include all 4 inputs.

Now if the travel agency wants to introduce a new type of extra, all they need to do is to build a new concrete class deriving from the VacationDecorator.

The original Beach class is completely unaware of the presence of decorators. In addition, we can mix and match vacations types with their decorators. Our class structure is easy to extend and maintain.

**Demo with an interface**

You can apply the decorator pattern to interface-type abstractions as well. As you now understand the components of the pattern let's go through one quick example.

Insert a class called Product and an interface called IProductService. Leave the product class empty. Insert a method into the interface:



    public interface IProductService
    	{
    		IEnumerable<Product> GetProducts();
    	}


Add a class called ProductService that implements IProductService:



    public class ProductService : IProductService
    	{
    		public IEnumerable<Product> GetProducts()
    		{
    			return new List<Product>();
    		}
    	}


Using poor man's dependency injection we can use the components as follows:



    IProductService productService = new ProductService();
    IEnumerable<Product> products = productService.GetProducts();
    Console.ReadKey();


Nothing awfully complicated there I hope. Let's say that you want to add caching to the Product service so that you don't need to ask the product repository for the list of products every time. One option is of course to add caching directly into the body of the implemented GetProducts method. However, that is not the most optimal decision: it goes against the Single Responsibility Principle and reduces meaningful testability. It's not straightforward to unit test a method that caches the results in its body. When you run the unit test twice, then you may not be able to get a reliable result from the method as it simply returns the cached outcome. Injecting some kind of caching strategy is a good way to go, check out the related [Adapter][1] pattern for more.

However, here we'll take another approach. We want to extend the functionality of the Product service with a decorator. So let's follow the same path as we did in the base class demo. Insert a decorator base as follows:



    public class ProductServiceDecorator : IProductService
    	{
    		private readonly IProductService _productService;

    		public ProductServiceDecorator(IProductService productService)
    		{
    			_productService = productService;
    		}

    		public IProductService ProductService { get { return _productService; } }

    		public virtual IEnumerable<Product> GetProducts()
    		{
    			return _productService.GetProducts();
    		}
    	}


We follow the same principle as with the VacationDecorator class: we wrap an IProductService and call upon its implementation of GetProducts(). Let's add the following cache decorator – you'll need to add a reference to System.Runtime.Caching in order to have access to the ObjectCache class:



    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Runtime.Caching;
    using System.Text;
    using System.Threading.Tasks;

    namespace Decorator
    {
    	public class ProductServiceCacheDecorator : ProductServiceDecorator
    	{

    		public ProductServiceCacheDecorator(IProductService productService)
    			: base(productService)
    		{ }

    		public override IEnumerable<Product> GetProducts()
    		{
    			ObjectCache cache = MemoryCache.Default;
    			string key = "products";
    			if (cache.Contains(key))
    			{
    				return (IEnumerable<Product>)cache[key];
    			}
    			else
    			{
    				IEnumerable<Product> products = base.ProductService.GetProducts();
    				CacheItemPolicy policy = new CacheItemPolicy();
    				policy.AbsoluteExpiration = new DateTimeOffset(DateTime.Now.AddMinutes(1));
    				cache.Add(key, products, policy);
    				return products;
    			}
    		}
    	}
    }


We check the cache for the presence of a key. If it's there then retrieve its contents otherwise ask the IProductRepository of the decorator base to carry out the query to the repository and cache the results. You can use these components as follows:



    IProductService productService = new ProductService();
    productService = new ProductServiceCacheDecorator(productService);
    IEnumerable<Product> products = productService.GetProducts();
    Console.ReadKey();


Run the programme and you'll see that ProductServiceCacheDecorator takes over, runs the caching strategy and retrieves the products from the wrapped IProductService object.

View the list of posts on Architecture and Patterns [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[2]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
