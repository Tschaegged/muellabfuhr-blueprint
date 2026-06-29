# 🗑️ Müllabfuhr-Erinnerung mit Bestätigung

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTschaegged%2Fmuellabfuhr-blueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fmuellabfuhr_erinnerung.yaml)

Ein Home Assistant Blueprint, der abends benachrichtigt wenn morgen die Mülltonne
rausgestellt werden muss – und die Erinnerung so lange wiederholt, bis jemand aktiv
bestätigt hat.

## Funktionsweise

```
18:00 Uhr ──► Benachrichtigung an alle Empfänger
                  ┌─ Companion App: Button „✓ Tonne rausgestellt"
                  └─ Telegram / andere: Textnachricht

    30 min nicht bestätigt ──► Erneute Benachrichtigung (bis max. 4×)

    Button getippt ──► ✅ Bestätigung an alle + Abbruch der Schleife

07:00 Uhr (Abholtag) ──► Falls noch nicht bestätigt: Morgen-Erinnerung
```

## Voraussetzungen

### 1. Bestätigungs-Helfer anlegen

Unter **Einstellungen → Geräte & Dienste → Helfer → Helfer hinzufügen → Schalter**:

- Name: `Tonne rausgestellt`
- Entity-ID: `input_boolean.tonne_rausgestellt`

### 2. Sensoren für die Abholbedingung

Das Blueprint benötigt Bedingungen, die angeben ob morgen bzw. heute Müll abgeholt wird.
Empfohlen: Template-Sensoren auf Basis der
[waste_collection_schedule](https://github.com/mampfes/hacs_waste_collection_schedule)
Integration.

**Beispiel-Sensor für „Abholung morgen?"** (`sensors/mullabholung.yaml`):

```yaml
- name: Müllabholung Morgen
  unique_id: mullabholung_morgen
  state: >
    {% set morgen = (now().date() + timedelta(days=1)).isoformat() %}
    {% set ns = namespace(types=[]) %}
    {% for attr, val in states.sensor.nachste_abholung_2.attributes.items() %}
      {% if attr == morgen %}{% set ns.types = ns.types + [val] %}{% endif %}
    {% endfor %}
    {% if ns.types %}
      Du musst morgen {{ ns.types | join(', ') }} rausstellen.
    {% else %}
      Du musst morgen keine Tonne rausstellen.
    {% endif %}
```

**Standardbedingung im Blueprint** (passt zu obigem Sensor):
```
{{ 'keine' not in states('sensor.mullabholung_morgen') | lower }}
```

## Konfiguration

| Parameter | Standard | Beschreibung |
|---|---|---|
| Uhrzeit (Abend) | 18:00 | Erste Erinnerung am Vorabend |
| Bedingung Abend | `sensor.mullabholung_morgen` | Template: `true` wenn morgen Abholung |
| Max. Wiederholungen | 4 | Abendliche Benachrichtigungen bis Aufgabe |
| Wiederholungsintervall | 30 min | Pause zwischen unbestätigten Erinnerungen |
| Empfänger | – | `notify.*`-Entities (mehrere wählbar) |
| Nachrichtentext | Sensor-State | Jinja2-Template möglich |
| Bestätigungs-Helfer | – | `input_boolean.*`-Entity |
| Morgen-Erinnerung | aktiv | Erinnerung am Abholtag bei fehlender Bestätigung |
| Uhrzeit (Morgen) | 07:00 | Morgen-Erinnerungszeit |

## Funktionsdetails

- **Eindeutige Action-ID**: Jede Blueprint-Instanz bekommt eine eigene Notification-Action,
  damit bei mehreren Instanzen keine Kreuzbestätigungen auftreten.
- **Companion App**: Persistente Benachrichtigung mit Bestätigungs-Button.
  Nach Bestätigung wird die Notification durch eine Erfolgsmeldung ersetzt.
- **Telegram / andere Dienste**: Erhalten die Textnachricht ohne Button. Bestätigung
  erfolgt über ein Companion-App-Gerät.
- **Morgen-Erinnerung**: Läuft in einer separaten Phase (mode: queued) nach der
  Abend-Schleife, falls diese ohne Bestätigung ausgelaufen ist.
- **Bestätigungs-Helfer**: Zeigt in Dashboards den aktuellen Status an.
  Wird zu Beginn jeder neuen Abend-Erinnerung automatisch zurückgesetzt.

## Bestehende Automation ersetzen

Falls bereits eine einfache Müll-Benachrichtigung vorhanden ist, diese deaktivieren
und durch eine Instanz dieses Blueprints ersetzen.

## Lizenz

MIT – siehe [LICENSE](LICENSE)
