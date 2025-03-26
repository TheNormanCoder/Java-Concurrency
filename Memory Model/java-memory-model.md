# Il Java Memory Model (JMM)

## Introduzione

Il Java Memory Model (JMM) è un aspetto fondamentale della concorrenza in Java, ampiamente trattato nel libro "Java Concurrency in Practice" di Brian Goetz. Il JMM definisce le regole per come la memoria viene gestita in un ambiente multi-thread, stabilendo le garanzie di visibilità e ordinamento delle operazioni di memoria tra i diversi thread.

## Concetti Chiave del JMM

### 1. Problema di Visibilità

Senza sincronizzazione adeguata, un thread non ha la garanzia di vedere i cambiamenti effettuati da un altro thread. Questo problema è noto come "problema di visibilità".

```java
public class VisibilityProblem {
    private static boolean ready;
    private static int number;
    
    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) {
                Thread.yield();
            }
            System.out.println(number);
        }
    }
    
    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

In questo esempio, senza adeguate garanzie di sincronizzazione, il thread lettore potrebbe non vedere mai il valore aggiornato di `ready` o potrebbe vedere `ready` come true ma `number` ancora come 0.

### 2. Riordinamento delle Istruzioni

Il JVM, i compilatori e i processori possono riordinare le istruzioni per ottimizzare le prestazioni. Questo riordinamento può creare comportamenti inattesi in contesti multi-thread.

**Prima del riordinamento:**
```java
number = 42;
ready = true;
```

**Dopo possibile riordinamento:**
```java
ready = true;
number = 42;
```

### 3. Happens-Before Relationship

Il JMM definisce un insieme di relazioni "happens-before" che garantiscono l'ordinamento delle operazioni di memoria:

1. **Program Order Rule**: Ogni azione in un thread happens-before ogni azione successiva in quell'ordine del programma.
2. **Monitor Lock Rule**: Un rilascio di un lock happens-before di ogni successiva acquisizione dello stesso lock.
3. **Volatile Variable Rule**: Una scrittura su una variabile volatile happens-before di ogni successiva lettura della stessa variabile.
4. **Thread Start Rule**: Una chiamata a `Thread.start()` happens-before di ogni azione nel thread avviato.
5. **Thread Termination Rule**: Qualsiasi azione in un thread happens-before di qualsiasi altro thread che rileva la terminazione di quel thread.
6. **Transitivity**: Se A happens-before B e B happens-before C, allora A happens-before C.

## Meccanismi di Sincronizzazione nel JMM

### 1. Synchronized

L'uso del blocco `synchronized` garantisce sia la mutua esclusione che la visibilità delle modifiche alla memoria.

```java
public class SynchronizedCounter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

### 2. Variabili Volatile

Le variabili dichiarate come `volatile` garantiscono che le letture accedano alla copia più recente e che le scritture siano immediatamente visibili a tutti i thread.

```java
public class VolatileVisibility {
    private volatile boolean ready = false;
    private int number = 0;
    
    public void writer() {
        number = 42;
        ready = true;
    }
    
    public void reader() {
        while (!ready) {
            Thread.yield();
        }
        // Qui avremo sempre la garanzia di vedere number = 42
        System.out.println(number);
    }
}
```

### 3. java.util.concurrent.atomic

Le classi atomiche, come `AtomicInteger`, forniscono operazioni atomiche e garantiscono la visibilità.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();
    }
    
    public int getCount() {
        return count.get();
    }
}
```

## Schemi di Memoria in Java

### 1. Memory Barrier (Barriera di Memoria)

```
Thread 1           |  Memory Barrier  |           Thread 2
-----------------  |  --------------  |  -----------------
write x = 1        |                  |  read x
write y = 2        |                  |  read y
```

Le operazioni `volatile` e l'entrata/uscita dai blocchi `synchronized` introducono memory barrier che garantiscono che le operazioni non vengano riordinate attraverso la barriera.

### 2. Schema di Pubblicazione di Oggetti

#### Pubblicazione non sicura

```java
// Thread 1
holder = new Holder(42);  // Pubblicazione prematura possibile

// Thread 2
if (holder != null) {
    int n = holder.n;  // Potrebbe vedere uno stato inconsistente
}
```

#### Pubblicazione sicura

```java
// Dichiarazione
private volatile Holder holder;

// Thread 1
holder = new Holder(42);  // Pubblicazione sicura

// Thread 2
if (holder != null) {
    int n = holder.n;  // Garantito di vedere lo stato corretto
}
```

## Modelli di Memoria Hardware e JMM

Il JMM è progettato per essere un'astrazione che funziona su diverse architetture hardware, ciascuna con il proprio modello di memoria:

1. **Modello x86/x64**: Relativamente forte, con garanzie di ordinamento più robuste.
2. **Modello ARM/POWER**: Più debole, permette più riordinamento per ottimizzare le prestazioni.

Il JMM fornisce un'interfaccia coerente su tutte queste architetture, permettendo ai programmatori Java di scrivere codice portabile.

```
           JMM (Java Memory Model)
               /           \
              /             \
Modello x86/x64         Modello ARM/POWER
```

## Esempi Pratici di Problemi JMM

### 1. Double-Checked Locking (Errato)

```java
public class SingletonBroken {
    private static SingletonBroken instance;
    
    private SingletonBroken() {}
    
    public static SingletonBroken getInstance() {
        if (instance == null) {           // Check 1
            synchronized (SingletonBroken.class) {
                if (instance == null) {    // Check 2
                    instance = new SingletonBroken();
                }
            }
        }
        return instance;
    }
}
```

Questo pattern può fallire perché `instance = new SingletonBroken()` non è atomico e può essere riordinato.

### 2. Double-Checked Locking (Corretto)

```java
public class SingletonFixed {
    private static volatile SingletonFixed instance;
    
    private SingletonFixed() {}
    
    public static SingletonFixed getInstance() {
        if (instance == null) {
            synchronized (SingletonFixed.class) {
                if (instance == null) {
                    instance = new SingletonFixed();
                }
            }
        }
        return instance;
    }
}
```

### 3. Inizializzazione Lazy con Holder Class

```java
public class SingletonHolder {
    private SingletonHolder() {}
    
    private static class Holder {
        static final SingletonHolder INSTANCE = new SingletonHolder();
    }
    
    public static SingletonHolder getInstance() {
        return Holder.INSTANCE;
    }
}
```

Questo pattern sfrutta le garanzie del JMM per l'inizializzazione delle classi.

## Considerazioni Finali

Il JMM è cruciale per la scrittura di codice concorrente corretto in Java. Comprendere le sue regole e i suoi meccanismi consente di evitare sottili bug di concorrenza e di scrivere codice performante e sicuro.

Ricorda sempre:
1. Utilizzare la sincronizzazione appropriata per garantire la visibilità.
2. Prestare attenzione ai possibili riordinamenti delle istruzioni.
3. Comprendere le relazioni happens-before per ragionare sulla correttezza del codice.
4. Utilizzare le classi di java.util.concurrent quando possibile, poiché forniscono implementazioni già verificate e ottimizzate.
