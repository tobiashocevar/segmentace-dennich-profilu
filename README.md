# Segmentace denních profilů spotřeby elektřiny

Semestrální práce z předmětu Strojové učení 1, varianta B. Práce srovnává k-means a Wardovo hierarchické shlukování na denních profilech londýnských domácností (Low Carbon London, rok 2013).

Oba modely dostávají stejnou matici 5 000 normalizovaných profilů a stejné \(k\). Dokumentace s rovnicemi, definicí vzorku a výsledky je v [dokumentace.md](dokumentace.md). Výpočet je v [segmentace-profilu.ipynb](segmentace-profilu.ipynb).

## Spuštění

```bash
pip install -r requirements.txt
jupyter notebook segmentace-profilu.ipynb
```

Notebook se spouští z této složky. Půlhodinová měření jsou ve `data/` jako `LCL-June2015v2_0.csv.gz` až `LCL-June2015v2_3.csv.gz`. Semeno generátoru je 42.

Licence zdrojového archivu je u veřejného záznamu nevyplněná. V repozitáři je jen výřez pro tuto práci.
