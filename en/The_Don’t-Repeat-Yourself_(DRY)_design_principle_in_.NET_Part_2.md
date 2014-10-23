[Source](http://dotnetcodr.com/2013/10/21/the-dont-repeat-yourself-dry-design-principle-in-net-part-2/ "Permalink to The Don’t-Repeat-Yourself (DRY) design principle in .NET Part 2")

# The Don’t-Repeat-Yourself (DRY) design principle in .NET Part 2

We'll continue with our discussion of the Don't-Repeat-Yourself principle where we left off in the [previous][1] post. The next issue we'll consider is repetition of logic.

_Repeated logic_

Consider that you have the following two domain objects:



    public class Product
    {
    	public long Id { get; set; }
    	public string Description { get; set; }
    }




    public class Order
    {
    	public long Id { get; set; }
    }


Let's say that the IDs are not automatically assigned when inserting a new row in the database. Instead, it must be calculated. So you come up with the following function to construct an ID which is probably unique:



    private long CalculateId()
    {
    	TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    	long id = Convert.ToInt64(ts.TotalMilliseconds);
    	return id;
    }


You might include this type of logic in both domain objects:



    public class Product
    {
    	public long Id { get; set; }
    	public string Description { get; set; }

    	public Product()
    	{
    		Id = CalculateId();
    	}

    	private long CalculateId()
    	{
    		TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    		long id = Convert.ToInt64(ts.TotalMilliseconds);
    		return id;
    	}
    }




    public class Order
    {
    	public long Id { get; set; }

    	public Order()
    	{
    		Id = CalculateId();
    	}

    	private long CalculateId()
    	{
    		TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    		long id = Convert.ToInt64(ts.TotalMilliseconds);
    		return id;
    	}
    }


This situation may arise if the two domain objects have been added to your application with a long time delay and you've forgotten about the ID generation solution. Also, if you want to keep the ID generation logic independent for each object, then you might continue with this solution thinking that some day the ID generation strategies may be different. However, at some point the rules change and all IDs of type long must be constructed using the CalculateId method. Then you absolutely want to have this logic in one place only. Otherwise if the rule changes, then you probably don't want to make the same change for every single domain object, right?

Probably a very common solution would be to factor out this logic to a static method:



    public class IdHelper
    {
    	public static long CalculateId()
    	{
    		TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    		long id = Convert.ToInt64(ts.TotalMilliseconds);
    		return id;
    	}
    }


The updated objects look as follows:



    public class Order
    {
    	public long Id { get; set; }

    	public Order()
    	{
    		Id = IdHelper.CalculateId();
    	}
    }




    public class Product
    {
    	public long Id { get; set; }
    	public string Description { get; set; }

    	public Product()
    	{
    		Id = IdHelper.CalculateId();
    	}
    }


If you've followed through the discussion on the [SOLID][2] design principles then you'll know by now that static methods can be a design smell that indicate tight coupling. In this case there's a hard dependency of the Product and Order classes on IdHelper.

If all objects in your domain must have an ID of type long then you may let every object derive from a superclass such as this:



    public abstract class EntityBase
    {
    	public long Id { get; private set; }

    	public EntityBase()
    	{
    		Id = CalculateId();
    	}

    	private long CalculateId()
    	{
    		TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    		long id = Convert.ToInt64(ts.TotalMilliseconds);
    		return id;
    	}
    }


The Product and Order objects will derive from this class:



    public class Product : EntityBase
    {
    	public string Description { get; set; }

    	public Product()
    	{}
    }




    public class Order : EntityBase
    {
    	public Order()
    	{}
    }


Then if you construct a new Order or Product class elsewhere then the ID will be assigned by the EntityBase constructor automatically.

In case you don't like the base class approach then [Constructor injection][3] is another approach that can work. We delegate the ID generation logic to an external class which we hide behind an interface:



    public interface IIdGenerator
    {
    	long CalculateId();
    }


We have the following implementing class:



    public class DefaultIdGenerator : IIdGenerator
    {
    	public long CalculateId()
    	{
    		TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    		long id = Convert.ToInt64(ts.TotalMilliseconds);
    		return id;
    	}
    }


You can inject the interface dependency into the Order object as follows:



    public class Order
    {
    	private readonly IIdGenerator _idGenerator;
    	public long Id { get; private set; }

    	public Order(IIdGenerator idGenerator)
    	{
    		if (idGenerator == null) throw new ArgumentNullException();
    		_idGenerator = idGenerator;
    		Id = _idGenerator.CalculateId();
    	}
    }


You can apply the same method to the Product object. Of course you can mix the above two solutions with the following EntityBase superclass:



    public abstract class EntityBase
    {
    	private readonly IIdGenerator _idGenerator;

    	public long Id { get; private set; }

    	public EntityBase(IIdGenerator idGenerator)
    	{
    		if (idGenerator == null) throw new ArgumentNullException();
    		_idGenerator = idGenerator;
    		Id = _idGenerator.CalculateId();
    	}
    }


These are some of the possible solutions that you can employ to factor out common logic so that it becomes available for different objects. Obviously if this logic occurs only within the same class then just simply create a private method for it:



    private void DoRepeatedLogic()
    {
    	Order order = new Order();
    	TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    	long orderId = Convert.ToInt64(ts.TotalMilliseconds);
    	order.Id = orderId;

    	Product product = new Product();
    	ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    	long productId = Convert.ToInt64(ts.TotalMilliseconds);
    	product.Id = productId;
    }


This is of course not very clever and you can quickly make it better:



    private void DoRepeatedLogic()
    {
    	Order order = new Order();
    	order.Id = CalculateId();

    	Product product = new Product();
    	product.Id = CalculateId();
    }

    private long CalculateId()
    {
    	TimeSpan ts = DateTime.UtcNow - (new DateTime(1970, 1, 1, 0, 0, 0));
    	long id = Convert.ToInt64(ts.TotalMilliseconds);
    	return id;
    }


This is more likely to occur in long classes and methods where you lose track of all the code you've written. At some point you realise that some logic is repeated over and over again but it's rooted deeply nested in a long, complicated method.

_If statements_

If statements are very important building blocks of an application. It would probably be impossible to write any real life app without them. However, it does not mean they should be used without any limitation. Consider the following domains:



    public abstract class Shape
    {
    }

    public class Triangle : Shape
    {
    	public int Base { get; set; }
    	public int Height { get; set; }
    }

    public class Rectangle : Shape
    {
    	public int Width { get; set; }
    	public int Height { get; set; }
    }


Then in Program.cs of a Console app we can simulate a database lookup as follows:



    private static IEnumerable&lt;Shape&gt; GetAllShapes()
    {
    	List&lt;Shape&gt; shapes = new List&lt;Shape&gt;();
    	shapes.Add(new Triangle() { Base = 5, Height = 3 });
    	shapes.Add(new Rectangle() { Height = 6, Width = 4 });
    	shapes.Add(new Triangle() { Base = 9, Height = 5 });
    	shapes.Add(new Rectangle() { Height = 3, Width = 2 });
    	return shapes;
    }


Say you want to calculate the total area of the shapes in the collection. The first approach may look like this:



    private static double CalculateTotalArea(IEnumerable&lt;Shape&gt; shapes)
    {
    	double area = 0.0;
    	foreach (Shape shape in shapes)
    	{
    		if (shape is Triangle)
    		{
    			Triangle triangle = shape as Triangle;
    			area += (triangle.Base * triangle.Height) / 2;
    		}
    		else if (shape is Rectangle)
    		{
    			Rectangle recangle = shape as Rectangle;
    			area += recangle.Height * recangle.Width;
    		}
    	}
    	return area;
    }


This is actually quite a common approach in a software design where our domain objects are mere collections of properties and are void of any self-contained logic. Look at the Triangle and Rectangle classes, they contain no logic whatsoever, they only have properties. They are reduced to the role of data-transfer-objects (DTOs). If you don't understand at first what's wrong with the above solution then I suggest you go through the Liskov Substitution Principle [here][4]. I won't repeat what's written in that post.

This post is about DRY so you may ask what this method has to do with DRY at all as we do not seem to repeat anything. Yes we do, although indirectly. Our initial intention was to create a class hierarchy so that we can work with the abstract class Shape elsewhere. Well, guess what, we've failed miserably. In this method we need to reveal not only the concrete implementation types of Shape but we're forcing an external class to know about the internals of those concrete types.

This is a typical example for how not to use if statements in software. In the posts on the SOLID design principles we mentioned the Tell-Don't-Ask (TDA) principle. It basically states that you should not ask an object questions about its current state before you ask it to perform something. Well, this piece of code is a clear violation of TDA although the lack of logic in the Triangle and Rectangle classes forced us to ask these questions.

The solution – or at least one of the viable solutions – will be to hide this calculation logic behind each concrete Shape class:



    public abstract class Shape
    {
    	public abstract double CalculateArea();
    }

    public class Triangle : Shape
    {
    	public int Base { get; set; }
    	public int Height { get; set; }

    	public override double CalculateArea()
    	{
    		return (Base * Height) / 2;
    	}
    }

    public class Rectangle : Shape
    {
    	public int Width { get; set; }
    	public int Height { get; set; }

    	public override double CalculateArea()
    	{
    		return Width * Height;
    	}
    }


The updated total area calculation looks as follows:



    private static double CalculateTotalArea(IEnumerable&lt;Shape&gt; shapes)
    {
    	double area = 0.0;
    	foreach (Shape shape in shapes)
    	{
    		area += shape.CalculateArea();
    	}
    	return area;
    }


We've got rid of the if statements, we don't violate TDA and the logic to calculate the area is hidden behind each concrete type. This allows us even to follow the above mentioned Liskov Substitution Principle.

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/10/17/the-dont-repeat-yourself-dry-design-principle-in-net-part-1/ "The Don't-Repeat-Yourself (DRY) design principle in .NET Part 1"
[2]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[3]: http://dotnetcodr.com/2013/08/29/solid-design-principles-in-net-the-dependency-inversion-principle-part-2-di-patterns/ "SOLID design principles in .NET: the Dependency Inversion Principle Part 2, DI patterns"
[4]: http://dotnetcodr.com/2013/08/19/solid-design-principles-in-net-the-liskov-substitution-principle/ "SOLID design principles in .NET: the Liskov Substitution Principle"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
