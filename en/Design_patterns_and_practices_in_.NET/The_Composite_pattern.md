[Source](http://dotnetcodr.com/2013/05/30/design-patterns-and-practices-in-net-the-composite-pattern/ "Permalink to Design patterns and practices in .NET: the Composite pattern")

# Design patterns and practices in .NET: the Composite pattern

**Introduction**

The Composite pattern deals with putting individual objects together to form a whole. In mathematics the relationship between the objects and the composite object they build can be described by a part-whole hierarchy. The ingredient objects are the parts and the composite is the whole.

In essence we build up a tree – a composite – that consists of one or more children – the leaves. The client calling upon the composite should be able to treat the individual parts of the whole in a uniform way.

A real life example is sending emails. If you want to send an email to all developers in your organisation one option is that you type in the names of each developer in the 'to' field. This is of course not efficient. Fortunately we can construct recipient groups, such as Developers. If you then also want to send the email to another person outside the Developers group you can simply put their name in the 'to' box along with Developers. We treat both the group and the individual emails in a uniform way. We can insert both groups and individual emails in the 'to' box. We rely on the email engine to take the group apart and send the email to each recipient in that group. We don't really care how it's done – apart from a couple network geeks I guess.

**Demo**

We will first build a demo application that does not use the pattern and then we'll refactor it. We'll simulate a game where play money is split among the players in a group if they manage to kill a monster.

Start up Visual Studio and create a new console application. Insert a new class called Player:



    public class Player
    	{
    		public string Name { get; set; }
    		public int Gold { get; set; }

    		public void Stats()
    		{
    			Console.WriteLine("{0} has {1} coins.", Name, Gold);
    		}
    	}


This is easy to follow I believe. A group of players is represented by the Group class:



    public class Group
    	{
    		public string Name { get; set; }
    		public List<Player> Members { get; set; }

    		public Group()
    		{
    			Members = new List<Player>();
    		}
    	}


The money splitting mechanism is run in the Main method as follows:



    static void Main(string[] args)
    {
    	int goldForKill = 1023;
    	Console.WriteLine("You have killed the Monster and gained {0} coins!", goldForKill);

    	Player andy = new Player { Name = "Andy" };
    	Player jane = new Player { Name = "Jane" };
    	Player eve = new Player { Name = "Eve" };
    	Player ann = new Player { Name = "Ann" };
    	Player edith = new Player { Name = "Edith" };
    	Group developers = new Group { Name = "Developers", Members = { andy, jane, eve } };

    	List<Player> individuals = new List<Player> { ann, edith };
    	List<Group> groups = new List<Group> { developers };

    	double totalToSplitBy = individuals.Count + groups.Count;
    	double amountForEach = goldForKill / totalToSplitBy;
    	int leftOver = goldForKill % totalToSplitBy;

    	foreach (Player individual in individuals)
    	{
    		individual.Gold += amountForEach + leftOver;
    		leftOver = 0;
    		individual.Stats();
    	}

    	foreach (Group group in groups)
    	{
    		double amountForEachGroupMember = amountForEach / group.Members.Count;
    		int leftOverForGroup = amountForEachGroupMember % group.Members.Count;
    		foreach (Player member in group.Members)
    		{
    			member.Gold += amountForEachGroupMember + leftOverForGroup;
    			leftOverForGroup = 0;
    			member.Stats();
    		}
    	}

    	Console.ReadKey();
    }


So our brilliant game starts off where the monster was killed and we're ready to hand out the reward among the players. We have 5 players. Three of them make up a group and the other two make up a list of individual players. We're then ready to split the gold among the participants where the group is counted as one unit i.e. we have 3 elements: the two individual players and the Developer group. Then we go through each individual and give them their share. We do the same to each group as well where we also divide the group's share among the individuals within that group.

Build and run the application and you'll see in the console that the 1023 pieces of gold was divided up. The code works but it's definitely quite messy. Keep in mind that our tree hierarchy is very simple: we can have individuals and groups. Think of a more complicated scenario: within the Developers group we can have subgroups, such as .NET developers, Java developers who are further subdivided into web and desktop developers and even individuals that do not fit into any group. In the code we iterate through the individuals and the groups manually. We also iterate the players in the group. Imagine that we'd have to iterate through the subgroups of the subgroups of the group if we are facing a deeper hierarchy. The foreach loop would keep growing and the splitting logic would become very challenging to maintain.

So let's refactor the code. The composite pattern states that the client should be able to treat the individual part and the whole in a uniform way. Thus the first step is to make the Person and the Group class uniform in some way. As it turns out the logical way to do this is that both classes implement an interface that the client can communicate with. So the client won't deal with groups and individuals but with a uniform object, such as Participant.

Insert an interface called IParticipant:



    public interface IParticipant
    {
    	int Gold { get; set; }
    	void Stats();
    }


Every participant of the game will have some gold and will be able to write out the current statistics regardless of them being individuals or groups. We let Player and Group implement the interface:



    public class Player : IParticipant
    	{
    		public string Name { get; set; }
    		public int Gold { get; set; }

    		public void Stats()
    		{
    			Console.WriteLine("{0} has {1} coins.", Name, Gold);
    		}
    	}


The Player class implements the interface without changes in its body.

The Group class will encapsulate the gold sharing logic we saw in the Main method above:



    public class Group : IParticipant
    	{
    		public string Name { get; set; }
    		public List<IParticipant> Members { get; set; }

    		public Group()
    		{
    			Members = new List<IParticipant>();
    		}

    		public int Gold
    		{
    			get
    			{
    				int totalGold = 0;
    				foreach (IParticipant member in Members)
    				{
    					totalGold += member.Gold;
    				}

    				return totalGold;
    			}
    			set
    			{
    				double eachSplit = value / Members.Count;
    				int leftOver = value % Members.Count;
    				foreach (IParticipant member in Members)
    				{
    					member.Gold += eachSplit + leftOver;
    					leftOver = 0;
    				}
    			}
    		}

    		public void Stats()
    		{
    			foreach (IParticipant member in Members)
    			{
    				member.Stats();
    			}
    		}
    	}


In the Gold property getter we simply loop through the group members and add up their amount of gold. In the setter we split up the total amount of gold among the group members. Note also that Group can have a list of IParticipant objects representing either individual players or subgroups. You can imagine that those subgroups in turn can also have subgroups so the setters and getters will automatically collect the information from the nested members as well. The leftover variable is set to 0 as the first member will be given all the left over, we don't care about such details.

In the Stats method we simply call the statistics of each group member – again group members can be individuals and subgroups. If it's a subgroup then the Stats method of the members of the subgroup will automatically be called.

The modified Main method looks as follows:



    static void Main(string[] args)
    {
    	int goldForKill = 1023;
    	Console.WriteLine("You have killed the Monster and gained {0} coins!", goldForKill);

    	IParticipant andy = new Player { Name = "Andy" };
    	IParticipant jane = new Player { Name = "Jane" };
    	IParticipant eve = new Player { Name = "Eve" };
    	IParticipant ann = new Player { Name = "Ann" };
    	IParticipant edith = new Player { Name = "Edith" };
    	IParticipant oldBob = new Player { Name = "Old Bob" };
    	IParticipant newBob = new Player { Name = "New Bob" };
    	IParticipant bobs = new Group { Members = { oldBob, newBob } };
    	IParticipant developers = new Group { Name = "Developers", Members = { andy, jane, eve, bobs } };

    	IParticipant participants = new Group { Members = { developers, ann, edith } };
    	participants.Gold += goldForKill;
    	participants.Stats();

    	Console.ReadKey();
    }


You can see that the client, i.e. the Main method calls the methods of IParticipant where IParticipant can be an individual, a group or a group within a group. When we set the level gold through the Gold property the gold distribution logic of each concrete type is called which even takes care of sharing the gold among the groups within a group. The participants variable includes all members of the game.

The main advantage of this pattern is that now the tree structure can be as deep as you can imagine and you don't have to change the logic within the Player and Group classes. Also, we contain the differences between a leaf and a group in the Player and Group classes separately. In addition, they can also tested independently.

Build and run the project and you should see the amount of gold split among all participants of the game.

View the list of posts on Architecture and Patterns [here][1].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
