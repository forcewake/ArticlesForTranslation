[Source](http://dotnetcodr.com/2013/07/25/design-patterns-and-practices-in-net-the-visitor-pattern/ "Permalink to Design patterns and practices in .NET: the Visitor pattern")

# Design patterns and practices in .NET: the Visitor pattern

**Introduction**

According to the formal definition a Visitor represents an operation to be performed on the elements of an object structure. A visitor lets you define a new operation without changing the classes of the elements on which it operates.

Let's dissect this a little. We start off by saying that we have some operation that we want to carry out. We will carry it out on an object structure, i.e. on a tree, an array or a collection, or some other structure and we want to perform the operation on all of them. A visitor will let us define a new operation without changing the classes of the elements of the structure. We should be able to create a new method on each one of those classes that exist in that structure but at the same time we cannot change those classes. In short we want to add some new functionality to each member of the object structure without changing the classes of that structure. Sounds impossible? Let's see.

**Demo**

Open Visual Studio and create a new console application. We want to simulate a calculator that calculates the net worth of a person's assets. We have a simple domain with 4 objects. Add the following classes to the application:



    public class Loan
    {
        public int Owed { get; set; }
        public int MonthlyPayment { get; set; }
    }


Where Owed is the remaining debt and MonthlyPayment is what a person pays to the bank per month.



    public class BankAccount
        {
            public int Amount { get; set; }
            public double MonthlyInterest { get; set; }
        }


The BankAccount class should be self-explanatory and so should RealEstate:



    public class RealEstate
        {
            public int EstimatedValue { get; set; }
            public int MonthlyRent { get; set; }
        }


Finally we have a Person class which represents the object whose net worth we'd like to calculate. A person simply has a list of the above objects:



    public class Person
    {
        private List<RealEstate> _realEstates = new List<RealEstate>();
        private List<BankAccount> _bankAccounts = new List<BankAccount>();
        private List<Loan> _loans = new List<Loan>();

        public List<RealEstate> RealEstates
         {
    	get
    	{
    		return _realEstates;
    	}
         }

    	public List<BankAccount> BankAccounts
    	{
    		get
    		{
    			return _bankAccounts;
    		}
    	}

    	public List<Loan> Loans
    	{
    		get
    		{
    			return _loans;
    		}
    	}
    }


We can calculate the net worth of a Person in Main as follows:



    static void Main(string[] args)
            {
                Person person = new Person();
                person.BankAccounts.Add(new BankAccount { Amount = 2000, MonthlyInterest = 0.02 });
                person.BankAccounts.Add(new BankAccount { Amount = 3000, MonthlyInterest = 0.03 });
                person.RealEstates.Add(new RealEstate { EstimatedValue = 85000, MonthlyRent = 600 });
                person.Loans.Add(new Loan { Owed= 45000, MonthlyPayment = 50 });

                int netWorth = 0;
                foreach (BankAccount bankAccount in person.BankAccounts)
                    netWorth += bankAccount.Amount;

                foreach (RealEstate realEstate in person.RealEstates)
                    netWorth += realEstate.EstimatedValue;

                foreach (Loan loan in person.Loans)
                    netWorth -= loan.Owed;

                Console.WriteLine(netWorth);
    			Console.ReadKey() ;
            }


Run the application and you should see that the net worth is calculated.

The goal now is to refactor this code according to the Visitor pattern, so we need a Visitor that calculates the net worth. In the above case it is the caller that calculates the net worth and it needs to know about the internals of each and every object in the calculation process. Also, we want to avoid that our domain objects take care of this logic as we'll then introduce unnecessary coupling between them. We want to keep them in isolation as much as possible to avoid this type of tight coupling. Finally, there's not only the net worth that can be calculated using these objects but a range of other things related to finance. We don't want to come back to these classes and add new methods that introduce yet more coupling. We want to let a visitor take care of those too eventually. Remember the pattern definition: we want to separate the operation from the structure so that when we introduce a new operation we can leave the class untouched.

The first step is to introduce a visitor interface:



    public interface IVisitor
    {
    	void Visit(RealEstate realEstate);
    	void Visit(BankAccount bankAccount);
    	void Visit(Loan loan);
    }


As you see we have a method for each type: Loan, BancAccount, RealEstate. They all have different properties, they behave differently so we want to handle them differently as well. If we create a new asset type then we create a new Visit method for that type but we don't have to change the asset type itself.

The next step is another interface for each of the visitable types to implement so that we can connect them to the visitor:



    public interface IVisitable
    {
    	void Accept(IVisitor visitor);
    }


The idea is that each visitable type will accept a visitor and the visitor will act upon the assets that accept it. Each asset type will be an IVisitable so we want to generalise the domain model. Let's look at Loan first:



    public class Loan : IVisitable
    {
            public int Owed { get; set; }
            public int MonthlyPayment { get; set; }
    	public void Accept(IVisitor visitor)
    	{
    		visitor.Visit(this);
    	}
    }


So Loan simply takes the visitor that's passed in and calls its Visit method. Loan passes itself as the method argument. The visitor will then do whatever it's designed to do with the Loan object. The other objects will also implement IVisitable:



    public class BankAccount : IVisitable
    {
           public int Amount { get; set; }
           public double MonthlyInterest { get; set; }

    	public void Accept(IVisitor visitor)
    	{
    		visitor.Visit(this);
    	}
    }




    public class RealEstate : IVisitable
    {
            public int EstimatedValue { get; set; }
            public int MonthlyRent { get; set; }

    	public void Accept(IVisitor visitor)
    	{
    		visitor.Visit(this);
    	}
    }


We can now generalise the list of assets for a person. Also we'll make it implement IVisitable so that it too can be visited:



    public class Person : IVisitable
    {
    	private List<IVisitable> _assets = new List<IVisitable>();

    	public List<IVisitable> Assets
    	{
    		get
    		{
    			return _assets;
    		}
    	}

    	public void Accept(IVisitor visitor)
    	{
    		foreach (IVisitable asset in Assets)
    		{
    			asset.Accept(visitor);
    		}
    	}
    }


We want each asset of the person to be visited so we loop through the asset list and call its accept method.

At this point we have a structure ready for a Visitor. Any concrete visitor we create can be used in conjunction with our domain structure. We want to calculate the net worth of the person's assets so we'll create a NetWorthVisitor:



    public class NetWorthVisitor : IVisitor
    {
    	public int Total { get; set; }

    	public void Visit(RealEstate realEstate)
    	{
    		Total += realEstate.EstimatedValue;
    	}

    	public void Visit(BankAccount bankAccount)
    	{
    		Total += bankAccount.Amount;
    	}

    	public void Visit(Loan loan)
    	{
    		Total -= loan.Owed;
    	}
    }


So we simply modify the Total according to the properties in each asset type: add the value of the real estate, add the amount of money on the bank account and subtract the sum the person owes the bank. As Person also implements IVisitable we make sure that the Visitor will visit each of his or her assets to calculate the net worth.

The modified Main method looks as follows:



    class Program
    {
            static void Main(string[] args)
            {
                Person person = new Person();
                person.Assets.Add(new BankAccount { Amount = 2000, MonthlyInterest = 0.02 });
                person.Assets.Add(new BankAccount { Amount = 3000, MonthlyInterest = 0.03 });
                person.Assets.Add(new RealEstate { EstimatedValue = 85000, MonthlyRent = 600 });
                person.Assets.Add(new Loan { Owed= 45000, MonthlyPayment = 50 });

            	NetWorthVisitor netWorthVisitor = new NetWorthVisitor();
    		person.Accept(netWorthVisitor);

                Console.WriteLine(netWorthVisitor.Total);
    		Console.ReadKey() ;
            }
    }


We fill up the asset list of the person. We got rid of the net worth calculation in the Main method. Instead we introduce a visitor and make the Person accept that visitor. That will cause the visitor to visit each asset in that person's asset list. Finally we can just call the visitor's Total property to get hold of the result.

Run the application and you'll see the same result as before. We haven't changed the logic of each asset type – we only let them accept a visitor in the Accept method, which is not really logic. We've successfully factored out logic that involves multiple different but related objects. We can now introduce other types of visitors that carry out some other logic on these objects. In fact, let's just do that: add a new class called IncomeVisitor. Have it implement the IVisitor interface:



    public class IncomeVisitor : IVisitor
    {
    	public double Amount;

    	public void Visit(RealEstate realEstate)
    	{
    		Amount += realEstate.MonthlyRent;
    	}

    	public void Visit(BankAccount bankAccount)
    	{
    		Amount += bankAccount.Amount * bankAccount.MonthlyInterest;
    	}

    	public void Visit(Loan loan)
    	{
    		Amount -= loan.MonthlyPayment;
    	}
    }


We visit each type of object and modify the Total accordingly. Again, we add new functionality to our domain without modifying its implementation: we don't need to bloat our classes, include new function etc. Let's see how we can use this new visitor:



    class Program
        {
            static void Main(string[] args)
            {
                Person person = new Person();
                person.Assets.Add(new BankAccount { Amount = 2000, MonthlyInterest = 0.02 });
                person.Assets.Add(new BankAccount { Amount = 3000, MonthlyInterest = 0.03 });
                person.Assets.Add(new RealEstate { EstimatedValue = 85000, MonthlyRent = 600 });
                person.Assets.Add(new Loan { Owed= 45000, MonthlyPayment = 50 });

       	NetWorthVisitor netWorthVisitor = new NetWorthVisitor();
    	IncomeVisitor incomeVisitor = new IncomeVisitor();
    	person.Accept(netWorthVisitor);
    	person.Accept(incomeVisitor);

                Console.WriteLine(netWorthVisitor.Total);
    	Console.WriteLine(incomeVisitor.Amount);
    	Console.ReadKey() ;
            }
    }


Run the programme to see the calculation results.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
