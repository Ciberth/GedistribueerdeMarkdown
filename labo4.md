# Sockts (TicTacToe)

## TicTacToe_Game project

### Game.java

```java

class Game implements Runnable {

    static final Character X = 'X';
    static final Character O = 'O';
    Map<Character, Player> players;
    BlockingQueue list;

    private final Player[] board = {
        null, null, null,
        null, null, null,
        null, null, null};

    Character currentPlayer;

    Game(BlockingQueue list, Player playerX, Player playerO) {
        players = new HashMap<>();
        players.put(X, playerX);
        playerX.start(X);
        players.put(O, playerO);
        playerO.start(O);
        this.list = list;
    }

    /**
     * Returns whether the current state of the board is indicating the victory
     * of any of the players.
     */
    public boolean hasWinner() {
        return (board[0] != null && board[0] == board[1] && board[0] == board[2])
                || (board[3] != null && board[3] == board[4] && board[3] == board[5])
                || (board[6] != null && board[6] == board[7] && board[6] == board[8])
                || (board[0] != null && board[0] == board[3] && board[0] == board[6])
                || (board[1] != null && board[1] == board[4] && board[1] == board[7])
                || (board[2] != null && board[2] == board[5] && board[2] == board[8])
                || (board[0] != null && board[0] == board[4] && board[0] == board[8])
                || (board[2] != null && board[2] == board[4] && board[2] == board[6]);
    }

    /**
     * Returns whether there are no more empty squares.
     */
    public boolean boardFilledUp() {
        for (Player board1 : board) {
            if (board1 == null) {
                return false;
            }
        }
        return true;
    }

    /**
     * Called by the player threads when a player tries to make a move. This
     * method checks to see if the move is legal: that is, the player requesting
     * the move must be the current player and the square in which he/she is
     * trying to move must not already be occupied. If the move is legal the
     * game state is updated (the square is set and the next player becomes
     * current) and the other player is notified of the move so it can update
     * its client.
     */
    public boolean legalMove(int location, Player player) {
        if (location >= 0 && location < 9 && board[location] == null) {
            board[location] = player;
            return true;
        }
        return false;
    }

    @Override
    public void run() {
        currentPlayer = X;
        Player player = players.get(currentPlayer);
        int location;
        boolean hasWinner = false;
        boolean boardFilledUp = false;
        boolean end = false;
        // Repeatedly get commands from the players and process them.
        do {
            // Tell the player that it is her turn.
            player.send("MESSAGE Your move");
            location = player.move();

            if (legalMove(location, player)) {
                player.send("VALID_MOVE");
                hasWinner = hasWinner();
                boardFilledUp = boardFilledUp();
                player.send(hasWinner ? "VICTORY"
                        : boardFilledUp ? "TIE"
                                : "");
                currentPlayer = currentPlayer == X ? O : X;
                player = players.get(currentPlayer);
                player.otherPlayerMoved(location);
                player.send(hasWinner ? "DEFEAT" : boardFilledUp ? "TIE" : "");
            } else if (location == -1) {
                player.close();
                end = true;
                currentPlayer = currentPlayer == X ? O : X;
                player = players.get(currentPlayer);
                player.send("QUIT");
            } else {
                player.send("MESSAGE ?");
            }
        } while (!boardFilledUp && !hasWinner && !end);

        if (end) {
            if (player.more()) {
                list.offer(player);
            }
        } else {
            for (Player p : players.values()) {
                if (p.more()) {
                    list.offer(p);
                }
            }
        }
    }
}


```

### Player

```java

class Player extends Thread implements AutoCloseable {

    char mark;
    Player opponent;
    Socket socket;
    BufferedReader input;
    PrintWriter output;

    public Player(Socket socket) throws IOException {
        this.socket = socket;
        input = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
        output = new PrintWriter(socket.getOutputStream(), true);
        output.println("MESSAGE Waiting for opponent to connect");
    }

    public void setOpponent(Player opponent) {
        this.opponent = opponent;
    }

    public void otherPlayerMoved(int location) {
        output.println("OPPONENT_MOVED" + location);
    }

    public void start(char mark) {
        this.mark = mark;
        output.println("WELCOME " + mark);
        // The thread is only started after everyone connects.
        output.println("MESSAGE All players connected");
    }

    protected int move() {
        int location = -1;
        try {
            String command = input.readLine();
            if (command.startsWith("MOVE")) {
                location = Integer.parseInt(command.substring(5));
            }
        } catch (IOException ex) {
            Logger.getLogger(Player.class.getName()).log(Level.SEVERE, null, ex);
        }
        return location;
    }

    protected void send(String bericht) {
        output.println(bericht);
    }

    protected boolean more() {
        boolean more = false;
        try {
            String command = input.readLine();
            if (command.startsWith("MOVE")) {
                input.readLine();
            }
            more = command.startsWith("CONTINUE");
        } catch (IOException ex) {
            Logger.getLogger(Player.class.getName()).log(Level.SEVERE, null, ex);
        }
        return more;
    }

    @Override
    public void close() {
        try {
            socket.close();
        } catch (IOException ex) {
            Logger.getLogger(Player.class.getName()).log(Level.SEVERE, null, ex);
        }
    }

}


```


### PlayerListener.java

```java

public class PlayerListener implements Runnable {

    private final AtomicBoolean active;
    private BlockingQueue<Player> players;
    ExecutorService execServ;
    

    PlayerListener(BlockingQueue<Player> list) {
        this.players = list;
        active = new AtomicBoolean(true);
    }


    @Override
    public void run() {
        execServ = Executors.newFixedThreadPool(10);
        while (active.get()) {
            try {
                Player player1 = players.take();
                Player player2 = players.take();
                Game game = new Game(players, player1, player2);
                execServ.submit(game);
            } catch (InterruptedException ex) {
                Logger.getLogger(PlayerListener.class.getName()).log(Level.SEVERE, null, ex);
            }
        }
        execServ.shutdown();
    }
}

```

### Listener.java

```java

public class Listener implements Runnable {

    // Server is listening at port '8901'
    private static final int POORT = 8901;
    private ServerSocket listener;
    private AtomicBoolean active;
    BlockingQueue<Player> players;

    Listener(BlockingQueue<Player> players) {
        this.players = players;
        try {
            listener = new ServerSocket(POORT);
            //listener.setSoTimeout(1000);
            System.out.println("Tic Tac Toe Server is up  ...");
            active = new AtomicBoolean(true);
        } catch (IOException ex) {
            Logger.getLogger(Listener.class.getName()).log(Level.SEVERE, null, ex);
            throw new RuntimeException(ex);
        }
    }

    @Override
    public void run() {

        while (active.get()) {
            try {
                Player player = new Player(listener.accept());
                players.add(player);
            } catch (IOException ex) {
                Logger.getLogger(ServerThread.class.getName()).log(Level.SEVERE, null, ex);
            }
        }
    }

}


```

### ServerThread.java

```java

public class ServerThread implements Runnable {

    private final BlockingQueue<Player> players;
    PlayerListener gamesThread;
    Listener listener;

    public ServerThread() {
        players = new LinkedBlockingQueue<>();
        gamesThread = new PlayerListener(players);
        listener = new Listener(players);
    }

    @Override
    public void run() {
        new Thread(gamesThread).start();
        new Thread(listener).start();
    }
}


```

### TicTacToeServer.java

```java

/**
 * A server for a network multi-player tic tac toe game. Here are the strings
 * that are sent for communication in the Tic Tac Toe game:
 *
 * Client -> Server Server -> Client ---------------- ---------------- MOVE <n>
 * (0 <= n <= 8) WELCOME <char> (char in {X, O}) QUIT VALID_MOVE CONTINUE
 * OTHER_PLAYER_MOVED <n>
 * VICTORY DEFEAT TIE MESSAGE <text>
 * QUIT
 *
 * This server allows an 10 pairs of players to play.
 */
public class TicTacToeServer {

    /**
     * Runs the application. Pairs up clients that connect.
     *
     * @param args
     * @throws java.lang.Exception
     */
    public static void main(String[] args) throws Exception {

        ServerThread ticTacToe = new ServerThread();
        Thread ticTacToeThread = new Thread(ticTacToe);
        ticTacToeThread.start();
        Scanner sc = new Scanner(System.in);
        if (sc.nextLine().equals("quit")) {
            System.exit(0);
        }
    }
}

```

### TicTacToeFrame.java

```java

public class TicTacToeFrame extends JFrame {
    
    private final JLabel messageLabel = new JLabel("");
    
    private static char PlayerMark;
    private static char OpponentMark;
    private Color PlayerColor;
    private Color OpponentColor;
    private Square[] board = new Square[9];
    private Square currentSquare;
    private int currentPosition;
    private final Font LabelFont = new Font("Arial", Font.BOLD, 16);
    
    
    public TicTacToeFrame() throws HeadlessException {
        super("Tic Tac Toe");
        // Layout GUI
        messageLabel.setBackground(Color.lightGray);
        getContentPane().add(messageLabel, "South");
        JPanel boardPanel = new JPanel();
        boardPanel.setBackground(Color.black);
        boardPanel.setLayout(new GridLayout(3, 3, 2, 2));
        for (int i = 0; i < board.length; i++) {
            final int j = i;
            board[i] = new Square();
            board[i].addMouseListener(new MouseAdapter() {
                @Override
                public void mousePressed(MouseEvent e) {
                    // current square is the square clicked by the mouse
                    currentSquare = board[j];
                    currentPosition = j;
                    // notify ...
                }
            });
            boardPanel.add(board[i]);
        }
        getContentPane().add(boardPanel, "Center");
    }
    
    protected void setMark(char mark) {
        PlayerMark = mark == 'X' ? 'X' : 'O';
        PlayerColor = mark == 'X' ? Color.RED : Color.BLUE;
        OpponentMark = mark == 'X' ? 'O' : 'X';
        OpponentColor = mark == 'X' ? Color.BLUE : Color.RED;
        setTitle("Tic Tac Toe - Player " + mark);
    }
    
    protected void validMove() {
        messageLabel.setText("Valid move --> PLEASE WAIT");
        currentSquare.label.setFont(LabelFont);
        currentSquare.label.setForeground(PlayerColor);
        currentSquare.label.setText(Character.toString(PlayerMark));
    }
    
    protected void opponentMoved(int loc) {
        messageLabel.setText("Opponent moved --> YOUR TURN");
        board[loc].label.setFont(LabelFont);
        board[loc].label.setForeground(OpponentColor);
        board[loc].label.setText(Character.toString(OpponentMark));
    }
    
    protected void bericht(String tekst) {
        messageLabel.setText(tekst);
    }
    
    protected int getPosition() {
        return currentPosition;
    }
    
  

    /**
     * Graphical square in the client window. Each square is a white panel
     * containing an X or O.
     */
    class Square extends JPanel {
        
        JLabel label = new JLabel();
        
        public Square() {
            setBackground(Color.WHITE);
            add(label);
        }
    }
}


```