# Pattern di Concorrenza in Java

## 1. Balking Pattern

### Descrizione
Il Balking Pattern è un pattern comportamentale che consente a un oggetto di rifiutare di eseguire un'azione se lo stato non è appropriato. In pratica, se l'oggetto si trova in uno stato in cui non può eseguire un certo metodo, il metodo ritorna immediatamente senza eseguire alcuna azione.

### Quando Usarlo
- Quando un'operazione ha senso solo in un particolare stato dell'oggetto
- Quando vogliamo evitare attese o blocchi se un'azione non può essere eseguita
- Per gestire richieste asincrone che potrebbero arrivare quando il sistema non è pronto

### Esempio

```java
public class DocumentEditor {
    private boolean isDocumentSaved = true;
    private String content = "";
    
    // Aggiunge contenuto al documento e lo segna come non salvato
    public synchronized void updateContent(String newContent) {
        content += newContent;
        isDocumentSaved = false;
    }
    
    // Salva il documento solo se è stato modificato
    public synchronized boolean save() {
        // Balk (rifiuta) se il documento è già salvato
        if (isDocumentSaved) {
            System.out.println("Il documento è già salvato. Nessuna azione necessaria.");
            return false;
        }
        
        System.out.println("Salvataggio del documento...");
        // Logica per salvare il documento
        isDocumentSaved = true;
        return true;
    }
    
    public boolean isDocumentSaved() {
        return isDocumentSaved;
    }
}
```

### Utilizzo

```java
DocumentEditor editor = new DocumentEditor();
editor.updateContent("Nuovo testo");
editor.save(); // Salva il documento
editor.save(); // "Il documento è già salvato. Nessuna azione necessaria."
```

## 2. Guarded Suspension

### Descrizione
Il Guarded Suspension pattern sospende un thread fino a quando una precondizione diventa vera. Il thread rimane in attesa finché la condizione non viene soddisfatta, e poi procede con l'esecuzione.

### Quando Usarlo
- Quando un'operazione può procedere solo se una certa condizione è soddisfatta
- In produttore-consumatore quando il consumatore deve attendere che ci siano dati disponibili
- Quando è necessario sincronizzare thread che operano a velocità diverse

### Esempio

```java
public class MessageQueue {
    private final LinkedList<String> queue = new LinkedList<>();
    private final int capacity;
    
    public MessageQueue(int capacity) {
        this.capacity = capacity;
    }
    
    // Il produttore inserisce messaggi nella coda
    public synchronized void put(String message) throws InterruptedException {
        // Attendi se la coda è piena
        while (queue.size() >= capacity) {
            wait(); // Sospende il thread corrente
        }
        
        queue.add(message);
        System.out.println("Prodotto messaggio: " + message);
        
        // Notifica i consumatori che c'è un nuovo messaggio
        notifyAll();
    }
    
    // Il consumatore preleva messaggi dalla coda
    public synchronized String take() throws InterruptedException {
        // Attendi se la coda è vuota
        while (queue.isEmpty()) {
            wait(); // Sospende il thread corrente
        }
        
        String message = queue.removeFirst();
        System.out.println("Consumato messaggio: " + message);
        
        // Notifica i produttori che c'è spazio disponibile
        notifyAll();
        return message;
    }
}
```

### Utilizzo

```java
MessageQueue queue = new MessageQueue(10);

// Thread produttore
new Thread(() -> {
    try {
        for (int i = 0; i < 20; i++) {
            queue.put("Messaggio " + i);
            Thread.sleep(100);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}).start();

// Thread consumatore
new Thread(() -> {
    try {
        for (int i = 0; i < 20; i++) {
            String message = queue.take();
            Thread.sleep(200); // Consuma più lentamente
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}).start();
```

## 3. Active Object

### Descrizione
Il pattern Active Object disaccoppia l'esecuzione di un metodo dalla sua invocazione. Ogni oggetto ha un proprio thread di controllo e una coda di messaggi/richieste. Quando viene invocato un metodo, la richiesta viene inserita in una coda e poi eseguita dal thread dell'oggetto.

### Quando Usarlo
- Per disaccoppiare chiamante e chiamato per aumentare la concorrenza
- Quando le richieste devono essere elaborate in un altro thread
- Per sistemi con molte operazioni asincrone
- Quando vuoi trasformare metodi sincroni in chiamate asincrone

### Esempio

```java
// Classe per rappresentare i risultati futuri
class Result<T> {
    private T value;
    private boolean isCompleted = false;
    
    public synchronized void setValue(T value) {
        this.value = value;
        this.isCompleted = true;
        notifyAll();
    }
    
    public synchronized T getValue() throws InterruptedException {
        while (!isCompleted) {
            wait();
        }
        return value;
    }
}

// Interfaccia per le richieste
interface Task<T> {
    T execute();
}

// Active Object
class ActiveObject {
    private final BlockingQueue<Runnable> queue;
    private final Thread worker;
    private volatile boolean running = true;
    
    public ActiveObject() {
        this.queue = new LinkedBlockingQueue<>();
        this.worker = new Thread(() -> {
            while (running) {
                try {
                    Runnable task = queue.take();
                    task.run();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        worker.start();
    }
    
    public <T> Result<T> enqueue(Task<T> task) {
        Result<T> result = new Result<>();
        queue.offer(() -> {
            T value = task.execute();
            result.setValue(value);
        });
        return result;
    }
    
    public void shutdown() {
        running = false;
        worker.interrupt();
    }
}
```

### Utilizzo

```java
ActiveObject activeObject = new ActiveObject();

// Creare ed inviare un task
Result<Integer> result = activeObject.enqueue(() -> {
    System.out.println("Esecuzione di un calcolo lungo...");
    try {
        Thread.sleep(2000); // Simulazione di elaborazione
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return 42;
});

// Fare altre cose mentre il task viene eseguito in background
System.out.println("Continuiamo a lavorare mentre il task è in esecuzione...");

try {
    // Quando abbiamo bisogno del risultato, lo attendiamo
    Integer value = result.getValue();
    System.out.println("Risultato: " + value);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}

// Chiudi l'active object quando non serve più
activeObject.shutdown();
```

## 4. Thread-Specific Storage

### Descrizione
Il Thread-Specific Storage (TSS) è un pattern che consente di memorizzare dati specifici per ogni thread. Ogni thread ha la propria copia privata dei dati, evitando così la necessità di sincronizzazione quando si accede a questi dati.

### Quando Usarlo
- Quando si ha bisogno di associare dati a un thread specifico
- Per ridurre la contesa e migliorare le prestazioni evitando la sincronizzazione
- Per gestire il contesto di un'operazione in ambiente multi-thread (es. gestione transazioni)
- Per memorizzare informazioni di sicurezza o identità legate a un thread

### Esempio

```java
public class ThreadSpecificUserContext {
    // ThreadLocal mantiene una copia separata dei dati per ogni thread
    private static final ThreadLocal<UserContext> userContext = new ThreadLocal<UserContext>() {
        @Override
        protected UserContext initialValue() {
            return new UserContext("Guest", false);
        }
    };
    
    public static void setUserContext(String username, boolean isAdmin) {
        userContext.set(new UserContext(username, isAdmin));
    }
    
    public static UserContext getUserContext() {
        return userContext.get();
    }
    
    public static void clearUserContext() {
        userContext.remove();
    }
    
    // Classe interna per memorizzare il contesto dell'utente
    public static class UserContext {
        private final String username;
        private final boolean isAdmin;
        
        public UserContext(String username, boolean isAdmin) {
            this.username = username;
            this.isAdmin = isAdmin;
        }
        
        public String getUsername() {
            return username;
        }
        
        public boolean isAdmin() {
            return isAdmin;
        }
        
        @Override
        public String toString() {
            return "UserContext{username='" + username + "', isAdmin=" + isAdmin + "}";
        }
    }
}
```

### Utilizzo

```java
public class WebServer {
    public void handleRequest(String requestId, String username, boolean isAdmin) {
        // Imposta il contesto utente per questo thread di richiesta
        ThreadSpecificUserContext.setUserContext(username, isAdmin);
        
        try {
            // Simula l'elaborazione di una richiesta
            System.out.println("Thread " + Thread.currentThread().getName() + 
                              " sta elaborando la richiesta " + requestId);
            
            // In qualsiasi punto del codice, possiamo accedere al contesto utente
            UserContext userCtx = ThreadSpecificUserContext.getUserContext();
            System.out.println("Richiesta " + requestId + 
                              " eseguita dall'utente " + userCtx.getUsername());
            
            // Verifica delle autorizzazioni
            if (userCtx.isAdmin()) {
                System.out.println("Accesso amministrativo concesso per la richiesta " + requestId);
            } else {
                System.out.println("Accesso standard per la richiesta " + requestId);
            }
            
        } finally {
            // Pulisci il contesto alla fine della richiesta
            ThreadSpecificUserContext.clearUserContext();
        }
    }
}
```

Esempio di utilizzo con thread multipli:

```java
WebServer server = new WebServer();

// Simula richieste multiple da diversi utenti
new Thread(() -> server.handleRequest("REQ-1", "alice", true), "RequestHandler-1").start();
new Thread(() -> server.handleRequest("REQ-2", "bob", false), "RequestHandler-2").start();
new Thread(() -> server.handleRequest("REQ-3", "charlie", false), "RequestHandler-3").start();
```

## 5. Dining Philosophers Solution

### Descrizione
Il problema dei filosofi a cena è un classico problema di sincronizzazione nella programmazione concorrente. Cinque filosofi siedono attorno a un tavolo rotondo. Davanti a ciascuno c'è un piatto di spaghetti e tra ogni piatto c'è una forchetta. Per mangiare, un filosofo ha bisogno di entrambe le forchette vicine. Il problema è come progettare un algoritmo che permetta ai filosofi di mangiare senza causare deadlock o starvation.

### Quando Usarlo
- Come esempio di risoluzione di problemi di deadlock in sistemi con risorse condivise
- Quando più thread competono per risorse scarse e devono acquisirle in un certo ordine
- Come modello per studiare e risolvere problemi di concorrenza complessi

### Esempio

```java
public class DiningPhilosophers {
    private static final int NUM_PHILOSOPHERS = 5;
    private final Object[] forks = new Object[NUM_PHILOSOPHERS];
    private final Philosopher[] philosophers = new Philosopher[NUM_PHILOSOPHERS];
    
    public DiningPhilosophers() {
        // Inizializza le forchette e i filosofi
        for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
            forks[i] = new Object();
        }
        
        for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
            // La soluzione di Dijkstra: l'ultimo filosofo prende prima la forchetta destra, poi la sinistra
            Object leftFork = forks[i];
            Object rightFork = forks[(i + 1) % NUM_PHILOSOPHERS];
            
            if (i == NUM_PHILOSOPHERS - 1) {
                // L'ultimo filosofo prende le forchette in ordine inverso
                philosophers[i] = new Philosopher(i, rightFork, leftFork);
            } else {
                philosophers[i] = new Philosopher(i, leftFork, rightFork);
            }
            
            new Thread(philosophers[i], "Philosopher " + i).start();
        }
    }
    
    // Classe interna per rappresentare un filosofo
    private static class Philosopher implements Runnable {
        private final int id;
        private final Object firstFork;
        private final Object secondFork;
        private final Random random = new Random();
        
        public Philosopher(int id, Object firstFork, Object secondFork) {
            this.id = id;
            this.firstFork = firstFork;
            this.secondFork = secondFork;
        }
        
        private void think() throws InterruptedException {
            System.out.println("Filosofo " + id + " sta pensando...");
            Thread.sleep(random.nextInt(1000));
        }
        
        private void eat() throws InterruptedException {
            System.out.println("Filosofo " + id + " sta mangiando...");
            Thread.sleep(random.nextInt(1000));
        }
        
        @Override
        public void run() {
            try {
                while (true) {
                    think();
                    
                    // Prendi la prima forchetta
                    synchronized (firstFork) {
                        System.out.println("Filosofo " + id + " ha preso la prima forchetta");
                        
                        // Prendi la seconda forchetta
                        synchronized (secondFork) {
                            System.out.println("Filosofo " + id + " ha preso la seconda forchetta");
                            
                            // Ora il filosofo può mangiare
                            eat();
                            
                            System.out.println("Filosofo " + id + " ha rilasciato la seconda forchetta");
                        }
                        
                        System.out.println("Filosofo " + id + " ha rilasciato la prima forchetta");
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    public static void main(String[] args) {
        new DiningPhilosophers();
    }
}
