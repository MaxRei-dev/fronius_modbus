[![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg)](https://github.com/custom-components/hacs)

# fronius_modbus

Dies ist ein Fork von redpomodoro/fronius_modbus, mit zusammengeführten Änderungen und Pull Requests.

Home Assistant Custom Component zum Auslesen von Daten aus Fronius GEN24- und Verto-Wechselrichtern, angeschlossenen Smartmetern und Batteriespeichern. Diese Integration verwendet ein Modbus-first + authentifiziertes Web-API-Verbindungsmodell.

Die authentifizierte Fronius Web API kann für die Einrichtungsunterstützung und Batteriesteuerungen verwendet werden, die über Modbus nicht verfügbar sind.

> [!CAUTION]
> Dies ist ein laufendes Projekt – es befindet sich noch in einer frühen Entwicklungsphase, daher sind noch Breaking Changes möglich.
>
> Dies ist eine inoffizielle Implementierung und wird von Fronius nicht unterstützt. Sie kann jederzeit aufhören zu funktionieren.
> Du verwendest dieses Modul (und seine Voraussetzungen/Abhängigkeiten) auf eigenes Risiko. Weder ich noch andere Mitwirkende an diesem oder abhängigen Projekten sind für Schäden jeglicher Art verantwortlich, die durch dieses Projekt oder seine Abhängigkeiten verursacht werden.

> [!IMPORTANT]
> Es wird empfohlen, den Wechselrichter aktuell zu halten – diese Integration wird nur auf aktuellen Firmwareversionen getestet. Es wird empfohlen, die GEN24-Firmware auf 1.40.0 oder höher zu aktualisieren, da in früheren Versionen Probleme mit der Solar API zu mehrfachen Ausfällen geführt haben.

# Installation

## HACS-Installation

- HACS öffnen
- Auf die 3 Punkte in der oberen rechten Ecke klicken
- „Benutzerdefinierte Repositories" auswählen
- Die [URL](https://github.com/MaxRei-dev/fronius_modbus) zum Repository hinzufügen
- Als Typ „Integration" auswählen
- Auf „HINZUFÜGEN" klicken

## Manuelle Installation

Den Inhalt des Ordners `custom_components` in den Ordner `config/custom_components` von Home Assistant kopieren.
Nach einem Neustart von Home Assistant kann die Integration über die Integrations-UI konfiguriert werden.

## Wechselrichter-Einrichtung

### Einrichtung mit Web API

Wenn beim Einrichten der Integration das Technikerpasswort der Wechselrichter-Web-API angegeben wird, kann die Integration:

- Modbus TCP während der Einrichtung und bei relevanten Konfigurationsänderungen automatisch aktivieren
- das automatisch aktivierte Modbus TCP optional auf die IP-Adresse des Home Assistant Hosts beschränken
- konfigurierte Smartmeter-Adressen aus `/api/components/PowerMeter/readable` auslesen
- authentifizierte Batteriesteuerungen aus `/api/config/batteries` bereitstellen
- Modbus-Servicediagnostik aus `/api/config/modbus` bereitstellen

![solar_login](images/solar_login.jpg?raw=true "storage")

Der Web-API-Benutzername ist fest auf `technician` eingestellt – das lokale Technikerkonto des Wechselrichters. Das Technikerpasswort wird bei der Inbetriebnahme vom Installateur festgelegt und unterscheidet sich vom Solar.web-Cloud-Login (z. B. https://www.solarweb.com/) sowie vom Passwort des `customer`-Kontos.
Die Integration speichert ein abgeleitetes Digest-Token im Home Assistant-Speicher und bewahrt das Passwort nicht im Konfigurationseintrag auf.
Bei Einrichtung, Neukonfiguration, Optionen oder Reparaturen wird das Passwort nur abgefragt, wenn für den ausgewählten Host kein gespeichertes Token vorhanden ist oder das bestehende Token erneuert werden muss.

### Migration älterer Einträge

Einträge, die mit älteren Nur-Modbus-Versionen erstellt wurden, werden mit sicheren Standardwerten migriert und funktionieren vorübergehend weiter.
Wenn ein Eintrag kein gültiges gespeichertes Web-API-Token für den konfigurierten Host hat, erstellt Home Assistant einen Reparatureintrag, über den die Host-Einstellungen geprüft und das Technikerpasswort für ein neues Token eingegeben werden kann.

## Laden aus dem Netz

Geplantes Laden/Entladen in der Web-UI deaktivieren, um unerwartetes Verhalten zu vermeiden.

> [!IMPORTANT]
> Bei der Verwendung mehrerer Integrationen, die das pymodbus-Paket nutzen, kann es zu Versionskonflikten kommen, da sie sich ein Paket in HA teilen. Dies kann behoben werden, indem ALLE Integrationen, die pymodbus verwenden, sowie die Modbus-configuration.yaml (für die eingebaute HA-Integration) entfernt, HA neu gestartet und die Integrationen sowie die Modbus-Konfiguration anschließend neu installiert werden.

# Verwendung

### Batteriespeicher

Wenn Web-API-Zugangsdaten konfiguriert sind, stellt die Integration sowohl Modbus-Batteriesteuerungen als auch authentifizierte Batterie-API-Steuerungen gemeinsam bereit.
Die einzige integrierte protokollübergreifende Synchronisierung ist der minimale SoC:

- Solange `Batterie-API-Modus` auf `Manuell` steht, schreibt das Setzen von `Minimaler SoC` auch den API-SoC-Minimalwert und erzwingt den API-SOC-Modus `manual`
- `Batterie-API-Modus` wird aus `HYB_EM_MODE` und `BAT_M0_SOC_MODE` abgeleitet
- Der Modbus-Modus `Aus dem Netz laden` aktiviert auch die Web-API-Schalter `Aus dem Netz laden` und `Aus AC laden`, wenn die Web API konfiguriert ist
- Das Einschalten des Web-API-Schalters `Aus dem Netz laden` aktiviert auch `Aus AC laden`
- `Ziel-Einspeisung` wird vom Wechselrichter ignoriert, wenn das Batterieladen nicht verfügbar ist
- Wenn die beiden API-Modussignale nicht übereinstimmen, zeigt `Batterie-API-Modus` keinen Wert, `Ziel-Einspeisung` und `Maximaler SoC` sind deaktiviert, und die API-Ladequellen-Schalter bleiben nutzbar

### Steuerungen

| Entität              | Beschreibung                                                                                                                                                                   |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Entladebegrenzung    | Maximale Entladeleistung des Speichers in Watt.                                                                                                                                |
| Netz-Ladeleistung    | Ladeleistung in Watt beim Laden des Speichers aus dem Netz. Hinweis: Das Laden aus dem Netz scheint hardwareseitig auf effektiv 50 % begrenzt zu sein.                         |
| Netz-Entladeleistung | Entladeleistung in Watt beim Entladen des Speichers ins Netz.                                                                                                                  |
| Minimaler SoC        | Gemeinsame minimale SoC-Steuerung. Auf der Web-API-Seite entspricht dies `BAT_M0_SOC_MIN`. Nur ganze Zahlen. Im manuellen API-Modus darf er nicht größer als `Maximaler SoC` sein. |
| PV-Ladebegrenzung    | Maximale PV-Ladeleistung des Speichers in Watt.                                                                                                                                |

### Batterie-API-Steuerungen

| Entität          | Beschreibung                                                                                                                                                                                                                                                                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Batterie-API-Modus | Fronius Web API Batteriemodus: `Automatisch` oder `Manuell`.                                                                                                                                                                                                                                                                                                                  |
| Aus AC laden     | Web-API-Schalter für `HYB_BM_CHARGEFROMAC`. Wird auch automatisch aktiviert, wenn der Modbus-Modus `Aus dem Netz laden` gewählt wird. Ausschalten deaktiviert beide Ladequellen-Flags.                                                                                                                                                                                          |
| Aus dem Netz laden | Web-API-Schalter für `HYB_EVU_CHARGEFROMGRID`. Einschalten aktiviert auch `Aus AC laden`. Ausschalten deaktiviert nur das Netz-Flag. Wird auch automatisch aktiviert, wenn der Modbus-Modus `Aus dem Netz laden` gewählt wird.                                                                                                                                                |
| Ziel-Einspeisung | Manueller Fronius-Zielwert für die Einspeisung in Watt. Positive Werte zielen auf Einspeise-Watt ab. Negative Werte zielen auf Netzbezug in Watt ab – der Wechselrichter hält diesen Netzbezug auch dann aufrecht, wenn PV-Leistung verfügbar ist. Diese Einstellung wird vom Wechselrichter ignoriert, wenn das Batterieladen nicht verfügbar ist. Nur aktiv wenn `HYB_EM_MODE=1` und `BAT_M0_SOC_MODE="manual"`. |
| Maximaler SoC    | `BAT_M0_SOC_MAX` aus der Web API. Nur verfügbar wenn `HYB_EM_MODE=1` und `BAT_M0_SOC_MODE="manual"`, und darf nicht unter `Minimaler SoC` gesetzt werden.                                                                                                                                                                                                                      |

### Wechselrichter-API-Steuerungen

| Entität               | Beschreibung                                                                                                                                                            |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Watt-Peak-Referenz    | Setzt den Watt-Peak-Referenzwert für die Leistungsgrenzen-Visualisierung (`wattPeakReferenceValue` über `/api/config/limit_settings/powerLimits`).                      |
| Solar-API             | Aktiviert oder deaktiviert die Fronius Solar API v1 am Wechselrichter. Deaktivierung wird bei Firmware unter 1.40.7-1 empfohlen, um ein bekanntes Speicherleck zu vermeiden. |
| Modbus-Steuerung zurücksetzen | Sendet einen Modbus-Reset-Befehl an den Wechselrichter (`/api/commands/ModbusReset`). Diagnose-Entität.                                                         |

### Speicher-Steuermodi

| Modus                             | Beschreibung                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Automatisch                       | Der Speicher kann bis zum konfigurierten `Minimalen SoC` geladen und entladen werden.                                                                                                                                                                                                                                                                                                                                                                               |
| PV-Ladebegrenzung                 | Der Speicher kann mit PV-Leistung mit begrenzter Rate geladen werden. Die Begrenzung wird nach der Modusänderung auf maximale Leistung gesetzt.                                                                                                                                                                                                                                                                                                                     |
| Entladebegrenzung                 | Der Speicher kann mit PV-Leistung geladen und mit begrenzter Rate entladen werden. Die Begrenzung wird nach der Modusänderung auf maximale Leistung gesetzt.                                                                                                                                                                                                                                                                                                        |
| PV-Lade- und Entladebegrenzung    | Ermöglicht das separate Einstellen von PV-Lade- und Entladebegrenzung. Beide Werte werden nach der Modusänderung auf maximale Leistung gesetzt.                                                                                                                                                                                                                                                                                                                     |
| Aus dem Netz laden                | Der Speicher wird aus dem Netz geladen, mit der Laderate aus „Netz-Ladeleistung". Die Leistung wird nach der Modusänderung auf 0 gesetzt. Die Netz-Ladeleistung muss in Watt als Vielfaches von 10 angegeben werden – andernfalls kommt es zu unerwartetem Verhalten (z. B. Laden mit 500 W). Wenn dieser Modus über die Integration gewählt wird und die Web API konfiguriert ist, werden `Aus dem Netz laden` und `Aus AC laden` ebenfalls automatisch aktiviert. |
| Ins Netz entladen                 | Der Speicher wird ins Netz entladen, mit der Entladerate aus „Netz-Entladeleistung". Die Leistung wird nach der Modusänderung auf 0 gesetzt.                                                                                                                                                                                                                                                                                                                        |
| Entladen sperren                  | Der Speicher kann nur mit PV-Leistung geladen werden. Die Ladebegrenzung wird auf maximale Leistung gesetzt.                                                                                                                                                                                                                                                                                                                                                        |
| Laden sperren                     | Der Speicher kann nur entladen werden und wird nicht mit PV-Leistung geladen. Die Entladebegrenzung wird auf maximale Leistung gesetzt.                                                                                                                                                                                                                                                                                                                              |

Hinweis: Zuerst den Modus wechseln, dann die für diesen Modus relevanten Steuerungen setzen.

### Steuerungen nach Modus

| Modus                          | Ladebegrenzung  | Entladebegrenzung | Netz-Ladeleistung | Netz-Entladeleistung | Min. SoC   |
| ------------------------------ | --------------- | ----------------- | ----------------- | -------------------- | ---------- |
| Automatisch                    | Ignoriert (100%)| Ignoriert (100%)  | Ignoriert (0%)    | Ignoriert (0%)       | Verwendet  |
| PV-Ladebegrenzung              | Verwendet       | Ignoriert (100%)  | Ignoriert (0%)    | Ignoriert (0%)       | Verwendet  |
| Entladebegrenzung              | Ignoriert (100%)| Verwendet         | Ignoriert (0%)    | Ignoriert (0%)       | Verwendet  |
| PV-Lade- und Entladebegrenzung | Verwendet       | Verwendet         | Ignoriert (0%)    | Ignoriert (0%)       | Verwendet  |
| Aus dem Netz laden             | Ignoriert       | Ignoriert         | Verwendet         | Ignoriert (0%)       | Verwendet  |
| Ins Netz entladen              | Ignoriert       | Ignoriert         | Ignoriert (0%)    | Verwendet            | Verwendet  |
| Entladen sperren               | Verwendet       | Ignoriert (0%)    | Ignoriert (0%)    | Ignoriert (0%)       | Verwendet  |
| Laden sperren                  | Ignoriert (0%)  | Verwendet         | Ignoriert (0%)    | Ignoriert (0%)       | Verwendet  |

### Zuordnung Fronius Web-UI

| Web-UI-Name            | Integrations-Steuerung | Integrations-Modus     |
| ---------------------- | ---------------------- | ---------------------- |
| Max. Ladeleistung      | PV-Ladebegrenzung      | PV-Ladebegrenzung      |
| Min. Ladeleistung      | Netz-Ladeleistung      | Aus dem Netz laden     |
| Max. Entladeleistung   | Entladebegrenzung      | Entladebegrenzung      |
| Min. Entladeleistung   | Netz-Entladeleistung   | Ins Netz entladen      |

### Batteriespeicher-Sensoren

| Entität         | Beschreibung                                                                                                                           |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Ladestatus      | Holding / Laden / Entladen                                                                                                             |
| Minimaler SoC   | Gemeinsamer minimaler SoC-Wert. Wenn die Web API konfiguriert und der API-Modus manuell ist, folgt dieser Wert dem Web-API-SoC-Minimum. |
| Ladezustand     | Aktueller Batterieladestand in Prozent.                                                                                                |

### Wechselrichter-Sensoren

| Entität                    | Beschreibung                                                                                          |
| -------------------------- | ----------------------------------------------------------------------------------------------------- |
| Hauslast                   | Aktueller Gesamtstromverbrauch, berechnet aus Smartmeter-AC-Leistung und Wechselrichter-AC-Leistung.  |
| AC-Strom                   | Gesamt-AC-Strom des Wechselrichters.                                                                  |
| AC-Strom L1 / L2 / L3      | Phasenweiser AC-Strom des Wechselrichters.                                                            |

### Smartmeter-Sensoren

| Entität                       | Beschreibung                                                                                                                  |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| AC-Strom / L1 / L2 / L3      | Gesamt- und phasenweiser AC-Strom des Smartmeters.                                                                            |
| Leistung                      | Netto-Netzleistung gemessen vom Smartmeter.                                                                                   |
| Leistung L1 / L2 / L3        | Phasenweise Wirkleistung des Smartmeters aus SunSpec `WphA`, `WphB` und `WphC`. Das Vorzeichen entspricht der Messrichtung.   |

### Wechselrichter-Diagnose

| Entität                                          | Beschreibung                                                                                                                                                                                               |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Netzstatus                                       | Netzstatus basierend auf Smartmeter- und Wechselrichterfrequenz. Bei ca. 53 Hz läuft der Wechselrichter im Inselbetrieb, normal sind 50 Hz. Im Schlafmodus wird die Smartmeter-Frequenz zur Statusermittlung herangezogen. |
| Status / Herstellerstatus                        | Standard-SunSpec-Wechselrichterstatus sowie der Fronius-herstellerspezifische Statuscode.                                                                                                                  |
| Referenzspannung / Referenzspannungs-Offset      | SunSpec-Modell-121-PCC-Spannungsreferenzwerte des Wechselrichters.                                                                                                                                         |
| Web-API-Modbus-Modus / -Steuerung / -SunSpec-Modus | Authentifizierte Modbus-Servicediagnostik aus `/api/config/modbus`.                                                                                                                                      |
| Web-API-Modbus-IP-Beschränkung / -IP             | Zeigt an, ob der Wechselrichter den Modbus-Zugriff per IP einschränkt.                                                                                                                                     |

### Wechselrichter-Steuerungen

| Entität                    | Beschreibung                                                                                                                            |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| AC-Begrenzung aktivieren   | Ermöglicht die Begrenzung der AC-Ausgangsleistung des Wechselrichters. Zuerst aktivieren, dann den Grenzwert setzen.                    |
| AC-Leistungsbegrenzung     | Setzt die AC-Leistungsbegrenzung in Watt. Intern wird dies auf SunSpec `WMaxLimPct` (% von `WMax`) mit dem Wechselrichter-Skalierungsfaktor abgebildet. |
| Leistungsfaktor-Steuerung  | Aktiviert oder deaktiviert die Modbus-Leistungsfaktorsteuerung (`OutPFSet_Ena`).                                                        |
| Leistungsfaktor            | Fester Leistungsfaktor (`OutPFSet`). Bereich: `-1,0` bis `1,0`. Negative Werte = übererregt, positive Werte = untererregt.             |

# Beispielgeräte

Diese Bilder sind Beispiele. Die Entitäten werden in zwei Kategorien gruppiert: eine für den Wechselrichter und eine für die Batterie.

Batteriespeicher
![battery storage](images/example_batterystorage.jpg?raw=true "storage")

Smartmeter
![smart meter](images/example_meter.jpg?raw=true "meter")

Wechselrichter
![inverter](images/example_inverter.jpg?raw=true "inverter")

# Referenzen

- https://www.fronius.com/~/downloads/Solar%20Energy/Operating%20Instructions/42,0410,2649.pdf
- https://github.com/binsentsu/home-assistant-solaredge-modbus/
- https://github.com/bigramonk/byd_charging
