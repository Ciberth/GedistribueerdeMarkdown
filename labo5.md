# JMS en NIO

2 Projecten

## QuizServer Project 

### Question.java

```java

public class Question {

    private final int id;
    private final String question, answer;

    public Question(int id, String question, String answer) {
        this.id = id;
        this.question = question;
        this.answer = answer;
    }

    public int getId() {
        return id;
    }

    public String getQuestion() {
        return question;
    }

    public boolean isCorrect(String myAnswer) {
        return answer.equals(myAnswer);
    }

}


```

### Quiz.java

```java

public class Quiz {

    List<Question> questions;
    Map<Integer,Question> allQuestions;
    Random r;

    public Quiz() {
        questions = new ArrayList<>();
        allQuestions = new HashMap<>();
        r = new Random();
        try {
            RowSetFactory myRowSetFactory = RowSetProvider.newFactory();
            CachedRowSet quizRS = myRowSetFactory.createCachedRowSet();

            quizRS.setDataSourceName("jdbc/quiz"); // <-- glassfish stuff!

            quizRS.setCommand("select id, question, answer from quiz");
            quizRS.execute();
            while (quizRS.next()) {
                Question q = new Question(quizRS.getInt(1), quizRS.getString(2), quizRS.getString(3));
                questions.add(q);
                allQuestions.put(q.getId(), q);
            }
            Logger.getLogger(Quiz.class.getName()).log(Level.INFO, "aantal vragen: {0}", questions.size());
        } catch (SQLException ex) {
            Logger.getLogger(Quiz.class.getName()).log(Level.SEVERE, null, ex);
        }
    }

    public Question getQuestion() {
        Question q = null;
        if (questions.size() > 0) {
            int index = r.nextInt(questions.size());
            q = questions.remove(index);
        }
        return q;
    }

    public boolean isCorrect(int id, String answer) {
        Question q = allQuestions.get(id);
        return q.isCorrect(answer);
    }
}


```

### AskQuestion.java

```java

public class AskQuestion extends TimerTask {
    Selector inputSelector;

    AskQuestion(Selector inputSelector) {
        this.inputSelector = inputSelector;
    }

    @Override
    public void run() {
        inputSelector.wakeup();
    }
    
}

```


### QuizServer.java

```java

public class QuizServer {
    
    private static final int PORT = 7777;
    @Resource(lookup = "java:comp/DefaultJMSConnectionFactory")
    private static ConnectionFactory connectionFactory;
    @Resource(lookup = "jms/MyQueue")
    private static Queue queue;

    
    public static void main(String[] args) {
        try {
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.socket().bind(new InetSocketAddress(PORT));
            QuizThread cs = new QuizThread(connectionFactory, queue, serverSocketChannel);
            Thread csThread = new Thread(cs);
            csThread.start();
            Scanner sc = new Scanner(System.in);
            if (sc.nextLine().equals("quit")) {
                cs.stop();
            }
        } catch (IOException ex) {
            Logger.getLogger(QuizServer.class.getName()).log(Level.SEVERE, null, ex);
        }
    }
    
}

```

### QuizThread.java

```java

public class QuizThread implements Runnable {

    private static final int BUFFER_SIZE = 255;
    private static final int MESSAGE_SIZE = 500;
    private static final int INTERVAL = 60 * 1000;
    private final ServerSocketChannel serverSocketChannel;
    private Selector inputSelector;
    private boolean running;
    private final LinkedList<SocketChannel> clients;
    private final CharsetDecoder asciiDecoder;
    private final Queue queue;
    private final Timer timer;
    private final JMSContext context;
    private final Quiz quiz;

    QuizThread(ConnectionFactory connectionFactory, Queue queue, ServerSocketChannel serverSocketChannel) {
        context = connectionFactory.createContext();
        this.queue = queue;
        this.serverSocketChannel = serverSocketChannel;
        running = true;
        clients = new LinkedList<>();
        asciiDecoder = Charset.forName("US-ASCII").newDecoder();
        timer = new Timer();
        quiz = new Quiz();
    }

    @Override
    public void run() {
        try {
            // get a selector for multiplexing the client channels
            inputSelector = Selector.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.register(inputSelector, SelectionKey.OP_ACCEPT);
            timer.scheduleAtFixedRate(new AskQuestion(inputSelector), 0, INTERVAL);
            while (running) {
                waitForInput();
            }
        } catch (IOException ex) {
            Logger.getLogger(QuizThread.class.getName()).log(Level.SEVERE, null, ex);
        } finally {
            context.close();
        }
    }

    void stop() {
        timer.cancel();
        running = false;
        inputSelector.wakeup();
    }

    private void waitForInput() {
        try {
            int readyChannels = inputSelector.select();
            if (readyChannels != 0) {
                for (SelectionKey selectedKey : inputSelector.selectedKeys()) {
                    if (selectedKey.isAcceptable()) {
                        addNewClient(selectedKey);
                    } else if (selectedKey.isReadable()) {
                        // a channel is ready for reading
                        readIncomingMessages(selectedKey);
                    }
                }
                inputSelector.selectedKeys().clear();
            } else { // timer finished
                sendQuestion();
            }
        } catch (IOException ex) {
            Logger.getLogger(QuizThread.class.getName()).log(Level.SEVERE, null, ex);
        }
    }

    private void addNewClient(SelectionKey key) {
        try {
            ServerSocketChannel channel = (ServerSocketChannel) key.channel();
            SocketChannel clientChannel = channel.accept();
            String newData = "Welkom";
            sendMessage(newData, clientChannel);

            Logger.getLogger(QuizThread.class.getName()).log(Level.INFO, "got connection from: {0}", clientChannel.socket().getInetAddress());

            // add to our list
            clients.add(clientChannel);

            // register the channel with the selector
            // store a new StringBuffer as the Key's attachment for holding partially read messages
            clientChannel.configureBlocking(false);
            clientChannel.register(inputSelector, SelectionKey.OP_READ, new StringBuffer());
        } catch (IOException ex) {
            Logger.getLogger(QuizThread.class.getName()).log(Level.SEVERE, "new client problems ", ex);
        }
    }

    private void sendMessage(String newData, SocketChannel clientChannel) throws IOException {
        ByteBuffer buf = ByteBuffer.allocate(MESSAGE_SIZE);
        buf.clear();
        buf.put(newData.getBytes());
        // send welcome
        buf.flip();
        while (buf.hasRemaining()) {
            clientChannel.write(buf);
        }
    }

    private void readIncomingMessages(SelectionKey key) {

        try {
            SocketChannel channel = (SocketChannel) key.channel();

            // grab the StringBuffer we stored as the attachment
            StringBuffer name = (StringBuffer) key.attachment();

            ByteBuffer readBuffer = ByteBuffer.allocate(BUFFER_SIZE);
            readBuffer.clear();

            // read from the channel into our buffer
            long nbytes = channel.read(readBuffer);

            // check for end-of-stream
            if (nbytes == -1) {
                Logger.getLogger(QuizThread.class.getName())
                        .log(Level.INFO, "disconnect: {0} from {1}, end-of-stream",
                                new Object[]{name, channel.socket().getInetAddress()});
                channel.close();
                clients.remove(channel);
            } else {
                // use a CharsetDecoder to turn those bytes into a string
                // and append to our StringBuffer
                readBuffer.flip();
                String line = asciiDecoder.decode(readBuffer).toString();
                readBuffer.clear();

                if (line.startsWith("QUIT")) {
                    // client is quitting, close their channel, remove them from the list and notify all other clients
                    Logger.getLogger(QuizThread.class.getName())
                            .log(Level.INFO, "got quit msg, closing channel for : {0} from {1}",
                                    new Object[]{name, channel.socket().getInetAddress()});
                    channel.close();
                    clients.remove(channel);
                } else if (name.length() == 0) { // not registerted
                    if (line.startsWith("REGISTER")) {
                        String[] tokens = line.split(" ");
                        name.append(tokens[1]);
                    } else {
                        sendMessage("REGISTER FIRST", channel);
                    }
                } else if (line.startsWith("ANSWER")) { // registered
                    int spatie = line.indexOf(' '); // first space
                    line = line.substring(spatie + 1); // ANSWER removed
                    spatie = line.indexOf(' '); // space between id en answer 
                    String id = line.substring(0, spatie);
                    String answer = line.substring(spatie + 1).trim();
                    registerAnswer(Integer.parseInt(id), name.toString(), answer);
                    Logger.getLogger(QuizThread.class.getName())
                            .log(Level.INFO, "got answer msg for question {0} from {1}: {2}",
                                    new Object[]{id, name, answer});
                } else {
                    Logger.getLogger(QuizThread.class.getName())
                            .log(Level.INFO, "wrong command: {0} from {1}",
                                    new Object[]{name, channel.socket().getInetAddress()});
                    sendMessage("WRONG COMMAND", channel);
                }
            }
        } catch (IOException ioe) {
            Logger.getLogger(QuizThread.class.getName()).log(Level.WARNING, "error during select(): {0}", ioe.getMessage());
        } catch (Exception e) {
            Logger.getLogger(QuizThread.class.getName()).log(Level.SEVERE, "exception in run(){0}", e.getMessage());
        }

    }

    private void sendBroadcastMessage(String mesg) {

        ByteBuffer writeBuffer = ByteBuffer.allocateDirect(BUFFER_SIZE);
        // fills the buffer from the given string
        // and prepares it for a channel write
        writeBuffer.clear();
        writeBuffer.put(mesg.getBytes());
        writeBuffer.putChar('\n');
        writeBuffer.flip();
        for (SocketChannel channel : clients) {
            channelWrite(channel, writeBuffer);
        }
    }

    private void channelWrite(SocketChannel channel, ByteBuffer writeBuffer) {
        long nbytes = 0;
        long toWrite = writeBuffer.remaining();

        // loop on the channel.write() call since it will not necessarily
        // write all bytes in one shot
        try {
            while (nbytes != toWrite) {
                nbytes += channel.write(writeBuffer);

            }
        } catch (IOException ex) {
            Logger.getLogger(QuizThread.class
                    .getName()).log(Level.SEVERE, null, ex);
        }

        // get ready for another write if needed
        writeBuffer.rewind();
    }

    private void sendQuestion() {
        StringBuilder message = new StringBuilder();
        Question question = quiz.getQuestion();
        if (question != null) {
            message.append("QUESTION ");
            int id = question.getId();
            message.append(id);
            message.append(" ");
            message.append(question.getQuestion());
            registerQuestion(id);
            sendBroadcastMessage(message.toString());
        } else {
            stop();
        }
    }

    private void registerQuestion(int id) {
        Message message = context.createTextMessage("SEND QUESTION " + id);
        context.createProducer().send(queue, message);
    }

    private void registerAnswer(int id, String name, String answer) {
        Message message = context.createTextMessage("ANSWER QUESTION " + id + " " + answer + " BY " + name);
        context.createProducer().send(queue, message);
    }
}


```


## CountPoints Project

### CountPoints.java

```java

public class CountPoints {

    @Resource(lookup = "java:comp/DefaultJMSConnectionFactory")
    private static ConnectionFactory connectionFactory;
    @Resource(lookup = "jms/MyQueue")
    private static Queue queue;

    /**
     * @param args the command line arguments
     */
    public static void main(String[] args) {
        /*
         * In a try-with-resources block, create context.
         * Create consumer.
         * Receive all text messages from destination until end of
         * message stream.
         */
        HashMap<String, Integer> result = new HashMap<>();
        int id = 0;
        boolean first = false;
        Quiz quiz = new Quiz();
        HashMap<String, HashMap<Integer, Boolean>> answered = new HashMap<>();
        try (JMSContext context = connectionFactory.createContext();) {
            JMSConsumer consumer = context.createConsumer(queue);

            System.out.println("consumer ready");
            Message m = consumer.receive(10000);
            System.out.println("message received");
            while (m != null) {
                if (m instanceof TextMessage) {
                    String message = m.getBody(String.class);
                    System.out.println(message);
                    if (message.startsWith("SEND QUESTION")) {
                        id = Integer.parseInt(message.substring(14).trim());
                        first = true;
                        System.out.println("Question " + id);
                    } else if (message.startsWith("ANSWER QUESTION")) {
                        message = message.substring(16);
                        int space = message.indexOf(' ');
                        int idQuestion = Integer.parseInt(message.substring(0, space));
                        int by = message.indexOf("BY");
                        String answer = message.substring(space + 1, by);
                        String name = message.substring(by + 3).trim();
                        if (id == idQuestion && quiz.isCorrect(id, answer)) {
                            int point = 1;
                            if (first) {
                                point = 5;
                                first = false;
                            }
                            Integer count = 0;
                            if (!answered.containsKey(name)) {
                                answered.put(name, new HashMap<>());
                            }
                            if (result.containsKey(name)) {
                                count = result.get(name);
                            }
                            if (!answered.get(name).containsKey(id)) {
                                count += point;
                                System.out.println("Answer question " + id + " by " + name + ": " + point);
                                result.put(name, count);
                            }
                        }
                    }
                }
                m = consumer.receive(10000);
            }
        } catch (JMSException e) {
            System.err.println("Exception occurred: " + e.toString());
            System.exit(1);
        }
        for (String name : result.keySet()) {
            System.out.println(name + ": " + result.get(name) + " points");
        }
        System.exit(0);

    }

}


```
