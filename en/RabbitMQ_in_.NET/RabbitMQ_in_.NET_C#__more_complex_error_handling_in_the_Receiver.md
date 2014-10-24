[Source](http://dotnetcodr.com/2014/06/19/rabbitmq-in-net-c-more-complex-error-handling-in-the-receiver/ "Permalink to RabbitMQ in .NET C#: more complex error handling in the Receiver")

# RabbitMQ in .NET C#: more complex error handling in the Receiver

**Introduction**

In the [previous part][1] on RabbitMQ .NET we looked at ways how to reject a message if there was an exception while handling the message on the Receiver's side. The message could then be discarded or re-queued for a retry. However, the exception handling logic was very primitive in that the same message could potentially be thrown at the receiver infinitely causing a traffic jam in the messages.

This post builds upon the basics of RabbitMQ in .NET. If you are new to this topic you should check out all the previous posts listed on [this][2] page. I won't provide any details on bits of code that we've gone through before.

Most of the posts on RabbitMQ on this blog are based on the work of RabbitMQ guru [Michael Stephenson][3].

So we cannot just keep retrying forever. We can instead finally discard the message after a certain amount of retries or depending on what kind of exception was encountered.

The logic around retries must be implemented in the receiver as there's no simple method in RabbitMQ .NET, like "BasicRetry". Why should there be anyway? Retry strategies can be very diverse so it's easier to let the receiver handle it.

The strategy here is to reject the message without re-queuing it. We'll then create a new message based on the one that caused the exception and attach an integer value to it indicating the number of retries. Then depending on a maximum ceiling we either create yet another message for re-queuing or discard it altogether.

We'll build on the demo we started on in the previous post referred to above so have it ready.

**Demo**

We'll reuse the queue from the previous post which we called "BadMessageQueue". We'll also reuse the code in BadMessageSender as there's no variation on the Sender side.

BadMessageReceiver will however handle the messages in a different way. Currently there's a method called ReceiveBadMessages which is called upon from Main. Comment out that method call. Insert the following method in ReceiveBadMessages.Program.cs and call it from Main:



    private static void ReceiveBadMessageExtended(IModel model)
    {
    	model.BasicQos(0, 1, false);
    	QueueingBasicConsumer consumer = new QueueingBasicConsumer(model);
    	model.BasicConsume(RabbitMqService.BadMessageBufferedQueue, false, consumer);
    	string customRetryHeaderName = "number-of-retries";
    	int maxNumberOfRetries = 3;
    	while (true)
    	{
    		BasicDeliverEventArgs deliveryArguments = consumer.Queue.Dequeue() as BasicDeliverEventArgs;
    		String message = Encoding.UTF8.GetString(deliveryArguments.Body);
    		Console.WriteLine("Message from queue: {0}", message);
    		Random random = new Random();
    		int i = random.Next(0, 3);
    		int retryCount = GetRetryCount(deliveryArguments.BasicProperties, customRetryHeaderName);
    		if (i == 2) //no exception, accept message
    		{
    			Console.WriteLine("Message {0} accepted. Number of retries: {1}", message, retryCount);
    			model.BasicAck(deliveryArguments.DeliveryTag, false);
    		}
    		else //simulate exception: accept message, but create copy and throw back
    		{
    			if (retryCount < maxNumberOfRetries)
    			{
    				Console.WriteLine("Message {0} has thrown an exception. Current number of retries: {1}", message, retryCount);
    				IBasicProperties propertiesForCopy = model.CreateBasicProperties();
    				IDictionary<string, object> headersCopy = CopyHeaders(deliveryArguments.BasicProperties);
    				propertiesForCopy.Headers = headersCopy;
    				propertiesForCopy.Headers[customRetryHeaderName] = ++retryCount;
    				model.BasicPublish(deliveryArguments.Exchange, deliveryArguments.RoutingKey, propertiesForCopy, deliveryArguments.Body);
    				model.BasicAck(deliveryArguments.DeliveryTag, false);
    				Console.WriteLine("Message {0} thrown back at queue for retry. New retry count: {1}", message, retryCount);
    			}
    			else //must be rejected, cannot process
    			{
    				Console.WriteLine("Message {0} has reached the max number of retries. It will be rejected.", message);
    				model.BasicReject(deliveryArguments.DeliveryTag, false);
    			}
    		}
    	}
    }


…where CopyHeaders and GetRetryCount look as follows:



    private static IDictionary<string, object> CopyHeaders(IBasicProperties originalProperties)
    {
    	IDictionary<string, object> dict = new Dictionary<string, object>();
    	IDictionary<string, object> headers = originalProperties.Headers;
    	if (headers != null)
    	{
    		foreach (KeyValuePair<string, object> kvp in headers)
    		{
    			dict[kvp.Key] = kvp.Value;
    		}
    	}

    	return dict;
    }

    private static int GetRetryCount(IBasicProperties messageProperties, string countHeader)
    {
    	IDictionary<string, object> headers = messageProperties.Headers;
    	int count = 0;
    	if (headers != null)
    	{
    		if (headers.ContainsKey(countHeader))
    		{
    			string countAsString = Convert.ToString( headers[countHeader]);
    			count = Convert.ToInt32(countAsString);
    		}
    	}

    	return count;
    }


Let's see what's going on here. We define a custom header to store the number of retries for a message. We also set an upper limit of 3 on the number of retries. Then we accept the messages in the usual way. A random number between 0 and 3 is generated – where the upper limit is exclusive – to decide whether to simulate an exception or not. If this number is 2 then we accept and acknowledge the message, so there's a higher probability of "throwing an exception" just to make this demo more interesting. We also extract the current number of retries using the GetRetryCount method. This helper method simply checks the headers of the message for the presence of the custom retry count header.

If we simulate an exception then we need to check if the current retry count has reached the max number of retries. If not then the exciting new stuff begins. We create a new message where we copy the elements of the original message. We also set the new value of the retry count header. We send the message copy back to where it came from and acknowledge the original message. Otherwise if the max number of retries has been reached we reject the message completely using the BasicReject method we saw in the previous part.

Run both the Sender and Receiver apps and start sending messages from the Sender. Depending on the random number generated in the Receiver you'll see a differing number of retries but you may get something like this:

![Advanced retry console output][4]

We can see the following here:

* Message hello was rejected at first and then accepted after 1 retry
* Message hi was accepted immediately
* Message bye was accepted after 2 retries
* Message seeyou was rejected completely

So we've seen how to add some more logic into how to handle exceptions.

Other considerations and extensions:

* You can specify different max retries depending on the exception type. In that case you can add the exception type to the headers as well
* You might consider storing the retry count somewhere else than the message itself, e.g. within the Receiver – the advantage of storing the retry count in the message is that if you have multiple receivers waiting for messages from the same queue then they will all have access to the retry property
* If there's a dependency between messages then exception handling becomes a bigger challenge: if message B depends on message A and message A throws an exception, what do we do with message B? You can force related messages to be processed in an ordered fashion which will have a negative impact on the message throughput. On the other hand you may simply ignore this scenario if it's not important enough for your case – "enough" depends on the cost of slower message throughput versus the cost of an exception in interdependent messages. Somewhere between these two extremes you can decide to keep the order of related messages only and let all others be delivered normally. In this case you can put the sequence number, such as "5/10″ in the header so that the receiver can check if all messages have come in correctly. If you have multiple receivers then the sequence number must be stored externally so that all receivers will have access to the same information. Otherwise you can have a separate queue or even a separate RabbitMQ instance for related messages in case the proportion of related messages in total number of messages is small.

View the list of posts on Messaging [here][2].

### Like this:

Like Loading...

### _Related_

[1]: http://dotnetcodr.com/2014/06/16/rabbitmq-in-net-c-basic-error-handling-in-receiver/ "RabbitMQ in .NET C#: basic error handling in Receiver"
[2]: http://dotnetcodr.com/messaging/ "Messaging"
[3]: http://geekswithblogs.net/michaelstephenson/Default.aspx "Michael Stephenson blog"
[4]: http://dotnetcodr.files.wordpress.com/2014/05/advanced-retry-console-output.png?w=630&h=466
