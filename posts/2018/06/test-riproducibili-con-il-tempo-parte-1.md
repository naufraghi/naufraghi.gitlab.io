<!--
.. title: Test riproducibili con il tempo (parte 1)
.. slug: test-riproducibili-con-il-tempo-parte-1
.. date: 2018-05-15 22:42:57 UTC
.. tags: rust
.. category: programming
.. type: text
-->

Ormai è difficile non aver sentito parlare di Rust, il nuovo linguaggio di Mozilla, che promette la velocità del C++ senza troppi grattacapi. Il bello di Rust è che pur essendo un linguaggio "di sistema", cioè un linguaggio con un controllo dell'esecuzione simile a quello offerto dal C, è anche un linguaggio che cura molto l'esperienza dello sviluppatore, con tool ergonomici e messaggi di errori di prima qualità. 

Quindi che siate un rodato sviluppatore C/C++ o uno sviluppatore Python / Javascript / Ruby, se avete voglia di aggiungere un nuovo linguaggio al vostro repertorio, fate una prova con Rust! 

Ormai sto usando Rust come hobby da un annetto, non ho scritto niente di serio ma mi piace cercare di trovare una soluzione ai tipici problemi che sono abituato ad affrontare in Python, magari riuscendo ad avere quel controllo in più a compile time che in Python è ancora un po' faticoso da ottenere. 

Quindi non prendete come oro colato quello che scrivo, anzi, se ci fossero correzioni o suggerimenti contattatemi pure. 

Immaginiamoci quindi programmatori Rust non navigati, stiamo scrivendo la nostra libreria in Rust, che sia una "indovina il numero" o una lista della spesa, avremo qualche evento a cui vorremo rispondere, e per rendere l'informazione più completa, vogliamo anche avere un timestamp. 

Una prima implementazione potrebbe essere qualcosa del tipo: 

```rust
extern crate chrono;
use chrono::prelude::*;

struct Event {
    message: String,
    stamp: DateTime<Utc>,
}

impl Event {
    fn new(message: String) -> Event {
        Event {
            message,
            stamp: Utc::now(),
        }
    }
}
``` 

Ma ora si pone un problema, come si possono scrivere test di una cosa del genere? In Python si possono usare mock nei modi più arditi, ma in Rust è tutta un'altra storia, dobbiamo preparare un po' la strada per iniettare questa dipendenza. 

Un approccio potrebbe essere quella di definire un `Trait` [^2] che rappresenti la funzionalità di poter ottenere l'ora corrente. 

```rust
trait Now {
    fn now() -> DateTime<Utc> {
        Utc::now()
    }
}
``` 

Magari non vogliamo che sia proprio `Event` ad implementare questo `Trait`, ma una altra struttura di utilità. 

```rust
struct SystemNow;
impl Now for SystemNow {};
``` 

Non serve scrivere molto, stiamo definendo una struttura vuota che usa l'implementazione di default del trait. 

Adesso possiamo far usare questo nuovo tipo al nostro `Event`, ma se facciamo la cosa semplice: 

```rust
impl Event {
    fn new(message: String) -> Event {
        Event {
            message,
            stamp: SystemNow::now(),
        }
    }
}
``` 

Non ci abbiamo guadagnato molto, non sappiamo ancora come modificare quel `now()` nei test. Potremmo passare una istanza come argomento, e chiamare il metodo `now()`, ma nell'uso reale non ci serve che sia una istanze, e non vogliamo creare una astrazione "costosa" solo per rendere il codice più testabile, più che altro quando ci sono alternative a costo zero! 

In Rust quasi tutto può essere reso generico rispetto al tipo, `Option<i32>` ed `Option<String>` sono due _oggetti_ molto simili, la stessa scatola con un contenuto diverso. Ed indipendentemente dal tipo avremo tutti i metodi di `Option` [^3] disponibili in entrambi i casi. 

Potremmo definire un `Event<T>` e poi usare come `T` il nostro `SystemNow`, proviamo: 

```rust
struct Event<T> {
    message: String,
    stamp: DateTime<Utc>,
}
``` 

Ma otteniamo un errore di compilazione: 

```text
error[E0392]: parameter `T` is never used
 --> src/lib.rs:97:14
  |
6 | struct Event<T> {
  |              ^ unused type parameter
  |
  = help: consider removing `T` or using a marker such as `std::marker::PhantomData`
``` 

Hmmm, qua c'è un suggerimento interessante, se vogliamo associare un tipo alla nostra struttura, ma non vogliamo averne una istanza, possiamo usare un marcatore [^1]. 

```rust
use std::marker::PhantomData;

struct Event<T> {
    message: String,
    stamp: DateTime<Utc>,
    mark: PhantomData<T>,
}

impl<T> Event<T> {
    fn new(message: String) -> Event<T> {
        Event {
            message,
            stamp: T::now(),
            mark: PhantomData,
        }
    }
}
``` 

Ma anche in questo caso abbiamo un errore: 

```text
error[E0599]: no function or associated item named `now` found for type `T` in the current scope
  --> src/lib.rs:131:20
   |
22 |             stamp: T::now(),
   |                    ^^^^^^ function or associated item not found in `T`
   |
   = help: items from traits can only be used if the trait is implemented and in scope
   = note: the following trait defines an item `now`, perhaps you need to implement it:
           candidate #1: `main::Now`
``` 

Anche in questo caso l'errore ci da un buon suggerimento, ed in effetti il compilatore ha ragione, non abbiamo ancora scritto da nessuna parte che il nostro generico `T` deve implementare il trait `Now`. Possiamo farlo semplicemente aggiungendo un vincolo alla definizione della nostra struttura `Event`: 

```rust
use std::marker::PhantomData;
struct Event<T> {
    message: String,
    stamp: DateTime<Utc>,
    mark: PhantomData<T>,
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

let event: Event<SystemNow> = Event::new("Messaggio".to_string());
``` 

Adesso il codice compila di nuovo, perché il compilatore ha modo di verificare che il generico `T` non è così generico come prima, ma deve implementare i metodi definiti nel trait `Now`, e tra questi (uno solo, in questo caso) c'è proprio un metodo `now()` con la firma giusta. 

Adesso abbiamo tutti gli ingredienti per usare un trait diverso nei test rispetto a quello usato nella libreria, ma come lo vedremo nella [seconda parte](https://naufraghi.slug.it/2018/06/test-riproducibili-con-il-tempo-parte-2.html) di questo post. 

I sorgenti di questo progetto sono disponibili qua: [gitlab.com/naufraghi/phantomtypes](https://gitlab.com/naufraghi/phantomtypes) 

_Grazie a [@tglman](https://twitter.com/tglman) e [@lucabruno](https://twitter.com/lucabruno) per la revisione della bozza._ 

[^1]: Potete verificare che il marcatore è a costo zero in [questo       snippet](https://play.rust-lang.org/?gist=ed6df3a019185dec49b233fce9e49604&version=stable&mode=debug). 

[^2]: I `Trait` sono una cosa simile alle interfacce del Go o alle interfacce dei linguaggi che usano       l'ereditarietà, per maggiori dettagli [Traits: Defining Shared       Behavior](https://doc.rust-lang.org/book/second-edition/ch10-02-traits.html). 

[^3]: `Option` ha un numero enorme di [metodi di       utilità](https://doc.rust-lang.org/std/option/enum.Option.html), che conviene tenere sott'occhio,       perché ce ne è uno adatto per quasi ogni caso d'uso che potrebbe capitarvi. 

