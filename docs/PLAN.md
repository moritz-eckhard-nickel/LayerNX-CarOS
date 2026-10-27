# LayerNX CarOS – Research- & Umsetzungsplan

> Status: **Planung, noch kein Code.** Stand: 27.10.2026.
> Dieses Dokument ist die Grundlage für alle Entscheidungen. Offene Punkte stehen in [Abschnitt 13](#13-offene-fragen-an-moritz) und werden vor dem ersten Code geklärt.
> Behauptungen, die erst auf echter Hardware bewiesen werden müssen, sind mit **⚠ Spike** markiert.

---

## 0. Kurzfassung (TL;DR)

| Frage | Antwort |
|---|---|
| Geht ein eigener Launcher statt Android-Fork? | **Ja, vollständig ohne Root.** Ein Launcher ist eine normale App mit der Rolle „Startbildschirm“ (`ROLE_HOME`). Läuft auf jedem Gerät ab Android 10. |
| Läuft er auf „jedem größeren Gerät“? | **Ja, wenn er von Anfang an adaptiv gebaut wird.** Ab Android 17 (API 37) *muss* er das sowieso: Auf Geräten ≥ 600 dp ignoriert Android Ausrichtungs- und Größensperren. Ein Geräte-Scanner wählt Layout, Animationsstufe und Funktionen automatisch. |
| Android 10 bis 16 – und neuer? | Plan: `minSdk 29` (Android 10), `targetSdk 36` (Pflicht für Google Play seit 31.08.2026), getestet bis **Android 17 (API 37)**, das seit Juni 2026 stabil auf Pixels läuft. Zukunftssicher durch: nur öffentliche APIs, kein Hidden-API-Zugriff im Kern, CI-Tests auf jeder neuen Beta. |
| Android Auto vom Handy auf dem Gerät anzeigen? | **Ja, aber nicht über eine offizielle Google-Schnittstelle.** Drei realistische Wege (Abschnitt 5): **A)** Open Headunit (Open Source) einbinden, **B)** Carlinkit-Dongle per USB, **C)** eigener Modus „LayerNX Link“ (Handy-Apps per scrcpy-Technik spiegeln). Ein komplett eigener Android-Auto-Empfänger von null wird **nicht** empfohlen. |
| Lohnt sich stattdessen ein 40-€-Gerät von AliExpress? | **Kommt drauf an, was „das Gerät“ ist** (Abschnitt 7). Willst du *nur* Android Auto/CarPlay → ein 40–70-€-Linux-Bildschirm mit eingebautem Wireless AA/CarPlay ist zuverlässiger als alles, was wir bauen können – aber darauf läuft unser Launcher **nicht**. Willst du die LayerNX-Oberfläche + eigene Apps + Projektion → Android-Gerät (Tablet oder Android-Radio mit **echten** ≥ 4 GB RAM) + unser Launcher. |
| Empfehlung | Launcher bauen (das ist der Kern und funktioniert garantiert), Projektion als **austauschbares Modul** mit Weg A als Standard und B/C als Optionen. Vorher 4 kurze Spikes auf echter Hardware (Phase 0). |

---

## 1. Ziele & Nicht-Ziele

### Ziele
1. **Launcher statt OS:** LayerNX CarOS ersetzt den Startbildschirm, nicht das Betriebssystem. Installation = APK installieren + als Standard-Startbildschirm wählen.
2. **Läuft überall ab Android 10:** Tablets, Android-Autoradios (2-DIN, 9"/10", 1280×720, 1024×600, 1920×720 Breitbild, 2000×1200 …), alte Handys im Querformat, Foldables.
3. **Automatisches Scannen:** Bildschirm, Leistung, Sensoren, installierte Apps und Radio-Plattform werden erkannt; der Launcher konfiguriert sich selbst. Kein Gerät wird „hart codiert“.
4. **Projektion vom Handy:** Musik, Maps, Messenger vom Handy auf dem Gerät nutzen – so ähnlich wie Android Auto.
5. **LayerNX-Design:** Schwarz / Weiß / Lila-Blau, OLED-schwarz, ruhig, hochwertig, fahrtauglich.
6. **Öffentlich & für alle:** Open Source, verständliche Installationsanleitung, Releases auf GitHub, später F-Droid/IzzyOnDroid.

### Nicht-Ziele (bewusst)
- Kein Android-Fork, keine Custom-ROM, **kein Root als Pflicht** (Root-Extras höchstens optional).
- Kein Nachbau des Google-Android-Auto-Protokolls von null (rechtlich und technisch zu fragil, Abschnitt 5.4).
- Keine Fahrzeugsteuerung (Klima, CAN-Schreibzugriffe). Nur Lesen (z. B. OBD-II-Werte), wenn überhaupt.
- Kein Video während der Fahrt als Standard (Abschnitt 9).

---

## 2. Machbarkeit: Was ein Launcher darf – und was nicht

**Darf (ohne Root, ab Android 10):**
- Startbildschirm sein (`<category android:name="android.intent.category.HOME"/>`, Rolle per `RoleManager.ROLE_HOME` anfragen).
- Alle startbaren Apps auflisten und starten (`LauncherApps`, ab Android 11 mit `<queries>`-Eintrag).
- Widgets anderer Apps anzeigen (`AppWidgetHost`).
- „Jetzt läuft“ jeder Musik-App anzeigen und steuern (Play/Pause/Skip, Cover) – über `MediaSessionManager` + einmalig erteilten Benachrichtigungszugriff.
- Navigation an beliebige Navi-App übergeben (`geo:`- und `google.navigation:`-Intents).
- GPS-Tempo, Uhr, Wetter, Helligkeitssensor für Tag/Nacht.
- USB-Geräte ansprechen (USB-Host: Dongles, Kameras, OBD-Adapter), Bluetooth (OBD-II ELM327).
- Beim Booten automatisch erscheinen – **weil er der Startbildschirm ist**, nicht über einen Autostart-Trick.

**Darf nicht / nur eingeschränkt:**
- Fremde Apps (Google Maps, Spotify) *in* den Launcher einbetten. Ein Launcher kann eine fremde App nicht als Fenster in seiner Oberfläche anzeigen. Ausweg: Split-Screen/Freeform, wo das Gerät es anbietet (⚠ Spike, herstellerabhängig), oder eine eigene Karte (MapLibre/OSM) im Dashboard und Maps-App per Tipp öffnen.
- Systemeinstellungen ändern (Bildschirm-Timeout, Lautstärkelimits) – nur über die Einstellungs-Seiten des Systems.
- Lenkradtasten des Autoradios direkt lesen: Das macht bei Android-Radios die Hersteller-Firmware (FYT, MTCD …). Viele leiten Medientasten als normale `KEYCODE_MEDIA_*` weiter – die können wir nutzen. Herstellerspezifische Broadcasts nur als optionale Plugins (⚠ Spike pro Plattform).
- Bildschirm beim Einstecken des Ladekabels einschalten: Auf vielen Tablets nur, wenn das Gerät es selbst kann. Ohne Root gibt es dafür keine zuverlässige API (⚠ Spike, Abschnitt 7.4).

**Fazit:** Der Launcher-Teil ist zu 100 % machbar. Unsicher ist nur die Projektion – deshalb wird sie modular gebaut.

---

## 3. Android-Versionen 10 → 17+: was sich ändert und wie wir es abfangen

| Android | API | Relevante Änderung | Maßnahme im Launcher |
|---|---|---|---|
| 10 | 29 | `RoleManager` für Home-Rolle; Hintergrund-Activity-Starts eingeschränkt; Scoped Storage | Rolle sauber anfragen; nichts aus dem Hintergrund starten; keine Dateipfade, nur SAF/MediaStore |
| 11 | 30 | **Package Visibility** – Apps sehen andere Apps nur mit `<queries>`; Wireless Debugging (für Weg C) | `<queries>` für LAUNCHER, MediaBrowserService, `geo:`; **kein** `QUERY_ALL_PACKAGES` (Play-Richtlinie) |
| 12 | 31 | Neue Bluetooth-Rechte `BLUETOOTH_SCAN/CONNECT`; `RenderEffect` (Blur); Splash-Screen-API; Material You | Rechte versionsabhängig; Blur nur ab 31, darunter halbtransparente Flächen |
| 12L | 32 | Taskbar/Large-Screen-Optimierungen | Layout testen mit Taskbar-Insets |
| 13 | 33 | `POST_NOTIFICATIONS`-Recht; themed Icons | Nur anfragen, wenn wirklich nötig; monochrome Icons für LayerNX-Look nutzen |
| 14 | 34 | Foreground-Service-Typen Pflicht; strengere Implicit-Intent-Regeln | Dienste mit `connectedDevice`/`location`/`mediaProjection` deklarieren |
| 15 | 35 | **Edge-to-Edge erzwungen**; 16-KB-Speicherseiten (native Libs) | Alle Insets selbst behandeln; native Libs 16-KB-ausgerichtet bauen |
| 16 | 36 | Auf ≥ 600 dp werden Orientierungs-/Größensperren ignoriert (Opt-out noch möglich); **Predictive Back**: `onBackPressed` wird nicht mehr aufgerufen | Von Anfang an voll adaptiv; Zurück über `OnBackInvokedCallback` |
| 17 | 37 | **Opt-out entfällt** – Launcher *muss* jede Größe/Ausrichtung können | Ist durch adaptives Design abgedeckt; CI testet API 37 |
| 18+ | 38+ | unbekannt | Jede neue Beta in CI aufnehmen; nur öffentliche APIs; jährlicher `targetSdk`-Sprung als festes Ritual |

**Build-Konfiguration (Vorschlag):** `minSdk 29`, `targetSdk 36`, `compileSdk 37`, AGP 9.x mit eingebautem Kotlin (wie in `kalorien-arschloch`), Java/Kotlin-Ziel 17.

---

## 4. Funktionsumfang

### 4.1 MVP (Version 0.1 – „fährt mit“)
- Startbildschirm mit **Dashboard**: Uhr/Datum, „Jetzt läuft“-Karte, Navi-Schnellstart, App-Dock (6–8 Favoriten).
- **App-Drawer** mit Suche, alphabetisch, Kategorien (Musik, Navigation, Telefon, Sonstiges – automatisch erkannt).
- **Geräte-Scanner** (Abschnitt 6) mit automatischer Layout-Wahl.
- **Tag/Nacht** automatisch (Sonnenauf-/-untergang aus GPS, Fallback Uhrzeit, optional Lichtsensor).
- Einrichtungsassistent: Home-Rolle, Benachrichtigungszugriff, Standort – jeweils mit Erklärung, alles überspringbar.
- Deutsch + Englisch.

### 4.2 Version 0.5 – „Projektion“
- Projektions-Hub mit Adaptern A/B/C (Abschnitt 5), ein Knopf „Handy verbinden“.
- Nach Ende der Projektion automatisch zurück zum Dashboard.

### 4.3 Version 1.0 – „fertig für alle“
- Widgets anderer Apps, frei anordnbares Dashboard (Bearbeitungsmodus per Lange-Drücken).
- GPS-Tacho, optional OBD-II (ELM327 per Bluetooth): Drehzahl, Kühlwasser, Spannung.
- Lenkrad-/Medientasten, D-Pad/Drehregler-Bedienung (Fokusnavigation).
- Rückfahrkamera per USB-UVC (optional, ⚠ Spike).
- Backup/Import der Einstellungen (ZIP, wie in `kalorien-arschloch`).
- Themes nur innerhalb LayerNX (Akzent-Varianten), kein Wildwuchs.

### 4.4 Später
- Plugin-Schnittstelle für Radio-Plattformen (FYT, MTCD/MTCE, TopWay …).
- Sprachsteuerung (Assistent-Intent weiterreichen).
- Mehrere Profile (Fahrer A/B).

---

## 5. Projektion („wie Android Auto“) – die ehrliche Analyse

Android Auto ist ein **proprietäres, verschlüsseltes Google-Protokoll**. Es gibt **keine** offizielle Möglichkeit für eine normale App, ein Android-Auto-Empfänger zu sein. Alle Lösungen sind Reverse Engineering oder Umwege. Deshalb: Projektion als **Adapter-Schnittstelle** (`ProjectionAdapter`), damit wir Wege tauschen können, wenn Google etwas kaputtmacht.

### Weg A – Open Headunit einbinden (Standard-Empfehlung)
- **Was:** [Open Headunit](https://github.com/andreknieriem/headunit-revived) (früher „Headunit Revived“), Open Source unter **AGPL-3.0**, macht ein Android-Tablet zum Android-Auto-Empfänger. Kostenlos im Play Store.
- **Verbindung:** USB (Handy per Kabel/OTG), WLAN über den „Headunit Server“ (Android-Auto-Entwickleroptionen, Port 5277), Wi-Fi Direct, Hotspot, Helper-App.
- **Integration:** Die App lässt sich per Intent steuern: `headunit://connect?ip=<IP>` und `headunit://exit`. Unser Launcher erkennt, ob sie installiert ist, zeigt eine „Android Auto“-Kachel und kehrt nach `exit` zurück.
- **Vorteile:** Echte Android-Auto-Oberfläche (Maps, Spotify, WhatsApp – alles, was AA kann), kein eigener Protokoll-Code, aktiv gepflegt.
- **Nachteile/Risiken:**
  - Laut README hat **Android Auto 17.4** fast alle bisherigen kabellosen Auslöser (inkl. Wireless Helper/Self-Mode) gebrochen; kabellos geht dann nur noch über den manuell gestarteten Headunit Server (oder Root-Workaround). **USB-Kabel ist der zuverlässige Weg.**
  - Google kann jederzeit weiter brechen. Das trifft dann alle solchen Apps gleichzeitig.
  - Nutzer muss in Android Auto die Entwickleroptionen freischalten (10× auf Version tippen) – das erklären wir im Assistenten.
- **Lizenz-Hinweis:** Solange wir Open Headunit nur **per Intent starten**, bleibt unser Code lizenzfrei wählbar. Würden wir den Code **einbauen**, muss LayerNX CarOS auch AGPL-3.0 sein (Frage 16).
- **Alternative im selben Weg:** „Headunit Reloaded“ (kommerziell, ca. 5 €, Closed Source) – gleiche Idee, als zweiter erkannter Anbieter unterstützbar.

### Weg B – Carlinkit-Dongle per USB
- **Was:** Ein USB-Dongle (z. B. Carlinkit CPC200-CCPA, ca. 50–90 €) wird ans Android-Gerät gesteckt. Das **Handy verbindet sich kabellos mit dem Dongle** (CarPlay und laut Hersteller auch Android Auto). Der Dongle liefert ein fertiges Videobild + Audio über USB.
- **Warum interessant:** Der Dongle macht den „zertifizierten“ Teil (bei CarPlay mit Apple-Chip). **Einziger Weg, auch iPhones (CarPlay) zu unterstützen.** Kabellos ohne Entwickleroptionen.
- **Wie wir es ansprechen:** Das USB-Protokoll ist durch das MIT-lizenzierte Projekt [node-CarPlay](https://github.com/rhysmorgan134/node-CarPlay) dokumentiert (Node.js/WebUSB). Wir portieren das Protokoll nach Kotlin (Android-USB-Host-API) und dekodieren das H.264-Video mit `MediaCodec` (Hardware).
- **Risiken:** node-CarPlay dokumentiert nur **CarPlay**; ob Android-Auto-Modus über denselben Weg kommt, ist **⚠ Spike**. Dongles, die nur kabelgebundenes CarPlay „kabellos machen“, funktionieren laut node-CarPlay **nicht**. Firmware-Updates des Dongles können das Protokoll ändern. Alternativ kann der Hersteller-App „AutoKit“ genutzt werden – dann nur als Kachel, wie bei Weg A.
- **Aufwand:** mittel bis hoch (USB-Protokoll, Video, Audio, Touch-Rückkanal, Mikrofon).

### Weg C – „LayerNX Link“ (eigene Erfindung, Open Source, ohne Google)
- **Idee:** Wie [scrcpy](https://github.com/Genymobile/scrcpy) (Apache-2.0): Seit scrcpy 3.0 kann auf dem Handy ein **eigener virtueller Bildschirm** in Wunschgröße erzeugt und eine App darauf gestartet werden (`--new-display=1920x1080 --start-app=…`). Das Handy-Display bleibt frei, die App läuft **in Tablet-Auflösung** und wird auf das Gerät gestreamt; Touch geht zurück.
- **Unser Launcher wäre der scrcpy-Client** (ADB-Verbindung per USB oder WLAN, Video per `MediaCodec`).
- **Vorteile:** **Jede** Handy-App (nicht nur AA-freigegebene), kein Google-Protokoll, kein Dongle, Lizenz sauber, sehr geringe Latenz per USB.
- **Nachteile/Risiken:**
  - Handy braucht **USB-Debugging bzw. Wireless Debugging** (Entwickleroptionen). Ab Android 11 Pairing per Code; Wireless Debugging geht nach Neustart/WLAN-Wechsel aus → per USB am stabilsten.
  - Apps mit `FLAG_SECURE` (Netflix, Banking) bleiben schwarz.
  - Audio-Weiterleitung erst ab Android 11 auf dem Handy; einfacher: Handy-Ton per Bluetooth ans Autoradio.
  - Verhalten bei gesperrtem Handy / ausgeschaltetem Handy-Display **⚠ Spike**.
  - Sicherheit: ADB-Zugriff = Vollzugriff aufs Handy. Schlüssel geschützt speichern, nur nach ausdrücklicher Bestätigung.
  - Keine „Auto-Oberfläche“: Apps sehen aus wie auf einem Tablet (für Maps & Spotify gut, für Messenger ok).
- **Aufwand:** mittel (ADB-Client in Kotlin, z. B. auf Basis der Bibliothek *dadb* – Eignung **⚠ Spike**; scrcpy-Server-JAR wird mitgeliefert).

### 5.4 Nicht empfohlen – eigener Android-Auto-Empfänger von null
Ein eigener Empfänger müsste Googles Protokoll inklusive TLS-Handshake mit einem Head-Unit-Zertifikat nachbauen. Bestehende Projekte nutzen dafür Zertifikate aus echten Geräten. Das ist rechtlich heikel (Markenrecht, Google-Bedingungen), Google kann es jederzeit sperren, und es wäre jahrelange Pflegearbeit, die Open Headunit schon leistet. **Nicht unser Kampf.**

### 5.5 Vergleich

| | A: Open Headunit | B: Dongle | C: LayerNX Link |
|---|---|---|---|
| Echte Android-Auto-Oberfläche | ✅ | ✅ (⚠ Spike) | ❌ (Handy-Apps direkt) |
| iPhone / CarPlay | ❌ | ✅ | ❌ |
| Kabellos | ⚠ seit AA 17.4 eingeschränkt | ✅ | ⚠ Wireless Debugging |
| Zusatzkosten | 0 € | 50–90 € | 0 € |
| Abhängig von Google | stark | mittel | gering |
| Unser Aufwand | gering | hoch | mittel |
| Einrichtung für Laien | mittel (Entwickleroptionen AA) | einfach | schwer (Entwickleroptionen Handy) |

**Empfehlung:** Reihenfolge **A → C → B**. A bringt sofort echtes Android Auto. C ist das Alleinstellungsmerkmal (Open Source, ohne Google). B nur, wenn iPhones wichtig sind (Frage 7).

---

## 6. Automatisches Scannen („Device Profiler“)

Beim ersten Start, bei jeder Konfigurationsänderung (Drehen, Fenstergröße, Monitor) und bei App-Installationen läuft ein Scan. Ergebnis ist ein `DeviceProfile`, aus dem alles andere abgeleitet wird. Jede Entscheidung ist in den Einstellungen sichtbar und überschreibbar („Automatisch ✔ / Manuell“).

### 6.1 Was gescannt wird

| Bereich | Quelle (öffentliche API) | Wofür |
|---|---|---|
| Fenstergröße, Breite/Höhe in dp, Seitenverhältnis | `WindowMetrics` / Jetpack WindowManager | Layout-Klasse |
| Dichte, physische Größe (Zoll) | `DisplayMetrics` | Schriftgrößen, Touch-Ziele |
| Ausrichtung, Insets (Statusleiste, Taskbar, Notch) | `WindowInsets` | Edge-to-Edge |
| RAM, „Low-RAM“-Gerät, CPU-Kerne | `ActivityManager.MemoryInfo`, `isLowRamDevice` | Leistungsstufe |
| Hardware-Videodekoder (H.264/H.265, max. Auflösung/FPS) | `MediaCodecList` | Ob Projektion in 1080p/720p möglich ist |
| GPS, Bluetooth, USB-Host, Lichtsensor, Mobilfunk | `PackageManager.hasSystemFeature`, `SensorManager` | Funktionen ein-/ausblenden |
| Radio-Plattform (FYT, MTCD/MTCE, TopWay, …) | `Build.MANUFACTURER/BOARD/HARDWARE` + lesbare System-Properties | Optionale Plattform-Plugins (Lenkradtasten, Rückfahrkamera) |
| Installierte Apps nach Fähigkeit | Intent-Abfragen: Launcher-Apps, `MediaBrowserService`, `geo:`, `tel:` | Kategorien, Vorschläge für Dock |
| Projektions-Anbieter | Paketnamen/Intents von Open Headunit, Headunit Reloaded, AutoKit/Zlink | Projektions-Kachel |

### 6.2 Layout-Klassen (Vorschlag)

| Klasse | Typische Geräte | Aufbau |
|---|---|---|
| **Bar** (Seitenverhältnis ≥ 2.4:1) | 1280×480, 1920×720 Breitbild-Radios | Dock links, Karten nebeneinander, große Uhr |
| **Wide** (Querformat, ≥ 840 dp breit) | 10"-Radios, Tablets quer | Dock + 2×2-Kartenraster |
| **Medium** (600–840 dp) | 7"-Radios 1024×600, kleine Tablets | Dock unten, 2 Karten |
| **Portrait** | Hochformat-Radios (Tesla-Stil), Tablet hoch | Karten untereinander, Dock unten |
| **Compact** (< 600 dp) | Handy als Testgerät | Einspaltig, nur zum Testen/Notfall |

### 6.3 Leistungsstufen
- **Hoch** (≥ 4 GB, modernes SoC): Animationen, Blur (ab Android 12), animierte Hintergründe.
- **Mittel** (2–4 GB): Animationen kurz, kein Blur.
- **Niedrig** (< 2 GB oder Low-RAM): keine Effekte, statisches Schwarz, App-Icons kleiner cachen.

---

## 7. Hardware: Tablet vs. AliExpress-Radio vs. 40-€-Bildschirm

### 7.1 Die drei Geräteklassen

| | Altes/günstiges Android-Tablet | Android-Autoradio (2-DIN) | „Portable CarPlay/AA Screen“ (7–10") |
|---|---|---|---|
| Preis | 0–150 € | **40–80 €** (Billigklasse) / 150–350 € (UIS7862-Klasse) | **40–80 €** |
| Betriebssystem | Android | Android (oft ältere Version als beworben) | **meist Linux, kein Android** |
| Unser Launcher läuft? | ✅ | ✅ | ❌ |
| Android Auto/CarPlay | über Weg A/B/C | oft vorinstalliert (Zlink/AutoKit) + Weg A/B/C | ✅ **eingebaut, kabellos** |
| Einbau | Halterung, Kabel | fester Einbau, Zündungsplus, Lenkradtasten, Rückfahrkamera | Saugnapf + Zigarettenanzünder |
| Typische Probleme | Akku + Hitze, Bildschirm geht nicht automatisch an | **gefälschte RAM/Android-Angaben**, schwaches GPS, Leistung | keine eigenen Apps |

### 7.2 Ehrliche Einschätzung der 40-€-Android-Radios
- Häufig **1–2 GB RAM**, manchmal weniger als auf der Packung (Berichte über 1-GB-Chips in „2-GB“-Geräten). Auch die angezeigte Android-Version ist bei Billiggeräten oft geschönt. **Nach dem Kauf mit einer Info-App (z. B. CPU-Z/AIDA64) prüfen.**
- Der Launcher selbst läuft darauf (Stufe „Niedrig“), aber Android-Auto-Projektion über Weg A/C dekodiert ständig Video → bei schwachen Chips ruckelig.
- **Faustregel:** Für unseren Launcher + Projektion **mindestens 2 GB echte RAM, besser 4 GB**, Hardware-H.264-Dekoder 1080p.

### 7.3 Empfehlung nach Ziel
- **„Ich will nur Android Auto im Auto, sofort, zuverlässig“** → 40–80-€-Portable-Screen. Kein Projekt nötig.
- **„Ich will die LayerNX-Oberfläche, eigene Apps, und Android Auto dazu“** → Android-Tablet (das du schon hast?) oder Android-Radio mit echten ≥ 4 GB + LayerNX CarOS.
- **„Fester Einbau, alles integriert“** → Android-Radio UIS7862-Klasse (oder vergleichbar) + LayerNX CarOS + optional Carlinkit-Dongle.

### 7.4 Tablet im Auto – Pflichtwarnungen
- **Lithium-Akku + Sommerhitze** hinter der Windschutzscheibe (deutlich über 60 °C möglich) → Akku kann sich aufblähen. Nicht in praller Sonne lassen, wenn möglich Gerät ohne Akku/mit Bypass – sonst rausnehmen.
- **Halterung nicht im Airbag-Bereich**, Sicht nicht verdecken.
- **Automatisch einschalten mit Zündung:** viele Tablets schalten sich beim Laden nicht selbst ein. Lösungen je nach Gerät (Hersteller-Option, Bootloader-Befehl „boot on charge“ – nur für Fortgeschrittene). ⚠ Spike auf deinem Gerät.

---

## 8. Design: LayerNX-Stil

> Endgültige Farben werden aus dem Logo abgeleitet (kommt im nächsten Prompt). Bis dahin Platzhalter.

### 8.1 Design-Tokens (Platzhalter)
| Token | Wert | Verwendung |
|---|---|---|
| `bg` | `#000000` | Hintergrund (OLED-schwarz) |
| `surface` | `#0E0E14` | Karten |
| `surfaceHigh` | `#1A1A24` | gedrückte/aktive Flächen |
| `onBg` | `#FFFFFF` | Primärtext |
| `onBgMuted` | `#A0A0B0` | Sekundärtext (Kontrast auf Schwarz ≥ 7:1 prüfen) |
| `accent` | `#7B6CFF` (Lila-Blau) | Aktive Elemente, Fortschritt, Fokus |
| `accentGlow` | `#7B6CFF` @ 30 % | Leuchten/Fokusring |
| `danger` | `#FF4D6A` | Warnungen |

- **Tag-Modus:** Standard bleibt dunkel (blendet nachts nicht, auf OLED stromsparend); tagsüber nur höhere Helligkeit/Kontrast. Ein echter heller Modus ist optional – Frage 11.
- **Typografie:** eine geometrische Sans (Vorschlag: *Inter* oder *Space Grotesk*, mitgeliefert, Open-Font-Lizenz). Frage 12.
- **Icons:** einheitlich monochrom (weiß) mit Akzent bei aktiv; App-Icons optional monochrom (themed icons ab Android 13, darunter Graustufen-Filter).

### 8.2 Fahrtauglichkeit (nicht verhandelbar)
- **Touch-Ziele ≥ 76 dp** (Richtwert aus Googles Android-for-Cars-Design).
- **Ein Blick ≤ 2 Sekunden**, Aufgaben ≤ 3 Taps (angelehnt an die NHTSA-Richtlinien zur Fahrerablenkung).
- Schrift: Primärinfos ≥ 24 sp, nichts unter 18 sp im Fahrmodus.
- Kein Scroll-Zwang auf dem Dashboard; App-Drawer mit großen Kacheln und Buchstaben-Schnellsprung.
- Animationen ≤ 250 ms, nichts Blinkendes.
- Vollständig per D-Pad/Drehregler/Lenkradtaste bedienbar (sichtbarer Fokusring in Akzentfarbe).

### 8.3 Bildschirme
1. **Dashboard** (Uhr, Jetzt läuft, Navi, Tacho/Wetter, Dock)
2. **App-Drawer**
3. **Projektion** (Vollbild, nur „Zurück zu LayerNX“-Geste)
4. **Medien-Vollbild** (großes Cover, Quellenwahl)
5. **Einstellungen** (Gerät/Scan-Ergebnis, Darstellung, Projektion, Fahrzeug, Backup, Über)
6. **Einrichtungsassistent**

Vor dem Bauen: klickbarer Entwurf (HTML-Prototyp oder Figma) zur Abstimmung – Frage 13.

---

## 9. Sicherheit & Recht

- **StVO § 23 Abs. 1a:** Bedienung nur mit kurzem Blick erlaubt. Daraus folgt: Fahrmodus mit reduzierter Oberfläche; Video-Apps standardmäßig nicht im Dock; Hinweis beim Einrichten. Der Launcher ersetzt keine Verantwortung des Fahrers – steht so im README.
- **Marken:** „Android Auto“, „CarPlay“, „Google Maps“ sind Marken. Projekt- und App-Name ohne diese Begriffe; in der Doku nur beschreibend („kompatibel mit …“ vermeiden, besser „nutzt Open Headunit“).
- **Datenschutz:** Kein Tracking, kein Konto, keine Cloud. Standort bleibt auf dem Gerät. Optionaler Absturzbericht nur per Opt-in (Frage 20).
- **Lizenzen:** Alle Abhängigkeiten in einer Lizenzseite in der App (Pflicht bei Apache/MIT/OFL).

---

## 10. Technische Architektur

### 10.1 Technologie (Vorschlag – siehe Frage 14)
- **Kotlin**, **Jetpack Compose** für die Oberfläche (adaptive Layouts sind damit deutlich einfacher und sicherer als mit Views), minimale AndroidX-Auswahl (`core`, `activity`, `window`, `compose`), sonst Framework.
- Datenhaltung: Einstellungen in `SharedPreferences`/DataStore, Layout des Dashboards als JSON (exportierbar).
- Kein Firebase, keine Google Play Services → läuft auch auf Radios ohne Google-Dienste und ist F-Droid-tauglich.

### 10.2 Module
```
:app                    Einstieg, Home-Activity, Navigation, Einrichtung
:core:design            LayerNX-Tokens, Typografie, Komponenten (Karte, Dock, Kachel, Fokusring)
:core:device            Device Profiler (Scan, Layout-Klasse, Leistungsstufe, Plattform)
:core:apps              App-Liste, Kategorien, Icons-Cache, Paket-Änderungen
:feature:dashboard      Dashboard + Karten
:feature:media          MediaSession-Erkennung, Jetzt-läuft, Medientasten
:feature:navigation     Navi-Übergabe, optionale Karte
:feature:projection     ProjectionAdapter-API + Adapter A (Open Headunit), B (Dongle), C (LayerNX Link)
:feature:vehicle        GPS-Tempo, OBD-II (optional)
:feature:settings       Einstellungen, Backup, Lizenzen
:platform:fyt …         optionale Plugins für Radio-Plattformen
```

### 10.3 Wichtige Schnittstellen
```kotlin
interface ProjectionAdapter {
    val id: String                       // "open-headunit", "carlinkit", "layernx-link"
    fun availability(profile: DeviceProfile): Availability   // Installiert? Hardware da? Codec ausreichend?
    fun start(context: Context, request: ProjectionRequest)  // startet/verbindet
    fun stop(context: Context)
}
```
Der Launcher kennt nur diese Schnittstelle. Bricht ein Weg, wird nur sein Adapter repariert oder deaktiviert.

### 10.4 Berechtigungen (vollständige Liste, alle mit Begründung im Assistenten)
| Recht | Wann | Pflicht? |
|---|---|---|
| Home-Rolle | Einrichtung | ja |
| Benachrichtigungszugriff | „Jetzt läuft“ | nein |
| `ACCESS_FINE_LOCATION` | Tacho, Tag/Nacht, Wetter | nein |
| `BLUETOOTH_CONNECT/SCAN` (ab 12) | OBD-II | nein |
| USB-Gerätezugriff | Dongle, Kamera, LayerNX Link | nein |
| `INTERNET` | Wetter, LayerNX Link per WLAN | nein |
| `REQUEST_INSTALL_PACKAGES` | Nur GitHub-Variante: Selbst-Update | nein |

---

## 11. Qualität & „Production Ready“

### 11.1 Tests
- **Unit-Tests** für Profiler-Logik (Layout-Klasse aus Maßen, Leistungsstufe), App-Kategorisierung, Layout-Serialisierung.
- **Screenshot-Tests** jeder Layout-Klasse in Tag/Nacht.
- **Emulator-Matrix in GitHub Actions** (wie in `kalorien-arschloch`, erweitert):
  - APIs: 29, 30, 31, 33, 34, 35, 36, 37
  - Bildschirme: 1024×600, 1280×720, 1280×480, 1920×720, 2000×1200, Hochformat 768×1024, Handy 1080×2400
  - Rauchtest: Starten, als Home setzen, durch alle Bildschirme, drehen, Fenstergröße ändern, kein Absturz.
- **Echte Geräte** vor jedem Release (Frage 3): mindestens ein Tablet, ein Android-Radio, ein Android-10-Gerät.

### 11.2 Leistungsbudget
- Kaltstart < 1,5 s auf Stufe „Mittel“, Rückkehr zum Home < 300 ms.
- 60 fps beim Scrollen im App-Drawer (Baseline Profiles).
- Speicher im Leerlauf < 150 MB.

### 11.3 Releases
- **Echter Release-Schlüssel** (nicht der öffentliche Debug-Schlüssel wie bei `kalorien-arschloch`!). Bei einem öffentlichen Launcher könnte sonst jeder ein „Update“ mit Schadcode signieren, das sich über die echte App installiert. Schlüssel als GitHub-Secret, Backup offline.
- Semantische Versionen, Changelog, signierte APKs auf GitHub Releases, Prüfsummen.
- Später: F-Droid oder IzzyOnDroid, Obtainium-kompatibel. Google Play optional (Frage 18).
- Zwei Varianten: `github` (mit Selbst-Update-Hinweis) und `store` (ohne).

### 11.4 Community
- README Deutsch/Englisch, Installationsanleitung mit Bildern, Gerätekompatibilitätsliste (Nutzer melden per Issue-Vorlage: Gerät, Android-Version, Scan-Export).
- „Scan exportieren“-Knopf: erzeugt anonymisierten Geräte-Bericht für Bug-Reports.
- Issue-Vorlagen, Beitragsrichtlinie, Verhaltenskodex.

---

## 12. Fahrplan

| Phase | Inhalt | Ergebnis / Abnahme |
|---|---|---|
| **0 – Spikes** (zuerst!) | S1: Open Headunit per Intent auf deinem Gerät starten und wieder zurückkehren. S2: LayerNX-Link-Prototyp: virtuelles Display vom Handy per USB anzeigen. S3: Profiler-Prototyp auf 3–5 Geräten/Emulatoren. S4: (nur wenn Frage 7 = ja) Dongle per USB: erstes Videobild. | Kurzer Bericht je Spike: geht / geht nicht / Einschränkungen. Danach endgültige Projektionsentscheidung. |
| **1 – Fundament** | Projektgerüst, CI, Design-System, Home-Rolle, App-Drawer, Profiler | Launcher ist als Startbildschirm nutzbar, alle CI-Emulatoren grün |
| **2 – Dashboard** | Karten, Jetzt läuft, Navi, Tag/Nacht, Einrichtungsassistent | **v0.1** auf GitHub |
| **3 – Projektion** | Adapter A, dann C (B optional) | **v0.5** |
| **4 – Auto-Funktionen** | Widgets, Bearbeitungsmodus, Tacho, Tasten/D-Pad, OBD optional | **v0.9 Beta** – öffentliche Testphase |
| **5 – Release** | Feinschliff, Barrierefreiheit, Übersetzungen, Doku, Kompatibilitätsliste | **v1.0** |

Jede Phase endet mit einem lauffähigen, getesteten Stand. Keine Phase beginnt, bevor die vorherige grün ist.

### Risiken
| Risiko | Wahrscheinlichkeit | Wirkung | Gegenmaßnahme |
|---|---|---|---|
| Google bricht Android-Auto-Projektion weiter (wie AA 17.4) | hoch | hoch | Adapter-Architektur, Weg C ohne Google, USB als stabiler Pfad |
| Hersteller-Launcher lässt sich auf Radio nicht ersetzen | mittel | mittel | Kompatibilitätsliste; Launcher funktioniert auch als normale App |
| Billig-Hardware zu schwach | hoch | mittel | Leistungsstufen, klare Mindestanforderungen |
| Neue Android-Version ändert Launcher-Verhalten | mittel | mittel | Beta-Tests in CI, nur öffentliche APIs |
| Rechtliche Probleme durch Markennamen | gering | hoch | Neutrale Benennung, keine fremden Logos |

---

## 13. Offene Fragen an Moritz

Antworten werden hier nachgetragen. In Klammern steht mein Vorschlag, falls du keine Meinung hast.

**Hardware & Nutzung**
1. Welches Gerät soll es konkret werden (Hersteller, Modell, Android-Version, Bildschirmgröße/Auflösung)? Oder kaufst du erst noch eins?
2. Fester Einbau als 2-DIN-Radio oder Tablet mit Halterung? (Vorschlag: erst Tablet zum Entwickeln, später Radio)
3. Welches Handy nutzt du fürs Auto (Z Flip 4?), welche Android- und Android-Auto-Version?
4. Hardware-Budget? (unter 50 € / 50–150 € / 150–350 €)
5. Welches Auto (Marke, Modell, Baujahr)? Wichtig für Einbau, OBD-II und Lenkradtasten.

**Projektion**
6. Was ist dir wichtiger: die echte Android-Auto-Oberfläche (Weg A) oder *jede* Handy-App ohne Google-Abhängigkeit (Weg C)? Oder beides? (Vorschlag: beides, A zuerst)
7. Sollen iPhone-Nutzer (CarPlay) unterstützt werden? Das geht nur mit Dongle (Weg B, 50–90 €). (Vorschlag: später, optional)
8. Ist ein USB-Kabel zum Handy okay, oder muss es zwingend kabellos sein? (Vorschlag: Kabel als stabiler Standard, kabellos als Option)
9. Ist es für dich (und andere Nutzer) okay, auf dem Handy Entwickleroptionen zu aktivieren (Android-Auto-Entwicklermodus bzw. USB-Debugging)?

**Funktionen**
10. Welche Apps müssen perfekt laufen? Musik (Spotify, YouTube Music, Apple Music …), Navi (Google Maps, Waze, OsmAnd …), Sonstiges (WhatsApp, Telefon, Podcasts)?
11. Tag-Modus: immer dunkel oder automatisch heller Modus am Tag? (Vorschlag: immer dunkel, tags heller)
12. Eigene Karte direkt im Dashboard (Open-Source-Karte, offline-fähig) oder reicht ein großer Knopf, der Maps/Waze öffnet? (Vorschlag: Knopf im MVP, Karte später)
13. Fahrzeugdaten per OBD-II (Drehzahl, Temperatur, Spannung)? Hast du einen ELM327-Bluetooth-Adapter? (Vorschlag: v1.0, optional)
14. Brauchst du Rückfahrkamera- oder Lenkradtasten-Unterstützung?
15. Dashboard frei anordnen (Bearbeitungsmodus) oder fest durchdesignt mit wenigen Varianten? (Vorschlag: fest im MVP, frei ab v1.0)
16. Wetter auf dem Dashboard? (Vorschlag: ja, über Open-Meteo – kostenlos, ohne Konto, ohne API-Schlüssel)
17. Optionale Root-Extras erlauben (z. B. Bildschirm bei Zündung einschalten), oder strikt ohne Root? (Vorschlag: Kern ohne Root, Extras klar getrennt)

**Design**
18. Logo: als SVG möglich? Soll das App-Icon und ein kurzer animierter Start-Bildschirm daraus entstehen?
19. Schrift: Vorliebe? (Vorschlag: Inter oder Space Grotesk)
20. Willst du vor dem Code einen klickbaren Design-Prototyp sehen und freigeben? (Vorschlag: ja, spart später Umbauten)

**Technik, Projekt & Veröffentlichung**
21. Technik: Jetpack Compose mit wenigen AndroidX-Bibliotheken, oder wie bei `kalorien-arschloch` nur Framework ohne AndroidX? (Vorschlag: Compose – adaptive Layouts für alle Bildschirmgrößen sind damit deutlich robuster)
22. Lizenz: AGPL-3.0, GPL-3.0, Apache-2.0 oder MIT? (Vorschlag: GPL-3.0 – frei für alle, Verbesserungen bleiben offen; AGPL nötig, falls wir Open-Headunit-Code einbauen)
23. Name und Paketname final: „LayerNX CarOS“ und `com.layernx.caros`?
24. Verteilung: nur GitHub Releases, zusätzlich F-Droid/IzzyOnDroid, oder auch Google Play? (Vorschlag: GitHub + IzzyOnDroid, Play später)
25. Sprachen: Deutsch + Englisch reichen? (Vorschlag: ja, Übersetzungen später per Community)
26. Absturzberichte: gar keine, oder freiwillig per Opt-in (lokal gespeichert, per Knopf teilbar)? (Vorschlag: Opt-in, kein Server)
27. Zeitrahmen und Priorität: lieber schnell ein schöner Launcher ohne Projektion (v0.1), oder erst veröffentlichen, wenn Projektion drin ist?

---

## 14. Quellen

- Open Headunit (ehem. Headunit Revived), AGPL-3.0, Intents & AA-17.4-Hinweis: https://github.com/andreknieriem/headunit-revived
- node-CarPlay (MIT), Carlinkit-Dongle-Protokoll: https://github.com/rhysmorgan134/node-CarPlay
- scrcpy (Apache-2.0), virtuelle Displays seit 3.0: https://github.com/Genymobile/scrcpy
- Android 17: Ausrichtung/Größe auf großen Bildschirmen: https://developer.android.com/blog/posts/prepare-your-app-for-the-resizability-and-orientation-changes-in-android-17
- Google Play Target-API-Vorgaben (API 36 ab 31.08.2026): https://developer.android.com/google/play/requirements/target-sdk
- Headunit Reloaded / Anleitungen Tablet als AA-Bildschirm: https://www.androidplanet.nl/tips/android-auto-op-je-tablet/ , https://www.xatakandroid.com/tutoriales/como-convertir-tu-vieja-tablet-android-auto-para-tu-coche
- Gefälschte RAM-Angaben bei Billig-Radios (Forenberichte): https://www.team-bhp.com/forum/car-entertainment/210960-android-head-unit-buying-guide-5.html
