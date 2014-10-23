[Source](http://dotnetcodr.com/2013/05/27/design-patterns-and-practices-in-net-the-command-pattern/ "Permalink to Design patterns and practices in .NET: the Command pattern")

# Design patterns and practices in .NET: the Command pattern

**Introduction**

The Command pattern is used to represent an action as an object. The action is abstracted away in an interface. In its simplest form the interface may only have a single method, e.g. Execute() which takes no parameters and returns no value. The interface gives an opportunity to the client to execute some command when appropriate.

The main goal of the pattern is to decouple the client that wants to execute a command from the details and dependencies of the command logic. The client only needs to talk to the interface and forget the implementation details.

The pattern also enables us to queue commands and execute them later in a certain order if that's necessary. We can even store these commands somewhere, e.g. in a database and have them execute at a later point.

The Command pattern is also known by different names such as the Action pattern or the Transaction pattern.

Examples of applicability include logging, validation and undo operations. Logging can be abstracted away in Actions and executed later. Undo can be performed if we keep track of the commands so that they can be reversed.

As we said before the structure of the ICommand interface can be very simple with only one Execute method. It can also allow for other operations such as Validate or Undo or whatever your Action may need. An important point to stress is that the correct implementation of the pattern will make sure that the Action has all the necessary dependencies in order to carry out the methods in the interface. The client will have a reference on an ICommand object and call its methods without worrying about the dependencies of the actual concrete implementation of the ICommand interface.

In other words the Commands must be self-contained as the client does not pass in any arguments. The Command must have all the dependencies and contexts ready and will not depend on any inputs from the client. A common technique is to use [Factories][1] to build the correct concrete command. This point makes the Command pattern very different from the [strategy pattern][2] where the client provides inputs to choose the correct strategy. A Strategy will typically be a very specific object – DhlPostingStrategy, FedExPostingStrategy, SchenkerPostingStrategy – whereas a Command can be a lot more general: paint the car, change the lightbulb, refresh the screen etc.

It is easy to extend our system with new concrete commands. In a naive solution you may have a long switch statement where you check some string value representing the command. A typical example is a command-line executable where the incoming command string is inspected in a long switch statement – case "Insert", case "Update" etc. If you have a new command you have to extend that switch statement which violates the Open/Closed principle of SOLID. Instead you can create new commands by implementing the ICommand interface.

**Demo**

In the demo we'll simulate a command-line order management system where users can perform a couple of actions on an Order.

Open Visual Studio and create a new Console application. Insert a new folder called Commands. The first element to add there is the ICommand interface:



    public interface ICommand
        {
            void Execute();
        }


It looks as we discussed before: very clean with only a single zero parameter Execute method.

As mentioned above the concrete Command will be located by a factory method whose interface looks as follows:



    public interface ICommandFactory
        {
            string CommandName { get; }
            string Description { get; }

            ICommand MakeCommand(string[] arguments);
        }


The key method is the MakeCommand which will build the correct Command object based on the arguments. It also has two additional properties to describe the Command.

The ICommand and ICommandFactory interfaces are implemented by the following 3 classes that create, update and ship and order:



    public class CreateOrderCommand : ICommand, ICommandFactory
    	{
    		public int Quantity { get; set; }
    		public string ProductName { get; set; }

    		public void Execute()
    		{
    			//create the order, code ignored

    			//simulate logging
    			Console.WriteLine(string.Format("Order entered: {0} pieces of {1}.", Quantity, ProductName));
    		}

    		public string CommandName
    		{
    			get { return "CreateOrder"; }
    		}

    		public string Description
    		{
    			get { return CommandName; }
    		}

    		public ICommand MakeCommand(string[] arguments)
    		{
    			return new CreateOrderCommand() { ProductName = arguments[1], Quantity = Convert.ToInt32(arguments[2]) };
    		}
    	}


The MakeCommand implementation returns a new instance of the CreateOrderCommand with the necessary arguments for order creation. Note the array index [1] for the product name. It is assumed that element 0 will be the name of the command in the command line, such as "Create" followed by the necessary input parameters.

You could of course have separate objects that implement ICommand and ICommandFactory but I thought this was really convenient for the little command-line demo: when adding a new command only a single object needs to be created that takes care of its own creation as well.

The ShipOrderCommand and UpdateQuantityCommand objects work the same way: implement the interfaces and build valid ICommand objects.



    public class ShipOrderCommand : ICommand, ICommandFactory
    	{
    		public int OrderId { get; set; }

    		public void Execute()
    		{
    			//ship the order, code ignored

    			//simulate logging
    			Console.WriteLine(string.Format("Order {0} shipped.", OrderId));
    		}

    		public string CommandName
    		{
    			get { return "ShipOrder"; }
    		}

    		public string Description
    		{
    			get { return CommandName; }
    		}

    		public ICommand MakeCommand(string[] arguments)
    		{
    			return new ShipOrderCommand() { OrderId = Convert.ToInt32(arguments[1]) };
    		}
    	}




    public class UpdateQuantityCommand : ICommand, ICommandFactory
    	{
    		public int NewQuantity { get; set; }

    		public void Execute()
    		{
    			// simulate updating a database
    			int oldQuantity = 5;
    			Console.WriteLine("DATABASE: Updated");

    			// simulate logging
    			Console.WriteLine("LOG: Updated order quantity from {0} to {1}", oldQuantity, NewQuantity);
    		}

    		public string CommandName { get { return "UpdateQuantity"; } }
    		public string Description { get { return "UpdateQuantity number"; } }

    		public ICommand MakeCommand(string[] arguments)
    		{
    			return new UpdateQuantityCommand { NewQuantity = int.Parse(arguments[1]) };
    		}
    	}


Finally we have a placeholder command for cases where the client requested an unavailable action:



    public class NotFoundCommand : ICommand
    	{
    		public string Name { get; set; }
    		public void Execute()
    		{
    			Console.WriteLine("Couldn't find command: " + Name);
    		}
    	}


This actually follows a mini pattern called the Null Object pattern available [here][3]: we want to return a valid object instead of a null to avoid NullReferenceExceptions.

The next class to insert is a helper class that selects the correct command.



    public class CommandParser
    	{
    		readonly IEnumerable<ICommandFactory> availableCommands;

    		public CommandParser(IEnumerable<ICommandFactory> availableCommands)
    		{
    			this.availableCommands = availableCommands;
    		}

    		internal ICommand ParseCommand(string[] args)
    		{
    			string requestedCommandName = args[0];

    			ICommandFactory command = FindRequestedCommand(requestedCommandName);
    			if (null == command)
    				return new NotFoundCommand { Name = requestedCommandName };

    			return command.MakeCommand(args);
    		}

    		ICommandFactory FindRequestedCommand(string commandName)
    		{
    			return availableCommands
    				.FirstOrDefault(cmd => cmd.CommandName == commandName);
    		}
    	}


We pass in the list of available commands to the constructor. The FindRequestedCommand picks the correct command based on the command name using a simple LINQ operation. This is where the CommandName property of each command becomes an important field which uniquely identifies them. If the command is not found then a NotFoundCommand is returned.

We have all the elements to build our glorious Main method that acts as the caller:



    class Program
    	{
    		static void Main(string[] args)
    		{
    			IEnumerable<ICommandFactory> availableCommands = GetAvailableCommands();

    			if (args.Length == 0)
    			{
    				PrintUsage(availableCommands);
    				return;
    			}

    			CommandParser parser = new CommandParser(availableCommands);
    			ICommand command = parser.ParseCommand(args);

    			command.Execute();
                            Console.ReadKey();
    		}

    		static IEnumerable<ICommandFactory> GetAvailableCommands()
    		{
    			return new ICommandFactory[]
                            {
                                new CreateOrderCommand(),
                                new UpdateQuantityCommand(),
                                new ShipOrderCommand(),
                            };
    		}

    		private static void PrintUsage(IEnumerable<ICommandFactory> availableCommands)
    		{
    			Console.WriteLine("Usage: LoggingDemo CommandName Arguments");
    			Console.WriteLine("Commands:");
    			foreach (ICommandFactory command in availableCommands)
    				Console.WriteLine("  {0}", command.Description);
    		}
    	}


We keep the list of available concrete Command objects in a list. This is the easiest way to store the commands but it needs to be maintained of course: if you add a new command then this list needs to be extended which is a violation of the Open/Closed principle actually. You can avoid this using Reflection and you can find an example [here][1] how to do it, watch out for the LoadTypesICanReturn() method. The CommandParser will then return the correct command based on the input parameters to the console application. Finally the command is executed.

As you see we don't have some long switch statement in the caller method – Main – that checks the incoming string elements of the args array. The Main method relies on specialised classes to take care of that. Also, the Main method is not aware of the actual workings of the command objects – it doesn't even know the type of the object returned by the CommandParser. An additional benefit is that you can separately test each Command object.

The easiest way to test the application is to add the command line arguments to the project properties:

![Command line arguments][4]

Enter these arguments and you'll see in the console that the order was created. Test with arguments for the Update and Ship operations. Finally enter some unimplemented command, such as DeleteOrder. You'll see that the command was not available. With the Command pattern in place you can easily add the new DeleteOrder command and test is in isolation.

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[2]: http://dotnetcodr.com/2013/04/29/design-patterns-and-practices-in-net-the-strategy-pattern/ "Design patterns and practices in .NET: the Strategy Pattern"
[3]: http://dotnetcodr.com/2013/05/06/design-patterns-and-practices-in-net-the-null-object-pattern/ "Design patterns and practices in .NET: the Null Object pattern"
[4]: http://dotnetcodr.files.wordpress.com/2013/04/commandlinepropertiesinvs.png?w=630&h=183
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
