# PCBprojects

Raccolta di progetti hardware realizzati con KiCad.

## SensorNode

Sistema a radiofrequenza basato su un oscillatore con risonatore piezoacustico (SAW) a circa 110 MHz. Il segnale dell'oscillatore viene miscelato con un riferimento commerciale a 120 MHz. Il battimento risultante, a circa 10 MHz, viene filtrato e squadrato da un comparatore, così la sua frequenza può essere misurata digitalmente.

Il sistema è diviso in due schede: una scheda RF e una scheda di alimentazione (PowerBoard).

### Architettura

```
Batteria Li-ion 2S ──► PowerBoard ──► 3.3 V / 5 V ──► Scheda RF

Scheda RF:
  Oscillatore Pierce + SAW (~110 MHz) ──► porta IF ┐
                                                   ├─ Mixer ──► Filtro (~10 MHz) ──► Comparatore ──► SMA_out
  Oscillatore commerciale 120 MHz ─────► porta LO  ┘
```

### Struttura del repository

```
PCBprojects/
├── .gitignore
├── README.md
└── SensorNode/
    └── PowerBoard/      Scheda di alimentazione (progetto KiCad)
```

La scheda RF verrà aggiunta in una sottocartella dedicata di `SensorNode/`.

---

## PowerBoard

Scheda di alimentazione separata, che tiene i convertitori a commutazione fuori dalla scheda RF. È progettata per bassi consumi e, nella versione di test, permette di confrontare diverse configurazioni di regolazione.

### Ingresso

| Parametro | Valore |
|---|---|
| Batteria | Li-ion 2S (6.0–8.4 V, nominale 7.4 V) |
| Connettore | XT30, versione da PCB |
| Protezioni | Inversione di polarità (P-MOSFET), fusibile, sottotensione (spegnimento a circa 6.0–6.2 V) |

### Uscite

| Uscita | Carichi | Corrente stimata | Regolazione |
|---|---|---|---|
| 3.3 V | Oscillatore 120 MHz, comparatore (lato 3.3 V) | ~45 mA | Buck + LDO TLV70233, oppure buck diretto |
| 5 V | Comparatore (lato 5 V) | ~5 mA | Buck + LDO, oppure LDO da batteria |
| −0.55 V (opzionale) | Gate del FET SKY65050 | ~10 µA | Charge pump invertente + operazionale invertente |

### Configurazioni di test

La scheda di test permette di confrontare:

- **Buck + LDO oppure buck diretto:** selezione tramite resistenze da 0 Ω. La tensione d'uscita del buck va cambiata di conseguenza: circa 3.5–3.6 V con LDO, 3.3 V senza.
- **Modalità PFM oppure PWM forzato:** ponticello sul pin MODE del buck.
- **Filtro LC:** footprint configurabili, con un secondo stadio opzionale.

Sono previsti shunt per la misura dell'efficienza, punti di misura del ripple con massa corta e footprint per resistenze di carico di prova.

> **Nota sulla configurazione:** prima di collegare la scheda RF, verifica che la tensione del buck sia impostata correttamente per la configurazione scelta. In configurazione diretta, una tensione di 3.5 V finirebbe direttamente sulla scheda RF.

### Linea a −0.55 V

La polarizzazione negativa del gate è opzionale. L'alternativa consigliata dal datasheet del SKY65050 è l'autopolarizzazione: una resistenza di source (27–47 Ω, con bypass RF) e il gate a 0 V in continua. Questa soluzione elimina la tensione negativa e la sequenza di accensione.

Se si usa la linea negativa:

- la tensione di gate deve essere presente **prima** dell'alimentazione del drain, e deve essere tolta **dopo**;
- la corrente di gate non deve superare 100 µA (valore massimo assoluto): serve una resistenza in serie di almeno 27–33 kΩ o un diodo di clamp;
- il ripple della charge pump va filtrato con RC sull'alimentazione negativa dell'operazionale e sull'uscita.

### Componenti

| Funzione | Componente | Stato |
|---|---|---|
| LDO 3.3 V | TI TLV70233 (SOT-23-5) | Scelto |
| Buck 3.3 V / 5 V | — | Da definire (ingresso ≥ 9 V richiesto per la 2S) |
| LDO 5 V | — | Da definire |
| Charge pump invertente | — | Da definire |
| Operazionale per −0.55 V | LT6003 | Candidato |
| Connettore batteria | Amass XT30 (UPB o PW) | Scelto |

---

## Scheda RF

| Blocco | Componente / note |
|---|---|
| Oscillatore a 110 MHz | Pierce con FET pHEMT SKY65050-372LF e risonatore SAW |
| Oscillatore LO | Oscillatore commerciale a 120 MHz |
| Mixer | 110 MHz sulla porta IF, 120 MHz sulla porta LO, uscita a ~10 MHz |
| Filtro d'uscita | Passa-basso / passa-banda attorno a 10 MHz |
| Comparatore | Alimentato a 5 V e 3.3 V, uscita su SMA_out |

> **Nota:** il SKY65050-372LF risulta fuori produzione. Per nuove revisioni valutare un sostituto.

### Note di progetto

- Tenere i due oscillatori (110 e 120 MHz) fisicamente separati e alimentati da rami filtrati separatamente, per evitare trascinamento reciproco e spurie.
- Il FET ha guadagno fino a diversi GHz: prevedere una resistenza in serie al gate o una ferrite sul drain contro le oscillazioni parassite, e verificare lo spettro fino ad alcuni GHz.
- Ritorno DC del gate con resistenza elevata (100 kΩ – 1 MΩ), per non abbassare il Q del risonatore.
- Condensatori C0G/NP0 per la rete di reazione del Pierce.
- Piano di massa continuo sotto tutta la parte RF, con via di stitching e disaccoppiamento locale (100 nF + nF) su ogni integrato.

---

## Bilancio di potenza stimato

| Voce | Potenza |
|---|---|
| Oscillatore 120 MHz (3.3 V × 40 mA) | ~0.13 W |
| Comparatore | ~0.04 W |
| Oscillatore SAW con SKY65050 (~3 V × 10–20 mA) | ~0.03–0.06 W |
| Perdite di regolazione | ~0.05–0.1 W |

Valori indicativi, da aggiornare con le misure sulla scheda di test.

---

## Lavorare sul progetto

### Requisiti

- KiCad (indicare la versione usata dal gruppo: i file salvati con una versione più recente non si aprono con una più vecchia)
- Git

### Clonare il repository

```
git clone https://github.com/GiacomoGobbo/PCBprojects.git
```

### Flusso di lavoro

1. Prima di iniziare: `git pull`.
2. Lavorare sul progetto in KiCad.
3. Alla fine:
   ```
   git add .
   git commit -m "Descrizione delle modifiche"
   git push
   ```

### Regole

- Non modificare lo stesso file `.kicad_sch` o `.kicad_pcb` in contemporanea: i conflitti sui file KiCad sono difficili da risolvere. Dividersi il lavoro per scheda o per foglio di schema.
- Le librerie personali di simboli, footprint e modelli 3D vanno tenute nella cartella del progetto, con percorsi relativi (`${KIPRJMOD}`).
- Il footprint XT30 si trova nella libreria ufficiale `Connector_AMASS`. Per lo schema si usa il simbolo `Conn_01x02`.
- Non creare repository Git annidati nelle sottocartelle dei progetti.
- Il `.gitignore` nella cartella principale esclude backup, autosalvataggi, cache, file di lock, preferenze locali (`.kicad_prl`) e la cronologia locale di KiCad (`.history/`).

---

## Autori

- Giacomo Gobbo
