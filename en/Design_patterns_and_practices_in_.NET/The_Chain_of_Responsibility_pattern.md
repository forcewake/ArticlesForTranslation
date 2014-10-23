[Source](http://dotnetcodr.com/2013/05/23/design-patterns-and-practices-in-net-the-chain-of-responsibility-pattern/ "Permalink to Design patterns and practices in .NET: the Chain of Responsibility pattern")

# Design patterns and practices in .NET: the Chain of Responsibility pattern

**Introduction**

The Chain of Responsibility is an ordered chain of message handlers that can process a specific type of message or pass the message to the next handler in the chain. This pattern revolves around messaging between a sender and one more receivers. This probably sounds very cryptic – just like the basic description of design patterns in general – so let's see a quick example in words.

Suppose we have a Sender that knows the first receiver in the messaging chain, call it Receiver A. Receiver A is in turn aware of Receiver B and so on. When a message comes in to a sender we can only perform one thing: pass it along to the first receiver. Receiver A inspects the message and decides whether it can process it or not. If not then it passes the message along to Receiver B and Receiver B will perform the same message inspection as Receiver A. It then decides to either process the Message or send it further down the messaging chain. If it processes the message then it will send a Response back to the Sender.

In this example the Sender sent a message to the first Receiver and received a Response from a different one. However, the Sender is not aware of Receiver B. If the messaging stops at Receiver B then Receiver C remained completely inactive: it has no knowledge of the Message and that the messaging actually occurred.

The example also showcases the traits of the pattern:

* The Sender is only aware of the first receiver
* Each receiver only knows of the next receiver down the messaging chain
* Receivers can process the Message or send it down the chain
* The Sender will have no knowledge about which Receiver received the message
* The first receiver that was able to process the message terminates the chain
* The order of the receiver list matters

**Demo**

In the demo we'll simulate the hierarchy of a company: an employee would like to make a large expenditure so he asks his manager. The manager is not entitled to approve the large sum and sends the request forward to the VP. The VP is not entitled either to approve the request so sends it to the President. The President is the highest authority in the hierarchy who will either approve or disapprove the request and sends the response back to the original employee.

Open Visual Studio and create a blank solution. Insert a class library called Domain. You can remove Class1.cs. We'll build up the components one by one. We'll start with the abstraction for an expense report, IExpenseReport:



    public interface IExpenseReport
        {
            Decimal Total { get; }
        }


An expense report can thus have a total sum.

The IExpenseApprover interface represents any object that is entitled to approve expense reports:



    public interface IExpenseApprover
    	{
    		ApprovalResponse ApproveExpense(IExpenseReport expenseReport);
    	}


…where ApprovalResponse is an enumeration:



    public enum ApprovalResponse
    	{
    		Denied,
    		Approved,
    		BeyondApprovalLimit,
    	}


The concrete implementation of the IExpenseReport is very straightforward:



    public class ExpenseReport : IExpenseReport
    	{
    		public ExpenseReport(Decimal total)
    		{
    			Total = total;
    		}

    		public decimal Total
    		{
    			get;
    			private set;
    		}
    	}


The Employee class implements the IExpenseApprover interface:



    public class Employee : IExpenseApprover
    	{
    		public Employee(string name, Decimal approvalLimit)
    		{
    			Name = name;
    			_approvalLimit = approvalLimit;
    		}

    		public string Name { get; private set; }

    		public ApprovalResponse ApproveExpense(IExpenseReport expenseReport)
    		{
    			return expenseReport.Total > _approvalLimit
    					? ApprovalResponse.BeyondApprovalLimit
    					: ApprovalResponse.Approved;
    		}

    		private readonly Decimal _approvalLimit;
    	}


As you see the constructor needs a name and an approval limit. The implemented ApproveExpense method simply checks if the total value of the expense report is above or below the approval limit. If total is lower than the limit, then the expense is approved, otherwise the method indicates that the total is too much for the employee to approve.

Add a Console application called Approval to the solution and add a reference to the Domain library. We'll first check how the approval process may look like without the pattern applied:



    static void Main(string[] args)
    		{
    			List<Employee> managers = new List<Employee>
                                              {
                                                  new Employee("William Worker", Decimal.Zero),
                                                  new Employee("Mary Manager", new Decimal(1000)),
                                                  new Employee("Victor Vicepres", new Decimal(5000)),
                                                  new Employee("Paula President", new Decimal(20000)),
                                              };

    			Decimal expenseReportAmount;
    			while (ConsoleInput.TryReadDecimal("Expense report amount:", out expenseReportAmount))
    			{
    				IExpenseReport expense = new ExpenseReport(expenseReportAmount);

    				bool expenseProcessed = false;

    				foreach (Employee approver in managers)
    				{
    					ApprovalResponse response = approver.ApproveExpense(expense);

    					if (response != ApprovalResponse.BeyondApprovalLimit)
    					{
    						Console.WriteLine("The request was {0}.", response);
    						expenseProcessed = true;
    						break;
    					}
    				}

    				if (!expenseProcessed)
    				{
    					Console.WriteLine("No one was able to approve your expense.");
    				}
    			}
    		}


…where ConsoleInput is a helper class that looks as follows:



    public static class ConsoleInput
    	{
    		public static bool TryReadDecimal(string prompt, out Decimal value)
    		{
    			value = default(Decimal);

    			while (true)
    			{
    				Console.WriteLine(prompt);
    				string input = Console.ReadLine();

    				if (string.IsNullOrEmpty(input))
    				{
    					return false;
    				}

    				try
    				{
    					value = Convert.ToDecimal(input);
    					return true;
    				}
    				catch (FormatException)
    				{
    					Console.WriteLine("The input is not a valid decimal.");
    				}
    				catch (OverflowException)
    				{
    					Console.WriteLine("The input is not a valid decimal.");
    				}
    			}
    		}
    	}


What can we say about the Main method? We first set up our employees with the approval limits in increasing order. The next step is to read an expense report from the command line. Using that sum we construct an expense report which is given to every employee in the list. Each employee is asked to approve the limit and we check the outcome. If the expense is approved then we break the foreach loop.

Build and run the application. Enter 5000 in the console and you'll see that the expense was approved. You'll recall that the VP had an approval limit of 5000 so it was that employee in the chain to approve. Enter 50000 and you'll see that nobody was able to approve the expense because it exceeds the limit of every one of them.

What is wrong with this implementation? After all we iterate through the employee list to see if anyone is able to approve the expense. We get our response and we get to know the outcome.

The problem is that the caller is responsible for iterating through the list. This means that the logic of handling expense reports is encapsulated at the wrong level. Imagine that you as an employee should not ask each one of the managers above you for a yes or no answer. You should only have to turn to your boss who in turn will ask his or her boss etc. Our code should reflect this.

In order to achieve that we need to insert a new interface:



    public interface IExpenseHandler
    	{
    		ApprovalResponse Approve(IExpenseReport expenseReport);
    		void RegisterNext(IExpenseHandler next);
    	}


The Approve method should look familiar from the previous descriptions. The RegisterNext method registers the next approver in the chain. It means that if I cannot approve the expense then I should go and ask the next person in line.

This interface represents a single link in the chain of responsibility.

The IExpenseHandler interface is implemented by the ExpenseHandler class:



    public class ExpenseHandler : IExpenseHandler
    	{
    		private readonly IExpenseApprover _approver;
    		private IExpenseHandler _next;

    		public ExpenseHandler(IExpenseApprover expenseApprover)
    		{
    			_approver = expenseApprover;
    			_next = EndOfChainExpenseHandler.Instance;
    		}

    		public ApprovalResponse Approve(IExpenseReport expenseReport)
    		{
    			ApprovalResponse response = _approver.ApproveExpense(expenseReport);

    			if (response == ApprovalResponse.BeyondApprovalLimit)
    			{
    				return _next.Approve(expenseReport);
    			}

    			return response;
    		}

    		public void RegisterNext(IExpenseHandler next)
    		{
    			_next = next;
    		}
    	}


This class will need an IExpenseApprover in its constructor. This approver is an Employee just like before. The constructor makes sure that there is always a special end of chain Employee in the approval chain through the EndOfChainExpenseHandler class. The Approve method receives an expense report. We ask the approver if they are able to approver the expense. If not, then we go to the next person in the hierarchy, i.e. to the "next" variable.

The implementation of the EndOfChainExpenseHandler class follows below. It also implements the IExpenseHandler method and it represents – as the name implies – the last member in the approval hierarchy. Its Instance property returns this special member of the chain according to the singleton pattern – more on that [here][1].



    public class EndOfChainExpenseHandler : IExpenseHandler
    	{
    		private EndOfChainExpenseHandler() { }

    		public static EndOfChainExpenseHandler Instance
    		{
    			get { return _instance; }
    		}

    		public ApprovalResponse Approve(IExpenseReport expenseReport)
    		{
    			return ApprovalResponse.Denied;
    		}

    		public void RegisterNext(IExpenseHandler next)
    		{
    			throw new InvalidOperationException("End of chain handler must be the end of the chain!");
    		}

    		private static readonly EndOfChainExpenseHandler _instance = new EndOfChainExpenseHandler();
    	}


The purpose of this class is to make sure that if the last person in the hierarchy, i.e. the President, is unable to approve the report then it is not passed on to a null reference – as there's nobody above the President – but that there's an automatic message handler that gives some default answer. Here we follow the [null object pattern][2]. In this case we reject the expense in the Approve method. If we made it this far in the approval chain then the expense must be rejected.

The revised Main method looks as follows:



    static void Main(string[] args)
    {
    	ExpenseHandler william = new ExpenseHandler(new Employee("William Worker", Decimal.Zero));
    	ExpenseHandler mary = new ExpenseHandler(new Employee("Mary Manager", new Decimal(1000)));
    	ExpenseHandler victor = new ExpenseHandler(new Employee("Victor Vicepres", new Decimal(5000)));
    	ExpenseHandler paula = new ExpenseHandler(new Employee("Paula President", new Decimal(20000)));

    	william.RegisterNext(mary);
    	mary.RegisterNext(victor);
    	victor.RegisterNext(paula);

    	Decimal expenseReportAmount;
    	if (ConsoleInput.TryReadDecimal("Expense report amount:", out expenseReportAmount))
    	{
    		IExpenseReport expense = new ExpenseReport(expenseReportAmount);
    		ApprovalResponse response = william.Approve(expense);
    		Console.WriteLine("The request was {0}.", response);
    	}
            Console.ReadKey();
    }


You'll see that we have not registered anyone for the President. This is where it becomes important that we set up a default end of chain approver in the ExpenseHandler constructor.

This is a significantly smaller amount of code than before. We start off by wrapping our employees into expense handlers, so each employee becomes an expense handler. Instead of putting them in a list we register the next employee in the hierarchy for each of them. Then as before we read the user's input from the console, create an expense report and then we go to the first approver in the chain – william. The response from william will be abstracted away in the management chain we set up through the RegisterNext method.

Build and run the application. Enter 1000 and you'll see that it is approved. Enter 30000 and you'll see that it is rejected and the caller is oblivious of who and why rejected the request.

So this is the Chain of Responsibility pattern for you!

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/05/09/design-patterns-and-practices-in-net-the-singleton-pattern/ "Design patterns and practices in .NET: the Singleton pattern"
[2]: http://dotnetcodr.com/2013/05/06/design-patterns-and-practices-in-net-the-null-object-pattern/ "Design patterns and practices in .NET: the Null Object pattern"
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
