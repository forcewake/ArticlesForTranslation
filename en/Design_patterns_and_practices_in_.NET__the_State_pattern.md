[Source](http://dotnetcodr.com/2013/05/16/design-patterns-and-practices-in-net-the-state-pattern/ "Permalink to Design patterns and practices in .NET: the State pattern")

# Design patterns and practices in .NET: the State pattern

**Introduction**

The State design pattern allows to change the behaviour of a method through the state of an object. A typical scenario where the state pattern is used when an object goes through different phases. An issue in a bug tracking application can have the following states: inserted, reviewed, rejected, processing, resolved and possibly many others. Depending on the state of the bug the behaviour of the underlying system may also change: some methods will become (un)available and some of them will change their behaviour. You may have seen or even produced code similar to this:



    public void ProcessBug()
    {
    	switch (state)
    	{
    		case "Inserted":
    			//call another method based on the current state
    			break;
    		case "Reviewed":
    			//call another method based on the current state
    			break;
    		case "Rejected":
    			//call another method based on the current state
    			break;
    		case "Resolved":
    			//call another method based on the current state
    			break;
    	}
    }


Here we change the behaviour of the ProcessBug() method based on the state of the "state" parameter, which represents the state of the bug. You can imagine that once a bug has reached the Rejected status then it cannot be Reviewed any more. Also, once it has been reviewed, it cannot be deleted. There are other similar scenarios like that where the available actions and paths depend on the actual state of an object.

Suppose you have public methods to perform certain operations on an object: Insert, Delete, Edit, Resolve, Reject. If you follow the above solution then you will have to insert a switch statement in each and check the actual state of the object and act accordingly. This is clearly not maintainable; it's easy to get lost in the chain of the logic, it gets difficult to update the code if the rules change and the class code grows unreasonably large compared to the amount of logic carried out.

There are other issues with the naive switch-statement approach:

* The states are hard coded which offers no or little extensibility
* If we introduce a new state we have to extend every single switch statement to account for it
* All actions for a particular state are spread around the actions: a change in one state action may have an effect on the other states
* Difficult to unit test: each method can have a switch statement creating many permutations of the inputs and the corresponding outputs

In the switch statement solution the states are relegated to simple string properties. In reality they are more likely to be more important objects that are part of the core Domain. Hence that logic should be encapsulated into separate objects that can be tested independently of the other concrete state types.

**Demo**

We'll simulate an e-commerce application where an order can go through the following states: New, Shipped, Cancelled. The rules are simple: a new order can be shipped or cancelled. Shipped and cancelled orders cannot be shipped or cancelled again.

Fire up Visual Studio and create a blank solution. Insert a Windows class library called Domains. You can delete Class1.cs. The first item we'll insert is a simple enumeration:



    public enum OrderStatus
    	{
    		New
    		, Shipped
    		, Cancelled
    	}


Next we'll insert the interface that each State will need to implement, IOrderState:



    public interface IOrderState
    	{
    		bool CanShip(Order order);
    		void Ship(Order order);
    		bool CanCancel(Order order);
    		void Cancel(Order order);
                    OrderStatus Status {get;}
    	}


Each concrete state will need to handle these methods independently of the other state types. The Order domain looks like this:



    public class Order
    	{
    		private IOrderState _orderState;

    		public Order(IOrderState orderState)
    		{
    			_orderState = orderState;
    		}

    		public int Id { get; set; }
    		public string Customer { get; set; }
    		public DateTime OrderDate { get; set; }
    		public OrderStatus Status
    		{
    			get
    			{
    				return _orderState.Status;
    			}
    		}
    		public bool CanCancel()
    		{
    			return _orderState.CanCancel(this);
    		}
    		public void Cancel()
    		{
    			if (CanCancel())
    				_orderState.Cancel(this);
    		}
    		public bool CanShip()
    		{
    			return _orderState.CanShip(this);
    		}
    		public void Ship()
    		{
    			if (CanShip())
    				_orderState.Ship(this);
    		}

    		void Change(IOrderState orderState)
    		{
    			_orderState = orderState;
    		}
    	}


As you can see each Order related action is delegated to the OrderState object where the Order object is completely oblivious of the actual state. It only sees the interface, i.e. an abstraction, which facilitates loose coupling and enhanced testability.

Let's implement the Cancelled state first:



    public class CancelledState : IOrderState
    	{
    		public bool CanShip(Order order)
    		{
    			return false;
    		}

    		public void Ship(Order order)
    		{
    			throw new NotImplementedException("Cannot ship, already cancelled.");
    		}

    		public bool CanCancel(Order order)
    		{
    			return false;
    		}

    		public void Cancel(Order order)
    		{
    			throw new NotImplementedException("Already cancelled.");
    		}

    		public OrderStatus Status
    		{
    			get
    			{
    				return OrderStatus.Cancelled;
    			}
    		}
    	}


This should be easy to follow: we incorporate the cancellation and shipping rules within this concrete state.

ShippedState.cs is also straighyforward:



    public class ShippedState : IOrderState
    	{
    		public bool CanShip(Order order)
    		{
    			return false;
    		}

    		public void Ship(Order order)
    		{
    			throw new NotImplementedException("Already shipped.");
    		}

    		public bool CanCancel(Order order)
    		{
    			return false;
    		}

    		public void Cancel(Order order)
    		{
    			throw new NotImplementedException("Already shipped, cannot cancel.");
    		}

    		public OrderStatus Status
    		{
    			get { return OrderStatus.Shipped; }
    		}
    	}


NewState.cs is somewhat more exciting as we change the state of the order after it has been shipped or cancelled:



    public class NewState : IOrderState
    	{
    		public bool CanShip(Order order)
    		{
    			return true;
    		}

    		public void Ship(Order order)
    		{
    			//actual shipping logic ignored, only changing the status
    			order.Change(new ShippedState());
    		}

    		public bool CanCancel(Order order)
    		{
    			return true;
    		}

    		public void Cancel(Order order)
    		{
    			//actual cancellation logic ignored, only changing the status;
    			order.Change(new CancelledState());
    		}

    		public OrderStatus Status
    		{
    			get { return OrderStatus.New; }
    		}
    	}


That's it really, the state pattern is not more complicated than this.

We separated out the state-dependent logic to standalone classes that can be tested independently. It's now easy to introduce new states later. We won't have to extend dozens of switch statements – the new state object will handle that logic internally. The Order object is no longer concerned with the concrete state objects – it delegates the cancellation and shipping actions to the states.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
