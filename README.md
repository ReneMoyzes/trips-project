# Evidence databáze pro sdílení nákladů na výlety

## 1. Nápad a popis systému

Systém pomáhá malé skupině naplánovat výlet a přehledně rozdělit společné výdaje mezi účastníky.  
Řeší situaci, kdy se náklady platí na různých místech a po návratu není jasné, kdo má kolik doplatit.  
Vedoucí výletu díky evidenci nemusí částky dohledávat v chatu, účtenkách a tabulkách.  
Systém slouží vedoucím výletů a jejich účastníkům.  
Uchovává členy, výlety, účast jednotlivých členů, kategorie služeb, výdaje, podíly účastníků na výdajích a platby.  
U každého výdaje lze zaznamenat částku, datum, kategorii a popis.  
Vedoucí může výdaj rozdělit mezi účastníky a zaznamenat jejich platby do společného rozpočtu.  
Systém umožní zjistit skutečné náklady výletu i součet nákladů připadající na každého účastníka.  
Porovnáním podílů na výdajích s potvrzenými platbami ukáže, kdo má ještě doplatit a kdo zaplatil navíc.  
Tento návrh je malou, připravenou základnou pro aplikaci; přihlášení uživatelů, nahrávání účtenek ani automatické platby nejsou jeho součástí.

## 2. Uživatelé a přínosy

- **Vedoucí výletu** zakládá výlet, přidává účastníky a služby, eviduje výdaje a potvrzuje platby. Na nákladech se nepodílí; výdaje se rozdělí mezi ostatní účastníky.
- **Účastník** může získat přehled o svém podílu na nákladech a o zaplacené částce.

Konkrétním přínosem je dohledatelné rozúčtování: namísto ručního sčítání je možné získat z databáze částku připadající na každého účastníka a porovnat ji s jeho potvrzenými platbami.

## 3. Od požadavků k databázovému modelu

| Požadavek | Návrh v databázi |
|---|---|
| Evidovat osoby, které výlety vedou nebo se jich účastní | `members` |
| Evidovat výlety, jejich termíny a očekávaný rozpočet | `trips` |
| Zaznamenat účastníky a vedoucí konkrétního výletu | `participants`, spojovací tabulka mezi členy a výlety s příznakem `is_leader` |
| Kategorizovat výdaje | `services` |
| Evidovat výdaj v rámci výletu | `expenses`, každý výdaj má kategorii |
| Rozdělit výdaj mezi účastníky | `expense_shares`, každému účastníkovi lze přiřadit podíl |
| Zaznamenat platby účastníků | `payments` |
| Zjistit neuhrazený rozdíl | Agregační `SELECT` nad podíly a potvrzenými platbami; zůstatek se neukládá jako ručně měněná hodnota |

Model má **7 tabulek**. Vztah člena k výletu je typu mnoho-k-mnoha a realizuje se tabulkou `participants`. Výdaj může mít podíl pro více účastníků a jeden účastník může mít podíly na více výdajích; tento vztah mnoho-k-mnoha realizuje `expense_shares`.

### ER diagram

Soubor [`trips.dbml`](trips.dbml) obsahuje diagram pro import do [dbdiagram.io](https://dbdiagram.io/). Hlavní vztahy jsou:

```text
members 1 ── N participants N ── 1 trips (role účastník/vedoucí)
trips 1 ── N expenses N ── 1 services
expenses 1 ── N expense_shares N ── 1 participants
participants 1 ── N payments
```

### Popis tabulek

| Tabulka | Účel a důležité údaje |
|---|---|
| `members` | Členové systému, jméno, příjmení, datum registrace a příznak aktivního účtu. |
| `trips` | Název výletu, datum začátku a konce a případné očekávané náklady. |
| `participants` | Přihlášení člena na výlet, příznak aktivní účasti a role vedoucího (`is_leader`). Role je vázaná na konkrétní výlet; jedinečná kombinace člena a výletu brání duplicitní účasti. |
| `services` | Kategorie výdajů, například ubytování, doprava nebo strava. |
| `expenses` | Konkrétní výdaje daného výletu, jejich kategorie, částka, datum a popis. |
| `expense_shares` | Částka konkrétního výdaje přiřazená konkrétnímu účastníkovi. |
| `payments` | Platby účastníka do společného rozpočtu; nepotvrzená platba se do vyúčtování nezapočítává. |

## 4. Pravidla a zvolená zjednodušení

- Částky jsou v ukázce vedené v jedné společné měně, bez samostatné tabulky měn.
- Peněžní hodnoty používají `numeric(10,2)`, aby bylo možné přesně ukládat haléře/centy; nepoužívá se typ `float`.
- `is_active`, `is_leader` a `is_confirmed` jsou booleovské příznaky uložené typem `boolean`, který je pro PostgreSQL přirozený.
- Role vedoucího je uložena v `participants.is_leader`, protože stejný člen může vést jeden výlet a být běžným účastníkem jiného. `false` znamená běžného účastníka. Výlet může mít i více vedoucích; databáze jejich počet neomezuje.
- Podíl se eviduje v `expense_shares` na úrovni jednotlivého výdaje a účastníka. Vedoucí v ukázkových datech nemá v `expense_shares` žádné řádky, proto se jeho podíl ve vyúčtování přirozeně zobrazí jako nula; dotaz pouze sčítá zadané podíly a nemusí zvlášť kontrolovat roli.
- Součet podílů jednoho výdaje má odpovídat jeho celkové částce a podíl má patřit účastníkovi stejného výletu, ke kterému patří výdaj. Tato pravidla se týkají více řádků nebo tabulek, takže je běžný `CHECK` nevynutí; v tomto jednoduchém modelu je kontroluje aplikace při zápisu (v pokročilejší verzi by šlo použít trigger nebo složené cizí klíče).
- Ukázkový dotaz zahrnuje jen účastníky s `participants.is_active = true`. Pokud neaktivní účastník může mít stále nevyrovnané náklady, je nutné upravit pravidla vyúčtování; příznak účasti sám o sobě nemaže jeho výdaje ani platby.
- Platba je chápána jako příspěvek účastníka do společného rozpočtu, nikoli jako přímá úhrada konkrétní účtenky. Nově zadaná platba je ve výchozím stavu nepotvrzená (`is_confirmed = false`) a do vyúčtování se započítá až po potvrzení vedoucím. Pokud má být evidováno, kdo výdaj původně zaplatil, lze v další verzi přidat do `expenses` odkaz na účastníka-platícího.
- Kladný rozdíl **podíly − potvrzené platby** znamená částku k doplacení; záporný rozdíl znamená příspěvek nad přidělené podíly. Potvrzené platby vedoucího se započítávají jako příspěvky do společného rozpočtu, přestože vedoucímu nejsou přiděleny náklady. Nejde o uložený sloupec, ale o výsledek dotazu, aby se předešlo zastaralým zůstatkům.
- Unikátní omezení brání duplicitní účasti člena na výletu i duplicitnímu podílu účastníka na jednom výdaji.
- Cizí klíče a `CHECK` omezení hlídají základní integritu: platnost termínu, nezáporný očekávaný rozpočet a podíly i kladné částky výdajů a plateb.
- Indexy jsou vytvořené na cizích klíčích, přes které vyúčtování vyhledává data výletu (`participants.trip_id`, `expenses.trip_id`, `expense_shares.expense_id`, `payments.participant_id`).

## 5. Ukázková data a dotaz

Skript [`trips.sql`](trips.sql) vytvoří schéma a všech sedm tabulek, vloží vzorová data pro víkendový výlet do Prahy a obsahuje dotaz pro výpočet vyúčtování. Anna (vedoucí) nepřebírá podíl na výdajích; celkových 7 800,00 se rozdělí mezi Davida a Evu, každému po 3 900,00. Potvrzené platby jsou 3 000,00, 2 000,00 a 2 800,00. Výsledné rozdíly jsou doplatek 1 900,00 u Davida, doplatek 1 100,00 u Evy a −3 000,00 u vedoucí Anny: Anna zaplatila 3 000,00 do společného rozpočtu, přestože nemá žádný podíl, a doplatky Davida a Evy jí tuto částku vrátí. Součet zůstatků je proto nula. Očekávaný rozpočet výletu je 8 000,00, tedy o 200,00 vyšší než evidované výdaje.

Ukázkový dotaz odpovídá na otázku: **Kolik má každý účastník doplatit, nebo kolik zaplatil navíc?** Je přiložen na konci `trips.sql` a samostatně také v souboru [`vzorovy_select.sql`](vzorovy_select.sql). Používá samostatné agregace podílů a plateb, aby spojení více výdajů a plateb neznásobilo částky. Podíly se k výletu přiřazují přes výdaj (`expenses.trip_id`), platby přes účastníka (`participants.trip_id`). Role se v agregaci podílů samostatně nefiltruje: rozhodující je, komu byl záznam v `expense_shares` přiřazen. Ve vzorovém souboru změňte `trip_id` v CTE `selected_trip`, pokud chcete zobrazit jiné vyúčtování.

## 6. Spuštění v PostgreSQL

1. V dbdiagram.io zvolte **Import** a nahrajte `trips.dbml`. Pro získání SQL z diagramu použijte export pro PostgreSQL; skript `trips.sql` je připravená varianta se vzorovými daty a kontrolními omezeními.
2. Pro novou databázi spusťte `trips.sql` jako uživatel s oprávněním vytvořit schéma a tabulky. Skript vloží ukázková data a na konci rovnou spustí vyúčtovací `SELECT`.
3. Skript vytváří objekty bez `DROP`, proto je určen pro databázi, kde tabulky schématu `trips` ještě neexistují. Pro opakované spuštění nejprve odstraňte celé schéma příkazem `DROP SCHEMA trips CASCADE;` (smaže všechny jeho tabulky i data). Samostatný dotaz v [`vzorovy_select.sql`](vzorovy_select.sql) lze spouštět opakovaně; pro jiný výlet změňte `trip_id` v CTE `selected_trip`.

DBML zachycuje strukturu a vztahy pro diagram a export. SQL skript je autoritativní spustitelná verze: kromě struktury doplňuje také kontroly hodnot a vzorová data.
