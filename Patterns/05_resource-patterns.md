# Pattern per Gestione delle Risorse

## 1. Resource Pool Pattern

### Descrizione
Il Resource Pool Pattern consiste nella gestione di un insieme limitato di risorse che vengono riutilizzate anziché create e distrutte ogni volta che sono necessarie. Le risorse vengono acquisite dal pool quando richieste e restituite al pool quando non sono più necessarie.

### Quando utilizzarlo
- Quando la creazione e la distruzione di risorse è costosa (in termini di tempo o memoria)
- Quando le risorse sono limitate e devono essere condivise tra più componenti del sistema
- Per migliorare le prestazioni limitando il numero di risorse utilizzabili contemporaneamente
- Per gestire connessioni a database, socket di rete, thread o altri oggetti costosi da creare

### Esempio di implementazione in Java

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.function.Supplier;

public class ResourcePool<T> {
    private final BlockingQueue<T> resources;
    private final Supplier<T> resourceFactory;
    private final int maxSize;
    private int created;

    public ResourcePool(int maxSize, Supplier<T> resourceFactory) {
        this.maxSize = maxSize;
        this.resourceFactory = resourceFactory;
        this.resources = new LinkedBlockingQueue<>(maxSize);
        this.created = 0;
    }

    public T acquire() throws InterruptedException {
        synchronized (this) {
            if (resources.isEmpty() && created < maxSize) {
                created++;
                return resourceFactory.get();
            }
        }
        // Se abbiamo raggiunto il limite, attendiamo che una risorsa venga rilasciata
        return resources.take();
    }

    public void release(T resource) throws InterruptedException {
        if (resource != null) {
            resources.put(resource);
        }
    }

    public void shutdown() {
        // Chiudere eventuali risorse se necessario
        resources.clear();
    }
}

// Esempio di utilizzo con connessioni a database
class DatabaseConnection {
    public DatabaseConnection() {
        System.out.println("Creazione di una nuova connessione al database");
        // Logica di connessione al database
    }
    
    public void query(String sql) {
        System.out.println("Esecuzione query: " + sql);
    }
    
    public void close() {
        System.out.println("Chiusura connessione al database");
    }
}

class Example {
    public static void main(String[] args) throws InterruptedException {
        // Creare un pool di 5 connessioni al database
        ResourcePool<DatabaseConnection> connectionPool = 
            new ResourcePool<>(5, DatabaseConnection::new);
        
        // Usare una connessione
        DatabaseConnection conn = connectionPool.acquire();
        try {
            conn.query("SELECT * FROM users");
        } finally {
            connectionPool.release(conn);  // Restituire la connessione al pool
        }
    }
}
```

## 2. Thread Pool Pattern

### Descrizione
Il Thread Pool Pattern consiste nella creazione di un insieme di thread riutilizzabili che vengono mantenuti in vita per eseguire task in modo asincrono. Invece di creare un nuovo thread per ogni operazione, i task vengono inseriti in una coda e i thread nel pool li prelevano ed eseguono.

### Quando utilizzarlo
- Quando si devono eseguire molti task brevi e indipendenti
- Per limitare il numero di thread attivi nel sistema, evitando sovraccarichi
- Per ridurre l'overhead dovuto alla creazione e distruzione continua di thread
- Per migliorare la reattività di applicazioni con molte operazioni asincrone

### Esempio di implementazione in Java

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ThreadPoolExample {
    public static void main(String[] args) {
        // Creazione di un thread pool con un numero fisso di thread
        ExecutorService executor = Executors.newFixedThreadPool(5);
        
        // Inviare 10 task al thread pool
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.submit(() -> {
                String threadName = Thread.currentThread().getName();
                System.out.println("Task " + taskId + " eseguito dal thread " + threadName);
                
                // Simulare un'operazione che richiede tempo
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                return "Risultato del task " + taskId;
            });
        }
        
        // Iniziare lo shutdown ordinato del thread pool
        executor.shutdown();
        
        try {
            // Attendere il completamento di tutti i task (con timeout)
            if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                // Forzare lo shutdown se il timeout scade
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
        
        System.out.println("Tutti i task sono stati completati");
    }
}
```

### Altri tipi di Thread Pool in Java
Java fornisce diverse implementazioni di thread pool tramite la classe `Executors`:

- `newFixedThreadPool(n)`: Pool con un numero fisso di thread
- `newCachedThreadPool()`: Pool che crea nuovi thread secondo necessità, ma riutilizza quelli esistenti quando sono disponibili
- `newSingleThreadExecutor()`: Pool con un singolo thread, utile per garantire che i task vengano eseguiti sequenzialmente
- `newScheduledThreadPool(n)`: Pool che supporta l'esecuzione programmata di task

## 3. Two-Phase Termination Pattern

### Descrizione
Il Two-Phase Termination Pattern è un approccio per terminare un thread o una risorsa in modo sicuro e controllato. Il processo di terminazione avviene in due fasi:
1. **Fase di richiesta di terminazione**: Si invia un segnale al thread per richiederne la terminazione
2. **Fase di cleanup**: Il thread esegue le operazioni di pulizia necessarie prima di terminare

### Quando utilizzarlo
- Quando è necessario terminare un thread in modo elegante, dando la possibilità di eseguire operazioni di pulizia
- Per evitare la perdita di risorse o lo stato inconsistente che può verificarsi con una terminazione brusca
- Nei sistemi dove i thread gestiscono risorse che devono essere rilasciate correttamente
- Nelle applicazioni server o nei servizi che devono gestire lo shutdown in modo ordinato

### Esempio di implementazione in Java

```java
import java.util.concurrent.atomic.AtomicBoolean;

public class WorkerThread {
    private final Thread thread;
    private final AtomicBoolean running = new AtomicBoolean(true);
    private final AtomicBoolean terminated = new AtomicBoolean(false);

    public WorkerThread() {
        thread = new Thread(() -> {
            try {
                // Inizializzazione
                init();
                
                // Loop principale
                while (running.get() && !Thread.currentThread().isInterrupted()) {
                    // Eseguire il lavoro
                    doWork();
                    
                    // Pausa breve per controllare il flag di interruzione
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                // Gestire l'interruzione
                Thread.currentThread().interrupt();
                System.out.println("Thread interrotto");
            } finally {
                // Fase di cleanup
                cleanup();
                terminated.set(true);
                System.out.println("Thread terminato con successo");
            }
        });
    }

    public void start() {
        thread.start();
    }

    // Fase 1: Richiedi la terminazione
    public void terminate() {
        running.set(false);
        thread.interrupt(); // Sveglia il thread se è in attesa
    }
    
    // Attende che la terminazione sia completata
    public void join() throws InterruptedException {
        thread.join();
    }
    
    // Controlla se il thread è stato completamente terminato
    public boolean isTerminated() {
        return terminated.get();
    }
    
    // Metodi da implementare o sovrascrivere
    protected void init() {
        System.out.println("Inizializzazione delle risorse");
    }
    
    protected void doWork() {
        System.out.println("Esecuzione del lavoro...");
    }
    
    protected void cleanup() {
        System.out.println("Pulizia delle risorse");
    }
    
    public static void main(String[] args) throws InterruptedException {
        WorkerThread worker = new WorkerThread();
        worker.start();
        
        // Lasciar eseguire il thread per un po'
        Thread.sleep(3000);
        
        // Iniziare la terminazione
        System.out.println("Richiesta terminazione...");
        worker.terminate();
        
        // Attendere il completamento della terminazione
        worker.join();
        System.out.println("Il worker è terminato? " + worker.isTerminated());
    }
}
```

### Benefici del Two-Phase Termination
- Evita la corruzione dei dati o perdite di risorse
- Permette al thread di completare operazioni critiche in corso
- Fornisce un meccanismo per liberare le risorse in modo ordinato
- È più robusto rispetto a metodi come `Thread.stop()` (deprecato)

## Confronto e considerazioni finali

| Pattern | Uso principale | Vantaggi | Svantaggi |
|---------|---------------|----------|-----------|
| Resource Pool | Gestione di risorse limitate e costose | Riutilizzo efficiente, controllo della concorrenza | Overhead di sincronizzazione, possibile contesa |
| Thread Pool | Esecuzione di task concorrenti | Riutilizzo di thread, controllo del parallelismo | Possibile deadlock, dimensionamento complesso |
| Two-Phase Termination | Terminazione sicura di thread | Terminazione ordinata, cleanup garantito | Complessità aggiuntiva, possibili ritardi nella terminazione |

Questi pattern sono spesso utilizzati insieme nei sistemi complessi. Ad esempio, un server web potrebbe utilizzare un Thread Pool per gestire le richieste dei client, Resource Pool per gestire le connessioni al database, e Two-Phase Termination per lo shutdown ordinato dell'intero sistema.
