[Source](http://dotnetcodr.com/2013/12/16/a-model-soa-application-in-net-part-1-the-fundamentals/ "Permalink to A model SOA application in .NET Part 1: the fundamentals")

# A model SOA application in .NET Part 1: the fundamentals

**Introduction**

SOA – Service Oriented Architecture – is an important buzzword in distributed software architecture. This term has been misused a lot to mean just any kind of API that spits out responses to the incoming requests regardless of the rules and patterns common to SOA applications.

We looked at the role of the Service layer in [this][1] post and the [one after that][2]. In short a service represents the connecting tissue between the back-end layers – typically the domain logic, infrastructure and repository – and the ultimate consumer of the application: an MVC controller, a Console app, a Web Forms aspx page etc., so any caller that is interested in the operation your system can provide. To avoid the situation where your callers have to dig around in your backend code to find the operations relevant to them you can put up a service which encapsulates the possible operations to the clients. Such a service – an **application service** – is usually void of any business logic and performs co-ordination tasks.

The goals of this series are the following:

* Provide an understanding of SOA, the rules and patterns associated with it
* Provide a down-to-earth example SOA project in .NET
* Concentrate on the fundamentals of SOA without going into complex architectures such as CQRS

The target audience is developers getting started with SOA.

The sample application will follow a layered structure similar to [this][3] application although in a simplified form. I won't concentrate on the Domain as much as in that project. We'll put most of our focus on service-related issues instead. However, I encourage you to at least skim through that series on Domain Driven Design to get an understanding of layered projects in .NET.

**Rules and patterns**

SOA is quite generic and can be applied in several different ways depending on the starting point of the – often legacy – application you're trying to transform. There are however a couple of rules and practices to keep in mind when building a SOA app.

_The 4 tenets of SOA_

* **Boundaries are explicit**: a service interface needs to be clean and simple. Note the term 'interface': it is what the clients see from the outside that must be clear and concise. Your code in the background can be as complex as you wish, but your clients should not be aware of it.
* **Services are autonomous**: service methods should be independent. The caller should not have to call certain methods in some order to achieve a goal. You should strive to let the client get the relevant response in one atomic operation. Service methods should be stateless: the client calls a service and "that's the end of the story", meaning there shouldn't be any part of the system left in a partially done state.
* **Interoperability**: a service should only expose an interface, not an entire implementation. Communication should happen with standard message types such as JSON or XML for complex objects or simple inputs like integers and strings for primitive inputs so that clients of very disparate types can reach your services.
* **Policy exposure**: a service interface should be well documented for clients so that they know what operations are supported, how they can be called and what type of response they can expect.

_Facade design pattern_

We discussed the Facade design pattern in [this][4] post. In short it helps to hide a complex backend system in form of a simplified, clear and concise interface. The main goal of this pattern is that your clients shouldn't be concerned with a complex API.

_The RequestResponse messaging pattern_

We discussed the RequestReponse pattern in [this][1] post. In short the main purpose of this pattern is to simplify the communication by encapsulating all the necessary parameters to perform a job in a single object. Make sure to read the referenced post as we'll see this pattern a lot in the model application.

_The Reservation pattern_

As stated above operations of a SOA service should be autonomous. It is not always guaranteed that this is possible though. At times it is necessary to break up a complex unit of operation into 2 or more steps and maintain the state between them. A typical example is e-commerce sites where you can order products. The complete check-out process might look like this:

1. Put items in a shopping cart
2. Provide payment information
3. Provide delivery information
4. Confirm the purchase

It would be difficult to put all these steps into a single interface method, such as ReserveAndBuyGoodsAndConfirmAddress(params). Instead, a reservation ID is provided when the items are reserved in the shopping cart. The reservation ID can be used in all subsequent messages to complete the purchase. Typically an expiration date is attached to the reservation ID so that the purchase must be completed within a specified time range otherwise your reservation is lost. This is very often applied when buying tickets to the concert of a popular band: you must complete the purchase within some minutes otherwise the tickets you've requested will be put up for sale again.

Here's a flow diagram of the pattern:

![Reservation pattern flow][5]

The client, very likely a web interface sends a reservation request to the service. This request includes the product ID and the quantity. The reservation response includes a reservation ID and en expiry date. The reservation must be completed with a purchase until that date. The client then asks for payment and delivery details – not shown in the flow chart, this happens within the client. When all the details are known then the purchase order is sent to the service with the reservation ID. The service checks the validity of the reservation ID and rejects the call if necessary. If it's validated then a confirmation ID is sent by the service.

_The Idempotent pattern_

An operation is idempotent in case it has no additional effects if it is called more than once with the same parameters. You cannot control how your public API is called by your clients so you should make sure it does not lead to any undesired effects if they repeat their calls over and over again.

A common solution is to provide a correlation ID to any potentially state-altering operations. The service checks in some repository whether the request has been processed. If yes then the previously stored response should be provided.

Here's a flow chart showing the communication with a correlation ID:

![Idempotent pattern flow][6]

The client sends a request to the API along with a unique correlation ID. The service checks in its cache whether it has already processed the request. If yes, then it sends the same answer as before other it asks the backend logic to perform some action and then put the response into the cache.

_Duplex_

Most often the communication with a web service is such that you send a HTTP, TCP etc. message to it and receive some type of acknowledgement. This kind of message exchange pattern (MEP) is well represented with the RequestResponse pattern mentioned above. You send off the payload in the query string or the HTTP message body and receive at least an "OK" or "Error" back. The service then "let's you go", i.e. it won't just remember your previous request, you'll need to send a reference to any previous communication, if it's available.

The Duplex MEP on the other hand is a two-way message channel. It is used in the following situations:

* The caller needs to be notified of the result of a long running request when it's complete – a classic "callback" situation. The caller sends a request and instead of waiting for the service to respond it provides a callback address where the service can notify the caller
* The caller wants ad-hoc messages from the service. Say that the service pushes periodic data on stock prices. The caller could provide a channel where the service can send the updates

We discussed the service contracts above. These are the interfaces of the service which the caller can call upon without worrying about the exact implementation details. In a Duplex scenario we have another contract, namely the Callback contract. The caller sends a one-way request – i.e. a request with no immediate response whatsoever – to the service and the service responds back after some time using the callback contract.

WCF has built-in attributes to support Duplex MEP. Here's a simple scenario with an addition service:



    [ServiceContract]
    interface IAdditionHandler
    {
        [OperationContract(IsOneWay = true)]
        void AdditionCompleted(int sum);
    }

    [ServiceContract(CallbackContract = typeof(IAdditionHandler))]
    interface IAdditionService
    {
        [OperationContract(IsOneWay = true)]
        void AddTwoNumbers(int numberOne, int numberTwo);
    }

    [ServiceBehavior(InstanceContextMode = InstanceContextMode.PerSession)]
    class AdditionService : IAdditionService
    {
        public void AddTwoNumbers(int numberOne, int numberTwo)
        {
            IAdditionHandler additionCallback = OperationContext.Current.GetCallbackChannel<IAdditionHandler>();
            additionCallback.AdditionCompleted(numberOne + numberTwo);
        }
    }


There are several difficulties with Duplex channels:

* The service needs a channel to the client which is problematic for security reasons. It might not even be possible to keep the channel open due to firewalls and [Network Address Translation][7] problems.
* Scaling the service becomes difficult due to the long running sessions between the client and the service. SOA scales best with the RequestResponse pattern where each request is independent
* Diminished interoperability: Duplex implemented with WCF cannot be consumed with from other client types such as Java. RequestResponse operates with primitives which are found in virtually every popular platform: Java, PHP, Objective C, you name it

In case the client request triggers a long-running process in the service then do it as follows:

1. Send a Request with the service with the necessary parameters
2. The service starts the long process on a separate thread and responds with a reservation ID or tracking ID etc. as mentioned above
3. The long running process updates the state of the job in some repository periodically: started, ongoing, about to finish, finished, exited with error, etc.
4. The service has an interface where the client can ask about the state of the job using the reservation ID
5. The client periodically checks the status of the job using the reservation ID and when its complete then it requests a result – or the result is sent automatically in the response in case the job is completed

In the next post we'll start building the model application using these patterns – with the exception of the Duplex MEP as it's too complex and rarely used nowadays.

View the list of posts on Architecture and Patterns [here][8].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/10/03/a-model-net-web-service-based-on-domain-driven-design-part-7-the-abstract-service/ "A model .NET web service based on Domain Driven Design Part 7: the abstract Service"
[2]: http://dotnetcodr.com/2013/10/07/a-model-net-web-service-based-on-domain-driven-design-part-8-the-concrete-service/ "A model .NET web service based on Domain Driven Design Part 8: the concrete Service"
[3]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[4]: http://dotnetcodr.com/2013/06/13/design-patterns-and-practices-in-net-the-facade-pattern/ "Design patterns and practices in .NET: the Facade pattern"
[5]: http://dotnetcodr.files.wordpress.com/2013/10/reservationpattern.png?w=630&h=436
[6]: http://dotnetcodr.files.wordpress.com/2013/10/idempotentpattern.png?w=630&h=393
[7]: http://en.wikipedia.org/wiki/Network_address_translation "Network Address Translation on Wikipedia"
[8]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
