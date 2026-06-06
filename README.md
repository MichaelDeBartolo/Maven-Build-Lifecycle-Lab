# Maven Build Lifecycle Lab — Ubuntu

Laboratorio pratico sul build lifecycle di Maven, eseguito su Ubuntu Linux.  
Obiettivo: comprendere le fasi di build, leggere i log, diagnosticare errori tipici in un contesto di integration support.

---

## Ambiente

| Componente | Versione |
|---|---|
| OS | Ubuntu 24.04 |
| Java | OpenJDK 17 (LTS) |
| Maven | 3.x |
| Progetto | `com.testlab:integration-demo` |

---

## Fase 1 — Installazione

Installazione di Java e Maven tramite apt, con verifica delle versioni.

```bash
# Aggiorna i pacchetti
sudo apt update && sudo apt install -y openjdk-17-jdk

# Verifica Java
java -version

# Installa Maven
sudo apt install -y maven

# Verifica Maven
mvn -version
```

**Risultato atteso:**
```
openjdk version "17.0.x"
Apache Maven 3.x.x
Java version: 17.0.x
```

---

## Fase 2 — Creazione del progetto

Maven genera automaticamente la struttura del progetto tramite un archetype.  
Non è necessario scrivere codice Java manualmente.

```bash
# Crea la directory di lavoro
mkdir ~/maven-lab && cd ~/maven-lab

# Genera il progetto con archetype quickstart
mvn archetype:generate \
  -DgroupId=com.testlab \
  -DartifactId=integration-demo \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

# Entra nel progetto
cd integration-demo

# Esplora la struttura generata
find . -type f
```

**Struttura prodotta:**
```
./pom.xml
./src/main/java/com/testlab/App.java
./src/test/java/com/testlab/AppTest.java
```

**Componenti chiave del `pom.xml`:**

| Campo | Valore | Significato |
|---|---|---|
| `groupId` | `com.testlab` | Identificatore dell'organizzazione |
| `artifactId` | `integration-demo` | Nome del progetto |
| `version` | `1.0-SNAPSHOT` | Versione del jar prodotto |

---

## Fase 3 — Build lifecycle

Maven segue un ordine fisso di fasi. Eseguire una fase successiva include automaticamente tutte le precedenti.

### compile
Trasforma i file `.java` in bytecode `.class`.

```bash
mvn compile

# Verifica output prodotto
ls target/classes/com/testlab/
# → App.class
```

### test
Esegue i test automatici JUnit.

```bash
mvn test
# → Tests run: 1, Failures: 0, Errors: 0
```

### package
Crea il file `.jar` distribuibile nella cartella `target/`.

```bash
mvn package

# Verifica il jar prodotto
ls -lh target/*.jar
# → integration-demo-1.0-SNAPSHOT.jar
```

### install
Copia il jar nel repository locale Maven (`~/.m2`), rendendolo disponibile ad altri progetti locali.

```bash
mvn install

# Verifica presenza nel repository locale
find ~/.m2/repository/com/testlab -type f
```

### clean
Elimina la cartella `target/` per ripartire da zero.

```bash
mvn clean
```

### Comando tipico in produzione

```bash
# Clean + package saltando i test — usato per deploy rapidi
mvn clean package -DskipTests
```

---

## Fase 4 — Simulazione errori

Provocare errori reali per allenarsi a leggere i log di build.

### Errore 1 — Errore di compilazione (sintassi Java)

```bash
# Rompe la sintassi in App.java
sed -i 's/System.out.println/System.out.ERRORE/' \
  src/main/java/com/testlab/App.java

# Lancia il compile
mvn compile
```

**Output:**
```
[ERROR] COMPILATION ERROR
[ERROR] App.java:[10,20] cannot find symbol
[ERROR] symbol: variable ERRORE
[ERROR] BUILD FAILURE
```

**Diagnosi:** `BUILD FAILURE` in fase `compile` = errore di sintassi nel codice sorgente. Da escalare al developer con il numero di riga indicato nel log.

```bash
# Ripristino
sed -i 's/System.out.ERRORE/System.out.println/' \
  src/main/java/com/testlab/App.java
```

---

### Errore 2 — Dipendenza non risolvibile

```bash
# Aggiungi una dipendenza inesistente nel pom.xml
# Apri pom.xml con un editor e inserisci dentro <dependencies>:
nano pom.xml
```

Aggiungi manualmente:
```xml
<dependency>
  <groupId>com.fake</groupId>
  <artifactId>fake-lib</artifactId>
  <version>99.0</version>
</dependency>
```

```bash
# Lancia il package
mvn package
```

**Output:**
```
[ERROR] Could not resolve dependencies for project com.testlab:integration-demo
[ERROR] Could not find artifact com.fake:fake-lib:jar:99.0
[ERROR] BUILD FAILURE
```

**Diagnosi:** `Could not resolve dependencies` = dipendenza mancante o versione errata nel `pom.xml`. Non è un errore di codice. Verificare il `pom.xml` e il repository Nexus/Artifactory aziendale.

```bash
# Ripristino: rimuovi le righe della dipendenza fake con nano
nano pom.xml
# Ctrl+O per salvare, Ctrl+X per uscire
```

---

## Fase 5 — Lettura avanzata dei log

Comandi per filtrare e analizzare log verbosi, utili in contesti di monitoring e incident response.

```bash
# Salva il log completo su file e mostralo a schermo
mvn package 2>&1 | tee build.log

# Filtra solo errori e warning
grep -E "ERROR|FAILURE|WARNING" build.log

# Mostra le fasi del lifecycle eseguite
grep "\-\-\-" build.log

# Build silenzioso — mostra solo errori
mvn package -q 2>&1 | grep -E "ERROR|BUILD"

# Albero completo delle dipendenze
mvn dependency:tree
```

**Output di `dependency:tree`:**
```
com.testlab:integration-demo:jar:1.0-SNAPSHOT
\- junit:junit:jar:4.11:test
   \- org.hamcrest:hamcrest-core:jar:1.3:test
```

Utile per diagnosticare conflitti tra versioni di librerie (dipendenze transitive).

---

## Riepilogo — fasi e artefatti prodotti

| Fase | Comando | Artefatto prodotto | Dove |
|---|---|---|---|
| compile | `mvn compile` | `.class` | `target/classes/` |
| test | `mvn test` | report test | `target/surefire-reports/` |
| package | `mvn package` | `.jar` | `target/` |
| install | `mvn install` | `.jar` | `~/.m2/repository/` |
| clean | `mvn clean` | — | elimina `target/` |

---

## Riepilogo — errori tipici e diagnosi

| Messaggio nel log | Fase | Causa | Azione |
|---|---|---|---|
| `cannot find symbol` | compile | Sintassi Java errata | Escalare al developer con numero di riga |
| `Could not resolve dependencies` | package | Dipendenza mancante o versione errata | Verificare `pom.xml` e repo Nexus |
| `Tests run: X, Failures: Y` | test | Test falliti | Escalare al developer con il report |
| `401 Unauthorized` | deploy | Credenziali errate su Nexus/Artifactory | Verificare configurazione `settings.xml` |
| `OutOfMemoryError` | qualsiasi | Heap JVM esaurita | Verificare memoria disponibile, non riavviare senza diagnosi |

---

## Note finali

Laboratorio realizzato su Ubuntu 24.04 come esercizio pratico di comprensione del build lifecycle Maven in contesto di integration support.

Il focus non è sulla scrittura di codice Java, ma sulla capacità di leggere output di build, identificare la fase in cui un processo fallisce, e distinguere tra errori di compilazione, dipendenze non risolte e problemi di deploy —
