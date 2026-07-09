# EMS – Energy Management System for Home Assistant

Zaawansowany system zarządzania energią zbudowany w Home Assistant, sterujący ładowaniem i rozładowaniem magazynu energii oraz sprzedażą energii do sieci. Decyzje podejmowane są w czasie rzeczywistym na podstawie prognozy PV (Solcast), historycznego zużycia (SQL), aktualnych cen sprzedaży RCE i warunków pogodowych. System jest zoptymalizowany pod taryfę G12 i model prosumencki z bilansowaniem godzinowym.

**Nomenklatura cen:** RCE oznacza cenę sprzedaży/eksportu energii do sieci i jest używane przez EMS do decyzji eksportowych. Import/zakup w EMS v1 jest liczony według statycznych stawek G12 (`var.magazyn_koszt_calkowity_strefa_*`). RDN to rynek/cena zakupu spot i w tym systemie występuje tylko poglądowo; dynamiczny import po RDN jest zakresem EMS2.

---

## W skrócie

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DOBOWY CYKL EMS                                 │
│                                                                         │
│  22:00        03:00–06:00      06:00–13:00     13:00–15:00  15:00–22:00 │
│  ┌──────┐     ┌───────────┐   ┌───────────┐   ┌─────────┐  ┌─────────┐ │
│  │ NOC  │     │   RANO    │   │  EKSPORT  │   │POŁUDNIE │  │EKSPORT  │ │
│  │      │     │           │   │  PORANNY  │   │         │  │WIECZORNY│ │
│  │Plan  │     │Ile naładow│   │PV spill / │   │Ile dołado│  │BAT→sieć │ │
│  │na noc│     │z sieci G12│   │BAT→sieć?  │   │pod eksport│  │przy cenie│ │
│  │      │     │do 13:00?  │   │co 15 min  │   │wieczorny│  │RCE > próg│ │
│  └──────┘     └───────────┘   └───────────┘   └─────────┘  └─────────┘ │
│                                                                         │
│  Dane wejściowe: Solcast (PV) · SQL (zużycie 7d) · RCE (ceny) · pogoda │
│  Ochrona:  bilans godzinowy · SOC floor · LOWPV · domykacz HH:45        │
└─────────────────────────────────────────────────────────────────────────┘
```

| | Falownik | Magazyn | Taryfa | Baza danych |
|---|---|---|---|---|
| **Wymagane** | Deye / Solarman | dowolna (Deye) | G12 | MariaDB |

---

## Wymagania

### Sprzęt i instalacja

| Element | Wymaganie |
|---|---|
| Falownik | **Deye** (hybrydowy) – sterowanie przez integrację `solarman` (SolarmanPV API v5) |
| Magazyn energii | Bateria podłączona do falownika Deye; pojemność konfigurowana w `var.magazyn_pojemnosc_brutto_kwh` (domyślnie 15 kWh) |
| Taryfa OSD | **G12** – z wyraźnym podziałem na strefę droższą (G2) i tańszą (G1): 22:00–06:00 oraz 13:00–15:00 jako tańsza |

### Integracje Home Assistant (wymagane)

| Integracja | Zastosowanie |
|---|---|
| [`solarman`](https://github.com/StephanJoubert/home_assistant_solarman) | Odczyty z falownika + sterowanie trybami (HACS) |
| [`solcast-solar`](https://github.com/BJReplay/ha-solcast-solar) | Prognoza PV – atrybut `detailedForecast` (sloty 30-min); wymagany darmowy token API |
| [`pirateweather`](https://github.com/alexander0042/pirate-weather-ha) | Prognoza pogody – korekta minimalnego SOC rano (HACS) |
| [`rce_prices`](https://github.com/AdamWeglarz/rce_prices) | Sensor cen sprzedaży RCE z VAT: `sensor.rce_prices_today_scaled` / `tomorrow_scaled` |
| [`snarky-snark/home-assistant-variables`](https://github.com/snarky-snark/home-assistant-variables) | Zmienne `var.*` – konfigurowalne parametry EMS (HACS) |

### Kalendarze (opcjonalne, modyfikują planowanie)

- `calendar.urlop` – obniża planowane zużycie do 0,5 kWh/h
- `calendar.sprzatanie` – podnosi planowane zużycie do SQL × 2,0

### Baza danych – **MariaDB (wymagana)**

System intensywnie korzysta z zapytań SQL po historii zużycia (średnie godzinowe i 30-minutowe z ostatnich dni). **SQLite (domyślna baza HA) nie jest obsługiwana** – brakuje wymaganych funkcji (`CONVERT_TZ`, `DECIMAL`, wydajność).

> ⚠️ **Uwaga:** Migracja z domyślnego rekordera SQLite do MariaDB **usuwa całą historię encji**. Zanim przejdziesz na MariaDB, zrób backup – historia sensorów zostanie utracona i system będzie potrzebował kilku dni do zgromadzenia danych SQL potrzebnych do poprawnego planowania.

Wymagana konfiguracja w `configuration.yaml`:

```yaml
recorder:
  db_url: mysql://ha_user:password@core-mariadb/homeassistant?charset=utf8mb4
```

---

## Lista funkcjonalności

- **Automatyczne ładowanie z sieci w oknie G12 RANO (02:00–06:00)** – precyzyjne wyznaczanie ilości energii do załadowania
- **Automatyczne ładowanie z sieci w oknie G12 POŁUDNIE (13:00–15:00)** – doładowanie przed wieczornym eksportem
- **Eksport PV (oddawanie nadwyżki fotowoltaiki) 06:00–13:00** – 4 tryby od PV-only do pełnego rozładowania baterii
- **Eksport wieczorny z baterii 15:00–22:00** – sprzedaż przy korzystnych cenach RCE
- **Blokada nocna (22:00)** – planowanie na całą noc i ochrona przed zbyt głębokim rozładowaniem
- **Tryb LOWPV** – progi niskiej prognozy PV bez wymuszania 100%; popołudniu utrzymuje wyższy SOC na koniec okna
- **Bilansowanie godzinowe** – przestrzeganie zasad rozliczenia prosumenckiego (nie oddawaj energii, którą odkupisz tej samej godziny)
- **Korekta SOC na podstawie pogody** – falownik rano (05:29) koryguje minimalny SOC zależnie od zachmurzenia
- **Obsługa kalendarzy** – automatyczna korekta planowanego zużycia podczas urlopu i sprzątania
- **Raporty finansowe (Actual vs Counterfactual)** – śledzenie oszczędności i przychodów w czasie rzeczywistym
- **Sensory stabilizujące odczyty falownika** – filtrowanie skoków i `unknown`/`unavailable`
- **Pełna konfigurowalność** – wszystkie parametry jako `var.*` edytowalne z UI bez zmiany kodu
- **Optymalny start AGD (pralka, suszarka)** – scheduler EMS-aware uruchamia urządzenia w najtańszym/najbardziej zielonym momencie dnia na podstawie spill PV i cen RCE

---

## Architektura – pliki packages

```
packages/
├── zmienne_zarzadzanie_pv.yaml   # Wszystkie parametry konfiguracyjne (var, input_boolean, input_number)
├── solarmansafe.yaml             # Sensory "safe" – stabilizacja odczytów falownika + statistics + min_max
├── sensors_sql_pv.yaml           # SQL: średnie zużycie godzinowe/30-min + energia oddana (slot 15min, 6-13, 15-22)
├── automations_magazyn.yaml      # Główne automatyzacje: RANO (02-06) i POŁUDNIE (13-15)
├── magazyn_nowyeksport.yaml      # Eksport: oddawanie poranne (06-13), wieczorne (15-22/23), blokada 22/23
├── magazynlimity.yaml            # Bilans eksportu, przeniesiona energia, sensory pomocnicze
├── finanse_pv.yaml               # Śledzenie oszczędności i przychodów ze sprzedaży
└── ems_agd.yaml                  # Scheduler AGD: optymalny start pralki i suszarki
```

---

## Tryby pracy falownika

System operuje na 3 trybach sterowanych przez `input_boolean`:

| Flaga | Tryb falownika | Zastosowanie |
|---|---|---|
| `battery_charge_from_grid` | Grid Charging (wszystkie 6 programów = Grid, docelowy SOC, prąd ładowania = 150A) | Nocne ładowanie z sieci |
| `pv_discharge_to_grid` | Export First, prąd rozładowania = 0 | Oddawanie nadwyżki PV (bez baterii) |
| `battery_discharge_to_grid` | Export First, prąd rozładowania = 150A | Oddawanie energii z baterii do sieci |

Zmiana dowolnej flagi wyzwala automatyzację `pv_battery_grid_modes_profile_sync_v2`, która synchronizuje parametry Solarmana (tryb pracy, 6 programów, SOC, prąd ładowania/rozładowania) z minimalną liczbą wywołań API.

Sensor `sensor.solarman_mode_status` pokazuje aktualnie rozpoznany tryb (np. _"Ładuje z Grid"_, _"Oddaje nadwyżkę PV"_, _"Normalna praca"_).

---

## Planowanie ładowania – okno RANO (02:00–05:45)

**Kiedy działa:** trigery co 30 min między 02:00 a 05:45.

**Cel:** doładować baterię z sieci (taryfa tańsza G12) tak, aby o 13:00 mieć zaplanowany SOC `var.cel_naladowania_o_13` (domyślnie 50%), nie spadając po drodze poniżej `var.magazyn_soc_min_rano_percent`.

**Algorytm:**
1. Pobiera z SQL średnie zużycie dla slotów 30-min (sloty 6–43, tj. 03:00–22:00) z ostatnich 7 dni.
2. Pobiera prognozę PV z Solcast (`detailedForecast`, sloty 30-min, 06:00–13:00 dziś).
3. Symuluje slot po slocie (0..47): aktualna energia w baterii − zużycie + PV.
4. Wyznacza minimalny `limit_soc` potrzebny do dotarcia do 13:00 bez zejścia poniżej `magazyn_soc_floor_percent` (20%).
5. Jeśli prognoza PV < `var.magazyn_lowpv_threshold_rano_kwh` (domyślnie 8 kWh), próg jest raportowany diagnostycznie; cel nadal wynika z planu na 13:00.
6. Jeśli `input_boolean.magazyn_doladowanie_pod_eksport_poranek` = ON → **ETAP 2**: doładowuje pod planowany eksport poranny (patrz niżej).
7. Robi pre-check popołudnia: szacuje wymagane `E15` dla okna 15–22, odejmuje saldo 13–15 oraz `var.magazyn_ladowanie_13_15_capacity_kwh`; jeśli okno 13–15 nie wystarczy, podnosi poranny cel, ale tylko do limitu bez spillu PV.

**Modyfikatory zużycia:**
- `calendar.urlop` aktywny w danej godzinie → z_h = `var.magazyn_konsumpcja_urlop_kwh_h` (domyślnie 0,5 kWh; priorytet nadrzędny)
- Przy aktywnym urlopie poranny plan dodaje stały bufor 1 kWh ponad wyliczony cel `E6_plan`; bufor nie narasta przy kolejnych triggerach 02:00/02:30/03:00/03:30/04:00.
- `calendar.sprzatanie` aktywny → z_h = SQL × `var.magazyn_konsumpcja_mult_sprzatanie` (domyślnie ×2,0)
- Brak kalendarza → z_h = SQL × `var.magazyn_konsumpcja_multiplier` (domyślnie ×1,15)

**Statystyki pomocnicze zużycia godzinnego:**
- `sensor.srednie_zuzycie_godzinne_bez_urlopu_bez_sprzatania` – średnia SQL dla bieżącej godziny bez modyfikatorów kalendarza
- `sensor.srednie_zuzycie_godzinne_z_urlopem` – stała wartość `var.magazyn_konsumpcja_urlop_kwh_h`
- `sensor.srednie_zuzycie_godzinne_ze_sprzataniem` – średnia normalna × `var.magazyn_konsumpcja_mult_sprzatanie`

---

## Planowanie ładowania – okno POŁUDNIE (13:00–15:00)

**Kiedy działa:** trigery co 15–30 min między 13:00 a 14:45.

**Cel:** opcjonalne doładowanie baterii z sieci (taryfa tańsza 13–15), aby przygotować się na wieczorne oddawanie lub zapewnić energię na noc.

**Algorytm:**
1. Pobiera z SQL średnie zużycie dla slotów 30-min (sloty 26–45, tj. 13:00–23:00) z ostatnich 7 dni.
2. Pobiera prognozę PV 13–22/23 (`detailedForecast`, sloty 30-min; odejmuje już wyprodukowaną energię).
3. Symuluje SOC slot po slocie do 22:00, a przy aktywnym planie eksportu po 22 do 23:00.
4. Wyznacza limit doładowania potrzebny do pokrycia zużycia wieczornego z zapasem.
5. Jeśli prognoza skoryg. < `var.magazyn_lowpv_threshold_popoludnie_kwh` (domyślnie 8 kWh), planner wymaga min. `var.magazyn_soc_min_wieczor_lowpv_percent` na koniec okna 22:00/23:00; w dniu LOWPV ustawia ten floor na 30%.
6. Jeśli `input_boolean.magazyn_doladowanie_pod_eksport_wieczor` = ON → **ETAP 2**: doładowuje pod planowany eksport wieczorny (patrz niżej).
   - Sloty eksportowe są rozpoznawane po kandydatach cenowych, a nie po aktualnie dostępnej nadwyżce baterii. Dzięki temu niski SOC o 13:00 nie daje fałszywego `NO_SLOTS`, tylko uruchamia doładowanie pod drogie sloty 15–22.
   - Jeśli slot 22:00–23:00 spełnia próg ceny, zapisywana jest flaga `var.magazyn_plan_export_po_22`, plan ładowania chroni energię domu do 23:00, a nocna blokada przesuwa się z 22:00 na 23:00.

Modyfikatory zużycia jak w oknie RANO.

---

## Eksport poranny (06:00–13:00)

**Plik:** `magazyn_nowyeksport.yaml`
**Trigery:** co 15 minut od 06:00 do 12:45.

**Logika spill (fizyczny overflow PV):**
Eksport ma sens tylko jeśli PV wytworzy więcej niż bateria + zużycie są w stanie wchłonąć.

System wylicza `var.spill_poranny` = prognozowana energia PV od teraz do końca produkcji − pojemność do naładowania − przewidywane zużycie.

**Tryby eksportu (od najtańszego warunkowo):**

| Tryb | Warunek cenowy | Warunek energetyczny |
|---|---|---|
| PV spill | cena RCE > `min_zysk_sprzedaz` | realny spill PV w slocie |
| BAT spill | cena RCE > `koszt_tansza` | realny spill, SOC > `soc_spill_bat_floor` (30%) |
| BAT semi | cena RCE > `koszt_tansza + min_zysk_sprzedaz` | cel min. 50% SOC o 13:00 |
| BAT full | cena RCE > `koszt_drozsza + min_zysk_sprzedaz` | do `min_soc_dynamic_13` |

Dodatkowy floor: `var.magazyn_soc_min_rano_percent` – żaden tryb BAT nie zejdzie poniżej tego SOC.

---

## Eksport wieczorny (15:00–22:00)

**Plik:** `magazyn_nowyeksport.yaml`
**Trigery:** co 15 minut od 15:00 do 21:45.

Analogiczna logika jak eksport poranny, ale dla okna 15–22. Warunek cenowy: cena RCE aktualnego slotu > próg minimalnego zysku.

Automatyzacja POŁUDNIE (13–15) co uruchomienie oblicza **najlepsze sloty cenowe** na wieczór (15–22h) i ewentualnie planuje dodatkowe ładowanie z sieci (`export_topup`), żeby mieć wystarczający SOC do sprzedania zaplanowanej energii. W trybie LOWPV sloty nadal są kalkulowane, ale zamiast celu 100% planner pilnuje min. 30% SOC na koniec okna 22/23.

SOC stop eksportu wieczornego: `var.magazyn_soc_stop_export_wieczor` (domyślnie 20%).
W trybie **FULL** próg uwzględnia prognozowane zużycie do końca bieżącej godziny
oraz dodatkowy bufor: 1 pp. SOC za każdy pełny kwadrans po bieżącym slocie,
minimum 1 pp. W rezultacie dla godziny 19 po slotach 19:00, 19:15, 19:30 i
19:45 pozostaje odpowiednio 3%, 2%, 1% i 1% ponad floor (po uwzględnieniu
energii potrzebnej domowi do 20:00).

Strategiczny eksport jest opcjonalny (`input_boolean.magazyn_rezerwa_pod_eksport_strategiczny`).
Plan 13–15 wyznacza próg `koszt_droższej + min_zysk × mnożnik` (domyślnie ×3),
zapisuje target SOC i sloty spełniające próg. Po 15:00 magazyn utrzymuje target
przez istniejący profil ładowania z sieci tylko, gdy PV nie pokrywa domu. W godzinie
eksportu target nie może przekroczyć aktualnego SOC, a w samym slocie eksportowym
ładowanie z sieci jest wyłączane. Rozpoznanie bieżącego slotu działa na całym
przedziale `HH:MM-HH:MM`, więc strażnik nie wraca do ładowania po pierwszych
5 minutach slotu.

---

## Doładowanie pod eksport (topup)

Funkcja pozwala naładować baterię **więcej niż wymaga samo pokrycie zużycia**, tak aby nadwyżka pojemności mogła zostać sprzedana do sieci w korzystnych slotach RCE. Bateria pokrywa konsumpcję domu, a wyprodukowana energia PV (rano) lub zakupiona energia (wieczorem) idzie bezpośrednio do sieci.

Włączana osobno dla każdego okna:

| Flaga | Okno | Działanie |
|---|---|---|
| `input_boolean.magazyn_doladowanie_pod_eksport_poranek` | RANO (02–06) | Planuje sprzedaż z baterii w slotach 06–11 z ceną ≥ BAT semi / BAT full; doładowuje o brakującą energię |
| `input_boolean.magazyn_doladowanie_pod_eksport_wieczor` | POŁUDNIE (13–15) | Planuje sprzedaż z baterii w slotach 15–22 z ceną ≥ próg; doładowuje o brakującą energię |

**Algorytm (ETAP 2 w obu automatyzacjach):**

1. Skanuje ceny RCE dla okna eksportu i wybiera sloty spełniające progi cenowe (BAT semi: `koszt_tańsza + min_zysk`; BAT full: `koszt_droższa + min_zysk`)
2. Sumuje energię bateryjną potrzebną do realizacji planu (`export_plan_batt_sum`)
3. Oblicza ile z tej energii już jest w baterii po podstawowym ładowaniu (`current_export_budget`)
4. Różnicę (`export_missing`) dodaje do `grid_add` — bateria jest ładowana o tyle więcej z sieci
5. Wynik planu (sloty, ceny, ilości) trafia do notyfikacji

**Zabezpieczenia:**
- Nie przekracza pojemności baterii (`max_ene_u`)
- Respektuje `sensor.aktualny_limit_dzienny` — limit dobowy oddanej energii
- W trybie LOWPV topup nie jest pomijany automatycznie; planner łączy cel eksportowy z wymogiem min. 30% SOC na koniec okna 22/23
- `was_discharging` guard w domykaczu — topup nie koliduje z domykaczem godzinowym

---

## Blokada nocna (22:00)

**Plik:** `magazyn_nowyeksport.yaml`

O 22:00, albo o 23:00 po eksporcie po 22, system planuje noc i następny poranek:
- Pobiera prognozę PV na jutro
- Symuluje zużycie na podstawie statycznych slotów 30-min dla poranka i popołudnia
- Wyznacza czy bieżący SOC wystarczy do 13:00 bez dołowania poniżej `magazyn_soc_blokada_noc_min_percent`
- Liczy preview celu porannego 02:00 tą samą logiką co planner RANO: target 13:00, floor rano, urlop, pre-check popołudnia i poranny export topup
- Jeśli nie – zatrzymuje eksport i przełącza falownik w tryb normalny (Zero Export To CT)

**Blokada rozładowania nocnego:** jeśli preview planera RANO zakłada wyższy cel niż klasyczny nocny booking, limit hold-SOC bierze ten cel pod uwagę, ale nigdy nie ustawia limitu powyżej aktualnego SOC. Dzięki temu noc nie pozwala zjechać poniżej energii, którą planner 02:00 i tak chciałby potem odtwarzać z sieci.

---

## Bilansowanie godzinowe

Zgodnie z prawem energetycznym, prosumenci są rozliczani w bilansowaniu **godzinowym** (pełna godzina zegarowa). System uwzględnia to przy planowaniu eksportu:

- Przed oddaniem energii w danej godzinie wylicza przewidywane zużycie do końca tej godziny
- Rezerwuje odpowiednią energię w baterii, żeby nie importować z sieci w tej samej godzinie rozliczeniowej – oddanie 1 kWh i zaciągnięcie 0,5 kWh w tej samej godzinie to strata netto
- Snapshoty importu/eksportu zapisywane na starcie każdej godziny w `var.magazyn_import_snapshot_hh00` / `var.magazyn_eksport_snapshot_hh00`

---

## Tryb LOWPV – ochrona przy słabej prognozie

System dysponuje dwoma niezależnymi progami LOWPV:

| Automatyzacja | Parametr | Domyślnie | Warunek |
|---|---|---|---|
| RANO (02–06) | `var.magazyn_lowpv_threshold_rano_kwh` | 8 kWh | Prognoza PV dziś (całodziennie) < próg → tylko diagnostyka; cel wynika z `var.cel_naladowania_o_13` |
| POŁUDNIE (13–15) | `var.magazyn_lowpv_threshold_popoludnie_kwh` | 8 kWh | Prognoza PV (skoryg. pozostało) < próg → planuj min. 30% na koniec okna 22/23 |

LOWPV nie wymusza już ładowania do 100%. Rano próg jest sygnałem diagnostycznym, a po południu dodaje warunek końcowego SOC: na koniec okna eksportu (`22:00` albo `23:00`, jeśli zaplanowano eksport po 22) magazyn ma mieć min. `30%`. Wartość jest trzymana w `var.magazyn_soc_min_wieczor_lowpv_percent` i resetowana nocnym bookingiem do bazowych `25%`.

---

## Korekta SOC na podstawie pogody

Automatyzacja uruchamiana codziennie o 05:29. Na podstawie prognozy `weather.pirateweather` (`condition`) koryguje `var.magazyn_soc_floor_percent`:

- `sunny` / `partlycloudy` → niższy floor SOC (PV wyprodukuje dużo)
- `cloudy` → środkowy floor
- `rainy` / `pouring` / inne → wyższy floor (zabezpieczenie przy słabej produkcji)

---

## Obsługa kalendarzy

| Kalendarz | Priorytet | Efekt na planowanie |
|---|---|---|
| `calendar.sprzatanie` | **1 (najwyższy)** | Zużycie = SQL × `magazyn_konsumpcja_mult_sprzatanie` (×2,0 kWh/h) |
| `calendar.urlop` | 2 | Zużycie = `magazyn_konsumpcja_urlop_kwh_h` (0,5 kWh/h) |
| Brak eventów | 3 | Zużycie = SQL × `magazyn_konsumpcja_multiplier` (×1,15) |

Sprzątanie ma wyższy priorytet niż urlop – przy nakładaniu się eventów obowiązuje mnożnik sprzątania (×2,0).

---

## Finanse PV

**Plik:** `finanse_pv.yaml`

Metodologia **Actual vs Counterfactual**:
```
Oszczędność = Koszt_G12_bez_PV − (Koszt_importu − Przychód_eksportu)
```

- Co 15 minut: delta konsumpcji × cena G12 (strefa droższa/tańsza) vs. faktyczne koszty/przychody
- Akumulatory dzienne resetują się o 23:59
- Skumulowany zysk całkowity ze sprzedaży z odliczeniem kosztu ładowania pod eksport
- Offset startowy wpisany ręcznie (oszczędność przed uruchomieniem systemu)
- Ceny RCE sprzedaży zerowane przy wartościach ujemnych (zgodnie z zasadami rozliczenia prosumenckiego)

---

## EMS1 Dashboard

**Pliki:** `packages/ems1_dashboard.yaml`, `dashboards/ems1_shadow.yaml`

Dashboard EMS1 ma kartę **Bilans dzienny: Load / Grid / PV / Magazyn**, która
zbiera w jednym miejscu dzienne statystyki energii: load, PV, grid import/eksport,
ładowanie/rozładowanie magazynu, netto magazynu i bilans kontrolny. Karta pokazuje
też `sensor.ems1_dzis_energia_pokryta_bez_importu` oraz live liczniki oszczędności
vs G12/G11 (`sensor.ems1_dzis_oszczednosc_g12`, `sensor.ems1_dzis_oszczednosc_g11`).
Oszczędności korzystają z tych samych akumulatorów `finanse_pv`, które trafiają do
podsumowania dziennego o 23:59.

---

## Sensory "safe"

**Plik:** `solarmansafe.yaml`

Falownik Deye przez SolarmanPV API zwraca nieraz `unknown`/`unavailable` lub skoki wartości przy restarcie. Sensory `_safe` filtrują te anomalie:

- `sensor.solarman_total_energy_sold_safe` – nie spada, ignoruje reset okienny (23:30–00:10)
- `sensor.solarman_total_energy_bought_safe` – analogicznie dla importu
- `sensor.solarman_daily_energy_sold_safe` – stabilizuje dzienny licznik sprzedaży
- `sensor.solarman_total_load_consumption_safe` – stabilizuje licznik całkowitego zużycia (trigger-based, `total_increasing`, nie spada)
- `sensor.moc_pobierana_przez_dom_safe` – filtruje skoki mocy pobieranej przez dom (odrzuca wartości poza ±20 kW i `unavailable`)

---

## Parametry konfiguracyjne

Wszystkie w `packages/zmienne_zarzadzanie_pv.yaml` jako `var:` (edytowalne z UI przez Developer Tools lub dedykowane karty Lovelace). Kluczowe:

| Parametr | Domyślnie | Opis |
|---|---|---|
| `magazyn_pojemnosc_brutto_kwh` | 15,0 kWh | Pojemność baterii brutto |
| `magazyn_soc_floor_percent` | 20% | Techniczne minimum SOC (korygowane przez pogodę) |
| `cel_naladowania_o_13` | 50% | Docelowy SOC o 13:00 |
| `magazyn_ladowanie_13_15_capacity_kwh` | 14 kWh | Zakładana energia możliwa do doładowania w oknie 13:00-15:00; konserwatywny limit do planowania, bo końcówka ładowania do 100% zwalnia |
| `magazyn_lowpv_threshold_rano_kwh` | 8 kWh | LOWPV próg RANO – diagnostyka, bez wymuszania 100% |
| `magazyn_lowpv_threshold_popoludnie_kwh` | 8 kWh | LOWPV próg POŁUDNIE – poniżej planuj wyższy SOC na koniec okna |
| `magazyn_min_zysk_sprzedaz_pln_kwh` | 0,38 PLN/kWh | Minimalny spread RCE do decyzji o sprzedaży |
| `magazyn_soc_min_rano_percent` | 25% | Twardy floor SOC w oknie 06–13 (blokuje eksport BAT) |
| `magazyn_soc_min_wieczor_lowpv_percent` | 25% | Floor SOC na koniec okna 22/23; w dniu LOWPV popołudnie podnosi do 30%, nocny booking resetuje do 25% |
| `magazyn_soc_stop_export_wieczor` | 20% | Twardy floor SOC eksportu wieczornego |
| `magazyn_soc_blokada_noc_min_percent` | 21% | Min SOC przy blokadzie nocnej (22:00) |
| `magazyn_soc_spill_bat_floor_percent` | 30% | Min SOC przy trybie BAT spill |
| `magazyn_konsumpcja_multiplier` | 1,15 | Mnożnik bezpieczeństwa zużycia (brak kalendarza) |
| `magazyn_konsumpcja_urlop_kwh_h` | 0,5 kWh/h | Stałe zużycie podczas urlopu |
| `magazyn_konsumpcja_mult_sprzatanie` | 2,0 | Mnożnik zużycia podczas sprzątania |
| `magazyn_sql_lookback_days_ladowanie` | 7 dni | Okno SQL dla ładowania |
| `magazyn_sql_lookback_days_spill` | 60 dni | Okno SQL dla spillu |
| `magazyn_bateria_sprawnosc_roundtrip` | 0,93 | Sprawność round-trip baterii |
| `magazyn_max_moc_ladowania_kw` | 6,0 kW | Maks. moc ładowania z sieci |
| `magazyn_max_moc_rozladowania_kw` | 6,0 kW | Maks. moc rozładowania do sieci |
| `magazyn_freeze_time` | 22:00 | Godzina blokady rozładowania (nocna) |
| `magazyn_koszt_calkowity_strefa_drozsza_pln_kwh` | 1,03 PLN/kWh | Pełny koszt energii – strefa droższa (G2) |
| `magazyn_koszt_calkowity_strefa_tansza_pln_kwh` | 0,65 PLN/kWh | Pełny koszt energii – strefa tańsza (G1) |
| `ems1_battery_avg_cost_pln_kwh` | dynamiczne | Średni koszt energii w magazynie EMS1: PV po koszcie alternatywnym RCE, grid po G12 |
| `ems1_battery_cost_basis_pln` | dynamiczne | Baza kosztowa energii objętej trackingiem EMS1 |
| `ems1_battery_costed_energy_kwh` | dynamiczne | Energia magazynu objęta średnim kosztem EMS1 |

---

## Scheduler AGD – optymalny start pralki i suszarki

System automatycznie uruchamia pralkę i suszarkę Siemens (Home Connect) w momencie najbardziej korzystnym energetycznie, bez ingerencji użytkownika po załadowaniu programu.

### Trigger

Automatyzacja odpala się gdy urządzenie sygnalizuje gotowość do zdalnego startu:
`binary_sensor.*_bsh_common_status_remotecontrolstartallowed` → `on`

Warunki wstępne: drzwi zamknięte (`doorstate = off`) oraz poprawny `operationstate`. Pralka akceptuje `Ready` i `Finished`; suszarka w aktualnej implementacji wymaga `Ready`.

### Logika wyboru czasu startu

| Czas | Warunek | Akcja |
|------|---------|-------|
| 06:00–13:00 | `var.spill_poranny > 1 kWh` | Czekaj aż RCE bieżącego slotu (15-min) < `var.magazyn_min_zysk_sprzedaz_pln_kwh` (220 PLN/MWh) — eksport i tak nieopłacalny, lepiej zużyć lokalnie |
| 06:00–13:00 | Prognoza PV < 8 kWh i SOC < cel_13 − 10% | Czekaj do 13:00 (lub wcześniej jeśli SOC ≥ cel_13 + 3%) |
| 06:00–13:00 | Inne | Natychmiast |
| 13:00–15:00 | Zawsze | Natychmiast (tania strefa G12) |
| 15:00–24:00 | `var.spill_poranny > 0.3 kWh` | Natychmiast |
| 15:00–24:00 | Brak spillu | O 23:30 |

### Architektura (restart-safe)

Zaplanowany czas startu zapisywany jest w `input_datetime` persystowanym przez restart HA:

- **Scheduler** – oblicza optymalny czas, ustawia `input_datetime.*_planned_start`, miga światłem `light.pralnia` (2s, przywraca stan) jeśli start odroczony
- **Executor** – trigger `time at: input_datetime.*_planned_start`, sprawdza warunki i wciska przycisk start; po wykonaniu resetuje timer do `1970-01-01`
- **SOC watcher** – monitoruje SOC w oknie 6-13 i przyspiesza timer do `now+1min` gdy bateria wystarczająco naładowana
- **Cancel** – gdy `remotecontrolstartallowed` → `off` (użytkownik anulował), timer jest kasowany

### Encje urządzeń

| | Pralka | Suszarka |
|---|---|---|
| Prefix | `siemens_wm14lphzpl_68a40e5ba649` | `siemens_wt7hy780pl_68a40e14bb26` |
| Timer | `input_datetime.pralka_planned_start` | `input_datetime.suszarka_planned_start` |

### Powiadomienia

Po uruchomieniu urządzenia wysyłane jest powiadomienie przez `script.ems_notify` (podlega filtrowi `input_boolean.ems_powiadomienia`).

---

## Znane ograniczenia

- **Tylko taryfa G12** – logika kosztu importu/zakupu (droższa/tańsza) jest zakodowana pod G12. Taryfa G11 ani G12W nie są obsługiwane bez modyfikacji kodu. Ceny TGE/RDN, jeśli są dostępne w HA, są w EMS v1 wyłącznie poglądowe; decyzje eksportowe opierają się na RCE.
- **Tylko falownik Deye/Solarman** – sterowanie przez encje `select.solarman_program_*`, `number.solarman_*`, `switch.solarman_battery_grid_charging`. Inne falowniki wymagają przeróbki całej warstwy sterującej.
- **Wymaga MariaDB** – zapytania SQL po historii zużycia działają wyłącznie z MySQL/MariaDB. SQLite (domyślna baza HA) nie obsługuje wymaganych funkcji.
- **Migracja do MariaDB usuwa historię** – przejście z SQLite wiąże się z utratą całej historii encji w HA.
- **Prognoza Solcast** – system zależy od jakości prognozy. Przy dużych odchyłach działa tryb LOWPV jako zabezpieczenie. Darmowe konto Solcast ma limit 10 wywołań API/dzień.
- **Planowanie w rozdzielczości 30-minutowej** – pętle planowania operują na slotach 30-min (0..47); bilansowanie godzinowe prosumenckie nadal respektowane (sloty parami per godzina RCE).
- **Jeden magazyn** – architektura zakłada pojedynczy magazyn energii. Wiele magazynów nie jest obsługiwanych.
- **Brak obsługi feed-in tariff (FIT)** – system zakłada model prosumencki z bilansowaniem, nie stały odkup energii.
- **Sterowanie manualne** – ręczna zmiana parametrów falownika z aplikacji Solarman może zostać nadpisana przez kolejną automatyzację EMS.
- **Produkcja PV tylko z mikrofalownika** – aktualnie system odczytuje produkcję wyłącznie z `sensor.solarman_microinverter_energy_today`. Panele podłączone bezpośrednio do falownika głównego (stringi DC) nie są uwzględniane w bilansie i prognozowaniu.

---

## Historia zmian

### 2026-07-07
- **EMS1: eksport wraca do podsumowania finansowego 23:59** (`packages/solarmansafe.yaml`, `packages/finanse_pv.yaml`): sensory `Solarman Total Energy Bought/Sold Safe` potrafią teraz wrócić do realnego totalizera po dużym historycznym przekłamaniu w górę, zamiast blokować się na zawyżonym `max(raw, prev)`. Akumulacja finansów dostała dodatkowy trigger `23:58:30`, żeby przed resetem 23:59 domknąć eksport/import z ostatniego slotu dnia.
- **EMS1: strategiczny hold SOC nie kasuje nocnej blokady** (`packages/magazyn_nowyeksport.yaml`): automatyzacja `Magazyn: Utrzymaj SOC pod eksport strategiczny` wyłącza teraz wspólny `input_boolean.battery_charge_from_grid` tylko wtedy, gdy strategiczny hold jest aktywny. Gdy `var.magazyn_strategiczny_hold_soc.enabled` jest false, nie dotyka trybu ładowania/hold SOC używanego przez nocną blokadę i poranny planner.
- **EMS1: nocna blokada używa preview planera 02:00** (`packages/magazyn_nowyeksport.yaml`): booking nocny nadal liczy `booked_soc`, ale przed ustawieniem hold-SOC wylicza `rano_preview_soc` i `rano_preview_charge` spójnie z logiką `Magazyn: RANO (02-06) cel na 13:00` (target 13:00, floor rano, urlop, pre-check popołudnia, poranny export topup). Limit falownika bierze wyższy z tych celów, nadal obcięty do aktualnego SOC, żeby blokować niepotrzebne nocne rozładowanie bez wymuszania natychmiastowego ładowania.
- **EMS1: LOWPV bez ładowania do 100%** (`packages/automations_magazyn.yaml`, `packages/magazyn_nowyeksport.yaml`, `packages/zmienne_zarzadzanie_pv.yaml`): usunięto poranne i popołudniowe wymuszanie celu 100% przy niskiej prognozie PV. Rano próg LOWPV jest diagnostyczny, a po południu niski forecast planuje min. 30% SOC na koniec okna 22/23 przez `var.magazyn_soc_min_wieczor_lowpv_percent`, resetowany nocnym bookingiem do 25%.

### 2026-07-05
- **EMS1 Dashboard: jedna grupa dziennego bilansu energii** (`packages/ems1_dashboard.yaml`, `dashboards/ems1_shadow.yaml`): dodano sensory Load/Grid/PV/Magazyn, licznik energii pokrytej bez importu oraz live liczniki oszczędności vs G12/G11 oparte o te same akumulatory, które są używane w nocnym podsumowaniu finansów.
- **EMS1: strategiczny hold SOC nie blokuje eksportu w środku slotu** (`packages/magazyn_nowyeksport.yaml`): automatyzacja `Magazyn: Utrzymaj SOC pod eksport strategiczny` rozpoznaje teraz cały przedział slotu `HH:MM-HH:MM`, a nie tylko dokładny start. Dodatkowo aktywny `input_boolean.battery_discharge_to_grid` ma priorytet i wymusza wyłączenie `battery_charge_from_grid`.

### 2026-06-30
- **EMS1: limit SOC dla porannego pre-checku popołudnia** (`packages/automations_magazyn.yaml`, `packages/zmienne_zarzadzanie_pv.yaml`): dodano `var.magazyn_soc_cap_precheck_popoludnie_percent` z domyślną wartością 90%. Poranny planner, gdy podbija cel 06:00 pod przyszłe zapotrzebowanie 13–15 / 15–22, obcina wyłącznie komponent `E6_req_afternoon` tak, żeby w symulowanych slotach PV nie przekraczać tego SOC. Twarde minimum rano (`var.magazyn_soc_min_rano_percent`) i zwykły target 13:00 nie są tym limitem zaniżane.
- **EMS1: tymczasowa statyczna baza zużycia dla plannerów** (`packages/automations_magazyn.yaml`, `packages/magazyn_nowyeksport.yaml`): 5 automatyzacji używających historii `sensor.srednie_zuzycie_w_obecnym_slocie_30min` dostało tymczasową tabelę `slot -> avg_h` wyliczoną z realnych danych `sensor.solarman_total_load_consumption_safe` z dni 2026-06-19..26. Dla okna 06:00-13:00 baza wynosi 10.216 kWh, więc przy kalendarzu sprzątania `x2` planner zakłada ok. 20.432 kWh zamiast zaniżonych wartości ze starej historii sensora. Zmiana jest oznaczona komentarzem `TEMP EMS1 2026-06-30` i powinna zostać usunięta po zebraniu poprawnej historii albo zastąpiona docelowym SQL-em z licznika.
- **EMS1: poprawka średniej zużycia 30-min** (`packages/sensors_sql_pv.yaml`): `sensor.srednie_zuzycie_w_obecnym_slocie_30min` liczy teraz historyczną średnią bezpośrednio z `sensor.solarman_total_load_consumption_safe` jako różnicę licznika między początkiem i końcem bieżącego slotu w poprzednich dniach. Poprzednie `MAX-MIN` wewnątrz slotu zaniżało zużycie przy rzadkich próbkach licznika, co psuło planowanie poranne przy kalendarzu sprzątania.

### 2026-06-29
- **EMS1: poranny raport spillu używa zużycia do końca dnia** (`packages/magazyn_nowyeksport.yaml`): zapytanie SQL dla `dict_z` w porannym eksporcie pobiera teraz sloty zużycia do `23:30` zamiast kończyć na `13:30`, dzięki czemu po południu nie wpada fallback `1.5 kWh/30min` przy kalendarzu sprzątania.

### 2026-06-28
- **EMS1: poranny pre-check ładowania pod popołudnie** (`packages/automations_magazyn.yaml`): poranny plan symuluje teraz 13–15 i 15–22, uwzględnia zakładane ładowanie 13–15 (`var.magazyn_ladowanie_13_15_capacity_kwh`) i podbija poranny target, jeśli samo okno 13–15 nie wystarczy do wymaganego `E15`; dodatkowe ładowanie jest ograniczone przez limit bez spillu PV.
- **EMS1: parametr pojemności ładowania 13-15** (`packages/zmienne_zarzadzanie_pv.yaml`): dodano `var.magazyn_ladowanie_13_15_capacity_kwh` z domyślną wartością 14 kWh jako konserwatywne założenie do przyszłej symulacji porannej.
- **EMS1: poranne ładowanie 02:00-06:00** (`packages/automations_magazyn.yaml`): start planowania/ładowania przesunięto z 03:00 na 02:00 przez dodanie triggerów 02:00 i 02:30. Stop pozostaje o 06:00, żeby magazyn miał cztery godziny na dobicie przy wysokim celu.
- **Finanse PV: koszt importu przy szybkim ładowaniu** (`packages/finanse_pv.yaml`): limit anty-spike dla delt importu/eksportu podniesiono z 3 do 10 kWh/15 min. Poprzedni próg odrzucał realne sloty ładowania magazynu powyżej 12 kW i zaniżał koszt importu poniżej minimalnej ceny G12 0.65 PLN/kWh.

### 2026-07-07
- **EMS1: blokada fałszywego eksportu po 22** (`packages/automations_magazyn.yaml`, `packages/magazyn_nowyeksport.yaml`): flaga `var.magazyn_plan_export_po_22` jest teraz aktywna tylko dla planu z bieżącej daty i tylko wtedy, gdy slot 22:00-23:00 faktycznie trafił do planu eksportu z energią. Sam dobry próg ceny po 22 nie wystarcza już do przesunięcia końca sesji na 23:00.

### 2026-06-24
- **EMS1: minimalny bufor SOC w eksporcie wieczornym FULL** (`packages/magazyn_nowyeksport.yaml`): poza energią prognozowaną na zużycie domu do końca bieżącej godziny, plan i stopper zachowują dodatkowo 1 pp. SOC na każdy pełny kwadrans pozostały po aktywnym slocie, z minimum 1 pp. Ostatni slot godziny kończy się więc przy co najmniej 21% przy floorze 20%.

### 2026-06-23
- **Nocny reset flooru SOC rano** (`packages/magazyn_nowyeksport.yaml`): uruchomienie nocnej blokady rozładowania (22:00 lub warunkowo 23:00) przywraca `var.magazyn_soc_min_rano_percent` do 25% przed bookingiem. Wartość ustalona pogodą o 05:29 dotyczy więc wyłącznie bieżącego poranka i nie przechodzi na kolejny dzień.

### 2026-06-21
- **EMS1: poprawka urlopowego bufora porannego ładowania** (`packages/automations_magazyn.yaml`):
  - `calendar.urlop` dodaje teraz stały bufor 1 kWh względem wyliczonego planu `E6_plan`, zamiast dodawać 1 kWh do aktualnego SOC przy każdym triggerze 03:00/03:30/04:00.
  - Zapobiega schodkowemu podbijaniu celu ładowania, np. 41% → 44% → 47%, gdy magazyn już dobił do poprzedniego targetu.
- **EMS1: warunkowy eksport wieczorny po 22:00** (`packages/automations_magazyn.yaml`, `packages/magazyn_nowyeksport.yaml`, `packages/zmienne_zarzadzanie_pv.yaml`):
  - Plan 13-15 wykrywa opłacalny slot 22:00-23:00 i zapisuje `var.magazyn_plan_export_po_22`.
  - Eksport wieczorny działa do 23:00 tylko przy aktywnej fladze; bez flagi zachowanie zostaje 15:00-22:00.
  - Blokada nocna o 22:00 pomija booking tylko przy aktywnym eksporcie po 22 i wykonuje go wtedy o 23:00; bez flagi 23:00 nic nie robi.
  - Domykacz HH:45 pozostaje ograniczony do godzin 15-21, bo po 22 obowiązuje tania taryfa.

### 2026-06-20
- **EMS1: eksport wieczorny PARTIAL netto** (`packages/magazyn_nowyeksport.yaml`):
  - Planowana energia baterii w slocie nadal steruje rozładowaniem.
  - `Szac. eksport` dla slotow PARTIAL liczy teraz eksport netto jako `BAT + prognoza PV - plan domu`, tak jak tryb FULL.

### 2026-06-18
- **Prąd ładowania i rozładowania magazynu 150A** (`packages/automations_magazyn.yaml`):
  - profil `battery_charge_from_grid` ustawia `number.solarman_battery_max_charging_current` na 150A
  - ładowanie z sieci ustawia `number.solarman_battery_grid_charging_current` na 150A
  - profile normalny i `battery_discharge_to_grid` ustawiają `number.solarman_battery_max_discharging_current` na 150A
  - ścieżki naprawcze dla stanów przejściowych wymuszają te same limity

### 2026-06-16
- Dodano trzy statystyki pomocnicze zużycia godzinnego: normalne bez kalendarzy, wariant urlopowy i wariant sprzątania.
- **EMS1: średni koszt energii w magazynie** (`packages/automations_magazyn.yaml`, `packages/zmienne_zarzadzanie_pv.yaml`):
  - Nowy tracker `ems1_battery_cost_tracker_5min` aktualizuje średni koszt co 5 minut na licznikach Solarman safe.
  - Mechanizm jest przeniesiony z EMS2: ładowanie z PV liczone po koszcie alternatywnym RCE, ładowanie z grid po G12, rozładowanie zdejmuje koszt po średniej.
  - Na tym etapie tracker tylko liczy koszt i diagnostykę; decyzje eksportowe EMS1 nie używają go jeszcze do wyboru godzin.

### 2026-06-01
- **EMS1: historycznie spójny cel LOWPV po południu** (`packages/automations_magazyn.yaml`):
  - Dawny tryb LOWPV do 100% przeliczał także `grid_add_u` na energię potrzebną do pełnego magazynu.
  - Ta ścieżka została później zastąpiona planowaniem min. 30% SOC na koniec okna 22/23.

### 2026-05-31
- **EMS1: popołudniowe doładowanie pod eksport przy niskim SOC** (`packages/automations_magazyn.yaml`):
  - `NO_SLOTS` w ETAPIE 2 oznacza teraz faktyczny brak drogich slotów RCE 15–22, a nie brak bieżącej nadwyżki energii w baterii.
  - Jeśli drogie sloty istnieją, ale SOC jest jeszcze niski, plan używa idealnej energii slotów do wyliczenia brakującego doładowania z sieci w oknie 13–15.

- **EMS1: naprawa podwójnego naliczania kosztów finansowych** (`packages/finanse_pv.yaml`):
  - Akumulacja finansów zeruje delty, gdy `var.finanse_prev_trigger_time` jest starszy niż 45 minut, zamiast naliczać w kółko ten sam stary przyrost licznika.
  - Zapis `finanse_prev_*` wykonywany jest od razu po policzeniu delt, żeby błąd w dalszych akumulatorach cyklu 6-6 nie powodował ponownego księgowania tego samego slotu.

### 2026-05-30
- **EMS1: notyfikacja osiągnięcia limitu ładowania 13:00** (`packages/automations_magazyn.yaml`, `packages/magazynlimity.yaml`):
  - Watchdog popołudniowy zatrzymuje ładowanie także po osiągnięciu limitu SOC ustawionego w planie 13:00 i wysyła notyfikację z powodem `limit SOC`.
  - Plan popołudniowy pokazuje wyliczenie limitu: `E_target`, `E_stop`, `PV credit`, floor i pojemność magazynu.
  - `script.ems_notify` dopisuje `GODZINA WYWOŁANIA` do każdej notyfikacji EMS.

### 2026-05-29
- **EMS1: popołudniowy watchdog ładowania po bilansie kWh** (`packages/automations_magazyn.yaml`, `packages/zmienne_zarzadzanie_pv.yaml`):
  - Notyfikacja planu popołudniowego dostała znacznik czasu `WYSTĄPIENIE`, żeby łatwo odróżnić przebiegi 13:00/14:00/14:30/14:45.
  - Watchdog stop ładowania popołudniowego czyta teraz `var.magazyn_soc_stop_ladowanie_popoludnie` zamiast nieistniejącego `input_number.*`.
  - Dodano snapshoty sesji ładowania: import, eksport, ładowanie/rozładowanie baterii, konsumpcja domu i PV.
  - Stop grid charge bazuje na wyliczeniu `kupione do baterii = min(max(import - szacowany import domu, 0), ładowanie baterii)`, a SOC jest tylko fallbackiem bez aktywnego celu kWh.
  - Notyfikacja STOP pokazuje pełny bilans sesji i cel importu z sieci.

### 2026-05-24
- **EMS2: blokada taniego eksportu PV poza spillem** (`custom_components/ems2/optimizer.py`):
  - `pv_export` jest używany tylko dla spillu PV w strefie pośredniej: cena sprzedaży powyżej `var.magazyn_min_zysk_sprzedaz_pln_kwh`, ale poniżej progu opłacalności baterii.
  - Poniżej marży solver zostaje w `idle`; powyżej progu baterii wybiera `discharge`/battery export z wyliczonym SOC końca slotu, zamiast zamrażać baterię przez `pv_export`.
  - Dodano testy optimizerowe dla niskiej ceny, strefy pośredniej `pv_export` i opłacalnego eksportu baterii.

### 2026-04-24 (4)
- **Domykacz HH:45 — action_mode w notyfikacji** (`packages/magazyn_nowyeksport.yaml`):
  - Tytuł notyfikacji: `Domykacz HH:45 | <skip_reason> | <action_mode>` gdzie action_mode = `PV` / `BAT` / `SKIP`
  - Body: dodano `| action=<action_mode>` — od razu widać czy i w jakim trybie domykacz zaczął oddawać energię

### 2026-04-25 (2)
- **Fix: AGD Pralka — operationstate blokował remote start** (`packages/ems_agd.yaml`):
  - Scheduler i executor wymagały `operationstate = Ready`, ale maszyna po zakończeniu cyklu jest w stanie `Finished`
  - Przy ładowaniu nowego prania: drzwi otwarte→zamknięte, wybór programu, remote ON — BSH ustawia `remotecontrolstartallowed = "on"` ale `operationstate` nadal `Finished` z poprzedniego cyklu
  - Automation condition failowała → pralka nie startowała zdalnie
  - Fix: warunek `condition: state` na operationstate zamieniony na `condition: template` akceptujący `Ready` **i** `Finished`

### 2026-04-25
- **Fix: `sensor.aktualny_limit_magazyn` — wykluczenie PV spill** (`packages/magazynlimity.yaml`):
  - Poprzednio `limit_mag` po zaniku PV = `pojemnosc - export_po_13`, gdzie `export_po_13` zawierał także spill PV (nie tylko eksport z bat.) — zaniżało `przeniesionaenergia` na kolejny dzień
  - Teraz: `battery_to_grid = min(daily_battery_discharge, export_po_13)` → bat. nie może "wyeksportować" więcej niż faktycznie rozładowała
  - Dodane atrybuty diagnostyczne: `battery_discharge_today_kwh`, `battery_to_grid_kwh`
  - Tryb `pv_zaniklo`: zmiana nazwy z `limit_brutto_minus_export_po_13` na `limit_brutto_minus_battery_to_grid`

### 2026-05-06
- **EMS1: śledzenie plan vs realizacja per slot** (`packages/magazyn_nowyeksport.yaml`, `packages/zmienne_zarzadzanie_pv.yaml`):
  - 3 nowe `var`: `ems1_slot_snap_kwh` (snapshot licznika eksportu na start slotu), `ems1_session_plan_total_kwh` (suma zaplanowanych kWh sesji), `ems1_session_actual_total_kwh` (suma realnego eksportu sesji)
  - Na starcie każdego 15-min slotu (06–13 i 15–22): zamknięcie poprzedniego slotu przez delta `solarman_total_energy_sold_safe`, akumulacja do `session_actual`; plan bieżącego slotu z `current_plan_energy` / `current_slot_export_energy` dodany do `session_plan`
  - Reset akumulatorów na start sesji (06:00 i 15:00)
  - Podsumowania 13:00 i 21:45 rozszerzone o sekcję **Plan vs Realizacja**: plan sesji, realizacja, efektywność (%), total dzienny z `solarman_daily_energy_sold`

### 2026-04-24 (3)
- **Domykacz HH:45 — PV-first** (`packages/magazyn_nowyeksport.yaml`, v2.0):
  - Jeśli nadwyżka PV (5-min avg) × 15 min ≥ deficyt godzinowy → domykacz włącza `pv_discharge_to_grid` zamiast BAT
  - Jeśli PV nie pokrywa deficytu → stare zachowanie: `battery_discharge_to_grid`
  - Stopper wyłącza oba booleany (`pv_discharge_to_grid` + `battery_discharge_to_grid`) gdy eksport osiągnie cel
  - Nowe zmienne diagnostyczne w notyfikacji: `pv_surplus`, `pv_can_close`, `was_pv_discharging`
  - SOC przy floorze nie blokuje domykacza gdy PV może samodzielnie zamknąć bilans

### 2026-04-24 (2)
- **PV export niezależny od BAT floor** (`packages/magazyn_nowyeksport.yaml`, `packages/zmienne_zarzadzanie_pv.yaml`):
  - Nowa zmienna `magazyn_soc_pv_floor_percent` (default 15%) — minimalny SOC do eksportu PV; poniżej tego PV też się zatrzymuje
  - Warunek `soc < soc_comfort_pct` w eksporcie porannym przestał blokować PV: teraz zatrzymuje tylko BAT, PV kontynuuje jeśli jest nadwyżka i SOC ≥ nowy floor (strażnik PV obsługuje dynamiczne stop/start przy zmianach produkcji)
  - Strażnik SOC: nowy warunek dla trybu `bat_spill/pv_bat_spill` — przy `soc ≤ spill_bat_floor − 1%` (histereza 1%) natychmiastowy przełącznik BAT→PV bez czekania 15 minut na kolejny trigger

### 2026-04-24
- **Poprawka liczenia "eksport za tanio"** (`packages/finanse_pv.yaml`): usunięto warunek `cena_sprzedazy_kwh > 0` — eksporty przy cenie RCE ≤ 0 (ujemna lub zerowa, spill PV) były pomijane w statystyce strat; teraz każdy eksport dzienny poniżej G12_tańsza jest liczony

### 2026-04-23
- **Poprawka `script.pralnia_mignij`** (`packages/ems_agd.yaml`): przy włączonym świetle `turn_on` był no-op — brak widocznego mignięcia; teraz logika rozgałęziona: jeśli ON → wyłącz 2s → włącz; jeśli OFF → włącz 2s → wyłącz
- **Powiadomienie po ustawieniu min SOC rano** (`packages/automations_magazyn.yaml`): automatyzacja `magazyn_soc_min_rano_pogoda` (5:29) wysyła `ems_notify` z wartością SOC i stanem pogody (`weather.pirateweather`)

### 2026-04-22
- **Straty małego magazynu — cykl 6-6** (`packages/finanse_pv.yaml`): nowe akumulatory i sensory mierzące ile pieniędzy "ucieka" przez zbyt małą pojemność baterii w cyklu 6:00→6:00
  - `eksport_tanio`: kWh oddane przy RCE < G12_tańsza + strata PLN
  - `import_noc`: kWh kupione 22-06 bo magazyn pusty + koszt PLN
  - `import_dzien_drogi`: kWh kupione z sieci w godzinach 06-13 i 15-22 (strefa droga G12) przy niskiej produkcji PV — nowy trzeci składnik straty łącznej (dodano 2026-04-22)
  - `eksport_dobry`: kWh sprzedane powyżej progu G12_tańsza + marża (informacyjnie)
  - Warianty: bieżący cykl, poprzedni cykl (zapis 06:00), skumulowany dożywotni; reset + powiadomienie codziennie o 06:00
- **Nowy pakiet `packages/ems_agd.yaml`** — optymalny start pralki i suszarki (Siemens BSH / Home Connect)
- Scheduler EMS-aware: 6-13 spill PV → start gdy RCE < marża (220 PLN/MWh, slot 15-min); mała produkcja + niski SOC → o 13:00 lub wcześniej; 13-15 od razu; po 15 z spillem od razu; bez spillu o 23:30
- Timer `input_datetime` persystuje przez restart HA (executor trigger: `time at:`)
- Anulowanie timera gdy `remotecontrolstartallowed` → OFF
- SOC watcher: przyspiesza start do now+1min gdy SOC ≥ cel_13 + 3% w oknie 6-13
- `script.pralnia_mignij`: mignięcie 2s z przywróceniem stanu światła
- Notyfikacje `script.ems_notify` przy starcie każdego urządzenia

### 2026-04-21
- **Migracja planowania na sloty 30-min (0..47)** – wszystkie pętle RANO, POŁUDNIE, eksport poranny, eksport wieczorny i blokada nocna operują na slotach 30-minutowych zamiast godzinowych
- SQL `GROUP BY slot = HOUR*2 + FLOOR(MINUTE/30)` we wszystkich automatyzacjach; sensor `srednie_zuzycie_w_obecnym_slocie_30min`
- Prognoza PV: `detailedHourly` → `detailedForecast` (30-min kWh/period) we wszystkich automatyzacjach
- `cons_default_slot` = `cons_default / 2`; `urlop_kwh_slot` = `urlop_kwh_h / 2`; `current_slot_factor` dla bieżącego slotu
- `pv_end_slot_excl` (domyślnie 28 = 14:00), `spill_start_slot` zamiast `spill_start_hour`
- Naprawiono `TypeError` w oddawaniu porannym: `"%02d:%02d" % [h, m]` → `(h, m)`
- Naprawiono przeliczanie `pv_estimate` z Solcast: wartości w kWh/h → kWh/30min (÷2) we wszystkich automatyzacjach
- Strażnik PV: mnożnik `pv_surplus_min_w` zmieniony z 1,25 na 1,0 (eksport gdy PV ≥ zużycie, bez buforu)

### 2026-04-20
- Dodano `sensor.solarman_total_load_consumption_safe` (trigger-based, `total_increasing`, nie spada przy reset)
- Zmieniono `moc_pobierana_przez_dom` → `sensor.moc_pobierana_przez_dom_safe` z pełnym wzorcem `_safe` (filtr skoków ±20 kW, `availability: true`)
- Naprawiono referencję w statistics sensor (wskazywał na nieistniejący `sensor.moc_pobierana_przez_dom_2`)
- Pojemność baterii w `sensor.bateria_energia_kwh`: zahardkodowane `15.0` → `var.magazyn_pojemnosc_brutto_kwh`
- `finanse_pv.yaml`: `cons_now` używa `sensor.solarman_total_load_consumption_safe`

### 2026-04-19
- Poprawiono wyliczanie `spill` i logikę limitu rano gdy spill

### 2026-04-18
- Poprawiono `export_topup` — używa idealnego planu zamiast ograniczonego energią

### 2026-04-14
- Domykacz: twardy stop przez `var` + stopper event-driven
- Przeniesiono sensory SQL do `sensors_sql_pv.yaml`
- Podzielono `lowpv_threshold` na rano i popołudnie + fix: brak slotów eksportu w trybie LOWPV
- Zamiana stałych wartości na zmienne `var.*`
- Sprzątanie ma wyższy priorytet niż urlop we wszystkich obliczeniach zużycia
- Poprawiono wzór `soc_stop_target` dla trybu partial w eksporcie wieczornym

---

## Backlog / TODO

| Zadanie | Opis |
|---|---|
| _(brak otwartych zadań)_ | – |
