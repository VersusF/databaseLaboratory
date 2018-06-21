# APPUNTI BASI DI DATI LABORATORIO

Piccolo file di appunti per l'esame di basi di dati laboratorio. Gli argomenti sono:

- [Transazioni](#transazioni)
- [Python: json e csv](#python-json-e-csv)
- [Python: psycopg2](#python-psycopg2)
- [Python: Flask](#python-flask)
- [HTML: Jinja2](#html-jinja2)
- [HTML: Form](#html-form)
- [Java: JDBC](#java-jdbc)

## Transazioni

Le transazioni non devono interferire l'una con l'altra, quindi è opportuno mantenere il giusto livello di isolamento. I 4 livelli sono:
- `READ UNCOMMITTED`, che però non è implementato in postgresql
- `READ COMMITTED`, livello di default per le transazioni
- `REPEATABLE READ`, I dati letti non cambieranno
- `SERIALIZABLE`, completo isolamento

In sql l'istruzione per selezionare il livello di isolamento è:

```SQL
SET TRANSACTION ISOLATION LEVEL [...]
```

Il livello **__`READ COMMITTED`__** vede tutti i dati aggiornati all'ultimo `COMMIT`, per questo motivo soffre di due anomalie. La prima è la **non-repeatable reads**, Cioè quando un'operazione che dovrebbe essere fatta viene annullata poiché le tuple interessate sono state modificate da un'altra transazione.
La seconda anomalia è il **phantom reads**, che avviene quando vengono effettuate due letture identiche fra loro ma che danno due risultati diversi poiché inframezzati dal commit di un'altra transazione.

Il livello **__`REPEATABLE READ`__** invece garantisce che due letture identiche diano risultati identici. Se su un dato è stato aggiornato e viene eseguito il commit l'operazione che lo voleva modificare (`UPDATE` o `DELETE`) viene bloccata con un messaggio di errore. Questo blocca i *non-repatable reads*, ma mantiene i **phantom-reads** nei casi in cui c'è un `INSERT` con `SELECT`. (Se invece c'è un `UPDATE` con `SELECT` questo livello di isolamento è sufficiente)

Il livello **__`SERIALIZABLE`__** garantisce il completo e massimo isolamento tra le transazioni. Corregge anche l'anomalia di *phantom-reads*, ma alcune transazioni potrebbero venir bloccate e quindi devono essere ripetute. 

Livello-Concorrenza | non-repeatable reads | phantom reads
--- | --- | ---
Read committed | X | X
Repeatable read | | X
Serializable | |


## Python: json e csv

Per leggere e scrivere dati in python nei formati json e csv si utilizzano le librerie `json` e `csv`.

#### JSON

Le funzioni principali sono 2, `dump` e `load`. Il primo vuole come parametro un file (aperto con la direttiva `open`) e l'oggetto da scrivere. Il secondo vuole solo il file (nello stesso formato) e restituisce l'oggetto letto.

```python 3
import json

with open('myFile.json', 'w') as f:
    json.dump(myDict, f)

with open('myFile.json', 'r') as f:
    myDict = json.load(f)
```

Se l'oggetto è una classe creata dall'utente alla funzione `dump` è necessario fornire una classe encoder, con un metodo default che costruisca un oggetto seriazilable (per esempio un `dict`)

```python 3
import json

class Person(object):
    def __init__(self, name):
        self.name = name

class PersonEncoder(json.JSONEncoder):
    def default(self, o):
        return {'name': o.name}

my_person = Person('Filippo')
with oper('myFile.json', 'w') as f:
    json.dump(f, my_person, cls=PersonEncoder)
```

#### CSV

Per salvare dati in formato csv le si utilizzano 2 classi principalmente: `csv.DictWriter` e `csv.DictReader`. Il costruttore della prima vuole un file come parametri un file e la lista delle chiavi dei dizionari. Il costruttore della seconda vuole solo il file da cui leggere.

I metodi di scrittura sono `writeHeader()` per scrivere la prima riga di intestazione, `writerow(dict)` per scrivere il dizionario come una riga del file e `writerows(list)` per scrivere più righe sul file. Ogni dizionario deve avere le stesse chiavi dichiarate in precedenza.

Per la lettura basta iterare sulla classe `DictReader`, in cui ogni riga diventa un `orderedDict` con le chiavi dichiarate nell'header. 

**N.B.:** I dati sono letti come stringhe, quindi eventuali dati numerici devono essere convertiti.

```python 3
import csv

with open('myFile.csv', 'w') as f:
    fields = ['nome', 'eta']
    writer = csv.DictWriter(f, fields)
    writer.writeheader()
    writer.writerows(myList)
    # for row in myList:
    #     writer.writerow(row)

with open('myFile.csv', 'r') as f:
    reader = csv.DictReader(f)
    myNewList = []
    for row in reader:
        myNewList.append(dict(row))
        # casting to dict
```

## Python: psycopg2

psycopg2 è una libreria per python per accedere al dbms di postgresql tramite cursore. L'accesso al database si ha tramite l'oggetto `Connection`, a cui bisogna fornire host, database, user, password. Dellla connessione è possibile modificare i campi `autocommit` e `isolation_level`. Per aprire una tranzazione basta usare l'oggetto connessione in un blocco `with`, al termine del quale viene eseguito un commit. Quando la connessione non è più necessaria la si chiude col metodo `close()`.

Per eseguire istruzioni SQL si usa il metodo `cursor()`, che restituisce un oggetto cursore col quale inviare comandi e ricevere risposte. Per l'invio di un comando si usa `execute(string, tuple)`, in cui la stringa è l'istruzione SQL, in cui ci possono essere dei segnaposti (marcati con `%`) che vengono sostituiti dai valori nella tupla facendo l'escaping automatico. **Mai creare l'istruzione concatenando stringhe**. Dopo aver lanciato un comando di query il cursore diventa un iterabile sulle tuple del risultato. Se invece l'istruzione è di un altro tipo il messaggio di risposta è presente nel campo `statusmessage` del cursore. E' consigliabile usare i cursori dentro ai blocchi `with`. Se alla creazione del cursore si può impostare `cursor_factory=psycopg2.extras.DictCursor` come parametro, così da avere la risposta alla query come un iteratore di dizionari

```python 3
import psycopg2

conn = psycopg2.connect(host='...', database='X', user='admin', password='admin')

with conn:
    with conn.cursor() as cur:
        cur.execute('...', (tupla,))
        for line in cur:
            print(str(line)

conn.close()
```

## Python: Flask

La libreria flask di python permette di scrivere un'applicazione web lato server. La classe principale è l'applicazione Flask, che viene istanziata globale all'inizio del main file. Poi si dichiarano, con dei decoratori, i metodi che gestiscono l'interazione con le pagine, `@app.route()`, e infine si avvia l'applicazione.

Per ogni route si possono definire i metodi accettati per la pagina, per esempio *POST* e *GET*. Per accedere ai parametri passati dalla richiesta si utilizza rispettivamente `request.form` e `request.args`, usati come dizionari.

I metodi devono restituire la pagina HTML da visualizzare, che può essere caricata tramite `render_template` della libreria jinja2, a cui è possibile passare dei parametri calcolati.

```python 3
from flask import Flask, request, render_template

app = Flask(__name__)

@app.route('/')
def main():
    return render_template('login.html')

@app.route('/login', methods=['POST'])
def login():
    text = request.form['text']
    return render_template('page2.html', text=text)

app.run()
```

## HTML: Jinja2

Jinja2 è un motore di rendering per le pagine HTML. Una pagina può avere dei parametri passati dal metodo render_template coi quali interagisce.

Per inserire comandi questi vanno racchiusi tra parentesi graffe e percentuali, es: `{% comando %}`, mentre per visualizzare i valori si usano le doppie parentesi, es: `{{ valore }}`.

I comandi più usati sono:

```HTML
{% for item in list%}
<p>{{item}}</p>
{% endfor %}

{% if win == true %}
<p>You Win</p>
{% else %}
<p>You Lose</p>
{% endfor %}
```
Si possono inserire in ogni parte del codice HTML

## HTML: Form

Le form in HTML sono il costrutto ideale per inviare dati alle pagine Flask con metodi *GET* e *POST*. Oltre ai metodi occorre specificare la pagina di destinazione della richiesta. Sono composte da una serie di input e un submit, a ognuno dei quali è associato un nome (la chiave del dizionario in python) e un tipo, che specifica il dominio dei valori che l'utente può inserire

Esempio:
```HTML
<form action="/targetPage" method="POST">
    <input type="text" name="testo1">
    <input type="number" name="numero">
    <input tyoe="date" name="data">
    <input type="text" name="targa" pattern="[A-Z]{2}\d{3}[A-Z]{2}">
    <input type="submit" value="Conferma">
</form>
```

## Java: JDBC

Per utilizzare postresql in java occorre importare una libreria apposita, la JDBC di postgresql. Per importarla si usa il comando `Class.forName("...")`.

Il primo passo per l'utilizzo è la creazione di una connessione, tramite `DriverManager.getConnection()`, normalmente in autocommit. Questa classe `Connection` fornisce il metodo `prepareStatement` che prepara un'istruzione, passata come parametro, che può avere dei segnaposto identificati col caratter `?`. Per riempire questi posti si usano i metodi `set<Type>(index, value)`. Una volta pronta si può eseguire il comando con `executeUpdate` o `executeQuery`. Il primo ritorna un intero contenente in numero di righe, il secondo ritorna un `ResultSet`, un iterabile sulle tuple risultato, dal quale si può estrarre un valore con `get<Type>('index')` o `get<Type>('key')`

**N.B.:** Tutte le istruzioni di JDBC possono lanciare `SQLException`, serve quindi un blocco try/catch.

```java
public static class Main{
    public static void main(String[] args){
        Class.forName("org.postgresql.Driver");
        idBiblioteca = Integer.parseInt(args[0]);
        idUtente = args[1];
        try (Connection conn = DriverManager.getConnection("jdbc:postgresql://localhost:5432/nomeDatabase", "user", "password")) {
            String queryString = "SELECT * FROM prestito " + \
                                 "WHERE idBiblioteca = ? AND idUtente = ?";
            PreparedStatement pst = conn.prepareStatement(queryString);
            pst.setInt(1, idBiblioteca);
            pst.setString(1, idUtente);
            ResultSet rs = pst.executeQuery();
            while (rs.next())
                system.out.println(rs.getInt("id") + " " + rs.getInt(2));
        } catch (SQLEcxeption e){
            e.printStackTrace();
        }
    }
}
```
