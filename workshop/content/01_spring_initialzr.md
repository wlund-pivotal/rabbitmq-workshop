---
pageTitle: Spring Initializr for spring-AMQP project setup
---

# Objective

In this lab you will use Spring Initialzr to generate the project shell that will contain the correct version of spring-boot and the required starters for spring-amqp to work correctly and
with the benefits of auto-configuration.  In addition we will add other dependencies that will be used later in the labs. 

We will create a simple app in this lab that will demonstrate that the configuration provided through start.spring.io and a few dependencies allow us to publishing and echo those messages 
onto the console. We will also understand the dependencies selected for the project that enable spring-amqp. 

# Using Spring Initialzr

Life for most Spring Projects begin at start.spring.io.  Navigate to [Spring Initializr](http://start.spring.io). ![Spring Initializr](images/spring-initializr.png)

Fill in the required fields with the following as shown in the diagram above:

1. Project: Maven Project
2. Language: Java
3. Spring Boot: 2.2.0
4. Group: spring.amqp.summit
5. Artifact: spring-amqp-resilient-start
6. Dependencies
   - Spring For RabbitMQ
   - Spring Boot Actuator
   - Spring Web
   - Cloud Connectors

Now download the project by selecting "Generate" and unzip it into a directory of choice (e.g. ~/home/rabbitmq-summit-2019).  You can now import the project into the  IDE of
choice.  If your IDE has project  types make sure you import it as a maven project. 

# Examining and enhancing the dependencies

Open the pom.xml file and notice the four dependencies that you added from Spring Initializr. They are:
that include:

- spring-boot-starter-actuator
- spring-boot-starter-amqp
- spring-boot-starter-cloud-connectors
- spring-boot-starter-web

You will also notice that Spring Initializr adds some default dependencies that will prove very helpful as we develop a spring-amqp project.  They are:
- spring-boot-starter-test
- spring-rabbit-test

Many Spring projects add testing modules specifically for the technology. It simplifies your unit and integration testing.

Finally, we will add a few dependencies that were not supplied by Spring Initializr.  Input the following with your editor into the dependencies section of you maven pom.xml
file as follows:

     <dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-localconfig-connector</artifactId>
    </dependency>
    <dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-cloudfoundry-connector</artifactId>
    </dependency>

We do not include versions because that is handle through spring's curation of dependency management or in the <properties/> section of the parent pom.

# Sending and Consuming Messages

You’ll use the Spring AMQP Template with an infinite loop to send messages. We will use an annotation @EnableScheduling to make that happen. Messages will be sent after every one second.
In addition, we will add some classes for the next few labs because we will not be able to use Spring's AutoConfiguration for all of our setup in the future labs. 

1. Add the following AMQPConnectionFactory class:

```java
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.amqp.rabbit.connection.AbstractConnectionFactory;
    import org.springframework.amqp.rabbit.connection.ConnectionFactory;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
    import org.springframework.cloud.config.java.AbstractCloudConfig;
    import org.springframework.cloud.config.java.ServiceConnectionFactory;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.context.annotation.Primary;
    import org.springframework.util.Assert;

    import com.rabbitmq.client.impl.MicrometerMetricsCollector;

    import io.micrometer.core.instrument.MeterRegistry;
    import io.micrometer.core.instrument.Tag;
    import io.micrometer.core.instrument.Tags;
    import io.micrometer.core.instrument.binder.MeterBinder;

    @Configuration
    public class AMQPConnectionFactoryConfig extends AbstractCloudConfig {
	private Logger logger = LoggerFactory.getLogger(AMQPConnectionFactoryConfig.class);

	@Autowired
	MeterRegistry meterRegistry;

	@Value("${spring.application.name}")
	String applicationName;

	@Bean("consumer")
	@Primary
	public ConnectionFactory rabbitFactory() {
	    ServiceConnectionFactory scf = connectionFactory();
	    return configureMetricCollections(scf.rabbitConnectionFactory(), "consumer");
	}

	@Bean("producer")
	public org.springframework.amqp.rabbit.connection.ConnectionFactory producer(ConnectionFactory consumerConnectionFactory) {
	    logger.info("Creating producer Spring ConnectionFactory ...");
	    ServiceConnectionFactory scf = connectionFactory();
	    return configureMetricCollections(scf.rabbitConnectionFactory().getPublisherConnectionFactory(), "producer");
	}

	@Bean
	MeterRegistryCustomizer<MeterRegistry> commonTags() {
	    return r -> r.config().commonTags(
		    "cf-app-name", cloud().getApplicationInstanceInfo().getAppId(),
		    "cf-app-id", cloud().getApplicationInstanceInfo().getInstanceId(),
		    "app-name", applicationName,
		    "cf-space-id", (String)cloud().getApplicationInstanceInfo().getProperties().get("space_id"));
	}

	private ConnectionFactory configureMetricCollections(ConnectionFactory cf, String connectionName) {
	    if (cf instanceof AbstractConnectionFactory) {
		AbstractConnectionFactory acf = (AbstractConnectionFactory)cf;
		RabbitMetrics rabbitMetrics = new RabbitMetrics(acf.getRabbitConnectionFactory(), connectionName, null);
		rabbitMetrics.bindTo(meterRegistry);
	    }
	    return cf;
	}
    }

    class RabbitMetrics implements MeterBinder {
	private final Iterable<Tag> tags;
	private final com.rabbitmq.client.ConnectionFactory connectionFactory;
	private String name;

	RabbitMetrics(com.rabbitmq.client.ConnectionFactory connectionFactory, String name, Iterable<Tag> tags) {
	    Assert.notNull(connectionFactory, "ConnectionFactory must not be null");
	    this.name = name;
	    this.connectionFactory = connectionFactory;
	    this.tags = tags != null ? Tags.concat(tags, "name", name) : Tags.of("name", name);
	}

	public void bindTo(MeterRegistry registry) {
	    this.connectionFactory.setMetricsCollector(new MicrometerMetricsCollector(registry, "rabbitmq", this.tags));
	}
    }
```

2. Add the following AMQPResourceConfig.java class:

```java
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.amqp.rabbit.connection.AbstractConnectionFactory;
    import org.springframework.amqp.rabbit.connection.ConnectionFactory;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
    import org.springframework.cloud.config.java.AbstractCloudConfig;
    import org.springframework.cloud.config.java.ServiceConnectionFactory;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.context.annotation.Primary;
    import org.springframework.util.Assert;

    import com.rabbitmq.client.impl.MicrometerMetricsCollector;

    import io.micrometer.core.instrument.MeterRegistry;
    import io.micrometer.core.instrument.Tag;
    import io.micrometer.core.instrument.Tags;
    import io.micrometer.core.instrument.binder.MeterBinder;

    @Configuration
    public class AMQPConnectionFactoryConfig extends AbstractCloudConfig {
	private Logger logger = LoggerFactory.getLogger(AMQPConnectionFactoryConfig.class);

	@Autowired
	MeterRegistry meterRegistry;

	@Value("${spring.application.name}")
	String applicationName;

	@Bean("consumer")
	@Primary
	public ConnectionFactory rabbitFactory() {
	    ServiceConnectionFactory scf = connectionFactory();
	    return configureMetricCollections(scf.rabbitConnectionFactory(), "consumer");
	}

	@Bean("producer")
	public org.springframework.amqp.rabbit.connection.ConnectionFactory producer(ConnectionFactory consumerConnectionFactory) {
	    logger.info("Creating producer Spring ConnectionFactory ...");
	    ServiceConnectionFactory scf = connectionFactory();
	    return configureMetricCollections(scf.rabbitConnectionFactory().getPublisherConnectionFactory(), "producer");
	}

	@Bean
	MeterRegistryCustomizer<MeterRegistry> commonTags() {
	    return r -> r.config().commonTags(
		    "cf-app-name", cloud().getApplicationInstanceInfo().getAppId(),
		    "cf-app-id", cloud().getApplicationInstanceInfo().getInstanceId(),
		    "app-name", applicationName,
		    "cf-space-id", (String)cloud().getApplicationInstanceInfo().getProperties().get("space_id"));
	}

	private ConnectionFactory configureMetricCollections(ConnectionFactory cf, String connectionName) {
	    if (cf instanceof AbstractConnectionFactory) {
		AbstractConnectionFactory acf = (AbstractConnectionFactory)cf;
		RabbitMetrics rabbitMetrics = new RabbitMetrics(acf.getRabbitConnectionFactory(), connectionName, null);
		rabbitMetrics.bindTo(meterRegistry);
	    }
	    return cf;
	}
    }
    class RabbitMetrics implements MeterBinder {
	private final Iterable<Tag> tags;
	private final com.rabbitmq.client.ConnectionFactory connectionFactory;
	private String name;

	RabbitMetrics(com.rabbitmq.client.ConnectionFactory connectionFactory, String name, Iterable<Tag> tags) {
	    Assert.notNull(connectionFactory, "ConnectionFactory must not be null");
	    this.name = name;
	    this.connectionFactory = connectionFactory;
	    this.tags = tags != null ? Tags.concat(tags, "name", name) : Tags.of("name", name);
	}

	public void bindTo(MeterRegistry registry) {
	    this.connectionFactory.setMetricsCollector(new MicrometerMetricsCollector(registry, "rabbitmq", this.tags));
	}
    }
```

3. Add the @EnableScheduling to our application class. Then Create a `RabbitTemplate` Spring bean and link it to the `ConnectionFactory`. It looks like the following:

```java
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.amqp.core.MessageBuilder;
    import org.springframework.amqp.rabbit.annotation.Exchange;
    import org.springframework.amqp.rabbit.annotation.Queue;
    import org.springframework.amqp.rabbit.annotation.QueueBinding;
    import org.springframework.amqp.rabbit.annotation.RabbitListener;
    import org.springframework.amqp.rabbit.core.RabbitTemplate;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.scheduling.annotation.EnableScheduling;
    import org.springframework.scheduling.annotation.Scheduled;

    @SpringBootApplication
    @EnableScheduling
    public class SpringAmqpResilientApplication {
	    private Logger logger = LoggerFactory.getLogger(SpringAmqpResilientApplication.class);

	    **@Autowired**
	    **RabbitTemplate template;**

	    @Scheduled(fixedRate = 5000)
	    public void sendMessage() {
		    try {
			    String body = String.format("hello @ %d", System.currentTimeMillis());
			    template.send(MessageBuilder.withBody(body.getBytes()).build());
		    }catch(RuntimeException e) {
			    logger.warn("Failed to send message due to {}", e.getMessage());
		    }
	    }

	    @RabbitListener(bindings = @QueueBinding(
			    value = @Queue,
			    exchange = @Exchange(name = "amq.direct"),
			    key = "hello"))
	    public void listenForHelloMessages(byte[] in) {
		    System.out.println(new String(in));
	    }
	    public static void main(String[] args) {
		    SpringApplication.run(SpringAmqpResilientApplication.class, args);
	    }

    }
```    
7. Execute the program.

