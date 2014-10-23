[Source](http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Permalink to Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract")

# Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract

**Introduction**

Factories are extremely popular among design patterns. I have never seen any reliable statistics on the usage of patterns but factories must be among the top three most used patterns. However, that is not to say that they are used correctly. Many developers misunderstand factories to factor out chunks of code to other classes where the factored-out code is encapsulated into a static method as follows:



    double cost = CostFactory.Calculate(input params);


That is definitely NOT a factory. That rather resembles some kind of service, except that methods in services are usually not static. However, you may come across such false implementations of factories – which is probably the case of other pattern types as well.

Factories build objects. They often do this using some parameters that help them decide what kind of concrete object to build but that's not a requirement. The return type of a factory is often some kind of abstraction, i.e. an interface or an abstract class and the factory builds a concrete implementation of the abstraction.

Why would you need such a factory? You may not know in advance which concrete type a certain class is going to use. Example: when a visitor to your route mapping web application can choose different strategies to calculate the route between two cities – fastest, cheapest, most scenic etc. – then how can you know in advance which implementation of the IRouteCalculation strategy to inject in the calculation service? (More about strategies [here][1]). This is definitely a task for a factory – evaluate the client inputs and then select the appropriate strategy.

Another area where factories can help is when the creation logic of an object is so complicated that it should not be encapsulated within its constructor. .NET has several examples of that, e.g.:



    Guid guid = Guid.NewGuid();


Here we don't have access to the setters of the Guid object and probably with a good reason. The implementation details of creating a new Guid are encapsulated in a factory method instead of the caller trying to guess the correct values of each property setter.

**Demo**

Start Visual Studio and create a new Console Application. We'll simulate a simple application that starts and stops machines. Insert the following interface:



    public interface IMachine
    {
            string Name { get; }
    	void TurnOn();
    	void TurnOff();
    }


We'll now create some concrete machine objects:

Robot.cs:



    public class Robot : IMachine
    	{
    		public string Name
    		{
    			get { return "robot"; }
    		}

    		public void TurnOn()
    		{
    			Console.WriteLine("Robot is starting.");
    		}

    		public void TurnOff()
    		{
    			Console.WriteLine("Robot is stopping.");
    		}
    	}


Car.cs:



    public class Car : IMachine
    	{
    		public string Name
    		{
    			get { return "car"; }
    		}

    		public void TurnOn()
    		{
    			Console.WriteLine("Car is starting.");
    		}

    		public void TurnOff()
    		{
    			Console.WriteLine("Car is stopping.");
    		}
    	}


MicrowaveOven.cs:



    public class MicrowaveOven : IMachine
    	{
    		public string Name
    		{
    			get { return "microwave oven"; }
    		}

    		public void TurnOn()
    		{
    			Console.WriteLine("Microwave oven is starting.");
    		}

    		public void TurnOff()
    		{
    			Console.WriteLine("Microwave oven is stopping.");
    		}
    	}


UnknownMachine.cs:



    public class UnknownMachine : IMachine
    	{
    		public string Name
    		{
    			get { return string.Empty; }
    		}

    		public void TurnOn()
    		{

    		}

    		public void TurnOff()
    		{

    		}
    	}


UnknownMachine.cs as you see performs nothing and has no name – this is the Null Object pattern implementation of the interface. It is used instead of a null value when no suitable machine implementation has been found by the factory. I'll write a post on that pattern as well later on.

The Main method if Program.cs looks as follows:



    static void Main(string[] args)
    		{
    			string description = args[0];
    			IMachine machine = GetMachine(description);
    			machine.TurnOn();
    			machine.TurnOff();

    			Console.ReadKey();
    		}

    		private static IMachine GetMachine(string description)
    		{
    			switch (description)
    			{
    				case "robot":
    					return new Robot();
    				case "car":
    					return new Car();
    				default:
    					return new UnknownMachine();
    			}
    		}


This is not terribly complicated I hope. We don't know in advance which machine the user wants to start and stop so we let a private static method take care of that. Note that we haven't yet included the microwave oven as an option. Imagine that our machine palette now includes the microwave oven. In case we want to ensure that the client can access this new machine as well we have to extend the switch statement as follows:



    case "oven":
    	return new MicrowaveOven();


This looks like a small price to pay but we violated letter 'O' in SOLID, i.e. the Open/Closed principle: a class is open for extension but closed for modification. Also, Program.cs must be aware of the different IMachine implementations. It is Program.cs that is made responsible for finding the correct concrete type which is not the correct approach. Ideally Program.cs should only be concerned with the IMachine interface, nothing else. Last, but not least every time we add a new IMachine implementation to our app we have to return to Program.cs and extend the GetMachine method.

With the help of the factory pattern we would like to:

* Separate out the object creation logic, i.e. relieve Program.cs of that task
* Add new implementations without breaking the Open/Closed principle
* Externalise object creation rules to a database or a configuration file: a classic example is the type of Membership object to use as defined in web.config or app.config. That is also an application of the factory pattern

**Solution with a concrete factory**

The first solution is based on the Concrete Factory pattern. It's called Concrete because we need to new up a factory class and then call some method on it whose name will surely include 'Create'. Add the following MachineFactory class into the solution:



    public class MachineFactory
    	{
    		Dictionary<string, Type> machines;

    		public MachineFactory()
    		{
    			LoadTypesICanReturn();
    		}

    		public IMachine CreateInstance(string description)
    		{
    			Type t = GetTypeToCreate(description);

    			if (t == null)
    				return new UnknownMachine();

    			return Activator.CreateInstance(t) as IMachine;
    		}

    		private Type GetTypeToCreate(string machineName)
    		{
    			foreach (var machine in machines)
    			{
    				if (machine.Key.Contains(machineName))
    				{
    					return machines[machine.Key];
    				}
    			}

    			return null;
    		}

    		private void LoadTypesICanReturn()
    		{
    			machines = new Dictionary<string, Type>();

    			Type[] typesInThisAssembly = Assembly.GetExecutingAssembly().GetTypes();

    			foreach (Type type in typesInThisAssembly)
    			{
    				if (type.GetInterface(typeof(IMachine).ToString()) != null)
    				{
    					machines.Add(type.Name.ToLower(), type);
    				}
    			}
    		}
    	}


As you can see we collect the types that implement the IMachine interface using Reflection in a dictionary. Then we try to find the concrete type using the dictionary key. Note the signature of CreateInstance: it returns an abstraction based on some input – this is very common to factories. Program.cs can be modified as follows:



    static void Main(string[] args)
    		{
    			string description = args[0];
    			IMachine machine = new MachineFactory().CreateInstance(description);
    			machine.TurnOn();
    			machine.TurnOff();

    			Console.ReadKey();
    		}


So the first solution is not terribly complicated. Program.cs is now free of the burden of creating the correct concrete IMachine object. We don't have to extend the MachineFactory class either as we add new Machine types later. All new types will be automatically picked up by Reflection. Program.cs is not any longer aware of the concrete Machine types.

A criticism is that the caller needs to know exactly which factory to call. This creates an unnecessary coupling between the caller and a concrete class. Ideally even the type of the factory should be hidden behind an interface – more on this later.

**Solution with a static factory**

I will not spend much time on this as it is the same as the concrete factory type except that the Create method is static:



    public static IMachine CreateInstance(string description)


And the call to this static method looks as follows:



    MachineFactory.CreateInstance(description);


It's easy to realise that this is the exact same solution as concrete factory implementation and doesn't take us any closer to the factory abstraction nirvana.

**Solution with abstract factory**

The solution that abstracts away the tight coupling to a factory is provided by the abstract factory. It is called 'abstract' because it is an interface type – and also returns abstractions in its methods. Insert the following interface to the project:



    public interface IMachineFactory
    	{
    		IMachine CreateInstance(string description);
    	}


Change the MachineFactory class declaration so that it implements this interface:



    public class MachineFactory : IMachineFactory


We're now ready to replace the code which news up a MachineFactory with something more flexible. There are several different ways to declare which implementation of the IMachineFactory we want to use. We'll take the Reflection approach again. Open the Properties of the project and select Settings:

![Project settings][2]

The editor should show a message saying that the project does not contain a default settings file. Click on that link to create one. Add a setting to declare the IMachineFactory type. Make sure to provide the fully qualified name of the class. In my case it looks like the following:

![Declare machine factory in settings][3]

Insert the following private method into Program.cs:



    private static IMachineFactory LoadFactory()
    {
    	string factoryName = Properties.Settings.Default.DefaultMachineFactory;
    	return Assembly.GetExecutingAssembly().CreateInstance(factoryName) as IMachineFactory;
    }


We read the fully qualified name of the default machine factory from the settings file and then instantiate the object using Reflection. In a real life Web or Desktop app this dependency would be injected into a Controller or Service – depending on where we need the Factory – using some IoC container such as StructureMap.

We can now update the Main method as follows:



    static void Main(string[] args)
    {
    	string description = args[0];
    	IMachine machine = LoadFactory().CreateInstance(description);
    	machine.TurnOn();
    	machine.TurnOff();
    	Console.ReadKey();
    }


This was the simplest implementation of the abstract factory pattern. You can have different implementations of the IMachineFactory of course, for example:

* LargeMachineFactory: factory to build large machines, such as cranes, trucks and the like
* DangerousMachineFactory: for machines that need special training
* HouseholdMachineFactory: for things like a fridge, oven etc.

You can then hide the details of creating the different machines behind these factories. It is reasonable to think that it may be programmatically difficult to build a new Crane object, so it's better to hide those details. The concrete factories will know if they have to call some special property setters or other methods in order to build up an object in a consistent state.

You can of course extend the IMachineFactory interface to include more options, such as CreateDefaultMachine(). The implementations will then build some default IMachine object making it even easier for the caller to get started.

You may ask how the application will get hold of the correct IMachineFactory type if we have 2 or more implementations. The most likely solution is of course a… …factory! Remember: if you have several implementations of an abstraction and you don't know which concrete type to use until the user provides some input then a factory is a good candidate to solve the issue.

We have successfully eliminated the tight coupling to classes: Program.cs has no knowledge of the concrete IMachine types or the concrete IMachineFactory types.

It's important to note that the concrete factories will be tightly coupled to the concrete types they produce. However, I think this is acceptable as long as the factories return objects that are located within the same domain boundary and the same assembly – the MachineFactory will return Machines and all Machine objects and the machine factory will sit in the same assembly, possibly the same namespace indicating that they are related objects belonging to the same context.

View the list of posts on Architecture and Patterns [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/04/29/design-patterns-and-practices-in-net-the-strategy-pattern/ "Design patterns and practices in .NET: the Strategy Pattern"
[2]: http://dotnetcodr.files.wordpress.com/2013/04/projectsettings.png?w=630
[3]: http://dotnetcodr.files.wordpress.com/2013/04/declaremachinefactoryoption.png?w=630&h=78
[4]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
