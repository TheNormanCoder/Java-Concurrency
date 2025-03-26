# Pattern di Concorrenza nelle Operazioni Composte

## Introduzione

Nei sistemi concorrenti, le operazioni che sembrano atomiche al livello di astrazione del codice possono in realtà coinvolgere più operazioni sottostanti che possono essere interrotte. Brian Goetz nel suo libro "Java Concurrency in Practice" identifica tre principali pattern problematici che richiedono particolare attenzione durante la programmazione concorrente.

## 1. Check-Then-Act (Verifica-poi-Agisci)

### Spiegazione
Il pattern Check-Then-Act si verifica quando un'operazione controlla una condizione (check) e poi esegue un'azione (act) basata sul risultato della verifica. Se un altro thread modifica lo stato tra la verifica e l'azione, si può verificare un problema di race condition.

### Quando si usa
Si utilizza questo pattern quando:
- È necessario verificare una condizione prima di procedere con un'operazione
- L'operazione deve essere eseguita solo se la condizione è soddisfatta
- La condizione e l'azione devono essere eseguite atomicamente

### Esempio non thread-safe
```java
// Esempio non thread-safe di Check-Then-Act
public class LazyInitialization {
    private static ExpensiveObject instance = null;
    
    public static ExpensiveObject getInstance() {
        if (instance == null) {     // Check
            instance = new ExpensiveObject();  // Act
        }
        return instance;
    }
}
```

In questo esempio di inizializzazione pigra (lazy initialization), se due thread chiamano `getInstance()` contemporaneamente quando `instance` è ancora null, entrambi potrebbero passare il controllo e creare due istanze diverse.

### Soluzione thread-safe
```java
// Soluzione thread-safe con sincronizzazione
public class SafeLazyInitialization {
    private static ExpensiveObject instance = null;
    
    public static synchronized ExpensiveObject getInstance() {
        if (instance == null) {
            instance = new ExpensiveObject();
        }
        return instance;
    }
}

// Soluzione moderna con Double-Checked Locking e volatile
public class ModernLazyInitialization {
    private static volatile ExpensiveObject instance = null;
    
    public static ExpensiveObject getInstance() {
        if (instance == null) {  // Prima verifica (non sincronizzata)
            synchronized (ModernLazyInitialization.class) {
                if (instance == null) {  // Seconda verifica (sincronizzata)
                    instance = new ExpensiveObject();
                }
            }
        }
        return instance;
    }
}
```

## 2. Read-Modify-Write (Leggi-Modifica-Scrivi)

### Spiegazione
Il pattern Read-Modify-Write si verifica quando un'operazione legge un valore, lo modifica basandosi sul valore letto, e quindi scrive il nuovo valore. Se un altro thread modifica il valore tra la lettura e la scrittura, il cambiamento può essere perso.

### Quando si usa
Si utilizza questo pattern quando:
- È necessario aggiornare un valore basandosi sul suo stato corrente
- L'operazione di aggiornamento non è atomica
- Il valore può essere acceduto da più thread

### Esempio non thread-safe
```java
// Esempio non thread-safe di Read-Modify-Write
public class Counter {
    private int value;
    
    public void increment() {
        value++;  // Operazione non atomica: legge value, aggiunge 1, scrive value
    }
    
    public int getValue() {
        return value;
    }
}
```

L'operazione `value++` sembra atomica ma è in realtà composta da tre operazioni: leggere il valore, incrementarlo, e scrivere il nuovo valore. Se due thread incrementano contemporaneamente, uno potrebbe sovrascrivere l'incremento dell'altro.

### Soluzione thread-safe
```java
// Soluzione thread-safe con sincronizzazione
public class SynchronizedCounter {
    private int value;
    
    public synchronized void increment() {
        value++;
    }
    
    public synchronized int getValue() {
        return value;
    }
}

// Soluzione con AtomicInteger
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private final AtomicInteger value = new AtomicInteger(0);
    
    public void increment() {
        value.incrementAndGet();  // Operazione atomica
    }
    
    public int getValue() {
        return value.get();
    }
}
```

## 3. Compound Actions (Azioni Composte)

### Spiegazione
Le azioni composte sono sequenze di operazioni che devono essere eseguite come un'unica unità atomica per mantenere l'invariante di un oggetto. Questo pattern è una generalizzazione dei due precedenti, e si riferisce a qualsiasi scenario in cui più operazioni devono essere considerate come un'unica operazione atomica.

### Quando si usa
Si utilizza questo pattern quando:
- Una serie di operazioni correlate deve essere eseguita come un'unica unità atomica
- L'invariante dell'oggetto deve essere mantenuto durante l'intera sequenza
- Le operazioni coinvolgono più variabili o condizioni

### Esempio non thread-safe
```java
// Esempio non thread-safe di azioni composte
public class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    
    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }
    
    public void put(T item) {
        if (queue.size() < capacity) {  // Check
            queue.add(item);  // Act
        }
    }
    
    public T take() {
        if (!queue.isEmpty()) {  // Check
            return queue.poll();  // Act
        }
        return null;
    }
}
```

In questo buffer limitato, le operazioni `put` e `take` non sono atomiche. Se due thread chiamano `put` contemporaneamente quando c'è solo uno spazio libero, entrambi potrebbero superare il controllo e inserire due elementi, superando la capacità.

### Soluzione thread-safe
```java
// Soluzione thread-safe con sincronizzazione
public class SynchronizedBoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    
    public SynchronizedBoundedBuffer(int capacity) {
        this.capacity = capacity;
    }
    
    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();  // Attende finché non c'è spazio
        }
        queue.add(item);
        notifyAll();  // Notifica i thread in attesa di take()
    }
    
    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();  // Attende finché non ci sono elementi
        }
        T item = queue.poll();
        notifyAll();  // Notifica i thread in attesa di put()
        return item;
    }
}
```

## Conclusioni e Best Practices

Per gestire correttamente questi pattern critici nella programmazione concorrente, si possono seguire queste pratiche consigliate:

1. **Utilizzare la sincronizzazione**: La sincronizzazione (con `synchronized`) garantisce che solo un thread alla volta possa eseguire il blocco di codice.

2. **Utilizzare classi atomiche**: Le classi come `AtomicInteger`, `AtomicBoolean`, ecc. forniscono operazioni atomiche senza richiedere la sincronizzazione esplicita.

3. **Utilizzare collezioni thread-safe**: Classi come `ConcurrentHashMap`, `CopyOnWriteArrayList`, ecc. forniscono implementazioni thread-safe di strutture dati comuni.

4. **Utilizzare lock espliciti**: I lock dell'interfaccia `java.util.concurrent.locks` offrono maggiore flessibilità rispetto alla sincronizzazione.

5. **Minimizzare lo stato condiviso**: Ridurre al minimo lo stato condiviso tra thread per minimizzare la necessità di sincronizzazione.

6. **Preferire l'immutabilità**: Gli oggetti immutabili sono intrinsecamente thread-safe e non richiedono sincronizzazione.

La chiave per gestire correttamente questi pattern è riconoscere quando si presentano e garantire che le operazioni composite vengano eseguite atomicamente rispetto ad altri thread che potrebbero accedere allo stesso stato condiviso.
