<!--
.. title: Test riproducibili con il tempo (parte 2)
.. slug: test-riproducibili-con-il-tempo-parte-2
.. date: 2018-06-14 22:52:15 UTC
.. tags: rust
.. category: programming
.. type: text
-->

Nella [parte 1](/2018/06/test-riproducibili-con-il-tempo-parte-1.html) di questo blog post eravamo arrivati ad avere una situazione del tipo: 

```rust
use std::marker::PhantomData;
struct Event<T> {
    message: String,
    stamp: DateTime<Utc>,
    mark: PhantomData<T>,
}
trait Now {
    fn now() -> DateTime<Utc> {
        Utc::now()
    }
}
impl<T: Now> Event<T> {
    fn new(message: String) -> Event<T> {
        Event {
            message,
            stamp: T::now(),
            mark: PhantomData,
        }
    }
}

struct SystemNow;
impl Now for SystemNow {};

let _event: Event<SystemNow> = Event::new("Messaggio".to_string());
``` 

Quindi adesso è facile creare un trait che restituisce un momento fisso nel tempo: 

<!-- TEASER_END -->

```rust
struct FixedNow;

impl Now for FixedNow {
    fn now() -> DateTime<Utc> {
        Utc.ymd(2012, 11, 23).and_hms(18, 53, 7)
    }
}

let event: Event<FixedNow> = Event::new("Messaggio 2012".to_string());
assert_eq!(event.stamp.year(), 2012);
``` 

Però, molto spesso avremo bisogno di avere una sequenza di particolari timestamp che vadano a testare alcuni aspetti critici della nostra implementazione. 

Nel caso del nostro trait abbiamo un problema, abbiamo definito un metodo statico, non abbiamo nessun `self` in cui memorizzare l'evoluzione della nostra sequenza di timestamp. Dobbiamo trovare qualche altra soluzione, e ci sono almeno un paio di alternative. 

Nel caso in cui siamo sicuri che nel test il metodo `now()` verrà chiamato sempre e solo da un singolo thread, ce la possiamo cavare con una variabile `static` [^1], anzi ad essere precisi `static mut`, condita con un pizzico di `unsafe`. 

Le variabili `static` hanno una posizione fissa nella memoria durante l'esecuzione del programma, se si cerca di definire una variabile `static mut` non c'è modo un Rust _safe_ di modificarla, perché non c'è modo di garantire l'atomicità delle modifiche ad una variabile che può essere potenzialmente modificata da più thread in contemporanea. Però nel nostro caso, in un ambiente di test controllato, possiamo usare una variabile `static` per avere uno stato in un metodo ... statico. 

```rust
static mut COUNTER: i32 = 0;
COUNTER = COUNTER + 1;
``` 

Provando a modificare direttamente direttamente la variabile `COUNTER` il compilatore ci segnala l'errore: 

```text
error[E0133]: use of mutable static requires unsafe function or block
 --> src/part2.rs:92:1
  |
4 | COUNTER = COUNTER + 1;
  | ^^^^^^^^^^^^^^^^^^^^^ use of mutable static
``` 

Per fortuna, ormai ci siamo abituati bene, il compilatore ci suggerisce anche una soluzione, usare un blocco `unsafe`. 

Un blocco `unsafe` in Rust è un blocco dove il compilatore permette di compiere operazioni più pericolose, e si _fida_ di quello che lo sviluppatore ha scritto. Appena usciti dal blocco `unsafe`, tutti i normali controlli torno attivi, quindi il blocco `unsafe` è la parte più pericolosa del codice, quella a cui in una review serve mettere doppia attenzione, ma non è diverso da scrivere in C o C++. 

A volte può essere utile (per questioni di performance) oppure necessario (come nel caso della comunicazione con altre librerie tramite l'interfaccia C), in questo caso è solo un esempio, perché come vedremo, la libreria standard ci offre qualche altra opzione. 

```rust
struct LoopNow;

impl Now for LoopNow {
    fn now() -> DateTime<Utc> {
        static mut COUNTER: i32 = 0;
        let inc = unsafe {
            COUNTER = (COUNTER + 1) % 10;
            COUNTER
        };
        Utc.ymd(2000 + inc, 9, 9).and_hms(1, 46, 40)
    }
}
let event: Event<LoopNow> = Event::new("Messaggio 2001".to_string());
assert_eq!(event.stamp.year(), 2001);
let event: Event<LoopNow> = Event::new("Messaggio 2002".to_string());
assert_eq!(event.stamp.year(), 2002);
``` 

Facciamo una breve digressione sulla variabile statica, cosa vuol dire, come funziona questa cosa magica che una funzione senza nessuna variabile globale può avere un suo stato? 

Beh, in realtà la `COUNTER` è in un certo senso globale, ed il suo valore iniziale viene definito prima della prima esecuzione della funzione, è proprio compilato nel codice del programma. Quello che succede nella funzione è andare a trovare l'indirizzo di quei 4 byte di memoria per leggerci e scriverci dentro, mentre sullo stack non c'è niente relativo a `COUNTER`. Per un approfondimento, nelle note c'è il link ad un articolo interessante [^5]. 

Questo è l'ASM generato con una versione ridotta dell'esempio [^6] che abbiamo usato qua (con un po' di cerimonia per non far ottimizzare tutto al compilatore). 

```asm
playground::count:
    addl    playground::count::COUNT(%rip), %edi
    movl    %edi, playground::count::COUNT(%rip)
    movl    %edi, %eax
    retq

playground::main:
    pushq   %rax
    movl    $1, %edi
    callq   playground::count
    movl    %eax, %edi
    callq   playground::count
    movl    %eax, %edi
    callq   std::process::exit@PLT
    ud2
``` 

La strana forma `playground::count::COUNT(%rip)` rappresenta simbolicamente l'offset rispetto all'indirizzo dell'istruzione corrente (+1 pare, ma è un dettaglio che non ci tange) a cui trovare la zona di memoria in cui scrivere. E l'indirizzo della zona di memoria è fisso e non cambia tra successive chiamate a `count`, mentre il valore contenuto cambia ogni volta. 

Adesso che abbiamo capito il trucco delle variabili static, abbiamo ancora un problema, non siamo ancora in grado di eseguire un test con più thread. Ma le moderne CPU ci vengono in aiuto. Possiamo definire una variabile atomica, una variabile speciale per cui non serviranno strutture di sincronizzazione particolari, perché è il processore stesso a garantire l'atomicità di un particolare set di operazioni. 

In un esempio minimale come il precedente [^7], con una variabile atomica l'asm generato per la funzione `count` è: 

```asm
playground::count:
    lock        xaddq   %rdi, playground::count::COUNTER(%rip)
    movq    %rdi, %rax
    retq
``` 

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

struct IncrementNow;

impl Now for IncrementNow {
    fn now() -> DateTime<Utc> {
        static COUNTER: AtomicUsize = AtomicUsize::new(1);
        let inc = COUNTER.fetch_add(1, Ordering::SeqCst) as i32;
        Utc.ymd(2000 + inc, 9, 9).and_hms(1, 46, 40)
    }
}
``` 

Le variabili atomiche [^3] sono il tassello base per la costruzione di primitive di sincronizzazione tra thread, in questo caso abbiamo semplicemente usato il valore più conservativo per l'`ordering`, perché approfondire questo argomento vorrebbe dire aprire un vaso di Pandora, che sinceramente non ho ancora aperto (almeno, ho solo sbirciato un po'). In breve, anche solo dal fatto che senza `usafe` il compilatore non ci segnala problemi, possiamo assumere che qualcun altro ha garantito che l'uso di questa primitiva è sicuro. 

Però così senza farsi notare abbiamo usato un concetto che ancora non abbiamo presentato, che invece sarà sicuramente saltato all'occhio al lettore attento: come possiamo mutare `COUNTER` se non lo abbiamo definito mutabile? 

Questo è un _pattern_ in Rust detto _interior mutability_, che sarebbe la possibilità di creare un contenitore ufficialmente non mutabile, che al suo interno si occupa di garantire alcune condizioni al contorno della modifica del valore interno. In questo caso tutte le variabili atomiche in `std::sync::atomic` sono `Sync`, cioè sono marcate in modo da garantire al compilatore che è possibile accedere a queste variabili da più thread contemporaneamente. 

Il più semplice tipo che offre una "scatola" immutabile per un contenuto mutabile è `Cell` [^4], ma in questo caso non c'è nessun marcatore `Sync`, quindi il compilatore non ci permetterà di usare `Cell` in contesti multithread, ma ci permette di avere più di un riferimento mutabile ad una singola variabile. E quindi cosa ci abbiamo guadagnato rispetto ad una normale refrerence stile C++? Semplicemente che nell'implementazione di `Cell` c'è un controllo ed una mutazione concorrente non prevista farà crashare il programma. Meglio un errore subito che corrompere silenziosamente i dati (per poi magari crashare comunque più tardi). 

```rust
type IncrementEvent = Event<IncrementNow>;

let event1 = IncrementEvent::new("IncrementEvent 1".to_owned());
assert_eq!(event1.stamp.year(), 2001);
assert_eq!(event1.stamp.minute(), 46);

let event2 = IncrementEvent::new("IncrementEvent 2".to_owned());
assert_eq!(event2.stamp.year(), 2002);
assert_eq!(event2.stamp.minute(), 46);
``` 

Verrebbe voglia di scrivere un test per verificare qualche corruzione della memoria nel caso dell'uso dello `static mut`, ma lo lascio come esercizio per il lettore. 

Un ringraziamento al gruppetto di Rustiti anonimi di [@develer](https://twitter.com/develer) per la review.

Grazie per la lettura, se questa mini serie vi piaciuta fatemelo sapere, potrebbe essere lo spunto per scrivere qualcos'altro, e non esitate a contattarmi (qua o [@naufraghi](https://twitter.com/naufraghi) su twitter) per segnalare errori od omissioni! 

[^1]: Vedi [`const` and `static`](https://doc.rust-lang.org/book/first-edition/const-and-static.html)       nella prima edizione del Rust Book oppure [Accessing or Modifying a Mutable Static       Variable](https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable)       nella seconda edizione del libro o anche [Static       items](https://doc.rust-lang.org/reference/items/static-items.html) nella reference del linguaggio. 

[^3]: [Module std::sync::atomic](https://doc.rust-lang.org/std/sync/atomic/) nella documentazione della       libreria standard. 

[^4]: [Module std::cell](https://doc.rust-lang.org/std/cell/index.html) nella documentazione della       libreria standard. 

[^5]: [Understanding C by learning       assembly](https://www.recurse.com/blog/7-understanding-c-by-learning-assembly), in particolare il       capitolo _Understanding static local variables_. 

[^6]: Esempio variabile [statica       mutabile](https://play.rust-lang.org/?gist=ec49981fd17228684cfe92a3e19f6ace&version=nightly&mode=release)       sul Playground. 

[^7]: Esempio variabile [statica       atomica](https://play.rust-lang.org/?gist=0ca32f2a756660b926327098db87158f&version=nightly&mode=release)       sul Playground. 

