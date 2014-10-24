[Source](http://dotnetcodr.com/2013/12/30/a-model-soa-application-in-net-part-4-the-service-layer-continued/ "Permalink to A model SOA application in .NET Part 5: the service layer continued")

# A model SOA application in .NET Part 5: the service layer continued

**Introduction**

In the previous post we started implementing the IProductService interface. We got as far as declaring a couple of private fields and a constructor. Here we'll implement the ReserveProduct and PurchaseProduct methods.

**The concrete service continued**

Open the project we've been working on in this series and locate ProductService.cs. The implemented ReserveProduct method looks as follows:



    public ProductReservationResponse ReserveProduct(ReserveProductRequest productReservationRequest)
    {
    	ProductReservationResponse reserveProductResponse = new ProductReservationResponse();
    	try
    	{
    		Product product = _productRepository.FindBy(Guid.Parse(productReservationRequest.ProductId));
    		if (product != null)
    		{
    			ProductReservation productReservation = null;
    			if (product.CanReserveProduct(productReservationRequest.ProductQuantity))
    			{
    				productReservation = product.Reserve(productReservationRequest.ProductQuantity);
    				_productRepository.Save(product);
    				reserveProductResponse.ProductId = productReservation.Product.ID.ToString();
    				reserveProductResponse.Expiration = productReservation.Expiry;
    				reserveProductResponse.ProductName = productReservation.Product.Name;
    				reserveProductResponse.ProductQuantity = productReservation.Quantity;
    				reserveProductResponse.ReservationId = productReservation.Id.ToString();
    			}
    			else
    			{
    				int availableAllocation = product.Available();
    				reserveProductResponse.Exception = new LimitedAvailabilityException(string.Concat(&quot;There are only &quot;, availableAllocation, &quot; pieces of this product left.&quot;));
    			}
    		}
    		else
    		{
    			throw new ResourceNotFoundException(string.Concat(&quot;No product with id &quot;, productReservationRequest.ProductId, &quot;, was found.&quot;));
    		}
    	}
    	catch (Exception ex)
    	{
    		reserveProductResponse.Exception = ex;
    	}
    	return reserveProductResponse;
    }


First we let the product repository locate the requested product for us. If it doesn't exist then we throw an exception with an appropriate message. We check using the CanReserveProduct method whether there are enough products left. If not then we let the client know in an exception message. Otherwise we reserve the product, save the current reservation status and populate the reservation response using the product reservation returned by the Save method. We wrap the entire code in a try-catch block to make sure that we catch any exception thrown during the process.

Here's the implemented PurchaseProduct method:



    public PurchaseProductResponse PurchaseProduct(PurchaseProductRequest productPurchaseRequest)
    {
    	PurchaseProductResponse purchaseProductResponse = new PurchaseProductResponse();
    	try
    	{
    		if (_messageRepository.IsUniqueRequest(productPurchaseRequest.CorrelationId))
    		{
    			Product product = _productRepository.FindBy(Guid.Parse(productPurchaseRequest.ProductId));
    			if (product != null)
    			{
    				ProductPurchase productPurchase = null;
    				if (product.ReservationIdValid(Guid.Parse(productPurchaseRequest.ReservationId)))
    				{
    					productPurchase = product.ConfirmPurchaseWith(Guid.Parse(productPurchaseRequest.ReservationId));
    					_productRepository.Save(product);
    					purchaseProductResponse.ProductId = productPurchase.Product.ID.ToString();
    					purchaseProductResponse.PurchaseId = productPurchase.Id.ToString();
    					purchaseProductResponse.ProductQuantity = productPurchase.ProductQuantity;
    					purchaseProductResponse.ProductName = productPurchase.Product.Name;
    				}
    				else
    				{
    					throw new ResourceNotFoundException(string.Concat(&quot;Invalid or expired reservation id: &quot;, productPurchaseRequest.ReservationId));
    				}
    				_messageRepository.SaveResponse&lt;PurchaseProductResponse&gt;(productPurchaseRequest.CorrelationId, purchaseProductResponse);
    			}
    			else
    			{
    				throw new ResourceNotFoundException(string.Concat(&quot;No product with id &quot;, productPurchaseRequest.ProductId, &quot;, was found.&quot;));
    			}
    		}
    		else
    		{
    			purchaseProductResponse = _messageRepository.RetrieveResponseFor&lt;PurchaseProductResponse&gt;(productPurchaseRequest.CorrelationId);
    		}
    	}
    	catch (Exception ex)
    	{
    		purchaseProductResponse.Exception = ex;
    	}
    	return purchaseProductResponse;
    }


If you recall from the [first part][1] of this series we talked about the Idempotent pattern. The IsUniqueRequest is an application of the pattern. We check in the message repository if a message with that correlation ID has been processed before. If yes, then we return the response stored in the repository to avoid making the same purchase again. This is not the only possible solution of the pattern, but only one of the viable ones. Probably the domain layer could have a similar logic as well, but I think this is more robust.

If this is a new request then we locate the product just like in the ReserveProduct method and throw an exception if the product is not found. If the product exists then we need to check if the reservation can be made using the reservation ID of the incoming message. If not, then the reservation either doesn't exist or has expired and a corresponding exception is thrown. Otherwise we confirm the purchase, save the product and dress up the product purchase response using the product purchase object returned by the ConfirmPurchaseWith method. Just like above, we wrap the code within a try-catch block.

This completes the service layer of the SOA project. [We'll look at the web client in the next post][2]. The web client will be a web service client based on the Web API technology so that we don't need to waste time and energy on presentation stuff such as HTML and CSS.

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/12/16/a-model-soa-application-in-net-part-1-the-fundamentals/ "A model SOA application in .NET Part 1: the fundamentals"
[2]: http://dotnetcodr.com/2014/01/02/a-model-soa-application-in-net-part-5-the-client-proxy/ "A model SOA application in .NET Part 6: the client proxy"
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
