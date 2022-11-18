---
layout: post
title: Asynchronous Request-Reply pattern with Azure Service Bus
tags: [Azure, dotnet]
feature-img: "assets/img/articles/2022/request-reply/queue-min.webp"
# thumbnail: "assets/img/articles/2022/request-reply/queue-min.webp"
excerpt_separator: <!--more-->
---

These days message brokers are widely used systems supporting our applications from choking when we have many long running tasks to execute. But how can we get feedback regarding those tasks?
<!--more-->

Azure Service Bus is a fully managed message broker with support for both queue messages and publish-subscribe topics. For whole description I'm redirecting to [Microsoft documentation](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview). Basically Service Bus works as a message buffer between our applications. On one end we push messages to the bus. On the second we have application listening for new messages and processing them one by one. We can set to allow only one message to be processed or many (concurrency). Beside basic queueing functionality Azure Service Bus offers many features, including sessions which gives us ability to implement Asynchronous Request-Reply pattern.

### Piece of theory

Modern web application are usually divided into UI and backend layer. Client applications often depend on APIs which provide business logic. In most cases these 2 layers are bounded with each other utilizing HTTP(S) protocol. Response to the UI layer should come relatively fast to keep front-end responsive, but there are situations when this may be hard to achieve. In some cases the UI may trigger long-running process. It can take from seconds up to even minutes or hours to get response from finished task. For this kind of situation Asynchronous Request-Reply pattern applies.

It is common to use queue-based message system not to overload applications with many resource-heavy processes. The con of this approach is introducing one-way communication. In this situation the client sending original request can start making interval HTTP calls to ask if the result is already there.

You can find more in-depth pattern description [here](https://learn.microsoft.com/en-us/azure/architecture/patterns/async-request-reply).

### Implementation

To reproduce my steps you will need an active Azure subscription. If you don't have any you can get a free one with 200$ to use in the first month [here](https://azure.microsoft.com/en-in/free/). Azure Service Bus Sessions feature is enabled in Standard and Premium pricing tier. For testing free limits should not be exceeded, but there is also a "Base Charge". You can check the full pricing info [here](https://azure.microsoft.com/en-us/pricing/details/service-bus/).

Let's start with creating necessary resources in Azure. I will use an Azure CLI for that. Check [this](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) for installation guides. First let us create a resource group and Azure Service Bus namespace:

```powershell
az group create --name RequestReplyPattern --location 'west europe'
az servicebus namespace create --name requestreplyasb --resource-group RequestReplyPattern --sku Standard
```

Then we need to create two queues. One will receive our requests and the other will be used to send responses on sessions. This feature needs to be explicitly enabled on queue.

```powershell
az servicebus queue create --name request-queue --namespace-name requestreplyasb --resource-group RequestReplyPattern
az servicebus queue create --name reply-queue --namespace-name requestreplyasb --resource-group RequestReplyPattern --enable-session true
```
After executing above commands you can check Azure Portal. You should find there a new resource group with Azure Service Bus namespace and two queues within it.

I am not going to describe every single step that I have done in code, there would be to much of it. The whole sample containing two applications, Producer and Consumer, I have put in this [repo](https://github.com/pbakun/asb-request-reply-pattern).

#### Produce initial message

In Producer code I'm sending initial message to `request-queue` queue. My custom `IServiceBusFactory` helps me creating a sender. In the `busMessage` I'm passing json string as message body. The `ReplyToSessionId` property is used to inform Consumer on which session identifier I will expect to receive the reply. Additionally I can pass queue name, but for this to work `SessionId` property has to be set as well, even if the queue is not session aware. Otherwise the `ReplyTo` property on the Consumer side will be `null`.

```csharp
public async Task<string> Produce(string jsonMessage)
{
    ServiceBusSender sender = _serviceBusFactory.CreateSender(_queueOptions.RequestQueueName);

    var busMessage = new ServiceBusMessage(jsonMessage)
    {
        ReplyToSessionId = Guid.NewGuid().ToString(),
        ReplyTo = _queueOptions.ReplyQueueName,
        SessionId = Guid.NewGuid().ToString()
    };

    _logger.LogInformation($"Send message with id {busMessage.ReplyToSessionId}");
    await sender.SendMessageAsync(busMessage);
    return busMessage.ReplyToSessionId;
}
```

#### Consume the message and send reply

In Consumer project I have created a `ServiceBusProcessor`, which listens to new messages on `request-queue`. This is not mandatory, there are many ways to receive queue messages. Instead there could be an Azure Function which gets triggered with each new Azure Service Bus message. The processor only listens for new messages and passes them to `QueueConsumer` instance for the sake of this sample.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    ServiceBusProcessor processor = _serviceBusFactory.CreateProcessor(_queueOptions.RequestQueueName);

    processor.ProcessMessageAsync += Processor_ProcessMessageAsync;
    processor.ProcessErrorAsync += Processor_ProcessErrorAsync;

    await processor.StartProcessingAsync(stoppingToken);
}

private Task Processor_ProcessErrorAsync(ProcessErrorEventArgs arg)
{
    _logger.LogError("Error in processed message");
    return Task.CompletedTask;
}

private async Task Processor_ProcessMessageAsync(ProcessMessageEventArgs arg)
{
    Message message = arg.Message.Body.ToObjectFromJson<Message>(new JsonSerializerOptions());
    using (var scope = _scopeFactory.CreateScope())
    {
        IQueueConsumer consumer = scope.ServiceProvider.GetRequiredService<IQueueConsumer>();
        await consumer.Consume(message, arg.Message.ReplyTo, arg.Message.ReplyToSessionId);
        await arg.CompleteMessageAsync(arg.Message);
    }
}
```

In the `QueueConsumer` class first thing I do is to set the session state, this action in fact initializes it. The sessionId is taken from `ReplyToSessionId` property from received queue message. I have an enum with 3 available states, but in fact to set it the same `BinaryData` object is used as for `ServiceBusMessage` Body property, so it could be anything within the limit of Service Bus message size.

> From the Service Bus perspective, the message session state is an opaque binary object that can hold data of the size of one message, which is 256 KB for Service Bus Standard, and 100 MB for Service Bus Premium.

Next I'm creating a new `ServiceBusMessage` object and assign session id (taken from `ReplyToSessionId`) to `SessionId` property. It must corresponds to the id that we will use to listen for the reply. It is also not possible to send message without setting `SessionId` on session aware queue. The rest is pretty straight forward. Message is sent to `reply-queue` and state is set to `Ready`. In `SetSessionState` method I'm using `using` statement to create a receiver, which has methods allowing me to modify the state. Without disposing `ServiceBusSessionReceiver` directly after usage I would lock the session and it would not be possible to read the state until the lock is released. There may be only one receiver at the time locked on a session!

```csharp
public async Task Consume<T>(T jsonMessage, string queueName, string sessionId)
{
    _logger.LogInformation("Received message: {0}", jsonMessage?.ToString());
    await SetSessionState(queueName, sessionId, MessageState.Processing);
    string message = JsonSerializer.Serialize(new Message
    {
        Content = $"Thanks for message with session id {sessionId}! This is my response"
    });

    ServiceBusSender sender = _serviceBusFactory.CreateSender(queueName);

    var busMessage = new ServiceBusMessage(message)
    {
        SessionId = sessionId
    };

    //delay to simulate longer running process
    await Task.Delay(TimeSpan.FromSeconds(15));

    _logger.LogInformation("Sending response to messageId {0}", sessionId);
    await sender.SendMessageAsync(busMessage);
    await SetSessionState(queueName, sessionId, MessageState.Ready);
}

private async Task SetSessionState(string queueName, string sessionId, MessageState messageState)
{
    await using ServiceBusSessionReceiver receiver = await _serviceBusFactory.CreateSessionReceiver(queueName, sessionId);
    _logger.LogInformation("Setting state session id{0} to {1}", sessionId, messageState.ToString());
    var state = new BinaryData(messageState.ToString());
    await receiver.SetSessionStateAsync(state);
}
```

#### Back to producer

Now that we have sent back the reply we can try to receive it. In the `GetMessageResponse` I'm creating a `ServiceBusSessionReceiver` and call `ReceiveMessageAsync` on it, passing `TimeSpan` that I'm willing to wait for the response. If nothing would come in the given time the result would be `null`. Otherwise we get back the message with its content in `Body` property.

```csharp
public async Task<string> GetMessageResponse(string messageId)
{
    await using ServiceBusSessionReceiver receiver = await _serviceBusFactory.CreateSessionReceiver(_queueOptions.ReplyQueueName, messageId);
    _logger.LogInformation($"Attempt to receive message with id {messageId}");
    ServiceBusReceivedMessage receivedMessage = await receiver.ReceiveMessageAsync(TimeSpan.FromSeconds(5));
    if (receivedMessage == null)
        return string.Empty;

    _logger.LogInformation("Received response from queue");
    await receiver.SetSessionStateAsync(null);
    Message responseMessage = receivedMessage.Body.ToObjectFromJson<Message>(new System.Text.Json.JsonSerializerOptions());
    return responseMessage.Content;
}
```

After ensuring I got a valid reply I need to set the session state back to `null`. It is very important, since the session state does not get deleted automatically, even if there are no active messages associated with it. Also in Azure Service Bus preview in Azure Portal it is not showed anywhere whether there are any active sessions. From what I have found at the time of writing this post that information can be obtained only when connected with [AMPQ](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-request-response#enumerate-sessions) to Azure Service Bus or using [QueueClient](https://learn.microsoft.com/en-us/dotnet/api/microsoft.servicebus.messaging.queueclient.getmessagesessions?view=azure-dotnet) from deprecated `Microsoft.ServiceBus.Messaging` namespace.

> Session state remains as long as it isn't cleared up (returning null), even if all messages in a session are consumed.

#### Reply receiver

Keep in mind that the receiver object used for setting state is the same one that is used for getting reply. State could be also set from within of processor listening on session queue. This is the implementation for receiver:

```csharp
public async Task<ServiceBusSessionReceiver> CreateSessionReceiver(string queueName, string sessionId)
{
    if (string.IsNullOrEmpty(queueName))
        throw new ArgumentNullException(queueName);
    if (string.IsNullOrEmpty(sessionId))
        throw new ArgumentNullException(sessionId);

    var receiverOptions = new ServiceBusSessionReceiverOptions
    {
        ReceiveMode = ServiceBusReceiveMode.ReceiveAndDelete
    };
    return await _client.AcceptSessionAsync(queueName, sessionId, receiverOptions);
}
```

There is also a method available within `ServiceBusClient` that allows to grab the first free session reply - `AcceptNextSessionAsync`.

### Wrap up

In this post I wanted to show a simple way to utilize Asynchronous Request-Reply pattern. It is a great way to run heavier processes in more controlled way, but preserving responsiveness of our application in clean way.

I hope the implementation is clear enough and I encourage you to check out the sample [repository](https://github.com/pbakun/asb-request-reply-pattern). The Producer contains 3 HTTP endpoints which can be used to send message, get state and receive response. Only creating your own Azure resources separates you from testing pattern implementation yourself.

As always if you have any questions or suggestions, or just want to reach out, send me a DM on [Twitter](https://twitter.com/piotrbakun). Till the next one!