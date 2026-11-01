# Dokumentace projektu

**Předmět:** Strojové učení 1  
**Autor:** Tobias Hočevar  
**Varianta:** B — jeden datový zdroj, více postupů  
**Název:** Segmentace denních profilů spotřeby elektřiny: k-means a Wardovo hierarchické shlukování

Práce srovnává dva postupy shlukování na denních profilech spotřeby elektřiny. Zdroj je jeden: londýnské chytré elektroměry. Postupy jsou k-means a hierarchické shlukování s Wardovou vazbou. Oba modely dostávají stejnou matici a stejný počet shluků. Výpočet, grafy a tabulky jsou v notebooku `segmentace-profilu.ipynb`.

## 1. Cíl a výzkumná otázka

Cílem je zjistit, zda se dny domácností skládají do opakovaných tvarů odběru a který z obou postupů takový popis na stejných datech podá přesvědčivěji.

Výzkumná otázka zní: lze denní profily spotřeby elektřiny v domácnostech rozdělit do smysluplných skupin (ranní špička, večerní špička, plochý odběr a podobně) a který z postupů, k-means nebo hierarchické shlukování s Wardovou vazbou, to dělá lépe?

Slovo „lépe“ v této práci znamená vyšší siluetový koeficient, nižší Davies–Bouldinův index, stabilnější přiřazení a čitelnější průměrný profil. Ke dnům neexistují správné štítky, takže výsledkem není přesnost klasifikace. Neuronové sítě práce nepoužívá.

## 2. Data

Měření pocházejí z projektu UK Power Networks Low Carbon London. Veřejně je vydává London Datastore jako *SmartMeter Energy Consumption Data in London Households*. Jde o půlhodinovou spotřebu v kWh u 5 567 domácností od listopadu 2011 do února 2014. Stejná měření nabízí Kaggle pod názvem *Smart meters in London* (Jean-Michel D). Výřez v této práci pochází z London Datastore a leží ve složce `data/`. Kaggle API práce nepoužívá.

Celý archiv má po rozbalení kolem 10 GB a je rozdělený do 168 bloků. Práce používá čtyři souvislé bloky `LCL-June2015v2_0.csv` až `LCL-June2015v2_3.csv`. V repozitáři leží jako gzip ve složce `data/`. Sloupce jsou `LCLid` (identifikátor domácnosti), `stdorToU` (tarif), `DateTime` a `KWH/hh (per half hour)`.

Ve výřezu je jen standardní tarif `Std`. Dynamický tarif dToU, u kterého se v roce 2013 měnila cena během dne, v těchto blocích není. Tvar dne se proto nemíchá s cenovým signálem.

## 3. Jednotka analýzy

Jednotka je domácnost-den. Profil je vektor 24 hodinových spotřeb. Hodina \(h\) je součet dvou půlhodin, které do ní spadají:

\[
r_{i,h} = e_{i,h}^{(0)} + e_{i,h}^{(30)}, \qquad h = 0, 1, \ldots, 23.
\]

\(i\) je jeden den jedné domácnosti. \(e_{i,h}^{(0)}\) je spotřeba v kWh na razítku `HH:00` a \(e_{i,h}^{(30)}\) na razítku `HH:30`. Hodina 0 je tedy součet 00:00 a 00:30, hodina 23 je součet 23:00 a 23:30. Razítka v těchto souborech leží uvnitř kalendářního dne od 00:00 do 23:30, takže oba konce dne patří témuž datu.

Surový profil dne je vektor

\[
\mathbf{r}_i = (r_{i,0}, r_{i,1}, \ldots, r_{i,23}).
\]

Do shlukování vstupuje jen těchto 24 čísel. Identifikátor domácnosti, den v týdnu a roční období v matici nejsou. Den v týdnu a sezóna slouží až k popisu hotových shluků.

## 4. Čištění a vzorek

Práce nechává den ve vzorku jen tehdy, když má právě jedno platné razítko pro každou z 48 půlhodin. Řádek vypadne, když je hodnota `Null`, záporná, nebo když razítko neleží přesně na `:00` či `:30`. Den s dírou nebo s duplicitním razítkem se nedoplňuje: doplněný průměr by do tvaru vnesl hodnotu, kterou elektroměr nezměřil. Den s nulovým součtem také vypadne, protože u něj nemá normalizace smysl.

Poslední identifikátor v bloku 3 (`MAC000131`) soubor utíná uprostřed řady, protože měření pokračují až v bloku 4. Tato domácnost ve vzorku není.

Po těchto pravidlech zůstává kalendářní rok 2013, 108 domácností a 35 959 platných dní. Wardovo shlukování je vůči počtu profilů kvadratické, proto matice bere náhodný podvýběr

\[
n = 5000
\]

dní. Generátor má semeno 42. Těch 5 000 dní pokrývá 107 ze 108 domácností. Domácnosti samotné nejsou náhodný výběr ze všech 5 567 identifikátorů. Jsou to souvislý úsek v blocích 0–3, který se vejde do plánovaného rozsahu zhruba 80–150 domácností za jeden rok.

## 5. Normalizace

Shluky mají zachytit tvar dne, ne velikost domácnosti. Každý profil se proto vydělí svou eukleidovskou normou (L2):

\[
\|\mathbf{r}_i\|_2 = \sqrt{\sum_{h=0}^{23} r_{i,h}^{2}}, \qquad
\mathbf{x}_i = \frac{\mathbf{r}_i}{\|\mathbf{r}_i\|_2}.
\]

\(\|\mathbf{r}_i\|_2\) je délka vektoru. Po dělení má každý \(\mathbf{x}_i\) délku 1, takže leží na jednotkové sféře. Dva dny se stejným průběhem a jinou úrovní spotřeby pak spadnou k sobě. Vzdálenost mezi body měří rozdíl tvaru, ne rozdíl v kilowatthodinách.

Průměrný profil v původních kWh se počítá zvlášť, z vektorů \(\mathbf{r}_i\) členů shluku. Průměr normalizovaných vektorů \(\mathbf{x}_i\) se znovu nenormalizuje: na obrázku tvaru je to průměrný směr, na obrázku v kWh je to průměrná úroveň. U každého shluku práce uvádí obojí, včetně průměrné denní spotřeby \(\sum_h r_{i,h}\).

## 6. Modely

### 6.1 Společné nastavení

Oba modely dostanou tutéž matici \(\mathbf{X}\) o rozměru \(5000 \times 24\) a totéž \(k\). Druhý postup je Wardova vazba a řez dendrogramu. K-means s jiným semenem do srovnání jako druhá metoda nevstupuje; jiné semeno slouží jen u kontroly stability.

### 6.2 Volba počtu shluků

\(k\) se vybírá na k-means, dřív než se metody srovnají, a pak zůstane fixní. Mřížka je \(k = 2, 3, \ldots, 8\).

Pro každé \(k\) model spočítá setrvačnost (součet čtverců uvnitř shluků) a siluetový koeficient na celé matici. Zvolené \(k\) je to s nejvyšší siluetou. Loketní graf setrvačnosti kontroluje, že toto maximum nesedí na náhodném zoubku mimo zlom křivky. Po volbě se \(k\) už nemění podle toho, které metodě by jiné číslo víc slušelo.

Na této matici má silueta maximum u \(k = 3\) (0,1013). Druhé je \(k = 7\) (0,1011). Pravidlo bere přísně vyšší hodnotu, tedy 3. Setrvačnost klesá pořád; nejprudší schod je mezi 2 a 3 a dál už bez druhého ostrého lokte. Absolutní silueta kolem 0,10 znamená, že se tvary překrývají. \(k = 3\) je výsledek tohoto pravidla, ne ostrá mezera v datech.

### 6.3 K-means

K-means hledá \(k\) centroidů \(\boldsymbol{\mu}_1, \ldots, \boldsymbol{\mu}_k\) a přiřazení dnů do shluků \(C_c\) tak, aby součet čtverců kolem centroidů byl co nejmenší:

\[
J = \sum_{c=1}^{k} \sum_{\mathbf{x}_i \in C_c} \|\mathbf{x}_i - \boldsymbol{\mu}_c\|_2^{2}.
\]

\(C_c\) je množina dní přiřazených shluku \(c\). \(\boldsymbol{\mu}_c\) je centroid, tedy aritmetický průměr vektorů v \(C_c\). \(\|\mathbf{x}_i - \boldsymbol{\mu}_c\|_2\) je eukleidovská vzdálenost dne od centroidu. Veličina \(J\) se v notebooku nazývá setrvačnost.

Algoritmus střídá dva kroky. Nejdřív každý den přiřadí k nejbližšímu centroidu. Pak každý centroid posune do průměru svých dní. Hlavní model bere `n_init = 10` a semeno 42: z deseti náhodných startů zůstane běh s nejmenším \(J\). Implementace je `sklearn.cluster.KMeans` s algoritmem Lloyd.

### 6.4 Wardovo hierarchické shlukování

Wardova vazba staví strom od jednotlivých dní. V každém kroku spojí dvojici shluků, u které nejméně naroste setrvačnost \(J\). Pro dvojici shluků \(A\) a \(B\) má tento nárůst uzavřený tvar

\[
\Delta(A, B) = \frac{|A|\,|B|}{|A| + |B|} \,\|\boldsymbol{\mu}_A - \boldsymbol{\mu}_B\|_2^{2}.
\]

\(|A|\) a \(|B|\) jsou počty dní ve shlucích. \(\boldsymbol{\mu}_A\) a \(\boldsymbol{\mu}_B\) jsou jejich centroidy. Čím jsou centroidy dál a čím jsou shluky větší, tím je \(\Delta\) větší a tím méně ochotně je vazba spojí. Malý odlehlý shluk se proto od zbytku oddělí vysoko ve stromu.

Strom se postaví na celé matici \(\mathbf{X}\) (`scipy.cluster.hierarchy.linkage`, metoda `ward`). Řez `fcluster` s `criterion="maxclust"` ponechá právě \(k\) shluků zvolených v části 6.2. Dendrogram v notebooku ukazuje jen posledních 12 spojení téhož stromu. Barvy v obrázku jsou orientační; metriky berou štítky z řezu.

## 7. Kritéria srovnání

### 7.1 Siluetový koeficient

Pro den \(i\) je \(a(i)\) průměrná vzdálenost k ostatním dnům ve stejném shluku a \(b(i)\) nejmenší průměrná vzdálenost k dnům v jiném shluku. Silueta dne je

\[
s(i) = \frac{b(i) - a(i)}{\max\{a(i), b(i)\}}.
\]

\(s(i)\) leží v intervalu od \(-1\) do \(1\). Hodnota blízko 1 znamená, že den sedí ve svém shluku a od ostatních je daleko. Hodnota blízko 0 znamená překryv. Záporná hodnota znamená, že den je blíž jinému shluku. Výsledná silueta modelu je průměr \(s(i)\) přes všechny dny matice. Vyšší průměr je lepší. Notebook ji počítá eukleidovsky na celé matici, bez dalšího podvýběru.

### 7.2 Davies–Bouldinův index

U shluku \(c\) je \(\sigma_c\) průměrná vzdálenost jeho dní od vlastního centroidu (rozptyl shluku) a \(d(\boldsymbol{\mu}_c, \boldsymbol{\mu}_{c'})\) vzdálenost centroidů. Index je

\[
DB = \frac{1}{k} \sum_{c=1}^{k} \max_{c' \neq c} \frac{\sigma_c + \sigma_{c'}}{d(\boldsymbol{\mu}_c, \boldsymbol{\mu}_{c'})}.
\]

Pro každý shluk se vezme nejhorší poměr: součet rozptylů dělený vzdáleností centroidů. \(DB\) je průměr těchto nejhorších poměrů. Kompaktní a navzájem vzdálené shluky dávají nízké číslo, proto je nižší hodnota lepší.

### 7.3 Upravený Randův index

Upravený Randův index (ARI) srovnává dvě rozdělení stejných dní. Dívá se na dvojice dní: shodují-li se oba řezy v tom, jestli dvojice patří do stejného shluku. Od této shody odečte shodu, která by vyšla při náhodném přiřazení se stejnými velikostmi shluků. ARI nezávisí na přečíslování štítků. Hodnota 1 znamená totožný řez, hodnota 0 znamená shodu na úrovni náhody.

V tabulce kontingence je \(n_{uv}\) počet dní, které první řez dá do shluku \(u\) a druhý do shluku \(v\). \(a_u = \sum_v n_{uv}\) a \(b_v = \sum_u n_{uv}\) jsou velikosti shluků, \(n\) je počet dní. Pak

\[
ARI = \frac{
\sum_{u,v} \binom{n_{uv}}{2}
- \big[\sum_u \binom{a_u}{2} \sum_v \binom{b_v}{2}\big] \big/ \binom{n}{2}
}{
\frac{1}{2}\big[\sum_u \binom{a_u}{2} + \sum_v \binom{b_v}{2}\big]
- \big[\sum_u \binom{a_u}{2} \sum_v \binom{b_v}{2}\big] \big/ \binom{n}{2}
}.
\]

\(\binom{n_{uv}}{2}\) počítá dvojice, na kterých se oba řezy shodnou uvnitř jedné buňky tabulky.

ARI mezi k-means a Wardem říká, jestli přiřazují tytéž dny stejně. Není to skóre kvality. Setrvačnost \(J\) notebook uvádí jen jako popis: k-means ji přímo minimalizuje, takže nižší \(J\) u k-means není nezávislý rozhodčí.

### 7.4 Stabilita

Stabilita se u každé metody měří jiným, předem popsaným pokusem.

U k-means jsou to dva samostatné běhy s `n_init = 1` a semeny 42 a 43 na celé matici. ARI mezi nimi říká, jestli start centroidů řez mění. Hlavní model, který vstupuje do srovnání, má pořád `n_init = 10`. Pár s jedním startem není druhá metoda.

U Wardovy vazby náhodná inicializace centroidů není. Stabilitu proto měří dva nezávislé podvýběry 60 % profilů (semena 11 a 12). Na každém se znovu postaví strom, uřízne se na stejné \(k\) a ARI se spočítá na průniku dní. Nízké ARI znamená, že řez závisí na tom, které dny ve vzorku zrovna jsou.

## 8. Pojmenování profilů

Jméno shluku je čtecí pomůcka z průměrného profilu v kWh, ne další model. Průměr se rozdělí na noc (hodiny 0–5), ráno (6–9) a večer (17–21).

- Noční odběr: vrchol v hodinách 0–4, nebo aspoň 35 % energie v hodinách 0–5.
- Večerní špička: vrchol v hodině 16 nebo později a aspoň 30 % energie v hodinách 17–21.
- Ranní špička: vrchol mezi 6 a 9 a aspoň 28 % energie v hodinách 6–9.
- Jinak rozložený denní odběr.

Při shodě jmen se ke jménu přidá hodina vrcholu. Barva v grafech sleduje hodinu vrcholu uvnitř jedné metody. Stejná barva u k-means a u Warda proto není párování shluků.

## 9. Výsledky

Na společné matici a při \(k = 3\):

| Metoda | Silueta | Davies–Bouldin | Setrvačnost \(J\) | Stabilita (ARI) |
| --- | ---: | ---: | ---: | ---: |
| k-means | 0,101 | 2,377 | 1193,8 | 0,934 |
| Ward | 0,073 | 2,827 | 1239,7 | 0,259 |

Vyšší silueta je lepší, nižší Davies–Bouldinův index je lepší. ARI mezi oběma řezy na týchž dnech je 0,369.

Velikosti a úrovně shluků:

| Metoda | Shluk | Počet dní | Podíl | kWh za den | Vrchol | Podíl noci | Podíl večera |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| k-means | noční odběr | 200 | 4,0 % | 18,7 | 0 h | 64 % | 10 % |
| k-means | rozložený denní odběr | 2 722 | 54,4 % | 10,5 | 12 h | 16 % | 24 % |
| k-means | večerní špička | 2 078 | 41,6 % | 12,2 | 19 h | 13 % | 38 % |
| Ward | noční odběr | 148 | 3,0 % | 20,7 | 0 h | 69 % | 8 % |
| Ward | rozložený denní odběr | 2 527 | 50,5 % | 10,3 | 15 h | 16 % | 24 % |
| Ward | večerní špička | 2 325 | 46,5 % | 12,4 | 19 h | 13 % | 36 % |

Podíl noci jsou hodiny 0–5, podíl večera hodiny 17–21, obojí z průměrného profilu v kWh.

## 10. Interpretace

K-means při \(k = 3\) dává tři tvary, které jdou pojmenovat bez násilí.

Noční odběr má 200 dní. Průměrný den má 18,7 kWh a asi 64 % z toho leží v hodinách 0–5. V první hodině po půlnoci je průměr kolem 3,6 kWh, přes den odběr spadne k několika desetinám. Úroveň je vyšší než u ostatních skupin, takže jde o energeticky těžký režim, ne o malou noční spotřebu svícení.

Rozložený denní odběr má 2 722 dní a průměr 10,5 kWh. V noci je kolem 0,3 kWh za hodinu, od pozdního rána do večera se průměr drží kolem 0,5 kWh. Číselný vrchol vychází kolem poledne, schod mezi dopolednem a večerem je malý. Samostatná ranní špička při tomto \(k\) nevzniká.

Večerní špička má 2 078 dní a průměr 12,2 kWh. Maximum je v hodině 19, kolem 1 kWh, a hodiny 17–21 nesou asi 38 % dne. Odpoledne odběr stoupá a po dvacáté hodině klesá.

Ward uříznutý na stejné \(k\) najde tytéž tři pojmenovatelné průměry. Noční skupina je menší a ostřejší (148 dní, 20,7 kWh). Rozložený den a večerní špička jsou v obrázku k-means velmi podobné. ARI 0,369 přesto říká, že stejné jméno průměru neznamená stejné přiřazení dne.

Den v týdnu a sezóna do modelu nevstupují. U k-means je podíl večerní špičky v pracovních dnech asi 42–45 %, v sobotu 34 % a v neděli 39 %. V zimě a na podzim je večerní podíl asi 48 %, v létě 34 % a na jaře 38 %. Rozložený den je častější v létě a v sobotu. Noční podíl zůstává ve všech sezónách kolem 3–5 %. Sezóna je meteorologická: zima je prosinec až únor, jaro březen až květen, léto červen až srpen, podzim zbytek roku. Wardovy podíly kopírují stejný směr.

Noční dny nejsou rozptýlené po celém panelu. U k-means jich je 200 a patří 22 domácnostem; pět z nich pokryje 83 % těchto dní. U Warda patří 148 nočních dní deseti domácnostem a pět z nich pokryje 96 %. Shluk tedy zachycuje hlavně domácnosti se stálým nočním režimem. To plyne z jednotky domácnost-den: sousední dny jedné domácnosti si jsou podobné.

Na společné matici má k-means vyšší siluetu, nižší Davies–Bouldinův index a výrazně klidnější řez. Průměrné profily umí pojmenovat obě metody. Podle kritérií z části 7 je na tomto vzorku přesvědčivější k-means. Čitelnost sama by mezi nimi nerozhodla.

## 11. Omezení

Vzorek je 108 domácností na standardním tarifu ze souvislých bloků 0–3 a rok 2013. Do matice vstupuje 5 000 z 35 959 platných dní. Výsledek nepopisuje všech 5 567 domácností ani celou délku projektu.

Den s dírou, duplicitou nebo `Null` v datech není a není ani doplněný. Svátky a Vánoce roku 2013 ve vzorku zůstávají a mohou jednotlivý den posunout. Řada měření končí v únoru 2014, takže výsledek nic neříká o pozdějších letech.

L2 schovává úroveň uvnitř modelu. Noční shluk je zároveň jiný tvar a vyšší denní součet; obojí je v tabulce uvedeno schválně. Dny jedné domácnosti nejsou nezávislé pozorování.

Silueta 0,10 je v absolutní hodnotě nízká a \(k = 7\) je v siluetě těsně v závěsu. Skupiny se překrývají a průměrný profil je zjednodušení. Setrvačnost není nestranný rozhodčí, protože ji k-means přímo minimalizuje.

U veřejného záznamu téhož archivu je licence v metadatech nevyplněná a kopie na Kaggle má vlastní podmínky. V repozitáři je malý výřez pro tuto práci.

Když soubory ve složce `data/` chybí, notebook doběhne na označené miniaturní náhradě se čtyřmi vtisknutými tvary. Čísla v této dokumentaci patří běhu na blocích London Datastore, ne té náhradě.

## 12. Reprodukce

Závislosti jsou v `requirements.txt` (NumPy, pandas, matplotlib, SciPy, scikit-learn). Notebook se spouští ze složky, ve které leží, protože data hledá v `data/`. Semeno generátoru je 42. Hlavní objekty v kódu jsou matice \(\mathbf{X}\), vektory \(\mathbf{r}_i\) a štítky obou řezů při \(k = 3\).
