# Jak powstrzymać tryb agenta GitHub Copilot przed ponownym czytaniem projektu przy każdym promcie (IntelliJ / JetBrains, GPT-5.4)

## W skrócie (TL;DR)
- **Ponowne czytanie to działanie zgodne z projektem narzędzia, a nie bug.** Lokalny agent Copilota w JetBrains utrzymuje bieżącą rozmowę, ale wyniki jego narzędzi (odczyty plików, grepy) są obcinane/streszczane w miarę zapełniania okna kontekstu, a przy każdym nowym promcie model na nowo odkrywa repo świeżymi wywołaniami narzędzi `read_file`/`search`. Przy GPT-5.4 jest to spotęgowane, bo GitHub osobno nalicza tokeny cache'owane, wejściowe i wyjściowe, a długie pętle agenta szybko spalają kredyty. **Pojedynczą poprawką o największym wpływie jest danie agentowi trwałej, spisanej „mapy projektu" w `.github/copilot-instructions.md` oraz samodzielnie utrzymywanego folderu `memory-bank/` (postęp + aktywny kontekst), który każesz mu czytać najpierw i aktualizować na końcu** — to zastępuje kosztowne skanowanie na żywo tanim, cache'owanym, zawsze wstrzykiwanym tekstem.
- **Prompt caching już pomaga automatycznie** (cache'owane wejście rozlicza się „mniej więcej po jednej dziesiątej stawki świeżego wejścia"), więc drugorzędnym celem jest utrzymanie stabilnych części kontekstu identycznych co do bajta między turami (stabilny prompt systemowy + pliki instrukcji + wczesne wiadomości), żeby maksymalizować trafienia w cache, oraz powstrzymanie agenta przed re-czytaniem przez *podanie mu gotowych odpowiedzi* (przypięte pliki, mapa projektu, scope'owane odwołania `#file`, mniej narzędzi MCP).
- **Wybór edytora ma znaczenie.** Prawdziwe semantyczne indeksowanie bazy kodu / `#codebase` to udokumentowana funkcja VS Code + github.com; plugin JetBrains **nie** jest wymieniany przez GitHub jako konsument w-edytorowego indeksu semantycznego, a istnieje kilka bugów kontekstu specyficznych dla JetBrains. Przy ciężkiej pracy w trybie agenta, gdzie re-czytanie zabija Twój budżet kredytów, VS Code obecnie obsługuje trwały kontekst (indeksowanie, `/compact`, pliki promptów, podgląd debugowania tokenów/cache) pełniej niż JetBrains.

## Najważniejsze ustalenia

### Dlaczego tak się dzieje
1. **Tryb agenta to pętla wywoływania narzędzi.** Opis trybu agenta w JetBrains od GitHub mówi: „Copilot can search the workspace, read file contents, execute terminal commands, retrieve compile or lint errors from the editor, and apply code changes via a speculative decoder endpoint" (GitHub Changelog, 19 maja 2025). Każdy prompt uruchamia świeżą rundę tych wywołań, żeby „zbudować kontekst wokół żądania". Nic nie przypina *treści* wcześniej odczytanego pliku jako wiążącej na kolejną turę — model za każdym razem na nowo decyduje, czy czytać ponownie.
2. **Zarządzanie oknem kontekstu odrzuca szczegóły.** Gdy rozmowa zbliża się do limitu okna, Copilot wykonuje streszczanie/kompaktowanie — zastępuje starsze wiadomości (w tym duże zrzuty plików i wyniki narzędzi) skompresowanym streszczeniem. Dokumentacja kontekstu Copilot CLI mówi „some detail is inevitably lost" oraz że po kompaktowaniu Copilot „may not have" konkretnych wcześniejszych szczegółów — więc czyta ponownie. Zgłoszenia społeczności (GitHub Discussion #162256) opisują, jak agent „tracił ważny kontekst" zaraz po pojawieniu się „Summarized conversation history", a potem ponownie pobierał, czasem w pętli.
3. **Specyfika GPT-5.4.** GPT-5.4 wprowadził „tool search" (leniwe ładowanie definicji narzędzi) i jest „najbardziej oszczędnym tokenowo modelem rozumującym OpenAI", ale w Copilocie jego okno jest duże (powszechnie podawane jako 272K domyślnie, do 400K/1M w niektórych konfiguracjach). Duże okno plus agresywna autonomiczna pętla sprawiają, że model raczej *na nowo wyprowadza* strukturę repo przez narzędzia, niż ufa nieaktualnemu tekstowi streszczenia — więc czyta ponownie częściej, a każde re-czytanie jest opomiarowane. (Zmiany OpenAI faktycznie redukują narzut *definicji narzędzi*, ale nie zatrzymują re-czytania plików wewnątrz pętli agenta.)
4. **Rozliczenia czynią to widocznym i bolesnym.** Wg CPO GitHub Mario Rodrigueza: „Credits will be consumed based on token usage, including input, output, and cached tokens, according to the published API rates for each model" (1 AI Credit = $0,01), od 1 czerwca 2026. Sesje agentowe to najdroższa powierzchnia — dokumentacja GitHub zauważa, że „a long Copilot cloud agent session using a frontier model across multiple files will cost more AI credits", a doniesienia z praktyki są dobitne: jeden developer zużył 1 180 kredytów (16% miesięcznego limitu Pro+) w jednej sesji Claude 4.8 (The Register), a inny zgłosił spalenie 8% limitu Pro+ w dwie godziny (community discussion #192948). Re-czytanie tych samych plików przy każdej turze to teraz bezpośredni, powtarzalny, opomiarowany koszt.

### Prompt caching — czy już redukuje koszt re-czytań?
- **Tak, częściowo i automatycznie.** Copilot używa prompt cachingu dostawcy. W Chat Debug View w VS Code widać `cached_tokens` wyceniane ~10× taniej niż świeże wejście — cache'owane wejście rozlicza się „mniej więcej po jednej dziesiątej stawki świeżego wejścia" (np. Claude Sonnet: wejście 300 AIC/MTok vs odczyt z cache 30 AIC/MTok; opublikowana stawka Sonnet 4.6 to $3,00/1M wejścia vs $0,30/1M cache). Udokumentowane żądanie Kena Muse'a miało ~98% tokenów promptu obsłużonych z cache, co ścięło prompt 45 882 tokenów do 1,94 kredytu.
- **Cache jest wrażliwy na kolejność i treść oraz krótkotrwały.** W logach cache był zapisywany z 5-minutowym oknem ulotnym (ephemeral); cache trafia tylko, gdy wiodące tokeny są *identyczne*. Maksymalizuj trafienia, utrzymując stabilny prefiks (prompt systemowy + pliki instrukcji + wczesny kontekst) bez zmian, nie edytując tych plików w trakcie sesji i nie zmieniając kolejności kontekstu. Gdy plik się zmieni, ta część cache się unieważnia.
- **Caching NIE naprawia re-czytania** — jedynie rabatuje powtarzane wejście. Nadal płacisz za tokeny wyjściowe wywołań narzędzi i za odczyt z cache ponownie wstrzykiwanej treści. Prawdziwe oszczędności biorą się z *zapobiegania re-czytaniu*.
- **Ekspozycja w JetBrains:** caching jest po stronie serwera i działa niezależnie od edytora, ale plugin JetBrains **nie** udostępnia Chat Debug View z rozbiciem tokenów/cache tak jak VS Code — więc nie zmierzysz łatwo współczynnika trafień w cache w IntelliJ.

### Mechanizmy trwałego kontekstu (rdzeń rozwiązania)

**A. Niestandardowe instrukcje repozytorium + „mapa projektu" — `.github/copilot-instructions.md`**
- Czytane automatycznie przy *każdym* żądaniu czatu/agenta w JetBrains (Settings → Languages & Frameworks → GitHub Copilot → Customizations, albo utwórz plik w katalogu głównym repo). Jego treść jest zawsze wstrzykiwana, staje się częścią cache'owalnego prefiksu i sprawia, że agent nie musi *odkrywać* Twojej struktury.
- Oficjalne wytyczne GitHub: trzymaj go zwięzłym (poniżej ~1 000 linii; prompt onboardingowy GitHub celuje w ~2 strony), ustrukturyzowanym: podsumowanie aplikacji, stos technologiczny, **struktura projektu**, standardy kodowania, testy oraz „istniejące narzędzia i zasoby". Mapa projektu mówiąca dokładnie, gdzie co leży, eliminuje potrzebę grepów po całym repo.

**B. AGENTS.md / CLAUDE.md**
- Wg GitHub Changelog (11 marca 2026): „We have added support for AGENTS.md and CLAUDE.md instruction files… Copilot automatically discovers and loads agent instruction files… from the current workspace or globally". Zarządzaj nimi w Settings → GitHub Copilot → Customizations; wygeneruj starter przez „Generate Agent Instructions". Przetrwają między sesjami, bo to pliki repo, ponownie wstrzykiwane przy każdym uruchomieniu. (Znany bug #1717: przełączniki „Use AGENTS.md/CLAUDE.md" nie zawsze były respektowane — zweryfikuj zachowanie.)

**C. Wzorzec memory-bank (trwała sztuczka na ciągłość między sesjami)**
- Sprawdzony przez społeczność sposób na ciągłość. Utwórz folder `memory-bank/` (`projectbrief.md`, `systemPatterns.md`, `techContext.md`, `activeContext.md`, `progress.md`) i dodaj w `copilot-instructions.md` instrukcje, żeby agent **czytał je najpierw na początku każdego zadania i aktualizował przed zakończeniem**. Memory Bank od Cline'a to spopularyzował („After each reset, I rely ENTIRELY on my Memory Bank… I MUST read ALL memory bank files at the start of EVERY task"); `LouisDesca/copilot-memory-bank` adaptuje to konkretnie pod `.github/copilot-instructions.md` Copilota. Ponieważ pliki są krótkie, ustrukturyzowane i zawsze ładowane (oraz cache'owane), agent ponownie wykorzystuje wcześniejsze decyzje zamiast wyprowadzać je od nowa.

**D. Natywna funkcja „Copilot Memory" — NIEdostępna dla lokalnego agenta JetBrains**
- Copilot Memory od GitHub (automatycznie zapisywane fakty o repo + preferencje użytkownika) jest w publicznym preview, ale dokumentacja mówi wprost: „Copilot Memory is currently used by Copilot cloud agent, Copilot code review, and Copilot CLI… any stored fact or preference that goes unused is automatically deleted after 28 days". **Natywny lokalny agent w IDE (JetBrains lub VS Code) nie znajduje się na tej liście.** Nie polegaj na tym w sesjach agenta w edytorze — plikowy memory-bank jest substytutem. (Ścieżka pośrednia: agent Copilot CLI jest teraz osadzalny wewnątrz JetBrains wg changelogu z 2 czerwca 2026, a CLI *używa* Memory — ale to osobna powierzchnia agenta niż natywny tryb agenta.)
- Komenda **`/memory` w JetBrains jest myląco nazwana:** wg changelogu z 11 marca 2026 „open[s] settings for agent instruction files", czyli zarządza AGENTS.md/CLAUDE.md/instrukcjami niestandardowymi — to **nie** jest trwały magazyn pamięci.

**E. Indeksowanie bazy kodu / wyszukiwanie semantyczne**
- Zdalny indeks semantyczny GitHub jest budowany z domyślnej gałęzi repo, współdzielony na poziomie repo i pozwala agentowi pobierać istotne fragmenty wg znaczenia zamiast czytać całe pliki. Ale oficjalna dokumentacja ogranicza w-edytorowy indeks semantyczny / `#codebase` do **VS Code i github.com** — JetBrains **nie** jest wymieniany, a GitHub nie opublikował oficjalnego oświadczenia, że tryb agenta w JetBrains używa zdalnego indeksu. Więc w IntelliJ nie możesz liczyć na indeksowanie semantyczne, żeby tłumiło re-czytanie tak jak w VS Code.
- Zdalny/semantyczny indeks bazy kodu zwykle wymaga Business/Enterprise. Skoro jesteś na Enterprise, niech administrator potwierdzi, że polityka indeksowania repozytoriów jest **włączona** (domyślnie wyłączona dla organizacji/enterprise).

**F. Serwery pamięci MCP**
- Możesz podłączyć serwer pamięci/grafu wiedzy MCP (serwer pamięci `@modelcontextprotocol` albo społecznościowy `agentmemory`) do pluginu JetBrains (ikona Copilota → Edit settings → MCP Servers). Daje to odpytywalny, trwały magazyn między sesjami. Kompromis: każde narzędzie MCP dokłada ~100–500 tokenów schematu na krok agenta, więc włączaj tylko to, czego używasz.

### Taktyki sesji i workflow, które tną re-czytanie
- **Przypnij/dołącz konkretne pliki** przez selektor kontekstu / `#file`, żeby agent używał dostarczonej treści zamiast wyszukiwać. (Ostrzeżenie o bugu JetBrains: otwarty plik auto-dołącza się i sam ponownie włącza — zgłoszenia #1207, #1221, #1251 — pilnuj, co jest dołączone.)
- **Trzymaj sesje krótkie i jednotematyczne; nowy czat na zadanie.** Długie sesje wyzwalają streszczanie, które powoduje re-czytanie.
- **Front-load research przez tryb Plan** (GA w JetBrains): zaplanuj raz, potem wykonuj, żeby kosztowne odkrywanie odbyło się jeden raz.
- **Streść do pliku przed wyczyszczeniem.** Przed zakończeniem sesji każ agentowi zapisać postęp do `memory-bank/activeContext.md` i `progress.md`. Natywny agent auto-kompaktuje przy ~80% okna, ale w JetBrains nie zawsze wyzwolisz to ręcznie — zapis do pliku to niezawodny mechanizm ciągłości. (`/compact` jest dostępny przez osadzonego agenta Copilot CLI.)
- **Zmniejsz szum narzędzi:** wyłącz nieużywane serwery MCP; podnoś „max requests per turn" (domyślnie 25 → 100) *tylko* przy naprawdę długich zadaniach modernizacyjnych, bo więcej żądań = więcej tokenów.
- **Użyj content exclusion** (Enterprise), żeby trzymać `target/`, generowane źródła, `*.log`, duże XML/YAML poza kontekstem — ale pamiętaj, że tryb agenta historycznie nie w pełni honorował content exclusion, więc polegaj też na `.gitignore` i po prostu nie otwieraj tych plików.

### Ograniczenia / znane problemy specyficzne dla JetBrains
- **Przełączniki kontekstu plików są zabugowane:** „File Context Control Ignores 'Off' Setting" (#1221) oraz „Agent mode constantly keeps adding the file context back on" (#1251) — agent ponownie dołącza pliki, które wyłączyłeś.
- **Pliki promptów (`.github/prompts/*.prompt.md`) są tylko częściowo obsługiwane** w JetBrains; komendy `/` działające w VS Code mogą nie działać, a auto-ładowanie plików promptów historycznie było opóźnione (community #171759). Wymagają też włączenia w organizacji `editor_preview_features`.
- **Bugi „agent nie widzi plików"** w Remote Dev/Gateway (JetBrains IJPL-188898), gdzie korzeń workspace jest źle wykrywany.
- **Limity/429 częstsze w JetBrains niż w VS Code** wg wielu zgłoszeń (#1126, wątki wsparcia JetBrains), przerywające pętle agenta — który po wznowieniu czyta ponownie, marnując więcej kredytów.
- **Brak Chat Debug View** w JetBrains, więc diagnostyka tokenów/cache jest słabsza niż w VS Code.

### Wybór modelu
- **GPT-5.4** w Copilocie: wejście **$2,50** / cache **$0,25** / wyjście **$15,00** za 1M tokenów (domyślnie ≤272K); poziom długiego kontekstu wejście **$5,00** / cache **$0,50** / wyjście **$22,50**. Duże okno zachęca do bardziej autonomicznego re-czytania. (GPT-5.4 mini — wejście $0,75 / wyjście $4,50 — jest zoptymalizowany pod nawigację w stylu grep i znacznie tańszy do rutynowego eksplorowania.)
- **Claude Sonnet 4.6:** wejście **$3,00** / cache wejścia **$0,30** / zapis do cache **$3,75** / wyjście **$15,00** za 1M. Sonnet jest powszechnie opisywany jako bardziej zdyscyplinowany w nieprzeczytywaniu zbyt wiele i dobrze radzi sobie z długimi sesjami agenta (do ~1M okna w Copilocie).
- **Automatyczny wybór modelu (Auto)** (GA w JetBrains) daje „10% rabatu na koszty modelu przy automatycznym wyborze" i kieruje tańsze modele do prostej pracy.
- **Zalecenie:** do rutynowych, dobrze zawężonych edycji używaj tańszego modelu (GPT-5.4 mini do nawigacji/pracy w stylu grep), a GPT-5.4/Sonnet zarezerwuj na architekturę. Jeśli re-czytanie to Twój główny ból, przetestuj **Claude Sonnet 4.6** do wielo-plikowych zadań agenta.

## Szczegóły — pliki gotowe do wklejenia

### 1. `.github/copilot-instructions.md` (mapa projektu + protokół memory-bank)
```markdown
# Copilot Instructions — <service-name> (Spring Boot 3 / Java 17 / DynamoDB)

## What this service does
<one paragraph: the bounded context, its responsibility, who calls it>

## Tech stack
- Java 17, Spring Boot 3.x, Maven/Gradle
- Persistence: AWS DynamoDB (Enhanced Client / SDK v2)
- Infra: separate AWS CDK project (see ../<cdk-repo>) — do NOT edit infra from this repo
- Migration context: replacing a legacy PostgreSQL app; some domain rules come from it

## Project map — WHERE THINGS LIVE (read this before searching)
- `src/main/java/.../api/`         REST controllers, request/response DTOs
- `src/main/java/.../domain/`      domain model + business rules
- `src/main/java/.../persistence/` DynamoDB repositories, table mappers
- `src/main/java/.../config/`      Spring config, DynamoDB client beans
- `src/test/java/...`              JUnit 5 + Testcontainers / DynamoDB Local
- Key entry points: <MainApplication.java>, <list 3–5 central classes>

## Conventions
- Constructor injection only; no field @Autowired
- Repository pattern for all DynamoDB access; no SDK calls outside `persistence/`
- DynamoDB keys: <PK/SK naming convention>; single-table vs multi-table: <state it>
- Errors via <GlobalExceptionHandler>; never leak SDK exceptions to the API

## Boundaries (always / ask first / never)
- NEVER edit the CDK project or generated sources
- ASK before changing table schema / GSIs
- ALWAYS run `<build+test command>` before declaring done

## MEMORY BANK PROTOCOL (continuity across sessions — IMPORTANT)
At the START of every task you MUST read these files before doing anything else:
- `memory-bank/activeContext.md`  (current focus, next steps, open decisions)
- `memory-bank/progress.md`       (what works, what's left, known issues)
Use them as the source of truth. Do NOT re-scan the whole repo if the answer is here.
Before you FINISH a task, you MUST update `activeContext.md` and `progress.md`
with what changed, decisions made, and the next step. Keep entries short and dated.
If a memory-bank file is missing, create it.
```

### 2. `memory-bank/activeContext.md` (zalążek)
```markdown
# Active Context
_Last updated: <date>_
## Current focus
<the migration slice you're on, e.g., "port Orders read path from PG to DynamoDB">
## Recent changes
- <bullet>
## Next steps
- <bullet>
## Open decisions / questions
- <bullet>
```

### 3. `memory-bank/progress.md` (zalążek)
```markdown
# Progress
_Last updated: <date>_
## Done / works
- <bullet>
## Left to build
- <bullet>
## Known issues / landmines
- <bullet>
## Migration mapping (PostgreSQL → DynamoDB)
- <entity> : <PG table> → <DynamoDB table + access pattern>
```

### Ustawienia JetBrains do zmiany
- **Settings → Languages & Frameworks → GitHub Copilot → Customizations:** potwierdź, że `copilot-instructions.md` i `AGENTS.md` są włączone i się ładują.
- **Agent → max requests per turn:** zostaw 25 do normalnej pracy; podnoś do 100 tylko przy długich zadaniach migracyjnych.
- **Wybór modelu:** ustaw **Auto** do rutynowej pracy (10% rabatu); świadomie wybieraj **Claude Sonnet 4.6** lub **GPT-5.4** do dużych, wielo-plikowych zadań; obniżaj „thinking effort" przy prostych zadaniach.
- **MCP Servers** (ikona Copilota → Edit settings): włącz tylko to, czego używasz; opcjonalnie dodaj serwer pamięci MCP.
- **Załączniki kontekstu:** przed każdym zadaniem agenta dołączaj tylko istotne pliki; usuń auto-dołączony otwarty plik, jeśli jest nieistotny.
- **Administrator Enterprise:** włącz politykę indeksowania repozytoriów; skonfiguruj content exclusion dla `target/`, generowanych źródeł, logów, sekretów.

## Zalecenia (etapami)
1. **Dzisiaj (największy zysk, zerowy koszt):** Utwórz `.github/copilot-instructions.md` z mapą projektu + protokołem memory-bank powyżej, plus zalążkowe `memory-bank/activeContext.md` i `progress.md`. Każ agentowi czytać je najpierw i aktualizować na końcu. To samo w sobie usuwa większość ponownego odkrywania całego repo, a zawsze wstrzykiwany tekst jest rabatowany przez cache.
2. **W tym tygodniu:** Przyjmij workflow per-zadanie — nowy czat na zadanie, tryb Plan, żeby raz front-loadować research, dołączaj konkretne pliki, aktualizuj memory-bank przed zamknięciem. Rutynową pracę przełącz na Auto / GPT-5.4 mini.
3. **Jeśli wciąż upływają kredyty:** Dodaj serwer pamięci MCP do faktów między sesjami; przytnij narzędzia MCP; zastosuj content exclusion Enterprise; przetestuj A/B Claude Sonnet 4.6 vs GPT-5.4 do zadań agenta.
4. **Jeśli JetBrains mimo wszystko nadal re-czyta:** Ciężkie sesje agenta prowadź w **VS Code** (semantyczny indeks `#codebase`, `/compact`, Chat Debug View dla widoczności cache, pełna obsługa plików promptów), a codzienne kodowanie zostaw w IntelliJ.

**Progi zmieniające plan:** Jeśli Twoje miesięczne spalanie AI-kredytów z jednego repo jest zdominowane przez sesje agenta (widać to w dashboardzie rozliczeń GitHub), priorytetyzuj krok 4 (zmiana edytora) i Sonnet. Jeśli plik instrukcji przekracza ~1 000 linii, podziel go na pliki `*.instructions.md` o zasięgu ścieżkowym, żeby ładowały się tylko wtedy, gdy są istotne, a nie przy każdym żądaniu.

## Zastrzeżenia
- Copilot rozwija się szybko; funkcje pluginu JetBrains (parytet indeksowania, `/memory`, pliki promptów) zmieniają się co miesiąc — weryfikuj w Settings → Customizations i w GitHub Changelog dla JetBrains.
- Natywna funkcja „Copilot Memory" jest dziś **tylko dla cloud-agenta / CLI / code review**; nie zakładaj, że utrwala kontekst Twojego agenta w IDE.
- Niektóre blogi zewnętrzne twierdzą, że JetBrains ma pełny parytet z VS Code, włącznie z wyszukiwaniem semantycznym; **oficjalna dokumentacja GitHub tego nie potwierdza** — traktuj twierdzenia o parytecie jako niezweryfikowane.
- Content exclusion nie jest w pełni honorowany w trybie agenta (udokumentowane ograniczenie); jako zapas używaj `.gitignore` i nie otwieraj wykluczonych plików.
- Dokładne liczby okna kontekstu GPT-5.4 różnią się w zależności od konfiguracji (272K / 400K / 1M podawane w różnych miejscach); praktyczny wniosek — duże okno + autonomiczna pętla = więcej re-czytań — pozostaje aktualny niezależnie od tego.
