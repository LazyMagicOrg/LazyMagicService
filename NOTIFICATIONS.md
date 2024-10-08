# LazyMagic.Notifications

The objective of the Notifications service is to provide a simple way to send notifications to clients.

LazyMagicMDD builds REST APIs. REST APIS are request/response so clients would typically poll the API to get updates. Notifications extend the API so clients don't have to poll the API, instead notifications are sent to clients when a change occurs.

Clients subscribe to notifications based on "topics" through the REST API. 

Notifications are sent to clients using a WebSocket. Clients connect to the WebSocket and receive notifications.

Notifications are typically sent when a Create, Update, or Delete operation is performed on a database entity. For example, when a new Order is created, a notification is sent to all clients that have subscribed to the topic "Orders". The definition of topics is application specific so the use of a topic "Orders" is just for illustration.

LazyMagic provides a number of service-side and client-side libraries to make it easy to implement notifications in your application.

## AWS versus Local Development
The implementation of the Notifications service is different for AWS deployment and local development.

*AWS Gateway:* For AWS deployment we use the API Gateway WebSocket API. Clients connect to the WebSocket API and receive notifications. These notifications are generated by a Lambda function that is triggered by DynamoDB Streams.

*WebApi:* For local development we use a local WebApi service that uses a simple WebSocket implementation (WebSocketServices) to send notifications to clients.

## Notifications Table

Both implementations write notifications to a tenant specific DynamoDB table. The LazyMagic.Notifications.Repo library provides CRL operations on notifications. Notification records are write-once and are deleted after a TTL period so Update and Delete operations are not supported.

A notification record contains:
Id - a unique identifier for the notification
Topics - a list of topics that the notification is associated with
UserId - the user that the notification 
PayloadId - the id of the resource that the notification is about
PayloadType - the type of the resource that the notification is about. Used to determine how to deserialize the Payload.
Payload - the json representation of the entity instance
PayloadAction - the action that was performed on the resource (Create, Update, or Delete)
SessionId - the session id of the client session that 
CreatedUtcTick - the time the notification was created in UTC ticks
UpdatedUtcTick - the time the notification was updated in UTC ticks (same as CreatedUtcTick becuase these are write-once records)

Notifications are stored in a separate table because the AWS Implementation relies on DynamoDB Streams to send notifications. When you set up a DynamoDB Stream, all the changes in the table are sent to the Lambda function. Since we only want to send notifications for certain entities, we store the notifications in a separate table. Also, we typically don't want to backup notifications so having them in a separate table excludes them from backups of the primary tenant table.

### Tenant Tables
LazyMagic uses a multi-tenant architecture. Each tenant has two DynamoDB tables: "tenantkey" and tenantkey-lznotifications". Remember that a DynamoDB table is not the same as a SQL table. Instead, it can, and should according the AWS Single Table design guidelines, contain multiple different types of entities.  In a typical LazyMagic application, the tenantkey table contains all the entities for the tenant. The tenantkey-lznotifications table contains only the notifications for the tenant.

## LazyMagic.DynamoDBRepo.DYDBRepository Class
The DYDBRepository class is a base class for a repository class providing CRUD operations on a DynamoDB table. 

The DYDBRepository class provides a simple way to instrument a repo class to send notifications on Create, Update, and Delete operations. For each entity repo class where we want to send notifications we implement the virtual methods and set UseNotifications to true:
```csharp
Task WriteDeleteNotificationAsync(ICallerInfo callerInfo, string dataType, string sk, string topics, long updatedUtcTick);
Task WriteNotificationAsync(ICallerInfo callerInfo, string dataType, string data, string topics, long updatedUtcTick, string action);
```

For an entity, we may use the DYDBRepository class this way:
```csharp
public class OrderRepo : DYDBRepository<OrderEnvelope, Order>, IOrderRepo 
{
    public OrderRepo(
    IAmazonDynamoDB client,
    ILzNotificationRepo lzNotificationRepo) : base(client) 
    { 
        PK = OrderEnvelope.DefaultPK; 
        this.lzNotifications = lzNotificationRepo;
        UseNotifications = true; 
    }

    protected ILzNotificationRepo lzNotificationRepo;  

    public override string SetTopic() => $"[\"Orders\"]"; 

    public override async Task WriteDeleteNotificationAsync(ICallerInfo callerInfo, string dataType, string sk, string topics, long updatedUtcTick)
        => await lzNotificationRepo.WriteDeleteNotificationAsync(callerInfo, dataType, sk, topics, updatedUtcTick);

    public override async Task WriteNotificationAsync(ICallerInfo callerInfo, string dataType, string data, string topics, long updatedUtcTick, string action)
        => await lzNotificationRepo.WriteNotificationAsync(callerInfo, dataType, data, topics, updatedUtcTick, action);
	
}
```
## LazyMagic.Notifications.DynamoDBRepo.DYDBRepositoryWithNotification Class
Using the DYDBRepository class gets repetitive so we have a derived class called DYDBRepositoryWithNotification to do all the repetitive work.
```csharp
public class PetRepo : DYDBRepositoryWithNotification<PetEnvelope, Pet>, IPetRepo 
{
    public OrderRepo(
    IAmazonDynamoDB client,
    ILzNotificationRepo lzNotificationRepo) : base(client, lzNotificationRepo) 
    { 
        PK = OrderEnvelope.DefaultPK; 
    }
    public override string SetTopic() => $"[\"Orders\"]"; 
}
```

## SetTopic() 
The SetTopic() method is used to determine the topics that a notification is associated with. The topics are used to determine which clients receive the notification. The topics are stored in the Topics property of the notification record. The Topics property is a json array of strings.

Topics are NOT usually the PK of the entity. They are usually a list of strings that are used to filter notifications. For example, if we want to send notifications to all clients that are interested in Orders, we would set the Topics property to ["Orders"].

 The choice of topics completely dependent on the application.
SetTopics() is called by the base class when a entity is created, updated or deleted and prior to the notification being written to the notification table.

## Notifications and Subscriptions
A client can subscribe to notifications. A subscription is a list of topics. When a client subscribes to notifications, the client receives all notifications that have topics that match the subscription topics.
The Subscription contains:
Id - a unique identifier for the subscription and usually the client session id 
ConnectionId - the websocket connection id
TopicIds - a list of topics that the client is interested in
CreatedUtcTick - the time the subscription was created in UTC ticks
UpdatedUtcTick - the time the subscription was updated in UTC ticks

## WebSocket Connections 
A client connects to the WebSocket API and receives notifications. The LazyMagic.Notifications.ViewModels library provides a ViewModel class called LzNotificationsSvc that manages the WebSocket connection. The ViewModel class provides methods to subscribe to notifications and to request notifications that have been missed. The ViewModel class also provides methods to handle notifications received from the WebSocket API.

- Socket Create: When the the client connects to the WebSocket API, the WebSocket service sends the client a message containing the connection id. that have been missed.
- Socket Close: When the client disconnects from the WebSocket API, existing subscriptions for that client are not deleted.

WebSocket connections may be closed for a number of reasons. For example, the client may have lost internet connectivity or the application may have been put in sleep mode by the OS and the WebSocket service closed the connection. When a client reconnects to the WebSocket API, the client receives a new connection id. The client must resubscribe to notifications using the new connection id. The LzNotificationsSvc view model will request notifications that have been missed since the last connection using the LzNotificationsSvc.ReadNotificationsAsync() method.

See the README.md in the LazyMagic.Notifications.ViewModel library for more details.

## ViewModel Updates 
The LazyMagic.ViewModel library provides the classes LzItemsViewModelNotificationsBase and LzItemViewModelNotificationsBase which handle notifications received by the LzNotificationsSvc view model. See the README.md in the LazyMagic.ViewModel library for more details.

## Libraries
### Both Client and Service Side - LazyMagicShared solution
LazyMagic.Notifications.Schema

### AWS Service Side - LazyMagicService solution
LazyMagic.Notifications.FromStreams
LazyMagic.Notifications.Notifications.Repo 
LazyMagic.Notifications.Notifications.WebSocket
LazyMagic.Notifications.SharedCode
LazyMagic.Notifications.DynamoDBRepo

### Local WebApi Service - LazyMagicService solution
LazyMagic.Notifications.WebSocketService

### Client Side Libraries - in the LazyMagicClient solution
LazyMagic.Notifications.ViewModels 
LazyMagic.Notifications.SharedSDK 

## Service Side Configuration 
Setting up your LazyMagic service-side solution to use the Notifications service is a multi-step process. The steps are:
1. Add external library references to these libraries:
    - LazyMagic.Notifications.DynamoDBRepo
    - LazyMagic.Notifications.FromStreams
    - LazyMagic.Notifications.Repo 
    - LazyMagic.Notifications.Schema
    - LazyMagic.Notifications.SharedCode 
    - LazyMagic.Notifications.WebSocket
    - LazyMagic.Notifications.WebSocketService 
2. Add these lines to your LazyMagic.yaml file
```
OpenApiSpecs:
- ..\LazyMagicService\LazyMagic.Notifications.SharedCode\NotificationsSvc.yaml
AwsTemplates:
- ..\LazyMagicService\LazyMagic.Notifications.SharedCode\NotificationsSvcSAM.yaml
```
3. Create a NotificationsFromStreams Lambda. See example in PetStore solution.
4. Create a NotificationsWebSocket Lambda. See example in PetStore solution.
5. Add this dependency to your WebApi project:
```
    <ProjectReference Include="..\..\LazyMagicService\LazyMagic.Notifications.WebSocketService\LazyMagic.Notifications.WebSocketService.csproj" />
```
6. Update your WebApi Startup.Configure method:
```
        var webSocketOptions = new WebSocketOptions()
        {
            KeepAliveInterval = TimeSpan.FromSeconds(120)
        };
        app.UseWebSockets(webSocketOptions);  	
```

Review the PetStore sample solution for an working example of how to set up the Notifications service.

## Client Side Configuration
Review the README.md in the LazyMagic.Notifications.ViewModels library for details on how to configure the client side.

