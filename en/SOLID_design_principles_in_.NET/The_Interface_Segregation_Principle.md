[Source](http://dotnetcodr.com/2013/08/22/solid-design-principles-in-net-the-interface-segregation-principle/ "Permalink to SOLID design principles in .NET: the Interface Segregation Principle")

# SOLID design principles in .NET: the Interface Segregation Principle

**Introduction**

The Interface Segregation Principle (ISP) states that clients should not be forced to depend on interfaces they do not use. ISP is about breaking down big fat master-interfaces to more specialised and cohesive ones that group related functionality. Imagine that your class needs some functionality from an interface but not all. If you have no direct access to that interface then all you can do is to implement the relevant bits and ignore the methods that are not relevant for your class.

The opposite is that an interface grows to allow for more functionality where there's a chance that some methods in the interface are not related, e.g. CalculateArea() and DoTheShopping(). All implementing classes will of course need to implement all methods in the interface where the unnecessary ones may only have a throw new NotImplementedException as their method body.

For the caller this won't be obvious:



    Person.DoTheShopping();
    Neighbour.DoTheShopping();

    Triangle.CalculateArea();
    Square.CalculateArea();


These look perfectly reasonable, right? However, if the interface they implement looks like this…:



    public interface IMasterInterface
    {
        void DoTheShopping();
        double CalculateArea();
    }


…then the client may as well make the following method calls:



    Person.CalculateArea();
    Triangle.DoTheShopping();


…where the true implementations will probably throw a NotImplementedException or will have an empty body or return some default value such as 0 at best. Also, if you loop through an enumeration of IMasterInterface objects then the caller expects to be able to simply call the CalculateArea() on every object in the list adhering to the LSP principle of the [previous][1] post. Then they will see exceptions thrown or inexplicable 0's returned where the method does not apply.

The above example is probably too extreme, nobody would do anything like that in real code I hope. However, there are more subtle differences between two 'almost related' objects than between a Person and a Triangle as we'll see in the demo.

Often such master interfaces are a result of a growing domain where more and more properties and behaviour are assigned to the domain objects without giving much thought to the true hierarchy between classes. Adhering to ISP will often result in small interfaces with only 1-3 methods that concentrate on some very specific tasks. This helps you create subclasses that only implement those interfaces that are meaningful to them.

Big fat interfaces – and abstract base classes for that matter – therefore introduce unnecessary dependencies. Unnecessary dependencies in turn reduce the cohesion, flexibility and maintainability of your classes and increase the coupling between dependencies.

**Demo**

We'll simulate a movie rental where movies can be rented in two formats: DVD and BluRay. The domain expert correctly recognises that these are indeed related products that both can implement the IProduct interface. Open Visual Studio, create a new Console app and add the following interface:



    public interface IProduct
    {
    	decimal Price { get; set; }
    	double Weight { get; set; }
    	int Stock { get; set; }
            int AgeLimit { get; set; }
    	int RunningTime { get; set; }
    }


Here I take the interface approach but it's perfectly reasonable to follow an approach based on an abstract base class instead, such as ProductBase.

The following two classes implement the interface:



    public class DVD : IProduct
    {
    	public decimal Price { get; set; }
    	public double Weight { get; set; }
    	public int Stock { get; set; }
            int AgeLimit { get; set; }
    	public int RunningTime { get; set; }
    }




    public class BluRay : IProduct
    {
    	public decimal Price { get; set; }
    	public double Weight { get; set; }
    	public int Stock { get; set; }
            int AgeLimit { get; set; }
    	public int RunningTime { get; set; }
    }


All is well so far, right? Now management comes along and says that they want to start selling T-Shirts with movie stars printed on them so the new product is more or less related to the main business. The programmer first thinks it's probably just OK to implement the IProduct interface again:



    public class TShirt : IProduct
    {
    	public decimal Price { get; set; }
    	public double Weight { get; set; }
    	public int Stock { get; set; }
            int AgeLimit { get; set; }
    	public int RunningTime { get; set; }
    }


Price, stock and weight are definitely relevant to the TShirt domain. Age limit? Not so sure… Running time? Definitely not. However, it's still there and can be set even if it doesn't make any sense. Also, the programmer may want to extend the TShirt class with the Size property so he adds this property to the IProduct interface. The DVD and BluRay classes will then also need to implement this property, but does it make sense to store the size of a DVD? Well, maybe it does, but it's not the same thing as the size of a TShirt. The TShirt size may be a string, such as "XL" or an enumeration, e.g. Size.XL, but the size of a DVD is more complex. It is more likely to be a dimension with 3 values: width, length and depth. So the type of the Size property will be different. Setting the type to Object is a quick fix but it's a horrendous crime against OOP I think.

The solution is to break up the IProduct interface into two smaller ones:



    public interface IProduct
    {
    	decimal Price { get; set; }
    	double Weight { get; set; }
    	int Stock { get; set; }
    }


The IProduct interface still fits the DVD and BluRay objects and it fits the new TShirt interface as well. The RunningTime property is only relevant to movie-related objects:



    public interface IMovie
    {
    	int RunningTime { get; set; }
    }


BluRay and DVD implement both interfaces:



    public class BluRay : IProduct, IMovie
    {
    	public decimal Price { get; set; }
    	public double Weight { get; set; }
    	public int Stock { get; set; }
    	public int RunningTime { get; set; }
    }




    public class DVD : IProduct, IMovie
    {
    	public decimal Price { get; set; }
    	public double Weight { get; set; }
    	public int Stock { get; set; }
    	public int RunningTime { get; set; }
    }


Now you can have a list of IProduct objects and call the Price property on all of them. You won't have access to the RunningTime property which is a good thing. By default it will be 0 for all TShirt objects so you could probably get away with that in this specific scenario, but it still goes against ISP. As soon as you're dealing with IMovie objects then the RunningTime property makes perfect sense.

A slightly different solution after breaking out the IMovie interface is that IMovie implements the IProduct interface:



    public interface IMovie : IProduct
    {
    	int RunningTime { get; set; }
    }


We can change the implementing objects as follows:



    public class TShirt : IProduct
    {
    	public decimal Price { get; set; }
    	public double Weight { get; set; }
    	public int Stock { get; set; }
    }




    public class BluRay : IMovie
    {
    	public decimal Price { get; set; }
    	public double Weight { get; set; }
    	public int Stock { get; set; }
    	public int RunningTime { get; set; }
    }




    public class DVD : IMovie
    {
    	public double Weight { get; set; }
    	public int Stock { get; set; }
    	public int RunningTime { get; set; }
    	public decimal Price { get; set; }
    }


So the DVD and BluRay classes are still of IProduct types as well because the IMovie interface implements it.

**A real life example**

If you've worked with ASP.NET then you must have come across the MembershipProvider object in the System.Web.Security namespace. It is used within .NET for built-in membership providers such as SqlMembershipProvider which provides functions to create new users, authenticate your users, lock them out, update their passwords and much more. If the built-in membership types are not suitable for your purposes then you can create your own custom membership mechanism by implementing the MembershipProvider abstraction:



    public class CustomMembershipProvider : MembershipProvider


I won't show the implemented version because it's too long. MembershipProvider forces you to implement a whole range of methods, e.g.:



    public override bool ChangePassword(string username, string oldPassword, string newPassword)
    public override bool DeleteUser(string username, bool deleteAllRelatedData)
    public override bool EnablePasswordRetrieval
    public override int MinRequiredPasswordLength


At the time of writing this post MembershipProvider includes 27 methods and properties for you to implement. It has been criticised due to its size and the fact that if you only need half of the methods then the rest will throw the dreaded NotImplementedException. If you're building a custom Login control then all you might be interested in is the following method:



    public override bool ValidateUser(string username, string password)


At best you can provide some [null objects][2] to quietly ignore the unnecessary methods but it may be confusing to the callers that expect a reasonable result from a method.

An alternative solution in cases where you don't own the code is to follow the [Adapter][3] pattern to hide the unnecessary parts in the original abstraction.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/19/solid-design-principles-in-net-the-liskov-substitution-principle/ "SOLID design principles in .NET: the Liskov Substitution Principle"
[2]: http://dotnetcodr.com/2013/05/06/design-patterns-and-practices-in-net-the-null-object-pattern/ "Design patterns and practices in .NET: the Null Object pattern"
[3]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
