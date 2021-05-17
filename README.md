- 👋 Hi, I’m @ObamaTheCoder
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
ObamaTheCoder/ObamaTheCoder is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
ClientController:

public void connectClicked(ActionEvent actionEvent) {
        s.setOnCloseRequest(new EventHandler<WindowEvent>() {
            @Override
            public void handle(WindowEvent windowEvent) {
                SocketMessage m = new SocketMessage(client, ">>DISCONNECT;");
                client.sendMessage(m);
            }
        });


        client = new SocketConnection(hostTextField.getText(), Integer.parseInt(portTextField.getText()));
        //client.setUsername(usernameTextField.getText());

        client.addSubscriber(new ISocketSubscriber() {
            @Override
            public void messageReceived(ISocketMessage msg) {
                if (msg.getMessage().contains(">>USERS;")) {

                    ObservableList<String> dataObeservable =
                            FXCollections.observableArrayList();

                    //hier drinn gehts wenn USERS nachricht zurückgeschickt wurde also jemand is gejoint/geleaved
                    String[] tmp = msg.getMessage().split(";");
                    String users = "";

                    //da muss tmp.length - 1

                    int i = 1; //weil 0 ja ">>USERS;" ist
                    while (i < tmp.length) {
                        dataObeservable.add(tmp[i]); //add damit index-out-of-bounds ned kommt und
                        ++i;                            //weil ich sowieso immer neues dataObservable mach
                    }

                    usersListView.setItems(dataObeservable); //@todo es kommt ein "FX application thread" error aber es funktioniert

                } else {
                    chatTextArea.appendText(msg.getMessage() + "\n");
                }
            }
        });

        if (client != null) {
            client.setUsername(usernameTextField.getText());
            client.start(); //start receiving after setting the username

            SocketMessage m = new SocketMessage(client, ">>USERNAME;" + usernameTextField.getText());
            client.sendMessage(m);

            hostTextField.setDisable(true);
            portTextField.setDisable(true);
            usernameTextField.setDisable(true);
        }
    }
        public void sendClicked(ActionEvent actionEvent) {
        SocketMessage m = new SocketMessage(client, messageTextArea.getText());
        client.sendMessage(m);
        messageTextArea.clear();

    }
}
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
ChatServer.java


public class ChatServer implements ISocketServer {
    private String host = "";
    private int port = 0;

    private ArrayList<SocketConnection> clients = new ArrayList<>();
    private ServerSocket server = null;
    private boolean running = true;

    public ChatServer(int port) {
        this.port = port;
    }

    @Override
    public void start() throws IOException {
        server = new ServerSocket(port);
        running = true;
        handleConnections();
    }

    @Override
    public void stop() {
        running = false;
        try {
            server.close();
        } catch (IOException e) {
        }
    }
    //--------------------------//
    //-----Server Functions-----//
    //--------------------------//

    void sendPrivateMessage(ISocketMessage msg) {

        //"@nutzer hallo" da wird immer ein abstand sein also passt eh
        //beim ersten Split wird die stelle vor dem " " genommen weil da in der message das "@nutzer" steht
        //und dann da splitte ich das @ damit ich nur den nutzername habe an den es geht und das wird
        //in diesem String gespeichert.
        String recieverName = (msg.getMessage().split(" "))[0].split("@")[1];

        //nimmt die msg und replaced den teil bis zum ersten abstand mit "" bzw. nixs
        String privMessage = msg.getMessage().replace(msg.getMessage().substring(0, msg.getMessage().indexOf(" ")), "");

        if (privMessage.length() < 1) {
            System.out.println("failed private Message! Reason: No Message given");
        } else {
            int i = 0;


            //@todo hier gibt es nen bug das wenn man zu schnell hier des durchlauft das manachmal bei beiden usern
            //@todo steht "[PRIVAT von: senderName]: msg"
            while (i < clients.size()) {    //schleife zum suchen des nutzer an den es geht
                if (clients.get(i).getUsername().equals(recieverName)) {
                    //msg an reciever
                    msg.setMessage("[PRIVAT von: " + msg.getSource().getUsername() + "]: " + privMessage);
                    clients.get(i).sendMessage(msg);

                    //msg an sender
                    msg.setMessage("[PRIVAT an: " + clients.get(i).getUsername() + "]: " + privMessage);
                    msg.getSource().sendMessage(msg);
                }
                ++i;
            }
        }

    }

    void sendServerMessage(ISocketMessage msg) {
        String[] befehl = msg.getMessage().split(";");// 0 = der Befehl; 1 = info die dazugegeben wurde

        if (befehl[0].equals(">>USERNAME")) {   //wenn jemand sich verbindet und den nutzernamen setzt
            msg.getSource().setUsername(befehl[1]);
        } else if (msg.getMessage().equals(">>DISCONNECT;")) { //wenn jemand disconnected
            msg.getSource().stop();
            clients.remove(msg.getSource());
        }

        //liste aller User wird geschickt, da in beiden fällen eine geupdatete liste benötigt wird
        //(DISCONNECT --> Nutzer muss weg) (USERNAME --> Nutzer kommt dazu)
        String s = ">>USERS;";
        int i = 0;
        while (i < clients.size()) {
            s += clients.get(i).getUsername() + ";";
            ++i;
        }

        //Message an jeden Nutzer schicken
        //"null" weil die message kommt vom server und als source soll ned irgendein nutzer sein
        SocketMessage serverMSG = new SocketMessage(null, s);
        for (SocketConnection c : clients) {
            c.sendMessage(serverMSG);
        }
    }

    void sendNormalMessage(ISocketMessage msg) {
        SocketMessage m = new SocketMessage(msg.getSource(), msg.getSource().getUsername() + ": " + msg.getMessage());

        //nachricht wird an jeden gesendet
        for (SocketConnection c : clients) {
            c.sendMessage(m);
        }
        //log im server
        System.out.println("Nachricht erhalten:" + msg.getMessage());
    }

    //--------------------------//
    //--------------------------//
    //--------------------------//

    @Override
    public void handleConnections() {
        ISocketSubscriber broadcastSubscriber = new ISocketSubscriber() {
            @Override
            public void messageReceived(ISocketMessage msg) {

                //@todo maybe mach ein disconnect button? idk wenn dir fad is mach

                if (msg.getMessage().startsWith("@")) { //private Message
                    sendPrivateMessage(msg);
                } else if (msg.getMessage().startsWith(">>")) { //server Befehl
                    sendServerMessage(msg);
                } else { //normale nachricht
                    sendNormalMessage(msg);
                }
            }
        };

        new Thread() {
            @Override
            public void run() {
                while (running) {
                    try {
                        System.out.println("Server startet! Listening for incoming connections...");
                        while (true) {
                            Socket s = server.accept();
                            System.out.println("New incoming connection: " + s.getInetAddress().getHostAddress()); //ip addresse ausgeben vom Client
                            SocketConnection client = new SocketConnection(s);

                            //client.sendMessage("<--- IP: " + s.getInetAddress().getHostAddress() + " || Name: <" + client.getUsername() + "> joined --->");

                            client.addSubscriber(broadcastSubscriber); // add broadcast - subscriber to client!

                            client.start();

                            clients.add(client); // add client to our list of clients!
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }   
            }
        }.start();
    }

    public static void main(String[] args) throws IOException {
        ChatServer myServer = new ChatServer(2020);
        myServer.start();
    }
}
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
SocketMessage.java


public class SocketMessage implements ISocketMessage {
    private ISocketConnection source = null;
    private String message = "";

    public SocketMessage(ISocketConnection source, String message) {
        this.source = source;
        this.message = message;
    }

    @Override
    public String getMessage() {
        return message;
    }

    @Override
    public void setMessage(String msg) {
        this.message = msg;
    }

    @Override
    public ISocketConnection getSource() {
        return source;
    }

    @Override
    public void setSource(ISocketConnection src) {
        this.source = src;
    }
}

------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
SocketConnection.java


public class SocketConnection implements ISocketConnection, ISocketPublisher {
    private ExecutorService executor = Executors.newSingleThreadExecutor();

    private int port = 0;
    private String host = "";

    private Socket socket = null;
    private Scanner input = null;
    private PrintWriter output = null;

    private boolean running = false;

    private String username = "";

    private final ArrayList<ISocketSubscriber> subscribers = new ArrayList<>();

    public SocketConnection(String host, int port) {
        this.host = host;
        this.port = port;

        try {
            socket = new Socket(host, port);
            input = new Scanner(socket.getInputStream());
            output = new PrintWriter(socket.getOutputStream(), true);
        } catch (Exception e) {
        }

    }

    public SocketConnection(Socket connection) {
        this.host = connection.getInetAddress().getHostAddress();
        this.port = connection.getPort();

        try {
            socket = connection;
            input = new Scanner(socket.getInputStream());
            output = new PrintWriter(socket.getOutputStream(), true);
        } catch (Exception e) {
        }

    }

    @Override
    public void start() {
        running = true;

        receiveMessage();
    }

    @Override
    public void stop() {
        running = false;
    }

    @Override
    public void sendMessage(ISocketMessage message) {
        executor.execute( // the executor handles the threads in the right order! --> not important for any test!
                new Thread() { // new thread
                    @Override
                    public void run() { // every thread needs a run - method

                        // send message in a new thread!
                        output.println(message.getMessage());
                    }
                }
        ); // start thread immediately
    }

    @Override
    public void receiveMessage() {
        // SocketConnection reference = this; --> methode 1

        new Thread() { // new thread
            @Override
            public void run() { // every thread needs a run - method


                /* @todo auskommentiert hier die logik die war das erste nachricht die dem server geschickt wird username ist
                @todo damit ich die andere variante im Chat server verwenden kann ka y
                if (getUsername().length() <= 0) {
                    String username = input.nextLine();
                    setUsername(username);
                }
*/
                while (running) { // receive messages in an endless-loop
                    String msg = input.nextLine();

                    // notifySubscribers(new Message(reference, msg)); --> possible, but ugly.
                    notifySubscribers(new SocketMessage(SocketConnection.this, msg)); // access outer class "SocketConnection" and get "this" - reference
                }
            }
        }.start();
    }

    @Override
    public void notifySubscribers(ISocketMessage msg) {
        for (ISocketSubscriber sub : subscribers) {
            sub.messageReceived(msg);
        }
    }

    @Override
    public void addSubscriber(ISocketSubscriber sub) {
        if (!subscribers.contains(sub)) {
            subscribers.add(sub);
        }
    }

    @Override
    public void removeSubscriber(ISocketSubscriber sub) {
        subscribers.remove(sub);
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

}
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
SocketMessage.java


public class SocketMessage implements ISocketMessage {
    private ISocketConnection source = null;
    private String message = "";

    public SocketMessage(ISocketConnection source, String message) {
        this.source = source;
        this.message = message;
    }

    @Override
    public String getMessage() {
        return message;
    }

    @Override
    public void setMessage(String msg) {
        this.message = msg;
    }

    @Override
    public ISocketConnection getSource() {
        return source;
    }

    @Override
    public void setSource(ISocketConnection src) {
        this.source = src;
    }
}

------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------------------
