[Source](http://dotnetcodr.com/2013/09/16/a-model-net-web-service-based-on-domain-driven-design-part-2-ddd-basics/ "Permalink to A model .NET web service based on Domain Driven Design Part 2: DDD basics")

# A model .NET web service based on Domain Driven Design Part 2: DDD basics

**Introduction**

In the [previous][1] post we discussed how a traditional data access based layering can cause grief and sorrow. We'll now move on to a better design and we'll start with – what else – the domain layer. It is the heart of software, as Eric Evans put it, which should be the centre of the application. It should reflect our business entities and rules as close as possible. It will be an ever-evolving layer where you add, modify and remove bits of code as you get to know the domain better and better. The initial model will grow to a deep model with experimenting, brainstorming and knowledge crunching.

Before we dive into any code we have to discuss a couple of important terms. The domain layer is so important that an entire vocabulary has evolved around it. I promise to get to some real C# code after the theory.

**Entities and value objects**

Domain objects can be divided into entities and value objects.

An **entity** is an object with a unique ID. This unique ID is the most important property of an entity: it helps distinguish between two otherwise identical objects. We can have two people with the same name but if their IDs are different then we're talking about two different people. Also, a person can change his/her name, but if the ID is the same then we know we're talking about the same person. An ID is typically some integer or a GUID or some other string value randomly generated based on some properties of the object. Once the entity has been persisted, its ID should never change otherwise we lose track of it. In Sweden every legal person and company receives a personal registration number. Once you are given this ID it cannot be changed. From the point of view of the Swedish authorities this ID is my most important "property". EntityA == EntityB if and only if EntityA.ID == EntityB.ID. Entities should ideally remain very clean POCO objects without any trace of technology-specific code. You may be tempted to add technology specific elements such s MVC attributes in here, e.g. [Required] or [StringLength(80)]. **DON'T DO THAT. EVER!** We'll see later where they belong.

A **value object** on the other hand lacks any sort of ID. E.g. an office building may have 100 windows that look identical. Depending on your structure of the building you may not care which exact window is located where. Just grab whichever and mount it in the correct slot. In this case it's futile to assign an ID to the window, you don't care which one is which. Value objects are often used to group certain properties of an Entity. E.g. a Person entity can have an Address property which includes other properties such as Street and City. In your business an Address may well be a value object. In other cases, such as a postal company business, an Address will almost certainly be an Entity. A value object can absolutely have its own logic. Note that if you save value objects in the database it's reasonable to assume that they will be assigned an automatic integer ID, but again – that's an implementation detail that only concerns how our domain is represented in the storage, the domain layer doesn't care, it will ignore the ID assigned by the data storage mechanism.

Sometimes it's not easy to decide whether a specific object is an entity or a value object. If two objects have identical properties but we still need to distinguish between them then it's most likely an entity with a unique ID. If two objects have identical properties and we don't care which one is which: a value object. If the same object can be shared, e.g. the same Name can be attached to many Person objects: most likely a value object.

**Aggregates and aggregate roots**

Objects are seldom "independent": they have other objects attached to them or are part of a larger object graph themselves. A Person object may have an Address object which in turn may have other Person objects if more than one person is registered there. It can be difficult to handle in software: if we delete a Person, then do we delete the Address as well? Then other Person objects registered there may be "homeless". We can leave the Address in place, but then the DB may be full of empty, redundant addresses. Objects may therefore have a whole web of interdependency. Where does an object start and end? Where are its boundaries? There's clearly a need to organise the domain objects into logical groups, so-called aggregates. An aggregate is a group of associated objects that are treated as a unit for the purpose of data changes. Every aggregate has a root and clear boundaries. The root is the only member of an aggregate that outside objects are allowed to hold references to.

_Example_

Take a Car object: it has a unique ID, e.g. the registration number to distinguish it from other cars. Each tire must also be measured somehow, but it's unlikely that they will be treated separately from the car they belong to. It's also unlikely that we make a DB search for a tire to see which car it belongs to. Therefore the Car is an aggregate root whose boundary includes the tires. Therefore consumers of the Car object should not be able to directly change the tire settings of the car. You may go as far as hiding the tires entirely within the Car aggregate and only let outside callers "get in touch with them" through methods such as "MoveCarForward". The internal logic of the Car object may well modify the tires' position, but to the outside world that is not visible.

An engine will probably also have a serial number and may be tracked independently of the car. It may happen that the engine is the root of its own aggregate and doesn't lie within the boundaries of the Car, it depends on the domain model of the application.

![Car aggregate root][2]

An important term in conjunction with Aggregates is **invariants**. Invariants are consistency rules that must be maintained whenever data changes, so they represent business rules.

**General guidelines for constructing the domain layer**

Make sure that you allocate the best and most experienced developers available at your company to domain-related tasks. Often experienced programmers spend their time on exploring and testing new technologies, such as WebSockets. Entry-level programmers are then entrusted to construct the domain layer as technologically it's not too difficult: write abstract and concrete classes, interfaces, methods, properties, that's it really. Every developer should be able to construct these elements after completing a beginners' course in C#. So people tend to think that this is an entry-level task. However, domain modelling and writing code in a loosely coupled and testable manner are no trivial tasks to accomplish.

Domains are the most important objects in Domain Driven Design. They are the objects that describe our business. They are so central that you should do your utmost to keep them clean, well organised and testable. It is worth building a separate Domain layer even during the mockup phase of a new project, even if you ignore the other layers. If you need to allocate time to maintain each layer within your solution then this layer should be allocated the most. This is important to keep in mind because you know how it usually goes with test projects – if management thinks it's good then you often need to continue to build upon your initial design. There's no time to build a better application because suddenly you are given a tight deadline. Then you're stuck with a domain-less solution.

Domain objects should always be the ones that take priority. If you have to change the domain object to make room for some specific technology, then you're taking the wrong direction. It is the technology that must fit the domain objects, not vice versa.

Domain objects are also void of persistence-related code making them technology agnostic or persistence ignorant. A Customer object will not care how it is stored in a database table, that is an irrelevant implementation detail.

Domain objects should be POCO and void of technology-specific attributes, such as the ones available in standard MVC, e.g. [Required]. Domain objects should be clear of all such 'pollution'. Also, all logic specific to a domain must be included in the domain. These business rules contain validation rules as well, such as "the quantity should not be negative" or "a customer must not buy more than 5 units of this product". Resist the temptation to write external files, such as CustomerHelper etc. where you put domain-specific logic, that's not right. A domain is responsible to implement its specific domain logic. Example: in a banking application we may have an Account domain with a rule saying that the balance cannot be negative. You must include this validation logic within Account.cs, not in some other helper class. If the Account class has a complex validation logic then it's fine to factor out the IMPLEMENTATION of the validation. i.e. the method body to a separate helper class. However, having a helper class that totally encapsulates domain validation and other domain-specific logic is the incorrect approach. Consumers of the domain will not be aware that there is some external class that they need to call. If domain specific logic is placed elsewhere then not even a visual inspection of the class file will reveal the domain logic to the consumer.

Make sure to include logic specific to a single domain object. Example: say you have an object called Observation with two properties of type double: min and max. If you want to calculate the average then Observation.cs will have a method called CalculateAverage and return the average of min and max, i.e. it performs logic on the level of a single domain object.

In the next post we'll look at how these concepts can be written in code.

View the list of posts on Architecture and Patterns [here][3].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2013/09/12/a-model-net-web-service-based-on-domain-driven-design-part-1-introduction/ "A model .NET web service based on Domain Driven Design Part 1: introduction"
[2]: http://dotnetcodr.files.wordpress.com/2013/08/aggregaterootstructure.png?w=630&h=327
[3]: http://dotnetcodr.com/architecture-and-patterns/ "Architecture and patterns"
