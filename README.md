# ScatoloneQuintet

Repository di metadati per il Cube "Scatolone" (Magic: The Gathering). Traccia
in git lo stato delle carte (rating, label, classificazione) tramite file CSV;
le immagini fisiche risiedono localmente e non sono versionate.

## Struttura

```
ScatoloneQuintet/
├── Source/          # immagini PNG (non tracciato in git — vedi .gitignore)
│   └── <anno>/<set>/<carta>.png
├── README.md
└── docs/            # CSV repository (generato da ScatoloneDownloader --export-git)
```

## Note generali

### Meccaniche bannate

Queste carte non compaiono nella lista delle bannate (da riguardare nel caso si
vogliano reintrodurre):

- The Ring tempts you
- Start your engines!