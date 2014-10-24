[Source](http://dotnetcodr.com/2014/05/12/messaging-with-rabbitmq-and-net-c-part-5-headers-and-scattergather/ "Permalink to Messaging with RabbitMQ and .NET C# part 5: headers and scatter/gather")

# Messaging with RabbitMQ and .NET C# part 5: headers and scatter/gather

**Introduction**

In the [previous post on RabbitMQ .NET][1] we looked at the Routing and Topics exchange patterns. In this post we'll continue looking at RabbitMQ in .NET. In particular we'll talk about routing messages using the following two patterns:

We'll use the demo application we've been working on in this series so have it ready in Visual Studio. Also, log onto the RabbitMQ management console on <http://localhost:15672/>

Most of the posts on RabbitMQ on this blog are based on the work of RabbitMQ guru [Michael Stephenson][2].

**Headers**

The Headers exchange pattern is very similar to Topics we saw in the previous part of this series. The sender sends a message of type Headers to RabbitMQ. The message is routed based on the header value. All queues with a matching key will receive the message. We'll dedicate an exchange to deliver the messages but the routing key will be ignored as it is the headers that will be the basis for the match. We can specify more than one header and a rule that says if all headers must match or just one using the "x-match" property which can have 2 values: "any" or "all". The default value of this property is "all" so all headers must match for a queue to receive a message.

We'll create one dedicated exchange and three queues. Add a new Console app to the solution called HeadersSender. Like before, add references to the RabbitMQ NuGet package and the RabbitMqService library in the solution. Insert the following code to Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.SetUpExchangeAndQueuesForHeadersDemo(model);


…where SetUpExchangeAndQueuesForHeadersDemo in AmqpMessagingService looks like this:



    public void SetUpExchangeAndQueuesForHeadersDemo(IModel model)
    {
    	model.ExchangeDeclare(_headersExchange, ExchangeType.Headers, true);
    	model.QueueDeclare(_headersQueueOne, true, false, false, null);
    	model.QueueDeclare(_headersQueueTwo, true, false, false, null);
    	model.QueueDeclare(_headersQueueThree, true, false, false, null);

    	Dictionary<string,object> bindingOneHeaders = new Dictionary<string,object>();
    	bindingOneHeaders.Add("x-match", "all");
    	bindingOneHeaders.Add("category", "animal");
    	bindingOneHeaders.Add("type", "mammal");
    	model.QueueBind(_headersQueueOne, _headersExchange, "", bindingOneHeaders);

    	Dictionary<string, object> bindingTwoHeaders = new Dictionary<string, object>();
    	bindingTwoHeaders.Add("x-match", "any");
    	bindingTwoHeaders.Add("category", "animal");
    	bindingTwoHeaders.Add("type", "insect");
    	model.QueueBind(_headersQueueTwo, _headersExchange, "", bindingTwoHeaders);

    	Dictionary<string, object> bindingThreeHeaders = new Dictionary<string, object>();
    	bindingThreeHeaders.Add("x-match", "any");
    	bindingThreeHeaders.Add("category", "plant");
    	bindingThreeHeaders.Add("type", "flower");
    	model.QueueBind(_headersQueueThree, _headersExchange, "", bindingThreeHeaders);
    }


The following private fields will be necessary as well:



    private string _headersExchange = "HeadersExchange";
    private string _headersQueueOne = "HeadersQueueOne";
    private string _headersQueueTwo = "HeadersQueueTwo";
    private string _headersQueueThree = "HeadersQueueThree";


We specify the headers in a dictionary. The first dictionary means that the queue will be interested in messages with headers of category = animal and type = mammal. The x-match property of "all" indicates that the queue wants to see both headers. You can probably understand the other two header bindings. As the default value of the x-match header is "all", we could ignore adding that header but I prefer to be explicit in a demo like this.

Set HeadersSender as the start up project and start the application. Check in the RabbitMQ management UI whether the exchange and the queues have been set up correctly. Check the bindings on the exchange as well, you should see the correct header values.

Comment out the call to messagingService.SetUpExchangeAndQueuesForHeadersDemo. Back in AmqpMessageService.cs add the following method to send a message with headers:



    public void SendHeadersMessage(string message, Dictionary<string,object> headers, IModel model)
    {
    	IBasicProperties basicProperties = model.CreateBasicProperties();
    	basicProperties.SetPersistent(_durable);
    	basicProperties.Headers = headers;
    	byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    	model.BasicPublish(_headersExchange, "", basicProperties, messageBytes);
    }


In HeadersSender.cs insert the following private method which reads the header values using delimiters and calls upon the SendHeadersMessage method:



    private static void RunHeadersDemo(IModel model, AmqpMessagingService messagingService)
    {
    	Console.WriteLine("Enter your message as follows: the header values for 'category' and 'type separated by a colon. Then put a semicolon, and then the message. Quit with 'q'.");
    	while (true)
    	{
    		string fullEntry = Console.ReadLine();
    		string[] parts = fullEntry.Split(new char[] { ';' }, StringSplitOptions.RemoveEmptyEntries);
    		string headers = parts[0];
    		string[] headerValues = headers.Split(new char[] { ',' }, StringSplitOptions.RemoveEmptyEntries);
    		Dictionary<string, object> headersDictionary = new Dictionary<string, object>();
    		headersDictionary.Add("category", headerValues[0]);
    		headersDictionary.Add("type", headerValues[1]);
    		string message = parts[1];
    		if (message.ToLower() == "q") break;
    		messagingService.SendHeadersMessage(message, headersDictionary, model);
    	}
    }


Add a call to this private method from Main:



    RunHeadersDemo(model, messagingService);


It's time to set up the receivers. They will be very similar to what we have seen before. In preparation for the receiver projects insert the following three methods into AmqpMessagingService.cs:



    public void ReceiveHeadersMessageReceiverOne(IModel model)
    {
    	model.BasicQos(0, 1, false);
    	Subscription subscription = new Subscription(model, _headersQueueOne, false);
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = subscription.Next();
    		StringBuilder messageBuilder = new StringBuilder();
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		messageBuilder.Append("Message from queue: ").Append(message).Append(". ");
    		foreach (string headerKey in deliveryArguments.BasicProperties.Headers.Keys)
    		{
    			byte[] value = deliveryArguments.BasicProperties.Headers[headerKey] as byte[];
    			messageBuilder.Append("Header key: ").Append(headerKey).Append(", value: ").Append(Encoding.UTF8.GetString(value)).Append("; ");
    		}

    		Console.WriteLine(messageBuilder.ToString());
    		subscription.Ack(deliveryArguments);
    	}
    }

    public void ReceiveHeadersMessageReceiverTwo(IModel model)
    {
    	model.BasicQos(0, 1, false);
    	Subscription subscription = new Subscription(model, _headersQueueTwo, false);
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = subscription.Next();
    		StringBuilder messageBuilder = new StringBuilder();
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		messageBuilder.Append("Message from queue: ").Append(message).Append(". ");
    		foreach (string headerKey in deliveryArguments.BasicProperties.Headers.Keys)
    		{
    			byte[] value = deliveryArguments.BasicProperties.Headers[headerKey] as byte[];
    			messageBuilder.Append("Header key: ").Append(headerKey).Append(", value: ").Append(Encoding.UTF8.GetString(value)).Append("; ");
    		}

    		Console.WriteLine(messageBuilder.ToString());
    		subscription.Ack(deliveryArguments);
    	}
    }

    public void ReceiveHeadersMessageReceiverThree(IModel model)
    {
    	model.BasicQos(0, 1, false);
    	Subscription subscription = new Subscription(model, _headersQueueThree, false);
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = subscription.Next();
    		StringBuilder messageBuilder = new StringBuilder();
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		messageBuilder.Append("Message from queue: ").Append(message).Append(". ");
    		foreach (string headerKey in deliveryArguments.BasicProperties.Headers.Keys)
    		{
    			byte[] value = deliveryArguments.BasicProperties.Headers[headerKey] as byte[];
    			messageBuilder.Append("Header key: ").Append(headerKey).Append(", value: ").Append(Encoding.UTF8.GetString(value)).Append("; ");
    		}
            	Console.WriteLine(messageBuilder.ToString());
    		subscription.Ack(deliveryArguments);
    	}
    }


The only new bit of code is that we're extracting the header values from the incoming payload. Otherwise the code should be very familiar by now.

Add three new console applications to the solution: HeadersReceiverOne, HeadersReceiverTwo, HeadersReceiverThree. Add references to the RabbitMQ NuGet package and the RabbitMqService library in all three. Insert the following bits of code…:

…to HeadersReceiverOne.Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveHeadersMessageReceiverOne(model);


…to HeadersReceiverTwo.Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveHeadersMessageReceiverTwo(model);


…and to HeadersReceiverThree.Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveHeadersMessageReceiverThree(model);


Perform these steps to run all relevant console apps:

1. Make sure that HeadersSender is set as the start up project and start the application
2. Start the receivers by right-clicking them on Visual Studio and selecting Debug, Start new instance
3. You should have 4 console windows up and running on your screen

Start sending messages from the HeadersSender. Be careful with the delimiters: ',' for the headers and ';' for the message. The message should be routed according to the specified routing rules:

![Headers MEP console][3]

**Scatter/gather**

This pattern is similar to the RPC message exchange pattern we saw in [a previous post of this series][4] in that the sender will be expecting a response from the receiver. The main difference is that in this scenario the sender can collect a range of responses from various receivers. The sender will set up a temporary response queue where the receivers can send their responses. It's possible to implement this pattern using any exchange type: fanout, direct, headers and topic depending on how you've set up the exchange/queue binding. You can also specify a routing key in the binding as we saw before.

I think this is definitely a message exchange pattern which can be widely used in real applications out there that require 2 way communication with more than 2 parties. Consider that you send out a request to construction companies asking for a price offer. The companies then can respond using the message broker and the temporary response queue.

We'll re-use several ideas and bits of code from the RPC pattern so make sure you understand the basics of that MEP as well. I won't explain the same ideas again.

Let's set up the exchange and the queue first as usual. Insert the following private fields to AmqpMessagingService.cs:



    private string _scatterGatherExchange = "ScatterGatherExchange";
    private string _scatterGatherReceiverQueueOne = "ScatterGatherReceiverQueueOne";
    private string _scatterGatherReceiverQueueTwo = "ScatterGatherReceiverQueueTwo";
    private string _scatterGatherReceiverQueueThree = "ScatterGatherReceiverQueueThree";


The following method in AmqpMessagingService.cs will set up the necessary pieces:



    public void SetUpExchangeAndQueuesForScatterGatherDemo(IModel model)
    {
    	model.ExchangeDeclare(_scatterGatherExchange, ExchangeType.Topic, true);
    	model.QueueDeclare(_scatterGatherReceiverQueueOne, true, false, false, null);
    	model.QueueDeclare(_scatterGatherReceiverQueueTwo, true, false, false, null);
    	model.QueueDeclare(_scatterGatherReceiverQueueThree, true, false, false, null);

    	model.QueueBind(_scatterGatherReceiverQueueOne, _scatterGatherExchange, "cars");
    	model.QueueBind(_scatterGatherReceiverQueueOne, _scatterGatherExchange, "trucks");

    	model.QueueBind(_scatterGatherReceiverQueueTwo, _scatterGatherExchange, "cars");
    	model.QueueBind(_scatterGatherReceiverQueueTwo, _scatterGatherExchange, "aeroplanes");
    	model.QueueBind(_scatterGatherReceiverQueueTwo, _scatterGatherExchange, "buses");

    	model.QueueBind(_scatterGatherReceiverQueueThree, _scatterGatherExchange, "cars");
    	model.QueueBind(_scatterGatherReceiverQueueThree, _scatterGatherExchange, "buses");
    	model.QueueBind(_scatterGatherReceiverQueueThree, _scatterGatherExchange, "tractors");
    }


You'll notice that we are going to go for the Topic exchange type and that we'll bind 3 queues to the exchange. The routing keys will tell you what each receiver is interested in. E.g. all queues will receive a message with a routing key of "cars".

Add a new Console application called ScatterGatherSender to the solution. Add a reference to the RabbitMQ NuGet package and the RabbitMqService library. Insert the following code to Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.SetUpExchangeAndQueuesForScatterGatherDemo(model);


Set ScatterGatherSender as the start up project and run the application. Check in the RabbitMQ console that all elements have been set up correctly. Comment out the call to messagingService.SetUpExchangeAndQueuesForScatterGatherDemo.

Next we'll set up the message sending logic in AmqpMessagingService.cs. Like in RPC we'll need a queue that the Sender will dynamically set up. Insert the following private fields in AmqpMessagingService:



    private QueueingBasicConsumer _scatterGatherConsumer;
    private string _scatterGatherResponseQueue;


The following method will take care of sending the message to the exchange and collect the responses from the receivers:



    public List<string> SendScatterGatherMessageToQueues(string message, IModel model, TimeSpan timeout, string routingKey, int minResponses)
    {
    	List<string> responses = new List<string>();
    	if (string.IsNullOrEmpty(_scatterGatherResponseQueue))
    	{
    		_scatterGatherResponseQueue = model.QueueDeclare().QueueName;
    	}

    	if (_scatterGatherConsumer == null)
    	{
    		_scatterGatherConsumer = new QueueingBasicConsumer(model);
    		model.BasicConsume(_scatterGatherResponseQueue, true, _scatterGatherConsumer);
    	}

    	string correlationId = Guid.NewGuid().ToString();
    	IBasicProperties basicProperties = model.CreateBasicProperties();
    	basicProperties.ReplyTo = _scatterGatherResponseQueue;
    	basicProperties.CorrelationId = correlationId;

    	byte[] messageBytes = Encoding.UTF8.GetBytes(message);
    	model.BasicPublish(_scatterGatherExchange, routingKey, basicProperties, messageBytes);

    	DateTime timeoutDate = DateTime.UtcNow + timeout;
    	while (DateTime.UtcNow <= timeoutDate)
    	{
    		BasicDeliverEventArgs deliveryArguments;
    		_scatterGatherConsumer.Queue.Dequeue(500, out deliveryArguments);
    		if (deliveryArguments != null && deliveryArguments.BasicProperties != null
    			&& deliveryArguments.BasicProperties.CorrelationId == correlationId)
    		{
    			string response = Encoding.UTF8.GetString(deliveryArguments.Body);
    			responses.Add(response);
    			if (responses.Count >= minResponses)
    			{
    				break;
    			}
    		}
    	}

    	return responses;
    }


This piece of code looks very much like what we saw with the RPC pattern. The first key difference is that we need to wait for a range of responses, not just a single one, hence the return type of List of string. The purpose of the minResponse input parameter is that in practice the sender will probably not know how many responses it could receive so it specifies a minimum. The Dequeue() method has an interesting overload for a scenario where the sender doesn't know how long it can take for each receiver to respond:



    Dequeue(int millisecondsTimeout, out BasicDeliverEventArgs eventArgs);


If the timeout is passed then the BasicDeliverEventArgs eventArgs out parameter will be null, so we effectively ignore all responses that came in after the timeout. In the RPC example code we didn't specify any such timeout so the Dequeue() code will block the code execution until there's a message. In reality the sender could wait for a long time or even for ever to get a response so a timeout parameter can be very useful. Imagine that the sender specifies a min response count of 5 and only 3 responses are received. Then without a timeout parameter in Dequeue the sender would have to wait for ever which is not optimal. Instead we periodically check the queue, wait for 500 milliseconds and then try again until the timeOut date parameter is up. If the response count reaches the minimum before that then the response list is returned. Otherwise a shorter list will be returned. The sender can of course omit a minimum response count and simply wait until the timeout has been passed. This simulates the scenario where applicants are allowed to participate in an open tender until some specified deadline and the number of applications can be anything from 0 to int.MaxValue.

This method can be called from ScatterGatherSender as follows:



    private static void RunScatterGatherDemo(IModel model, AmqpMessagingService messagingService)
    {
    	Console.WriteLine("Enter your message as follows: the routing key, followed by a semicolon, and then the message. Quit with 'q'.");
    	while (true)
    	{
    		string fullEntry = Console.ReadLine();
    		string[] parts = fullEntry.Split(new char[] { ';' }, StringSplitOptions.RemoveEmptyEntries);
    		string key = parts[0];
    		string message = parts[1];
    		if (message.ToLower() == "q") break;
    		List<string> responses = messagingService.SendScatterGatherMessageToQueues(message, model, TimeSpan.FromSeconds(20), key, 3);
    		Console.WriteLine("Received the following messages: ");
    		foreach (string response in responses)
    		{
    			Console.WriteLine(response);
    		}
    	}
    }


So the receivers will have 20 seconds to respond.

Call this private method from Main:



    RunScatterGatherDemo(model, messagingService);


Back in AmqpMessagingService.cs we'll prepare the code which will receive the scatter/gather messages and send the responses from the receivers. The code is actually identical to ReceiveRpcMessage(IModel model) we saw earlier so I won't explain it again:



    public void ReceiveScatterGatherMessageOne(IModel model)
    {
    	ReceiveScatterGatherMessage(model, _scatterGatherReceiverQueueOne);
    }

    public void ReceiveScatterGatherMessageTwo(IModel model)
    {
    	ReceiveScatterGatherMessage(model, _scatterGatherReceiverQueueTwo);
    }

    public void ReceiveScatterGatherMessageThree(IModel model)
    {
    	ReceiveScatterGatherMessage(model, _scatterGatherReceiverQueueThree);
    }

    private void ReceiveScatterGatherMessage(IModel model, string queueName)
    {
    	model.BasicQos(0, 1, false);
    	QueueingBasicConsumer consumer = new QueueingBasicConsumer(model);
    	model.BasicConsume(queueName, false, consumer);
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


Insert threw new Console applications: ScatterGatherReceiverOne, ScatterGatherReceiverTwo, ScatterGatherReceiverThree. Add references to the RabbitMQ NuGet package and the RabbitMqService library to all 3. Insert the following bits of code.

To ScatterGatherReceiverOne.Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveScatterGatherMessageOne(model);


…to ScatterGatherReceiverTwo.Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveScatterGatherMessageTwo(model);


…and to ScatterGatherReceiverThree.Main:



    AmqpMessagingService messagingService = new AmqpMessagingService();
    IConnection connection = messagingService.GetRabbitMqConnection();
    IModel model = connection.CreateModel();
    messagingService.ReceiveScatterGatherMessageThree(model);


Follow these steps to start the demo:

1. Make sure that ScatterGatherSender is set as the start up project and start the application
2. Start all 3 receivers by the usual technique: right-click in VS, Debug, Start new instance
3. You'll have 4 console windows up and running on your screen

Start sending messages from the Sender. Take care when entering the message so you delimit the routing key and the message:

![scatter gather console][5]

Read the next part of this series [here][6].

View the list of posts on Messaging [here][7].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/05/08/messaging-with-rabbitmq-and-net-c-part-4-routing-and-topics/ "Messaging with RabbitMQ and .NET C# part 4: routing and topics"
[2]: http://geekswithblogs.net/michaelstephenson/Default.aspx "Michael Stephenson blog"
[3]: http://dotnetcodr.files.wordpress.com/2014/03/headersmepconsole.png?w=630&h=151
[4]: http://dotnetcodr.com/2014/05/05/messaging-with-rabbitmq-and-net-c-part-3-message-exchange-patterns/ "Messaging with RabbitMQ and .NET C# part 3: message exchange patterns"
[5]: http://dotnetcodr.files.wordpress.com/2014/03/scattergatherconsole.png?w=630&h=150
[6]: http://dotnetcodr.com/2014/06/05/rabbitmq-in-net-data-serialisation/ "RabbitMQ in .NET: data serialisation I"
[7]: http://dotnetcodr.com/messaging/ "Messaging"
