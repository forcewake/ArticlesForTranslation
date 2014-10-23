[Source](http://dotnetcodr.com/2013/05/20/design-patterns-and-practices-in-net-the-template-method-design-pattern/ "Permalink to Design patterns and practices in .NET: the Template Method design pattern")

# Design patterns and practices in .NET: the Template Method design pattern

**Introduction**

The Template Method pattern is best used when you have an algorithm consisting of certain steps and you want to allow for different implementations of these steps. The implementation details of each step can vary but the structure and order of the steps are enforced.

A good example is games:

1. Set up the game
2. Take turns
3. Game is over
4. Display the winner

A large number of games can implement this algorithm, such as Monopoly, Chess, card games etc. Each game is set up and played in a different way but they follow the same order.

The Template pattern is very much based around inheritance. The algorithm represents an abstraction and the concrete game types are the implementations, i.e. the subclasses of that abstraction. It is of course plausible that some steps in the algorithm will be implemented in the abstraction while the others will be overridden in the implementing classes.

Note that a prerequisite for this pattern to be applied properly is the rigidness of the algorithm steps. The steps must be known and well defined. The pattern relies on inheritance, rather than composition, and merging two child algorithms into one can prove difficult. If you find that the Template pattern is too limiting in your application then consider the [Strategy][1] or the [Decorator][2] patterns.

This pattern helps to implement the so-called Hollywood principle: Don't call us, we'll call you. It means that high level components, i.e. the superclasses, should not depend on low-level ones, i.e. the implementing subclasses. A base class with a template method is a high level component and clients should depend on this class. The base class will include one or more template method that the subclasses implement, i.e. it is the base class calling the implementation and not vice versa. In other words, the Hollywood principle is applied from the point of view of the base classes: dear implementing classes, don't call us, we'll call you.

**Demo**

Open Visual Studio and create a new blank solution. We'll simulate a simple dispatch service where shipping an item must go through specific steps regardless of which specific service completes the shipment. Insert a new Console application to the solution.

We'll start with the most important component of the pattern: the base class that must be respected by each implementation. Add a class called OrderShipment:



    public abstract class OrderShipment
        {
            public string ShippingAddress { get; set; }
            public string Label { get; set; }
            public void Ship(TextWriter writer)
            {
                VerifyShippingData();
                GetShippingLabelFromCarrier();
                PrintLabel(writer);
            }

            public virtual void VerifyShippingData()
            {
                if (String.IsNullOrEmpty(ShippingAddress))
                {
                    throw new ApplicationException("Invalid address.");
                }
            }
            public abstract void GetShippingLabelFromCarrier();
            public virtual void PrintLabel(TextWriter writer)
            {
                writer.Write(Label);
            }
        }


The template method that implements the order of the steps is Ship. It calls three methods in a specific order. Two of them – VerifyShippingData and PrintLabel are virtual and have a default implementation. They can of course be overridden. The third method, i.e. GetShippingLabelFromCarrier is the abstract method that the base class cannot implement. The superclass has no way of knowing what a service-specific shipping label looks like – it is delegated to the implementations. We'll simulate two services, UPS and FedEx:



    public class FedExOrderShipment : OrderShipment
    	{
    		public override void GetShippingLabelFromCarrier()
    		{
    			// Call FedEx Web Service
    			Label = String.Format("FedEx:[{0}]", ShippingAddress);
    		}
    	}




    public class UpsOrderShipment : OrderShipment
    	{
    		public override void GetShippingLabelFromCarrier()
    		{
    			// Call UPS Web Service
    			Label = String.Format("UPS:[{0}]", ShippingAddress);
    		}
    	}


The implementations should be quite straighforward: they create service-specific shipping labels and set those values to the Label property. There's of course nothing stopping the concrete classes from overriding any other step in the algorithm. Adding new shipping services is very easy: just create a new implementation. Let's see how a client would communicate with the services:



    static void Main(string[] args)
    {
    	OrderShipment service = new UpsOrderShipment();
    	service.ShippingAddress = "New York";
    	service.Ship(Console.Out);

    	OrderShipment serviceTwo = new FedExOrderShipment();
    	service.ShippingAddress = "Los Angeles";
    	service.Ship(Console.Out);

            Console.ReadKey();
    }


Run the programme and you'll see the service-specific labels in the Console. The client calls the Template method Ship which ensures that the steps in the shipping algorithm are carried out in a certain order.

It is of course not optimal to create the specific OrderShipment classes like that, i.e. directly with the new keyword as it introduces tight coupling. Consider using a [factory][3] for building the correct implementation. However, this solution is satisfactory for demo purposes.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/04/29/design-patterns-and-practices-in-net-the-strategy-pattern/ "Design patterns and practices in .NET: the Strategy Pattern"
[2]: http://dotnetcodr.com/2013/05/13/design-patterns-and-practices-in-net-the-decorator-design-pattern/ "Design patterns and practices in .NET: the Decorator design pattern"
[3]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
