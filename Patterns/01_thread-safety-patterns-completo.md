# Pattern Fondamentali di Thread Safety in Java

## 1. Immutability Pattern

### Descrizione
L'Immutability Pattern si basa sul principio che gli oggetti, una volta creati, non possono essere modificati. Poiché gli oggetti immutabili non cambiano stato, possono essere condivisi liberamente tra thread senza bisogno di sincronizzazione.

### Quando utilizzarlo
- Quando hai bisogno di oggetti che possono essere condivisi tra thread senza rischi
- Quando vuoi garantire l'integrità dei dati senza utilizzare meccanismi di sincronizzazione
- Quando vuoi implementare un modello funzionale di programmazione
- Quando hai necessità di oggetti che possono essere utilizzati come chiavi in una mappa

### Esempio di codice

```java
public final class ImmutablePoint {
    private final int x;
    private final int y;
    
    public ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public int getX() {
        return x;
    }
    
    public int getY() {
        return y;
    }
    
    // Metodo che crea un nuovo oggetto anziché modificare l'esistente
    public ImmutablePoint translate(int deltaX, int deltaY) {
        return new ImmutablePoint(x + deltaX, y + deltaY);
    }
}
```

### Vantaggi
- Garantisce thread safety senza sincronizzazione
- Semplifica il design e il debug
- Migliora le prestazioni eliminando la necessità di lock
- Non può entrare in stati inconsistenti

### Svantaggi
- Può comportare un maggior utilizzo di memoria (creazione di nuovi oggetti per ogni modifica)
- Non adatto quando lo stato deve essere frequentemente modificato

## 2. Thread Confinement Pattern

### Descrizione
Il Thread Confinement Pattern consiste nel limitare l'accesso ai dati a un singolo thread. Se un oggetto è accessibile da un solo thread, non c'è bisogno di sincronizzazione.

### Quando utilizzarlo
- Quando puoi delegare operazioni specifiche a thread dedicati
- Quando usi framework come Swing (che utilizza l'Event Dispatch Thread)
- Quando vuoi evitare overhead di sincronizzazione
- Per implementare componenti con alta concorrenza (come pool di risorse)

### Esempio di codice

```java
public class ThreadLocalExample {
    // ThreadLocal garantisce che ogni thread abbia la propria copia della variabile
    private static final ThreadLocal<SimpleDateFormat> dateFormat = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
    
    public String formatDate(Date date) {
        // Ogni thread usa la propria istanza di SimpleDateFormat
        return dateFormat.get().format(date);
    }
}
```

### Tipi di Thread Confinement
1. **Ad-hoc Thread Confinement**: Gestito dal design dell'applicazione
2. **Thread Local Storage**: Utilizzo della classe `ThreadLocal`
3. **Stack Confinement**: Variabili locali accessibili solo dal thread corrente

### Vantaggi
- Elimina la necessità di sincronizzazione
- Migliora le prestazioni
- Facilita la scalabilità

### Svantaggi
- Richiede attenta progettazione
- Difficile da verificare/garantire (soprattutto per l'ad-hoc confinement)
- Può portare a sprechi di risorse (ogni thread ha la propria copia dei dati)

## 3. Monitor Pattern

### Descrizione
Il Monitor Pattern utilizza blocchi `synchronized` per proteggere l'accesso a risorse condivise. In Java, ogni oggetto ha un monitor intrinseco che può essere utilizzato per sincronizzare l'accesso.

### Quando utilizzarlo
- Quando più thread devono accedere a risorse condivise
- Quando è necessario garantire l'atomicità di operazioni complesse
- Quando hai bisogno di garantire la visibilità delle modifiche tra thread
- Per implementare invarianti su oggetti mutabili condivisi

### Esempio di codice

```java
public class BankAccount {
    private double balance;
    
    public BankAccount(double initialBalance) {
        this.balance = initialBalance;
    }
    
    // Il synchronized garantisce che un solo thread alla volta possa eseguire questo metodo
    public synchronized void deposit(double amount) {
        if (amount > 0) {
            double newBalance = balance + amount;
            // Simulazione di operazione che richiede tempo
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            balance = newBalance;
        }
    }
    
    public synchronized void withdraw(double amount) {
        if (amount > 0 && balance >= amount) {
            balance -= amount;
        } else {
            throw new IllegalArgumentException("Invalid withdrawal");
        }
    }
    
    public synchronized double getBalance() {
        return balance;
    }
}
```

### Vantaggi
- Meccanismo semplice e integrato nel linguaggio
- Garantisce atomicità, visibilità e ordinamento
- Permette di implementare invarianti complesse

### Svantaggi
- Può causare overhead di performance
- Rischio di deadlock se non gestito correttamente
- Granularità fissa (l'intero metodo è bloccato)

## 4. Safe Publication Pattern

### Descrizione
Il Safe Publication Pattern garantisce che un oggetto sia pubblicato (reso visibile ad altri thread) in modo che sia completamente costruito e inizializzato prima che altri thread possano accedervi.

### Quando utilizzarlo
- Quando condividi oggetti tra thread dopo la loro inizializzazione
- Quando implementi pattern come lazy initialization
- Quando utilizzi cache condivise
- Per garantire la corretta visibilità di oggetti immutabili o mutable in modo thread-safe

### Esempio di codice

```java
public class SafePublicationExample {
    // L'uso di volatile garantisce la visibilità tra thread
    private volatile ExpensiveObject instance = null;
    
    // Lazy initialization thread-safe
    public ExpensiveObject getInstance() {
        if (instance == null) {
            synchronized (this) {
                if (instance == null) {
                    instance = new ExpensiveObject();
                }
            }
        }
        return instance;
    }
    
    // Alternativa più moderna (Java 5+)
    private static class InstanceHolder {
        static final ExpensiveObject INSTANCE = new ExpensiveObject();
    }
    
    public static ExpensiveObject getInstanceStatic() {
        return InstanceHolder.INSTANCE;
    }
}

class ExpensiveObject {
    // Un oggetto complesso che richiede inizializzazione
}
```

### Meccanismi di Safe Publication
1. **Inizializzazione statica**: Il classloader garantisce la pubblicazione sicura
2. **Riferimento volatile**: Garantisce la visibilità tra thread
3. **Accesso sincronizzato**: Usando `synchronized`
4. **Concurrent Collections**: Come `ConcurrentHashMap`
5. **Classi di utilità concorrenti**: Come `BlockingQueue` o `Future`

### Vantaggi
- Previene l'accesso a oggetti parzialmente costruiti
- Garantisce la visibilità delle modifiche tra thread
- Può essere implementato in vari modi secondo le esigenze

### Svantaggi
- Può introdurre overhead (specialmente con volatile)
- Richiede attenzione ai dettagli dell'implementazione
- Facile da implementare in modo errato

## Considerazioni sulla scelta del pattern

La scelta del pattern di concorrenza dipende da diversi fattori:

1. **Requisiti di prestazione**: Alcuni pattern introducono overhead maggiore di altri
2. **Complessità del codice**: Alcuni pattern sono più semplici da implementare e mantenere
3. **Natura dei dati**: Se i dati cambiano frequentemente o raramente
4. **Scalabilità richiesta**: Alcuni pattern si scalano meglio con un alto numero di thread
5. **Compatibilità con il design esistente**: Alcuni pattern si integrano meglio con architetture specifiche

In molti casi reali, una combinazione di questi pattern può offrire la soluzione ottimale.
