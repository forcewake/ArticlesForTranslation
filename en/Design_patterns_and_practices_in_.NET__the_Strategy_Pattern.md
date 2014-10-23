[Source](http://dotnetcodr.com/2013/04/29/design-patterns-and-practices-in-net-the-strategy-pattern/ "Permalink to Design patterns and practices in .NET: the Strategy Pattern")

# Design patterns and practices in .NET: the Strategy Pattern

**Introduction**

The strategy pattern is one of the simplest patterns to implement. Have you ever written code with ugly if-else statements where you check some condition and then call another method accordingly? Even worse: have you written if-else statements to check the type of an object and then called another method depending on that? That's not really object-oriented, right? The strategy pattern will help you clean up the mess by turning the if statements into objects – aka strategies – where the objects implement the same interface. Therefore they can be injected into another object that has a dependency of that interface and which will have no knowledge of the actual concrete type.

**Starting point**

Open Visual Studio and create a new blank solution. We'll simulate a simple shipping cost calculation application where the calculation will depend on the type of the carrier: FedEx, UPS, Schenker. Add a Class Library called Domain with the following three classes:

Address.cs:



    public class Address
    	{
    		public string ContactName { get; set; }
    		public string AddressLine1 { get; set; }
    		public string AddressLine2 { get; set; }
    		public string AddressLine3 { get; set; }
    		public string City { get; set; }
    		public string Region { get; set; }
    		public string Country { get; set; }

    		public string PostalCode { get; set; }
    	}


ShippingOptions enumeration



    public enum ShippingOptions
    	{
    		UPS = 100,
    		FedEx = 200,
    		Schenker = 300,
    	}


Order.cs:



    public class Order
    	{
    		public ShippingOptions ShippingMethod { get; set; }
    		public Address Destination { get; set; }
    		public Address Origin { get; set; }
    	}


The domain objects shouldn't be terribly difficult to understand. The shipping cost calculation is definitely a domain logic so we'll include the following domain service in the Domain layer:



    public class ShippingCostCalculatorService
    	{
    		public double CalculateShippingCost(Order order)
    		{
    			switch (order.ShippingMethod)
    			{
    				case ShippingOptions.FedEx:
    					return CalculateForFedEx(order);

    				case ShippingOptions.UPS:
    					return CalculateForUPS(order);

    				case ShippingOptions.Schenker:
    					return CalculateForSchenker(order);

    				default:
    					throw new Exception("Unknown carrier");

    			}

    		}

    		double CalculateForSchenker(Order order)
    		{
    			return 3.00d;
    		}

    		double CalculateForUPS(Order order)
    		{
    			return 4.25d;
    		}

    		double CalculateForFedEx(Order order)
    		{
    			return 5.00d;
    		}
    	}


If you are not familiar with patterns and practices then this implementation might look perfectly normal to you. We check the type of the shipping method in an enumeration and then call on a method to calculate the cost accordingly.

**What's wrong with this code?**

It is perfectly reasonable that we may introduce a new carrier in the future, say PerfectHaulage. If we pass an order with this shipping method to the CalculateShippingCost method then we'll get an exception. We'd have to manually extend the switch statement to account for the new shipment type. In case of a new carrier we'd have to come back to this domain service and modify it accordingly. That breaks the Open/Closed principle of SOLID: a class is open for extensions but closed for modifications. In addition, if there's a change in the implementation of one of the calculation algorithms then again we'd have to come back to this method and modify it. That's generally not a good practice: if you make a change to one of your classes, then you should not have to go an modify other classes and public methods just to accommodate that change.

If a change in one part of the project necessitates changes in another part of the project then chances are that we're facing some serious coupling issues between two or more classes. If you repeatedly have to revisit other parts of your software after changing some class then you probably have a design issue.

Also, the methods that calculate the costs are of course ridiculously simple in this demo – in reality there may well be calls to other services, the weight of the package may be checked etc., so the ShippingCostCalculatorService class may grow very large and difficult to maintain. The calculator class becomes bloated with logic belonging to UPS, FedEx, DHL etc, violating the Single Responsibility Principle. The service class is trying to take care of too much.

There are problems related to the design of the domain. We could argue that it is not the responsibility of the Order domain to know which shipping method a customer prefers. This is up to the domain experts to decide, but this is certainly a possible implementation problem.

However, whether or not this is the correct distribution of responsibility the switch statement may possibly reveal a flaw in our domain design. Isn't this some domain logic that we're hiding in the ShippingCostCalculatorService class? Shouldn't this logic be transformed into a domain on its own? The strategy pattern will help to elevate these hidden concepts to proper objects.

**Solution**

The solution is basically to create a class for each calculation – we can call each implemented calculation a strategy. Each class will need to implement the same interface. If you check the calculation methods in the service class then the following interface will probably fit our needs:



    public interface IShippingStrategy
    	{
    		double Calculate(Order order);
    	}


Next we'll create the implemented strategies:

Schenker:



    public class SchenkerShippingStrategy : IShippingStrategy
    	{
    		public double Calculate(Order order)
    		{
    			return 3.00d;
    		}
    	}


UPS:



    public class UpsShippingStrategy : IShippingStrategy
    	{
    		public double Calculate(Order order)
    		{
    			return 4.25d;
    		}
    	}


FedEx:



    public class FedexShippingStrategy : IShippingStrategy
    	{
    		public double Calculate(Order order)
    		{
    			return 5.00d;
    		}
    	}


The cost calculation service is now ready to accept the strategy from the outside. The new and improved service looks as follows:



    public class CostCalculationService_WithStrategy
    	{
    		private readonly IShippingStrategy _shippingStrategy;

    		public CostCalculationService_WithStrategy(IShippingStrategy shippingStrategy)
    		{
    			_shippingStrategy = shippingStrategy;
    		}

    		public double CalculateShippingCost(Order order)
    		{
    			return _shippingStrategy.Calculate(order);
    		}
    	}


I believe you agree that this class structure is a lot more polished and follows good software engineering principles. You can now implement the IShippingStrategy as new carriers come into the picture and the calculation service can continue to function without knowing anything about the concrete strategy classes. The concrete strategy classes are self-containing, they can be tested individually and they can be mocked – more on mocking [here][1].

**Variation**

This is not the only way to implement the strategy pattern. You can use the delegate approach as well.

The strategies can be written as delegates, for example:



    Func<Order, double> upsStrategy = delegate(Order order) { return 4.00d; };


Then the shipping cost calculator service may look like this:



    public class ShippingCostCalculatorService
        {
            public double CalculateShippingCost(Order order, Func<Order, double> shippingCostStrategy)
            {
               return shippingCostStrategy(order);
            }
        }


So we inject the delegate to the method performing the calculation rather than through a constructor injection as in the original solution.

View the list of posts on Architecture and Patterns [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/04/08/test-driven-development-in-net-part-5-verifying-system-under-test-with-mocks-using-the-moq-framework/ "Test Driven Development in .NET Part 5: verifying system under test with mocks using the Moq framework"
[2]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
