# Snippets

## Webservices en Handlers in java

```java

```



```java
@WebService(serviceName = "Catalogus")
public class Catalogus {

    @WebMethod(operationName = "hello")
    public String hello(@WebParam(name = "name") String txt) {
        return "Hello " + txt + " !";
    }
    
    @WebMethod(operationName = "geefBoek")
    public Boek geefBoek(@WebParam(name = "isbn") String isbn) {
        //..
    }
}

```


### EquationService

```java

@WebService(serviceName = "EquationService")
@HandlerChain(file = "EquationService_handler.xml")
public class EquationService {

    @WebMethod(operationName = "hello")
    public String hello(@WebParam(name = "name") String txt) {
        return "Hello " + txt + " !";
    }
    
    @WebMethod(operationName = "bepaalNulpunten")
    public double[] solveQuadratic(
            @WebParam(name = "c1") double a, 
            @WebParam(name = "c2") double b, 
            @WebParam(name = "c3") double c) {
         // solve a x^2 + b x + c
            }
    }


```

### EquationService_handler

```xml

<?xml version="1.0" encoding="UTF-8"?>
<handler-chains xmlns="http://java.sun.com/xml/ns/javaee">
  <handler-chain>
    <handler>
      <handler-name>ws.TestLogicalHandler</handler-name>
      <handler-class>ws.TestLogicalHandler</handler-class>
    </handler>
<!--    <handler>
      <handler-name>ws.TestMessageHandler</handler-name>
      <handler-class>ws.TestMessageHandler</handler-class>
    </handler>-->
  </handler-chain>
</handler-chains>


```













## JMS

### AsynchConsumer

```java

    @Resource(lookup = "jms/__defaultConnectionFactory")
    private static ConnectionFactory connectionFactory;

    @Resource(lookup = "jms/MyQueue")
    private static Queue queue;
    
    @Resource(lookup = "jms/MyTopic")
    private static Topic topic;

            /*
             * In a try-with-resources block, create context.
             * Create consumer.
             * Register message listener (TextListener).
             * Receive text messages from destination.
             * When all messages have been received, enter Q to quit.
             */
             
            try (JMSContext context = connectionFactory.createContext();) {
                JMSConsumer consumer = context.createConsumer(dest);
                TextListener listener = new TextListener();
                consumer.setMessageListener(listener);
                System.out.println("To end program, enter Q or q, then <return>");
                InputStreamReader inputStreamReader = new InputStreamReader(System.in);

                char answer = '\0';

                while (!((answer == 'q') || (answer == 'Q'))) {
                    try {
                        answer = (char) inputStreamReader.read();
                    } catch (IOException e) {
                        System.err.println("I/O exception: " + e.toString());
                    }
                }
            }

```

### DurableConsumer

```java

    @Resource(lookup = "jms/DurableConnectionFactory")
    private static ConnectionFactory durableConnectionFactory;
    
    @Resource(lookup = "jms/MyTopic")
    private static Topic topic;
    
    public static void main(String[] args) {        
        TextListener listener;
        JMSConsumer consumer;

        InputStreamReader inputStreamReader;
        char answer = '\0';

        /*
         * In a try-with-resources block, create context.
         * Create durable consumer, if it does not exist already.
         * Register message listener (TextListener).
         * Receive text messages from destination.
         * When all messages have been received, enter Q to quit.
         */
        try (JMSContext context = durableConnectionFactory.createContext();) {
            System.out.println("Creating consumer for topic");
            //context.stop();
            consumer = context.createDurableConsumer(topic, "MakeItLast");
            listener = new TextListener();
            consumer.setMessageListener(listener);
            System.out.println("Starting consumer");
            //context.start();

            System.out.println("To end program, enter Q or q, then <return>");
            inputStreamReader = new InputStreamReader(System.in);
            
            //while(..)
        }
    }

``` 

### MessageBrowser

```java

@Resource(lookup = "java:comp/DefaultJMSConnectionFactory")
    private static ConnectionFactory connectionFactory;
    
    @Resource(lookup = "jms/MyQueue")
    private static Queue queue;

   
    public static void main(String[] args) {

        /*
         * In a try-with-resources block, create context.
         * Create QueueBrowser.
         * Check for messages on queue.
         */
        try (JMSContext context = connectionFactory.createContext();) {
            QueueBrowser browser = context.createBrowser(queue);
            Enumeration msgs = browser.getEnumeration();
        }
    }

```


### Producer

```java

    @Resource(lookup = "jms/__defaultConnectionFactory")
    private static ConnectionFactory connectionFactory;
    
    @Resource(lookup = "jms/MyQueue")
    private static Queue queue;
    
    @Resource(lookup = "jms/MyTopic")
    private static Topic topic;


    String destType; // args[0]
    Destination dest;
    // dest = queue of topic

            /*
             * Within a try-with-resources block, create context.
             * Create producer and message.
             * Send messages, varying text slightly.
             * Send end-of-messages message.
             */
            try (JMSContext context = connectionFactory.createContext();) {
                int count = 0;

                for (int i = 0; i < NUM_MSGS; i++) {
                    String message = "This is message " + (i + 1)
                            + " from producer";
                    // Comment out the following line to send many messages
                    System.out.println("Sending message: " + message);
                    context.createProducer().send(dest, message);
                    count += 1;
                }
                System.out.println("Text messages sent: " + count);

                /*
                 * Send a non-text control message indicating end of
                 * messages.
                 */
                context.createProducer().send(dest, context.createMessage());
                // Uncomment the following line if you are sending many messages
                // to two synchronous consumers
                //context.createProducer().send(dest, context.createMessage());
            }
            
```

### SynchConsumer

```java

    @Resource(lookup = "jms/__defaultConnectionFactory")
    private static ConnectionFactory connectionFactory;
    
    @Resource(lookup = "jms/MyQueue")
    private static Queue queue;
    
    @Resource(lookup = "jms/MyTopic")
    private static Topic topic;

    String destType; // something (args[0])
    Destination dest; // = queue or topic

    /*
             * In a try-with-resources block, create context.
             * Create consumer.
             * Receive all text messages from destination until
             * a non-text message is received indicating end of
             * message stream.
             */
            try (JMSContext context = connectionFactory.createContext();) {

                JMSConsumer consumer = context.createConsumer(dest);
                int count = 0;

                while (true) {
                    Message m = consumer.receive(1000);

                    if (m != null) {
                        if (m instanceof TextMessage) {
                            // Comment out the following two lines to receive
                            // a large volume of messages
                            System.out.println(
                                    "Reading message: " + m.getBody(String.class));
                            count += 1;
                        } else {
                            break;
                        }
                    }
                }
            }
```

### Unsubscriber

```java

public class Unsubscriber {
    
    @Resource(lookup = "jms/DurableConnectionFactory")
    private static ConnectionFactory durableConnectionFactory;

    public static void main(String[] args) {
        try (JMSContext context = durableConnectionFactory.createContext();) {
            System.out.println("Unsubscribing from durable subscription");
            context.unsubscribe("MakeItLast");
        } catch (JMSRuntimeException e) {
            System.err.println("Exception occurred: " + e.toString());
            System.exit(1);
        }
        System.exit(0);
    }
    
}


```