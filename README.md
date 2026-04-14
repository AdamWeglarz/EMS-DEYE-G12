# EMS – Energy Management System for Home Assistant

Zaawansowany system zarządzania energią zbudowany w Home Assistant, sterujący ładowaniem i rozładowaniem magazynu energii oraz sprzedażą energii do sieci. Decyzje podejmowane są w czasie rzeczywistym na podstawie prognozy PV (Solcast), historycznego zużycia (SQL), aktualnych cen RCE i warunków pogodowych. System jest zoptymalizowany pod taryfę G12 i model prosumencki z bilansowaniem godzinowym.

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
| [`solcast-solar`](https://github.com/BJReplay/ha-solcast-solar) | Prognoza PV – tryb `detailedHourly`; wymagany darmowy token API |
| [`pirateweather`](https://github.com/alexander0042/pirate-weather-ha) | Prognoza pogody – korekta minimalnego SOC rano (HACS) |
| [`rce_prices`](https://github.com/AdamWeglarz/rce_prices) | Sensor cen RCE z VAT: `sensor.rce_prices_today_scaled` / `tomorrow_scaled` |
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

- **Automatyczne ładowanie z sieci w oknie G12 RANO (03:00–06:00)** – precyzyjne wyznaczanie ilości energii do załadowania
- **Automatyczne ładowanie z sieci w oknie G12 POŁUDNIE (13:00–15:00)** – doładowanie przed wieczornym eksportem
- **Eksport PV (oddawanie nadwyżki fotowoltaiki) 06:00–13:00** – 4 tryby od PV-only do pełnego rozładowania baterii
- **Eksport wieczorny z baterii 15:00–22:00** – sprzedaż przy korzystnych cenach RCE
- **Blokada nocna (22:00)** – planowanie na całą noc i ochrona przed zbyt głębokim rozładowaniem
- **Tryb awaryjny LOWPV** – wymuszenie ładowania do 100% przy słabej prognozie PV (oddzielne progi dla RANO i POŁUDNIE)
- **Bilansowanie godzinowe** – przestrzeganie zasad rozliczenia prosumenckiego (nie oddawaj energii, którą odkupisz tej samej godziny)
- **Korekta SOC na podstawie pogody** – falownik rano (05:29) koryguje minimalny SOC zależnie od zachmurzenia
- **Obsługa kalendarzy** – automatyczna korekta planowanego zużycia podczas urlopu i sprzątania
- **Raporty finansowe (Actual vs Counterfactual)** – śledzenie oszczędności i przychodów w czasie rzeczywistym
- **Sensory stabilizujące odczyty falownika** – filtrowanie skoków i `unknown`/`unavailable`
- **Pełna konfigurowalność** – wszystkie parametry jako `var.*` edytowalne z UI bez zmiany kodu

---

## Architektura – pliki packages

```
packages/
├── zmienne_zarzadzanie_pv.yaml   # Wszystkie parametry konfiguracyjne (var, input_boolean, input_number)
├── solarmansafe.yaml             # Sensory "safe" – stabilizacja odczytów falownika + statistics + min_max
├── sensors_sql_pv.yaml           # SQL: średnie zużycie godzinowe/30-min + energia oddana (slot 15min, 6-13, 15-22)
├── automations_magazyn.yaml      # Główne automatyzacje: RANO (03-06) i POŁUDNIE (13-15)
├── magazyn_nowyeksport.yaml      # Eksport: oddawanie poranne (06-13), wieczorne (15-22), blokada 22
├── magazynlimity.yaml            # Bilans eksportu, przeniesiona energia, sensory pomocnicze
└── finanse_pv.yaml               # Śledzenie oszczędności i przychodów ze sprzedaży
```

---

## Tryby pracy falownika

System operuje na 3 trybach sterowanych przez `input_boolean`:

| Flaga | Tryb falownika | Zastosowanie |
|---|---|---|
| `battery_charge_from_grid` | Grid Charging (wszystkie 6 programów = Grid, docelowy SOC) | Nocne ładowanie z sieci |
| `pv_discharge_to_grid` | Export First, prąd rozładowania = 0 | Oddawanie nadwyżki PV (bez baterii) |
| `battery_discharge_to_grid` | Export First, prąd rozładowania = 120A | Oddawanie energii z baterii do sieci |

Zmiana dowolnej flagi wyzwala automatyzację `pv_battery_grid_modes_profile_sync_v2`, która synchronizuje parametry Solarmana (tryb pracy, 6 programów, SOC, prąd rozładowania) z minimalną liczbą wywołań API.

Sensor `sensor.solarman_mode_status` pokazuje aktualnie rozpoznany tryb (np. _"Ładuje z Grid"_, _"Oddaje nadwyżkę PV"_, _"Normalna praca"_).

---

## Planowanie ładowania – okno RANO (03:00–05:45)

**Kiedy działa:** trigery co 30 min między 03:00 a 05:45.

**Cel:** doładować baterię z sieci (taryfa tańsza G12) tak, aby o 13:00 mieć zaplanowany SOC `var.cel_naladowania_o_13` (domyślnie 50%), nie spadając po drodze poniżej `var.magazyn_soc_min_rano_percent`.

**Algorytm:**
1. Pobiera z SQL średnie zużycie godzinowe z ostatnich 7 dni dla godzin 3–12.
2. Pobiera prognozę PV z Solcast (godziny 6–13 dziś).
3. Symuluje godzina po godzinie: aktualna energia w baterii − zużycie + PV.
4. Wyznacza minimalny `limit_soc` potrzebny do dotarcia do 13:00 bez zejścia poniżej `magazyn_soc_floor_percent` (20%).
5. Jeśli prognoza PV < `var.magazyn_lowpv_threshold_rano_kwh` (domyślnie 8 kWh) → **tryb LOWPV**: ładuje do 100%.
6. Jeśli jest planowany poranny eksport (spill) → może dodać bufor `spill_poranny`.

**Modyfikatory zużycia:**
- `calendar.urlop` aktywny w danej godzinie → z_h = `var.magazyn_konsumpcja_urlop_kwh_h` (domyślnie 0,5 kWh; priorytet nadrzędny)
- `calendar.sprzatanie` aktywny → z_h = SQL × `var.magazyn_konsumpcja_mult_sprzatanie` (domyślnie ×2,0)
- Brak kalendarza → z_h = SQL × `var.magazyn_konsumpcja_multiplier` (domyślnie ×1,15)

---

## Planowanie ładowania – okno POŁUDNIE (13:00–15:00)

**Kiedy działa:** trigery co 15–30 min między 13:00 a 14:45.

**Cel:** opcjonalne doładowanie baterii z sieci (taryfa tańsza 13–15), aby przygotować się na wieczorne oddawanie lub zapewnić energię na noc.

**Algorytm:**
1. Pobiera z SQL średnie zużycie dla godzin 13–22.
2. Pobiera prognozę PV 13–22 (odejmuje już wyprodukowaną energię).
3. Symuluje SOC godzina po godzinie do 22:00.
4. Wyznacza limit doładowania potrzebny do pokrycia zużycia wieczornego z zapasem.
5. Jeśli prognoza skoryg. < `var.magazyn_lowpv_threshold_popoludnie_kwh` (domyślnie 8 kWh) **i** nie wystarcza na full + konsumpcję do 15:00 → **tryb LOWPV**: ładuje do 100%.
6. Uwzględnia planowany eksport wieczorny i opcjonalne doładowanie pod eksport (`input_boolean.magazyn_doladowanie_pod_eksport_wieczor`).

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

Automatyzacja POŁUDNIE (13–15) co uruchomienie oblicza **najlepsze sloty cenowe** na wieczór (15–22h) i ewentualnie planuje dodatkowe ładowanie z sieci (`export_topup`), żeby mieć wystarczający SOC do sprzedania zaplanowanej energii. Nawet w trybie LOWPV (cel = 100%) sloty są kalkulowane i wyświetlane w notyfikacji – bateria jest już pełna, więc `export_topup` jest pomijany.

SOC stop eksportu wieczornego: `var.magazyn_soc_stop_export_wieczor` (domyślnie 20%).

---

## Blokada nocna (22:00)

**Plik:** `magazyn_nowyeksport.yaml`

O 22:00 system planuje całą noc JUTRO (22:00 → 13:00 następnego dnia):
- Pobiera prognozę PV na jutro (06:00–13:00)
- Symuluje zużycie na podstawie SQL dla godzin 22–24 i 0–13
- Wyznacza czy bieżący SOC wystarczy do 13:00 bez dołowania poniżej `magazyn_soc_blokada_noc_min_percent`
- Jeśli nie – zatrzymuje eksport i przełącza falownik w tryb normalny (Zero Export To CT)

---

## Bilansowanie godzinowe (wymóg prawny)

Zgodnie z prawem energetycznym, prosumenci są rozliczani w bilansowaniu **godzinowym** (pełna godzina zegarowa). System uwzględnia to przy planowaniu eksportu:

- Przed oddaniem energii w danej godzinie wylicza przewidywane zużycie do końca tej godziny
- Rezerwuje odpowiednią energię w baterii, żeby nie importować z sieci w tej samej godzinie rozliczeniowej – oddanie 1 kWh i zaciągnięcie 0,5 kWh w tej samej godzinie to strata netto
- Snapshoty importu/eksportu zapisywane na starcie każdej godziny w `var.magazyn_import_snapshot_hh00` / `var.magazyn_eksport_snapshot_hh00`

---

## Tryb LOWPV – ochrona przy słabej prognozie

System dysponuje dwoma niezależnymi progami LOWPV:

| Automatyzacja | Parametr | Domyślnie | Warunek |
|---|---|---|---|
| RANO (03–06) | `var.magazyn_lowpv_threshold_rano_kwh` | 8 kWh | Prognoza PV dziś (całodziennie) < próg → ładuj do 100% |
| POŁUDNIE (13–15) | `var.magazyn_lowpv_threshold_popoludnie_kwh` | 8 kWh | Prognoza PV (skoryg. pozostało) < próg **i** nie wystarcza na full + konsumpcję do 15:00 → ładuj do 100% |

W trybie LOWPV cel SOC = 100%; nadal kalkulowane są wieczorne sloty eksportu (do celów informacyjnych i przyszłego planowania), ale dodatkowe ładowanie pod eksport (`export_topup`) jest pomijane — bateria jest już pełna.

---

## Korekta SOC na podstawie pogody

Automatyzacja uruchamiana codziennie o 05:29. Na podstawie prognozy `weather.pirateweather` (`condition`) koryguje `var.magazyn_soc_floor_percent`:

- `sunny` / `partlycloudy` → niższy floor SOC (PV wyprodukuje dużo)
- `cloudy` → środkowy floor
- `rainy` / `pouring` / inne → wyższy floor (zabezpieczenie przy słabej produkcji)

---

## Obsługa kalendarzy

| Kalendarz | Efekt na planowanie |
|---|---|
| `calendar.urlop` | Zużycie = `magazyn_konsumpcja_urlop_kwh_h` (0,5 kWh/h; **priorytet nadrzędny**) |
| `calendar.sprzatanie` | Zużycie = SQL × `magazyn_konsumpcja_mult_sprzatanie` (×2,0 kWh/h) |
| Brak eventów | Zużycie = SQL × `magazyn_konsumpcja_multiplier` (×1,15) |

Urlop ma wyższy priorytet niż sprzątanie – przy nakładaniu się eventów obowiązuje 0,5 kWh/h.

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

## Sensory "safe"

**Plik:** `solarmansafe.yaml`

Falownik Deye przez SolarmanPV API zwraca nieraz `unknown`/`unavailable` lub skoki wartości przy restarcie. Sensory `_safe` filtrują te anomalie:

- `sensor.solarman_total_energy_sold_safe` – nie spada, ignoruje reset okienny (23:30–00:10)
- `sensor.solarman_total_energy_bought_safe` – analogicznie dla importu
- `sensor.solarman_daily_energy_sold_safe` – stabilizuje dzienny licznik sprzedaży

---

## Parametry konfiguracyjne

Wszystkie w `packages/zmienne_zarzadzanie_pv.yaml` jako `var:` (edytowalne z UI przez Developer Tools lub dedykowane karty Lovelace). Kluczowe:

| Parametr | Domyślnie | Opis |
|---|---|---|
| `magazyn_pojemnosc_brutto_kwh` | 15,0 kWh | Pojemność baterii brutto |
| `magazyn_soc_floor_percent` | 20% | Techniczne minimum SOC (korygowane przez pogodę) |
| `cel_naladowania_o_13` | 50% | Docelowy SOC o 13:00 |
| `magazyn_lowpv_threshold_rano_kwh` | 8 kWh | LOWPV próg RANO – poniżej: ładuj do 100% |
| `magazyn_lowpv_threshold_popoludnie_kwh` | 8 kWh | LOWPV próg POŁUDNIE – poniżej: ładuj do 100% |
| `magazyn_min_zysk_sprzedaz_pln_kwh` | 0,38 PLN/kWh | Minimalny spread RCE do decyzji o sprzedaży |
| `magazyn_soc_min_rano_percent` | 25% | Twardy floor SOC w oknie 06–13 (blokuje eksport BAT) |
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

---

## Znane ograniczenia

- **Tylko taryfa G12** – logika stref cenowych (droższa/tańsza) jest zakodowana pod G12. Taryfa G11 ani G12W nie są obsługiwane bez modyfikacji kodu. Ceny TGE RDN dostępne są w systemie wyłącznie poglądowo.
- **Tylko falownik Deye/Solarman** – sterowanie przez encje `select.solarman_program_*`, `number.solarman_*`, `switch.solarman_battery_grid_charging`. Inne falowniki wymagają przeróbki całej warstwy sterującej.
- **Wymaga MariaDB** – zapytania SQL po historii zużycia działają wyłącznie z MySQL/MariaDB. SQLite (domyślna baza HA) nie obsługuje wymaganych funkcji.
- **Migracja do MariaDB usuwa historię** – przejście z SQLite wiąże się z utratą całej historii encji w HA.
- **Prognoza Solcast** – system zależy od jakości prognozy. Przy dużych odchyłach działa tryb LOWPV jako zabezpieczenie. Darmowe konto Solcast ma limit 10 wywołań API/dzień.
- **Planowanie w rozdzielczości godzinowej** – pętle planowania operują na pełnych godzinach; zgodne z godzinowym bilansowaniem prosumenckim, ale mniej dokładne niż sloty 30-minutowe.
- **Jeden magazyn** – architektura zakłada pojedynczy magazyn energii. Wiele magazynów nie jest obsługiwanych.
- **Brak obsługi feed-in tariff (FIT)** – system zakłada model prosumencki z bilansowaniem, nie stały odkup energii.
- **Sterowanie manualne** – ręczna zmiana parametrów falownika z aplikacji Solarman może zostać nadpisana przez kolejną automatyzację EMS.

---

## Backlog / TODO

| Zadanie | Opis |
|---|---|
| **Migracja planowania na sloty 30-min (konsumpcja)** | Pętle planowania (`range(h)`) w RANO, POŁUDNIE i eksportach przejdą na sloty `0..47`. SQL queries: `GROUP BY slot = HOUR*2 + FLOOR(MINUTE/30)`. |
| **Migracja prognozy PV na detailedForecast** | Przejście z `detailedHourly` na `detailedForecast` (półgodzinny) w Solcast – dokładniejsze dane PV per slot. Zależne od migracji konsumpcji. |
