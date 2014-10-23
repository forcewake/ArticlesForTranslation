[Source](http://dotnetcodr.com/2013/12/19/a-model-soa-application-in-net-part-2-the-domain/ "Permalink to A model SOA application in .NET Part 2: the domain")

# A model SOA application in .NET Part 2: the domain

**Introduction**

The SOA model application will simulate a Product purchase system. The Product must be reserved first and the purchase must be then confirmed. So this is a scenario familiar from e-commerce sites, although in reality the flow may be more complicated. The point here is that the purchase is not a one-step process and the web service must be able to track the registration somehow. So the Product can be anything you can think of: tickets, books, whatever, that's a detail we don't care about.

We'll continue from the [previous][1] post which set the theoretical basis for the model we're about to build.

If you have followed along the [series on DDD][2] then you'll notice that the Domain structure I'm going to present is a lot simpler: there's no aggregate root and entity base etc., and that's for a good reason. Those belong to the DDD-specific discussion whereas here we want to concentrate on SOA. Therefore I decided to remove those elements that might distract us from the main path and keep the Domain layer simple. Otherwise if you are not familiar with DDD but still want to learn about SOA you may find the code hard to follow.

**Demo**

Open Visual Studio and insert a new blank solution called SoaIntroNet. Add a C# class library called SoaIntroNet.Domain to it and remove Class1.cs. Add a folder called ProductDomain and a class called Product in it. Here's the Product domain code with some explanation to follow. The code will not compile at first as we need to implement some other objects as well in a bit.



    public class Product
    {
    	public Product()
    	{
    		ReservedProducts = new List<ProductReservation>();
    		PurchasedProducts = new List<ProductPurchase>();
    	}

    	public Guid ID { get; set; }
    	public string Name { get; set; }
    	public string Description { get; set; }
    	public int Allocation { get; set; }
    	public List<ProductReservation> ReservedProducts { get; set; }
    	public List<ProductPurchase> PurchasedProducts { get; set; }

    	public int Available()
    	{
    		int soldAndReserved = 0;
    		PurchasedProducts.ForEach(p => soldAndReserved += p.ProductQuantity);
    		ReservedProducts.FindAll(p => p.IsActive()).ForEach(p => soldAndReserved += p.Quantity);

    		return Allocation - soldAndReserved;
    	}

    	public bool ReservationIdValid(Guid reservationId)
    	{
    		if (HasReservation(reservationId))
    		{
    			return GetReservationWith(reservationId).IsActive();
    		}
    		return false;
    	}

    	public ProductPurchase ConfirmPurchaseWith(Guid reservationId)
    	{
    		if (!ReservationIdValid(reservationId))
    		{
    			throw new Exception(string.Format("Cannot confirm the purchase with this Id: {0}", reservationId));
    		}

    		ProductReservation reservation = GetReservationWith(reservationId);
    		ProductPurchase purchase = new ProductPurchase(this, reservation.Quantity);
    		reservation.HasBeenConfirmed = true;
    		PurchasedProducts.Add(purchase);
    		return purchase;
    	}

    	public ProductReservation GetReservationWith(Guid reservationId)
    	{
    		if (!HasReservation(reservationId))
    		{
    			throw new Exception(string.Concat("No reservation found with id {0}", reservationId.ToString()));
    		}
    		return (from r in ReservedProducts where r.Id == reservationId select r).FirstOrDefault();
    	}

    	private bool HasReservation(Guid reservationId)
    	{
    		return ReservedProducts.Exists(p => p.Id == reservationId);
    	}

    	public bool CanReserveProduct(int quantity)
    	{
    		return Available() >= quantity;
    	}

    	public ProductReservation Reserve(int quantity)
    	{
    		if (!CanReserveProduct(quantity))
    		{
    			throw new Exception("Can not reserve this many tickets.");
    		}

    		ProductReservation reservation = new ProductReservation(this, 1, quantity);
    		ReservedProducts.Add(reservation);
    		return reservation;
    	}
    }


We maintain a list of purchased and reserved products in the corresponding List objects. The Allocation property means the initial number of products available for sale.

* In Available() we check what's left based on Allocation and the reserved and purchased tickets.
* CanReserveProduct: we check if there are at least as many products available as the 'quantity' value
* Reserve: we reserve a product and add it to the reservation list. We set the expiry date of the reservation to 1 minute. In reality this value should be higher of course but in the demo we don't want to wait 30 minutes to test for the reservation being too old
* HasReservation: check if the reservation exists in the reservation list using the reservation id
* GetReservationWith: we retrieve the reservation based on its GUID
* ReservationIdValid: check if a product can be purchased with the reservation id
* ConfirmPurchaseWith: confirm the reservation by adding the constructed product to the purchase list

Here comes the ProductReservation class:



    public class ProductReservation
    {
    	public ProductReservation(Product product, int expiryInMinutes, int quantity)
    	{
    		if (product == null) throw new ArgumentNullException("Product cannot be null.");
    		if (quantity < 1) throw new ArgumentException("The quantity should be at least 1.");
    		Product = product;
    		Id = Guid.NewGuid();
    		Expiry = DateTime.Now.AddMinutes(expiryInMinutes);
                    Quantity = quantity;
    	}

    	public Guid Id { get; set; }
    	public Product Product { get; set; }
    	public DateTime Expiry { get; set; }
    	public int Quantity { get; set; }
    	public bool HasBeenConfirmed { get; set; }

    	public bool Expired()
    	{
    		return DateTime.Now > Expiry;
    	}

    	public bool IsActive()
    	{
    		return !HasBeenConfirmed && !Expired();
    	}
    }


Note the expiry date property which fits in well with our discussion in the previous post: the client needs to reserve the product first and then confirm the purchase within the expiry date.

The ProductPurchase class is equally straightforward:



    public class ProductPurchase
    {
    	public ProductPurchase(Product product, int quantity)
    	{
    		Id = Guid.NewGuid();
    		if (product == null) throw new ArgumentNullException("Product cannot be null.");
    		if (quantity < 1) throw new ArgumentException("The quantity should be at least 1.");
    		Product = product;
    		ProductQuantity = quantity;
    	}

    	public Guid Id { get; set; }
    	public Product Product { get; set; }
    	public int ProductQuantity { get; set; }
    }


That's all the domain logic we have in the model application. In the next post we'll implement the repository.

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/12/16/a-model-soa-application-in-net-part-1-the-fundamentals/ "A model SOA application in .NET Part 1: the fundamentals"
[2]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
