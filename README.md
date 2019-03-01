# Ascend.Queue.Middleware
Ascend.Queue.Middleware is a .NET Core Middleware intended to interface with preferred queuing technologies used by Ascend Learning.  The implementation is built on top of the [Ascend.Queue](https://github.com/timothyfranzke/Ascend.Queue) library's Subscriber implementation. The current implementation v.0.0.1.275 communicates strictly with Kafka

## Terms

 1. **Subscriber** - The application that is listening or checking for updates from the queue.  In Kafka terms, it is synonymous with Consumer.  More about Kafka Consumers can be found [here](https://kafka.apache.org/documentation/#intro_consumers).
 2. **Publisher** - The application that will be writing information to the queue.  In Kafka terms, it is synonymous with Producer.  More about Kafka Producers can be found [here](https://kafka.apache.org/documentation/#intro_producers)
## Installation
Ensure your package
To install Ascend.Queue.Middleware from within Visual Studio, search for Ascend.Queue.Middleware in the NuGet Package Manager UI, or run the following command in the Package Manager Console:

    Install-Package Ascend.Queue.Middleware
## Set Up Steps

 1. Create a class that extends ConfigurableSubscriber available in Ascend.Queue.Middleware.Implemetations include Ascend.Queue.Middleware.Implemetations, Ascend.Queue.Middleware.Interfaces, Ascend.Queue.Models and Microsoft.Extensions.Options

		using Ascend.Queue.Middleware.Implemetations;
	    using Ascend.Queue.Middleware.Interfaces;
	    using Ascend.Queue.Models;
	    using Microsoft.Extensions.Options;
	 

 2. Add a constructor that takes IConfigurationHandler and IOptions<Connection> and pass to the base

		 public ExampleSubscriber(IConfigurationHandler configurationHandler, IOptions<Connection> connection) 
		        : base(configurationHandler, connection)
		    {
		    }

 3. Override the Log and Message methods

		 public override void Log(Log log)
	    {
	        System.Console.WriteLine(log.Message);
	    }

	    public override void Message(object message)
	    {
	        var queueMessage = message as Message<string, string>;
	        System.Console.WriteLine(queueMessage.Value);
	    }
	*See a full example of the ConfigurableSubscriber Implementation below*
 4. In the StartUp.cs class, include both Ascend.Queue.Middleware.Extensions and Ascend.Queue.Models

    	using Ascend.Queue.Middleware.Extensions;
	    using Ascend.Queue.Models;
	
 5. In ConfigureServices(IServiceCollection services) use services.Configure<Connection>((options) => {} to configure your consumer

		 services.Configure<Connection>((options) =>
            {
                options.ApplicationName = "Example Application";
                options.Endpoint = "dev-kafkabroker.ascendlearning.com:9092";
                options.QueueNames = new List<string> { "foo" };
                options.Password = "Password1";
                options.Settings = new KafkaSettings
                {
                    NumberOfConsumerThreads = 5,
                    EnableAutoCommit = true
            };
        });

 6. Use services.AddAscendQueueMonitor<> to use your implementation
	 

	    services.AddAscendQueueMonitor<ExampleSubscriber>();

 7. In the Configure method, use app.UseAscendConfigurableMonitor() 

		 public void Configure(IApplicationBuilder app, IHostingEnvironment env)
		    {
		        app.UseAscendConfigurableMonitor();
		    }
		    
	*See a full example of the StartUp.cs Implementation below*

## ConfigurableSubscriber Full Implementation

    using Ascend.Queue.Middleware.Implemetations;
    using Ascend.Queue.Middleware.Interfaces;
    using Ascend.Queue.Models;
    using Microsoft.Extensions.Options;
    
    namespace Ascend.Queue.Middleware.Example
    {
        public class ExampleSubscriber: ConfigurableSubscriber<string, string>
        {
            public ExampleSubscriber(IConfigurationHandler configurationHandler, IOptions<Connection> connection) 
                : base(configurationHandler, connection)
            {
            }
    
            public override void Log(Log log)
            {
                System.Console.WriteLine(log.Message);
            }
    
            public override void Message(object message)
            {
                var queueMessage = message as Message<string, string>;
                System.Console.WriteLine(queueMessage.Value);
            }
        }
    }
## StartUp.cs Full Implementation

    using System.Collections.Generic;
    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore.Hosting;
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.DependencyInjection;
    using Ascend.Queue.Middleware.Extensions;
    using Ascend.Queue.Models;
    
    namespace Ascend.Queue.Middleware.Example
    {
        public class Startup
        {
            public Startup(IConfiguration configuration)
            {
                Configuration = configuration;
            }
    
            public IConfiguration Configuration { get; }
            public void ConfigureServices(IServiceCollection services)
            {
                services.Configure<Connection>((options) =>
                {
                    options.ApplicationName = "Example Application";
                    options.Endpoint = "dev-kafkabroker.ascendlearning.com:9092";
                    options.QueueNames = new List<string> { "foo" };
                    options.Password = "Password1";
                    options.Settings = new KafkaSettings
                    {
                        NumberOfConsumerThreads = 5
                    };
                });
                services.AddAscendQueueMonitor<ExampleSubscriber>();
            }
            public void Configure(IApplicationBuilder app, IHostingEnvironment env)
            {
                app.UseAscendConfigurableMonitor();
            }
        }
    }
## Connection
The Connection object is used to tell the Subscriber and the Publisher needed information about connecting to the Queue (currently Kafka).    

### Connection Members

 - **ApplicationName (required)** - This is important as it gives the subscriber a specific name to monitor.  When using Kafka, this will be considered the "group.id" Giving all instances of your subscriber the same group name will ensure that only one instance of the group will receive a unique message.
 - **QueueNames (required)** - A list of queues (topics in Kakfa).  
	 - A subscriber can subscribe to multiple queues. The ***Message*** object contains metadata that will differentiate between the topics.  ***There can only be one topic per Publisher.  If multiple are given, only the first topic in the list will be used.*** 
 - **Endpoint (required)** - The queue server your want to connect to
 - **Username** - Required if authentication is used on the queue
 - **Password** - Required if authentication is used on the queue
 - **Settings** - Queue technology specific settings.  For Kafka use the [KafkaSettings](#kafkasettings) object

	   var connection = new Connection()
	         {
	             ApplicationName = "TimsLocalMachine",
	             QueueNames = new List<string> { "foo" },
	             Endpoint = "dev-kafkabroker.ascendlearning.com:9092",
	             Username = "ati.stuqmon.stg-consumer",
	             Settings = new KafkaSettings()
	             {
	                 ConsumerPollIntervalMS = 5000,
	                 AutoCommitIntervalMS = 100,
	                 AutoOffsetReset = Ascend.Queue.Enumerations.OffsetResets.Earliest,
	                 EnableAutoCommit = false,
	                 DebugLogging = new List<Ascend.Queue.Enumerations.Debug>() { Ascend.Queue.Enumerations.Debug.consumer, Ascend.Queue.Enumerations.Debug.msg }
	             }
	         };
	         var subscriber = SubscriberFactory.GetSubscriber<Null, string>(connection);

### KafkaSettings
 - **NumberOfConsumerThreads** - The number of threads running on an application instance.  
	 - This is specific to ***Subscribers*** only.  
	 - Used to scale Kafka Consumers in a Consumer Group.  Read more about that [here](https://kafka.apache.org/documentation/#intro_consumers)
	 - *Default 1*
 - **PollTimeSpanMS** - The time a Consumer waits to timeout on a poll to the Broker. 
	 - *Default 100MS* 
 - **ConsumerPollIntervalMS** - The time between a Consumer poll attempt
	 - *Default 100MS*
 - **EnableAutoCommit** - 
	 - If true, commits will be handled by the framework based on a schedule. 
	 - If false, the Subscriber application will need to use the CommitAsync() method to commit it's own offsets
	 - *Default true*
 - **AutoCommitIntervalMS** - The time between committing a batch of offsets.  ***Only used if EnableAutoCommit is true***
	 - *Default 1000MS*
 - **AutoOffsetReset** - When a Consumer is starting for the first time, and has not committed an offset, it needs to determine whether it wants to read from the beginning of the Queue or start with the next incoming message.  
	 *Default OffsetResets.Earliest*
- **DebugLogging** - Takes a list of Ascend.Queue.Enumerations.Debug.  Read more about each of these debug settings [here](https://docs.confluent.io/4.1.2/clients/librdkafka/CONFIGURATION_8md.html).
 
## Message
### Message Object
- **ConsumerId** - If provide, this id will persist with each message sent with a specific consumer.  This gets set on the [Subscriber](#subscriber) object.
-  **QueueName** - The name of the queue the message is coming from
-  **Key** - The key that is given for the specific message.  If no key was given, this will be null.
- **Value** - The message body of the message in the queue.  This will be the value your application is most likely looking for. 
- **Metadata** - A Dictionary<string, string> of queue specific information.  For Kafka these will be "offset", "partition" and "topic"

## Log 
### Log Object
- **Level** - Type of Ascend.Queue.Enumerations.LogLevel
	- Error
	- Warn
	- Info
	- Debug
- **Message** - A string containing the information of the log
