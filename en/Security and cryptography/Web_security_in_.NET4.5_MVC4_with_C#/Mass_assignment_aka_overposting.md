[Source](http://dotnetcodr.com/2013/01/31/web-security-in-mvc4-net4-5-with-c-mass-assignment-aka-overposting/ "Permalink to Web security in MVC4 .NET4.5 with C#: mass assignment aka overposting")

# Web security in MVC4 .NET4.5 with C#: mass assignment aka overposting

This blog post will describe what mass assignment means and how you can protect your MVC4 web site against such an attack. Note that mass assignment is also called overposting.

Before we go into the details let's set up our MVC4 project. Start Visual Studio 2012 and create an MVC4 internet application with .NET4.5 as the underlying framework.

Locate the Model folder and add a class called Customer. Add the following properties to it:



    public class Customer
        {
            public String Name { get; set; }
            public int Age { get; set; }
            public String Address { get; set; }
            public int AccountBalance { get; set; }
        }


Let's say that our business rules say that every new customer starts with an account balance of zero so the AccountBalance property should have its default value of 0 when creating a new customer.

Add a new controller to the Controllers folder called CustomerController. In the Add Controller window select the Empty MVC controller template. The only action you'll see in this controller at first is "Index".

In this controller we'll simulate fetching our customers from the database. We of course do not have a database and it's a very bad idea to read directly from the database in a controller but that's not relevant for our discussion. Add the following method underneath the Index action:



    private List<Customer> GetCustomersFromDb()
            {
                return new List<Customer>()
                {
                    new Customer(){Address ="Chicago",
                        Age = 30, Name = "John", AccountBalance = 1000}
                    , new Customer(){Address = "New York",
                        Age = 35, Name = "Jill", AccountBalance = 500}
                    , new Customer(){Address = "Stockholm",
                        Age = 25, Name = "Andrew", AccountBalance = 800}
                };
            }


This code is quite clear I suppose.

Right-click the Index action and select "Add view" from the context menu. In the Add View windows set the View name to "Index", check the "Create a strongly-typed view" checkbox and select Customer as the Model class as follows:

![Values in Add View to Customer][1]

The corresponding view file Index.cshtml will be created in the Views/Customer folder. Open that file and modify its model declaration to use a List of Customer instead of a single customer as follows:



    @model List<MassAssignment.Models.Customer>


In this view we only want to list our customers in an unordered list:



    @model List<MassAssignment.Models.Customer>

    @{
        ViewBag.Title = "Index";
    }

    <h2>Customers</h2>

    <ul>
        @foreach (var customer in Model)
        {
            <li>Name: @customer.Name</li>
            <li>Address: @customer.Address</li>
            <li>Age: @customer.Age</li>
            <li>Account balance: @customer.AccountBalance</li>
        }
    </ul>


We'd like to have a link to this view so open _Layout.cshtml in the Views/Shared folder and locate the 'nav' tag. The navigation will already have the following three links:



    <li>@Html.ActionLink("Home", "Index", "Home")</li>
    <li>@Html.ActionLink("About", "About", "Home")</li>
    <li>@Html.ActionLink("Contact", "Contact", "Home")</li>


Add one more link to the list as follows:



    <li>@Html.ActionLink("Customers", "Index", "Customer")</li>


Now run the application and click the Customers link on the opening screen. You should be directed to the View showing our customers as follows:

![List of customers][2]

It will probably not win the yearly CSS contest, but this will serve as a good starting point.

The next step will be to build the elements necessary to insert a new Customer.

We need a screen with the necessary fields to add a new Customer and an action that returns that view. Open CustomerController.cs and add the following method:



            [HttpGet]
            public ActionResult Create()
            {
                return View();
            }


This will listen to HTTP Get requests and return a View which we don't have yet. Right click 'Create' and select Add view from the context menu. The View name can be Create. Check the Create a strongly-typed view checkbox and select the Customer class as the underlying model. This will insert the View file Create.cshtml in the Views/Customer folder. At first it's quite empty:



    @model MassAssignment.Models.Customer

    @{
        ViewBag.Title = "Create";
    }

    <h2>Create</h2>


Add the following HTML/Razor code underneath the h2 tag:



    @using (Html.BeginForm())
    {
        <fieldset>
            <legend>Customer</legend>
            <div class="editor-label">
                @Html.LabelFor(model => model.Name)
            </div>
            <div class="editor-field">
                @Html.EditorFor(model => model.Name)
            </div>
            <div class="editor-label">
                @Html.LabelFor(model => model.Address)
            </div>
            <div class="editor-field">
                @Html.EditorFor(model => model.Address)
            </div>
            <div class="editor-label">
                @Html.LabelFor(model => model.Age)
            </div>
            <div class="editor-field">
                @Html.EditorFor(model => model.Age)
            </div>
            <p>
                <input type="submit" value="Create" />
            </p>
        </fieldset>
    }


We're only adding the bare minimum that's necessary for creating a new customer: labels and fields for all Customer properties bar one. Remember, that the business rules say that every new customer must start with an AccountBalance of 0, this property should not be set to any other value at creation time. Also this form lacks all types of data validation, but that's not important for our purposes.

We need a link that leads to this View. Open Index.cshtml within the Customer folder and add the following Razor code below the unordered list:



    @Html.ActionLink("Create new", "Create", "Customer")


Test the application now to see if you can navigate to the Create view. We need to add the code that will receive the user inputs. Go back to the Customer controller where we just added the Create method. The inputs on Create.cshtml will be posted against the Create view so we need a Create method that will listen to POST requests and accepts a Customer object as parameter.

Insert a method into CustomerController.cs that simulates saving the new Customer object in the database:



            private List<Customer> AddCustomerToDb(Customer customer)
            {
                List<Customer> existingCustomers = GetCustomersFromDb();
                existingCustomers.Add(customer);
                return existingCustomers;
            }


We can now add the overloaded Create method listening to POST requests:



            [HttpPost]
            public ActionResult Create(Customer customer)
            {
                List<Customer> updatedCustomers = AddCustomerToDb(customer);
                Session["Customers"] = updatedCustomers;
                return RedirectToAction("Index");
            }


The incoming Customer object will be populated using the values from the form we created on Create.cshtml. We add the customer to our 'database', save the updated list in a Session and redirect to the Index action to see the new list of Customers. Update the Index action to inspect the Session before checking the 'database':



    public ActionResult Index()
            {
                List<Customer> customers = null;
                if (Session["Customers"] != null)
                {
                    customers = Session["Customers"] as List<Customer>;
                }
                else
                {
                    customers = GetCustomersFromDb();
                }
                return View(customers);
            }


Run the application and navigate to the Create customer view. Try to add a new Customer by filling out the Name, Address and Age fields. Remember that there is no validation so make sure you insert acceptable values, such as an integer in the Age field. Upon pressing the Create button you will be redirected to the Index view of the Customer controller and the new Customer will be added to the unordered list as follows:

![Added a new customer][3]

Note that the AccountBalance property was assigned the default value of 0 according to our business rules. So, we are perfectly safe, right? No-one ever could insert a customer with a positive or negative account balance. Well, it's not so simple…

When the Create(Customer customer) action receives the POST request then the MVC binding mechanism tries its best to find all constituents of the Customer object. Run the application and navigate to the Create New Customer view. Check the generated HTML code; the form should look similar to this:



    <form action="/Customer/Create" method="post">    <fieldset>
            <legend>Customer</legend>
            <div class="editor-label">
                <label for="Name">Name</label>
            </div>
            <div class="editor-field">
                <input class="text-box single-line" id="Name" name="Name" type="text" value="" />
            </div>
            <div class="editor-label">
                <label for="Address">Address</label>
            </div>
            <div class="editor-field">
                <input class="text-box single-line" id="Address" name="Address" type="text" value="" />
            </div>
            <div class="editor-label">
                <label for="Age">Age</label>
            </div>
            <div class="editor-field">
                <input class="text-box single-line" data-val="true" data-val-number="The field Age must be a number." data-val-required="The Age field is required." id="Age" name="Age" type="number" value="" />
            </div>
            <p>
                <input type="submit" value="Create" />
            </p>
        </fieldset>
    </form>


Check the 'name' attribute of each input tag. They match the names of the properties of the Customer object. These values will be read by the MVC model binding mechanism to build the Customer object in the Create method. An attacker can easily send a HTTP POST request to our action which includes the property value for AccountBalance. You could achieve this with C# as follows:



    Uri uri = new Uri("http://validuri/Customer/Create");
    HttpRequestMessage message = new HttpRequestMessage(HttpMethod.Post, uri);
    List<KeyValuePair<String, String>> postValues = new List<KeyValuePair<string, string>>();
    postValues.Add(new KeyValuePair<string, string>("name", "hacker"));
    postValues.Add(new KeyValuePair<string, string>("address", "neverland"));
    postValues.Add(new KeyValuePair<string, string>("age", "1000"));
    postValues.Add(new KeyValuePair<string, string>("accountbalance", "5000000"));
    message.Content = new FormUrlEncodedContent(postValues);
    HttpClient httpClient = new HttpClient();
    Task<HttpResponseMessage> responseTask = httpClient.SendAsync(message);
    HttpResponseMessage response = responseTask.Result;


Not too difficult, right? There are all sorts of tools out there which can generate POST requests for you, such as Curl, SoapUI and many more, so you don't even have to write any real code to circumvent the business rule.

This is a common technique to send all sorts of values that you would not expect in your application logic. Attackers will try to populate your models testing for combinations of property names. In a banking application it is quite feasible that a Customer object will have a property name called exactly AccountBalance. After all the developers and domain owners will most likely not give the AccountBalance property some completely unrelated name, such as "Giraffe" and then create a translation table where it says that Giraffe really means AccountBalance.

What can you do to protect your application against overposting?

**Property blacklist**

You can use the Bind attribute to build a black list of property names i.e. the names of properties that should be excluded from the model binding process:



    public ActionResult Create([Bind(Exclude="AccountBalance")] Customer customer)


You can specify a comma-separated list of strings as the Exclude parameter. These properties will be excluded from the model binding process. Even if the key-value pair from the form includes the AccountBalance property it will be ignored.

**Property whitelist**

You can use the Bind attribute to build a white-list of property names as follows:



    public ActionResult Create([Bind(Include="Age,Name,Address")] Customer customer)


This white-list will include all property names that you want to be bound.

**Use the UpdateModel/TryUpdateModel methods**

You can use the overloads of UpdateModel or TryUpdateModel methods within the Create method as follows:



    UpdateModel(customer, "Customer", new string{"Age","Name","Address"});


You can include the names of the properties in the string array that you want to use in the binding process.

**Create a ViewModel**

Probably the most elegant solution is to create an interim object – a viewmodel – that will hold the properties you want to bind upon a HTTP POST request. Insert a class called CustomerViewModel in the Models folder:



    public class CustomerViewModel
        {
            public String Name { get; set; }
            public int Age { get; set; }
            public String Address { get; set; }
        }


Modify the Create method as follows:



            [HttpPost]
            public ActionResult Create(CustomerViewModel customer)
            {
                Customer newCustomer = new Customer()
                {
                    AccountBalance = 0
                    ,Address = customer.Address
                    ,Age = customer.Age
                    ,Name = customer.Name
                };
                List<Customer> updatedCustomers = AddCustomerToDb(newCustomer);
                Session["Customers"] = updatedCustomers;
                return RedirectToAction("Index");
            }


This way you can be sure that you only receive the properties that you expect and nothing else.

You can view the list of posts on Security and Cryptography [here][4].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.files.wordpress.com/2013/01/addviewcustomer.png?w=300&h=300
[2]: http://dotnetcodr.files.wordpress.com/2013/01/customerslist.png?w=250&h=300
[3]: http://dotnetcodr.files.wordpress.com/2013/01/addednewcustomer.png?w=260&h=300
[4]: http://dotnetcodr.com/security-and-cryptography/ "Security and cryptography"
