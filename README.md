# EMS-Planung

# Neuentwicklung Energiemanagement

*Stand: 29. Oktober 2025*

---

## 1) Zusammenfassung

Ziel ist eine modulare, wartbare Plattform zur Erfassung, Verarbeitung und Visualisierung von Energiedaten (Edge → Backend → Visualisierung), funktional angelehnt an OpenEMS Edge/Backend – jedoch ohne OSGi. Wir setzen auf .NET 8 LTS (Linux-first), klare Dependency Injection, YAML-Konfiguration, und eine transparente Treiber-API, damit neue Geräteadapter innerhalb von Stunden implementierbar sind. Visualisierung zu Beginn über Grafana; Automation/Regeln folgen nach Alpha. Lizenzkandidaten: Apache 2.0, AGPL3, MIT. Tooling: vollständig telemetriefrei betreibbar.

**Nicht-Ziele in Phase 1:** kein UI-Monolith, kein OSGi/Gradle, kein proprietäres Buildsystem. Fokus: FoxESS H3 Read‑Only, stabiler Datenpfad, Dashboards.

---

## 2) Ziele & Nicht-Ziele

**Ziele (Alpha):**

* FoxESS H3 Read‑Only, stabiler 24h-Dauerbetrieb.
* Datenfluss Edge → Backend → TimescaleDB → Grafana-Dashboards.
* YAML‑Konfiguration mit Validierung und klaren Fehlermeldungen.
* Telemetriefreie Toolchain-Doku.

**Nicht‑Ziele (Alpha):**

* Keine umfassende Automations-/Regel-Engine.
* Kein eigener GUI‑Monolith (Grafana genügt vorerst).
* Kein Remote‑Device‑Management (nur Basis-Configs).

---

## 3) Architekturüberblick

* **Edge Agent (C#/.NET 8 LTS):** schlanker Prozess; lädt *DeviceAdapter* (Treiber) dynamisch (DLL Files im selben Ordner), sammelt Messwerte/Ereignisse, publiziert an Backend.
* **Adapter SDK:** minimale, stabile Schnittstellen (Init, Capabilities, Read/Query, optional Commands), klare Fehler- und Rate‑Limit‑Semantik; YAML‑Konfiguration pro Adapter.
* **Transport:** MQTT+TLS/SSL or Eigenimplementation via Websockets.
* **Backend/Ingestion:** Normalisierung, Persistenz in TimescaleDB; REST‑API.
* **Visualisierung:** Grafana mit vorkonfigurierten Dashboards.
* **Automation (später):** YAML/DSL‑Regeln, Aktorik über Adapter‑Commands, Simulationsmodus.
* **Ops:** NixOS Module u./o. Docker‑Compose;
* **Lizenzkandidaten:** Apache 2.0, AGPL3, MIT

---

## 4) Komponenten & Verantwortlichkeiten

| Komponente        | Kernaufgabe                      | Tech/Protokoll        | Akzeptanzkriterien (Alpha)                                |
| ----------------- | -------------------------------- | --------------------- | --------------------------------------------------------- |
| Edge Agent        | Adapter laden, Sampling, Publish | .NET 8; YAML; systemd | Start/Stop stabil; Offline‑Queue; 24h Dauertest ohne Leak |
| Adapter SDK       | Interface & Helpers              | NuGet‑Package         | H3‑Adapter implementierbar < 1 Tag                        |
| FoxESS H3 Adapter | Register/Frames lesen            | Modbus/TCP            | ≥95 % relevante Messpunkte; Retry/Backoff                 |
| Transport         | Edge → Backend                   | MQTT/TLS              | QoS1; Backpressure                                        |
| Ingestion         | Normalisieren & Persistieren     | C# Service            | 1 M Messpunkte/Tag fehlerfrei                             |
| TS‑DB             | Zeitreihen                       | TimescaleDB           | Retention/Compression; 30d‑Query < 2 s                    |
| API               | Datenzugriff                     | REST (JSON)           | Filter/Range stabil                                       |
| Grafana           | Visualisierung                   | Dashboards            | Standard‑Dashboards für H3 & Site                         |

---

## 5) Konfiguration

* **Global:** `edge.yml` (Knoten‑ID, Broker‑URL, TLS, Sampling‑Intervalle, Puffergrößen).
* **Adapter:** `devices/h3.yml` (IP/Host, Register‑Mapping, Sampling‑Profile).
* **Validierung:** Schema‑Validierung beim Start; präzise Fehlermeldungen (YAML Pointer). Secrets via Datei/Env‑Var.

**Beispiel edge.yml (Draft):**

```yaml
node:
  id: edge-001
  site: demo
mqtt:
  host: broker.ems.tele-centrum.net.net
  port: 8883
  clientId: edge-001
  username: ${MQTT_USER}
  passwordFile: /etc/edge/mqtt.pass
sampling:
  interval: 5s
adapters:
  - type: foxess-h3
    name: Foxess 1
    deviceid: fox_ess_01
    config: $include devices/h3.yml
```

## 6) Sicherheit & Datenschutz

* TLS‑Transport; optional Mutual‑TLS.
* Least‑Privilege‑Principle (getrennte Service‑Accounts, Rollen).
* Audit‑Logs für Config‑Changes & Adapter‑Fehler.
* Keine erzwungene externe Telemetrie.

---

## 7) Entwicklung & Betrieb (Dev/Ops)

* **Repo‑Layout:** `edge/`, `adapters/`, `backend/`, `infrastructure/`, `docs/`.
* **CI:** Build, Tests, Lint; Container‑Images; SBOM (Syft) & Signierung (Cosign).
* **Deploy:** Docker‑Compose + Nix Module; Health‑Checks; Prometheus/OTel‑Metriken.
* **Doku:** `docs/` + ADRs; Diagrams‑as‑Code (Mermaid); ReadTheDocs‑artiger Output.

---

## 8) Roadmap & Timeline (Ein‑Entwickler‑Annahme)

| Phase                                  | Kalendertage | Kern‑Deliverables                                                                      |
| -------------------------------------- | ------------ | -------------------------------------------------------------------------------------- |
| Phase 0 – Projektsetup                 | 7            | Repo, Lizenz, CI‑Skeleton; Edge‑Skeleton (DI); YAML‑Schema; Systemd/Docker Boilerplate |
| Phase 1 – Adapter‑SDK & FoxESS H3 (RO) | 14           | SDK‑Interfaces & Helpers; H3‑Adapter; Polling‑Loop; 24h Stabilität                     |
| Phase 2 – Transport & Backend          | 10           | MQTT/TLS; Ingestion‑Service; Timescale‑Schema; Basis‑API                               |
| Phase 3 – Dashboards & Alpha           | 7            | Grafana‑Dashboards; Install‑Guides; Feinschliff/Docs; Alpha‑Tag                        |
| Phase 4 – Controling                   | 14           | Kontrolieren eines bzw mehrerer Anlagen                                                |
---

## 9) Meilensteine

* **M1:** Edge‑Skeleton + YAML‑Validierung; CI/CD‑Builds; systemd‑Start/Stop → *Go bei stabil*.
* **M2:** H3‑Adapter: ≥95 % Kern‑Messwerte; 24h Dauerlauf → *Go*.
* **M3:** Datenfluss Edge→Backend→TS‑DB → *Go*.
* **Alpha:** Dashboards + Install‑Docs; SemVer eingeführt.
* **M1:** Einstellungen in Anlagen Schreiben (Edge Only)
* **M2:** Einstellungen von Backend → Edge
* **Alpha 2:** Im Kern funktionierendes Energiemanagement

---

## 10) Risiken & Gegenmaßnahmen

| Risiko                | Auswirkung                  | Mitigation                                                                 |
| --------------------- | --------------------------- | -------------------------------------------------------------------------- |
| Vendor‑Sorgen (C#)    | MS‑Bindungs‑Bedenken        | .NET ist OSS; Telemetrie per Env/Build aus; VSCodium dokumentiert          |
| Treiber‑Spezifika H3  | Unklare Register/Änderungen | Abstraktionsschicht + robustes Mapping; Feature‑Flags; saubere Fehlercodes |
| Leistung/Skalierung   | Spikes, Datenverlust        | QoS1, Backpressure, Edge‑Pufferung, Lasttests in CI                        |
| Broker‑Ausfall (MQTT) | Datenstau/Verlust           | Persistente Sessions, Retain/Queue; NATS‑Option evaluieren (Phase 4)       |

---

## 11) Telemetriefreie Toolchain (Linux)

* VSCodium + offene C#‑Erweiterung (ohne Telemetrie).
* .NET 8 LTS SDK/Runtimes aus Distro‑Repos oder offiziellen OSS‑Repos; Telemetrie via `DOTNET_CLI_TELEMETRY_OPTOUT=1` + SLN File Deaktiviern
* Optional: Rider/VSC/Codium nur als persönliche Wahl; Builds bleiben ohne proprietäre IDE möglich.

---

## 12) Nuget Packete (Ideen)

* [https://www.nuget.org/packages/MQTTnet/](https://www.nuget.org/packages/MQTTnet/)
* [https://github.com/NModbus/NModbus](https://github.com/NModbus/NModbus) (im Notfall bekomme ich Modbus auch selbst hin, ist sehr simples Protokoll)
* [https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection/](https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection/)  (Das ist unter MIT Lizenz ;-) )
* [https://www.nuget.org/packages/YamlDotNet](https://www.nuget.org/packages/YamlDotNet)
