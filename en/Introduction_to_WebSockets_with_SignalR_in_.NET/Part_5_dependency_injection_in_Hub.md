[Source](http://dotnetcodr.com/2014/05/29/introduction-to-websockets-with-signalr-in-net-part-5-dependency-injection-in-hub/ "Permalink to Introduction to WebSockets with SignalR in .NET Part 5: dependency injection in Hub")

# Introduction to WebSockets with SignalR in .NET Part 5: dependency injection in Hub

**Introduction**

We've now gone through the basics of SignalR. We've looked at a messaging project and a primitive, but working stock ticker. Actually most of the work is put on the client in form of JavaScript code. The WebSockets specific server side code is not too complex and follows standard C# coding syntax.

However, the relative server side simplicity doesn't mean that our code shouldn't follow good software engineering principles. I mean principles such as [SOLID][1], with special subjective weight given to my favourite, the letter '[D][2]'.

You'll recall from the [previous post][3] that the stock prices were retrieved from a concrete StockService class, an instance of which was created within the body of StartStockMonitoring(). The goal now is to clean up that code so that the stock prices are retrieved from an interface instead of a concrete class. The interface should then be a parameter of the ResultsHub constructor.

We'll use [StructureMap][4] to inject the dependency into ResultsHub.cs.

**Demo**

I'll use two sources from StackOverflow to solve the problem:

Let's start by creating the abstraction. Add the following interface to the Stocks folder:



    public interface IStockService
    {
    	dynamic GetStockPrices();
    }


Then modify the StockService declaration so that it implements the abstraction:



    public class StockService : IStockService


Change the ResultsHub constructor to accept the interface dependency:



    private readonly IStockService _stockService;

    public ResultsHub(IStockService stockService)
    {
    	if (stockService == null) throw new ArgumentNullException("StockService");
    	_stockService = stockService;
    	StartStockMonitoring();
    }


Make sure you refer to the private field in the Task body of StartStockMonitoring:



    Task stockMonitoringTask = Task.Factory.StartNew(async () =>
    				{
    					while(true)
    					{
    						dynamic stockPriceCollection = _stockService.GetStockPrices();
    						Clients.All.newStockPrices(stockPriceCollection);
    						await Task.Delay(5000);
    					}
    				}, TaskCreationOptions.LongRunning);


Next, create an interface for ResultsHub in the Stocks folder:



    public interface IResultsHub
    {
    	void Hello();
    	void SendMessage(String message);
    }


…and have ResultsHub implement it:



    public class ResultsHub : Hub, IResultsHub


Next import the StructureMap NuGet package:

![StructureMap NuGet package][5]

We'll need a custom HubActivator to find the implementation of IResultsHub through StructureMap. Add the following class to the solution:



    public class HubActivator : IHubActivator
    {
    	private readonly IContainer container;

    	public HubActivator(IContainer container)
    	{
    		this.container = container;
    	}

    	public IHub Create(HubDescriptor descriptor)
    	{
    		return (IHub)container.GetInstance(descriptor.HubType);
    	}
    }


IContainer is located in the StructureMap namespace.

Next insert the following class that will initialise the StructureMap container:



    public static class IoC
    {
    	public static IContainer Initialise()
    	{
    		ObjectFactory.Initialize(x =>
    			{
    				x.Scan(scan =>
    					{
    						scan.AssemblyContainingType<IResultsHub>();
    						scan.WithDefaultConventions();
    					});
    			});
    		return ObjectFactory.Container;
    	}
    }


The last step is to register our hub activator when the application starts. Add the following code to Application_Start() in Global.asax.cs:



    IContainer container = IoC.Initialise();
    GlobalHost.DependencyResolver.Register(typeof(IHubActivator), () => new HubActivator(container));


That's it actually. Set a breakpoint within the ResultsHub constructor and start the application. If all goes well then code execution should stop at the breakpoint. Inspect the incoming stockService parameter. If it's null then something's gone wrong but it shouldn't be. Let the code execution continue and you'll see that it works as before except that we've come a good step closer to loosely coupled code.

Read the finishing post in this series [here][6].

View the list of posts on Messaging [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[2]: http://dotnetcodr.com/2013/08/26/solid-design-principles-in-net-the-dependency-inversion-principle-and-the-dependency-injection-pattern/ "SOLID design principles in .NET: the Dependency Inversion Principle and the Dependency Injection pattern"
[3]: http://dotnetcodr.com/2014/05/26/introduction-to-websockets-with-signalr-in-net-part-4-stock-price-ticker/ "Introduction to WebSockets with SignalR in .NET Part 4: stock price ticker"
[4]: http://docs.structuremap.net/ "StructureMap homepage"
[5]: http://dotnetcodr.files.wordpress.com/2014/04/structuremap-nuget-package.png?w=630
[6]: http://dotnetcodr.com/2014/06/02/introduction-to-websockets-with-signalr-in-net-part-6-the-basics-of-publishing-to-groups/ "Introduction to WebSockets with SignalR in .NET Part 6: the basics of publishing to groups"
[7]: http://dotnetcodr.com/messaging/ "Messaging"
