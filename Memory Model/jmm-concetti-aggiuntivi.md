# Concetti Aggiuntivi del JMM secondo Brian Goetz


## Thread Confinement

Una strategia fondamentale discussa da Goetz per evitare problemi di concorrenza è il "thread confinement", ovvero garantire che i dati siano accessibili da un solo thread.

### 1. Ad-hoc Thread Confinement

Quando il programmatore garantisce, tramite la struttura del codice, che un oggetto venga utilizzato solo da un thread specifico, senza supporto esplicito dal linguaggio.

```java
public class AdHocThreadConfinement {
    // Questa collezione non è thread-safe ma viene utilizzata solo dal thread della GUI
    private final List<PendingEvent> pendingEvents = new ArrayList<>();
    
    public void dispatchEvents() {
        // Questo metodo viene chiamato solo dal thread della GUI
        for (PendingEvent event : pendingEvents) {
            dispatchEvent(event);
        }
        pendingEvents.clear();
    }
    
    // Chiamato da altri thread per accodare eventi
    public void queueEvent(PendingEvent event) {
        synchronized(pendingEvents) {
            pendingEvents.add(event);
        }
        // Richiedi esecuzione nel thread della GUI
        SwingUtilities.invokeLater(this::dispatchEvents);
    }
}
```

### 2. Stack Confinement

Le variabili locali sono intrinsecamente confinate al thread che esegue il metodo, senza necessità di sincronizzazione.

```java
public int calculateSum(final int[] array) {
    // La variabile 'sum' è confinata nello stack di questo thread
    int sum = 0;
    for (int value : array) {
        sum += value;
    }
    return sum;
}
```

### 3. ThreadLocal

La classe `ThreadLocal` fornisce un meccanismo per associare dati specifici a ciascun thread.

```java
public class ConnectionManager {
    private static final ThreadLocal<Connection> connectionHolder = 
        ThreadLocal.withInitial(() -> DriverManager.getConnection(DB_URL));
    
    public static Connection getConnection() {
        return connectionHolder.get();
    }
    
    public static void setConnection(Connection conn) {
        connectionHolder.set(conn);
    }
}
```

## Safe Publication

Goetz dedica molte pagine ai meccanismi di "pubblicazione sicura" degli oggetti, cioè rendere un oggetto disponibile ad altri thread in modo che sia visibile correttamente.

### 1. Metodi Non Sicuri di Pubblicazione

```java
// Pubblicazione non sicura attraverso un campo public
public Holder holder;

// Thread 1
holder = new Holder(42);

// Thread 2 potrebbe vedere un oggetto Holder parzialmente inizializzato
```

### 2. Metodi Sicuri di Pubblicazione

```java
// 1. Inizializzazione di un oggetto in un campo statico durante il caricamento della classe
private static final Holder holder = new Holder(42);

// 2. Pubblicazione attraverso un campo volatile
private volatile Holder holder;
public void initialize() {
    holder = new Holder(42);
}

// 3. Pubblicazione attraverso una operazione atomica
private final AtomicReference<Holder> holder = new AtomicReference<>();
public void initialize() {
    holder.set(new Holder(42));
}

// 4. Pubblicazione tramite blocco synchronized
private Holder holder;
public synchronized void initialize() {
    holder = new Holder(42);
}
```

### 3. Immutabilità e Final Fields

Gli oggetti immutabili possono essere condivisi liberamente senza sincronizzazione:

```java
public class ImmutableValue {
    private final int[] array;
    
    public ImmutableValue(int[] array) {
        // Difesa contro modifiche esterne
        this.array = array.clone();
    }
    
    public int[] getArray() {
        // Protezione contro modifiche esterne
        return array.clone();
    }
}
```

Il JMM fornisce garanzie speciali per i campi `final` correttamente inizializzati:

```java
public class FinalFieldExample {
    private final int x;
    private int y;
    
    public FinalFieldExample() {
        x = 1;  // Scrittura di un campo final
        y = 2;  // Scrittura di un campo ordinario
    }
    
    // Un thread che vede un riferimento a questo oggetto
    // è garantito di vedere x=1, ma potrebbe non vedere y=2
}
```

## Escape Analysis e JIT Compiler Optimizations

Goetz menziona come il JIT compiler possa ottimizzare alcune operazioni in base all'escape analysis:

```java
public String createMessage() {
    StringBuilder sb = new StringBuilder();
    sb.append("Hello, ");
    sb.append("World!");
    return sb.toString();
}
```

In questo caso, il JIT compiler potrebbe determinare che `StringBuilder` non "sfugge" dal metodo e quindi eliminare la sincronizzazione implicita.

## Fence Instructions e Memory Barriers

Goetz spiega come le istruzioni `volatile` e i blocchi `synchronized` introducano barriere di memoria (memory barriers):

```
LoadLoad Barrier: Impedisce il riordinamento di due operazioni di lettura
StoreStore Barrier: Impedisce il riordinamento di due operazioni di scrittura
LoadStore Barrier: Impedisce il riordinamento di una lettura seguita da una scrittura
StoreLoad Barrier: Impedisce il riordinamento di una scrittura seguita da una lettura
```

Le operazioni `volatile` implementano tutte queste barriere:
- Una scrittura volatile agisce come una StoreStore barrier
- Una lettura volatile agisce come una LoadLoad barrier
- L'operazione volatile completa agisce come una StoreLoad barrier

## Atomic Variables e Non-Blocking Algorithms

Goetz spiega come le variabili atomiche supportino algoritmi non bloccanti:

```java
public class NonBlockingCounter {
    private final AtomicInteger value = new AtomicInteger(0);
    
    public int increment() {
        int current;
        int next;
        do {
            current = value.get();
            next = current + 1;
        } while (!value.compareAndSet(current, next));
        return next;
    }
}
```

## Lock Elision e Biased Locking

Goetz menziona ottimizzazioni della JVM che possono eliminare operazioni di lock non necessarie:

- **Lock Elision**: Rimozione completa di operazioni di lock quando l'analyzer determina che non sono necessarie
- **Biased Locking**: Ottimizzazione dell'acquisizione ripetuta di un lock dallo stesso thread

## CAS (Compare-and-Swap) e Hardware Support

Le operazioni atomiche in Java dipendono da istruzioni hardware a basso livello come Compare-and-Swap:

```java
// Implementazione concettuale di compareAndSet
public boolean compareAndSet(int expectedValue, int newValue) {
    // In realtà implementato con istruzioni hardware atomiche
    synchronized (this) {
        if (value == expectedValue) {
            value = newValue;
            return true;
        }
        return false;
    }
}
```

## Modelli di Concorrenza Avanzati

### 1. Fork-Join Framework

```java
public class ForkJoinSum extends RecursiveTask<Long> {
    private final long[] numbers;
    private final int start;
    private final int end;
    private final int THRESHOLD = 10_000;
    
    // Costruttore e implementazione...
    
    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            // Calcolo diretto per array piccoli
            return computeDirectly();
        }
        // Divide il problema in parti più piccole
        int middle = start + length / 2;
        ForkJoinSum leftTask = new ForkJoinSum(numbers, start, middle);
        ForkJoinSum rightTask = new ForkJoinSum(numbers, middle, end);
        
        // Fork del task di destra e calcolo del task di sinistra
        rightTask.fork();
        Long leftResult = leftTask.compute();
        Long rightResult = rightTask.join();
        
        return leftResult + rightResult;
    }
}
```

### 2. Work Stealing

Goetz descrive l'algoritmo di work stealing utilizzato nel Fork-Join framework:

```
1. Ogni worker thread ha la propria coda di task
2. Se un thread completa tutti i suoi task, "ruba" task dalla coda di altri thread
3. Questo bilancia automaticamente il carico di lavoro
```

## Contention e Performance Scaling

Goetz dedica una sezione all'impatto della contention sulle performance:

```
              ^ Performance
              |
              |       Ideale
              |      /
              |     /
              |    /
              |   /      Reale
              |  /      /
              | /      /
              |/      /
              +----------------> Thread
```

Dimostrando come il throughput reale si discosti da quello ideale all'aumentare del numero di thread a causa della contention.

## False Sharing

Goetz descrive il problema di "false sharing" che può verificarsi quando due thread accedono a dati diversi che risiedono però nella stessa cache line:

```java
public class FalseSharingExample {
    // In una CPU moderna, questi campi probabilmente condividono la stessa cache line
    public long value1;
    public long value2;
    
    // Versione migliorata per evitare il false sharing
    public long value1;
    public long padding1, padding2, padding3, padding4, padding5, padding6, padding7;
    public long value2;
}
```

## JSR-133 e Memory Model Evolution

Goetz spiega come il JMM sia stato rivisto con JSR-133 per risolvere vari problemi riscontrati nel modello precedente:

```
JMM pre-Java 5:
- Regole di sincronizzazione e visibilità poco chiare
- Implementazione inconsistente tra JVM diverse
- Pattern come double-checked locking non funzionavano correttamente

JMM post-Java 5 (JSR-133):
- Definizione chiara delle garanzie di memoria
- Semantiche migliorate per volatile
- Supporto per modelli di programmazione concorrente senza lock
```

## Conclusione

Questi concetti aggiuntivi completano la comprensione del Java Memory Model come presentato nel libro di Brian Goetz. La padronanza di questi aspetti è essenziale per sviluppare applicazioni Java concorrenti che siano corrette, robuste e performanti.
