# Pattern di Sincronizzazione in Java

In programmazione concorrente, i pattern di sincronizzazione sono essenziali per coordinare l'esecuzione di thread multipli e garantire l'accesso sicuro alle risorse condivise. Ecco una spiegazione dettagliata dei pattern di sincronizzazione più comuni in Java.

## 1. Barrier Pattern (CyclicBarrier)

### Descrizione
Il Barrier Pattern permette a un gruppo di thread di attendere finché tutti non raggiungono un punto comune di esecuzione (la "barriera") prima di procedere. In Java, questo pattern è implementato principalmente tramite la classe `CyclicBarrier`.

### Quando usarlo
- Quando hai computazioni parallele che devono sincronizzarsi a determinati punti
- Quando stai implementando algoritmi che richiedono più fasi, e ogni fase deve completarsi per tutti i thread prima di iniziare la successiva
- Simulazioni in cui tutti gli attori devono completare una mossa prima che inizi il prossimo turno

### Esempio di codice

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class BarrierExample {
    public static void main(String[] args) {
        final int THREAD_COUNT = 3;
        
        // Crea una barriera che attende 3 thread e poi esegue l'azione specificata
        CyclicBarrier barrier = new CyclicBarrier(THREAD_COUNT, 
                                                 () -> System.out.println("Fase completata!"));
        
        for (int i = 0; i < THREAD_COUNT; i++) {
            final int threadNum = i;
            new Thread(() -> {
                try {
                    System.out.println("Thread " + threadNum + " in esecuzione...");
                    Thread.sleep((long) (Math.random() * 3000)); // Simula lavoro
                    System.out.println("Thread " + threadNum + " in attesa alla barriera");
                    
                    barrier.await(); // Attende che tutti i thread raggiungano la barriera
                    
                    System.out.println("Thread " + threadNum + " prosegue dopo la barriera");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### Considerazioni
- A differenza di `CountDownLatch`, `CyclicBarrier` può essere riutilizzato dopo che tutti i thread lo hanno attraversato
- Se un thread viene interrotto mentre è in attesa alla barriera, la barriera entra in uno stato rotto (`broken`) e tutti gli altri thread riceveranno `BrokenBarrierException`

## 2. Latch Pattern (CountDownLatch)

### Descrizione
Il Latch Pattern permette a uno o più thread di attendere finché un insieme di operazioni eseguite in altri thread non viene completato. In Java, questo pattern è implementato principalmente tramite la classe `CountDownLatch`.

### Quando usarlo
- Per implementare dipendenze tra thread (alcuni thread devono attendere che altri completino prima di iniziare)
- Come un semplice "trigger" che permette a un thread di procedere quando altri hanno terminato
- Per attendere che determinate risorse o servizi siano inizializzati prima di usarli

### Esempio di codice

```java
import java.util.concurrent.CountDownLatch;

public class LatchExample {
    public static void main(String[] args) {
        // Crea un latch con conteggio iniziale di 3
        CountDownLatch latch = new CountDownLatch(3);
        
        // Thread principale che attende il completamento
        new Thread(() -> {
            try {
                System.out.println("Thread principale in attesa che tutti i servizi siano inizializzati...");
                latch.await(); // Blocca finché il conteggio non raggiunge zero
                System.out.println("Tutti i servizi sono pronti! L'applicazione può partire.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        
        // Simula l'inizializzazione di servizi
        for (int i = 0; i < 3; i++) {
            final int serviceNum = i;
            new Thread(() -> {
                try {
                    System.out.println("Inizializzazione del servizio " + serviceNum);
                    Thread.sleep((long) (Math.random() * 5000)); // Simula il tempo di avvio
                    System.out.println("Servizio " + serviceNum + " avviato");
                    
                    latch.countDown(); // Decrementa il conteggio
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### Considerazioni
- A differenza di `CyclicBarrier`, `CountDownLatch` non può essere riutilizzato una volta che il conteggio raggiunge zero
- Non è possibile reimpostare il conteggio
- È possibile chiamare `await()` con un timeout per evitare attese indefinite

## 3. Semaphore Pattern

### Descrizione
Il Semaphore Pattern limita il numero di thread che possono accedere contemporaneamente a una risorsa o eseguire una sezione critica di codice. In Java, questo pattern è implementato tramite la classe `Semaphore`.

### Quando usarlo
- Per limitare il numero di connessioni contemporanee a una risorsa (come un database)
- Per implementare pool di risorse con un numero limitato di istanze
- Per implementare il throttling (limitazione di velocità) in applicazioni

### Esempio di codice

```java
import java.util.concurrent.Semaphore;

public class ConnectionPool {
    private final Semaphore semaphore;
    
    public ConnectionPool(int maxConnections) {
        // Crea un semaforo con il numero specificato di permessi
        semaphore = new Semaphore(maxConnections, true); // true per fairness
    }
    
    public void useConnection() {
        try {
            System.out.println("Thread " + Thread.currentThread().getId() + 
                              " richiede una connessione...");
            
            semaphore.acquire(); // Acquisisce un permesso o blocca
            
            try {
                System.out.println("Thread " + Thread.currentThread().getId() + 
                                 " ha ottenuto una connessione");
                Thread.sleep(2000); // Simula uso della connessione
            } finally {
                System.out.println("Thread " + Thread.currentThread().getId() + 
                                 " rilascia la connessione");
                semaphore.release(); // Rilascia sempre il permesso
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    public static void main(String[] args) {
        ConnectionPool pool = new ConnectionPool(3); // Max 3 connessioni contemporanee
        
        // 10 thread cercano di usare le connessioni
        for (int i = 0; i < 10; i++) {
            new Thread(pool::useConnection).start();
        }
    }
}
```

### Considerazioni
- Un semaforo con un solo permesso funziona come un mutex (esclusione mutua)
- È possibile acquisire più permessi contemporaneamente con `acquire(n)`
- Il parametro fairness garantisce che i thread vengano serviti nell'ordine di arrivo
- Si può verificare la disponibilità di permessi senza bloccare con `tryAcquire()`

## 4. Read-Write Lock Pattern

### Descrizione
Il Read-Write Lock Pattern permette accessi concorrenti in lettura ma garantisce accessi esclusivi in scrittura. In Java, questo pattern è implementato tramite l'interfaccia `ReadWriteLock` e la sua implementazione `ReentrantReadWriteLock`.

### Quando usarlo
- Per strutture dati o risorse che vengono lette frequentemente ma scritte raramente
- Per cache o configurazioni condivise
- Per migliorare le prestazioni rispetto a un semplice lock esclusivo in scenari con molte letture

### Esempio di codice

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class CacheWithReadWriteLock {
    private final Map<String, Object> cache = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public Object read(String key) {
        lock.readLock().lock(); // Acquisisce il lock di lettura
        try {
            System.out.println("Thread " + Thread.currentThread().getId() + 
                              " legge dalla cache (lettori multipli concorrenti consentiti)");
            Thread.sleep(1000); // Simula operazione di lettura
            return cache.get(key);
        } catch (InterruptedException e) {
            e.printStackTrace();
            return null;
        } finally {
            lock.readLock().unlock(); // Rilascia sempre il lock
        }
    }
    
    public void write(String key, Object value) {
        lock.writeLock().lock(); // Acquisisce il lock di scrittura
        try {
            System.out.println("Thread " + Thread.currentThread().getId() + 
                              " scrive nella cache (accesso esclusivo)");
            Thread.sleep(2000); // Simula operazione di scrittura
            cache.put(key, value);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.writeLock().unlock(); // Rilascia sempre il lock
        }
    }
    
    public static void main(String[] args) {
        CacheWithReadWriteLock cache = new CacheWithReadWriteLock();
        
        // Inizializza la cache
        cache.write("key1", "value1");
        
        // 5 thread di lettura
        for (int i = 0; i < 5; i++) {
            new Thread(() -> cache.read("key1")).start();
        }
        
        // 2 thread di scrittura
        for (int i = 0; i < 2; i++) {
            final int num = i;
            new Thread(() -> cache.write("key" + num, "newValue" + num)).start();
        }
    }
}
```

### Considerazioni
- Letture multiple possono avvenire contemporaneamente
- Nessuna lettura può avvenire durante una scrittura
- Nessuna scrittura può avvenire durante una lettura o un'altra scrittura
- Alcuni `ReadWriteLock` supportano il "downgrading" (conversione da lock di scrittura a lettura) ma non l'"upgrading"
- A seconda dell'implementazione, può dare priorità ai lettori o agli scrittori

## 5. Future Pattern

### Descrizione
Il Future Pattern rappresenta il risultato di un'operazione asincrona che potrebbe non essere ancora disponibile. In Java, questo pattern è implementato tramite l'interfaccia `Future` e le varie classi che la implementano come `CompletableFuture`.

### Quando usarlo
- Per operazioni asincrone dove il risultato sarà disponibile in futuro
- Per parallelizzare computazioni indipendenti
- Per implementare pattern di programmazione reattiva
- Per gestire chiamate remote o I/O non bloccante

### Esempio di codice

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class FutureExample {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        
        // Esempio con Future tradizionale
        Future<Integer> future = executor.submit(() -> {
            System.out.println("Calcolo intensivo in esecuzione...");
            Thread.sleep(3000); // Simula operazione lunga
            return 42;
        });
        
        System.out.println("Continua l'esecuzione mentre il calcolo avviene in background");
        
        try {
            // Blocca fino a quando il risultato non è disponibile
            Integer result = future.get();
            System.out.println("Risultato: " + result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        
        // Esempio con CompletableFuture (più flessibile)
        CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
            try {
                System.out.println("Primo task in esecuzione...");
                Thread.sleep(2000);
                return "Hello";
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }).thenApply(s -> {
            System.out.println("Secondo task in esecuzione...");
            return s + " World";
        }).thenApply(String::toUpperCase);
        
        // Registra un callback invece di bloccare
        cf.thenAccept(result -> System.out.println("Risultato completato: " + result));
        
        // Attendi il completamento per evitare che il programma termini
        try {
            cf.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        } finally {
            executor.shutdown();
        }
    }
}
```

### Considerazioni avanzate con CompletableFuture

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class AdvancedCompletableFutureExample {
    public static void main(String[] args) {
        // Combinare due future
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(2);
                return "Risultato 1";
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
                return "Risultato 2";
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        
        // Combina i due risultati quando entrambi sono pronti
        CompletableFuture<String> combined = future1.thenCombine(future2, 
            (r1, r2) -> r1 + " + " + r2);
        
        // Attende il primo future a completarsi
        CompletableFuture<Object> anyOf = CompletableFuture.anyOf(future1, future2);
        
        // Gestione degli errori
        CompletableFuture<String> futureWithErrorHandling = CompletableFuture
            .supplyAsync(() -> {
                if (Math.random() > 0.5) {
                    throw new RuntimeException("Errore simulato");
                }
                return "Successo";
            })
            .exceptionally(ex -> {
                System.out.println("Si è verificato un errore: " + ex.getMessage());
                return "Fallimento";
            });
            
        // Attende i risultati
        try {
            System.out.println("Combined result: " + combined.get());
            System.out.println("AnyOf result: " + anyOf.get());
            System.out.println("With error handling: " + futureWithErrorHandling.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### Considerazioni
- `Future` rappresenta un risultato che sarà disponibile in futuro
- `Future.get()` è bloccante e può causare deadlock se non gestito correttamente
- `CompletableFuture` offre API più ricche per la composizione di operazioni asincrone
- Con `CompletableFuture` è possibile concatenare operazioni, gestire errori e combinare risultati
- È possibile specificare l'`Executor` su cui eseguire le operazioni

## Conclusioni

I pattern di sincronizzazione sono strumenti fondamentali nella programmazione concorrente. La scelta del pattern giusto dipende dalle specifiche esigenze:

- **Barrier Pattern (CyclicBarrier)**: Quando tutti i thread devono sincronizzarsi a un punto comune prima di procedere.
- **Latch Pattern (CountDownLatch)**: Quando uno o più thread devono attendere il completamento di un insieme definito di operazioni.
- **Semaphore Pattern**: Quando è necessario limitare il numero di thread che possono accedere contemporaneamente a una risorsa.
- **Read-Write Lock Pattern**: Quando una risorsa viene letta frequentemente ma modificata raramente.
- **Future Pattern**: Quando è necessario rappresentare il risultato di un'operazione asincrona che sarà disponibile in futuro.

La comprensione e l'uso appropriato di questi pattern possono migliorare significativamente le prestazioni e la robustezza delle applicazioni concorrenti.
