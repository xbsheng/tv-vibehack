> Traduzione italiana di [docs/privacy.md](../privacy.md). In caso di differenze fa fede la pagina in inglese.

# Il microfono, e cosa si può dire

[Torna al README](../../README.it.md#il-microfono-e-cosa-si-può-dire)

«Quale di questi pacchetti ascolta?» non ha una risposta onesta in una riga, e una scansione dei permessi la
complica:

- Su molti apparecchi la maggior parte dei pacchetti gira come `android.uid.system`, quindi una cinquantina mostrano
  `RECORD_AUDIO: granted=true` e nessuno registra nulla.
- Su altri il microfono sta nel telecomando, non nel pannello, e un pacchetto senza alcun permesso audio vede comunque
  tutto ciò che c'è sullo schermo: i client ACR come `tv.inscape.tvclient.inscapeacr` riconoscono i contenuti dalle
  immagini e hanno bisogno solo di `INTERNET`.

`tools/privacy.py` riporta ciò che la TV dice davvero, invece di tirare a indovinare:
`TV_HOST=192.168.1.50 tools/venv/bin/python -I tools/privacy.py` stampa i candidati di voce, ACR e telemetria in
al massimo quattro gruppi (i tre tipi di cui sa indicare la funzione e un quarto intitolato «Matched by name only,
nothing else known», cioè riconosciuti solo dal nome), ciascuno con:

- la sua prova: indicato dal `voice_interaction_service` del firmware stesso, un componente noto di
  [`examples/`](../../examples/), oppure nient'altro che un nome corrispondente;
- se il pacchetto dichiara `RECORD_AUDIO` e se condivide l'UID di sistema;
- se la cronologia AppOps restituita contiene accessi al microfono. Una modalità configurata o un tentativo respinto, da
  soli, non sono una prova di accesso.

`--write privacy.txt` trasforma i candidati identificati in un normale elenco di app da disattivare, così applicarlo e
annullarlo sono i comandi che già usi (`tools/disable.sh`, `tools/restore.sh`). I pacchetti di quel quarto gruppo, quelli
riconosciuti solo dal nome, restano fuori a meno che tu non passi `--include-low`; sono quelli da controllare prima con
`dumpsys package <nome>`.

L'audit legge `dumpsys package` per al massimo 25 pacchetti, quindi su una TV con un elenco più lungo di nomi
corrispondenti il rapporto mostra i primi 25 e dice quanti ne ha lasciati fuori, e `--write` scrive soltanto quelli:
controlla a mano gli altri.

Ciò che l'audit non può fare è provare le intenzioni. `appops` mostra che un'operazione è stata toccata, non cosa è
successo al flusso dopo, e nessuna scansione dei nomi dei pacchetti prova che una TV stia registrando o no.
Disattivare un assistente è uno scambio che fai con una perdita precisa (il tasto del microfono del telecomando, la
ricerca vocale dentro le app), ed è per questo che il rapporto nomina quella perdita per ogni pacchetto, invece di
chiamarla una vittoria.

## ACR e il blocco dei suoi server

Un client ACR non ha sempre un nome che lo dica. Sulle TV Google TV di TCL il frame server di Samba TV vive dentro
`com.tcl.ttvs`, accanto al servizio di aggiornamento del firmware, quindi una ricerca dei soli nomi dei pacchetti lo
manca. `dumpsys package r | grep -i -E 'samba|acr'` elenca i componenti che lo portano, qualunque pacchetto li
contenga, e `pm list features | grep -i acr` mostra se il firmware attiva la funzione. L'audit nomina i pacchetti ACR
noti da [`examples/`](../../examples/), e disattivarli è la soluzione: un client spento non invia nulla.

Un blocco DNS è un secondo livello, per un client che decidi di tenere o che torna con un aggiornamento. Android TV
non ha un file hosts che puoi modificare senza root, quindi il blocco sta sulla rete: un Pi-hole o un AdGuard Home
che il router distribuisce come DNS, oppure un resolver Private DNS che accetta le tue regole (un profilo AdGuard DNS
o NextDNS personale). I server pubblici di AdGuard DNS della
[FAQ](faq.md#può-bloccare-la-pubblicità-anche-dentro-le-app) non bloccavano gli host `cid.samba.tv` di Samba TV
quando sono stati controllati a novembre 2026. I domini di secondo livello della
[blocklist Watching the Watchers](https://github.com/IlgarNEU/Watching-the-Watchers/tree/main/Blocklist)
(Northeastern University, 2026), con il servizio ACR a cui appartiene ciascuno:

| Blocco | Servizio ACR |
|---|---|
| `samba.tv` | Samba TV (TV Sony, Philips e TCL Google TV) |
| `alphonso.tv` | Alphonso (LG webOS, alcuni box Android TV) |
| `samsungacr.com`, `acr-us-prd.samsungcloud.tv` | Samsung |
| `tvinteractive.tv` | Inscape (Vizio) |
| `acr.unruly.co` | Unruly (VIDAA) |
| `scribe.logs.roku.com` | Roku |

La stessa lista dice che Samba TV si collega anche a un indirizzo fisso, `216.183.117.80`, che solo una regola del
firewall del router ferma. Una TV che risolve i nomi tramite il proprio DNS over HTTPS salta del tutto il blocco DNS,
e i produttori rinominano gli host, quindi controlla il log del resolver per i nomi nuovi invece di fidarti che la
lista resti completa.
