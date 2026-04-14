# EMS – Energy Management System

System zarządzania energią oparty na Home Assistant, sterujący ładowaniem i rozładowaniem magazynu energii oraz sprzedażą energii do sieci w oparciu o prognozę PV, historyczne zużycie i ceny RCE (Rynkowa Cena Energii).

---

## Wymagania sprzętowe i taryfowe

| Element | Wymaganie |
|---|---|
| Falownik | Deye (SolarmanPV API v5) – integracja `solarman` |
| Magazyn energii | Bateria podłączona do falownika Deye, pojemność skonfigurowana w `var.magazyn_pojemnosc_brutto_kwh` (domyślnie 15 kWh) |
| Prognoza PV | Solcast (integracja HA) z modelem `detailedHourly` |
| Prognoza pogody | Pirate Weather (`weather.pirateweather`) – warunki pogodowe do korekty minimalnego SOC rano |
| Ceny energii | Sensor `sensor.rce_prices_today_scaled` / `tomorrow_scaled` – ceny RCE z VAT (PLN/kWh); ceny TGE RDN dostępne poglądowo |
| Taryfa OSD | **G12** – strefa droższa i tańsza (22:00–06:00 + 13:00–15:00) |
| Baza danych | MariaDB (MySQL) – wymagana do zapytań SQL po historii zużycia |
| Kalendarz HA | `calendar.urlop` i `calendar.sprzatanie` – opcjonalne, modyfikują planowanie |

System **nie działa** z falownikami innych producentów, taryfą G11 (flat), ani bez lokalnej bazy MariaDB.

---

## Architektura – pliki packages

```
packages/
├── zmienne_zarzadzanie_pv.yaml   # Wszystkie parametry konfiguracyjne (var, input_boolean, input_number)
├── solarmansafe.yaml             # Sensory "safe" – stabilizacja odczytów falownika + statistics + min_max
├── magazyn_import.yaml           # SQL: średnie zużycie godzinowe i 30-minutowe z historii
├── sensors_sql_pv.yaml           # SQL: energia oddana (slot 15min, 6-13, 15-22)
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
5. Jeśli prognoza PV < `magazyn_lowpv_threshold_kwh` (8 kWh) → ładuje do 100% (tryb LOWPV).
6. Jeśli jest planowany poranny eksport (spill) → może dodać bufor `spill_poranny`.

**Modyfikatory zużycia:**
- `calendar.urlop` aktywny w danej godzinie → z_h = 0,5 kWh (niskie zużycie)
- `calendar.sprzatanie` aktywny → z_h = SQL × 2,0 (podwójne zużycie)
- Brak kalendarza → z_h = SQL × 1,15 (mnożnik bezpieczeństwa)

---

## Planowanie ładowania – okno POŁUDNIE (13:00–15:00)

**Kiedy działa:** trigery co 15–30 min między 13:00 a 14:45.

**Cel:** opcjonalne doładowanie baterii z sieci (taryfa tańsza 13–15), aby przygotować się na wieczorne oddawanie lub zapewnić energię na noc.

**Algorytm:**
1. Pobiera z SQL średnie zużycie dla godzin 13–22.
2. Pobiera prognozę PV 13–22 (odejmuje już wyprodukowaną energię).
3. Symuluje SOC godzina po godzinie do 22:00.
4. Wyznacza limit doładowania potrzebny do pokrycia zużycia wieczornego z zapasem.
5. Uwzględnia planowany eksport wieczorny (jeśli aktywny).

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
| BAT spill | cena RCE > `koszt_tansza` | realny spill, SOC > 30% |
| BAT semi | cena RCE > `koszt_tansza + min_zysk` | cel min. 50% SOC o 13:00 |
| BAT full | cena RCE > `koszt_drozsza + min_zysk` | do `min_soc_dynamic_13` |

Dodatkowy floor: `var.magazyn_soc_min_rano_percent` – żaden tryb BAT nie zejdzie poniżej tego SOC.

---

## Eksport wieczorny (15:00–22:00)

**Plik:** `magazyn_nowyeksport.yaml`  
**Trigery:** co 15 minut od 15:00 do 21:45.

Analogiczna logika jak eksport poranny, ale dla okna 15–22. Warunek cenowy: cena RCE aktualnego slotu > próg minimalnego zysku. SOC stop eksportu wieczornego: `var.magazyn_soc_stop_export_wieczor` (domyślnie 20%).

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

## Obsługa kalendarzy

| Kalendarz | Efekt na planowanie |
|---|---|
| `calendar.urlop` | Zużycie = 0,5 kWh/h (priorytet nadrzędny) |
| `calendar.sprzatanie` | Zużycie = SQL × 2,0 kWh/h (sprzątanie intensywne) |
| Brak eventów | Zużycie = SQL × 1,15 (mnożnik bezpieczeństwa) |

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
- Obsługa ujemnych cen RCE (bez limitu na 0 PLN)

---

## Sensory "safe"

**Plik:** `solarmansafe.yaml`

Falownik Deye przez SolarmanPV API zwraca nieraz `unknown`/`unavailable` lub skoki wartości przy restarcie. Sensory `_safe` filtrują te anomalie:

- `sensor.solarman_total_energy_sold_safe` – nie spada, ignoruje reset okienny (23:30–00:10)
- `sensor.solarman_total_energy_bought_safe` – analogicznie dla importu
- `sensor.solarman_daily_energy_sold_safe` – stabilizuje dzienny licznik sprzedaży

---

## Parametry konfiguracyjne (kluczowe)

Wszystkie w `packages/zmienne_zarzadzanie_pv.yaml` jako `var:` (edytowalne z UI przez panel Developer Tools lub dedykowane karty):

| Parametr | Domyślnie | Opis |
|---|---|---|
| `magazyn_pojemnosc_brutto_kwh` | 15,0 kWh | Pojemność baterii |
| `magazyn_soc_floor_percent` | 20% | Techniczne minimum SOC |
| `cel_naladowania_o_13` | 50% | Docelowy SOC o 13:00 |
| `magazyn_lowpv_threshold_kwh` | 8 kWh | Poniżej: ładuj do 100% |
| `magazyn_min_zysk_sprzedaz_pln_kwh` | 0,38 PLN/kWh | Minimalny spread do sprzedaży |
| `magazyn_soc_min_rano_percent` | 25% | Floor SOC w oknie 06–13 |
| `magazyn_konsumpcja_multiplier` | 1,15 | Mnożnik bezpieczeństwa zużycia |
| `magazyn_sql_lookback_days_ladowanie` | 7 dni | Okno SQL dla ładowania |
| `magazyn_bateria_sprawnosc_roundtrip` | 0,93 | Sprawność round-trip baterii |
| `magazyn_freeze_time` | 22:00 | Godzina blokady rozładowania |

---

## Backlog / TODO

Otwarte zadania z Asany:

| Zadanie | Opis |
|---|---|
| **Migracja prognozy PV na detailedForecast** | Przejście z `detailedHourly` na `detailedForecast` (półgodzinny) w Solcast – dokładniejsze dane PV per slot. Zależne od migracji konsumpcji. |
| **Migracja planowania na sloty 30-min (konsumpcja)** | Pętle planowania (`range(h)`) w RANO, POŁUDNIE i eksportach przejdą na sloty `0..47`. SQL queries: `GROUP BY slot = HOUR*2 + FLOOR(MINUTE/30)`. |
| **Definicja struktury danych – SQL sprzedaży 15-min** | Zapytanie SQL liczące oddaną energię w slotach 15-minutowych na bazie `sensor.solarman_total_energy_sold_safe`. |
| **System raportowy ładowania/rozładowania** | Raport dzienny/miesięczny cykli ładowania i rozładowania baterii. |

---

## Znane ograniczenia

- **Tylko taryfa G12** – logika stref cenowych (droższa/tańsza) jest zakodowana pod G12. Taryfa G11, G12W ani dynamiczna nie są obsługiwane bez modyfikacji.
- **Tylko falownik Deye/Solarman** – sterowanie przez encje `select.solarman_program_*`, `number.solarman_*`, `switch.solarman_battery_grid_charging`. Inne falowniki wymagają przeróbki całej warstwy sterującej.
- **Wymaga MariaDB** – zapytania SQL po historii zużycia działają wyłącznie z MySQL/MariaDB. SQLite (domyślna baza HA) nie obsługuje wymaganych funkcji (`CONVERT_TZ`, `DECIMAL`).
- **Prognoza Solcast** – system zależy od jakości prognozy. Przy dużych odchyłach (zachmurzenie) działa tryb LOWPV jako zabezpieczenie.
- **Planowanie w rozdzielczości godzinowej** – pętle planowania operują na pełnych godzinach, co jest zgodne z godzinowym bilansowaniem prosumenckim.
- **Jeden magazyn** – architektura zakłada pojedynczy magazyn energii. Wiele magazynów nie jest obsługiwanych.
- **Brak obsługi feed-in tariff** – system zakłada model prosumencki z bilansowaniem (nie FIT stały).
- **Sterowanie manualne** – podczas gdy automatyzacje działają, ręczna zmiana parametrów falownika z aplikacji Solarman może zostać nadpisana przez kolejną automatyzację.
