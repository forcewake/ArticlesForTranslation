[Source](http://dotnetcodr.com/2013/07/29/design-patterns-and-practices-in-net-the-bridge-pattern/ "Permalink to Design patterns and practices in .NET: the Bridge pattern")

# Design patterns and practices in .NET: the Bridge pattern

**Introduction**

This pattern was originally defined by the Gang of Four as follows: decouple an abstraction from its implementation so the two can very independently.

You probably know what abstractions are: base classes and interfaces, i.e. objects to generalise our code and/or make a group of objects related in some way. We abstract away ideas that can have multiple concrete implementations. Normally if a class implements an interface or derives from an abstract superclass then the abstraction and the implementation are tightly coupled. If you change the structure of the interface, such as add a new method to it, then all implementations will need to change as well. The bridge pattern aims to decouple them so that we get rid of this coupling. We'll introduce a higher level of abstraction to abstract away the implementation from the original abstraction. We'll end up with two levels of abstraction instead of just one.

Let's try and imagine a real-world example: a travel agent putting together different types of holiday in a brochure. The agency organises trips to 4 countries: Cyprus, Turkey, Greece and Italy. It also offers a couple of extras: private pool, free gym, all-inclusive and massage. The agency wants to show the full programme as follows:

* Italy – private pool – free gym
* Cyprus – private pool – free gym
* Turkey – private pool – free gym
* Greece – private pool – free gym
* Italy – all inc – massage
* Cyprus – all inc – massage
* Turkey – all inc – massage
* Greece – all inc – massage
* …so on and so forth

In other words the agency wants to show the full combos as single items. You'll see a lot of duplication here: the combination of extras is repeated several times, such as 'all inc – massage'. However, the person putting together this brochure was committed to show all the options in one place. We can think of the concept 'trip' as the abstraction and the combos as the implementations: we can have Italy with private-pool-and-free-gym; we can have Italy with all-inclusive-and-massage; we can have Cyprus with private-pool-and-free-gym etc.

Another agent looks at this and 'refactors' the brochure. She realises that the offers can be simplified greatly to:

* Cyprus
* Turkey
* Greece
* Italy

with

* private pool – free gym
* all inc – massage

Now it's easier to build combos: we can add new countries and new combinations of extras. In the first example if we introduce a new combination of extras, such as private-pool-with-massage then the agent will need to create 4 new items in the brochure, one for each country. We have a similar problem if we add a new country. In the second solution the client can choose how to combine the destination with the extra combos. It is not the agent, i.e. the developer, that builds all the combos, but the client.

**Demo**

In the demo we'll simulate a building management application. Imagine that some authority is responsible for maintaining state-owned buildings and they need help with creating a manager application. Open Visual Studio and create a new console application. We'll start with 3 basic domain objects with their own properties and a Print method in common:



    public class Apartment
    {
    	public string Description { get; set; }
    	public Dictionary<string, string> Rooms { get; set; }

    	public Apartment()
    	{
    		Rooms = new Dictionary<string, string>();
    	}

    	public void Print()
    	{
    		Console.WriteLine("Apartment: ");
    		Console.WriteLine(Description);
    		foreach (KeyValuePair<string, string> room in Rooms)
    		{
    			Console.WriteLine(string.Concat("Room: ", room.Key, ", room description: ", room.Value));
    		}
    	}
    }




    public class House
    {
    	public string Description { get; set; }
    	public string Owner { get; set; }
    	public string Address { get; set; }

    	public void Print()
    	{
    		Console.WriteLine("House: ");
    		Console.WriteLine(string.Concat("Description: ", Description, " owner:  ", Owner, " address: ", Address));
    	}
    }




    public class TrainStation
    {
    	public int NumberOfTrains { get; set; }
    	public string Location { get; set; }
    	public int NumberOfPassangers { get; set; }
    	public string Director { get; set; }

    	public void Print()
    	{
    		Console.WriteLine("Train station: ");
    		Console.WriteLine(string.Concat("Number of trains: ", NumberOfTrains, ", Location: ", Location, ", number of passangers: ", NumberOfPassangers
    			, ", director: ", Director));
    	}
    }


We construct and print the objects in Main as follows:



    class Program
    {
    	static void Main(string[] args)
    	{
    		Apartment apartment = new Apartment();
    		apartment.Description = "Nice apartment.";
    		apartment.Rooms.Add("1/A", "Cozy little room");
    		apartment.Rooms.Add("2/C", "To be renovated");
    		apartment.Print();

    		House house = new House();
    		house.Address = "New Street";
    		house.Description = "Large family home.";
    		house.Owner = "Mr. Smith.";
    		house.Print();

    		TrainStation trainStation = new TrainStation();
    		trainStation.Director = "Mr. Kovacs";
    		trainStation.Location = "Budapest";
    		trainStation.NumberOfPassangers = 100000;
    		trainStation.NumberOfTrains = 100;
    		trainStation.Print();

    		Console.ReadKey();
    	}
    }


Run the app to see the output. Up to now this is extremely basic I guess. All domain objects have a different set of properties and structure. Also, there are a couple of similarities. They are definitely all buildings and they can be printed. The first step towards an abstraction – and so the bridge pattern – is to introduce an interface in the domain:



    public interface IBuilding
    {
    	void Print();
    }


Make the 3 domain objects implement the interface. In Main we can then have a list of IBuilding objects and call their Print method in a loop:



    static void Main(string[] args)
    {
    	List<IBuilding> buildings = new List<IBuilding>();
    	Apartment apartment = new Apartment();
    	apartment.Description = "Nice apartment.";
    	apartment.Rooms.Add("1/A", "Cozy little room");
    	apartment.Rooms.Add("2/C", "To be renovated");
    	buildings.Add(apartment);

    	House house = new House();
    	house.Address = "New Street";
    	house.Description = "Large family home.";
    	house.Owner = "Mr. Smith.";
    	buildings.Add(house);

    	TrainStation trainStation = new TrainStation();
    	trainStation.Director = "Mr. Kovacs";
    	trainStation.Location = "Budapest";
    	trainStation.NumberOfPassangers = 100000;
    	trainStation.NumberOfTrains = 100;
    	buildings.Add(trainStation);

    	foreach (IBuilding building in buildings)
    	{
    		building.Print();
    	}

    	Console.ReadKey();
    }


Run the app and you should see the same output as before.

If the requirements stop here then we don't need the bridge pattern. We only introduced a bit of abstraction to generalise our domain model. However, note that this is the first step towards the pattern: we now have the first level of abstraction and we'll need a second if the requirements change.

As requirements do change in practice this is what we'll simulate: the customer wants to be able to print out the objects in different styles and formats. A possible solution is to create a different type of House, or even derive from it, call it FormattedHouse and override the Print method according to the new requirements – need to make the Print method of House overridable. Make the same changes to the other 2 objects and then we end up with 6 domain objects. As we add new types of building we always have to create 2: one with a normal printout and one with the fancy style. And when the customer asks for yet another type of printout then we'll have 9 domain objects where our domain should really only have 3.

We could prevent this with some kind of parameter, like "bool fancyPrint" leading to an if-else statement within the Print method. Then as we introduce new formatting types we may switch to an enumeration instead leading to an even longer if-else statement and the need to extend each and every Print() implementation as the enumeration grows. This is not good software engineering practice.

Instead we need another level of abstraction that takes care of the formatting of the printout. We still want to print the attributes of the domain objects but we want a formatter to take care of that. In addition as our objects have different structures we want to preserve the liberty of printing the specific properties of the domain objects. We don't want to force the House object to have a NumberOfTrains property just to make the domains universal.

Let's start out with the following formatter interface:



    public interface IFormatter
    {
        string Format(string key, string value);
    }


This is quite simple and general: the key will be the formatting style, such as "Description: ", and the value is the value to be printed, such as House.Description.

We'll need to inject this formatter to the Print method of each domain object, so update the IBuilding interface as follows:



    public interface IBuilding
    {
    	void Print(IFormatter formatter);
    }


Update each Print implementation to accept the IFormatter parameter, example:



    public void Print(IFormatter formatter)


A short side note: if you want to force your domain objects to have a formatter then you will need to create an abstract base class that each domain object inherit from, such as BuildingBase. The constructor of the base class will accept an IFormatter so that you'll need to update the constructor of each domain object to accept an IFormatter. However, here I'll just stick to the interface solution.

Let's update the Print method in each of the domain objects to use the formatter:



    public class House : IBuilding
    {
    	public string Description { get; set; }
    	public string Owner { get; set; }
    	public string Address { get; set; }

    	public void Print(IFormatter formatter)
    	{
    		Console.WriteLine(formatter.Format("Description: ", Description));
    		Console.WriteLine(formatter.Format("Owner: ", Owner));
    		Console.WriteLine(formatter.Format("Address: ", Address));
    	}
    }




    public class TrainStation : IBuilding
    {
    	public int NumberOfTrains { get; set; }
    	public string Location { get; set; }
    	public int NumberOfPassangers { get; set; }
    	public string Director { get; set; }

    	public void Print(IFormatter formatter)
    	{
    		Console.WriteLine(formatter.Format("Number of trains: ", NumberOfTrains.ToString()));
    		Console.WriteLine(formatter.Format("Location: ", Location));
    		Console.WriteLine(formatter.Format("Number of passangers:", NumberOfPassangers.ToString()));
    		Console.WriteLine(formatter.Format("Director: ", Director));
    	}
    }




    public class Apartment : IBuilding
    {
    	public string Description { get; set; }
    	public Dictionary<string, string> Rooms { get; set; }

    	public Apartment()
    	{
    		Rooms = new Dictionary<string, string>();
    	}

    	public void Print(IFormatter formatter)
    	{
    		Console.WriteLine(formatter.Format("Description: ", Description));
    		foreach (KeyValuePair<string, string> room in Rooms)
    		{
    			Console.WriteLine(string.Concat(formatter.Format("Room: ", room.Key), formatter.Format(", room description: ", room.Value)));
    		}
    	}
    }


Now we're ready for some implementations of the IFormatter interface. The most obvious choice is a standard formatter that prints the properties as in our previous solution:



    class StandardFormatter : IFormatter
        {
            public string Format(string key, string value)
            {
                return string.Format("{0}: {1}", key, value);
            }
        }


The updated Main method looks like follows:



    static void Main(string[] args)
    {
    	IFormatter formatter = new StandardFormatter();
    	List<IBuilding> buildings = new List<IBuilding>();
    	Apartment apartment = new Apartment();
    	apartment.Description = "Nice apartment.";
    	apartment.Rooms.Add("1/A", "Cozy little room");
    	apartment.Rooms.Add("2/C", "To be renovated");
    	buildings.Add(apartment);

    	House house = new House();
    	house.Address = "New Street";
    	house.Description = "Large family home.";
    	house.Owner = "Mr. Smith.";
    	buildings.Add(house);

    	TrainStation trainStation = new TrainStation();
    	trainStation.Director = "Mr. Kovacs";
    	trainStation.Location = "Budapest";
    	trainStation.NumberOfPassangers = 100000;
    	trainStation.NumberOfTrains = 100;
    	buildings.Add(trainStation);

    	foreach (IBuilding building in buildings)
    	{
    		building.Print(formatter);
    	}
    	Console.ReadKey();
    }


Run the app and you'll see that the properties of each object are printed out in the console.

Now if the customer wants new types of format then you can just say "ah, OK, give me five minutes!". 5 minutes of programming can give you a fancy formatter such as this:



    public class FancyFormatter : IFormatter
        {
            public string Format(string key, string value)
            {
                return string.Format("-=! {0} ----- !=- {1}", key, value);
            }
        }


Then just change the type of the formatter in the Main method to test the output. There you are, you have a new formatter without changing any of the domain objects. We change the formatting behaviour of each object without having to add extra parameters and inhering from existing classes to override the Print method. It's now a breeze to add new types of formatters. The formatting behaviour is decoupled from the Print() method. We still print out each property of our objects but it is the formatter that takes care of formatting that printout. We can even apply a different formatter to each of our domain objects.

In the language of the pattern we have the following components:

* IBuilding: the **abstraction**
* Building types: the **refined abstractions** that implement the abstraction
* IFormatter: the **implementor** which is the second level abstraction
* Formatter types: the **concrete implementors** which implement the implementor abstraction

We've successfully abstracted away printing format mechanism and introduced a second level of abstraction required by the pattern. In fact we're **bridging** the ability to print and the way the actual printing is carried out. You may not even want to print on the Console window but on a real printer. You can abstract away that logic using this pattern.

Let's add yet another formatter:



    public class UglyFormatter : IFormatter
    {
    	public string Format(string key, string value)
    	{
    		return string.Format("*^*_##@!!!!!!!&/(%&{0}:{1}", key, value);
    	}
    }


Again, change the formatter type in the Main method to see the effect. We can even have a menu list where the customer can pick the format.

Also, our formatters are proper objects, not primitives such as a boolean flag or an enumeration where we inspect the parameter in a long if-else statement.

We could even add other types of implementors, such as language formatters, colour formatters etc. to the Print method using this pattern. We can vary them as we wish: ugly formatting – English – red, fancy formatting – German – yellow.

Common usages of this pattern include:

* Graphics: if you run a Java desktop application on Windows or a Mac then the GUI elements, such as the button, may be draw differently depending on the operating system
* Persistence: database persistence, file persistence, memory persistence etc. where the object may have Persist method with an IPersistence interface taking care of the persistence logic
* .NET providers: profiles and membership have their built-in providers in .NET that you may have used in your code. Those are also based on the bridge pattern: we bridge the ability to authorise a user and the actual way of performing the authorisation

In case you feel the need to build a large class hierarchy where the concrete classes only change some type of logic but are not truly different domain objects then you may have a candidate for the bridge pattern.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
