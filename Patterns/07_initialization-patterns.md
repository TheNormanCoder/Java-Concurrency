# Pattern di Concorrenza per Inizializzazione in Java

## 1. Lazy Initialization

### Spiegazione
La Lazy Initialization (inizializzazione pigra) è un pattern che ritarda la creazione di un oggetto fino al momento in cui viene effettivamente utilizzato. Questo approccio è utile quando la creazione dell'oggetto è costosa in termini di risorse e l'oggetto potrebbe non essere sempre necessario durante l'esecuzione del programma.

### Quando usarlo
- Quando la creazione dell'oggetto è costosa (I/O, rete, database)
- Quando l'oggetto potrebbe non essere sempre necessario
- Quando si vuole ridurre il tempo di avvio dell'applicazione

### Problema in ambiente multi-thread
In un ambiente single-thread, il pattern è semplice da implementare, ma in un contesto multi-thread può generare problemi di race condition, con conseguente possibile creazione di più istanze o accesso a istanze parzialmente inizializzate.

### Esempio base (non thread-safe)

```java
public class ResourceManager {
    private ExpensiveResource resource;
    
    public ExpensiveResource getResource() {
        if (resource == null) {
            resource = new ExpensiveResource(); // Creazione costosa
        }
        return resource;
    }
}
```

### Esempio thread-safe con synchronized (semplice ma inefficiente)

```java
public class ResourceManager {
    private ExpensiveResource resource;
    
    public synchronized ExpensiveResource getResource() {
        if (resource == null) {
            resource = new ExpensiveResource(); 
        }
        return resource;
    }
}
```

Questo approccio è thread-safe ma introduce un collo di bottiglia poiché il metodo `synchronized` blocca l'accesso per tutti i thread anche dopo che l'oggetto è stato inizializzato.

## 2. Double-Checked Locking (DCL)

### Spiegazione
Il Double-Checked Locking è un tentativo di ottimizzare la Lazy Initialization in ambiente multi-thread, riducendo il costo della sincronizzazione. Verifica due volte la condizione null: una volta senza blocco e, se necessario, una seconda volta all'interno di un blocco sincronizzato.

### Quando usarlo
- **ATTENZIONE**: Questo pattern era problematico nelle versioni di Java precedenti alla 1.5 a causa del modello di memoria di Java
- In Java moderno (≥ 5) funziona correttamente solo se il campo è dichiarato `volatile`
- Generalmente sconsigliato; esistono alternative migliori

### Esempio (corretto solo con Java ≥ 5 e campo volatile)

```java
public class ResourceManager {
    private volatile ExpensiveResource resource; // volatile è essenziale!
    
    public ExpensiveResource getResource() {
        // Prima verifica (senza sincronizzazione)
        if (resource == null) {
            // Sincronizzazione solo se sembra necessario
            synchronized (this) {
                // Seconda verifica (con sincronizzazione)
                if (resource == null) {
                    resource = new ExpensiveResource();
                }
            }
        }
        return resource;
    }
}
```

### Perché era problematico
Senza la parola chiave `volatile`, l'inizializzazione dell'oggetto può essere riordinata dal compilatore o dalla JVM, portando a situazioni in cui un thread potrebbe vedere un riferimento non nullo a un oggetto non completamente inizializzato.

## 3. Initialization-on-demand Holder

### Spiegazione
Questo idioma sfrutta la garanzia di Java che una classe viene caricata e inizializzata solo quando necessario. L'inizializzazione delle classi è thread-safe secondo la specifica JVM, quindi questo pattern offre lazy initialization senza bisogno di sincronizzazione esplicita.

### Quando usarlo
- Quando si ha bisogno di inizializzazione lazy thread-safe per una classe singleton
- Quando si vuole evitare il costo della sincronizzazione esplicita
- Considerato il miglior approccio per l'inizializzazione lazy thread-safe in Java

### Esempio

```java
public class Singleton {
    // Costruttore privato per prevenire l'istanziazione diretta
    private Singleton() {
        System.out.println("Singleton instance created");
    }
    
    // Classe interna statica che contiene l'istanza
    private static class SingletonHolder {
        // Inizializzato solo quando la classe interna viene caricata
        static final Singleton INSTANCE = new Singleton();
    }
    
    // Metodo pubblico per accedere all'istanza
    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
    
    public void doSomething() {
        System.out.println("Singleton is doing something");
    }
}
```

### Vantaggi
1. Thread-safe senza sincronizzazione esplicita
2. Inizializzazione veramente lazy (avviene solo al primo accesso)
3. Semplice da implementare
4. Prestazioni migliori rispetto ad approcci che usano sincronizzazione esplicita

### Come funziona
- La classe `SingletonHolder` non viene caricata fino a quando il metodo `getInstance()` non viene chiamato
- Quando la classe viene caricata, la JVM garantisce l'inizializzazione sicura della variabile `INSTANCE`
- Ogni successiva chiamata a `getInstance()` restituisce semplicemente il riferimento all'istanza già creata

## Confronto dei pattern

| Pattern | Thread-Safety | Efficienza | Complessità | Raccomandazione |
|---------|--------------|------------|------------|-----------------|
| Lazy Initialization (base) | No | Alta | Bassa | Solo ambiente single-thread |
| Lazy Initialization (synchronized) | Sì | Bassa | Bassa | Sconsigliato per alte prestazioni |
| Double-Checked Locking | Sì (con volatile) | Media | Media | Meglio evitare, tranne casi specifici |
| Initialization-on-demand Holder | Sì | Alta | Bassa | Raccomandato per singleton |

## Conclusione

Per la maggior parte dei casi di inizializzazione lazy thread-safe, l'Initialization-on-demand Holder è il pattern preferito in Java moderno. È semplice, thread-safe, veramente lazy e non introduce overhead di sincronizzazione. Il Double-Checked Locking è stato a lungo problematico e, sebbene ora funzioni con `volatile` in Java moderno, è generalmente considerato più complesso e meno elegante dell'approccio con holder class.
