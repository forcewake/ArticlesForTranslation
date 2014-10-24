[Source](http://dotnetcodr.com/2014/05/05/messaging-with-rabbitmq-and-net-c-part-3-message-exchange-patterns/ "Permalink to Messaging with RabbitMQ and .NET C# part 3: message exchange patterns")

# Messaging with RabbitMQ and .NET C# part 3: message exchange patterns

**Introduction**

In this part of the series we'll look at 4 basic message exchange patterns (MEP):

* One way
* Worker queues
* Publish/Subscribe
* Remote Procedure Call (RPC)

Most of the posts on RabbitMQ on this blog are based on the work of RabbitMQ guru [Michael Stephenson][1].

For the demos you can start a new Visual Studio solution or re-use the one [we've been working on][2] so that you have all code references in one place.

A general note: we'll write a lot of example code in this post. We'll be concentrating on writing code that works and will not follow any software design principles such as [SOLID][3] or [DRY][4]. That would only slow us down in a large topic like this. Use the link provided to improve the library as you wish.

**One way messaging**

This is the simplest MEP: a message is sent to the broker which is then processed by the receiver.

Open the RabbitMQ management UI at <http://localhost:15672/> and have it ready throughout the demo. Fire up Visual Studio and either open the same solution as before or create a new blank one. Add a C# class library called RabbitMqService. Add the NuGet RabbitMQ package to it as we did in the first part of this series. Add new class called AmqpMessagingService. Add the following private fields:



    private string _hostName = "localhost";
    private string _userName = "guest";
    private string _password = "guest";
    private string _exchangeName = "";
    private string _oneWayMessageQueueName = "OneWayMessageQueue";
    private bool _durable = true;


Add the following method to create a connection to the RabbitMQ server:



    public IConnection GetRabbitMqConnection()
    {
    	ConnectionFactory connectionFactory = new ConnectionFactory();
    	connectionFactory.HostName = _hostName;
    	connectionFactory.UserName = _userName;
    	connectionFactory.Password = _password;

    	return connectionFactory.CreateConnection();
    }


This method will set up the queue we'll use for the one way message demo:



    public void SetUpQueueForOneWayMessageDemo(IModel model)
    {
    	model.QueueDeclare(_oneWayMessageQueueName, _durable, false, false, null);
    }


Next add a new Console application to the solution called OneWayMessageSender. Add the RabbitMQ NuGet package there as well and also add a reference to the RabbitMqService library. Insert the following code to Main and run the Sender app:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.SetUpQueueForOneWayMessageDemo(model);


Check in the RabbitMQ console that the queue called "OneWayMessageQueue" has been set up. Comment out the call to…



    messagingService.SetUpQueueForOneWayMessageDemo(model);


Add the following code to send a single message to the queue in AmqpMessagingService.cs:



    public void SendOneWayMessage(string message, IModel model)
    {
    	IBasicProperties basicProperties = model.CreateBasicProperties();
    	basicProperties.SetPersistent(_durable);
    	byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    	model.BasicPublish(_exchangeName, _oneWayMessageQueueName, basicProperties, messageBytes);
    }


This code should be familiar from the previous part. Add the following method to Program.cs in the Sender application:



    private static void RunOneWayMessageDemo(IModel model, AmqpMessagingService messagingService)
    {
    	Console.WriteLine("Enter your message and press Enter. Quit with 'q'.");
    	while (true)
    	{
    		string message = Console.ReadLine();
    		if (message.ToLower() == "q") break;

    		messagingService.SendOneWayMessage(message, model);
    	}
    }


We send the message entered by the Sender to the appropriate queue.

Add a call to this method in Main:



    RunOneWayMessageDemo(model, messagingService);

    Console.ReadKey();


Create another Console application called OneWayMessageReceiver to the solution. Add the NuGet RabbitMQ package to it. Add a project reference to RabbitMqService. Insert the following code to Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveOneWayMessages(model);


The first three lines of code should be familiar. ReceiveOneWayMessages has the following implementation in AmqpMessagingService:



    public void ReceiveOneWayMessages(IModel model)
    {
    	model.BasicQos(0, 1, false); //basic quality of service
    	QueueingBasicConsumer consumer = new QueueingBasicConsumer(model);
    	model.BasicConsume(_oneWayMessageQueueName, false, consumer);
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = consumer.Queue.Dequeue() as BasicDeliverEventArgs;
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		Console.WriteLine("Message received: {0}", message);
    		model.BasicAck(deliveryArguments.DeliveryTag, false);
    	}
    }


BasicQos means basic quality of service. The parameters mean that we require one message at a time and we don't want to process any additional messages until this one has been processed. You can use these parameters to receive messages in batches.

QueueingBasicConsumer is built into RabbitMQ and is used to consume messages from a specified queue. We use the IModel's BasicConsume method to consume messages and specify the queue name and the consumer. With 'false' we also indicate that we don't want to auto-acknowledge the messages. Then in the loop we constantly pull message from the queue and acknowledge them with BasicAck. The Queue.Dequeue() method will block the thread until a message has been delivered into the queue. We extract the message byte array from the BasicDeliverEventArgs object. The acknowledgement will release the message from the queue and will allow us to receive the next message.

Let's see if this works. Set the Receiver as the start up project and start the application. The Receiver app will start. Then in VS right-click the Sender application, click Debug, Start new instance. Enter a message in the Sender windows and press Enter. If everything works fine then the message should show up in the Receiver window:

![One way message in console][5]

Send a couple more messages to confirm that the setup works. Set a breakpoint within the while-loop of ReceiveOneWayMessages. You'll see that execution will stop at…



    BasicDeliverEventArgs deliveryArguments = consumer.Queue.Dequeue() as BasicDeliverEventArgs;


…and will only continue if there's a message in the queue. In other words the loop won't just continue asking for new data all the time.

**Worker queues**

In this MEP a message is sent by the sender. There will be many listeners waiting for messages from the same queue. However, those listeners compete to receive the message and only one of them will receive it. The purpose is that if an application is expecting to receive a large load of messages then it can create different threads/processes to process those messages. The benefit is better scalability. For the demo we'll set up a sender and two receivers.

Add the following private field to AmqpMessagingService:



    private string _workerQueueDemoQueueName = "WorkerQueueDemoQueue";


…and the following method to create the queue for this sample:



    public void SetUpQueueForWorkerQueueDemo(IModel model)
    {
    	model.QueueDeclare(_workerQueueDemoQueueName, _durable, false, false, null);
    }


Add a new console application to the solution called WorkerQueueSender. Add the RabbitMQ NuGet package and a reference to the RabbitMqService library. Insert the following code in Main to set up the queue:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.SetUpQueueForWorkerQueueDemo(model);


Set WorkerQueueSender as the startup project and run the application. Check the RabbitMQ UI that the queue has been set up. Comment out the call to SetUpQueueForWorkerQueueDemo in Main.

Add the following method in AmqpMessagingService:



    public void SendMessageToWorkerQueue(string message, IModel model)
    {
    	IBasicProperties basicProperties = model.CreateBasicProperties();
    	basicProperties.SetPersistent(_durable);
    	byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    	model.BasicPublish(_exchangeName, _workerQueueDemoQueueName, basicProperties, messageBytes);
    }


…and the one below to receive messages from the worker queue:



    public void ReceiveWorkerQueueMessages(IModel model)
    {
    	model.BasicQos(0, 1, false); //basic quality of service
    	QueueingBasicConsumer consumer = new QueueingBasicConsumer(model);
    	model.BasicConsume(_workerQueueDemoQueueName, false, consumer);
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = consumer.Queue.Dequeue() as BasicDeliverEventArgs;
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		Console.WriteLine("Message received: {0}", message);
    		model.BasicAck(deliveryArguments.DeliveryTag, false);
    	}
    }


It is identical to ReceiveOneWayMessages except for the queue name.

Back in WorkerQueueSender.Program.cs add the following method and add a call to it from Main:



    private static void RunWorkerQueueMessageDemo(IModel model, AmqpMessagingService messagingService)
    {
    	Console.WriteLine("Enter your message and press Enter. Quit with 'q'.");
    	while (true)
    	{
    		string message = Console.ReadLine();
    		if (message.ToLower() == "q") break;
    		messagingService.SendMessageToWorkerQueue(message, model);
    	}
    }


As you see it is identical to what we had in the previous demo. We'll create two Receivers and they will be identical to the receiver we had in the previous demo. Add two new Console apps: WorkerQueueReceiverOne and WorkerQueueReceiverTwo. In both projects do the following:

* Add RabbitMQ package through NuGet
* Add a library reference to RabbitMqService
* Add the following code to Program.cs.Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveWorkerQueueMessages(model);


Follow these steps to start the demo:

1. Set WorkerQueueSender as the startup project
2. Start the application
3. Right-click WorkerQueueReceiverOne, Debug, Start new instance
4. Do the same for WorkerQueueReceiverTwo

You should have 3 console windows up and running on your screen. Start sending messages in the Sender window. You should see that messages will alternate between receiver one and two:

![Worker queue console][6]

You should never see that the same message is delivered to both receivers.

The Worker Queue MEP can be implemented with very little extra effort compared to the One Way Message MEP. This MEP helps you create a horizontally scalable server where multiple receivers are set up to collect the incoming messages.

**Publish/Subscribe**

In this MEP a message is sent to an exchange and the exchange distributes the message to all queues bound to it. Each queue will have its listener to process the message. If you recall the different exchange types then this sounds like the Fan-out type. We'll set up a dedicated exchange for this, i.e. not use the default one in RabbitMQ.

Enter the following private fields in AmqpMessagingService.cs:



    private string _publishSubscribeExchangeName = "PublishSubscribeExchange";
    private string _publishSubscribeQueueOne = "PublishSubscribeQueueOne";
    private string _publishSubscribeQueueTwo = "PublishSubscribeQueueTwo";


…and the following method where we set up the exchange, 2 queues and bind both queues to the exchange:



    public void SetUpExchangeAndQueuesForDemo(IModel model)
    {
    	model.ExchangeDeclare(_publishSubscribeExchangeName, ExchangeType.Fanout, true);
    	model.QueueDeclare(_publishSubscribeQueueOne, true, false, false, null);
    	model.QueueDeclare(_publishSubscribeQueueTwo, true, false, false, null);
    	model.QueueBind(_publishSubscribeQueueOne, _publishSubscribeExchangeName, "");
    	model.QueueBind(_publishSubscribeQueueTwo, _publishSubscribeExchangeName, "");
    }


Consult the first part in this series if don't recall what these methods do.

Add a new Console project to the solution called PublishSubscribeSender. Perform the usual actions:

* Add RabbitMQ via NuGet
* Add a reference to RabbitMqService

In Main insert the following code to set up the necessary infrastructure:



    public void SendMessageToPublishSubscribeQueues(string message, IModel model)
    {
    	IBasicProperties basicProperties = model.CreateBasicProperties();
    	basicProperties.SetPersistent(_durable);
    	byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    	model.BasicPublish(_publishSubscribeExchangeName, "", basicProperties, messageBytes);
    }


Set PublishSubscribeSender as the startup application and then run it. Check in the RabbitMQ UI whether the exchange and the two queues have been created and if the bindings are OK. Then comment out the call to messagingService.SetUpExchangeAndQueuesForDemo. Add the following method to start sending messages:



    private static void RunPublishSubscribeMessageDemo(IModel model, AmqpMessagingService messagingService)
    {
    	Console.WriteLine("Enter your message and press Enter. Quit with 'q'.");
    	while (true)
    	{
    		string message = Console.ReadLine();
    		if (message.ToLower() == "q") break;

    		messagingService.SendMessageToPublishSubscribeQueues(message, model);
    	}
    }


As you see it's not much different from what we had in the previous demos. SendMessageToPublishSubscribeQueues looks like this in AmqpMessagingService:



    public void SendMessageToPublishSubscribeQueue(string message, IModel model)
    {
    	IBasicProperties basicProperties = model.CreateBasicProperties();
    	basicProperties.SetPersistent(_durable);
    	byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    	model.BasicPublish(_publishSubscribeExchangeName, "", basicProperties, messageBytes);
    }


We're sending the message to the designated exchange with no routing key specified.

Add two new Console applications: PublishSubscribeReceiverOne and PublishSubscribeReceiverTwo. Apply the following to both:

* Add RabbitMQ via NuGet
* Add a reference to RabbitMqService

In PublishSubscribeReceiverOne.Program.cs.Main add the following code:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceivePublishSubscribeMessageReceiverOne(model);


In PublishSubscribeReceiverTwo.Program.cs.Main add the following code:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceivePublishSubscribeMessageReceiverTwo(model);


…where ReceivePublishSubscribeMessageReceiverOne and ReceivePublishSubscribeMessageReceiverTwo look like this in AmqpMessagingService:



    public void ReceivePublishSubscribeMessageReceiverOne(IModel model)
    {
    	model.BasicQos(0, 1, false);
    	Subscription subscription = new Subscription(model, _publishSubscribeQueueOne, false);
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = subscription.Next();
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		Console.WriteLine("Message from queue: {0}", message);
    		subscription.Ack(deliveryArguments);
    	}
    }

    public void ReceivePublishSubscribeMessageReceiverTwo(IModel model)
    {
    	model.BasicQos(0, 1, false);
    	Subscription subscription = new Subscription(model, _publishSubscribeQueueTwo, false);
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = subscription.Next();
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		Console.WriteLine("Message from queue: {0}", message);
    		subscription.Ack(deliveryArguments);
    	}
    }


As you see there's not much difference compared to how the Receiver extracted the messages before. The subscription model is represented by the Subscription object in RabbitMQ .NET. The BasicDeliverEventArgs object is returned by the Next() method of the subscription. We then show the message and acknowledge it.

To run this demo:

1. Run PublishSubscribeSender
2. Start a new instance of PublishSubscribeReceiverOne the way we did above with WorkerQueueReceiverOne
3. Start a new instance of PublishSubscribeReceiverTwo the way we did above with WorkerQueueReceiverTwo
4. You should have three black console screens up and running

Start sending messages on the Sender window. The message should appear on both receivers:

![Publish/message MEP console][7]

The receivers are listening on two different queues hence they are not competing with each other like in the Worker Queue MEP.

**Remote Procedure Call (RPC)**

RPC is slightly different from the above three MEPs in that there's a response queue involved. The sender will first start listening on a response queue before sending any message. It then sends a message to a destination queue via the default exchange where the message includes a property indicating the response queue. The response queue will be dynamically created by the sender. The receiver processes the message and responds using the response queue extracted from the message. The sender then processes the response.

Add the following method to AmqpMessagingService.cs that sets up the queue for this demo:



    public void SetUpQueueForRpcDemo(IModel model)
    {
    	model.QueueDeclare(_rpcQueueName, _durable, false, false, null);
    }


…where _rpcQueueName is a new private field:



    private string _rpcQueueName = "RpcQueue";


Add a new Console app called RpcSender. Add the usual references: RabbitMQ NuGet, RabbitMqService. Insert the following code to Main to set up the queue:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.SetUpQueueForRpcDemo(model);


Set RpcSender as the startup project and run the application. Check in the RabbitMQ management UI that the queue has been set up. Comment out the call to messagingService.SetUpQueueForRpcDemo(model). This queue will be used as the default queue by the sender to send messages. The response queue will be dynamically set up.

Insert the following method to RpcSender.Program.cs to start sending messages:



    private static void RunRpcMessageDemo(IModel model, AmqpMessagingService messagingService)
    {
    	Console.WriteLine("Enter your message and press Enter. Quit with 'q'.");
    	while (true)
    	{
    		string message = Console.ReadLine();
    		if (message.ToLower() == "q") break;
    		String response = messagingService.SendRpcMessageToQueue(message, model, TimeSpan.FromMinutes(1));
    		Console.WriteLine("Response: {0}", response);
    	}
    }


This setup is very similar to what we've seen up to this point. Note, however, that the SendRpcMessageToQueue method returns a string, which will be the response from the Receiver. We also specify a timeout parameter for the response to arrive.

Declare a new method in AmqpMessagingService:



    public string SendRpcMessageToQueue(string message, IModel model, TimeSpan timeout)
    {

    }


The sender in this case will also need to listen to messages so it will need a QueueingBasicConsumer object we saw before. Also, the response queue will be set up dynamically. The QueueDeclare() method without any parameter will create a temporary response queue. The name of the temporary queue will be randomly generated, e.g. "amq.gen-3tj4jtzMauwolYqc7CUj9g". While you're running the demo in a bit you can check the list of queues in the RabbitMQ management UI. The temporary queue will be available as long as the Sender is running. After that it will be removed automatically. Insert the following code to SendRpcMessageToQueue:



    if (string.IsNullOrEmpty(_responseQueue))
    {
    	_responseQueue = model.QueueDeclare().QueueName;
    }

    if (_rpcConsumer == null)
    {
    	_rpcConsumer = new QueueingBasicConsumer(model);
    	model.BasicConsume(_responseQueue, true, _rpcConsumer);
    }


…where _rpcConsumer and _responseQueue are private variables:



    private QueueingBasicConsumer _rpcConsumer;
    private string _responseQueue;


The sender will listen on that temporary response queue. Append the following code to SendRpcMessageToQueue:



    string correlationId = Guid.NewGuid().ToString();

    IBasicProperties basicProperties = model.CreateBasicProperties();
    basicProperties.ReplyTo = _responseQueue;
    basicProperties.CorrelationId = correlationId;

    byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    model.BasicPublish("", _rpcQueueName, basicProperties, messageBytes);

    DateTime timeoutDate = DateTime.UtcNow + timeout;
    while (DateTime.UtcNow <= timeoutDate)
    {
    	BasicDeliverEventArgs deliveryArguments = (BasicDeliverEventArgs)_rpcConsumer.Queue.Dequeue();
    	if (deliveryArguments.BasicProperties != null
    	&& deliveryArguments.BasicProperties.CorrelationId == correlationId)
    	{
    		string response = Encoding.UTF8.GetString(deliveryArguments.Body);
            	return response;
    	}
    }
    throw new TimeoutException("No response before the timeout period.");


We create a message correlation ID to be able to match the sender's message to the response from the receiver. If the receiver is responding to another message then it will be ignored. We then set up the IBasicProperties object and specify the temporary queue name to reply to and the correlation ID. Next we publish the message using BasicPublish like before.

Then we enter something that only receivers have done up to now: listen. The sender will listen for the duration of the timeout date. When a response comes then the correlation IDs must be compared. If there's a match then the response is returned. Otherwise it's ignored. If there's no response before the timeout then an exception is thrown.

Let's look at the receiver now. Add a new Console application called RpcReceiver, add RabbitMQ and RabbitMqService to the reference list. Insert the following code to Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveRpcMessage(model);


…where ReceiveRpcMessage in AmqpMessagingService looks like this in AmqpMessagingService:



    public void ReceiveRpcMessage(IModel model)
    {
    	model.BasicQos(0, 1, false);
    	QueueingBasicConsumer consumer = new QueueingBasicConsumer(model);
    	model.BasicConsume(_rpcQueueName, false, consumer);

    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = consumer.Queue.Dequeue() as BasicDeliverEventArgs;
    		string message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		Console.WriteLine("Message: {0} ; {1}", message, " Enter your response: ");
    		string response = Console.ReadLine();
    		IBasicProperties replyBasicProperties = model.CreateBasicProperties();
    		replyBasicProperties.CorrelationId = deliveryArguments.BasicProperties.CorrelationId;
    		byte[] responseBytes = Encoding.UTF8.GetBytes(response);
    		model.BasicPublish("", deliveryArguments.BasicProperties.ReplyTo, replyBasicProperties, responseBytes);
                    model.BasicAck(deliveryArguments.DeliveryTag, false);
    	}
    }


Most of this code looks familiar by now I hope. We extend the "normal" receiving logic with the ability to send a response. We extract the correlation ID from the sender's message so that our response will have the same ID. We send the response to the ReplyTo queue which was also extracted from the sender's message. We finally acknowledge the reception of the message from the sender.

Let's run this:

1. Make sure that RpcSender is the startup project and run the application
2. Start RpcReceiver the same way as before (Run new instance)
3. You should have 2 console screens up and running

Send a message from the sender to the receiver. Then send a response. It looks like a very primitive chat application:

![RPC console][8]

I hope you agree that it wasn't too difficult to implement these 4 basic message exchange patterns.

Read the next part in this series [here][9].

View the list of posts on Messaging [here][10].

### Like this:

Like Loading...

### _Related_

[1]: http://geekswithblogs.net/michaelstephenson/Default.aspx "Michael Stephenson blog"
[2]: http://dotnetcodr.com/2014/05/01/messaging-with-rabbitmq-and-net-c-part-2-persistence/ "Messaging with RabbitMQ and .NET C# part 2: persistence"
[3]: http://dotnetcodr.com/2013/08/12/solid-design-principles-in-net-the-single-responsibility-principle/ "SOLID design principles in .NET: the Single Responsibility Principle"
[4]: http://dotnetcodr.com/2013/10/17/the-dont-repeat-yourself-dry-design-principle-in-net-part-1/ "The Don't-Repeat-Yourself (DRY) design principle in .NET Part 1"
[5]: http://dotnetcodr.files.wordpress.com/2014/03/onewaymessageinconsole.png?w=630&h=62
[6]: http://dotnetcodr.files.wordpress.com/2014/03/workerqueuemepconsole.png?w=630&h=238
[7]: http://dotnetcodr.files.wordpress.com/2014/03/publishsubscribemepconsole.png?w=630&h=118
[8]: http://dotnetcodr.files.wordpress.com/2014/03/rpcmepconsole.png?w=630&h=70
[9]: http://dotnetcodr.com/2014/05/08/messaging-with-rabbitmq-and-net-c-part-4-routing-and-topics/ "Messaging with RabbitMQ and .NET C# part 4: routing and topics"
[10]: http://dotnetcodr.com/messaging/ "Messaging"
