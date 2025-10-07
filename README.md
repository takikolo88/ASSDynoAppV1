# ASSDynoAppV1
Android Dynomet
IO/porty (USB-serial):

Automatyczny handshake UBX: CFG-PRT (8N1, 57600/115200), CFG-RATE (10/20 Hz), CFG-MSG (NAV-PVT, NAV-VELNED, GGA/RMC/VTG minimalnie), zapis do BBR/flash i weryfikacja odczytem.

Back-pressure & framing: ramki > 4 KB -> chunked parse; detekcja „frame slip” (resync po 0xB5 0x62 / „$GP”).

Reconnect z wykładniczym backoff i szybkim resume bez utraty sesji (ring-buffer 10 s).

Czas i Hz:

Zegar bazowy: SystemClock.elapsedRealtimeNanos() (monotonic) + znacznik z GNSS (czas GPS) → drift monitor, korekta Age.

Estymator Hz: EMA + okno 10, wykrywaj aliasing (np. „10 Hz udające 5 Hz”).

Jakość danych:

Guard „jitter clamp”: jeżeli dt > 250 ms → drop; jeżeli |Δv| > 8 km/h lub Δv_rel > 12% → drop; jeżeli HDOP>1.5/Sats<10/Age>400 ms → drop (jak w planie).

Auto-profil 10/20 Hz: wybór Alpha=0.35 (10 Hz) / 0.25 (20 Hz) dynamicznie z aktualnego Hz.

Kalibracja k (rpm = k·v_kmh):

Bufor kalibracyjny z IQR i odrzutami ±10% od mediany; sukces range ≤ 8% (jak w planie) + sanity-check: bieg 3/4 → k mieści się w oknie oczekiwanym dla przełożeń (czujka błędu).

KAdjustmentPercent (±5%) żywo wpływa tylko na UI RPM, nie na obliczenia mocy (opcjonalny przełącznik „apply to calc”).

Straty (LossBins):

P_loss liczone w „coast” z hamowaniem silnikiem odciętym; kosze 100 RPM; P90 + P50 (raportuj obie).

Walidacja bins: minimalna liczba próbek/bin (np. 10) – inaczej bin „niedojrzały”.

FSM i bezpieczeństwo:

Arm wymaga READY ≥1 s i brak „degraded” w oknie 4 próbek.

EndAccel: średnia a_smooth w 600 ms ≤ 0, oraz rpm ≥ 1200; dodatkowy „timeout” (np. 30 s) jako bezpiecznik.

Coast N s z kontrolą jakości (spadki Hz → wydłuż coast).

UI/UX (Compose):

Pasek READY = zielony tylko gdy Hz ≥ 9.8 i HDOP ≤ 1.5 i Age ≤ 400 ms.

„Live RPM” + suwak KAdj ±5% z przyciskiem Reset.

Wykresy: P_wheel, Torque, P_loss (toggle), markery MAX; eksport CSV (sesja + bins).

Mini-log (ostatnie 3 linie) + licznik „drops”.

Trwałość i dzienniki:

DataStore/JSON (config, k dla biegów, masa, progi).

Log CSV: nagłówek z metadanymi sesji (masa, Hz, filtr, wersja algorytmu).

Rotacja plików (max 50 MB/plik), folder dzienny.

Testy:

Unit: bramki jakości, filtry, FSM przejścia, kalibracja (IQR/±10%/≤8% range), LossBins P90.

Property-based: monotonic time, brak NaN/Inf; brak dzielenia przez 0 przy niskim RPM.

System:

Foreground service z notyfikacją, WAKE_LOCK opcjonalnie; FEATURE_USB_HOST; permisje runtime „location”.

Tryb awaryjny: fallback do GnssStatus/GnssMeasurements + estymacja HDOP z pdop/acc.

PROMPT (Android – final)

„Zbuduj Android app ASSDyno V7 (GPS dynamometr) w Kotlin + Jetpack Compose, minSdk 26+, targetSdk najnowszy. Struktura pakietów:

data/serial (USB-serial, fallback GNSS), domain/measurement (modele + algorytmy), ui/main (Compose), config (DataStore).
Źródła danych:

USB-serial do u-blox: użyj stabilnej biblioteki USB-Serial dla Androida. Obsłuż enumerację, uprawnienia hosta, reconnect (backoff), log surowych ramek (CSV). Po połączeniu wyślij i potwierdź: CFG-PRT, CFG-RATE (10/20 Hz), CFG-MSG (NAV-PVT, NAV-VELNED, GGA/RMC/VTG), zapisz do BBR/flash i zweryfikuj odczytem.

Fallback GNSS: Fused/Location + GnssStatus/GnssMeasurements; wyznacz HDOP (z PDOP/acc), sats, Age, Hz.

Parsery: NMEA (GGA/RMC/VTG) i UBX (NAV-PVT, opcj. VELNED). Licz Age/Hz z bufora 10 znaczników. Zegar bazowy: elapsedRealtimeNanos() (monotonic) + czas GPS (drift monitor).
Modele: GpsData, MeasurementConfig, MeasurementSession, MeasurementSample, LossStats.
Serwisy: GpsService (USB-serial + fallback), NmeaParser, UbxParser (lub w GpsService), ConfigService (DataStore/JSON), MeasurementProcessor.

MeasurementProcessor (1:1 algorytmy):

Bramki: DeltaTLimitMs=250, DeltaVLimitKmh=8, DeltaVRelPercent=12, HdopMax=1.5, SatsMin=10, AgeMaxMs=400, VmaxKmh=350, IncludeDegraded=false.

Filtry: mediana (okno 5) + EMA (Alpha10Hz=0.35, Alpha20Hz=0.25), AmaxAbs=8.

RPM: rpm = k * v_kmh, wygładzanie BetaRpm=0.25, EpsilonRpm=200.

Moc/moment: P_wheel = m * a_smooth * v_ms / 1000, Torque[Nm] = P_kW * 9549 / RPM.

FSM: Idle → Arm(READY≥1 s/4 próbki) → Countdown(N s) → Accel → Coast(N s) → Done/Abort.

Koniec Accel: średnia a_smooth w oknie EndAccelWindowMs=600 ≤ 0 i rpm ≥ 1200.

LossBins: kosze RpmBin=100, zbieraj P_loss w Coast, raportuj P90 (i P50), min 10 próbek/bin.

Kalibracja k: TargetRpm=3000; zbieraj k=targetRpm/v_kmh; próbek ≥ 60% z NTarget10Hz=50 lub NTarget20Hz=100; usuń outliery (IQR i ±10% od mediany); sukces gdy range ≤ 8%; zapisz dla biegu 3/4; KAdjustmentPercent ±5% (aplikacja tylko na UI RPM lub również na kalkulacje – flaga w config).

Hz: estymuj z dt (EMA), detekcja aliasingu.

UI (Compose):

Pasek statusu: port USB, Hz (zielone ≥ 9.8), drops, mini-log (3 ostatnie).

„GPS Status”: READY/DEGRADED/LOST (kolory), Fix, HDOP, Sats, Hz, Age.

Parametry pojazdu: masa, bin size, A-guard, IncludeDegraded, MinRPM, Coast[s], Countdown[s].

Kalibracja k: wybór biegu (3/4), Target RPM, Start/Cancel, progress, wynik, Save config.

RPM Live: wartość, progress, suwak KAdjustmentPercent (−5..+5), Reset.

Pomiary: Start/Arm, Abort, stan FSM (kolory), countdown, telemetria (v_kmh, a_smooth, P_wheel, Nm).

Wykresy: linie P_wheel / Torque / P_loss (toggles), markery MAX, aktualizacja po Done, Export CSV.

Nawigacja: jeden ekran + dialogi (export, USB perms).

Stan: ViewModel + StateFlow; IO na Dispatchers.IO.

Manifest/Uprawnienia/Feature’y: USB host, foreground service (notification), location runtime, optional WAKE_LOCK.
Pliki i logi: DataStore/JSON (config), CSV (sesja + bins) z rotacją i metadanymi nagłówka.
Testy: unit dla bramek/filtrów/FSM/kalibracji/LossBins, property-based brak NaN/Inf, test aliasingu Hz.
Akceptacja: Stabilny odczyt 10–20 Hz z USB-serial, poprawne Age/Hz, bramki jakości i FSM, wykresy + eksport CSV, READY/Hz zielone ≥ 9.8, skuteczna kalibracja k (range ≤ 8%) i trwały zapis konfiguracji, reconnect bez utraty sesji.