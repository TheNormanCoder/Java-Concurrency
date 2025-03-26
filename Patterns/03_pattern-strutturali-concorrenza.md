# Pattern Strutturali per la Programmazione Concorrente

## 1. Producer-Consumer

### Descrizione
Il pattern Producer-Consumer è un modello di comunicazione tra thread in cui un gruppo di thread (i producer) generano dati che vengono elaborati da un altro gruppo di thread (i consumer). La comunicazione avviene attraverso una struttura dati condivisa, tipicamente una coda.

### Quando usarlo
- Quando è necessario disaccoppiare la generazione dei dati dalla loro elaborazione
- Per gestire carichi di lavoro asincroni
- Per bilanciare la velocità di elaborazione tra producer e consumer
- Quando i dati devono essere elaborati nell'ordine in cui vengono generati

### Esempio in Java

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class ProducerConsumerExample {
    
    private static BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);
    
    // Classe Producer
    static class Producer implements Runnable {
        @Override
        public void run() {
            try {
                for (int i = 0; i < 50; i++) {
                    queue.put(i);  // Blocca se la coda è piena
                    System.out.println("Producer ha prodotto: " + i);
                    Thread.sleep(100);  // Simula il tempo necessario per produrre
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    // Classe Consumer
    static class Consumer implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    Integer value = queue.take();  // Blocca se la coda è vuota
                    System.out.println("Consumer ha consumato: " + value);
                    Thread.sleep(200);  // Simula il tempo necessario per consumare
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    public static void main(String[] args) {
        // Avvia il producer
        Thread producerThread = new Thread(new Producer());
        producerThread.start();
        
        // Avvia il consumer
        Thread consumerThread = new Thread(new Consumer());
        consumerThread.start();
    }
}
```

## 2. Task Execution

### Descrizione
Il pattern Task Execution si concentra sull'organizzazione e l'esecuzione di task indipendenti in modo efficiente, sfruttando il parallelismo. Questo pattern generalmente utilizza un pool di thread per eseguire i task.

### Quando usarlo
- Quando si hanno molti task indipendenti da eseguire
- Per limitare il numero di thread attivi simultaneamente
- Per riutilizzare i thread invece di crearne di nuovi per ogni task
- Per migliorare le prestazioni in applicazioni con molte operazioni brevi e indipendenti

### Esempio in Java con ExecutorService

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class TaskExecutionExample {
    
    // Una semplice classe Task
    static class Task implements Runnable {
        private final int id;
        
        public Task(int id) {
            this.id = id;
        }
        
        @Override
        public void run() {
            try {
                System.out.println("Task " + id + " in esecuzione su thread " + 
                                   Thread.currentThread().getName());
                // Simula il lavoro
                Thread.sleep((long) (Math.random() * 1000));
                System.out.println("Task " + id + " completato");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    public static void main(String[] args) {
        // Crea un pool di thread con un numero fisso di thread
        int numThreads = Runtime.getRuntime().availableProcessors();
        ExecutorService executor = Executors.newFixedThreadPool(numThreads);
        
        // Sottometti 20 task all'executor
        for (int i = 0; i < 20; i++) {
            executor.submit(new Task(i));
        }
        
        // Chiudi l'executor quando tutti i task sono stati completati
        executor.shutdown();
        try {
            if (!executor.awaitTermination(1, TimeUnit.MINUTES)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

## 3. Work Stealing

### Descrizione
Il pattern Work Stealing permette ai thread inattivi di "rubare" lavoro dalle code di lavoro di altri thread occupati. Questo approccio bilancia dinamicamente il carico di lavoro tra i thread.

### Quando usarlo
- Quando i task hanno durate imprevedibili o molto variabili
- Per bilanciare automaticamente il carico tra i thread
- In sistemi con molti processori o core
- Per task ricorsivi, dove nuovi subtask possono essere generati durante l'esecuzione

### Esempio in Java con ForkJoinPool

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class WorkStealingExample {
    
    // Task ricorsivo per calcolare il fattoriale
    static class FactorialTask extends RecursiveTask<Long> {
        private final int n;
        private static final int THRESHOLD = 10;
        
        public FactorialTask(int n) {
            this.n = n;
        }
        
        @Override
        protected Long compute() {
            if (n <= THRESHOLD) {
                return computeDirectly();
            }
            
            // Dividi il problema
            FactorialTask subtask = new FactorialTask(n - 1);
            subtask.fork();  // Esegui il subtask in modo asincrono
            
            // Calcola la parte locale e combina i risultati
            long result = n * subtask.join();
            return result;
        }
        
        private long computeDirectly() {
            long result = 1;
            for (int i = 1; i <= n; i++) {
                result *= i;
            }
            System.out.println("Calcolato fattoriale di " + n + " su thread " + 
                               Thread.currentThread().getName());
            return result;
        }
    }
    
    public static void main(String[] args) {
        // Crea un pool con il meccanismo di work stealing
        ForkJoinPool pool = new ForkJoinPool();
        
        // Crea e inizia il task
        FactorialTask task = new FactorialTask(20);
        long result = pool.invoke(task);
        
        System.out.println("Fattoriale di 20 = " + result);
        pool.shutdown();
    }
}
```

## 4. Task Splitting

### Descrizione
Il pattern Task Splitting (o Divide and Conquer) consiste nel suddividere compiti grandi in attività più piccole che possono essere eseguite in parallelo e poi combinare i risultati.

### Quando usarlo
- Per problemi che possono essere suddivisi in sottoproblemi indipendenti
- Quando l'elaborazione di grandi set di dati può essere parallelizzata
- Per algoritmi ricorsivi come quicksort, mergesort, ecc.
- Per migliorare le prestazioni di operazioni computazionalmente intensive

### Esempio in Java (Calcolo parallelo della somma di un array)

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class TaskSplittingExample {
    
    static class SumTask extends RecursiveTask<Long> {
        private final int[] array;
        private final int start;
        private final int end;
        private static final int THRESHOLD = 1000;
        
        public SumTask(int[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected Long compute() {
            int length = end - start;
            
            if (length <= THRESHOLD) {
                // Calcola direttamente per array piccoli
                return computeDirectly();
            }
            
            // Dividi il problema in due parti
            int middle = start + length / 2;
            
            SumTask leftTask = new SumTask(array, start, middle);
            SumTask rightTask = new SumTask(array, middle, end);
            
            // Esegui il task sinistro in modo asincrono
            leftTask.fork();
            
            // Calcola il task destro
            long rightResult = rightTask.compute();
            
            // Attendi il completamento del task sinistro
            long leftResult = leftTask.join();
            
            // Combina i risultati
            return leftResult + rightResult;
        }
        
        private long computeDirectly() {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            System.out.println("Calcolata somma da " + start + " a " + end + 
                              " su thread " + Thread.currentThread().getName());
            return sum;
        }
    }
    
    public static void main(String[] args) {
        // Crea un array di esempio
        int size = 100000;
        int[] array = new int[size];
        for (int i = 0; i < size; i++) {
            array[i] = i + 1;
        }
        
        // Utilizza ForkJoinPool per eseguire il task
        ForkJoinPool pool = new ForkJoinPool();
        SumTask task = new SumTask(array, 0, size);
        long sum = pool.invoke(task);
        
        System.out.println("Somma totale: " + sum);
        
        // Verifica del risultato
        long expectedSum = ((long)size * (size + 1)) / 2;
        System.out.println("Risultato atteso: " + expectedSum);
        
        pool.shutdown();
    }
}
```

## 5. Lock Striping

### Descrizione
Il Lock Striping è una tecnica che consiste nel dividere un singolo lock in più lock indipendenti, ciascuno responsabile di proteggere una parte differente della struttura dati. Questo riduce la contesa e migliora la concorrenza.

### Quando usarlo
- Per strutture dati concorrenti che supportano operazioni parallele
- Quando un singolo lock diventa un collo di bottiglia
- Per hash table e altre strutture indicizzate
- In sistemi con elevata concorrenza e molti thread

### Esempio in Java (Implementazione di una HashMap concorrente con lock striping)

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantLock;

public class LockStripingExample<K, V> {
    
    // Numero di stripe (lock separati)
    private final int STRIPE_COUNT = 16;
    
    // Array di lock
    private final ReentrantLock[] locks;
    
    // Array di mappe separate
    private final Map<K, V>[] segments;
    
    @SuppressWarnings("unchecked")
    public LockStripingExample() {
        // Inizializza i lock
        locks = new ReentrantLock[STRIPE_COUNT];
        for (int i = 0; i < STRIPE_COUNT; i++) {
            locks[i] = new ReentrantLock();
        }
        
        // Inizializza le mappe
        segments = new HashMap[STRIPE_COUNT];
        for (int i = 0; i < STRIPE_COUNT; i++) {
            segments[i] = new HashMap<>();
        }
    }
    
    // Calcola l'indice del segmento basato sulla chiave
    private int getSegmentIndex(K key) {
        // Utilizziamo l'hashcode della chiave per determinare il segmento
        int hash = key.hashCode();
        // Assicurati che sia positivo e prendi il modulo rispetto a STRIPE_COUNT
        return Math.abs(hash % STRIPE_COUNT);
    }
    
    // Metodo per inserire una coppia chiave-valore
    public V put(K key, V value) {
        int segmentIndex = getSegmentIndex(key);
        locks[segmentIndex].lock();
        try {
            return segments[segmentIndex].put(key, value);
        } finally {
            locks[segmentIndex].unlock();
        }
    }
    
    // Metodo per ottenere un valore data una chiave
    public V get(K key) {
        int segmentIndex = getSegmentIndex(key);
        locks[segmentIndex].lock();
        try {
            return segments[segmentIndex].get(key);
        } finally {
            locks[segmentIndex].unlock();
        }
    }
    
    // Metodo per rimuovere una chiave
    public V remove(K key) {
        int segmentIndex = getSegmentIndex(key);
        locks[segmentIndex].lock();
        try {
            return segments[segmentIndex].remove(key);
        } finally {
            locks[segmentIndex].unlock();
        }
    }
    
    // Esempio di utilizzo
    public static void main(String[] args) {
        final LockStripingExample<String, Integer> map = new LockStripingExample<>();
        
        // Crea più thread che operano sulla mappa
        Runnable task = () -> {
            String threadName = Thread.currentThread().getName();
            for (int i = 0; i < 1000; i++) {
                String key = threadName + "-" + i;
                map.put(key, i);
                
                // Simula altre operazioni
                if (i % 10 == 0) {
                    String lookupKey = threadName + "-" + (i / 2);
                    Integer value = map.get(lookupKey);
                    System.out.println(threadName + " ha trovato il valore " + 
                                      value + " per la chiave " + lookupKey);
                }
                
                if (i % 100 == 0) {
                    String removeKey = threadName + "-" + (i / 4);
                    map.remove(removeKey);
                    System.out.println(threadName + " ha rimosso la chiave " + removeKey);
                }
            }
        };
        
        // Avvia 8 thread
        Thread[] threads = new Thread[8];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(task, "Thread-" + i);
            threads[i].start();
        }
        
        // Attendi il completamento di tutti i thread
        for (Thread thread : threads) {
            try {
                thread.join();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        System.out.println("Tutti i thread hanno completato le operazioni sulla mappa");
    }
}
```

## Conclusione

Questi pattern strutturali per la programmazione concorrente rappresentano soluzioni collaudate a problemi comuni nella programmazione multi-thread. L'uso appropriato di questi pattern può portare a significativi miglioramenti delle prestazioni e della scalabilità delle applicazioni, specialmente quando si lavora con hardware multi-core.

La scelta del pattern giusto dipende dalle specifiche esigenze dell'applicazione, dal tipo di operazioni da eseguire e dalle caratteristiche del carico di lavoro. Spesso, combinazioni di questi pattern possono essere utilizzate per ottenere i migliori risultati.
