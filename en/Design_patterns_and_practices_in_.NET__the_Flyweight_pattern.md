[Source](http://dotnetcodr.com/2013/06/10/design-patterns-and-practices-in-net-the-flyweight-pattern/ "Permalink to Design patterns and practices in .NET: the Flyweight pattern")

# Design patterns and practices in .NET: the Flyweight pattern

**Introduction**

The main intent of the Flyweight pattern is to structure objects so that they can be shared among multiple contexts. It is often mistaken for [factories][1] that are responsible for object creation. The structure of the pattern involves a flyweight factory to create the correct implementation of the flyweight interface but they are certainly not the same patterns.

Object oriented programming is considered a blessing by many .NET and Java programmers and by all others that write code in other object oriented languages. However, it has a couple of challenges. Consider a large object domain where you try to model every component as an object. Example: an Excel document has a large amount of cells. Imagine creating a new Cell object when Excel opens – that would create a large amount of identical objects. Another example is a skyscraper with loads of windows. The windows are potentially identical but there may be some variations to them. As soon as you create a skyscraper object your application may need thousands of window objects as well. If you then create a couple more skyscraper objects your application may eat up all the available memory of your machine.

The flyweight pattern can come to the rescue as it helps reduce the storage cost of a large number of objects. It also allows us to share objects across multiple contexts simultaneously.

The pattern lets us achieve these goals by retaining object oriented granularity and flexibility at the same time.

An anti-pattern solution to the skyscraper-window problem would be to build a superobject that incorporates all window types. You may think that the number of objects may be reduced if you create one type of object instead of 2 or more. However, why should that number decrease??? You still have to new up the window objects, right? In addition, such superobjects can quickly become difficult to maintain as you need to accommodate several different types of objects within it and you'll end up with lots of if statements, possibly nested ones.

The ideal solution in this situation is to create shared objects. Why build 1000 window objects if 1 suffices or at least as few as possible?

The key to creating shared objects is to distinguish between the intrinsic and extrinsic state of an object. The shared objects in the pattern are called Flyweights.

Extrinsic state is supplied to the flyweight from the outside as a parameter when some operation is called on it. This state is not stored inside the flyweight. Example: a Window object may have a Draw operation where the object draws itself. The initial implementation of the Window object may have X and Y co-ordinates plus Width and Height. Those states are contextual can be externalised as parameters to the Draw method: Draw(int x, int y, int width, int height).

Intrinsic state on the other hand is stored inside the flyweight. It does not depend on the context hence it is shareable. The Window object may have a Brush object that is used to draw it. The Brush used to draw the window is the same irrespective of the co-ordinates and size of the window. Thus a single brush can be shared across window objects to draw the windows of the same size.

We still need to make sure that the clients do not end up creating their own flyweights. Even if we implement the extrinsic and intrinsic states everyone is free to create their own copies of the object, right? The answer to that challenge is to use a Flyweight factory. This factory creates and manages flyweights. The client will communicate with the factory if it needs a flyweight. The factory will either provide an existing one or create a new one depending on inputs coming from the client. The client doesn't care which.

Also, we can have distinct Window objects that are somehow unique among all window objects. There may only be a handful of those on a skyscraper. These may not be shared and they store all their state. These objects are unshared flyweights.

Note however that if the objects must be identified by an ID then this pattern will not be applicable. In other words if you need to distinguish between the second window from the right on the third floor and the sixth window from the left on the fifth floor then you cannot possibly share the objects. In Domain Driven Design such id-less objects are called Value Objects as opposed to Entities that have a unique ID. Value Objects have no ID so it doesn't make any difference which specific window object you put in which position. If you have such objects in your domain model then they are a good candidate for flyweights.

**Demo**

In the demo we'll demonstrate sharing Window objects. Fire up Visual Studio and create a new blank solution. Insert a class library called Domain. Every Window will need to implement the IWindow interface:



    public interface IWindow
    {
    	void Draw(Graphics g, int x, int y, int width, int height);
    }


You'll need to add a reference to the System.Drawing library. Note that we pass in parameters that you may first introduce as object properties: x, y, width, height. These are the parameters that represent the extrinsic state mentioned before. They are computed and supplied by the consumer of the object. They can even be stored in a database table if the Window objects have pre-set sizes which is very likely.

We have the following concrete window types:



    public class RedWindow : IWindow
    	{
    		public static int ObjectCounter = 0;
    		Brush paintBrush;

    		public RedWindow()
    		{
    			paintBrush = Brushes.Red;
    			ObjectCounter++;
    		}

    		public void Draw(Graphics g, int x, int y, int width, int height)
    		{
    			g.FillRectangle(paintBrush, x, y, width, height);
    		}
    	}




    public class BlueWindow : IWindow
    	{
    		public static int ObjectCounter = 0;

    		Brush paintBrush;

    		public BlueWindow()
    		{
    			paintBrush = Brushes.Blue;
    			ObjectCounter++;
    		}

    		public void Draw(Graphics g, int x, int y, int width, int height)
    		{
    			g.FillRectangle(paintBrush, x, y, width, height);
    		}
    	}


You'll see that we have a static object counter. This will help us verify how many objects were really created by the client. The Brush object represents an intrinsic state as mentioned above. It is stored within the object.

The Window objects are built by the WindowFactory:



    public class WindowFactory
    	{
    		static Dictionary<string, IWindow> windows = new Dictionary<string, IWindow>();

    		public static IWindow GetWindow(string windowType)
    		{
    			switch (windowType)
    			{
    				case "Red":
    					if (!windows.ContainsKey("Red"))
    						windows["Red"] = new RedWindow();
    					return windows["Red"];
    				case "Blue":
    					if (!windows.ContainsKey("Blue"))
    						windows["Blue"] = new BlueWindow();
    					return windows["Blue"];
    				default:
    					break;
    			}
    			return null;
    		}
    	}


The client will contact the factory to get hold of a Window object. It will send in a string parameter which describes the type of the window. You'll note that the factory has a dictionary where it stores the available Window types. This is a tool for the factory to manage the pool of shared tiles. Look at the switch statement: the factory checks if the requested window type is already available in the dictionary using the window type description as the key. If not then it creates a new concrete window and adds it to the dictionary. Finally it returns the correct window object. Note that the factory only creates a new window the first time it is contacted. It returns the existing object on all subsequent requests.

How would a client use this code? Add a new Windows Forms Application called SkyScraper to the solution. Rename Form1 to WindowDemo. Put a label control on the form and name it lblObjectCounter. Put it as far to one of the edges of the form as possible.

We'll use a random number generator to generate the size parameters of the window objects. We will paint 40 windows on the form: 20 red and 20 blue ones. The total number of objects created should however be 2: one blue and one red. The WindowDemo code behind looks as follows:



    public partial class WindowDemo : Form
    	{
    		Random random = new Random();

    		public WindowDemo()
    		{
    			InitializeComponent();
    		}

    		protected override void OnPaint(PaintEventArgs e)
    		{
    			base.OnPaint(e);

    			for (int i = 0; i < 20; i++)
    			{
    				IWindow redWindow = WindowFactory.GetWindow("Red");
    				redWindow.Draw(e.Graphics, GetRandomNumber(),
    					GetRandomNumber(), GetRandomNumber(), GetRandomNumber());
    			}

    			for (int i = 0; i < 20; i++)
    			{
    				IWindow stoneTile = WindowFactory.GetWindow("Blue");
    				stoneTile.Draw(e.Graphics, GetRandomNumber(),
    					GetRandomNumber(), GetRandomNumber(), GetRandomNumber());
    			}

    			this.lblObjectCounter.Text = "Total Objects Created : " +
    				Convert.ToString(RedWindow.ObjectCounter
    				+ BlueWindow.ObjectCounter);
    		}

    		private int GetRandomNumber()
    		{
    			return (int)(random.Next(100));
    		}
    	}


You'll need to add a reference to the Domain project.

We'll paint the Window objects in the overridden OnPaint method. Otherwise the code should be pretty easy to follow. Compile and run the application. You should see red and blue squares painted on the form. The object counter label should say 2 verifying that our flyweight implementation was correct.

Before I close this post try the following bit of code:



    string s1 = "flyweight";
    string s2 = "flyweight";
    bool areEqual = ReferenceEquals(s1, s2);


Can you guess what value areEqual will have? You may think it's false as s1 and s2 are two different objects and strings are reference types. However, .NET maintains a string pool to manage space and replaces the individual strings to a shared instance.

View the list of posts on Architecture and Patterns [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[2]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
