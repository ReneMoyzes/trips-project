# Evidence databáze pro sdílení nákladů na výlety

## 1. Nápad a popis systému

Systém pomáhá malé skupině naplánovat výlet a přehledně rozdělit společné výdaje mezi účastníky.  
Řeší situaci, kdy se náklady platí na různých místech a po návratu není jasné, kdo má kolik doplatit.  
Organizátor díky evidenci nemusí částky dohledávat v chatu, účtenkách a tabulkách.  
Systém slouží organizátorům výletů a jejich účastníkům.  
Uchovává členy, výlety, účast jednotlivých členů, kategorie služeb, výdaje, podíly účastníků na výdajích a platby.  
U každého výdaje lze zaznamenat částku, datum, kategorii a popis.  
Organizátor může výdaj rozdělit mezi účastníky a zaznamenat jejich platby do společného rozpočtu.  
Systém umožní zjistit skutečné náklady výletu i součet nákladů připadající na každého účastníka.  
Porovnáním podílů na výdajích s potvrzenými platbami ukáže, kdo má ještě doplatit a kdo zaplatil navíc.  
Tento návrh je malou, připravenou základnou pro aplikaci; přihlášení uživatelů, nahrávání účtenek ani automatické platby nejsou jeho součástí.

## 2. Uživatelé a přínosy

- **Organizátor** zakládá výlet, přidává účastníky a služby, eviduje výdaje a platby.
- **Účastník** může získat přehled o svém podílu na nákladech a o zaplacené částce.

Konkrétním přínosem je dohledatelné rozúčtování: namísto ručního sčítání je možné získat z databáze částku připadající na každého účastníka a porovnat ji s jeho potvrzenými platbami.

## 3. Od požadavků k databázovému modelu

| Požadavek | Návrh v databázi |
|---|---|
| Evidovat osoby, které výlety organizují nebo navštěvují | `members` |
| Evidovat výlety a jejich termíny a očekávaný rozpočet | `trips` |
| Zaznamenat členy přihlášené na konkrétní výlet | `participants`, spojovací tabulka mezi členy a výlety |
| Kategorizovat výdaje | `services` |
| Evidovat výdaj v rámci výletu | `expenses`, každý výdaj má kategorii |
| Rozdělit výdaj mezi účastníky | `expense_shares`, každému účastníkovi lze přiřadit podíl |
| Zaznamenat platby účastníků | `payments` |
| Zjistit neuhrazený rozdíl | Agregační `SELECT` nad podíly a potvrzenými platbami; zůstatek se neukládá jako ručně měněná hodnota |

Model má **7 tabulek**. Vztah člena k výletu je typu mnoho-k-mnoha a realizuje se tabulkou `participants`. Výdaj může mít podíl pro více účastníků a jeden účastník může mít podíly na více výdajích; tento vztah mnoho-k-mnoha realizuje `expense_shares`.

### ER diagram

Soubor [`trips.dbml`](trips.dbml) obsahuje diagram pro import do [dbdiagram.io](https://dbdiagram.io/). Hlavní vztahy jsou:

```text
members 1 ── N participants N ── 1 trips
trips 1 ── N expenses N ── 1 services
expenses 1 ── N expense_shares N ── 1 participants
participants 1 ── N payments
```

### Popis tabulek

| Tabulka | Účel a důležité údaje |
|---|---|
| `members` | Členové systému, jméno, příjmení, datum registrace a příznak aktivního účtu. |
| `trips` | Název výletu, datum začátku a konce, případně očekávané náklady. |
| `participants` | Přihlášení člena na výlet. Jedinečná kombinace člena a výletu brání duplicitní účasti. |
| `services` | Kategorie výdajů, například ubytování, doprava nebo strava. |
| `expenses` | Konkrétní výdaje daného výletu, jejich kategorie, částka, datum a popis. |
| `expense_shares` | Částka konkrétního výdaje přiřazená konkrétnímu účastníkovi. |
| `payments` | Platby účastníka do společného rozpočtu; nepotvrzená platba se do vyúčtování nezapočítává. |

## 4. Pravidla a zvolená zjednodušení

- Částky jsou v ukázce vedené v jedné společné měně, bez samostatné tabulky měn.
- Peněžní hodnoty používají `numeric(10,2)`, aby bylo možné přesně ukládat haléře/centy; nepoužívá se typ `float`.
- `is_active` a `is_confirmed` jsou booleovské příznaky. Původní `bit` je nahrazeno typem `boolean`, který je pro PostgreSQL přirozenější.
- Podíl se eviduje na úrovni jednotlivého výdaje a účastníka. Součet podílů jednoho výdaje má odpovídat jeho celkové částce; tuto souhrnnou podmínku běžný `CHECK` přes více řádků bezpečně nevynucuje, proto ji při zápisu musí kontrolovat aplikace nebo databázový trigger.
- Platba je chápána jako příspěvek účastníka do společného rozpočtu, nikoli jako přímá úhrada konkrétní účtenky. Pokud má být evidováno, kdo výdaj původně zaplatil, lze v další verzi přidat do `expenses` odkaz na účastníka-platícího.
- Kladný rozdíl **podíly − potvrzené platby** znamená částku k doplacení; záporný rozdíl znamená přeplatek. Nejde o uložený sloupec, ale o výsledek dotazu, aby se předešlo zastaralým zůstatkům.
- Původní název `Outcome` je nahrazen výstižnějším `expenses`, překlep `ammount` je opraven a odvozený `balance` není uložen v tabulce účasti.
- Cizí klíče a unikátní omezení hlídají základní integritu. Kontrola data konce výletu a nezáporných částek je uvedena přímo v PostgreSQL skriptu.

## 5. Ukázková data a dotaz

Skript [`trips.sql`](trips.sql) vytvoří schéma a všech sedm tabulek, vloží vzorová data pro víkendový výlet do Prahy a obsahuje dotaz pro výpočet vyúčtování. Příklad má tři členy, tři výdaje v celkové výši 7 800,00 a tři platby. Každý účastníkovi připadá na výdaje 2 600,00; potvrzené platby jsou 3 000,00, 2 000,00 a 2 800,00. Výsledný rozdíl tedy ukáže přeplatek 400,00, doplatek 600,00 a přeplatek 200,00.

Ukázkový dotaz odpovídá na otázku: **Kolik má každý účastník doplatit, nebo kolik zaplatil navíc?** Je přiložen na konci `trips.sql` a používá samostatné agregace podílů a plateb, aby spojení více výdajů a plateb neznásobilo částky.

## 6. Spuštění v PostgreSQL

1. V dbdiagram.io zvolte **Import** a nahrajte `trips.dbml`. Pro získání SQL z diagramu použijte export pro PostgreSQL; skript `trips.sql` je připravená varianta se vzorovými daty a kontrolními omezeními.
2. Připojte se k požadované PostgreSQL databázi a spusťte `trips.sql` jako uživatel s oprávněním vytvořit schéma a tabulky.
3. Skript je určen pro jednorázové spuštění v čistém schématu `trips`. Po vložení ukázkových dat spusťte závěrečný `SELECT`; v ukázce použijte `trip_id = 1`.

DBML zachycuje strukturu a vztahy pro diagram a export. SQL skript je autoritativní spustitelná verze: kromě struktury doplňuje také kontroly hodnot a vzorová data.
