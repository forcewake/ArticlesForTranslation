[Source](http://dotnetcodr.com/2014/03/17/extension-to-the-ddd-skeleton-project-domain-specific-rules-part-1/ "Permalink to Extension to the DDD skeleton project: domain specific rules Part 1")

# Extension to the DDD skeleton project: domain specific rules Part 1

**Introduction**

This post is a direct continuation of the [series of posts on Domain Driven Design][1]. Johannes asked in [this][2] post how to include rules in the domain that may vary from instance to instance. The idea is that the Validate() method may need to check different types of rules for a domain. I provided a simplified answer with code examples in my answer but I'd like to formalise the solution within the DDD skeleton project.

The scenario is the following:

* We have an international e-commerce site where customers can order goods
* The data required from a customer on the order form can vary depending on the country they are ordering from
* We need to make sure that the country specific rules are enforced when a customer is created

There are potentially many countries the customers can order from. Let's see some possible solutions that can come to one's mind:

* Extend the abstract Validate method in the EntityBase class so that it accepts some context parameter where we can send in some country identifier. The implemented method can check the context parameter and validate the Customer object in a series of if-else statements: if (country == "GER") then. etc. That would result in a potentially very long if-else statement if we have a large number of countries on our commercial map. That is clearly not maintainable and does not properly reflect the importance of the rule. We would end up with a bloated and hard-to-test Customer class. Also, all other domain objects would have to implement the overloaded Validate method and send in some parameter that may not even be used. Therefore we can quickly rule out this option.
* Make Customer abstract and create country specific concrete classes, such as GermanCustomer, FrenchCustomer etc. That would result in a very messy class structure and the importance of the country rule would still be hidden.
* Continue with the current Customer class and let some external service do an extra validation on its behalf. One of the lessons we've learned from the DDD project is that each domain object should contain its own logic and should be able to validate itself. Domain logic should not be spread out in the solution, especially not in assemblies outside the domain

After dismissing the above proposals we come to the conclusion that we need a more object oriented solution and we need to elevate the country specific rule to an "objectified" form.

**New components in the Domain layer**

We'll introduce a new domain: Country. This object won't need an ID: we don't need to differentiate between two Country objects with the same country code and country name. Therefore it will be a value object. Add the following class to the ValueObjects folder of the Domain layer:



    public abstract class Country : ValueObjectBase
    {
    	public abstract string CountryCode { get; }
    	public abstract string CountryName { get; }

    	protected override void Validate()
    	{
    		if (string.IsNullOrEmpty(CountryCode)) AddBrokenRule(ValueObjectBusinessRule.CountryCodeRequired);
    		if (string.IsNullOrEmpty(CountryName)) AddBrokenRule(ValueObjectBusinessRule.CountryNameRequired);
    	}
    }


The 2 new business rules in the ValueObjectBusinessRule container are as follows:



    public static readonly BusinessRule CountryCodeRequired = new BusinessRule("Country must have a country code");
    public static readonly BusinessRule CountryNameRequired = new BusinessRule("Country must have a name");


We won't be adding new countries through some service call so the Validate method implementation is not strictly required. However it's good to have the Validate method ready for the future.

Create 3 specific countries:



    public class Germany : Country
    {
    	public override string CountryCode
    	{
    		get { return CountryCodes.Germany; }
    	}

    	public override string CountryName
    	{
    		get { return "Germany"; }
    	}
    }

    public class Hungary : Country
    {
    	public override string CountryCode
    	{
    		get { return CountryCodes.Hungary; }
    	}

    	public override string CountryName
    	{
    		get { return "Hungary"; }
    	}
    }

    public class Sweden : Country
    {
    	public override string CountryCode
    	{
    		get { return CountryCodes.Sweden; }
    	}

            public override string CountryName
    	{
    		get { return "Sweden"; }
    	}
    }


The country codes are maintained in a separate container class:



    public class CountryCodes
    {
    	public readonly static string Germany = "GER";
    	public readonly static string Hungary = "HUN";
    	public readonly static string Sweden = "SWE";
    }


In reality the codes will probably be maintained and retrieved from a data store, but this quick solution will do for the time being.

We'll let a [factory][3] return the specific country implementations:



    public class CountryFactory
    {
    	private static IEnumerable<Country> AllCountries()
    	{
    		return new List<Country>() { new Hungary(), new Germany(), new Sweden() };
    	}

    	public static Country Create(string countryCode)
    	{
    		return (from c in AllCountries() where c.CountryCode.ToLower() == countryCode.ToLower() select c).FirstOrDefault();
    	}
    }


Let's say we have the following country-related Customer rules:

* All customers must have a first name
* Swedish customers must be over 18
* Hungarian customers must have a nickname
* German customers must have an email address

The country specific rules will derive from the following base class:



    public abstract class CountrySpecificCustomerRule
    {
    	public abstract Country Country { get; }
    	public abstract List<BusinessRule> GetBrokenRules(CountrySpecificCustomer customer);
    }


I've decided not to touch the Customer object that we created earlier in this series. We can have it for reference. Instead we have a new domain object: CountrySpecificCustomer. We'll add that new class in a second. Before that let's create the country specific implementations of the abstract rule:



    public class GermanCustomerRule : CountrySpecificCustomerRule
    {
    	public override Country Country
    	{
    		get { return CountryFactory.Create(CountryCodes.Germany); }
    	}

    	public override List<BusinessRule> GetBrokenRules(CountrySpecificCustomer customer)
    	{
    		List<BusinessRule> brokenRules = new List<BusinessRule>();
    		if (string.IsNullOrEmpty(customer.Email))
    		{
    			brokenRules.Add(new BusinessRule("German customers must have an email"));
    		}
    		return brokenRules;
    	}
    }

    public class HungarianCustomerRule : CountrySpecificCustomerRule
    {
    	public override Country Country
    	{
    		get { return CountryFactory.Create(CountryCodes.Hungary); }
    	}

    	public override List<BusinessRule> GetBrokenRules(CountrySpecificCustomer customer)
    	{
    		List<BusinessRule> brokenRules = new List<BusinessRule>();
    		if (string.IsNullOrEmpty(customer.NickName))
    		{
    			brokenRules.Add(new BusinessRule("Hungarian customers must have a nickname"));
    		}
    		return brokenRules;
    	}
    }

    public class SwedishCustomerRule : CountrySpecificCustomerRule
    {
    	public override Country Country
    	{
    		get { return CountryFactory.Create(CountryCodes.Sweden); }
    	}

    	public override List<BusinessRule> GetBrokenRules(CountrySpecificCustomer customer)
    	{
    		List<BusinessRule> brokenRules = new List<BusinessRule>();
    		if (customer.Age < 18)
    		{
    			brokenRules.Add(new BusinessRule("Swedish customers must be at least 18."));
    		}
    		return brokenRules;
    	}
    }


You'll see that the implemented GetBrokenRules() methods reflect the country-specific requirements listed above.

The creation of the correct rule implementation will come from another factory:



    public class CountrySpecificCustomerRuleFactory
    {
    	private static IEnumerable<CountrySpecificCustomerRule> GetAllCountryRules()
    	{
    		List<CountrySpecificCustomerRule> implementingRules = new List<CountrySpecificCustomerRule>()
    		{
    			new HungarianCustomerRule()
    			, new SwedishCustomerRule()
    			, new GermanCustomerRule()
    		};
    		return implementingRules;
    	}

    	public static CountrySpecificCustomerRule Create(Country country)
    	{
    		return (from c in GetAllCountryRules() where c.Country.CountryCode == country.CountryCode select c).FirstOrDefault();
    	}
    }


We're now ready to insert the new domain:



    public class CountrySpecificCustomer : EntityBase<int>, IAggregateRoot
    {
    	private Country _country;

    	public CountrySpecificCustomer(Country country)
    	{
    		_country = country;
    	}

    	public string FirstName { get; set; }
    	public int Age { get; set; }
    	public string NickName { get; set; }
    	public string Email { get; set; }

    	protected override void Validate()
    	{
    		//overall rule
    		if (string.IsNullOrEmpty(FirstName))
    		{
    			AddBrokenRule(new BusinessRule("All customers must have a first name"));
    		}
    		List<BusinessRule> brokenRules = new List<BusinessRule>();
    		brokenRules.AddRange(CountrySpecificCustomerRuleFactory.Create(_country).GetBrokenRules(this));
    		foreach (BusinessRule brokenRule in brokenRules)
    		{
    			AddBrokenRule(brokenRule);
    		}
    	}
    }


We'll also need an abstract repository:



    public interface ICountrySpecificCustomerRepository : IRepository<CountrySpecificCustomer, int>
    {
    }


This completes the extension to the Domain layer.

**New elements in the Repository.Memory layer**

We'll add a couple of stub implementations and objects related to the ICountrySpecificCustomerRepository interface. The goal is to demonstrate how the specific rules can be enforced across different CountrySpecificCustomer instances. We'll not waste time around building a dummy repository around CountrySpecificCustomer like we did with the Customer object in the series. The implementations would be very similar anyway.

Enter the following empty database representation of the CountrySpecificCustomer object in the Database folder of the Repository layer:



    public class DatabaseCountrySpecificCustomer
    {
    	//leave it empty
    }


Enter the following stub implementation of the ICountrySpecificCustomerRepository interface:



    public class CountrySpecificCustomerRepository : Repository<CountrySpecificCustomer, int, DatabaseCountrySpecificCustomer>
    	, ICountrySpecificCustomerRepository
    {
    	public CountrySpecificCustomerRepository(IUnitOfWork unitOfWork, IObjectContextFactory objectContextFactory)
    		: base(unitOfWork, objectContextFactory)
    	{}

    	public override CountrySpecificCustomer FindBy(int id)
    	{
    		return null;
    	}

    	public override DatabaseCountrySpecificCustomer ConvertToDatabaseType(CountrySpecificCustomer domainType)
    	{
    		return null;
    	}

    	public IEnumerable<CountrySpecificCustomer> FindAll()
    	{
    		return null;
    	}
    }


Again, we don't care about the implementation of the Find methods. You can check the CustomerRepository for reference if you'd like to practice.

We'll continue with the Service and Web layers and the tests in the [next post][4].

View the list of posts on Architecture and Patterns [here][5].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[2]: http://dotnetcodr.com/2013/09/19/a-model-net-web-service-based-on-domain-driven-design-part-3-the-domain/ "A model .NET web service based on Domain Driven Design Part 3: the Domain"
[3]: http://dotnetcodr.com/2013/05/02/design-patterns-and-practices-in-net-the-factory-patterns-concrete-static-abstract/ "Design patterns and practices in .NET: the Factory Patterns – concrete, static, abstract"
[4]: http://dotnetcodr.com/2014/03/20/extension-to-the-ddd-skeleton-project-domain-specific-rules-part-2/ "Extension to the DDD skeleton project: domain specific rules Part 2"
[5]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
