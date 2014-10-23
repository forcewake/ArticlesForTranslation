[Source](http://dotnetcodr.com/2013/08/01/design-patterns-and-practices-in-net-the-observer-pattern/ "Permalink to Design patterns and practices in .NET: the Observer pattern")

# Design patterns and practices in .NET: the Observer pattern

**Introduction**

As its name implies the pattern has to do with the interaction between two or more objects. The objects may or may not be related, but one object is interested in the changes of the other object. In other words there's some kind of dependency between them. Changing one object may require changing one or more other objects. The most interesting case, however, is when changing an object should allow notification to others without any knowledge of them.

The pattern is used extensively in .NET:

* GUI controls: events such as OnClick are handled through event handlers which are waiting for changes in the control
* Data binding of controls: e.g. a GridView control in ASP.NET web forms can be bound to a data source upon which its item templates will be filled with data from the source
* File watchers: you can monitor folders and files for changes. You can wire up the events so that you get notified if somebody has added a file to a certain folder

**Starting point**

Open Visual Studio and start a new Console application. We'll simulate a financial application where people trade commodities, like in the Kansas City Board of Trade and similar exchanges. Create a new object called Commodity:



    public class Commodity
    {
    	public string Name { get; set; }
    	public decimal Price { get; set; }
    }


Insert the following rudimentary repository:



    public class CommodityRepository
    {
    	public IEnumerable<Commodity> GetAllCommodities()
    	{
    		return new List<Commodity>()
    		{
    			new Commodity(){Name = "milk", Price= 1}
    			, new Commodity() {Name = "milk", Price = 1.2m}
    			, new Commodity() {Name = "milk", Price = 1.3m}
    			, new Commodity() {Name = "cocoa", Price = 2.1m}
    			, new Commodity() {Name = "milk", Price = 3.2m}
    			, new Commodity() {Name = "cocoa", Price = 2.9m}
    			, new Commodity() {Name = "milk", Price = 1.8m}
    			, new Commodity() {Name = "cocoa", Price = 1.7m}
    		};
    	}
    }


Insert the following in Main:



    static void Main(string[] args)
    {
    	RunNaiveExample();
    	Console.ReadKey();
    }

    private static void RunNaiveExample()
    {
    	IEnumerable<Commodity> commodities = new CommodityRepository().GetAllCommodities();
    	foreach (Commodity commodity in commodities)
    	{
    		if (commodity.Name == "cocoa")
    		{
    			Console.WriteLine("The current price of cocoa is {0}", commodity.Price);
    		}

    		if (commodity.Name == "milk" && commodity.Price > 2m)
    		{
    			Console.WriteLine("The price of milk has now reached {0}", commodity.Price);
    		}
    	}
    }


The intent is quite simple here, right? We're looping through the list of commodities in the list and if we find something interesting then we print it out in the console. The multiple entries in the Commodities list simulates that we ask some service periodically, like in a ticker. This is probably the simplest version of commodity monitoring. We perform one or more actions based on the filtering in the if statements. Run the app to see the output: milk exceeds the target price of 2 once, and cocoa appears 3 times.

Even in this short application we can see several issues, particularly with the separation of concerns. The loop in Main corresponds to a ticker. However, a ticker doesn't need to know that we're monitoring specific commodities. It doesn't need to know about the price of milk and cocoa to perform its job. All it needs to do is to read the commodities one by one and report on them. Also, we're mixing prices in the loop: the milk price has nothing to do with the cocoa price – at least not from a software design point of view.

These are all independent actions that are mixed together in the same programme. In case we want to monitor a different commodity then we have to extend the foreach loop, i.e. we have to modify the main application.

The observer pattern allows us to separate out those filters, which are called observers, and the actions that they're taking.

**Events and delegates**

.NET supports events and delegates which are excellent candidates for implementing the observer pattern. The event is created on the subject and allows for the registration of observers through a delegate callback mechanism. The observers will provide implementations for the delegate methods that will be called by the subject when the event is raised.

If you're familiar with .NET desktop apps and ASP.NET web forms then you must have seen a lot of events: the standard button has a click event – among others – and a corresponding OnClick event handler. Event handlers are also called callbacks. As events and delegates are built-in objects in .NET there's nothing stopping you from implementing the observer pattern using your own events and delegates. You can also pass event arguments to event handlers.

Let's start our implementation with the event arguments. All such objects must derive from the EventArgs object:



    public class CommodityChangeEventArgs : EventArgs
    {
    	public Commodity Commodity { get; set; }

    	public CommodityChangeEventArgs(Commodity commodity)
    	{
    		this.Commodity = commodity;
    	}
    }


So when there's a change in the commodity, its price, its name or any other property, then we can send it along with all other properties that can be interesting to the event handler waiting for such a change. An object waiting for such a change event might be the following CommodityMonitor object:



    public class CommodityMonitor
    {
    	private Commodity _commodity;
    	public event EventHandler<CommodityChangeEventArgs> CommodityChange;

    	public Commodity Commodity
    	{
    		get
    		{
    			return _commodity;
    		}
    		set
    		{
    			_commodity = value;
    			this.OnCommodityChange(new CommodityChangeEventArgs(_commodity));
    		}
    	}

    	protected virtual void OnCommodityChange(CommodityChangeEventArgs e)
    	{
    		if (CommodityChange != null)
    		{
    			CommodityChange(this, e);
    		}
    	}
    }


As the CommodityMonitor monitors commodities it will need a Commodity object. If you are new to events then the event declaration might look unusual but that is how we register an observer. The OnCommodityChange method has the notifier role in this setup and it accepts the appropriate event arguments. This method is called by the Commodity setter: if there's a change then run the notification logic. The notifier then checks if the observer has been set, i.e. whether it's null or not. If yes then it raises the event. Events have a common pair of arguments: the sender, i.e. the object that sends the change event and the event arguments. The sender will simply be "this", i.e. the CommodityMonitor object. It sends out a signal saying that something has changed. What has changed? Anyone who's interested can find it out from the event arguments. Any objects can sign up as observers and they will all be notified.

Here we only set up one event, but an object can raise many events. We may raise separate events for price changes, name changes, weather changes, football score changes etc. They can have their own event arguments as well. So this model provides a high level of granularity and object orientation.

The next thing we want to do is create our observers. We're interested in milk and cocoa so we'll insert two observers. We'll start with MilkObserver:



    public class MilkObserver
    {
    	public MilkObserver(CommodityMonitor monitor)
    	{
    		monitor.CommodityChange += monitor_CommodityChange;
    	}

    	void monitor_CommodityChange(object sender, CommodityChangeEventArgs e)
    	{
    		CheckFilter(e.Commodity);
    	}

    	private void CheckFilter(Commodity commodity)
    	{
    		if (commodity.Name == "milk" && commodity.Price > 2m)
    		{
    			Console.WriteLine("The price of milk has now reached {0}", commodity.Price);
    		}
    	}
    }


The funny looking monitor.CommodityChange += monitor_CommodityChange part performs the registration. We want to register the milk observer to the CommodityChange event of the monitor. It is the monitor_CommodityChange that's going to handle the event. Check its signature, it follows the sender + event args standard. The event handler must have this signature otherwise it cannot be registered as the event handler of the commodity change event. Furthermore the type of the event arguments must match the type declared in CommodityMonitor.

The plus sign declares that we want to register. We could revoke the registration with a minus: monitor.CommodityChange -= monitor_CommodityChange if we are not interested in the changes any more. In fact as you type '+' then IntelliSense will give you the option to create a new event handler – or select an existing one if there's any with the correct signature.

So what's happening if the event is raised? The CheckFilter is run which accepts a Commodity object. The filter will be familiar to you from the first naive implementation: we check the name and the price and if it matches the criteria then we print out the message in the console.

What's even more important I think is that we turned our original primitives-based solution into an object oriented one. We raised an if-statement in the client to a proper object acknowledging its importance in our domain model. It is now an independent object that can be tested separately.

The CocoaObserver looks similar:



    public class CocoaObserver
    {
    	public CocoaObserver(CommodityMonitor monitor)
    	{
    		monitor.CommodityChange += monitor_CommodityChange;
    	}

    	void monitor_CommodityChange(object sender, CommodityChangeEventArgs e)
    	{
    		CheckFilter(e.Commodity);
    	}

    	private void CheckFilter(Commodity commodity)
    	{
    		if (commodity.Name == "cocoa")
    		{
    			Console.WriteLine("The current price of cocoa is {0}", commodity.Price);
    		}
    	}
    }


Insert the following method in Program.cs and call it from Main:



    private static void RunEventBasedExample()
    {
    	CommodityMonitor monitor = new CommodityMonitor();

    	CocoaObserver cocoaObserver = new CocoaObserver(monitor);
    	MilkObserver milkObserver = new MilkObserver(monitor);

    	IEnumerable<Commodity> commodities = new CommodityRepository().GetAllCommodities();
    	foreach (Commodity commodity in commodities)
    	{
    		monitor.Commodity = commodity;
    	}
    }


We create a new commodity monitor and then sign up our two observers. We could add as many observers as we want to. Then we run through our list of commodities and set them as the Commodity property of the monitor object. The property setter then raises the event as explained above. Run through the code by setting breakpoints and pressing F11 in order to follow the exact code execution.

**IObserver-based solution**

.NET4 introduced a new type of interface: IObserver of T and IObservable of T. As the names of the interfaces imply the CommodityMonitor class will have something to do with the IObservable interface as it can be observed. In fact CommodityMonitor will implement this interface. The Milk and CocoaObservers will implement the IObserver interface.

The IObserver interface will force us to implement several methods:

* OnCompleted: indicates that there will be no more changes to the subject
* OnError: when there's an error in processing the subject
* OnNext: when getting the next value of the subject, equivalent to the Commodity setter in the event-based solution

IObservable has a Subscribe method that must be implemented. It represents the registration of observers and returns an IDisposable object. As it returns an IDisposable it will be easier for us to release an observer from the subject properly. This is actually a drawback of the event-based method above as we only subscribe to the event but never release the observer. We think that the garbage collector will take care of that but that's not the case; there's still a reference to the observer from the subject so this resource is not released for garbage collection. With the Subscribe method we get a reference to the IDisposable method so that we can keep track of it and release it when we don't use it any longer.

We'll see in the demo how this is done.

This setup implements the "push" approach of the pattern: whenever there's a change in the subject we push them out to the observers.

Let's implement the pattern using these interfaces.

Here come the updated observers first. They are very simple; they implement the three methods in IObserver mentioned above. You'll recognise the usual filter in the OnNext method. Recall that this runs when we get an update from the subject.



    public class CocoaObserverModern : IObserver<Commodity>
    {
    	public void OnCompleted()
    	{
    		Console.WriteLine("Shop closed.");
    	}

    	public void OnError(Exception error)
    	{
    		Console.WriteLine("Oops: {0}", error.Message);
    	}

    	public void OnNext(Commodity commodity)
    	{
    		if (commodity.Name == "cocoa")
    		{
    			Console.WriteLine("The current price of cocoa is {0}", commodity.Price);
    		}
    	}
    }


We send in the updated Commodity to the OnNext method so that we can check its properties. We're also prepared to handle exceptions that occur in the commodity monitor during the update of the subject. Lastly we tell the user when we're done updating the subjects.



    public class MilkObserverModern : IObserver<Commodity>
    {
    	public void OnCompleted()
    	{
    		Console.WriteLine("Shop closed.");
    	}

    	public void OnError(Exception error)
    	{
    		Console.WriteLine("Oops: {0}", error.Message);
    	}

    	public void OnNext(Commodity commodity)
    	{
    		if (commodity.Name == "milk" && commodity.Price > 2m)
    		{
    			Console.WriteLine("The price of milk has now reached {0}", commodity.Price);
    		}
    	}
    }


Insert a class called ObservableCommodity which is the IObservable version of the previous CommodityMonitor object:



    public class ObservableCommodity : IObservable<Commodity>
    {
            private List<IObserver<Commodity>> _observers = new List<IObserver<Commodity>>();
    	private Commodity _commodity;
    	public Commodity Commodity
    	{
    		get
    		{
    			return _commodity;
    		}
    		set
    		{
    			_commodity = value;
    			this.Notify(_commodity);
    		}
    	}

    	private void Notify(Commodity commodity)
    	{
    		foreach (IObserver<Commodity> observer in _observers)
    		{
    			if (commodity.Name == null || commodity.Price < 0)
    			{
    				observer.OnError(new Exception("Bad Commodity data"));
    			}
    			else
    			{
    				observer.OnNext(commodity);
    			}
    		}
    	}

    	private void Stop()
    	{
    		foreach (IObserver<Commodity> observer in _observers)
    		{
    			if (_observers.Contains(observer))
    			{
    				observer.OnCompleted();
    			}
    		}
    		_observers.Clear();
    	}


    	public IDisposable Subscribe(IObserver<Commodity> observer)
    	{
    		if (!_observers.Contains(observer))
    		{
    			_observers.Add(observer);
    		}
    		return new Unsubscriber(_observers, observer);
    	}

    	private class Unsubscriber : IDisposable
    	{
    		private List<IObserver<Commodity>> _observers;
    		private IObserver<Commodity> _observer;

    		public Unsubscriber(List<IObserver<Commodity>> observers, IObserver<Commodity> observer)
    		{
    			_observers = observers;
    			_observer = observer;
    		}
    		public void Dispose()
    		{
    			if (_observer != null && _observers.Contains(_observer))
    			{
    				_observers.Remove(_observer);
    			}
    		}
    	}
    }


This is more complicated than the CommodityMonitor class. As before we have the Commodity getter and setter. We call the Notify method in the setter. This method will notify each observer in some way: it calls the OnError method of the observer if the Commodity object has an invalid property. It also sends along an Exception object. Otherwise it just calls the OnNext method and sends in the commodity object.

Now check out the Subscribe method. As we said before it returns an IDisposable object. We add the incoming observer to the collection of observers, but we check first if the observer has been added before. We return an object which is the unsubscriber of the observer from the observers collection.

Check out the private Unsubscriber class. It implements the IDisposable interface meaning it has to implement the Dispose method. It holds a reference to the list of observers and the observer, both populated from the Subscribe method. In the Dispose method we check if the observer is not null and if it is contained by the observers list. If both conditions apply then we remove the observer from the observers list.

The Stop method of ObservableCommodity is not actually used. I've included it for your reference; it could be used when there are no more commodity objects in the collection. In that case we can call the OnCompleted method of each observer.

Let's run our code. Insert the following method into Program.cs and call it from Main:



    private static void RunModernObserableBasedApproach()
    {
    	ObservableCommodity oc = new ObservableCommodity();
    	MilkObserverModern milkObserver = new MilkObserverModern();
    	CocoaObserverModern cocoaObserver = new CocoaObserverModern();
    	IEnumerable<Commodity> commodities = new CommodityRepository().GetAllCommodities();
    	using (oc.Subscribe(milkObserver))
    	{
    		using (oc.Subscribe(cocoaObserver))
    		{
    			foreach (Commodity commodity in commodities)
    			{
    				oc.Commodity = commodity;
    			}
    		}
    	}
    }


This looks pretty much like the RunEventBasedExample method. We have our observable object and the two observers. We don't need to send in the commodity monitor to the constructor of the observers this time. Observer registration is taken care of by the Subscribe method. We're using "using" statements as Subscribe returns an IDisposable object. This technique will make sure that we release the observer from the subject.

Run the app and you'll see that even the modern approach works fine and returns the same output as before.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
