[Source](http://dotnetcodr.com/2013/06/06/design-patterns-and-practices-in-net-the-mediator-pattern/ "Permalink to Design patterns and practices in .NET: the Mediator pattern")

# Design patterns and practices in .NET: the Mediator pattern

**Introduction**

The Mediator pattern can be applicable when several objects of a related type need to communicate with each other and this communication is complex. Consider a scenario where incoming aircraft need to carefully communicate with each other for safety reasons. They constantly need to know the position of all other planes, meaning that each aircraft needs to communicate with all other aircraft.

Think of a first naive solution in this case. You have 3 types of aircraft in your domain model: Boeing, Airbus and Fokker. Consider that each type needs to communicate with the other two types. The first approach would be to check the type of the other aircraft directly in code such as this in the Airbus class:



    if (otherAircraft is Boeing)
    {
        //do something
    }
    else if (otherAircraft is Fokker)
    {
        //do something else
    }


You would have similar if-else statements in the other two classes. You can imagine how this gets out of control as we add new types of aircraft. You'll need to revisit the code of all other types and extend the if-else statements to accommodate the new type thereby violating the open-close design principle. Also, it's bad practice to let one class intimately know about the inner workings of another class, which is the case here.

We need to decouple the related objects from each other. This is where a Mediator enters the scene. A mediator encapsulates the interaction logic among related objects. The pattern allows loose coupling between objects by keeping them from directly referring to each other explicitly. The interaction logic is centralised in one place only.

The above problem has been solved through air traffic controllers in the real world. It is those professionals that will monitor the position of each aircraft in their zone and communicate with them directly. I don't know if pilots of different commercial planes directly contact each other but I can imagine that it occurs very rarely. If we applied the same solution in this case then the pilots would need to know if every type of aircraft they may encounter during their flight.

There are a couple of formal elements to the Mediator pattern:

* **Colleagues**: components that need to communicate with each other, very often of the same base type. These objects will have no knowledge of each other but will know about the Mediator component
* **Mediator**: a centralised component that manages communication between the colleagues. The colleagues will have a dependency on this object through an abstraction

**Demo**

We'll build on the idea mentioned above: the colleague elements are the incoming aircraft and the mediator is represented by an air traffic controller.

Start up Visual Studio and create a new Console application. Insert a base class for all colleagues called Aircraft:



    public abstract class Aircraft
    	{
    		private readonly IAirTrafficControl _atc;
    		private int _currentAltitude;

    		protected Aircraft(string callSign, IAirTrafficControl atc)
    		{
    			_atc = atc;
    			CallSign = callSign;
    			_atc.RegisterAircraftUnderGuidance(this);
    		}

    		public abstract int Ceiling { get; }

    		public string CallSign { get; private set; }

    		public int Altitude
    		{
    			get { return _currentAltitude; }
    			set
    			{
    				_currentAltitude = value;
    				_atc.ReceiveAircraftLocation(this);
    			}
    		}

    		public void Climb(int heightToClimb)
    		{
    			Altitude += heightToClimb;
    		}

    		public override bool Equals(object obj)
    		{
    			if (obj.GetType() != this.GetType()) return false;

    			var incoming = (Aircraft)obj;
    			return this.CallSign.Equals(incoming.CallSign);
    		}

    		public override int GetHashCode()
    		{
    			return CallSign.GetHashCode();
    		}

    		public void WarnOfAirspaceIntrusionBy(Aircraft reportingAircraft)
    		{
    			//do something in response to the warning
    		}
    	}


Every aircraft will have a call sign and a dependency on an air flight controller in the form of the IAirTrafficController interface. We'll take a look at that interface shortly but you'll see that we put the aircraft under the responsibility of that air traffic control. We tell the mediator that there's a new object that it needs to communicate with.

You can imagine that as commercial aircraft fly to their destinations they enter and leave the zones of various air traffic controls on their way. So in a more complete interface would have a de-register method as well but we can omit that to keep the demo simple.

Then comes an abstract property called Ceiling that shows the maximum flying altitude of the aircraft. Each concrete type will need to communicate this property about itself. This is followed by the current Altitude of the aircraft. You'll see that in the property setter we send the current location to the air traffic controller.

The rest of the class is pretty simple: we let the aircraft climb, we make them comparable and we let them receive a warning signal if there is another aircraft too close.

The IAirTrafficControl interface looks as follows:



    public interface IAirTrafficControl
    	{
    		void ReceiveAircraftLocation(Aircraft location);
    		void RegisterAircraftUnderGuidance(Aircraft aircraft);
    	}


The type that implements the IAirTrafficControl interface will be responsible to implement these methods. The Aircraft object doesn't care how its position is registered at the control.

We have the following concrete types of aircraft:



    public class Boeing : Aircraft
    	{
    		public Boeing(string callSign, IAirTrafficControl atc)
    			: base(callSign, atc)
    		{
    		}

    		public override int Ceiling
    		{
    			get { return 33000; }
    		}
    	}




    public class Fokker : Aircraft
    	{
    		public Fokker(string callSign, IAirTrafficControl atc) : base(callSign, atc)
            {
            }

    		public override int Ceiling
    		{
    			get { return 40000; }
    		}
    	}




    public class Airbus : Aircraft
    	{
    		public Airbus(string callSign, IAirTrafficControl atc)
    			: base(callSign, atc)
    		{
    		}

    		public override int Ceiling
    		{
    			get { return 40000; }
    		}
    	}


These should be fairly easy to follow. If you later want to introduce a new type of aircraft just derive from the Aircraft base class and then it will automatically become a colleague component to the existing types. The important thing to note is that in any concrete type there is no reference to any other type. The colleagues are completely independent. That dependency is replaced by the IAirTrafficControl abstraction which is the definition of the mediator. You can imagine that we can pass in different types of air traffic control as the plane flies towards its destination: Stockholm, Copenhagen, Hamburg etc. They may all treat the aircraft in their zones little differently.

Let's take a look at the concrete mediator:



    public class Tower : IAirTrafficControl
    	{
    		private readonly IList<Aircraft> _aircraftUnderGuidance = new List<Aircraft>();

    		public void ReceiveAircraftLocation(Aircraft reportingAircraft)
    		{
    			foreach (Aircraft currentAircraftUnderGuidance in _aircraftUnderGuidance.
    				Where(x => x != reportingAircraft))
    			{
    				if (Math.Abs(currentAircraftUnderGuidance.Altitude - reportingAircraft.Altitude) < 1000)
    				{
    					reportingAircraft.Climb(1000);
    					//communicate to the class
    					currentAircraftUnderGuidance.WarnOfAirspaceIntrusionBy(reportingAircraft);
    				}
    			}
    		}

    		public void RegisterAircraftUnderGuidance(Aircraft aircraft)
    		{
    			if (!_aircraftUnderGuidance.Contains(aircraft))
    			{
    				_aircraftUnderGuidance.Add(aircraft);
    			}
    		}
    	}


The Tower maintains a list of Aircraft that belong under its control. The list is augmented using the implemented RegisterAircraftUnderGuidance method.

The ReceiveAircraftLocation method includes a bit of logic. When an aircraft reports its position then the Tower loops through the list of aircraft currently under its control – except for the one reporting its position – and if any other plane is within 1000 feet then the reporting aircraft needs to climb 1000 feet and the current aircraft in the loop is warned of another aircraft flying too close. This emergency call is a form of indirect communication between two colleagues: the reporting aircraft communicates tells the other aircraft of the violation of the flying distance. The communication is mediated using the Tower class, the two concrete aircraft still have no knowledge about each other, all communication is handled through abstractions.

Let's look at the Main method:



    static void Main(string[] args)
    {
    	IAirTrafficControl tower = new Tower();

    	Aircraft flight1 = new Airbus("AC159", tower);
    	Aircraft flight2 = new Boeing("WS203", tower);
    	Aircraft flight3 = new Fokker("AC602", tower);

    	flight1.Altitude += 1000;
    }


We create a mediator and the aircraft currently flying. That's all we need to introduce a new aircraft: tell it about the mediator it can use for its communication purposes through its constructor.

The last row says that the Airbus will increase its altitude by 1000 feet. If you recall then the Altitude property setter will initiate a communication with the air traffic control. The aircraft indicates its new altitude and the Tower will loop through the list of aircraft currently under its control and see of any other aircraft object is too close to the reporting one.

The main advantage of the mediator pattern is abstraction: we hide the communicating colleagues from each other and let them talk to each other through another abstraction, i.e. the mediator. An aircraft can only belong to a single mediator and a mediator can have many colleagues under its control, i.e. this is a one-to-many relationship. If we remove the mediator then we're immediately dealing with a many-to-many relationship among colleagues. If you're like me then you probably prefer the former type of relationship to the latter.

The disadvantage of the mediator lies in its possible complexity. Our example is still very simple but in real life examples the communication can become very messy with if statements checking the type of the colleague. The mediator can grow very large as more and more communication logic enters the picture. The problem can be mitigated by breaking down the mediator to smaller chunks adhering to the single responsibility principle.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
