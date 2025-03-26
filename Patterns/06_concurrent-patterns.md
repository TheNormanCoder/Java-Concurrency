# Pattern di Concorrenza nelle Strutture Dati

In questa guida analizzeremo tre importanti pattern di concorrenza per le strutture dati come descritti da Brian Goetz nel suo libro "Java Concurrency in Practice":

1. Copy-On-Write
2. Non-blocking Algorithms
3. Read-Copy-Update (RCU)

Per ciascun pattern, fornirò una spiegazione dettagliata, casi d'uso appropriati e un esempio di implementazione in Java.

## 1. Copy-On-Write

### Spiegazione
Il pattern Copy-On-Write (COW) è una strategia di gestione della concorrenza in cui le operazioni di modifica non alterano la struttura dati originale, ma creano una nuova copia con le modifiche applicate. Questo approccio garantisce che i lettori abbiano sempre accesso a una versione coerente e immutabile della struttura, senza necessità di sincronizzazione.

### Quando utilizzarlo
- Scenari in cui le operazioni di lettura sono molto più frequenti delle operazioni di scrittura
- Quando è importante garantire che le letture non vengano mai bloccate
- Casi in cui è accettabile un costo maggiore per le operazioni di modifica (in termini di memoria e tempo)
- Strutture dati di dimensioni moderate (per evitare overhead eccessivo durante la copia)

### Svantaggi
- Costo elevato per le operazioni di modifica (copia dell'intera struttura)
- Consumo di memoria maggiore
- Non adatto a strutture dati molto grandi o con frequenti modifiche

### Esempio: CopyOnWriteArrayList

```java
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.Iterator;

public class CopyOnWriteExample {
    public static void main(String[] args) {
        // Creo una lista thread-safe che utilizza il pattern copy-on-write
        CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
        
        // Aggiungo elementi (ogni operazione crea una nuova copia interna)
        cowList.add("Uno");
        cowList.add("Due");
        cowList.add("Tre");
        
        // Creo un iteratore
        Iterator<String> iterator = cowList.iterator();
        
        // Modifico la lista mentre l'iteratore è attivo
        cowList.add("Quattro"); // Crea una nuova copia
        
        // L'iteratore continua a vedere la versione originale (snapshot)
        System.out.println("Elementi nell'iteratore:");
        while (iterator.hasNext()) {
            System.out.println(iterator.next());  // Stampa solo Uno, Due, Tre
        }
        
        // La lista contiene tutti gli elementi
        System.out.println("\nElementi nella lista aggiornata:");
        for (String element : cowList) {
            System.out.println(element);  // Stampa Uno, Due, Tre, Quattro
        }
    }
}
```

In questo esempio, `CopyOnWriteArrayList` mantiene la coerenza dell'iteratore anche quando la lista viene modificata. L'iteratore lavora su una snapshot immutabile della lista al momento della sua creazione, mentre le modifiche successive operano su nuove copie, garantendo che i lettori non vengano mai disturbati.

## 2. Non-blocking Algorithms (Algoritmi Non Bloccanti)

### Spiegazione
Gli algoritmi non bloccanti sono tecniche di sincronizzazione che permettono ai thread di progredire senza l'utilizzo di lock tradizionali. Si basano su primitive di sincronizzazione atomica come Compare-And-Swap (CAS) per coordinare le operazioni concorrenti senza bloccare i thread.

Le operazioni CAS permettono di confrontare un valore attuale con un valore atteso e, se corrispondono, sostituirlo atomicamente con un nuovo valore. Questo meccanismo consente di implementare strutture dati concorrenti senza i problemi tipici dei lock (deadlock, livelock, inversione di priorità).

### Quando utilizzarlo
- Sistemi ad alte prestazioni dove il blocking tradizionale crea colli di bottiglia
- Applicazioni con elevata contesa tra thread
- Casi in cui la latenza deve essere prevedibile (evitando blocchi indefiniti)
- Quando si vuole massimizzare la scalabilità su sistemi multi-core

### Livelli di garanzia
Gli algoritmi non bloccanti offrono diverse garanzie:
1. **Wait-freedom**: Ogni thread completa l'operazione in un numero finito di passi
2. **Lock-freedom**: Almeno un thread fa progressi in qualsiasi momento
3. **Obstruction-freedom**: Un thread isolato fa sempre progressi

### Esempio: AtomicReference con CAS

```java
import java.util.concurrent.atomic.AtomicReference;

public class NonBlockingStack<T> {
    private static class Node<T> {
        final T value;
        Node<T> next;
        
        Node(T value) {
            this.value = value;
        }
    }
    
    private final AtomicReference<Node<T>> head = new AtomicReference<>(null);
    
    public void push(T value) {
        Node<T> newHead = new Node<>(value);
        Node<T> oldHead;
        
        do {
            oldHead = head.get();
            newHead.next = oldHead;
        } while (!head.compareAndSet(oldHead, newHead));
        // L'operazione CAS fallisce se un altro thread ha modificato head
        // In tal caso, ripetiamo il tentativo con il nuovo valore di head
    }
    
    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        
        do {
            oldHead = head.get();
            if (oldHead == null) {
                return null;  // Stack vuoto
            }
            newHead = oldHead.next;
        } while (!head.compareAndSet(oldHead, newHead));
        
        return oldHead.value;
    }
    
    public static void main(String[] args) {
        NonBlockingStack<String> stack = new NonBlockingStack<>();
        
        // Test concorrente con più thread
        Runnable pushTask = () -> {
            for (int i = 0; i < 1000; i++) {
                stack.push("Item " + i);
                try {
                    Thread.sleep(1);  // Simulazione di lavoro
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        };
        
        Runnable popTask = () -> {
            for (int i = 0; i < 1000; i++) {
                String item = stack.pop();
                if (item != null) {
                    System.out.println("Popped: " + item);
                }
                try {
                    Thread.sleep(1);  // Simulazione di lavoro
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        };
        
        Thread pusher = new Thread(pushTask);
        Thread popper = new Thread(popTask);
        
        pusher.start();
        popper.start();
    }
}
```

In questo esempio, implementiamo uno stack non bloccante usando `AtomicReference` e operazioni CAS. L'algoritmo tenta di modificare lo stato dello stack e, se rileva che un altro thread ha effettuato modifiche nel frattempo, riprova l'operazione con i valori aggiornati. Questo approccio elimina la necessità di lock espliciti.

## 3. Read-Copy-Update (RCU)

### Spiegazione
Read-Copy-Update è un pattern di sincronizzazione ottimizzato per carichi di lavoro con letture frequenti e scritture rare. RCU consente ai lettori di accedere ai dati senza alcun overhead di sincronizzazione, mentre gli scrittori creano copie dei dati, le modificano e poi aggiornano i riferimenti in modo atomico.

A differenza del pattern Copy-On-Write, RCU non copia immediatamente tutta la struttura, ma gestisce versioni multiple dei dati e rimuove le versioni obsolete solo quando tutti i lettori hanno completato le loro operazioni.

### Quando utilizzarlo
- Sistemi con accessi in lettura estremamente frequenti e scritture rare
- Scenari dove le prestazioni di lettura sono critiche
- Kernel di sistemi operativi e software di basso livello
- Strutture dati con costi di copia elevati

### Fasi principali
1. **Read**: I lettori accedono ai dati senza blocchi
2. **Copy**: Gli scrittori creano una copia dei dati da modificare
3. **Update**: Gli scrittori modificano la copia
4. **Commit**: Aggiornamento atomico del riferimento alla nuova versione
5. **Clean-up**: Rimozione delle versioni obsolete quando non sono più in uso

### Esempio: Implementazione semplificata di RCU

```java
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Function;

public class RCUList<T> {
    private static class Node<T> {
        final T value;
        final Node<T> next;
        
        Node(T value, Node<T> next) {
            this.value = value;
            this.next = next;
        }
    }
    
    private final AtomicReference<Node<T>> head = new AtomicReference<>(null);
    
    // Operazione di lettura - non richiede sincronizzazione
    public void forEach(Function<T, Void> action) {
        // "Entrata" nella sezione RCU
        Node<T> current = head.get();
        
        // Attraversamento della lista (read-side critical section)
        while (current != null) {
            action.apply(current.value);
            current = current.next;
        }
        
        // "Uscita" dalla sezione RCU
        // In una implementazione completa, qui notificheremmo il completamento della lettura
    }
    
    // Operazione di modifica
    public void update(T newValue) {
        // Fase 1: Crea una copia della lista
        Node<T> oldHead = head.get();
        Node<T> newHead = new Node<>(newValue, oldHead);
        
        // Fase 2: Aggiorna atomicamente il riferimento
        head.set(newHead);
        
        // Fase 3: In una implementazione completa, qui si gestirebbe
        // la rimozione sicura dei nodi obsoleti quando tutti i lettori
        // hanno completato le loro operazioni
    }
    
    public static void main(String[] args) {
        RCUList<String> list = new RCUList<>();
        
        // Aggiungiamo elementi
        list.update("Uno");
        list.update("Due");
        list.update("Tre");
        
        // Thread di lettura
        Runnable reader = () -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Lettura (Thread " + Thread.currentThread().getId() + "):");
                list.forEach(item -> {
                    System.out.println("  - " + item);
                    try {
                        Thread.sleep(10);  // Simulazione di operazione lenta
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                    return null;
                });
            }
        };
        
        // Thread di scrittura
        Runnable writer = () -> {
            for (int i = 0; i < 3; i++) {
                String newValue = "Item " + System.currentTimeMillis();
                System.out.println("Aggiornamento: " + newValue);
                list.update(newValue);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        };
        
        // Avvio dei thread
        Thread[] readers = new Thread[3];
        for (int i = 0; i < readers.length; i++) {
            readers[i] = new Thread(reader);
            readers[i].start();
        }
        
        Thread writerThread = new Thread(writer);
        writerThread.start();
    }
}
```

Questa implementazione è semplificata rispetto a una vera implementazione RCU, che richiederebbe meccanismi di grace period per la rimozione sicura dei dati obsoleti. Tuttavia, illustra il concetto fondamentale: i lettori accedono ai dati senza blocchi, mentre gli scrittori creano nuove versioni e aggiornano atomicamente i riferimenti.

## Confronto tra i Pattern

| Pattern | Vantaggi | Svantaggi | Casi d'uso ideali |
|---------|----------|-----------|-------------------|
| **Copy-On-Write** | - Letture senza sincronizzazione<br>- Semplicità d'implementazione<br>- Letture mai bloccate | - Alto costo per le modifiche<br>- Consumo di memoria elevato<br>- Non adatto a strutture grandi | - Letture molto frequenti<br>- Modifiche rare<br>- Strutture di dimensioni moderate |
| **Non-blocking** | - Evita problemi dei lock<br>- Alta scalabilità<br>- Nessun blocco dei thread | - Complessità implementativa<br>- Rischio di starvation<br>- Difficile debug | - Sistemi ad alte prestazioni<br>- Applicazioni con alta contesa<br>- Microservizi | 
| **RCU** | - Letture a costo zero<br>- Scalabilità eccellente<br>- Efficienza di memoria | - Complessità di gestione<br>- Ritardo nella liberazione memoria<br>- Implementazione complessa | - Kernel di sistemi operativi<br>- Database ad alte prestazioni<br>- Sistemi real-time |

## Conclusioni

La scelta del pattern di concorrenza dipende fortemente dal caso d'uso specifico e dai requisiti prestazionali:

- **Copy-On-Write** è la scelta più semplice per strutture di dimensioni moderate con poche modifiche
- **Non-blocking Algorithms** offrono il miglior compromesso tra prestazioni e complessità per la maggior parte delle applicazioni
- **RCU** è ideale per scenari estremamente asimmetrici, dove le letture sono ordini di grandezza più frequenti delle scritture

In molti sistemi reali, potrebbe essere necessario combinare diversi pattern per ottenere le migliori prestazioni complessive.
