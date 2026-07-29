# CLAUDE.md

Regole per lavorare su questo progetto. Leggerle prima di ogni modifica.

## ⚠️ REGOLA CRITICA — massima priorità

**Se una richiesta è ambigua o sottospecificata, FERMARSI E CHIEDERE prima di
implementare.** Non indovinare, non interpretare a piacere, non scegliere in
autonomia il comportamento "più sensato" quando esistono più interpretazioni
plausibili e diverse tra loro. Questo vale in particolare per: comportamenti
UX non specificati nel dettaglio (es. cosa deve succedere esattamente quando
un utente interagisce con un elemento in uno stato particolare), nuove
funzionalità descritte in modo vago ("aggiungi un link", "migliora X"), e
qualsiasi modifica che tocca dati condivisi/produzione. Meglio una domanda in
più che una modifica sbagliata o, peggio, dati persi.

## ⚠️ REGOLA CRITICA — niente emoji non richieste

**Non aggiungere emoji a UI, testo, commit message o altro contenuto a meno
che l'utente non le richieda esplicitamente per quel caso specifico.**
Questo vale anche per emoji già presenti nel progetto: non riusarle altrove
né aggiungerne di nuove "per coerenza" o "per abbellire" di propria
iniziativa. Se l'utente chiede un'emoji specifica in un punto preciso,
aggiungerla solo lì.

## Cos'è

Webapp gratuita e multi-device per tenere traccia delle serie TV guardate
(stagione/episodio), usata senza login da un piccolo gruppo di persone
(io + partner) che vedono e modificano gli stessi dati.

## Decisioni architetturali (non cambiare senza discuterne)

- **Stack**: React + Vite + Tailwind CSS v4 (`@tailwindcss/vite`). Nessun
  framework SSR: non serve, l'app è puramente client-side.
- **Hosting**: GitHub Pages (statico, gratuito). Deploy automatico via
  `.github/workflows/deploy.yml` ad ogni push su `main`.
  - `vite.config.js` ha `base: '/tv-series-tracker/'`: se il nome del repo
    cambia, va aggiornato anche qui.
  - Routing con `HashRouter` (non `BrowserRouter`): GitHub Pages non supporta
    rewrite lato server per client-side routing, l'hash evita 404 sui
    refresh/link diretti.
- **Storage dati**: Firebase Firestore (piano gratuito Spark), **non**
  localStorage (serve multi-device) e **non** un database tradizionale da
  amministrare. Firestore è stato scelto rispetto a Supabase perché il piano
  free di Supabase mette in pausa i progetti dopo ~7 giorni di inattività;
  Firestore no — coerente con il requisito "niente servizi che si
  disattivano se non usati costantemente".
  - Un unico documento condiviso: `library/shared`. Nessun login, nessun
    codice personale: chiunque apra l'app vede e modifica la stessa
    libreria (vedi `src/lib/firebase.js` e `src/store/VaultContext.jsx`).
  - **Compromesso di sicurezza noto e accettato**: essendo un'app statica
    senza autenticazione, la config Firebase è visibile nel bundle JS (è
    inevitabile lato client) e le regole Firestore permettono lettura/
    scrittura aperta su quel singolo documento. Chiunque conosca l'URL
    dell'app potrebbe in teoria leggere/scrivere i dati. Accettabile per un
    tracker personale a basso rischio tra poche persone fidate. Non
    aggiungere dati sensibili a questo documento.
  - Aggiornamenti concorrenti (io + partner che spuntano episodi diversi
    nello stesso momento) usano `updateDoc` con path puntati
    (`series.<id>.watched.S1E2`), mai riscritture dell'intero documento:
    mantenere questo pattern per ogni nuova mutazione sui dati.
  - **Trappola nota (bug reale già capitato, non ripetere)**: `setDoc(ref,
    { campo: {} }, { merge: true })` con un oggetto **vuoto** NON lascia
    intatto il campo esistente — Firestore calcola la maschera di merge dai
    percorsi delle chiavi annidate, e un oggetto vuoto non ne ha nessuna,
    quindi il campo viene sostituito interamente con `{}`. Questo ha
    cancellato l'intera libreria condivisa (bug corretto in
    `VaultContext.jsx`: mai fare un "ensure doc exists" con un campo vuoto su
    un doc che può già contenere dati). Se serve creare il documento al primo
    utilizzo, farlo solo con `setDoc` di un oggetto che contiene già i dati
    reali da scrivere (es. dentro `addSeries`), mai con un placeholder vuoto
    su un mount/effect che gira ad ogni apertura dell'app.
- **Dati serie**: ricerca tramite TMDB API (`src/lib/tmdb.js`) con fallback
  di inserimento manuale (tab "Manuale" in `AddSeriesModal`) per le serie di
  nicchia assenti su TMDB. Non rimuovere l'opzione manuale.
  - **Icone in stile unico**: `SearchIcon.jsx`, `RefreshIcon.jsx`,
    `CloseIcon.jsx` e `ShareIcon.jsx` condividono deliberatamente la stessa
    "famiglia" — `viewBox="0 0 24 24"`, `stroke="currentColor"`,
    `strokeWidth="2"`, `strokeLinecap="round"` — invece di mischiare SVG
    disegnate a mano con caratteri Unicode testuali (`↻`, `✕`) come nelle
    versioni precedenti: da un lato quei glifi rendono in modo incoerente
    tra i font dei dispositivi (spesso visibilmente più piccoli/sottili di
    altre icone allo stesso font-size, anche dopo aver provato a compensare
    con `text-lg`), dall'altro anche mettendo a posto le dimensioni un
    glifo di testo e una SVG disegnata a mano restano stilisticamente
    diversi l'uno dall'altra (spessore del tratto, stile). Un'unica libreria
    di icone SVG elimina entrambi i problemi in una volta. `RefreshIcon.jsx`
    è stato inoltre ridisegnato: non più un singolo arco a 270° con una
    freccia (asimmetrico, poco leggibile a 18px), ma il classico refresh a
    due frecce circolari opposte (`refresh-cw`), più bilanciato e
    immediatamente riconoscibile. `CloseIcon.jsx` sostituisce **ogni**
    occorrenza del carattere `✕` nell'app (bottone di chiusura di `Modal`,
    bottone elimina di `SeriesDetail`, bottoni "cancella" nelle barre di
    ricerca, rimozione riga stagione in `AddSeriesModal`) — stessa azione
    "X" ovunque appaia, stesso componente, mai duplicata come glifo diverso
    in un punto e SVG in un altro. `SearchIcon.jsx` resta invariata (era già
    nella stessa famiglia). Bottone "Aggiorna da TMDB" nella riga del titolo
    in `SeriesDetail.jsx`,
    visibile solo per `series.source === 'tmdb'` (`refreshFromTmdb` in
    `VaultContext.jsx`); ruota (`animate-spin`) mentre è in corso, eventuale
    errore mostrato come riga di testo sotto il titolo. Ri-scarica **solo**
    titolo/poster/stagioni/flag `ongoing`/durate episodi da TMDB — non tocca
    mai `watched`/`status`/`rating`/`link`/`watchDays`. Unica eccezione
    deliberata: se il refresh rivela episodi nuovi rispetto a prima su una
    serie il cui stato era "Completata" o "In attesa di nuova stagione"
    (cioè tutto il conosciuto era già visto), lo stato torna a "Da vedere" —
    così il normale meccanismo planned→watching (vedi sotto) si riattiva da
    solo quando l'utente segna visto il primo episodio nuovo, senza bisogno
    di logica ad-hoc separata.
  - **Link Wikipedia EN/IT**: mostrati in `SeriesDetail.jsx`, calcolati di
    default dal titolo (`src/lib/wikipedia.js`, `wikipediaUrl(title, lang)`)
    — nessuna chiamata API, è un link "indovinato" diretto all'URL
    dell'articolo con quel titolo. **Modificabile/sovrascrivibile** per
    singola serie (`series.wikipediaEn`/`wikipediaIt`, `setWikipediaLink` in
    `VaultContext.jsx`) per i casi in cui il titolo indovinato punta
    all'articolo sbagliato o a una pagina di disambiguazione — stesso
    pattern edit/display di `LinkRow`. Cancellare l'override torna al link
    calcolato, non lo rimuove del tutto. Non introdurre una vera
    integrazione API Wikipedia per questo.
- **Env vars**: tutte le chiavi (Firebase, TMDB) vanno lette da
  `import.meta.env.VITE_*`, mai hardcoded nel codice. Vedi `.env.example`.
- **Icona app**: cuori blu e viola su un divano in stile illustrazione
  glossy/3D. Varianti derivate perché il divano intero non si legge sotto
  i ~40px:
  - `public/favicon-16.png`/`favicon-32.png` — dedicated pre-sized crops of
    the heart pair (same "crop tight on just the hearts" idea as the
    original single `favicon.png`, since the full sofa+hearts scene isn't
    legible under ~40px), one file per exact pixel size instead of one file
    scaled by the browser. Referenced with explicit `sizes="16x16"`/
    `"32x32"` (`<link rel="icon">`, two tags, in `index.html`) so each
    browser picks the exact match instead of downscaling a single source.
    Replaces the earlier single `favicon.png` (crop stretto della sola
    coppia di cuori, sfondo `accent-solid` pieno, ~22% di raggio per
    continuità visiva col vecchio `favicon.svg` disegnato a mano) — quel
    file resta nel repo ma non è più referenziato da nessuna parte
    dell'app; se non serve più a nulla può essere rimosso in un secondo
    momento.
  - `public/icon-192.png`/`icon-512.png` — **artwork illustrata** (non un
    export da un SVG vettoriale: sono l'immagine sorgente originale,
    fornita/curata direttamente dall'utente, sfondo **trasparente**
    intenzionale), usata in `manifest.json` come icone `"purpose": "any"` e
    come base per la variante maskable sotto, e in `Header.jsx` come logo
    dell'app (`<img>`, non più le emoji `💙💜` di testo — sostituite
    perché non stavano allo stesso livello qualitativo dell'artwork vera
    e propria, né lo stesso testo `💙💜` nel `<title>` di `index.html` e
    nel `name` di `manifest.json`, per lo stesso motivo). Il logo
    nell'header è **solo l'icona, a ogni larghezza** (non più icona+nome:
    con l'icona reale al posto delle emoji, il nome accanto risultava
    ridondante — vedi sotto la scelta di lasciare "Calendario" come
    unico link di navigazione), dimensionata `h-9 w-9 sm:h-10 sm:w-10`,
    tramite `${import.meta.env.BASE_URL}icon-512.png` (mai un path
    assoluto tipo `/icon-512.png`: romperebbe il base path
    `/tv-series-tracker/` su GitHub Pages — `BASE_URL` lo include già).
    Il logo resta un link a `/` (Libreria): con l'icona sola a fare da
    home-link, il precedente link di testo "Libreria" nell'header
    portava alla stessa identica pagina — rimosso perché puramente
    ridondante (il pattern logo-porta-alla-home è una convenzione web
    universale e a costo zero, un secondo link esplicito allo stesso
    posto non aggiungeva nulla), lasciando "Calendario" come unico link
    di navigazione esplicito testuale — sotto `sm`, per spazio (l'header
    ospita anche il filtro 💙/💜 e la ricerca, vedi sotto), diventa
    un'icona (`CalendarIcon.jsx`, stessa famiglia SVG di Search/Share/
    Refresh/Close) invece del testo; da `sm` in su resta il testo
    "Calendario" come prima — stesso `<Link>`, solo il contenuto interno
    cambia per breakpoint (`sm:hidden`/`hidden sm:inline`), non due link
    separati. Stesso trattamento per il bottone "+ Aggiungi serie":
    sotto `sm` mostra solo "+", da `sm` in su il testo completo — la sua
    `onClick` non cambia, resta lo stesso bottone.
    `apple-touch-icon.png`
    (180×180, sfondo pieno perché iOS non supporta trasparenza) è
    generato dalla stessa artwork. Se si aggiorna l'icona, farlo
    sostituendo direttamente questi PNG (l'artwork stessa è la fonte,
    niente da "rigenerare" da un SVG) e ri-derivare `apple-touch-icon.png`
    e la variante maskable sotto dalla nuova immagine. **Trappola reale già
    capitata**: `apple-touch-icon.png` era rimasto quello vecchio (design
    piatto pre-artwork, mai rigenerato quando l'artwork vera fu caricata),
    mentre le varianti maskable erano state correttamente rigenerate — la
    discrepanza è saltata fuori perché diverse superfici di condivisione
    link (share sheet di Android, anteprime in app di messaggistica)
    usano proprio `apple-touch-icon` come icona "di qualità" per un URL,
    mostrando quindi la versione vecchia e sgradevole invece
    dell'artwork attuale. Rigenerato compositando `icon-512.png` (RGBA)
    su sfondo `accent-solid` pieno e ridimensionando a 180×180 — stessa
    fonte delle altre varianti, nessun redesign separato. Verificare che
    tutte le varianti derivate (maskable, apple-touch-icon) corrispondano
    visivamente all'artwork corrente ogni volta che questa viene
    aggiornata, non solo quelle usate più visibilmente in-app.
  - **Icona maskable separata** (`icon-maskable-192.png`/
    `icon-maskable-512.png`, `"purpose": "maskable"` in `manifest.json`,
    entry **separata** dagli icon "any" sopra — mai `"purpose": "any
    maskable"` sulla stessa entry, Chrome lo segnala come problema in
    DevTools): su Android 8+ ogni app usa un'icona adattiva che richiede
    uno sfondo **opaco**; senza una variante maskable, Chrome/Android
    impacchetta l'icona "any" (trasparente) dentro un riquadro bianco per
    soddisfare comunque quel requisito — non è un bug nostro, è
    documentato su web.dev. La variante maskable risolve il riquadro
    bianco componendo l'artwork PNG sopra (non una ricostruzione
    vettoriale separata: sarebbe un'illustrazione diversa, non la stessa
    icona) su uno sfondo `accent-solid` (`#4f46e5`) pieno, scalata al 75%
    e centrata — percentuale calcolata dal bounding box reale dei pixel
    non trasparenti dell'artwork (non una stima), per stare dentro la
    "safe zone" circolare (~80% di diametro) che i launcher Android
    possono ritagliare. **Richiede necessariamente la rinuncia alla
    trasparenza per questa sola variante** (scelta esplicita dell'utente:
    preferita al riquadro bianco). Se si aggiorna l'artwork, ricalcolare
    lo scale factor dal nuovo bounding box invece di riusare 75% a
    memoria.
  - **Nota Firefox Android**: "Aggiungi a schermata Home" da Firefox può
    ignorare del tutto icona/nome del manifest e mostrare l'icona
    generica del robottino Android — bug noto e ricorrente di Firefox
    (Bugzilla #1440661, #1234558), non risolvibile lato nostro
    manifest/codice. Non provare a "correggerlo" di nuovo: non è un
    problema della nostra configurazione.
- **Calendario** (`src/pages/Calendar.jsx` + `src/lib/schedule.js`), griglia
  mensile (lun-dom, navigazione libera avanti/indietro):
  - Ogni serie ha un campo opzionale `watchDays` (giorni della settimana in
    cui si prevede di guardarla).
  - **Futuro (oggi incluso)**: le date mostrate **non sono mai salvate**, si
    ricalcolano ad ogni apertura della pagina da oggi + `watchDays` +
    numero di episodi non ancora visti rimanenti
    (`upcomingCalendarEntries`). La scansione include **oggi stesso**: una
    serie il cui giorno di visione è oggi ma il cui episodio non è ancora
    segnato visto deve comparire nella cella di oggi, non saltare
    all'occorrenza successiva (altrimenti una serie con giorno di visione
    solo la domenica, controllata proprio di domenica, sparirebbe per una
    settimana intera). Intenzionale: se si salta un giorno di visione
    previsto, la serie non sparisce dal calendario, lo slot si sposta
    semplicemente in avanti (il conteggio dipende dagli episodi rimasti,
    non da una data fissa). Non persistere le occorrenze calcolate né
    introdurre uno stato "saltato". Nota storica: una versione precedente
    escludeva deliberatamente oggi dalla proiezione ("comincia da domani") —
    l'utente ha poi corretto esplicitamente questo comportamento perché
    causava lo slittamento sopra descritto; questa è la versione valida.
  - **Passato/oggi**: mostra la cronologia reale di cosa è stato visto,
    presa dalla data effettiva salvata per ogni episodio (vedi sotto), non
    dal piano `watchDays`. La data di un episodio già visto è modificabile
    direttamente dal Calendario (non dalla pagina serie, che resta un
    semplice toggle segna/non-segna visto).
  - Navigazione tra i mesi: bottoni ← →, rotellina del mouse (desktop) e
    swipe orizzontale (touch), tutti e tre attivi contemporaneamente sulla
    stessa griglia — non rimuovere nessuno dei tre in favore degli altri.
    Ogni cambio mese rigioca un'animazione slide+fade direction-aware
    (classi `.animate-slide-next`/`.animate-slide-prev` in `index.css`,
    rigiocata rimontando la griglia con una `key` React legata a
    `${year}-${month}`). Lo scroll con rotellina ha un cooldown breve
    (`useRef`, non state React, 150ms) solo per evitare che un singolo gesto
    del trackpad salti più mesi insieme — non allungarlo: uno scroll rapido
    e deliberato deve poter cambiare mese rapidamente in sequenza.
- **`series.watched[SxEy]` è una data (`YYYY-MM-DD`), non `true`**: il
  giorno reale in cui l'episodio è stato segnato visto (vedi `dateKey` in
  `src/lib/schedule.js` — usa i componenti locali della data, MAI
  `toISOString()`, che converte a UTC e sfalsa il giorno vicino alla
  mezzanotte per chi non è in UTC). Tutto il codice che verifica se un
  episodio è visto deve continuare a controllare la sola verità del valore
  (`Boolean(watched[key])`), mai assumere che sia booleano.

## Stile — palette e principi

Stile minimal. La palette è definita **in un solo posto**,
`src/index.css` (`@theme` + override in `@media (prefers-color-scheme: dark)`),
come variabili CSS/token Tailwind semantici:

| Token              | Uso                                  |
|--------------------|---------------------------------------|
| `bg` / `surface`   | sfondo pagina / sfondo card-modali    |
| `text` / `muted`   | testo principale / testo secondario   |
| `border`           | bordi sottili                         |
| `accent`           | testo/link/badge d'accento (adatta al tema) |
| `accent-solid`     | sfondo bottoni pieni (**costante** tra i temi, sempre abbinato a `text-white`) |
| `success` / `danger` | stati positivi/negativi (badge "Completata", elimina, ecc.) |

Segue automaticamente lo schema chiaro/scuro del sistema operativo
dell'utente (`prefers-color-scheme`), nessun toggle manuale da mantenere.

## Style guide di riferimento — `design/style-guide.html`

Esiste una pagina HTML statica e autonoma, `design/style-guide.html`, che
elenca **ogni** token colore (nome variabile CSS + hex, in chiaro e scuro,
copiabile al click), e i pattern/classi di ogni componente (card, modale,
badge, bottoni, tabs, progress bar, episodi...) effettivamente usati
nell'app. Apribile direttamente nel browser, non fa parte della build React.

**Perché esiste**: permette di riferirsi a elementi/colori per nome (es.
"il bottone accent-solid", "il colore success-soft") invece che dover
descrivere tutto da zero ogni volta, ed evita la deriva tra "come pensiamo
sia fatta l'UI" e "come è fatta davvero".

**Va tenuta aggiornata SEMPRE**: ogni volta che si aggiunge/modifica un
componente, una classe, un colore o un token, aggiornare
`design/style-guide.html` nello stesso commit. Una style guide disallineata
dal codice reale è peggio che non averla.

## Regola di coerenza (esplicitamente richiesta)

**Stile e funzionalità devono essere coerenti in tutta la webapp.** In pratica:

- Usare sempre le utility semantiche sopra (`bg-surface`, `text-muted`,
  `bg-accent-solid`, ecc.), mai colori Tailwind grezzi (`bg-indigo-500`,
  `text-gray-400`...) o colori hardcoded in inline style.
- Riutilizzare i componenti condivisi (`Modal`, `ProgressBar`, `StatusBadge`,
  `SeriesCard`, `StatusTabs`, `EmptyState`) invece di reimplementarli con
  markup/stile leggermente diverso in una pagina specifica.
- Stesso raggio di bordo (`rounded-xl`/`rounded-2xl` per card e modali,
  `rounded-full` per pillole/badge/bottoni di stato) ovunque.
- La logica di progresso/episodio successivo/stato vive **solo** in
  `src/lib/progress.js` — non duplicare questi calcoli in un componente.
- Ogni azione utente disponibile in un punto dell'app (es. "Aggiungi serie")
  deve comportarsi allo stesso modo ovunque sia raggiungibile (bottone
  nell'header, sempre visibile, stesso modale).
- Prima di aggiungere una nuova pagina/componente, controllare se esiste già
  un pattern equivalente da riusare invece di introdurne uno nuovo.

## Regole generali di sviluppo

- Non introdurre un database o backend "vero" (Postgres, MySQL, server
  custom...) a meno che l'utente non lo richieda esplicitamente: la scelta
  di Firestore come backend gestito e senza manutenzione è deliberata.
- Non aggiungere login/autenticazione a meno che l'utente non lo richieda
  esplicitamente: il flusso "nessun login, dati condivisi" è voluto.
- Niente funzionalità speculative non richieste (feature flags, ruoli
  utente, notifiche push, offline-first/service worker...): tenere lo scope
  minimo e coerente con quanto discusso.
- Prima di un deploy reale, verificare che `npm run build` passi e che le
  env var richieste siano documentate in `README.md`.
- **Lo stato "Completata" (o "In attesa di nuova stagione") viene impostato
  automaticamente** non appena l'ultimo episodio rimasto viene segnato
  visto (vedi `autoStatusUpdates` in `VaultContext.jsx`, usato sia da
  `toggleEpisode` che da `setSeasonWatched`). Quale dei due stati dipende
  dal flag `series.ongoing` (impostato da TMDB in base al campo `status`
  della serie — "Ended"/"Canceled" → `false`, tutto il resto → `true`, vedi
  `isOngoing` in `tmdb.js`): se la serie è ancora in corso di rinnovo
  diventa "In attesa di nuova stagione", altrimenti "Completata". Le serie
  manuali non hanno `ongoing` (nessuna fonte TMDB) e vanno sempre su
  "Completata", comportamento invariato. Non c'è auto-revert quando si
  smarca un episodio dopo il completamento: lo stato resta quello
  automatico finché non viene cambiato manualmente (o finché un refresh
  TMDB non rivela episodi nuovi, vedi sopra). Nota storica: una versione
  precedente di questa regola diceva l'opposto (mai automatico) — quella
  era un'incomprensione, corretta esplicitamente dall'utente; questa è la
  versione valida.
- **La valutazione è doppia**: un cuore blu 💙 e un cuore viola 💜, una
  valutazione indipendente a testa (i due membri della coppia), non un
  singolo numero condiviso. Ogni cuore accetta un decimale da 1 a 10 con
  **fino a due cifre decimali** (`step="0.01"`, es. 7.88). I due cuori come
  emoji sono una scelta esplicita dell'utente per questo punto preciso —
  non riusarli altrove nell'app (vedi regola no-emoji in cima al file).
  - **Valutazione per episodio** (`series.episodeRatings[SxEy] = { blue?,
    purple? }`, `setEpisodeRating` in `VaultContext.jsx`): votabile appena
    l'episodio è segnato visto, **indipendentemente dallo stato della
    serie** (non solo a "Completata" — scelta esplicita dell'utente, per
    poter votare mentre si guarda). Righe per-episodio in `SeasonBlock`
    (`SeriesDetail.jsx`), stesso componente `HeartRating` riusato sia lì che
    per il voto totale.
  - **Voto totale calcolato automaticamente**: `series.ratingBlue` /
    `ratingPurple` sono la **media dei voti per episodio di quella persona**
    (`aggregateHeartRating` in `progress.js`), ricalcolata e riscritta ad
    ogni voto-episodio — **il calcolo automatico vince sempre** (scelta
    esplicita dell'utente): se esiste anche un solo voto per episodio per
    quel cuore, `VaultContext.setEpisodeRating` sovrascrive il totale con
    quel calcolo ad ogni cambio. La modifica manuale del totale
    (`setRating`, stesso `HeartRating`) resta **permessa nell'interfaccia**
    finché non tutti gli episodi hanno un voto per quel cuore
    (`heartFullyRated` in `progress.js`) — non "finché non ne esiste
    nessuno": un utente che aggiunge una serie già vista (niente voti per
    episodio possibili, es. vista settimane fa) deve poter impostare un
    voto totale a mano e vederlo restare, e lo stesso vale mentre i voti
    per episodio sono solo parziali. Il bottone "Modifica"/"+ Aggiungi"
    del totale **sparisce del tutto** (non solo disabilitato) solo quando
    **ogni** episodio ha già un voto per quel cuore — a quel punto nessun
    futuro voto-episodio scatterà mai più un ricalcolo (non restano
    episodi da votare), quindi un valore digitato a mano lì rimarrebbe per
    sempre come override permanente invece di perdere sempre contro il
    calcolo, il vero motivo per bloccare la modifica solo in quel caso
    preciso e non prima.
  - Il "voto finale" mostrato (card e pagina serie) resta la **media dei
    due totali** (`averageRating` in `progress.js`, invariato): se uno dei
    due non ha ancora un totale, la media mostrata è semplicemente quello
    presente. `formatRating()` arrotonda a 2 decimali e toglie gli zeri
    finali per la visualizzazione (8 invece di 8.00, 7.5 invece di 7.50).
    **L'ordinamento "Valutazione" in `Home.jsx` usa questo stesso
    `averageRating(series)`** (con `?? -Infinity` per le serie senza voto,
    così finiscono sempre tutte in fondo, mai mischiate tra quelle votate)
    — bug reale corretto: leggeva ancora il vecchio campo `series.rating`,
    rimosso quando il voto singolo è stato sostituito dai due cuori, quindi
    era sempre `undefined` e l'ordinamento non funzionava più.
  - Il **voto totale** (riga "Valutazione: X/10" + i due `HeartRating` in
    cima alla pagina serie, `RatingRow` in `SeriesDetail.jsx`) resta
    visibile/modificabile **solo** quando lo stato della serie è
    "Completata" (non anche "In attesa di nuova stagione") **e** non esiste
    già un grafico voti per episodio (`chartData.length === 0`, prop
    `hasChart` passata a `RatingRow`). Appena esiste almeno un voto per
    episodio il grafico compare (vedi sotto) e questa riga sparisce del
    tutto — la legenda sempre visibile sopra il grafico mostra già gli
    stessi identici valori (💙/💜/Media), quindi tenerla sarebbe
    duplicazione pura. Resta l'**unica** UI di visualizzazione/modifica del
    voto totale per una serie "Completata" senza nessun voto per episodio
    (es. serie aggiunta già vista, niente da votare episodio per episodio):
    lì niente grafico esiste per cui fare da fonte alternativa, quindi non
    va nascosta. Il voto **per episodio**, invece, non ha nessuno di questi
    vincoli (vedi sopra). Nessuno dei due valori viene cancellato se lo
    stato cambia successivamente o se compaiono voti per episodio (stesso
    comportamento non distruttivo del link) — solo la riga smette di essere
    mostrata, il dato resta e ricompare se la condizione torna vera (es. si
    rimuove l'ultimo voto per episodio).
  - **Grafico voti per episodio** (`src/components/RatingChart.jsx`, al
    posto del voto totale in `SeriesDetail.jsx` quando presente, solo se
    almeno un episodio ha un
    voto): grafico a linee responsive, senza libreria esterna (SVG
    disegnato a mano con `viewBox`). **Larghezza reattiva ma unità sempre
    fisse**: il contenitore (`overflow-x-auto`) viene misurato con
    `ResizeObserver` e quel valore è il **pavimento** della larghezza del
    grafico (`Math.max(280, containerWidth, neededWidth)`, dove
    `neededWidth` dipende da quanti punti ci sono e dallo zoom) — il
    grafico riempie sempre lo spazio disponibile nella card invece di stare
    sempre alla sua larghezza minima "clippato" in un angolo, e cresce oltre
    il contenitore (attivando lo scroll orizzontale) solo quando i punti
    (o lo zoom) hanno davvero bisogno di più spazio di quanto offerto.
    Fondamentale: l'SVG non usa **mai** `width: 100%` — il suo `style`
    (`width`, `height: HEIGHT`) combacia sempre esattamente col `viewBox`,
    quindi 1 unità SVG = 1px reale **sempre**, a prescindere da quanto è
    largo il contenitore o quanti punti ci sono. Bug reale corretto: prima
    di questa architettura, `width: 100%` riscalava l'intero sistema di
    coordinate (font-size delle etichette, spessore linee, raggio dei
    pallini — tutti espressi in unità SVG "fisse") per riempire il
    contenitore, quindi una serie con pochi episodi votati produceva
    etichette e pallini enormi e sfocati su una card desktop larga. Un
    primo tentativo di correzione fissava la larghezza SOLO al minimo
    necessario per i punti (niente più elementi giganti, ma il grafico
    restava piccolo/clippato anche quando la card aveva ovviamente più
    spazio) — la versione corretta doveva essere reattiva **e** con unità
    fisse insieme, non l'una o l'altra. Un punto per episodio votato con tre linee — 💙,
    💜 e una linea tratteggiata "Media" in grigio (`--color-muted`) perché
    è un valore derivato, non un terzo voto. Sopra al grafico, una riga
    sempre visibile (non solo in hover) mostra i totali correnti —
    "💙 X/10", "💜 X/10", "Media X/10" — passati come props
    (`totalBlue`/`totalPurple`/`totalAverage`) calcolati dal chiamante con
    gli helper di `progress.js`, mai ricalcolati dentro il componente.
    **Ogni voce della legenda è cliccabile** per nascondere/mostrare quella
    linea (utile per isolare il voto di una sola persona senza che l'altra
    linea ci passi sopra) — `opacity-40` sulla legenda quando nascosta,
    linea e pallini rimossi dal grafico, e il valore corrispondente
    escluso anche dal tooltip hover (non solo dal grafico), così la "vista
    tabellare" del tooltip resta coerente con cosa è davvero visibile.
    Crosshair + tooltip al passaggio del mouse/tocco (valori sempre
    visibili anche senza hover nelle righe per-episodio sopra, che fanno da
    "vista tabellare"). **Supporto touch**: oltre a `onPointerDown`/
    `onPointerMove` (mouse/pen), gestori dedicati `onTouchStart`/
    `onTouchMove` che leggono `e.touches`/`e.changedTouches` direttamente —
    verificato che affidarsi ai soli eventi pointer non basta: un tocco
    reale genera comunque un evento nativo `pointerdown` con
    `pointerType: "touch"`, ma senza un handler touch dedicato il
    sollevamento del dito genera anche un `pointerleave` che cancella
    subito il tooltip appena mostrato. Per questo `onPointerLeave` ignora
    esplicitamente `pointerType === 'touch'` (solo un vero mouse che esce
    dall'area deve nascondere il tooltip) — senza questo controllo il
    tooltip touch lampeggia e sparisce invece di restare visibile fino al
    tocco successivo. **Nessuna restrizione `touch-action` sull'SVG** (una
    versione precedente impostava `touch-action: pan-y` per lasciare
    intatto lo scroll verticale della pagina durante l'interazione) — bug
    reale corretto: `pan-y` dice al browser di trattare uno swipe
    orizzontale come "gestito da script", il che impediva anche allo
    scroll orizzontale nativo del contenitore `overflow-x-auto` di
    funzionare, bloccando di fatto la visione di episodi oltre quelli
    visibili nella larghezza iniziale su schermi stretti.
  - **Zoom**: pinch a due dita su mobile, rotellina del mouse su desktop —
    non più bottoni "−"/"+" su schermo, rimossi su richiesta esplicita
    dell'utente (competevano con la legenda per lo spazio su mobile e
    avevano causato proprio loro il bug "la legenda va a capo" descritto
    sopra, spostandolo invece di risolverlo). `zoom` da 0.6× a 2× in passi
    di 0.2, moltiplica `MIN_POINT_SPACING`: utile tanto su mobile (episodi
    ravvicinati, difficili da toccare singolarmente) quanto su desktop
    (serie con molti episodi, vedere più punti insieme riducendo lo zoom).
    **Listener nativi, non le prop JSX `onWheel`/`onTouchStart`/
    `onTouchMove`/`onTouchEnd`** (`svg.addEventListener(..., { passive:
    false })` dentro un `useEffect`, richiamato quando cambiano
    `zoom`/`data`/dimensioni): bug reale scoperto testando — React registra
    i listener `wheel`/`touch*` come passivi di default, quindi
    `e.preventDefault()` chiamato da una prop JSX come `onWheel` viene
    **ignorato silenziosamente** (in console: "Unable to preventDefault
    inside passive event listener invocation", non un errore che blocca
    l'esecuzione). Risultato osservato: la rotellina cambiava lo zoom MA
    il browser scrollava comunque nativamente il grafico in orizzontale
    sotto al puntatore fermo — al tick di rotellina successivo il cursore
    non era più sopra l'SVG (ora scrollato altrove) e ogni evento
    successivo veniva perso in silenzio, quindi si poteva zoomare "in" ma
    mai più tornare indietro. Un `addEventListener` esplicito con
    `passive: false` è l'unico modo per bloccare davvero il comportamento
    di default del browser in questo caso; lo stesso vale per il touch a
    due dita (altrimenti lo zoom nativo della pagina interferirebbe col
    pinch). Il singolo dito (crosshair, scroll) resta come prima — non
    chiama mai `preventDefault`, quindi lo scroll nativo continua a
    funzionare in entrambe le direzioni. **Filtri** (solo se la
    serie ha più di una stagione, gestiti in `RatingChartSection` dentro
    `SeriesDetail.jsx`, non dentro `RatingChart.jsx` che resta un
    componente "dumb" che disegna solo i dati ricevuti): "Tutti gli
    episodi" (default, cronologico su tutte le stagioni), "Per stagione"
    (`seasonRatingChartData` in `progress.js`), "Confronta stesso episodio
    tra stagioni" (`episodeAcrossSeasonsChartData`, es. ogni "episodio 2"
    di ogni stagione messo a confronto, punti sull'asse X etichettati per
    stagione — S1, S2, ... — invece che per episodio; stagioni troppo
    corte per raggiungere quel numero di episodio, o senza voto per
    quell'episodio, semplicemente escluse, stessa regola "escludi non
    stimare" di `remainingMinutes`). Il picker delle stagioni/episodi
    disponibili cambia lo zoom/hover: `RatingChart` riceve una `key`
    (modalità+stagione+episodio) che lo rimonta ad ogni cambio di vista,
    così lo zoom non resta "appiccicato" da una vista completamente
    diversa. **Colori**:
    `--chart-blue`/`--chart-purple` in
    `index.css`, deliberatamente separati dalla palette semantica dell'app
    (vedi tabella sotto) perché qui il colore deve portare *identità*
    (persona A vs persona B), non stato — validati con lo script
    `validate_palette.js` della skill dataviz per separazione CVD e
    contrasto contro `bg-surface`, sia chiaro che scuro. Non toccare questi
    due valori senza rivalidarli.
- **Serie solo-viste da una persona sola** (`series.viewer`, opzionale —
  assente = condivisa/entrambi, comportamento invariato per ogni serie
  precedente a questo campo): non ogni serie viene guardata da entrambi, es.
  una serie che interessa solo a 💙. `viewer: 'blue' | 'purple'` marca quella
  serie come solo-vista da quella persona — l'altro cuore non è "vuoto", è
  proprio **non applicabile**, quindi ogni superficie di valutazione lo
  nasconde del tutto invece di mostrarlo vuoto/disabilitato:
  - **`RatingRow`** (voto totale): l'`HeartRating` dell'altro cuore non
    viene renderizzato.
  - **`SeasonBlock`** (voto per episodio): stessa cosa, riga per riga.
  - **`RatingChart`** (prop `viewer`, passata da `RatingChartSection` via
    `soloViewer()` in `progress.js`): la legenda/linea/tooltip dell'altro
    cuore sparisce, e la linea "Media" sparisce **anch'essa del tutto** (non
    solo se nascosta a mano) — con un solo votante la media coincide sempre
    con quell'unico valore, quindi mostrarla sarebbe la stessa
    duplicazione pura già risolta sopra per "Valutazione: X/10".
  - **Impostabile/modificabile in qualsiasi momento** (`ViewerPicker` in
    `src/components/ViewerPicker.jsx`, condiviso tra `AddSeriesModal` — scelta
    all'aggiunta, default "Entrambi" — e `SeriesDetail` — modificabile dopo,
    riga sotto le pillole di stato, `setViewer` in `VaultContext.jsx`).
    **Non distruttivo**, stesso principio di link/Wikipedia/visibilità voto:
    passare da condivisa a solo (o viceversa) non cancella mai
    `ratingBlue`/`ratingPurple`/`episodeRatings` dell'altro cuore, si limita
    a nascondere/mostrare i controlli. Caso concreto verificato: una serie
    guardata da 💙 da sola per i primi episodi, poi diventata condivisa a
    metà stagione quando 💜 si aggiunge — non serve nessuna logica ad-hoc,
    perché `aggregateHeartRating()` già media ciascun cuore
    **indipendentemente** sui soli episodi che quel cuore ha votato: il
    totale di 💙 resta la media dei suoi voti (magari su tutti gli episodi),
    quello di 💜 la media dei suoi (magari solo da metà stagione in poi),
    senza bisogno che 💜 recuperi voti sugli episodi visti solo da 💙.
  - **Filtro in Libreria, nell'header** (`ViewerHeaderFilter` in
    `Header.jsx`, non in `Home.jsx`): due bottoni icona 💙/💜 nell'header,
    quindi raggiungibili da **ogni** pagina (l'header sta fuori da
    `<Routes>` in `App.jsx`, non solo su `Home`) — cliccarli naviga a `/`
    con `?viewer=blue|purple` impostato nell'URL. Una prima versione li
    metteva come terza riga di pillole (con una "Tutte" esplicita) sotto
    stato+ordinamento in `Home.jsx`, stesso componente `PillTabs` della riga
    di stato — spostata nell'header su richiesta esplicita dell'utente per
    renderla raggiungibile ovunque, non solo dalla Libreria. Senza spazio
    per una terza pillola "Tutte" accanto alle due icone, **cliccare
    l'icona già attiva la disattiva** (torna al default, vedi sotto) invece
    di avere un bottone dedicato — stesso risultato, gesto diverso. Da una
    pagina diversa da `/` il click naviga comunque a `/?viewer=...` (nessuno
    stato attivo da disattivare, quindi solo "attiva"); da `/` preserva gli
    altri parametri (`status`/`sort`) già in URL. Lettura del filtro
    (`?viewer=blue|purple`, stesso pattern di fallback su valore
    sconosciuto/assente di `?status=`/`?sort=`, chiave `VIEWER_KEYS`) resta
    in `Home.jsx`, che applica il filtro alla griglia — solo la scrittura
    (i due bottoni) si è spostata. **Il default (nessun 💙/💜 attivo) mostra
    solo le condivise, non tutte le serie mischiate insieme** — bug reale
    corretto: la prima versione trattava "nessun filtro" come "ogni serie a
    prescindere dal viewer" (comportamento ereditato da prima che
    esistessero le serie solo, quando ogni serie era per forza condivisa),
    il che faceva comparire le serie solo-💙/solo-💜 anche nella vista
    principale della libreria invece che solo dietro il filtro esplicito.
    La vista di default della Libreria è la libreria condivisa; le solo
    sono una vista a parte, mai mescolata. "Solo 💙" mostra **solo** le
    serie marcate `viewer: 'blue'`, non anche quelle condivise (scelta
    esplicita, non "tutto ciò che guardo io incluse le condivise") — le tre
    viste (default/condivise, solo-💙, solo-💜) sono quindi reciprocamente
    esclusive, senza sovrapposizioni. La sezione "Da vedere oggi" in cima a
    `Home.jsx` **non** è scoped da questo filtro (continua a includere le
    solo), stesso motivo per cui non lo è già da stato/ordinamento (vedi
    sopra: è un richiamo fisso, non una vista filtrata).
- **Durata episodi e tempo rimanente**:
  - `series.episodeDurations[SxEy]` (minuti). Per le serie TMDB, scaricata
    automaticamente — `getEpisodeDurations` in `tmdb.js` fa una chiamata
    per stagione (`/tv/{id}/season/{n}`, non inclusa nella chiamata
    principale usata per titolo/poster/stagioni) sia all'aggiunta sia ad
    ogni "Aggiorna da TMDB"; gli episodi non ancora andati in onda
    (`runtime` nullo su TMDB) restano senza durata finché non lo
    diventano. Per le serie manuali non esiste nessuna fonte automatica:
    la durata si inserisce a mano per episodio (`setEpisodeDuration` in
    `VaultContext.jsx`), disponibile **anche prima di segnare l'episodio
    visto** (a differenza del voto, che richiede l'episodio già visto) —
    serve a poter calcolare il tempo rimanente anche su episodi non ancora
    guardati.
  - **Tempo rimanente** (`remainingMinutes`/`formatDuration` in
    `progress.js`): somma delle durate degli episodi **non visti** di cui
    si conosce la durata — un episodio senza durata nota viene
    semplicemente escluso dalla somma (mai contato come 0), quindi il
    totale può sottostimare ma mai sovrastimare. Ricalcolato dal vivo ad
    ogni render (mai salvato), esattamente come `progressRatio`. Mostrato
    in due punti: il totale dell'intera serie accanto a "X/Y episodi
    visti" nell'header, e uno scoped per singola stagione accanto
    all'intestazione "Stagione N" — entrambi nascosti quando il valore è 0
    (che sia perché non resta nulla da vedere o perché non si conosce
    nessuna durata: in entrambi i casi nascondere è la scelta giusta, non
    c'è bisogno di distinguerli).
  - Riga per-episodio in `SeasonBlock`: mostra la durata (testo semplice
    per le serie TMDB, editabile con lo stesso pattern edit/display/add
    di `LinkRow` per le manuali) affiancata ai voti a cuore quando
    l'episodio è visto. La riga compare per ogni episodio visto (voto) o,
    per le serie manuali, per **ogni** episodio indipendentemente dal
    visto (durata sempre editabile); per le serie TMDB un episodio non
    visto compare solo se la sua durata è già nota.
- **`StatusTabs` su mobile** scorre orizzontalmente (`overflow-x-auto` +
  classe di utilità `.no-scrollbar` in `index.css` per nascondere la
  scrollbar senza disabilitare lo scroll, bottoni con `shrink-0`) invece di
  andare a capo su due righe: andare a capo spingerebbe in basso la griglia
  delle serie sotto. Ogni riga di filtri/ordinamento che rischia di non
  entrare su schermi stretti deve seguire lo stesso pattern (scroll, non
  wrap), non introdurne di nuovi.
- **Filtro di stato e ordinamento in `Home.jsx` vivono nell'URL**
  (`?status=...&sort=...`, dopo l'hash di `HashRouter`, tramite
  `useSearchParams`) invece che in `useState` locale: così una vista
  specifica (es. "In corso" ordinato per Valutazione) è salvabile nei
  preferiti/condivisibile e si ripresenta identica alla riapertura del
  link. Un valore mancante o sconosciuto nell'URL (parametro assente,
  digitato a mano, o non più valido) ricade sul default (`all`/`updated`)
  invece di mostrare una libreria vuota o rompere l'ordinamento — i valori
  ammessi sono verificati contro le chiavi note (`STATUS_KEYS`/
  `SORT_KEYS` in `Home.jsx`) prima dell'uso.
- **"Da vedere oggi" (`Home.jsx`)**: sezione in cima alla Libreria, sopra i
  filtri di stato, che riusa `upcomingCalendarEntries` (`schedule.js` — la
  stessa proiezione su cui è costruito il Calendario) filtrata alla sola
  data di oggi, mostrando ogni serie con un episodio non ancora visto
  previsto oggi. **Sempre visibile indipendentemente dal filtro di
  stato/ordinamento** scelto sotto: non è una vista filtrata della
  libreria, è un richiamo fisso "cosa guardare oggi" — nascosta del tutto
  quando non c'è nessuna serie in programma oggi, stesso principio
  "nascondi se vuoto" usato altrove (tempo rimanente, sezione voto).
  - **Righe in stile lista (`TodayListItem`), non `SeriesCard`**: una
    piccola miniatura (`h-11 w-11`, non il poster intero) invece della
    card verticale piena — scelta esplicita dell'utente perché questa
    sezione siede sopra un'intera pagina di card e non deve competere
    visivamente con quella griglia allo stesso modo di una seconda
    griglia di card. Tutte le informazioni della card sono comunque
    presenti, solo re-impaginate su due righe per riga di lista invece di
    scomparire: riga 1 = titolo (troncato) + bottone link esterno 🔗 (se
    presente, stesso comportamento/icona di `SeriesCard`) + `StatusBadge`;
    riga 2 = `ProgressBar` + conteggio "X/Y" + bottone rapido "Segna
    SxEy". Componente locale in `Home.jsx` (non un file separato: uso
    unico, solo in questa sezione), che riusa `ProgressBar`/`StatusBadge`
    condivisi invece di reinventarli.
- **Layout `SeriesDetail.jsx`**: titolo, pillole di stato, barra di
  progresso (+ episodi visti/tempo rimanente), link, Wikipedia restano
  nella colonna stretta accanto al poster (l'header "di identità" della
  serie) — sono compatti e ci stanno bene. "Condividi", "Aggiorna da TMDB"
  ed "Elimina serie" sono bottoni icona nella **stessa riga del titolo**,
  allineati a destra in quest'ordine (dal meno al più distruttivo — non
  righe di testo separate: con azioni icona ci sta bene condividere la
  riga con l'`<h1>`) — tutti `h-8 w-8` con `flex items-center
  justify-center`, così l'area cliccabile è identica a prescindere
  dall'icona. Tutte e tre usano `ShareIcon.jsx`/`RefreshIcon.jsx`/
  `CloseIcon.jsx` (stessa famiglia di icone SVG, vedi sopra) — mai testo
  Unicode. Tutti e tre hanno stato hover (`hover:bg-surface-hover`
  + colore, `hover:text-accent`/`hover:text-danger`) e stato `active` uguale
  allo hover, perché su mobile l'hover non scatta mai (nessun puntatore) e
  senza un `active` esplicito il tocco non darebbe nessun feedback visivo.
  - **Condividi**: `navigator.share({ title, url })` con l'URL della
    pagina corrente (deep link diretto alla serie, grazie a `HashRouter`)
    quando l'API Web Share è disponibile (mobile, alcuni browser
    desktop) — apre lo share sheet nativo del sistema operativo. Se non
    disponibile (la maggior parte dei browser desktop), fallback a
    `navigator.clipboard.writeText(url)` con una riga di conferma
    temporanea ("Link copiato negli appunti", 2s, stesso punto sotto il
    titolo dell'eventuale errore di refresh) — o un messaggio di errore se
    anche la clipboard fallisce (permessi negati, contesto non sicuro).
    Non condivide altro (né l'intera libreria né un riassunto testuale):
    solo il link diretto alla serie aperta.
  "Giorni di visione" e
  "Valutazione + grafico" sono invece sezioni proprie a piena larghezza
  sotto l'header (card `rounded-2xl border border-border bg-surface p-4`),
  non più schiacciate nella colonna stretta: sono le due sezioni "pesanti"
  (7 pillole, grafico responsive) che avevano davvero bisogno di tutta la
  larghezza pagina. Non spostare di nuovo tutto in un'unica colonna
  stretta accanto al poster: è il motivo per cui la pagina risultava
  compressa e disordinata prima di questa modifica.
  - **Sotto `sm`, poster e colonna info si impilano verticalmente**
    (`flex-col items-center` → `sm:flex-row sm:items-start`), poster
    centrato sopra, colonna info a piena larghezza sotto — non più sempre
    affiancati. Con caratteri/accessibilità a dimensione maggiore del
    default, tenerli sempre affiancati costringeva ogni riga (link,
    Wikipedia, pillole di stato) ad andare a capo dentro una colonna
    già stretta, con il risultato di spaziature enormi e disordinate.
    Impilare verticalmente su mobile è la correzione robusta: non dipende
    da quanto è grande il font dell'utente.
  - **Eliminazione serie**: conferma tramite modale (`ConfirmDeleteModal`,
    riusa `Modal`) — mai `window.confirm()` nativo, che non segue lo stile
    dell'app ed è più rischioso da confermare per errore con un bottone
    icona piccolo.
  - **"Segna non vista" di un'intera stagione** (`SeasonBlock`) chiede
    conferma tramite modale (stesso pattern di `ConfirmDeleteModal`,
    riusa `Modal`) prima di eseguire — a differenza di "Segna tutta
    vista" (che aggiunge solo dati, nessuna conferma necessaria), smarcare
    un'intera stagione **cancella le date di visione registrate** per
    ogni episodio; se poi vengono ri-segnati visti, la data sarà quella
    del giorno corrente, non quella originale — una perdita di cronologia
    reale, non solo uno stato che si può ripristinare con un click,
    quindi merita lo stesso livello di attenzione della cancellazione
    serie.
  - **Giorni di visione, sezione intera nascosta** quando lo stato è
    "Completata", "In attesa di nuova stagione" o "Abbandonata"
    (`showWatchDays` in `SeriesDetail.jsx`): in questi tre stati non c'è
    più nulla da pianificare (niente episodi futuri da guardare, o non
    ancora usciti), quindi la sezione toglierebbe solo spazio a
    episodi/valutazione senza aggiungere nulla — non distruttivo,
    `series.watchDays` resta salvato e riappare modificabile se lo stato
    torna "In corso"/"Da vedere".
  - **Giorni di visione (`WatchDaysRow`)**: `grid grid-cols-7`, **stessa
    griglia a tutte le larghezze** (mobile e desktop), non solo sotto
    `sm` — una versione precedente usava `grid-cols-7` solo su mobile e
    tornava a `flex flex-wrap` con bottoni a larghezza fissa (`sm:w-28`)
    da `sm` in su, assumendo che lì lo spazio non mancasse mai; bug reale
    segnalato dall'utente controllando la pagina in modalità "sito
    desktop" da telefono, dove il breakpoint `sm:` scattava ma il
    contenitore reale (`max-w-3xl`) restava più stretto di quanto servisse
    per 7 pillole da `sm:w-28` + gap, quindi andavano comunque a capo su
    due righe — la stessa identica classe di bug della versione mobile,
    solo spostata su un altro breakpoint. Usare `grid-cols-7` **ovunque**
    (nessuna eccezione per-breakpoint) elimina la classe di bug alla
    radice: la griglia divide sempre la larghezza **effettiva** del
    contenitore in 7 frazioni uguali, quindi non esiste più una larghezza
    di viewport per cui possa silenziosamente tornare a wrappare.
    Etichetta comunque dipendente dal
    breakpoint (`WEEKDAYS` in `schedule.js` ha `shortLabel`/`fullLabel`
    oltre a `label`): su mobile una sola lettera/due
    (`shortLabel`: L, Ma, Me, G, V, S, D); da `sm` in su il nome completo
    del giorno (`fullLabel`: Lunedì, Martedì, ...) — questo cambia solo il
    testo dentro la pillola, mai la larghezza della colonna che lo
    contiene. Il
    `label` a 3 lettere (Lun, Mar, ...) resta invariato ed è usato **solo**
    dall'intestazione a griglia fissa a 7 colonne del Calendario, non da
    questi bottoni.
- **Ricerca serie**: due varianti per piattaforma, stessa logica di
  filtro condivisa (`filterSeriesByTitle` in `src/lib/search.js`, semplice
  substring case-insensitive sul titolo).
  - **Desktop** (`src/components/HeaderSearch.jsx`, nascosta sotto `sm`):
    campo di ricerca inline nell'header con dropdown risultati (poster +
    titolo, clic naviga alla serie). **Sempre attivo**, anche sulla pagina
    dettaglio di una serie (nota storica: una versione precedente lo
    disabilitava lì, corretta esplicitamente dall'utente — saltare da una
    serie a un'altra tramite ricerca è un uso legittimo). Scorciatoia da
    tastiera **`/`** per mettere a fuoco il campo (ignorata se si sta già
    scrivendo in un altro input/textarea/contenteditable); `Escape` svuota
    e toglie il focus, `Enter` seleziona il primo risultato.
  - **Mobile** (`src/components/MobileSearchModal.jsx` + pulsante flottante
    in `src/App.jsx` — non `src/pages/Home.jsx`: vive nello shell globale
    apposta per restare visibile su **ogni** pagina (Libreria, Calendario,
    dettaglio serie), non solo sulla Libreria come una versione precedente —
    `fixed bottom-5 right-5`, icona `SearchIcon.jsx` — SVG disegnata a mano,
    non emoji: non esiste un buon simbolo Unicode non-emoji per una lente
    d'ingrandimento): apre una modale con campo di ricerca + stessa lista
    risultati.
  - Entrambe le varianti hanno un bottone "cancella" (✕) dentro il campo,
    visibile solo quando c'è testo.
- **Modali e tasto Back**: ogni modale condivide `src/components/Modal.jsx`,
  che alla apertura fa un `history.pushState` e tratta un successivo
  popstate (tasto Back del browser/Android, incluso il gesto swipe-back)
  come una chiusura — così Back chiude la modale invece di navigare via
  dalla pagina sotto o uscire dalla PWA. Fixato una volta sola qui, vale per
  tutte (ricerca mobile, aggiungi serie, conferma eliminazione, ecc.), non
  duplicare per singola modale. **Trappola nota (bug reale già capitato)**:
  la versione precedente chiamava `history.back()` incondizionatamente
  dentro il cleanup dell'effect quando la chiusura non veniva da Back (X,
  click sull'overlay, Escape) — ma `history.back()` è asincrono (il
  popstate risultante arriva in un task successivo), e React StrictMode in
  sviluppo fa mount→cleanup→mount **sincrono** dello stesso effect: il
  cleanup "fantasma" del primo mount chiamava `history.back()`, il cui
  popstate arrivava poi in ritardo mentre il secondo mount (quello vero)
  aveva già il suo listener attivo — quel listener lo scambiava per un vero
  Back e chiudeva la modale un istante dopo l'apertura, prima ancora che
  fosse visibile. Fix: **mai** chiamare `history.back()` da un cleanup.
  Ogni chiusura non-Back (X, overlay, Escape) passa invece per
  `closeViaHistory()`, che chiama esso stesso `history.back()` in risposta
  diretta al click/tasto — il popstate risultante è l'**unico** posto che
  chiama `onClose`; il cleanup dell'effect si limita a rimuovere il
  listener, senza nessun effetto collaterale da cui possa nascere la stessa
  race. Un ref (`pushedRef`, non variabile locale nella closure) traccia se
  l'entry è già stata pushata, per restare corretto anche sotto il
  doppio mount di StrictMode. **Nota**: il dropdown di `HeaderSearch.jsx`
  (desktop) non passa da questo meccanismo — non è una modale a schermo
  intero, si chiude con `Escape`/click fuori/selezione risultato, non con
  Back.
- **PWA, tasto back Android**: mitigazione nota per il frame vuoto grigio che
  Android può mostrare per un istante sulla home page prima che un secondo
  back chiuda l'app (vedi `useEffect` in `App.jsx`: se
  `display-mode: standalone`, un `history.pushState` all'avvio imbottisce
  lo stack di history con una voce in più, così il primo back consuma
  quella invece di uscire a vuoto). È un mitigamento, non una garanzia: il
  frame grigio può essere un artefatto di rendering di Android/Chrome
  durante lo smontaggio della WebAPK, non completamente risolvibile da puro
  JS lato client senza un wrapper nativo.
