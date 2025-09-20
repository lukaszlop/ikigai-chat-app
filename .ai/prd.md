# Dokument wymagań produktu (PRD) - IkigAI Chat App (MVP)

## 1. Przegląd produktu
IkigAI Chat App (MVP) to lekka aplikacja webowa umożliwiająca prowadzenie rozmów z modelem językowym oraz dołączanie plików multimedialnych i tekstowych w obrębie jednej konwersacji. Produkt ma zademonstrować kluczowe możliwości: streamowanie odpowiedzi, obsługę załączników, prostą personalizację profilu i mockowaną autentykację.

Zakres MVP koncentruje się na doświadczeniu pojedynczego użytkownika w środowisku deweloperskim, bez trwałej bazy danych i bez hostingu produkcyjnego. Aplikacja ma UI w języku angielskim; model dostosowuje język odpowiedzi do języka użytkownika. Persona asystenta: przyjazny, pomocny psycholog, udzielający klarownych, punktowanych odpowiedzi.

Główne technologie i założenia:
- Model/dostawca: OpenAI gpt-4o-mini (vision + tekst) via Vercel AI SDK `streamText()`.
- Streaming odpowiedzi w czasie rzeczywistym z skeletonem do TTFT.
- Przechowywanie lokalne: sessionStorage (historia czatu, stan UI), localStorage (profil).
- Budżet kosztowy: limit dzienny 0,5 USD z twardą blokadą po przekroczeniu.
- Desktop-first, podstawowa obsługa mobile; a11y standardowe.

## 2. Problem użytkownika
Użytkownicy potrzebują prostego, zintegrowanego narzędzia do interakcji z zaawansowanymi modelami AI, które łączy rozmowę tekstową z możliwością dołączania i analizowania plików (obrazy, dokumenty). Obecne rozwiązania są rozproszone, brakuje im intuicyjnego UX dla multimediów lub płynnego streamingu odpowiedzi, co obniża efektywność i satysfakcję użytkownika.

## 3. Wymagania funkcjonalne
3.1. Autentykacja i dostęp
- Mockowane logowanie: stałe dane testowe (email/login: `test@example.com`, password: `password123`).
- Po sukcesie: zapis stanu zalogowania w sessionStorage; przekierowanie do czatu.
- Brak rejestracji, resetu hasła, zarządzania kontem.
- Dostęp do czatu i profilu wyłącznie po zalogowaniu w ramach bieżącej sesji.

3.2. Nawigacja
- Podstawowa nawigacja Chat ↔ Profile.
- Wskazanie aktywnej sekcji; powrót do czatu z profilu bez utraty stanu konwersacji.

3.3. Czat z AI i streaming
- Integracja z `streamText()`; odpowiedzi streamowane w czasie rzeczywistym.
- Skeleton w bąbelku odpowiedzi do czasu pierwszego fragmentu (TTFT), bez spinnera typing.
- Auto-scroll z zatrzymaniem po interakcji użytkownika (np. przewinięcie w górę).
- Sterowanie: Stop (zachowuje dotychczasowy fragment; brak Continue), Retry (po błędzie), Regenerate (nowa odpowiedź na bazie tej samej wiadomości).
- Kontekst: okno do ostatnich ~6 wiadomości + skróty starszych.
- Limity odpowiedzi: 512 tokenów (tekst) / 768 tokenów (z plikami).

3.4. Wiadomości i limity treści
- Limit treści tekstu w polu wiadomości: do 10 000 znaków; licznik 0/10 000.
- Limit łączny treści: do 30 000 znaków (tekst + zawartość `txt`/`md`).
- Przy przekroczeniu któregokolwiek limitu: twarda blokada wysyłki i czytelny komunikat.
- Definicja liczenia znaków w MVP: długość łańcucha znaków w JS (UTF-16 code units; nowe linie liczone jako znaki). Kwestie emoji/znaków złożonych mogą skutkować minimalnymi odchyleniami.

3.5. Załączniki
- Obsługiwane typy: `jpg`, `png`, `txt`, `md`.
- Limity: maks. 3 pliki/wiadomość; maks. 5 MB/pliki; 15 MB łącznie na wiadomość.
- Dodawanie: drag & drop i paste (np. `react-dropzone`) oraz selektor plików.
- Podgląd: miniatury dla obrazów; dla `txt`/`md` skrót (nazwa i rozmiar). Dla `md` w podglądzie tylko nazwa i rozmiar.
- Usuwanie: możliwość usunięcia załączników przed wysyłką.
- Mieszane pliki w jednej wiadomości dozwolone.
- Strukturyzowanie promptu: dla każdego załącznika dodawany krótki nagłówek "Attachment N: …".
- `txt`/`md`: treść dołączana do promptu i liczona do limitu 30 000.

3.6. Przetwarzanie obrazów po stronie serwera
- Normalizacja: auto-orient, maksymalny dłuższy bok 2048 px.
- Konwersja: WEBP q≈80; PNG dla treści z tekstem/przezroczystością.
- Prywatność: usuwanie metadanych EXIF.
- Brak dodatkowej kompresji po stronie klienta.

3.7. Linki, markdown i kod
- Automatyczne wykrywanie linków w odpowiedziach; otwieranie w nowej karcie z bezpiecznymi atrybutami.
- W środowisku deweloperskim subtelne ostrzeżenie dla URL.
- Render markdown; bloki kodu z podświetlaniem i przyciskiem "Copy".

3.8. Tryb offline
- Wysyłka blokowana w trybie offline z komunikatem: "You’re offline… Please reconnect and retry.".
- Po odzyskaniu sieci możliwość ponownego wysłania.

3.9. Profil użytkownika
- Pola: Name (edytowalne), Email (read-only, równy loginowi), Avatar (okrągły).
- Avatar: upload typów `jpg`/`png`/`webp`, min 256×256 px, maks. 2 MB; crop 1:1 po stronie klienta; podgląd i zapis.
- Zapis profilu w localStorage.

3.10. Sesja i przechowywanie
- Brak timeoutu sesji w MVP.
- Historia czatu i stan UI w sessionStorage (przetrwa odświeżenie karty/przeglądarki).
- Brak trwałego przechowywania historii czatu poza sesją; brak wielu wątków.

3.11. Budżet kosztowy
- Dzienne ograniczenie kosztów: 0,5 USD/dzień; po osiągnięciu limitu twarda blokada wysyłki z komunikatem.
- Globalny limit tokenów/kosztów dla modelu; prosty licznik zużycia.
- Reset doby: do potwierdzenia (wstępnie o północy czasu systemowego przeglądarki).

3.12. Dostępność i język
- Standardowe wymagania a11y dla elementów interfejsu i focus management.
- UI w języku angielskim; model dostosowuje język odpowiedzi do języka użytkownika.

3.13. Środowisko i hosting
- Brak planu hostingu produkcyjnego na etapie MVP; środowisko deweloperskie.
- Opcjonalna lokalna strona `/metrics` (jeśli nie skomplikuje MVP) do przeglądu TTFT, czasu odpowiedzi, błędów, upload success, kosztu/wiadomość.

## 4. Granice produktu
4.1. W zakresie MVP
- Mockowana autentykacja i bramkowanie dostępu do czatu/profilu.
- Jedna ciągła konwersacja z AI z obsługą załączników i streamingiem.
- Edycja prostego profilu (Name, Avatar; Email read-only).
- Local/session storage zgodnie z opisem.

4.2. Poza zakresem MVP
- Prawdziwy system kont (rejestracja, reset hasła, trwała baza użytkowników).
- Trwałe przechowywanie historii czatu i wielu wątków.
- Zaawansowane zarządzanie plikami (biblioteka mediów, usuwanie wysłanych plików, rzadkie formaty).
- Funkcjonalność speech-to-text.
- Zmiana parametrów modelu (np. temperatura) z UI.

4.3. Założenia
- Użytkownik akceptuje ograniczenia typów/rozmiarów plików i limitów treści.
- Jednoosobowe użytkowanie w ciągu dnia (budżet 0,5 USD/dzień).
- Desktop-first; mobile wspierane w podstawowym zakresie.

4.4. Otwarte kwestie (do doprecyzowania)
- Docelowe SLO (TTFT, średni czas odpowiedzi) i wyjątki dla złożonych plików.
- Strona `/metrics`: czy wchodzi do MVP.
- Metoda przeliczania tokenów/kosztu dla gpt-4o-mini (stawki, margines bezpieczeństwa, reset doby).
- Minimalne wersje przeglądarek/OS (wstępnie: ostatnie stabilne wersje Chrome, Firefox, Safari i Edge na desktopie).
- Schemat ewentów analitycznych i miejsce logowania (lokalnie vs. brak w MVP).
- Precyzja liczenia znaków do limitu 30 000 (znaki vs. bajty; nowe linie/emoji) – w MVP stosujemy definicję z sekcji 3.4.
- Finalna treść komunikatów budżet/offline/błędy (angielski UI; język odpowiedzi modelu według użytkownika).

## 5. Historyjki użytkowników
US-001
Tytuł: Logowanie mock – poprawne dane
Opis: Jako użytkownik chcę zalogować się przy użyciu stałych danych testowych, aby uzyskać dostęp do czatu i profilu.
Kryteria akceptacji:
- Given ekran logowania, When wpiszę `test@example.com` i `password123`, Then widzę potwierdzenie sukcesu i przejście do czatu.
- Stan zalogowania zapisany w sessionStorage do końca sesji.

US-002
Tytuł: Bramka dostępu – brak sesji
Opis: Jako niezalogowany użytkownik nie powinienem móc wejść na czat/profil, aby chronić dostęp.
Kryteria akceptacji:
- Given brak sesji, When otworzę strony czatu/profilu, Then następuje przekierowanie do ekranu logowania.

US-003
Tytuł: Nawigacja Chat ↔ Profile
Opis: Jako użytkownik chcę przełączać się między czatem a profilem bez utraty stanu konwersacji.
Kryteria akceptacji:
- Given zalogowany użytkownik, When przejdę do profilu i wrócę do czatu, Then historia i stan inputu pozostają bez zmian.

US-004
Tytuł: Wczytanie historii z sessionStorage
Opis: Jako użytkownik po odświeżeniu karty chcę widzieć dotychczasową rozmowę i stan UI.
Kryteria akceptacji:
- Given istniejące wpisy w sessionStorage, When odświeżę stronę czatu, Then wiadomości i stan interfejsu są przywracane.

US-005
Tytuł: Edycja wiadomości z licznikiem znaków
Opis: Jako użytkownik chcę widzieć licznik znaków do 10 000, aby kontrolować długość wiadomości.
Kryteria akceptacji:
- Given pole wiadomości, When wpisuję tekst, Then licznik aktualizuje się w formacie X/10 000.

US-006
Tytuł: Blokada >10 000 znaków w polu wiadomości
Opis: Jako użytkownik nie mogę wysłać wiadomości przekraczającej 10 000 znaków.
Kryteria akceptacji:
- Given tekst >10 000 znaków, When kliknę Wyślij, Then przycisk jest zablokowany i widzę komunikat o przekroczeniu limitu.

US-007
Tytuł: Dodanie pliku przez drag & drop
Opis: Jako użytkownik chcę przeciągnąć plik do obszaru wiadomości i dodać go jako załącznik.
Kryteria akceptacji:
- Given obszar DnD, When upuszczę obsługiwany plik, Then widzę go na liście załączników.

US-008
Tytuł: Dodanie pliku przez paste
Opis: Jako użytkownik chcę wkleić obraz/plik do inputu i dodać go.
Kryteria akceptacji:
- Given schowek z obrazem/plikem, When użyję paste, Then plik pojawia się jako załącznik.

US-009
Tytuł: Podgląd załączników
Opis: Jako użytkownik chcę widzieć miniatury obrazów oraz nazwy i rozmiary plików txt/md.
Kryteria akceptacji:
- Given lista załączników, When dodam `jpg/png`, Then widzę miniaturę; When dodam `txt/md`, Then widzę nazwę i rozmiar.

US-010
Tytuł: Usuwanie załącznika przed wysyłką
Opis: Jako użytkownik chcę usunąć wybrany załącznik przed wysłaniem wiadomości.
Kryteria akceptacji:
- Given lista załączników, When kliknę usuń przy pliku, Then plik znika z listy.

US-011
Tytuł: Walidacja liczby i rozmiaru plików
Opis: Jako użytkownik nie powinienem móc dodać >3 plików, pliku >5 MB ani przekroczyć 15 MB łącznie.
Kryteria akceptacji:
- Given 3 pliki już dodane, When próbuję dodać kolejny, Then otrzymuję komunikat o limicie.
- Given plik >5 MB lub suma >15 MB, When próbuję dodać/wysłać, Then blokada i komunikat.

US-012
Tytuł: Wysyłka mieszanych plików
Opis: Jako użytkownik mogę dodać różne typy (obrazy + txt/md) w jednej wiadomości.
Kryteria akceptacji:
- Given mieszane pliki, When wyślę, Then wysyłka jest poprawna i każdy załącznik otrzymuje nagłówek "Attachment N: …" w prompcie.

US-013
Tytuł: Dołączanie treści txt/md do promptu
Opis: Jako użytkownik oczekuję, że zawartość `txt/md` zostanie dołączona do promptu i policzona do łącznego limitu 30 000.
Kryteria akceptacji:
- Given pliki `txt/md`, When wyślę, Then treść jest w promopcie; licznik/limit 30 000 uwzględnia te treści.

US-014
Tytuł: Blokada >30 000 znaków łącznie
Opis: Jako użytkownik nie mogę wysłać wiadomości (tekst + txt/md) >30 000 znaków.
Kryteria akceptacji:
- Given suma znaków >30 000, When kliknę Wyślij, Then blokada i czytelny komunikat o limicie.

US-015
Tytuł: Normalizacja obrazów na serwerze
Opis: Jako użytkownik chcę, aby obrazy były automatycznie orientowane, skalowane i konwertowane oraz by EXIF został usunięty.
Kryteria akceptacji:
- Given obraz, When wyślę, Then serwer zwraca wersję z auto-orient, max 2048 px, WEBP q≈80 (lub PNG dla tekstu/przezroczystości), bez EXIF.

US-016
Tytuł: Streaming odpowiedzi z skeletonem i auto-scrollem
Opis: Jako użytkownik chcę widzieć reakcję modelu na żywo oraz automatyczne przewijanie do nowej odpowiedzi.
Kryteria akceptacji:
- Given wysłana wiadomość, When model zaczyna odpowiadać, Then pojawia się skeleton do TTFT, a następnie strumień tekstu; auto-scroll działa do czasu ręcznej interakcji użytkownika.

US-017
Tytuł: Stop streamingu i Regenerate
Opis: Jako użytkownik chcę zatrzymać generowanie i ewentualnie wygenerować odpowiedź od nowa.
Kryteria akceptacji:
- Given odpowiedź w toku, When kliknę Stop, Then fragment pozostaje w czacie; brak opcji Continue.
- Given zatrzymana/ukończona odpowiedź, When kliknę Regenerate, Then powstaje nowa odpowiedź dla tej samej wiadomości.

US-018
Tytuł: Retry po błędzie
Opis: Jako użytkownik chcę ponowić żądanie w przypadku błędu bez zmiany treści wiadomości.
Kryteria akceptacji:
- Given błąd żądania, When kliknę Retry, Then żądanie jest ponowione; w razie kolejnego błędu widzę komunikat z możliwością kolejnego Retry.

US-019
Tytuł: Linki w odpowiedziach
Opis: Jako użytkownik chcę, aby wykryte linki otwierały się w nowej karcie i były bezpieczne.
Kryteria akceptacji:
- Given odpowiedź z URL, When kliknę link, Then otwiera się nowa karta z atrybutami bezpieczeństwa; w dev widoczne subtelne ostrzeżenie.

US-020
Tytuł: Bloki kodu z podświetlaniem i Copy
Opis: Jako użytkownik chcę czytelnie sformatowanych bloków kodu i przycisku kopiowania.
Kryteria akceptacji:
- Given odpowiedź zawiera markdown code blocks, When najadę/kliknę Copy, Then zawartość jest skopiowana do schowka.

US-021
Tytuł: Obsługa offline
Opis: Jako użytkownik w trybie offline nie mogę wysłać wiadomości, a aplikacja komunikuje ten stan.
Kryteria akceptacji:
- Given brak sieci, When kliknę Wyślij, Then widzę "You’re offline… Please reconnect and retry." i wysyłka jest zablokowana.

US-022
Tytuł: Limit dzienny kosztów
Opis: Jako właściciel budżetu chcę, aby wysyłka została zablokowana po przekroczeniu 0,5 USD/dzień.
Kryteria akceptacji:
- Given licznik kosztów ≥0,5 USD, When próbuję wysłać, Then blokada i komunikat o przekroczonym limicie.
- Given nowy dzień, When otworzę aplikację, Then limit jest zresetowany (szczegóły do potwierdzenia).

US-023
Tytuł: Kontekst ostatnich ~6 wiadomości
Opis: Jako użytkownik chcę, aby model brał pod uwagę ostatnią część rozmowy, aby odpowiedzi były trafne.
Kryteria akceptacji:
- Given długa rozmowa, When wyślę kolejną wiadomość, Then do promptu trafia ok. 6 ostatnich wiadomości oraz skróty starszych.

US-024
Tytuł: Ograniczenie długości odpowiedzi
Opis: Jako użytkownik oczekuję, że odpowiedzi nie będą nadmiernie długie i nie będą generować zbędnych kosztów.
Kryteria akceptacji:
- Given odpowiedź bez plików, When model generuje, Then limit 512 tokenów jest respektowany; z plikami – 768.

US-025
Tytuł: Zapis sesji czatu i stanu UI
Opis: Jako użytkownik chcę, aby po odświeżeniu karty zachować rozmowę i stan interfejsu.
Kryteria akceptacji:
- Given aktywna sesja, When odświeżę stronę, Then historia rozmowy i stan UI są przywrócone z sessionStorage.

US-026
Tytuł: Profil – edycja Name, Email read-only
Opis: Jako użytkownik chcę edytować swoje imię, a e-mail ma być tylko do odczytu.
Kryteria akceptacji:
- Given profil, When zmienię Name i zapiszę, Then wartość trafia do localStorage i jest widoczna w UI.
- Email jest nieedytowalny i równy loginowi.

US-027
Tytuł: Upload i crop avatara 1:1
Opis: Jako użytkownik chcę wgrać avatar, przyciąć go do koła (1:1) i zapisać.
Kryteria akceptacji:
- Given obraz `jpg/png/webp` ≥256×256 i ≤2 MB, When wgram i dokonam cropu 1:1, Then zapis w localStorage i odświeżony podgląd.
- Given nieobsługiwany typ/wymiary/rozmiar, When wgram, Then widzę komunikat i brak zapisu.

US-028
Tytuł: Dostępność i focus management
Opis: Jako użytkownik korzystający z klawiatury/AT oczekuję poprawnych etykiet, focusów i kontrastu.
Kryteria akceptacji:
- Wszystkie kontrolki mają etykiety, focus nie ginie podczas streamingu, kontrast zgodny z a11y w podstawowym zakresie.

US-029
Tytuł: Język UI i odpowiedzi modelu
Opis: Jako użytkownik oczekuję UI po angielsku oraz odpowiedzi modelu w języku, w którym piszę.
Kryteria akceptacji:
- Given UI, When zmienię język komunikacji w wiadomości (np. PL/EN), Then model dostosowuje język odpowiedzi.

US-030
Tytuł: Bezpieczny dostęp po zalogowaniu
Opis: Jako użytkownik chcę mieć pewność, że dane czatu/profilu nie są dostępne bez ważnej sesji.
Kryteria akceptacji:
- Given wyczyszczona sessionStorage, When otworzę stronę czatu/profilu, Then następuje redirect do logowania; bez możliwości obejścia przez bezpośredni URL.

## 6. Metryki sukcesu
6.1. Kryteria sukcesu MVP
- Użytkownik wykonuje scenariusz E2E: logowanie → wysłanie wiadomości z plikiem → merytoryczna odpowiedź odnosząca się do załącznika.
- Streaming działa płynnie w 100% sesji MVP na dev (brak przerwań aplikacyjnych).
- Upload success rate ≥95% dla `jpg/png/txt/md` w typowych warunkach sieci.
- Brak przekroczeń dziennego budżetu 0,5 USD; prawidłowe blokowanie po osiągnięciu limitu.

6.2. Proponowane SLO (do potwierdzenia)
- TTFT ≤ 1,5 s (tekst) / ≤ 2,0 s (z plikiem).
- Średni czas odpowiedzi ≤ 10 s (tekst) / ≤ 20 s (z plikiem).

6.3. Metody pomiaru
- Ręczne testy E2E i smoke tests dla podstawowych ścieżek.
- Opcjonalna lokalna strona `/metrics`: wykresy/metryki TTFT, czas odpowiedzi, error rate, upload success, koszt/wiadomość (bez PII).
- Logika liczenia kosztów/tokenów z marginesem bezpieczeństwa; reset dzienny zgodnie z ustaleniami.

6.4. Gotowość do wdrożenia kolejnych iteracji
- Zamknięte otwarte kwestie z sekcji 4.4 lub zaplanowane w backlogu.
- Pokrycie testami kryteriów akceptacji kluczowych historyjek (US-001–US-030).
