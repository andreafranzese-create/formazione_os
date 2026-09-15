# Gestione CSR per la richiesta di un nuovo certificato

---

## Come funziona

Un certificato dice *"questa chiave pubblica appartiene a questo soggetto"* ed è
firmato da qualcuno di cui il client si fida.

La Root CA è l'unico anello **self-signed**: Issuer e Subject coincidono, si firma
da sola. Il certificato del server invece è firmato da lei.

I tre oggetti sono:

- **chiave privata** (`.key`) — il numero segreto con cui si firma. Non esce mai
  dalla macchina che la possiede.
- **CSR** (`.csr`) — la richiesta: contiene la chiave **pubblica** e i dati
  identificativi, firmata con la chiave privata per dimostrare di possederla.
  È l'unica cosa che si manda alla CA.
- **certificato** (`.crt`/`.pem`) — la CSR dopo che la CA l'ha firmata.

---

## Struttura delle cartelle

```bash
mkdir csr certs private ca
```

```
track4/
├── ca/       rootCA.cnf · rootCA.pem · rootCA.srl
├── csr/      server.csr
├── certs/    server.crt
└── private/  rootCA.key · test.local.key
```

---

## FASE 1 — Root CA

### Chiave privata della CA

```bash
openssl genrsa -aes256 -out private/rootCA.key 4096
```

| Parametro | Significato |
|---|---|
| `genrsa` | genera una coppia di chiavi RSA |
| `-aes256` | cifra la chiave con una passphrase, chiesta interattivamente. Se qualcuno ruba il file, senza password non può firmare nulla |
| `-out private/rootCA.key` | file di output |
| `4096` | lunghezza in bit |

### Il file `ca/rootCA.cnf`

Invece di passare i dati da riga di comando si usa un file di configurazione,
perché servono le **estensioni X.509v3**: senza di esse il certificato non verrebbe
mai riconosciuto come CA.

```ini
[ req ]
default_bits       = 4096
prompt             = no
default_md         = sha256
distinguished_name = dn
x509_extensions    = v3_ca

[ dn ]
C  = IT
ST = Lazio
L  = Roma
O  = Sourcesense
OU = IT Lab
CN = Root CA Locale

[ v3_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign
```

**`[ req ]`** — come si comporta il comando:

- `prompt = no` → non chiedere niente interattivamente, prendi i dati dalla
  sezione `[ dn ]`
- `default_md = sha256` → algoritmo di hash per la firma
- `distinguished_name = dn` → i campi del soggetto stanno in `[ dn ]`
- `x509_extensions = v3_ca` → le estensioni da mettere nel certificato stanno in
  `[ v3_ca ]`

**`[ dn ]`** — il Distinguished Name, cioè l'identità: `C` country (2 lettere),
`ST` regione, `L` città, `O` organizzazione, `OU` reparto, `CN` il nome della CA
(quello che si vedrà nell'elenco delle autorità).

**`[ v3_ca ]`** — le estensioni:

- `basicConstraints = critical, CA:true, pathlen:0` → dichiara che questo
  certificato **è una CA**: senza, nessun software accetterebbe le sue firme.
  `critical` significa che chi non sa interpretare l'estensione deve rifiutare il
  certificato invece di ignorarla. `pathlen:0` significa che questa CA può firmare
  solo certificati finali, non altre CA intermedie.
- `keyUsage = critical, digitalSignature, cRLSign, keyCertSign` → a cosa può
  servire la chiave: `keyCertSign` (firmare certificati) è l'uso essenziale,
  `cRLSign` serve per le liste di revoca.
- `subjectKeyIdentifier` / `authorityKeyIdentifier` → identificativi usati dai
  client per ricostruire la catena di fiducia.

### Certificato self-signed

`rootCA.pem` è il certificato pubblico della CA. Contiene la sua chiave pubblica, che i client usano per verificare le firme, ed è il file che va importato nel **trust store** dei computer che vogliamo che si fidino della CA. È self-signed perché si firma da solo: Issuer e Subject coincidono.

```bash
openssl req -x509 -new -key private/rootCA.key -sha256 -days 3650 -out ca/rootCA.pem -config ca/rootCA.cnf
```

| Parametro | Significato |
|---|---|
| `req` | sottocomando per creare richieste di certificato |
| `-x509` | invece di una CSR genera direttamente un **certificato self-signed**: è il modo standard di creare una root CA |
| `-new` | nuovo certificato da zero |
| `-key private/rootCA.key` | la chiave privata: da lì si ricava la chiave pubblica da inserire e con essa si firma |
| `-sha256` | algoritmo di hash della firma |
| `-days 3650` | validità: 10 anni |
| `-out ca/rootCA.pem` | il certificato prodotto |
| `-config ca/rootCA.cnf` | il file con dati ed estensioni |

Il comando chiede la passphrase della chiave: è la prova che sta firmando.

---

## FASE 2 — Creare la CSR

Ora cambia il ruolo: siamo il server che chiede un certificato. La sua
chiave è **diversa** da quella della CA.

```bash
openssl req -newkey rsa:2048 -nodes -keyout private/server.key -out csr/server.csr
```

| Parametro | Significato |
|---|---|
| `req` senza `-x509` | il risultato è una CSR, non un certificato |
| `-newkey rsa:2048` | genera al volo una nuova chiave RSA da 2048 bit, senza doverla creare prima con `genrsa` |
| `-nodes` | la chiave viene salvata **senza passphrase**, così il server può ripartire da solo senza che qualcuno digiti una password |
| `-keyout private/server.key` | dove salvare la chiave appena creata |
| `-out csr/server.csr` | la CSR prodotta |

---

## FASE 3 — Firmare la CSR con la Root CA

```bash
openssl x509 -req -in csr/server.csr -CA ca/rootCA.pem -CAkey private/rootCA.key -CAcreateserial -out certs/server.crt -days 825
```

| Parametro | Significato |
|---|---|
| `x509 -req` | l'input non è un certificato ma una **CSR**: leggila e produci un certificato |
| `-in csr/server.csr` | la CSR da firmare |
| `-CA ca/rootCA.pem` | il certificato della CA firmataria: il suo Subject diventa l'**Issuer** del nuovo certificato |
| `-CAkey private/rootCA.key` | la chiave privata della CA, con cui si calcola la firma (chiede la passphrase) |
| `-CAcreateserial` | crea o aggiorna `ca/rootCA.srl`, il file che tiene il numero seriale: ogni certificato emesso da una CA deve averne uno univoco |
| `-out certs/server.crt` | il certificato finale |
| `-days 825` | validità, circa 2 anni e 3 mesi. Senza questo parametro OpenSSL userebbe un default di 30 giorni |