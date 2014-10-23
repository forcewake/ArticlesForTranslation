[Source](http://dotnetcodr.com/2013/08/19/solid-design-principles-in-net-the-liskov-substitution-principle/ "Permalink to SOLID design principles in .NET: the Liskov Substitution Principle")

# SOLID design principles in .NET: the Liskov Substitution Principle

**Introduction**

After visiting the letters '[S][1]' and '[O][2]' in SOLID it's time to discuss what 'L' has to offer. L stands for the Liskov Substitution Principle (LSP) and states that you should be able to use any derived class in place of a parent class and have it behave in the same manner without modification. It ensures that a derived class does not affect the behaviour of the parent class, i.e. that a derived class must be substitutable for its base class.

The principle is named after [Barbara Liskov][3] who first described the problem in 1988.

More specifically substitutability means that a caller that communicates with an abstraction, i.e. a base class or an interface, should not be aware of and should not be concerned with the different concrete types of those abstractions. The client should be able to call BaseClass.DoSomething() and get a perfectly usable answer regardless of what the concrete class is in place of BaseClass. For this to work the derived class must also "behave well", meaning:

* They must not remove any base class behaviour
* They must not violate base [class invariants][4], i.e. the rules and constraints of a class, in order to preserve its integrity

The first point means the following: if a base class defines two abstract methods then a derived class must give meaningful implementations of both. If a derived class implements a method with 'throw new NotImplementedException' then it means that the derived class is not fully substitutable for its base class. It is a sign that the base class is 'NOT-REALLY-A' base class type. In that case you'll probably need to reconsider your class hierarchy.

All who study OOP must at some point come across the 'IS-A' relationship between a base class and a derived class: a Dog is an Animal, a Clerk is an Employee which is a Person, a Car is a vehicle etc. LSP refines this relationship with 'IS-SUBSTITUTABLE-FOR', meaning that an object is substitutable with another object in all situations without running into exceptions and unexpected behaviour.

**Demo**

As usual in this series on SOLID we'll start with some code which violates LSP. We'll then see why it's bad and then correct it. The demo is loosely connected to the one we worked on in the SRP and OCP posts: an e-commerce application that can refund your money in case you send back the item(s) you purchased. At this company you can pay using different services such as PayPal. Consequently the refund will happen through the same service as well.

Open Visual Studio and create a new console application. We'll start off with an enumeration of the payment services:



    public enum PaymentServiceType
    {
    	PayPal = 1
    	, WorldPay = 2
    }


It would be great to explore the true web services these companies have to offer to the public but the following mockup APIs will suffice:



    public class PayPalWebService
    	{
    		public string GetTransactionToken(string username, string password)
    		{
    			return "Hello from PayPal";
    		}

    		public string MakeRefund(decimal amount, string transactionId, string token)
    		{
    			return "Auth";
    		}
    	}




    public class WorldPayWebService
    	{
    		public string MakeRefund(decimal amount, string transactionId, string username,
    			string password, string productId)
    		{
    			return "Success";
    		}
    	}


We concentrate on the Refund logic which the two services carry out slightly differently. What's common is that the MakeRefund methods return a string that describes the result of the action.

We'll eventually need a refund service that interacts with these API's somehow but it will need some object that represents the payments. As the payments can go through the two services mentioned above, and possible others in the future, we'll need an abstraction for them. An abstract base class seems appropriate:



    public abstract class PaymentBase
    	{
    		public abstract string Refund(decimal amount, string transactionId);
    	}


We can now create the concrete classes for the PayPal and WorldPay payments:



    public class PayPalPayment : PaymentBase
    	{
    		public string AccountName { get; set; }
    		public string Password { get; set; }

    		public override string Refund(decimal amount, string transactionId)
    		{
    			PayPalWebService payPalWebService = new PayPalWebService();
    			string token = payPalWebService.GetTransactionToken(AccountName, Password);
    			string response = payPalWebService.MakeRefund(amount, transactionId, token);
    			return response;
    		}
    	}




    public class WorldPayPayment : PaymentBase
    	{
    		public string AccountName { get; set; }
    		public string Password { get; set; }
    		public string ProductId { get; set; }

    		public override string Refund(decimal amount, string transactionId)
    		{
    			WorldPayWebService worldPayWebService = new WorldPayWebService();
    			string response = worldPayWebService.MakeRefund(amount, transactionId, AccountName, Password, ProductId);
    			return response;
    		}
    	}


Each concrete Payment class will communicate with the appropriate payment service to log on and request a refund. This follows the [Adapter][5] pattern in that we're wrapping the real API:s in our own classes. We'll need to be able to identify the correct payment type. In the previous post we used a variable called IsMatch in each concrete type – here we'll take the [Factory][6] approach just to see another way of selecting a concrete class:



    public class PaymentFactory
    	{
    		public static PaymentBase GetPaymentService(PaymentServiceType serviceType)
    		{
    			switch (serviceType)
    			{
    				case PaymentServiceType.PayPal:
    					return new PayPalPayment();
    				case PaymentServiceType.WorldPay:
    					return new WorldPayPayment();
    				default:
    					throw new NotImplementedException("No such service.");
    			}
    		}
    	}


The factory selects the correct implementation using the incoming enumeration. Read the blog post on the Factory pattern if you're not sure what's happening here.

We're ready for the actual refund service which connects the above ingredients:



    public class RefundService
    {
    	public bool Refund(PaymentServiceType paymentServiceType, decimal amount, string transactionId)
    	{
    		bool refundSuccess = false;
    		PaymentBase payment = PaymentFactory.GetPaymentService(paymentServiceType);
    		if ((payment as PayPalPayment) != null)
    		{
    			((PayPalPayment)payment).AccountName = "Andras";
    			((PayPalPayment)payment).Password = "Passw0rd";
    		}
    		else if ((payment as WorldPayPayment) != null)
    		{
    			((WorldPayPayment)payment).AccountName = "Andras";
    			((WorldPayPayment)payment).Password = "Passw0rd";
    			((WorldPayPayment)payment).ProductId = "ABC";
    		}

    		string serviceResponse = payment.Refund(amount, transactionId);

    		if (serviceResponse.Contains("Auth") || serviceResponse.Contains("Success"))
    		{
    			refundSuccess = true;
    		}

    		return refundSuccess;
    	}
    }


We get the payment type using the factory. We then immediately need to check its type in order to be able to assign values to the the different properties in it. There are multiple problems with the current implementation:

* We cannot simply take the payment object returned by the factory, we need to check its type – therefore we cannot substitute the subtype for its base type, hence we break LSP. Such if-else statements where you branch your logic based on some object's type are telling signs of LSP violation
* We need to extend the if-else statements as soon as a new provider is implemented, which also violates the Open-Closed Principle
* We need to extend the serviceResponse.Contains bit as well if a new payment provider returns a different response, such as "OK"
* The client, i.e. the RefundService object needs to intimately know about the different types of payment providers and their internal setup which greatly increases coupling
* The client needs to know how to interpret the string responses from the services and that is not the correct approach – the individual services should be the only ones that can do that

The goal is to be able to take the payment object returned by the factory and call its Refund method without worrying about its exact type.

First of all let's introduce a constructor in each Payment class that force the clients to provide all the necessary parameters:



    public class PayPalPayment : PaymentBase
    	{
    		public PayPalPayment(string accountName, string password)
    		{
    			AccountName = accountName;
    			Password = password;
    		}

    		public string AccountName { get; set; }
    		public string Password { get; set; }

    		public override string Refund(decimal amount, string transactionId)
    		{
    			PayPalWebService payPalWebService = new PayPalWebService();
    			string token = payPalWebService.GetTransactionToken(AccountName, Password);
    			string response = payPalWebService.MakeRefund(amount, transactionId, token);
    			return response;
    		}
    	}




    public class WorldPayPayment : PaymentBase
    	{
    		public WorldPayPayment(string accountId, string password, string productId)
    		{
    			AccountName = accountId;
    			Password = password;
    			ProductId = productId;
    		}

    		public string AccountName { get; set; }
    		public string Password { get; set; }
    		public string ProductId { get; set; }

    		public override string Refund(decimal amount, string transactionId)
    		{
    			WorldPayWebService worldPayWebService = new WorldPayWebService();
    			string response = worldPayWebService.MakeRefund(amount, transactionId, AccountName, Password, ProductId);
    			return response;
    		}
    	}


We need to update the factory accordingly:



    public class PaymentFactory
    	{
    		public static PaymentBase GetPaymentService(PaymentServiceType serviceType)
    		{
    			switch (serviceType)
    			{
    				case PaymentServiceType.PayPal:
    					return new PayPalPayment("Andras", "Passw0rd");
    				case PaymentServiceType.WorldPay:
    					return new WorldPayPayment("Andras", "Passw0rd", "ABC");
    				default:
    					throw new NotImplementedException("No such service.");
    			}
    		}
    	}


The input parameters are hard-coded to keep things simple. In reality these can be read from a configuration file or sent in as parameters to the GetPaymentService method. We can now improve the RefundService class as follows:



    public class RefundService
    {
    	public bool Refund(PaymentServiceType paymentServiceType, decimal amount, string transactionId)
    	{
    		bool refundSuccess = false;
    		PaymentBase payment = PaymentFactory.GetPaymentService(paymentServiceType);

    		string serviceResponse = payment.Refund(amount, transactionId);

    		if (serviceResponse.Contains("Auth") || serviceResponse.Contains("Success"))
    		{
    			refundSuccess = true;
    		}

    		return refundSuccess;
    	}
    }


We got rid of the downcasting issue. We now need to do something about the need to inspect the strings in the Contains method. This if statement still has to be extended if we introduce a new payment service and the client still has to know what "Success" means. If you think about it then ONLY the payment service objects should be concerned with this type of logic. The Refund method returns a string from the payment service but instead the string should be evaluated within the payment service itself, right? Let's update the return type of the PaymentBase object:



    public abstract class PaymentBase
    {
    	public abstract bool Refund(decimal amount, string transactionId);
    }


We can transfer the response interpretation logic to the respective Payment objects:



    public class WorldPayPayment : PaymentBase
    {
    	public WorldPayPayment(string accountId, string password, string productId)
    	{
    		AccountName = accountId;
    		Password = password;
    		ProductId = productId;
    	}

    	public string AccountName { get; set; }
    	public string Password { get; set; }
    	public string ProductId { get; set; }

    	public override bool Refund(decimal amount, string transactionId)
    	{
    		WorldPayWebService worldPayWebService = new WorldPayWebService();
    		string response = worldPayWebService.MakeRefund(amount, transactionId, AccountName, Password, ProductId);
    		if (response.Contains("Auth"))
    			return true;
    		return false;
    	}
    }




    public class PayPalPayment : PaymentBase
    {
    	public PayPalPayment(string accountName, string password)
    	{
    		AccountName = accountName;
    		Password = password;
    	}

    	public string AccountName { get; set; }
    	public string Password { get; set; }

    	public override bool Refund(decimal amount, string transactionId)
    	{
    		PayPalWebService payPalWebService = new PayPalWebService();
    		string token = payPalWebService.GetTransactionToken(AccountName, Password);
    		string response = payPalWebService.MakeRefund(amount, transactionId, token);
    		if (response.Contains("Success"))
    			return true;
    		return false;
    	}
    }


The RefundService has been greatly simplified:



    public class RefundService
    {
    	public bool Refund(PaymentServiceType paymentServiceType, decimal amount, string transactionId)
    	{
    		PaymentBase payment = PaymentFactory.GetPaymentService(paymentServiceType);
    		return payment.Refund(amount, transactionId);
    	}
    }


There's no need to downcast anything or to extend this method if a new service is introduced. Strict proponents of the Single Responsibility Principle may argue that the Payment classes are now bloated, they should not know how to process the string response from the web services. However, I think it's well worth refactoring the initial code this way. It eliminates the drawbacks we started out with. Also, in a Domain Driven Design approach it's perfectly reasonable to include the logic belonging to a single object within that object and not anywhere else.

A related principle is called '**Tell, Don't Ask**'. We violated this principle in the initial solution where we asked the Payment object about its exact type: if you see that you need to interrogate an object about its internal state in order to branch your code then it may be a candidate for refactoring. Move that logic into the object itself within a method and simply call that method. Meaning don't ask an object about its state, instead tell it to perform what you want it do.

View the list of posts on Architecture and Patterns [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[2]: http://dotnetcodr.com/2013/08/15/solid-design-principles-in-net-the-open-closed-principle/ "SOLID design principles in .NET: the Open-Closed Principle"
[3]: http://en.wikipedia.org/wiki/Barbara_Liskov "Wikipedia article on Barbara Liskov"
[4]: http://en.wikipedia.org/wiki/Class_invariants "Wikipedia on class invariants"
[5]: http://dotnetcodr.com/2013/04/25/design-patterns-and-practices-in-net-the-adapter-pattern/ "Design patterns and practices in .NET: the Adapter Pattern"
[6]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[7]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
