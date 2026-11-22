# Laboratorio IDS con Suricata su Linux — Guida Passo Passo

Guida pratica di **Network Security Monitoring (NSM)** e **Intrusion Detection System (IDS)** con **Suricata** su Linux (Ubuntu).  
L'obiettivo di questo laboratorio è mostrare come installare un sensore IDS, avviarlo in modalità di rilevamento (senza blocco del traffico), comprendere la cartella dei log e analizzare gli eventi di rete con strumenti da terminale come `jq`.

> **Nota sul perimetro (IDS vs. IPS):** In questo laboratorio Suricata viene utilizzato in modalità **IDS** (*Intrusion Detection System*): osserva il traffico sull'interfaccia di rete, lo analizza rispetto alle regole e registra eventi o alert nei file di log **senza bloccare** le connessioni. La modalità **IPS** (*Intrusion Prevention System*, con azioni come `drop`/`reject` e traffico in-line) è prevista in un laboratorio successivo dedicato.

---

## Cosa fa in breve un IDS (e perché si parte da qui)

Immagina un sensore IDS come una **telecamera di sorveglianza con allarme sonoro o registro eventi** lungo una strada:
* Guarda i veicoli che passano (i **pacchetti e i flussi di rete**).
* Controlla se qualcuno ha un comportamento o una targa nota in un elenco (le **regole** di rilevamento).
* Scrive cosa ha visto nei **log** (e genera un **alert** se scatta una regola), **ma non abbassa la sbarra per bloccare il traffico**.

Partire in modalità **IDS** è fondamentale in un laboratorio perché permette di osservare il traffico reale, capire cosa è normale sulla rete, leggere i log strutturati e verificare eventuali falsi positivi senza interrompere la connessione.

---

## Prerequisiti e Ambiente di Test

* **Sistema Operativo:** Linux (es. Ubuntu) con accesso a un terminale e permessi `sudo`.
* **Sensore:** Suricata `8.0.x` in modalità `SYSTEM` (IDS).
* **Interfaccia di rete di esempio:** `wlp0s20f3` (sostituiscila con la tua interfaccia attiva, visibile con `ip a` o `dumpcap -D`).
* **Strumenti di analisi:** `jq` (per filtrare e leggere i log JSON) e `curl` (per generare traffico di test).

---

## Passo 1 — Installazione di Suricata

Aggiorna l'elenco dei pacchetti disponibili e installa Suricata:

```bash
sudo apt update && sudo apt install -y suricata
```

### Verifica del passaggio
Controlla che il programma sia installato e leggi la versione:

```bash
suricata -V
```
Se compare la versione di Suricata senza errori, l'installazione di base è riuscita.

---

## Passo 2 — Aggiornamento delle Regole e Validazione della Configurazione

Un IDS ha bisogno di **regole** (firme e condizioni di controllo) per sapere cosa segnalare come potenzialmente sospetto o malevolo.

### 2.1 Aggiornamento delle regole
Scarica e prepara il set di regole tramite `suricata-update`:

```bash
sudo suricata-update
```
Questo comando genera/aggiorna il file delle regole utilizzato dal motore (tipicamente in `/var/lib/suricata/rules/suricata.rules`).

### 2.2 Controllo sintattico della configurazione
Prima di avviare il sensore, verifica che il file di configurazione principale (`/etc/suricata/suricata.yaml`) non contenga errori:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

### Verifica del passaggio
L'output deve terminare confermando che la configurazione è valida senza messaggi di errore bloccanti (`ERR`).

---

## Passo 3 — Avvio di Suricata in Modalità IDS

Avvia Suricata in ascolto sull'interfaccia di rete scelta (nell'esempio `wlp0s20f3`) usando il file di configurazione:

```bash
sudo suricata -i wlp0s20f3 -c /etc/suricata/suricata.yaml
```

All'avvio vedrai righe simili a queste:

```text
i: suricata: This is Suricata version 8.0.3 RELEASE running in SYSTEM mode
i: mpm-hs: Rule group caching - loaded: 121 newly cached: 0 total cacheable: 121
i: threads: Threads created -> W: 16 FM: 1 FR: 1   Engine started.
```

### Come interpretare questo stato
* **Non è bloccato o crashato:** `Engine started.` significa che il motore è attivo e sta catturando e analizzando i pacchetti in background.
* **A differenza di Wireshark:** Suricata non fa scorrere i pacchetti sullo schermo; scrive i risultati nei file di log.
* **Come fermarlo:** quando hai finito la sessione di test, torna su questo terminale e premi **`Ctrl + C`** per arrestarlo in modo pulito.

---

## Passo 4 — Generare Traffico di Prova

Con Suricata in esecuzione nel primo terminale, apri un **secondo terminale** (o usa il browser) per generare del traffico di rete reale da osservare nei log:

```bash
curl -I [http://example.com](http://example.com)
```

### Verifica del passaggio
`curl` deve restituire una risposta HTTP (ad esempio `HTTP/1.1 200 OK` o un redirect), confermando che la rete funziona e che il sensore sta ricevendo traffico.

---

## Passo 5 — La Cartella dei Log (`/var/log/suricata/`)

Suricata salva tutto ciò che osserva e registra nella directory `/var/log/suricata/`. Puoi elencarne il contenuto dal secondo terminale:

```bash
sudo ls -lh /var/log/suricata/
```

Ecco a cosa serve ciascun file principale:

| File | Formato | A cosa serve |
| :--- | :--- | :--- |
| **`eve.json`** | JSON (una riga = un evento) | **Il log più importante per l'analisi.** Contiene flussi di rete (`flow`), richieste DNS (`dns`), traffico web (`http`, `tls`, `quic`), statistiche (`stats`) ed eventuali alert (`alert`) con molti dettagli. |
| **`fast.log`** | Testo semplice | **Elenco rapido degli alert.** Se scatta una regola, qui compare una riga sintetica con data/ora, nome della regola (`msg`), IP/porte coinvolti e priorità. Se è a `0` byte, non sono ancora scattati alert. |
| **`stats.log`** | Testo | **Statistiche del sensore.** Mostra quanti pacchetti sono stati letti, quanti protocolli sono stati riconosciuti e se ci sono stati pacchetti persi (`drops`) o problemi di memoria. |
| **`suricata.log`** | Testo | **Log operativo di Suricata.** Registra avvio, caricamento delle regole, avvisi di configurazione o arresto del servizio. |

---

## Passo 6 — Come Leggere i Log con `jq`

Il file `eve.json` è in formato **JSON Lines** (ogni riga è un oggetto JSON). Leggerlo a occhio nudo è difficile: con **`jq`** puoi filtrare, contare e mostrare solo i campi che ti interessano.

### 6.1 Installare `jq` (se non è già presente)
```bash
sudo apt install -y jq
```

### 6.2 Contare i tipi di eventi catturati (`event_type`)
Questo comando ti dà subito una mappa di cosa ha visto Suricata:

```bash
sudo jq -r '.event_type' /var/log/suricata/eve.json | sort | uniq -c
```

Esempio di risultato:
```text
     14 dns
     47 flow
      1 http
     36 mdns
     59 quic
     23 stats
      4 tls
```
**Come leggerlo:**
* `flow`: connessioni/flussi di rete osservati.
* `dns`: richieste e risposte legate alla risoluzione dei nomi a dominio.
* `http` / `tls` / `quic`: traffico applicativo web (in chiaro o cifrato).
* `stats`: contatori periodici sullo stato del motore.
* Se non compare `alert`, significa che nessuna regola ha segnalato una corrispondenza durante quel periodo.

### 6.3 Leggere i flussi di rete (`flow`)
Per vedere chi ha comunicato con chi, su quale protocollo/porta e quanti byte sono stati scambiati:

```bash
sudo jq -c 'select(.event_type=="flow") | {
  time: .timestamp,
  src: .src_ip,
  sport: .src_port,
  dest: .dest_ip,
  dport: .dest_port,
  proto: .proto,
  app: .app_proto,
  bytes_out: .flow.bytes_toserver,
  bytes_in: .flow.bytes_toclient
}' /var/log/suricata/eve.json | head -n 15
```

**Come leggerlo:**
* `src` / `sport` e `dest` / `dport`: indirizzo IP e porta di origine e destinazione.
* `proto` e `app`: protocollo di trasporto (`TCP`/`UDP`) e protocollo applicativo riconosciuto (es. `dns`, `tls`, `quic`, `http`).
* `bytes_out` / `bytes_in`: volume di dati inviati verso il server e ricevuti dal client (utile per individuare trasferimenti insoliti o anomalie).

### 6.4 Leggere le query DNS (`dns`)
Per vedere quali nomi a dominio sono stati interrogati (in Suricata 8 le domande sono in `.dns.queries[]`):

```bash
sudo jq -c 'select(.event_type=="dns") | {
  time: .timestamp,
  src: .src_ip,
  dest: .dest_ip,
  queries: [.dns.queries[]? | {rrname, rrtype}]
}' /var/log/suricata/eve.json
```

**Come leggerlo:**
* `rrname`: il nome del dominio richiesto.
* `rrtype`: il tipo di record DNS (es. `A` per IPv4, `AAAA` per IPv6, `HTTPS`).

### 6.5 Leggere `HTTP` e `TLS` (Server Name Indication)
Per vedere l'host HTTP o il nome del server indicato nell'handshake TLS (`sni`):

```bash
sudo jq -c 'select(.event_type=="http" or .event_type=="tls") | {
  time: .timestamp,
  type: .event_type,
  src: .src_ip,
  dest: .dest_ip,
  http_host: .http.hostname,
  http_uri: .http.url,
  tls_sni: .tls.sni
}' /var/log/suricata/eve.json
```

### 6.6 Controllare eventuali Alert (`alert`)
Quando una regola scatta, puoi filtrare solo gli eventi `alert` in `eve.json`:

```bash
sudo jq -c 'select(.event_type=="alert") | {
  time: .timestamp,
  src: .src_ip,
  sport: .src_port,
  dest: .dest_ip,
  dport: .dest_port,
  signature: .alert.signature,
  category: .alert.category,
  severity: .alert.severity
}' /var/log/suricata/eve.json
```

Oppure seguire in tempo reale `fast.log`:

```bash
sudo tail -f /var/log/suricata/fast.log
```

---

## Come si legge una regola di Suricata (panoramica)

Una regola definisce **cosa** osservare e **quale azione** intraprendere. In modalità IDS l'azione tipica è `alert`:

```suricata
alert http any any -> any any (msg:"Test Lab - Richiesta HTTP GET"; flow:to_server; http.method; content:"GET"; sid:1000001; rev:1;)
```

* **`alert`**: genera un alert senza bloccare il traffico (caratteristica della modalità IDS).
* **`http any any -> any any`**: protocollo `HTTP`, qualsiasi IP/porta di origine e destinazione nella direzione indicata.
* **`msg:"..."`**: descrizione dell'alert che apparirà nei log.
* **`flow:to_server; http.method; content:"GET";`**: condizione di match sul traffico verso il server.
* **`sid:1000001; rev:1;`**: identificativo univoco della regola locale (da `1000000` in su per regole personalizzate) e revisione.

---

## Competenze Dimostrate in Questo Laboratorio

* Installazione, aggiornamento regole (`suricata-update`) e validazione della configurazione (`suricata -T`) di un sensore **IDS** su Linux.
* Comprensione pratica della modalità **IDS** (rilevamento e logging senza blocco) e distinzione rispetto alla modalità **IPS**.
* Navigazione e interpretazione dei log in `/var/log/suricata/` (`eve.json`, `fast.log`, `stats.log`, `suricata.log`).
* Estrazione e analisi mirata di flussi, DNS, HTTP/TLS e alert tramite `jq` da riga di comando.
