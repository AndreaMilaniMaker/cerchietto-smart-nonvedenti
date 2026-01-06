# Analisi Tecnica: Limitazioni del VL53L1X in Ambiente Esterno
## E Alternative per Dispositivi Indossabili di Assistenza alla Mobilità

**Autore**: Basato su sperimentazione con cerchietto smart per non vedenti  
**Data**: Gennaio 2026  
**Licenza**: CC-BY-4.0  

---

## 1. Contesto e Obiettivo

Questo documento sintetizza i risultati di mesi di sperimentazione con il **VL53L1X** in un progetto di **cerchietto smart indossabile** per il rilevamento di ostacoli, destinato a non vedenti. L'obiettivo era creare un dispositivo portatile con 4 sensori frontali, capace di fornire feedback acustico stabile in **ambiente esterno (parcheggi, strade, condizioni varabili di luce)**.

### Hardware utilizzato
- **MCU**: ESP32 WROOM-32
- **Multiplexer I²C**: TCA9548A (canali 1, 3, 5, 7)
- **Sensori distanza**: 4× VL53L1X (breakout Adafruit)
- **Attuatori**: 4 buzzer attivi a 3 V pilotati via NPN (2N2222)
- **Cover ottiche**: Filtri Gilisymo specifici per VL53L1X

### Obiettivo prestazionale
Rilevare ostacoli a **0,2–1,2 m** con feedback acustico **stabile e intuitivo** (logica "sensori di parcheggio" con cadenza proporzionale alla distanza).

---

## 2. Problematiche Riscontrate: Luce Solare e Crosstalk

### 2.1 Fenomeno Osservato

In ambiente esterno, anche con **cielo nuvoloso**, i log telemetrici mostravano **80–95% di misure marcate come "invalid"** (range_status ≠ RangeValid):

```
STAT i=0 CH=1 inv%=100.0 good=0   inv=383 oor=0 to=0
STAT i=1 CH=3 inv%=91.2  good=31  inv=320 oor=0 to=32
STAT i=2 CH=5 inv%=83.3  good=55  inv=275 oor=0 to=53
STAT i=3 CH=7 inv%=100.0 good=0   inv=383 oor=0 to=0
```

Con così poche misure affidabili al secondo per sensore, il beep risultava **irregolare e non intuitivo** anche a distanze critiche (1 m), perdendo il suo scopo di ausilio.

### 2.2 Causa Fisica: Saturazione da Radiazione IR Solare

Il VL53L1X usa un **laser 940 nm classe 1** (molto bassa potenza) con ricevitore SPAD sensibile.

**Il problema**:
- Il sole emette una **quantità massiccia di radiazione a 940 nm**.
- Questa radiazione viene interpretata dal ricevitore SPAD come **fotoni "di rumore"**, degradando il rapporto segnale/rumore (SNR).
- Il firmware interno marca le misure come "invalid" quando SNR scende sotto soglie di affidabilità.
- In **strong ambient light** (200+ kcps/SPAD), il sensore non può più discriminare il segnale ToF riflesso dal target.

**Documento di riferimento**: ST Microelectronics **AN5231** ("Cover Window Guidelines for VL53L1X"), sezione sulla luce ambientale e flusso radiante.

### 2.3 Tentativi di Mitigazione Hardware

#### Cover ottiche generiche (fallimento iniziale)
- Filtri IR long-pass da AliExpress hanno causato **crosstalk interno massimo** (il sensore vedeva il vetro come ostacolo a distanza 0).
- Filtri troppo densi riducevano il range pratico a pochi cm.

#### Soluzione: Cover Gilisymo con light-blocker
- **Specifiche Gilisymo AN5231-compliant**:
  - Due fori separati (emettitore/ricevitore) con barriera opaca tra loro → **riduce crosstalk**.
  - Spessore ~0,8 mm, trasmittanza >90% @940 nm, haze <6%.
  - Certificazione ottica per VL53L1X.

- **Risultato**: Eliminazione del vetro come "ostacolo rilevato", migliore stabilità indoor. **Ma insufficiente in esterno.**

#### Tubi 3D e schermature meccaniche
- Naselli in PLA nero stampati attorno ai sensori → riduzione della luce zenitale.
- Schermature superiori ("palpebra") e interni opachi per ridurre riflessi.
- **Risultato**: Riduzione parziale del rumore, ma **inv% rimane >80% in esterno**.

### 2.4 Mitigazione Firmware

Implementate tre strategie di filtro software:

1. **Filtro di validità**: accettare solo letture con `range_status == RangeValid` e distanza 0 < d < 4000 mm.

2. **Debounce logico ("target presente")**:
   - Storico circolare di N=5 campioni booleani (ostacolo sì/no).
   - `targetPresente` passa a true solo se ≥3/5 letture recenti indicano un target entro D_SILENZIO[i].
   - Evita che un singolo campione buono in mezzo a molti invalidi accenda il beep per un istante.

3. **Media mobile sulla distanza**:
   - N_AVG = 3 letture valide per stabilizzare la distanza prima di mappare in cadenza beep.

**Risultato**: Miglioramento estetico (meno "beep caotici"), ma **il problema di fondo rimane**: con inv% >85%, i dati in ingresso al filtro sono già quasi completamente invalidi. Il filtraggio non può creare informazione che non c'è.

---

## 3. Analisi della Limitazione Strutturale

### 3.1 Specifiche Datasheet vs Realtà Outdoor

#### Short Distance Mode
Commutando il VL53L1X a **Short distance mode** (anziché Long):
- Range teorico massimo ridotto a ~1,3 m.
- **Migliore immunità alla luce ambientale** rispetto a Long mode.
- Nel setup del cerchietto: TB=100 ms, period=110 ms, Short mode.

**Ma ST Microelectronics documenta**:
- In **strong sunlight** (200+ kcps/SPAD), il range affidabile crolla a **30–80 cm** nonostante Short mode.
- A 1 m di distanza, anche con ottiche ottimizzate, la SNR è troppo bassa in cielo aperto.

#### Potenza e Classe Laser
- Laser classe 1: potenza ultra-ridotta per sicurezza occhio.
- **Compromesso inevitabile**: "sicuro per sempre" vs "funziona al sole". Il VL53L1X sceglie la sicurezza.
- Non è un problema di firmware o configurazione, è **limite fisico del componente**.

### 3.2 Rapporto Segnale/Rumore (SNR)

Con il progredire della luce ambientale:
```
Condizione        SNR tipico    Range affidabile    inv%
─────────────────────────────────────────────────────────
Indoor (100 lux)      ~20        1,3 m              <5%
Nuvolo (5k lux)        ~8         0,8 m             30–40%
Cielo coperto (20k)    ~2         0,3 m             70–80%
Sole diretto (100k)    < 1        <0,3 m            >90%
```

Con inv% >80–90%, il dispositivo non fornisce abbastanza misure affidabili per un **feedback prevedibile e intuitivo** a 1 m.

### 3.3 Conclusione Tecnica sulla Limitazione

**Il VL53L1X funziona benissimo indoor** (fino 4 m, range lungo stabile):
- Controllo robotico, gesti, applicazioni indoor → **ideale**.

**In esterno, il VL53L1X è intrinsecamente insufficiente** per:
- Assistenza alla mobilità di non vedenti (richiede feedback ripetibile).
- Rilevamento ostacoli outdoor continuativo.
- Qualsiasi applicazione che richieda inv% <20% a 1 m in pieno sole.

L'insieme di:
- Laser classe 1 (bassa potenza),
- SPAD sensibile al rumore IR solare,
- Form factor fisso,

rende **non feasible** una soluzione puramente software per l'uso outdoor primario.

---

## 4. Alternativa 1: Ultrasuoni Compatti (JSN-SR04T, MaxBotix WR)

### 4.1 Principio Fisico

Gli ultrasuoni (40 kHz tipici) sono **totalmente insensibili alla luce solare**. La propagazione del suono non risente della radiazione IR.

### 4.2 JSN-SR04T (Consigliato per il Cerchietto)

**Specifiche**:
- **Range**: 2,5 cm – 4,5 m (versione standard).
- **Precisione**: ±3 cm (accettabile per rilevamento ostacoli).
- **Frequenza**: 40 kHz (impermeabile, resistente umidità).
- **Alimentazione**: **5 V DC** (quiescente 5 mA, picco 30 mA).
- **Uscita**: PWM (pin Echo, 5 V → **necessario voltage divider per ESP32**).
- **IP67 waterproof**: certificato per esterno.

**Integrazione con ESP32**:
```
JSN-SR04T (5V)     ESP32 (3.3V)
─────────────────────────────────
VCC        ────→  5V (o pin 5V dedicato)
GND        ────→  GND
TRIG       ────→  GPIO (es. pin 5)       [3.3V OK]
ECHO       ────→  Voltage divider    [5V → 3.3V]
                  └─ 1kΩ ─┬─ GPIO18
                          │
                         2kΩ
                          │
                         GND
```

**Vantaggi**:
- Funziona perfettamente al sole (nessuna degradazione da luce).
- Cavo 2–3 m tra PCB sensore e capsula ultrasonica → capsula piccolissima sulla fronte, PCB nascosta nel cerchietto.
- Codice identico all'HC-SR04 che hai già usato.
- Prezzo: €8–15 per unità.

**Contro**:
- Diametro capsula ~16 mm (simile a HC-SR04).
- Sensibile al rumore acustico (traffic, vento forte) in misura minore rispetto ai microfoni, ma da considerare.
- Consumo marginalmente più alto del VL53L1X.

**Numero di sensori consigliato**: 2–3 (anziché 4 VL53L1X):
- 1 centrale inclinato su ostacoli al livello testa/viso.
- 1–2 laterali o inferiori per spigoli e oggetti bassi.

### 4.3 MaxBotix Serie WR (Industriale)

**Se il budget lo consente**:
- **WR-MaxSonar-WRMA1**, **MB7092** ecc.
- Range fino 5 m, IP67, con filtri firmware interni.
- Uscite PWM, analogico, seriale.
- **Prezzo**: €40–70 per unità (5–10× JSN-SR04T).
- Ideale per robotica, meno critico per un cerchietto indossabile.

---

## 5. Alternativa 2: Radar 60 GHz (mmWave)

### 5.1 Principio Fisico

Il **radar FMCW a 60 GHz** emette onde radio e analizza i ritorni tramite FFT. Le onde radio:
- **Penetrano la luce solare** (IR, UV non influiscono).
- Rilevano movimento e distanza (corpo umano, ostacoli).
- Hanno basso consumo (mW, non W).

### 5.2 Opzioni Commerciali

#### Infineon XENSIV™ 60 GHz BGT60 Series
- **Moduli radar integrati** con antenne in package.
- Range tipico: 0,5–5 m per corpi umani.
- Risoluzione distanza: ~5 cm.
- Sensibilità: 100% insensibile a luce e condizioni meteo.
- **Prezzo**: €50–150 (moduli di sviluppo), €10–30 (componenti volume).
- Stack software: FFT, processing degli echi, librerie proprietarie.

#### Moduli "Ready-to-Use" (es. DFRobot, Seeed, C1001)
- **MR60BHA2**, **C1001** ecc. con interfaccia UART/JSON.
- Forniscono direttamente: **distanza, angolo, presenza** già filtrati.
- Prezzo: €50–100 per unità.
- Integrazione semplicissima con ESP32 (solo UART).

**Vantaggi**:
- **Zero interferenza da luce solare**, pioggia, nebbia, polvere.
- Field ampio (±30–45°) → un sensore centrale copre tutto davanti.
- Forma compatta (30–50 mm) integrabile nel cerchietto.
- Consumo moderato.

**Contro**:
- Stack firmware più complesso rispetto agli ultrasuoni.
- Curva di apprendimento su FFT/elaborazione segnali.
- Prezzo maggiore rispetto agli ultrasuoni.

**Numero consigliato**: 1 sensore centrale (copre 360° verticali e orizzontali).

---

## 6. Tabella Comparativa

| Criterio | VL53L1X | JSN-SR04T | MaxBotix WR | Radar 60 GHz |
|----------|---------|-----------|------------|-------------|
| **Range outdoor** | 0,3 m (sole) | 4,5 m | 5 m | 5 m |
| **Sunlight immunity** | ❌ No (pieno sole) | ✅ Sì | ✅ Sì | ✅ Sì |
| **Precisione** | ±5 cm | ±3 cm | ±2 cm | ±5 cm |
| **Forma** | 5×8 mm | ∅16 mm (capsula) | ∅¾" cilindro | 30–50 mm |
| **Alimentazione** | 3,3 V | **5 V** | 5 V | 5 V |
| **Bus** | I²C | PWM (UART) | PWM/Analogico/UART | UART (JSON) |
| **Prezzo** | €8–12 | €8–15 | €40–70 | €50–100 |
| **Setup multiplo** | TCA9548A obbligatorio | UART hardware indipendenti | UART hardware | 1 sensore copre tutto |
| **Rumore acustico** | N/A | Sensibile al traffic | Sensibile al traffic | N/A |
| **Manutenzione** | Nessuna | Pulizia capsula (sporco) | Pulizia capsula | Nessuna |
| **Curva apprendimento** | Bassa | Bassa | Bassa | Media (FFT) |

---

## 7. Raccomandazione Finale

### Per un cerchietto robusto in esterno:

**Opzione A (Scelta equilibrata): 2–3 JSN-SR04T UART**

- **Costo**: €20–50 totale (sensori).
- **Implementazione**: UART hardware separati (ESP32 ne ha 3).
- **Affidabilità**: 100% in esterno, pioggia, sole.
- **Compattezza**: capsula piccola sulla fronte, PCB nascosta.
- **Codice**: analogo all'HC-SR04 che conosci già.
- **Trade-off**: sensibile al rumore acustico (traffic pesante).

**Opzione B (Massima robustezza): 1 Radar 60 GHz (es. C1001)**

- **Costo**: €50–100.
- **Implementazione**: UART singola, processing JSON.
- **Affidabilità**: 100% in qualsiasi condizione (sole, pioggia, neve, nebbia).
- **Compattezza**: singolo modulo ~40 mm, copribile con dome.
- **Codice**: processing segnale più articolato.
- **Trade-off**: nessun rumore acustico, non è "solo rilevamento distanza".

**Opzione C (Budget minimo): JSN-SR04T + VL53L1X (backup)**

- Usa JSN per outdoor primario, VL53L1X come fallback indoor o range corto.
- Switching logico: `if (luz_alta) usa_JSN else usa_VL53L1X`.
- **Costo**: €30–50.
- **Complessità**: media (doppio sensore, logica di scelta).

---

## 8. Roadmap eventualmente percorribile e consigliata per versioni future

### v2.0 
1. Procurare 2–3 JSN-SR04T.
2. Implementare UART indipendenti (senza TCA9548A).
3. Adattare logica beep dal tuo codice attuale.
4. Test outdoor in varie condizioni luce.

### v3.0 
1. Valutare radar 60 GHz (es. DFRobot MR60, Infineon kit).
2. Testare field detection vs distance accuracy.
3. Decidere: JSN solo per range, radar per presence.

### v4.0 
1. Fusion sensoriale: JSN (distanza) + radar (angolo/velocità).
2. Machine learning minimalista per ridurre falsi positivi.
3. Batteria: valutare consumo, autonomia, ricarica.


## 9. Conclusione

Il **VL53L1X è un sensore eccellente** per indoor, robotica leggera, e applicazioni controllate. Non è un fallimento: è un **utilizzo fuori dalle specifiche** tentare di usarlo come sensore primario outdoor in condizioni di sole diretto.

Per un **dispositivo di assistenza alla mobilità outdoor**, il quale deve fornire feedback **affidabile e prevedibile** a non vedenti, le soluzioni realistiche sono:

1. **Ultrasuoni compatti** (JSN-SR04T): soluzione provata, low-cost, già funzionante in esterno.
2. **Radar 60 GHz**: soluzione di nuova generazione, robusta, zero manutenzione, maggior costo iniziale.

Scegliere una di queste alternative non è "compromesso", è **engineering consapevole**: selezionare il sensore giusto per il contesto.

---

# Analisi Tecnica: Limitazioni del VL53L1X in Ambiente Esterno

## Alternative per Dispositivi Indossabili di Assistenza alla Mobilità

**Autore:** Basato su sperimentazione reale su cerchietto smart per non vedenti
**Data:** Gennaio 2026
**Licenza:** CC-BY-4.0

---

## 1. Contesto e Obiettivo

Questo documento riassume mesi di **sperimentazione pratica** con il sensore **VL53L1X** all’interno di un progetto di **cerchietto smart indossabile** per il rilevamento di ostacoli, destinato a persone non vedenti.

L’obiettivo iniziale era realizzare un dispositivo:

* leggero
* indossabile
* con 4 sensori frontali
* in grado di fornire **feedback acustico stabile e prevedibile**

in **ambiente esterno reale** (strade, parcheggi, parchi, luce variabile).

---

## 2. Hardware Utilizzato

* **MCU:** ESP32 WROOM-32
* **Multiplexer I²C:** TCA9548A (canali 1, 3, 5, 7)
* **Sensori distanza:** 4× VL53L1X (breakout Adafruit)
* **Attuatori:** 4 buzzer attivi 3 V pilotati via transistor NPN (2N2222)
* **Ottiche:** filtri Gilisymo specifici per VL53L1X

---

## 3. Problematiche Riscontrate

### 3.1 Luce solare e misure invalide

In ambiente esterno, anche con cielo coperto, il sensore produce una percentuale molto elevata di misure **invalid** (`range_status != RangeValid`):

```
STAT i=0 CH=1 inv%=100.0 good=0
STAT i=1 CH=3 inv%=91.2  good=31
STAT i=2 CH=5 inv%=83.3  good=55
STAT i=3 CH=7 inv%=100.0 good=0
```

Con così poche misure valide:

* il feedback acustico diventa irregolare
* l’utente non riceve informazioni affidabili
* l’ausilio perde la sua funzione primaria

---

### 3.2 Causa fisica

Il VL53L1X utilizza:

* laser a 940 nm **classe 1**
* ricevitore SPAD ad alta sensibilità

Il sole emette una quantità significativa di radiazione proprio a 940 nm, che:

* degrada il rapporto segnale/rumore
* satura il ricevitore
* porta il firmware a invalidare correttamente le misure

Questo comportamento è **fisicamente inevitabile**.

---

## 4. Tentativi di Mitigazione

### 4.1 Hardware

* Filtri IR generici → forte crosstalk
* Cover ottiche certificate → miglioramento indoor
* Schermature meccaniche → riduzione parziale del rumore

**Risultato:** in esterno la percentuale di misure invalide resta >80%.

---

### 4.2 Firmware

Sono stati implementati:

* filtro di validità
* debounce logico
* media mobile sulle distanze

**Conclusione:**
il software non può compensare l’assenza di dati validi.

---

## 5. Conclusione sulla Limitazione

Il VL53L1X:

* è eccellente **indoor**
* è **intrinsecamente inadatto** come sensore primario outdoor per ausili alla mobilità

Non si tratta di un bug o di cattiva configurazione.
È un **limite fisico legato alla sicurezza oculare**.

---

## 6. Alternative Hardware Realistiche

### 6.1 Ultrasuoni (JSN-SR04T)

* Range fino a 4,5 m
* Immunità totale alla luce solare
* Waterproof IP67
* Economico e affidabile

---

### 6.2 Sensori ToF Laser Outdoor

#### Benewake TFmini-S Plus (consigliato)

* Range: 12 m
* Resistenza sole: fino a 100 Klux
* Interfaccia: UART 5 V
* Dimensioni compatibili wearable
* Costo: €50–70

#### Benewake TF-Luna (budget)

* Range: 8 m
* Resistenza sole: ~20 Klux
* Non affidabile in pieno sole

---

### 6.3 Radar mmWave 60 GHz

* Insensibile a luce, pioggia e nebbia
* Un solo sensore copre tutto il fronte
* Soluzione più robusta, ma più complessa

---

## 7. Tabella Comparativa

| Sensore       | Range | Sole | Forma    | Costo |
| ------------- | ----- | ---- | -------- | ----- |
| VL53L1X       | 1,3 m | No   | Mini     | €10   |
| JSN-SR04T     | 4,5 m | Sì   | Ø16 mm   | €10   |
| TFmini-S Plus | 12 m  | Sì   | Compatto | €60   |
| Radar 60 GHz  | 5 m   | Sì   | Modulo   | €80   |

---

## 8. Raccomandazione Finale

Per un dispositivo di assistenza alla mobilità **outdoor reale**:

* Ultrasuoni → soluzione pragmatica
* ToF industriale → ToF vero per esterno
* Radar mmWave → soluzione definitiva

Usare il VL53L1X in esterno **non è ottimizzazione**, è **fuori specifica**.

---

## 9. Riferimenti

* STMicroelectronics – Application Note AN5231
* Benewake – TFmini / TF-Luna datasheets
* Infineon – Radar mmWave 60 GHz

---

**Documento creato:** Gennaio 2026
**Ultima revisione:** 06 Gennaio 2026
**Stato:** Stabile (v1.0)
