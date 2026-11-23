# Maven Build Lifecycle Lab su Ubuntu — Guida Passo Passo

Guida pratica sul **build lifecycle di Apache Maven** su Linux (Ubuntu `24.04`) con **OpenJDK 17**.  
L'obiettivo di questo laboratorio è mostrare come installare Java e Maven, generare un progetto di esempio, eseguire le fasi del ciclo di build, interpretare i log in caso di errore e distinguere rapidamente le cause in un contesto di **integration support** o **incident triage**.

> **Nota sul perimetro:** In questo laboratorio il focus non è sullo sviluppo applicativo Java, ma sulla comprensione del ciclo di vita della build, sulla lettura dei log e sulla diagnosi degli errori più comuni (errori di sintassi, dipendenze non risolte, fallimento test o problemi di autenticazione/risorse).

---

## Cosa fa in breve Maven (e perché si parte da qui)

Immagina di dover trasformare un progetto sorgente in un artefatto eseguibile o distribuibile (come un `.jar`) e verificare che tutto sia coerente prima del rilascio:
* **Maven** automatizza compilazione, esecuzione dei test, impacchettamento e gestione delle dipendenze seguendo un ordine preciso di fasi (**lifecycle**).
* Quando si lancia una fase avanzata del lifecycle (ad esempio `package`), Maven esegue automaticamente anche le fasi precedenti che la precedono nell'ordine standard.
* Capire **in quale fase** si ferma una build e **quale messaggio** riporta il log permette di indirizzare subito il problema al team o alla risorsa giusta.

---

## Architettura e Ambiente di Test

| Componente | Ruolo | Dettagli nel laboratorio |
| :--- | :--- | :--- |
| **Sistema Operativo** | Host Linux | Ubuntu `24.04` con accesso terminale e permessi `sudo` |
| **Java** | Runtime e compilatore | OpenJDK `17` (LTS) |
| **Build Tool** | Gestione build e dipendenze | Apache Maven `3.x` |
| **Progetto di test** | Archetype `quickstart` | `com.testlab:integration-demo` (`1.0-SNAPSHOT`) |

---

## Passo 1 — Installazione di Java e Maven

Aggiorna i pacchetti del sistema, installa **OpenJDK 17** e **Apache Maven**, quindi verifica le versioni installate:

```bash
sudo apt update && sudo apt install -y openjdk-17-jdk
java -version

sudo apt install -y maven
mvn -version
```

### Verifica del passaggio
I comandi `java -version` e `mvn -version` devono restituire le versioni attese (`OpenJDK 17` e `Apache Maven 3.x`) senza errori.

---

## Passo 2 — Creazione del Progetto con Archetype

Crea una cartella di lavoro e genera la struttura base del progetto usando il template `maven-archetype-quickstart`:

```bash
mkdir -p ~/maven-lab && cd ~/maven-lab

mvn archetype:generate \
  -DgroupId=com.testlab \
  -DartifactId=integration-demo \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

cd integration-demo
find . -type f
```

### Struttura generata e file `pom.xml`
```text
./pom.xml
./src/main/java/com/testlab/App.java
./src/test/java/com/testlab/AppTest.java
```

Nel file `pom.xml` sono definiti i parametri identificativi del progetto:

| Campo | Valore nel lab | Significato |
| :--- | :--- | :--- |
| `groupId` | `com.testlab` | Identificatore dell'organizzazione o del gruppo |
| `artifactId` | `integration-demo` | Nome dell'artefatto / progetto |
| `version` | `1.0-SNAPSHOT` | Versione del pacchetto generato |

---
## Passo 3 — Eseguire le Fasi del Build Lifecycle

All'interno della cartella `integration-demo`, puoi eseguire le singole fasi del ciclo di vita di Maven per osservare quali artefatti vengono prodotti:

### 3.1 `compile` — Compilazione dei sorgenti
Compila i file `.java` in bytecode `.class`:

```bash
mvn compile
ls target/classes/com/testlab/
```

### 3.2 `test` — Esecuzione dei test automatici
Esegue la suite di test (es. JUnit):

```bash
mvn test
```

### 3.3 `package` — Creazione del pacchetto `.jar`
Impacchetta il codice compilato e le risorse nel file `.jar` all'interno di `target/`:

```bash
mvn package
ls -lh target/*.jar
```

### 3.4 `install` — Installazione nel repository locale (`~/.m2`)
Copia il `.jar` nel repository locale di Maven (`~/.m2`), rendendolo riutilizzabile da altri progetti locali sulla macchina:

```bash
mvn install
find ~/.m2/repository/com/testlab -type f
```

### 3.5 `clean` e comando combinato tipico
Rimuove la cartella `target/` per ripulire gli artefatti generati, oppure combina pulizia e impacchettamento:

```bash
mvn clean
```

In contesti operativi o di deploy rapido si incontra spesso una combinazione come:

```bash
mvn clean package -DskipTests
```
*(Nota: saltare i test con `-DskipTests` va usato con cautela e solo quando previsto dalla procedura operativa, perché riduce la verifica automatica prima del rilascio.)*

---

## Passo 4 — Simulazione e Diagnosi degli Errori di Build

Allenarsi a leggere il log quando una build fallisce aiuta a capire subito se il problema riguarda il codice sorgente, le dipendenze o l'ambiente.

### 4.1 Errore di compilazione (sintassi Java)
Simula un errore di sintassi in `App.java` e lancia `mvn compile`:

```bash
sed -i 's/System.out.println/System.out.ERRORE/' src/main/java/com/testlab/App.java
mvn compile
```

Output tipico:
```text
[ERROR] COMPILATION ERROR
[ERROR] App.java:[10,20] cannot find symbol
[ERROR] symbol: variable ERRORE
[ERROR] BUILD FAILURE
```

**Come leggerlo e ripristinarlo:** un errore `COMPILATION ERROR` con `cannot find symbol` in fase di compilazione indica un errore nel codice sorgente (con file e posizione indicati nel log). Ripristina il file:

```bash
sed -i 's/System.out.ERRORE/System.out.println/' src/main/java/com/testlab/App.java
```

### 4.2 Dipendenza non risolvibile (`pom.xml`)
Aggiungi temporaneamente una dipendenza inesistente dentro `<dependencies>` in `pom.xml` (con `nano pom.xml`):

```xml
<dependency>
  <groupId>com.fake</groupId>
  <artifactId>fake-lib</artifactId>
  <version>99.0</version>
</dependency>
```

Esegui `mvn package`:

```bash
mvn package
```

Output tipico:
```text
[ERROR] Could not resolve dependencies for project com.testlab:integration-demo
[ERROR] Could not find artifact com.fake:fake-lib:jar:99.0
[ERROR] BUILD FAILURE
```

**Come leggerlo e ripristinarlo:** `Could not resolve dependencies` indica che Maven non trova l'artefatto richiesto (coordinate errate, versione non disponibile o problemi di accesso al repository remoto/privato), non un errore di sintassi Java. Rimuovi poi le righe aggiunte in `pom.xml`.

---
## Passo 5 — Lettura Avanzata dei Log e Dipendenze

Quando l'output di build è esteso, puoi salvare il log su file, filtrare gli avvisi/errori e ispezionare l'albero delle dipendenze:

```bash
mvn package 2>&1 | tee build.log
grep -E "ERROR|FAILURE|WARNING" build.log
grep "\-\-\-" build.log
mvn package -q 2>&1 | grep -E "ERROR|BUILD"
mvn dependency:tree
```

Esempio di albero dipendenze (`mvn dependency:tree`):
```text
com.testlab:integration-demo:jar:1.0-SNAPSHOT
\- junit:junit:jar:4.11:test
   \- org.hamcrest:hamcrest-core:jar:1.3:test
```
Utile per verificare dipendenze transitive e individuare eventuali conflitti di versione.

---

## Tabelle di Riepilogo

### Fasi e artefatti principali

| Fase | Comando | Artefatto / Risultato | Percorso tipico |
| :--- | :--- | :--- | :--- |
| **compile** | `mvn compile` | File `.class` compilati | `target/classes/` |
| **test** | `mvn test` | Report dei test eseguiti | `target/surefire-reports/` |
| **package** | `mvn package` | Archvio `.jar` | `target/` |
| **install** | `mvn install` | Copia `.jar` nel repo locale | `~/.m2/repository/` |
| **clean** | `mvn clean` | Pulizia artefatti di build | Rimuove `target/` |

### Errori frequenti e prima diagnosi

| Segnale nel log | Fase / Ambito | Causa probabile | Prima azione di verifica |
| :--- | :--- | :--- | :--- |
| `cannot find symbol` | `compile` | Errore di sintassi o simbolo non definito nel codice | Controllare file e riga indicati e segnalare allo sviluppo |
| `Could not resolve dependencies` | `package` / risoluzione | Dipendenza o versione non trovata / repo non raggiungibile | Verificare coordinate in `pom.xml` e configurazione repository |
| `Tests run: X, Failures: Y` | `test` | Uno o più test automatici sono falliti | Ispezionare i report in `target/surefire-reports/` e il dettaglio del test |
| `401 Unauthorized` | `deploy` / repo | Credenziali non valide o scadute | Verificare credenziali/token e configurazione in `settings.xml` |
| `OutOfMemoryError` | Qualsiasi | Memoria heap JVM insufficiente | Verificare risorse di sistema e impostazioni memoria della JVM |

---

## Competenze Dimostrate in Questo Laboratorio

* Installazione e verifica di **Java (OpenJDK 17)** e **Apache Maven** su Linux Ubuntu.
* Generazione e ispezione di un progetto Maven (`pom.xml`, sorgenti, test e cartella `target/`).
* Esecuzione delle fasi del lifecycle (`compile`, `test`, `package`, `install`, `clean`) e analisi dell'albero dipendenze con `mvn dependency:tree`.
* Diagnosi rapida di errori di compilazione e dipendenze tramite analisi dei log e filtri CLI (`grep`, `tee`).
