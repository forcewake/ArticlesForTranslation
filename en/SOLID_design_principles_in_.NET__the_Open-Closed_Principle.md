[Source](http://dotnetcodr.com/2013/08/15/solid-design-principles-in-net-the-open-closed-principle/ "Permalink to SOLID design principles in .NET: the Open-Closed Principle")

# SOLID design principles in .NET: the Open-Closed Principle

**Introduction**

In the [previous][1] post we talked about the letter 'S' in SOLID, i.e. the Single Responsibility Principle. Now it's time to move to the letter 'O' which stands for the Open-Closed Principle (OCP). OCP states that classes should be open for extension and closed for modification. You should be able to add new features and extend a class without changing its internal behaviour. You can always add new behaviour to a class in the future. At the same time you should not have to recompile your application just to make room for new things. The main goal of the principle is to avoid breaking changes in an existing class as it can introduce bugs and errors in other parts of your application.

How is this even possible? The key to success is identifying the areas in your domain that are likely to change and programming to abstractions. Separate out behaviour into abstractions: interfaces and abstract classes. There's then no limit to the variety of implementations that the dependent class can accept.

**Demo**

In the demo we'll first write some code that calculates prices and does not follow OCP. We'll then refactor that code to a better design. The demo project is very similar to the e-commerce one in the previous post and partially builds upon it so make sure to check it out as well.

Open Visual Studio and create a new console application. Insert a new folder called Model. The following three basic domain objects are the same as in the previous demo:



    public class OrderItem
    {
    	public string Identifier { get; set; }
    	public int Quantity { get; set; }
    }




    public enum PaymentMethod
    {
    	CreditCard
    	, Cheque
    }




    public class PaymentDetails
    {
    	public PaymentMethod PaymentMethod { get; set; }
    	public string CreditCardNumber { get; set; }
    	public DateTime ExpiryDate { get; set; }
    	public string CardholderName { get; set; }
    }


ShoppingCart looks a bit different. It now includes a price calculation function depending on the Identifier property:



    public class ShoppingCart
    {
    	private readonly List<OrderItem> _orderItems;

    	public ShoppingCart()
    	{
    		_orderItems = new List<OrderItem>();
    	}

    	public IEnumerable<OrderItem> OrderItems
    	{
    		get { return _orderItems; }
    	}

    	public string CustomerEmail { get; set; }

    	public void Add(OrderItem orderItem)
    	{
    		_orderItems.Add(orderItem);
    	}

    	public decimal TotalAmount()
    	{
    		decimal total = 0m;
    		foreach (OrderItem orderItem in OrderItems)
    		{
    			if (orderItem.Identifier.StartsWith("Each"))
    			{
    				total += orderItem.Quantity * 4m;
    			}
    			else if (orderItem.Identifier.StartsWith("Weight"))
    			{
    				total += orderItem.Quantity * 3m / 1000; //1 kilogram
    			}
    			else if (orderItem.Identifier.StartsWith("Spec"))
    			{
    				total += orderItem.Quantity * .3m;
    				int setsOfFour = orderItem.Quantity / 4;
    				total -= setsOfFour * .15m; //discount on groups of 4 items
    			}
    		}
    		return total;
    	}
    }


The TotalAmount function counts the total price in the cart. You can imagine that shops use many different strategies to calculate prices:

* Price per unit
* Price per unit of weight, such as price per kilogram
* Special discount prices: buy 3, get 1 for free
* Price depending on the Customer's loyalty: loyal customers get 10% off

And there are many other strategies out there. Some of these are represented in the TotalAmount function by magic strings retrieved from the Identifier of the product. The decimals '5m' etc. are the dollar prices. So here every product has the same price for simplicity.

Such pricing rules are probably changing a lot in a real word business. Meaning that programmer will need to revisit this if-else statement quite often to extend it with new rules and modify the existing ones. That type of code gets quickly out of hand. Imagine 100 else-if statements with possibly nested ifs with more complex rules. If it's Christmas AND you are a loyal customer AND you have a special coupon then the final price may depend on each of these conditions. Debugging and maintaining that code would soon become a nightmare. It would be a lot better if this particular method didn't have to be modified at all. In other words we'd like to apply OCP so that we don't need to extend this particular code every time there's a change in the pricing rules.

Extending the if-else statements can introduce bugs and the application must be re-tested. We'll need to test the ShoppingCart whereas we're only interested in testing the pricing rule(s). Also, the pricing logic is tightly coupled with the ShoppingCart domain. Therefore if we change the pricing logic in the ShoppingCart object we'll need to test all other objects that depend on ShoppingCart even if they absolutely have nothing to do with pricing rules. A more intelligent solution is to separate out the pricing logic to different classes and hide them behind an abstraction that ShoppingCart can refer to. The result is that you'll have a higher number of classes but they are typically small and concentrate on some very specific functionality. This idea refers back to the Single Responsibility Principle of the previous post.

There are other advantages to creating new classes: they can be tested in isolation, there's no other class that's dependent on them – at least to begin with-, and as they are NEW classes in your code they have no legacy coupling to make them hard to design or test.

There are at least two design patterns that can come to the rescue: the [Strategy Pattern][2] and the [Template Pattern][3]. We'll solve our particular problem using the strategy pattern. If you don't know what it is about then make sure to check out the link I've provided, I won't introduce the pattern from scratch here.

Let's first introduce an abstraction for a pricing strategy:



    public interface IPriceStrategy
    {
    	bool IsMatch(OrderItem item);
    	decimal CalculatePrice(OrderItem item);
    }


The purpose of the IsMatch method will be to determine which concrete strategy to pick based on the OrderItem. This could be performed by a [factory][4] as well but it would probably make the solution more complex than necessary.

Let's translate the if-else statements into concrete pricing strategies. We'll start with the price per unit strategy:



    public class PricePerUnitStrategy : IPriceStrategy
    {
    	public bool IsMatch(OrderItem item)
    	{
    		return item.Identifier.StartsWith("Each");
    	}

    	public decimal CalculatePrice(OrderItem item)
    	{
    		return item.Quantity * 4m;
    	}
    }


We still base the strategy selection strategy on the product identifier. This may be good or bad, but that's a separate discussion. The main point is that the strategy selection and price calculation logic is encapsulated within this separate class. We'll do something similar to the other strategies:



    public class PricePerKilogramStrategy : IPriceStrategy
    {
    	public bool IsMatch(OrderItem item)
    	{
    		return item.Identifier.StartsWith("Weight");
    	}

    	public decimal CalculatePrice(OrderItem item)
    	{
    		return item.Quantity * 3m / 1000;
    	}
    }




    public class SpecialPriceStrategy : IPriceStrategy
    {
    	public bool IsMatch(OrderItem item)
    	{
    		return item.Identifier.StartsWith("Spec");
    	}

    	public decimal CalculatePrice(OrderItem item)
    	{
    		decimal total = 0m;
    		total += item.Quantity * .3m;
    		int setsOfFour = item.Quantity / 4;
    		total -= setsOfFour * .15m;
    		return total;
    	}
    }


The next step is to introduce a calculator that will calculate the correct price. We'll hide the calculator behind an interface to follow good programming practices:



    public interface IPriceCalculator
    {
    	decimal CalculatePrice(OrderItem item);
    }


That's quite minimalistic but it will suffice. Often good OOP software will have many small classes and interfaces that concentrate on very specific tasks.

The implementation will select the correct strategy and calculate the price:



    public class DefaultPriceCalculator : IPriceCalculator
    {
    	private readonly List<IPriceStrategy> _pricingRules;

    	public DefaultPriceCalculator()
            {
                _pricingRules = new List<IPriceStrategy>();
                _pricingRules.Add(new PricePerKilogramStrategy());
                _pricingRules.Add(new PricePerUnitStrategy());
                _pricingRules.Add(new SpecialPriceStrategy());
            }

    	public decimal CalculatePrice(OrderItem item)
    	{
    		return _pricingRules.First(r => r.IsMatch(item)).CalculatePrice(item);
    	}
    }


We store the list of possible strategies in the constructor. In the CalculatePrice method we select the suitable pricing strategy based on LINQ and the IsMatch implementations and we call its CalculatePrice method.

Now we're ready to simplify the ShoppingCart object:



    public class ShoppingCart
    {
    	private readonly List<OrderItem> _orderItems;
            private readonly IPriceCalculator _priceCalculator;

            public ShoppingCart(IPriceCalculator priceCalculator)
            {
                _priceCalculator = priceCalculator;
                _orderItems = new List<OrderItem>();
            }

            public IEnumerable<OrderItem> OrderItems
            {
                get { return _orderItems; }
            }

            public string CustomerEmail { get; set; }

            public void Add(OrderItem orderItem)
            {
                _orderItems.Add(orderItem);
            }

            public decimal TotalAmount()
            {
                decimal total = 0m;
                foreach (OrderItem orderItem in OrderItems)
                {
                    total += _priceCalculator.CalculatePrice(orderItem);
                }
                return total;
            }
    }


All the consumer of the ShoppingCart class needs to do is to specify a concrete IPriceCalculator object, such as the DefaultPriceCalculator one and let it calculate the price based on the items in the shopping cart. The ShoppingCart is no longer responsible for the actual price calculation. That has been factored out to abstractions and smaller classes that are easy to test and carry out very specific tasks.

What if the domain owner comes along and tell you that there's a new pricing rule? Now instead of having to go through the if-else statements you can simply create a new pricing strategy:



    public class BuyThreeGetOneFree : IPriceStrategy
    {
    	public bool IsMatch(OrderItem item)
    	{
    		return item.Identifier.StartsWith("Buy3OneFree");
    	}

    	public decimal CalculatePrice(OrderItem item)
    	{
    		decimal total = 0m;
    		total += item.Quantity * 1m;
    		int setsOfThree = item.Quantity / 3;
    		total -= setsOfThree * 1m;
    		return total;
    	}
    }


Add this new concrete class to the DefaultPriceCalculator class constructor and it will be found by the LINQ statement.

Now you may think that you'll need to introduce abstractions everywhere in your code for every little task. That's not entirely correct. If you have a domain whose functionality changes a lot then you can apply OCP right away. Otherwise you may be better off not to introduce abstractions at first because they also make your code somewhat more complex. This may be the case with brand new domains in your application where you just don't have enough experience and the domain expert cannot help you either. In such a case start off with the simplest possible design, even if it involves an if statement with a magic string. It may even be acceptable to later introduce an else statement with another magic string to accommodate a change in the logic. However, as soon as you see that you have to change and/or extend that particular functionality then factor it out to an abstraction. The following motto applies here:

"Fool me once, shame on you;fool me twice, shame on me."

OCP doesn't come for free. Implementing OCP will cost you some hours of refactoring and will add complexity to your design. Also, keep in mind that there's probably no design that guarantees that you won't have to change it at some point. The key is to identify those areas in your domain that are volatile and likely to change over time.

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[2]: http://dotnetcodr.com/2013/04/29/design-patterns-and-practices-in-net-the-strategy-pattern/ "Design patterns and practices in .NET: the Strategy Pattern"
[3]: http://dotnetcodr.com/2013/05/20/design-patterns-and-practices-in-net-the-template-method-design-pattern/ "Design patterns and practices in .NET: the Template Method design pattern"
[4]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
