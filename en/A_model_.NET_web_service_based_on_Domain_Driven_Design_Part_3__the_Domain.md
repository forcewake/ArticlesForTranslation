[Source](http://dotnetcodr.com/2013/09/19/a-model-net-web-service-based-on-domain-driven-design-part-3-the-domain/ "Permalink to A model .NET web service based on Domain Driven Design Part 3: the Domain")

# A model .NET web service based on Domain Driven Design Part 3: the Domain

**Introduction**

In the [previous][1] post we laid the theoretical foundation for our domains. Now it's finally time to see some code. In this post we'll concentrate on the Domain layer and we'll also start building the Infrastructure layer.

_Infrastructure?_

Often the word infrastructure is taken to mean the storage mechanism, like an SQL database, or the physical layers of a system, such as the servers. However, in this case we mean something different. The infrastructure layer is a place for all sorts of cross-cutting concerns and objects that can be used in any project. They are not specific to any single project or domain. Examples: logging, file operations, security, caching, helper classes for Date and String operations etc. Putting these in a separate infrastructure layer helps if you want to employ the same logging, caching etc. policy across all your projects. You could put these within the project solution but then when you start your next project then you may need to copy and paste a lot of code.

**Infrastructure**

The Entities we discussed in the previous post will all derive from an abstract EntityBase class. We could put it directly into the domain layer. However, think of this class as the base for all entities across all your DDD projects where you put all common functionality for your domains.

Create a new blank solution in VS and call it DDDSkeletonNET.Portal. Add a new C# class library called DDDSkeletonNET.Infrastructure.Common. Remove Class1 and add a new folder called Domain. In that folder add a new class called EntityBase:



    public abstract class EntityBase<IdType>
    {
    	public IdType Id { get; set; }

    	public override bool Equals(object entity)
    	{
    		return entity != null
    		   && entity is EntityBase<IdType>
    		   && this == (EntityBase<IdType>)entity;
    	}

    	public override int GetHashCode()
    	{
    		return this.Id.GetHashCode();
    	}

    	public static bool operator ==(EntityBase<IdType> entity1, EntityBase<IdType> entity2)
    	{
    		if ((object)entity1 == null && (object)entity2 == null)
    		{
    			return true;
    		}

    		if ((object)entity1 == null || (object)entity2 == null)
    		{
    			return false;
    		}

    		if (entity1.Id.ToString() == entity2.Id.ToString())
    		{
    			return true;
    		}

    		return false;
    	}

    	public static bool operator !=(EntityBase<IdType> entity1, EntityBase<IdType> entity2)
    	{
    		return (!(entity1 == entity2));
    	}
    }


We only have one property at this point, the ID, whose type can be specified through the IdType type parameter. Often this will be an integer or maybe a GUID or even some auto-generated string. The rest of the code takes care of comparison issues so that you can compare two entities with the '==' operator or the Equals method. Recall that entityA == entityB if an only if their IDs are identical, hence the comparison being based on the ID property.

Domains need to validate themselves when insertions or updates are executed so let's add the following abstract method to EntityBase.cs:



    protected abstract void Validate();


We can describe our business rules in many ways but the simplest format is a description in words. Add a class called BusinessRule to the Domain folder:



    public class BusinessRule
    {
    	private string _ruleDescription;

    	public BusinessRule(string ruleDescription)
    	{
    		_ruleDescription = ruleDescription;
    	}

    	public String RuleDescription
    	{
    		get
    		{
    			return _ruleDescription;
    		}
    	}
    }


Coming back to EntityBase.cs we'll store the list of broken business rules in a private variable:



    private List<BusinessRule> _brokenRules = new List<BusinessRule>();


This list represents all the business rules that haven't been adhered to during the object composition: the total price is incorrect, the customer name is empty, the person's age is negative etc., so it's all the things that make the state of the object invalid. We don't want to save objects in an invalid state in the data storage, so we definitely need validation.

Implementing entities will be able to add to this list through the following method:



    protected void AddBrokenRule(BusinessRule businessRule)
    {
    	_brokenRules.Add(businessRule);
    }


External code will collect all broken rules by calling this method:



    public IEnumerable<BusinessRule> GetBrokenRules()
    {
    	_brokenRules.Clear();
    	Validate();
    	return _brokenRules;
    }


We first clear the list so that we don't return any previously stored broken rules. They may have been fixed by then. We then run the Validate method which is implemented in the concrete domain classes. The domain will fill up the list of broken rules in that implementation. The list is then returned. We'll see in a later post on the application service layer how this method can be used from the outside.

We're now ready to implement the first domain object. Add a new C# class library called DDDSkeleton.Portal.Domain to the solution. Let's make this easy for us and create the most basic Customer domain. Add a new folder called Customer and in it a class called Customer which will derive from EntityBase.cs. The Domain project will need to reference the Infrastructure project. Let's say the Customer will have an id of type integer. At first the class will look as follows:



    public class Customer : EntityBase<int>
    {
    	protected override void Validate()
    	{
    		throw new NotImplementedException();
    	}
    }


We know from the domain expert that every Customer will have a name, so we add the following property:



    public string Name { get; set; }


Our first business rule says that the customer name cannot be null or empty. We'll store these rule descriptions in a separate file within the Customer folder:



    public static class CustomerBusinessRule
    {
    	public static readonly BusinessRule CustomerNameRequired = new BusinessRule("A customer must have a name.");
    }


We can now implement the Validate() method in the Customer domain:



    protected override void Validate()
    {
    	if (string.IsNullOrEmpty(Name))
    	{
    		AddBrokenRule(CustomerBusinessRule.CustomerNameRequired);
    	}
    }


Let's now see how value objects can be used in code. The domain expert says that every customer will have an address property. We decide that we don't need to track Addresses the same way as Customers, i.e. we don't need to set an ID on them. We'll need a base class for value objects in the Domain folder of the Infrastructure layer:



    public abstract class ValueObjectBase
    {
    	private List<BusinessRule> _brokenRules = new List<BusinessRule>();

    	public ValueObjectBase()
    	{
    	}

    	protected abstract void Validate();

    	public void ThrowExceptionIfInvalid()
    	{
    		_brokenRules.Clear();
    		Validate();
    		if (_brokenRules.Count() > 0)
    		{
    			StringBuilder issues = new StringBuilder();
    			foreach (BusinessRule businessRule in _brokenRules)
    			{
    				issues.AppendLine(businessRule.RuleDescription);
    			}

    			throw new ValueObjectIsInvalidException(issues.ToString());
    		}
    	}

    	protected void AddBrokenRule(BusinessRule businessRule)
    	{
    		_brokenRules.Add(businessRule);
    	}
    }


…where ValueObjectIsInvalidException looks as follows:



    public class ValueObjectIsInvalidException : Exception
    {
         public ValueObjectIsInvalidException(string message)
                : base(message)
            {}
    }


You'll recognise the Validate and AddBrokenRule methods. Value objects can of course also have business rules that need to be enforced. In the Domain layer add a new folder called ValueObjects. Add a class called Address in that folder:



    public class Address : ValueObjectBase
    {
    	protected override void Validate()
    	{
    		throw new NotImplementedException();
    	}
    }


Add the following properties to the class:



    public string AddressLine1 { get; set; }
    public string AddressLine2 { get; set; }
    public string City { get; set; }
    public string PostalCode { get; set; }


The domain expert says that every Address object must have a valid City property. We can follow the same structure we took above. Add the following class to the ValueObjects folder:



    public static class ValueObjectBusinessRule
    {
    	public static readonly BusinessRule CityInAddressRequired = new BusinessRule("An address must have a city.");
    }


We can now complete the Validate method of the Address object:



    protected override void Validate()
    {
    	if (string.IsNullOrEmpty(City))
    	{
    		AddBrokenRule(ValueObjectBusinessRule.CityInAddressRequired);
    	}
    }


Let's add this new Address property to Customer:



    public Address CustomerAddress { get; set; }


We'll include the value object validation in the Customer validation:



    protected override void Validate()
    {
    	if (string.IsNullOrEmpty(Name))
    	{
    		AddBrokenRule(CustomerBusinessRule.CustomerNameRequired);
    	}
    	CustomerAddress.ThrowExceptionIfInvalid();
    }


As the Address value object is a property of the Customer entity it's practical to include its validation within this Validate method. The Customer object doesn't need to concern itself with the exact rules of an Address object. Customer effectively tells Address to go and validate itself.

We'll stop building our domain layer now. We could add several more domain objects but let's keep things as simple as possible so that you will be able to view the whole system without having to track too many threads. We now have an example of an entity, a value object and a couple of basic validation rules. We have missed how aggregate roots play a role in code but we'll come back to that in the next post. You probably want to see how the repository, service and UI layers can be wired together.

We'll start building the repository layer in the next post.

View the list of posts on Architecture and Patterns [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/16/a-model-net-web-service-based-on-domain-driven-design-part-2-ddd-basics/ "A model .NET web service based on Domain Driven Design Part 2: DDD basics"
[2]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
