# MS Teams SBC Patch für Asterisk 20

## Übersicht

Dieser Patch fügt Unterstützung für Microsoft Teams SBC (Session Border Controller) Funktionalität zu Asterisk 20 hinzu. Der Patch ermöglicht es, einen separaten Hostnamen für MS Teams Signaling zu konfigurieren, der in SIP Contact- und Via-Headern verwendet wird.

## Problemstellung

Microsoft Teams als SBC erfordert spezifische SIP-Header-Werte für eine korrekte Kommunikation:

- **Contact-Header**: Muss den korrekten Hostnamen enthalten, den MS Teams erwartet
- **Via-Header**: Muss den korrekten Hostnamen für Routing enthalten

Asterisk verwendet standardmäßig die `external_signaling_address` (als IP-Adresse oder Hostname) für beide Header. MS Teams benötigt jedoch manchmal einen separaten Hostnamen, der sich von der normalen externen Adresse unterscheidet.

## Lösung

Der Patch fügt eine neue Konfigurationsoption `ms_signaling_address` für SIP-Transports hinzu. Wenn diese Option gesetzt ist, wird dieser Hostname in Contact-URI- und Via-Headern verwendet, anstatt der normalen `external_signaling_address`.

## Anwendung des Patches

### Voraussetzungen

- Asterisk 20.x Quellcode
- Standard Build-Tools (gcc, make, etc.)

### Patch anwenden

1. Wechseln Sie in das Asterisk-Quellverzeichnis:
   ```bash
   cd /path/to/asterisk-20.x.x
   ```

2. Wenden Sie den Patch an:
   ```bash
   patch -p1 < /path/to/asterisk-20-ms-teams-sbc.patch
   ```

3. Kompilieren Sie Asterisk neu:
   ```bash
   ./configure
   make
   make install
   ```

## Konfiguration

### pjsip.conf

**WICHTIG:** Der Parameter `ms_signaling_address` kann **NUR** im `[transport]` Abschnitt verwendet werden, **NICHT** im `[endpoint]` Abschnitt!

Fügen Sie die neue Option `ms_signaling_address` zu Ihrem Transport hinzu:

```ini
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060
external_signaling_address=your.public.ip.address
ms_signaling_address=sbc.yourdomain.com
external_signaling_port=5060
```

**Beispiel-Konfiguration für MS Teams:**

```ini
; Transport mit MS Teams Signaling
[transport-udp-teams]
type=transport
protocol=udp
bind=0.0.0.0:5060
external_signaling_address=203.0.113.1
ms_signaling_address=sbc.yourdomain.com
external_signaling_port=5060

; Endpoint für MS Teams (verwendet den Transport)
[teams-endpoint]
type=endpoint
transport=transport-udp-teams
context=teams-incoming
allow=!all,ulaw,alaw
aors=teams-aor
direct_media=no
```

**Häufiger Fehler:**
```ini
; FALSCH - ms_signaling_address gehört NICHT in einen endpoint!
[teams-endpoint]
type=endpoint
ms_signaling_address=sbc.yourdomain.com  ; ❌ FEHLER!
```

```ini
; RICHTIG - ms_signaling_address gehört in den transport!
[transport-udp-teams]
type=transport
ms_signaling_address=sbc.yourdomain.com  ; ✅ KORREKT!
```

### Erklärung der Optionen

- **`external_signaling_address`**: Die normale externe IP-Adresse oder der Hostname für SIP-Signaling (wird verwendet, wenn `ms_signaling_address` nicht gesetzt ist)
- **`ms_signaling_address`**: Spezifischer Hostname für MS Teams Signaling (optional, hat Vorrang wenn gesetzt)
- **`external_signaling_port`**: Externe Port-Nummer für SIP-Signaling

### Verhalten

- **Parameter-Name:** `ms_signaling_address`
- **Verfügbar in:** Nur `[transport]` Abschnitten (NICHT in `[endpoint]`!)
- Wenn `ms_signaling_address` im Transport gesetzt ist, wird dieser Hostname in Contact- und Via-Headern verwendet
- Wenn `ms_signaling_address` nicht gesetzt ist, wird `external_signaling_address` verwendet (Standard-Verhalten)
- Der Port wird weiterhin von `external_signaling_port` bestimmt
- Der Endpoint muss den Transport mit `transport=transport-name` referenzieren

## Technische Details

### Geänderte Dateien

1. **`include/asterisk/res_pjsip.h`**
   - Fügt `ms_signaling_address` Feld zur `ast_sip_transport` Struktur hinzu

2. **`res/res_pjsip/config_transport.c`**
   - Fügt Handler-Funktionen für `ms_signaling_address` hinzu
   - Registriert das neue Feld in der Sorcery-Konfiguration

3. **`res/res_pjsip_nat.c`**
   - Modifiziert die NAT-Rewrite-Logik, um `ms_signaling_address` zu verwenden
   - Betrifft Contact-URI und Via-Header

4. **`res/res_pjsip/pjsip_config.xml`**
   - Fügt Dokumentation für die neue Option hinzu

5. **`res/res_pjsip/pjsip_manager.xml`**
   - Fügt Dokumentation für den AMI-Parameter hinzu

### Unterschiede zu Asterisk 18.3

Dieser Patch wurde von einem ähnlichen Patch für Asterisk 18.3 adaptiert. Die Hauptunterschiede:

- In Asterisk 20 wird `ast_sip_get_contact_sip_uri()` statt `nat_get_contact_sip_uri()` verwendet
- Die Bedingung für die Contact-URI-Überschreibung wurde erweitert (behandelt auch 302 Redirects)
- Die Dokumentation wurde in separate XML-Dateien verschoben

## Verbesserungen in Asterisk 20 für MS Teams SBC

Asterisk 20 bietet mehrere Verbesserungen gegenüber Asterisk 18.3 im Kontext dieses MS Teams SBC Patches:

### 1. **Bessere Behandlung von 302 Redirects**

**Asterisk 18.3:**
- Überschreibt Contact-Header auch bei 302 (MOVED_TEMPORARILY) Redirects
- Dies kann zu Problemen führen, wenn MS Teams einen Redirect sendet

**Asterisk 20:**
- Erkennt explizit 302 Redirects und überschreibt den Contact-Header in diesem Fall **nicht**
- Verhindert Probleme bei Call-Transfers und Redirects von MS Teams
- Code-Beispiel:
```c
// Asterisk 20: Behandelt 302 Redirects korrekt
if ((!cseq || tdata->msg->type == PJSIP_REQUEST_MSG ||
    pjsip_method_cmp(&cseq->method, &pjsip_register_method)) &&
    (tdata->msg->type != PJSIP_RESPONSE_MSG ||
    pjsip_method_cmp(&cseq->method, &pjsip_invite_method) ||
    tdata->msg->line.status.code != PJSIP_SC_MOVED_TEMPORARILY )) {
    // Contact-URI wird nur überschrieben, wenn es KEIN 302 Redirect ist
}
```

### 2. **Zentralisierte API-Funktion**

**Asterisk 18.3:**
- Verwendet lokale statische Funktion `nat_get_contact_sip_uri()`
- Code-Duplikation in verschiedenen Modulen möglich

**Asterisk 20:**
- Verwendet öffentliche API-Funktion `ast_sip_get_contact_sip_uri()`
- Zentralisierte, getestete Implementierung
- Wiederverwendbar in anderen Modulen
- Bessere Wartbarkeit und Konsistenz

### 3. **Verbesserte Code-Struktur**

**Asterisk 20:**
- Klarere Trennung von Verantwortlichkeiten
- Bessere Dokumentation der Funktionen
- Öffentliche API-Funktionen sind dokumentiert und stabil

### 4. **Bessere Kompatibilität mit MS Teams**

**Asterisk 20:**
- Korrekte Behandlung von Call-Transfers (302 Redirects)
- Weniger Probleme bei komplexen Szenarien mit MS Teams
- Stabileres Verhalten bei NAT-Traversierung

### Zusammenfassung der Vorteile

| Feature | Asterisk 18.3 | Asterisk 20 |
|---------|---------------|-------------|
| 302 Redirect Handling | ❌ Überschreibt Contact auch bei 302 | ✅ Überschreibt Contact NICHT bei 302 |
| API-Funktionen | Lokale statische Funktion | Öffentliche API-Funktion |
| Code-Wiederverwendung | Eingeschränkt | Gut wiederverwendbar |
| MS Teams Kompatibilität | Gut | Sehr gut |
| Wartbarkeit | Mittel | Hoch |

## Testing

Nach dem Anwenden des Patches sollten Sie:

1. Asterisk neu starten
2. Die Konfiguration mit `asterisk -rx "pjsip show transports"` überprüfen
3. Einen Test-Call von MS Teams durchführen
4. Die SIP-Header in den Logs überprüfen, um sicherzustellen, dass der korrekte Hostname verwendet wird

## Fehlerbehebung

### Patch lässt sich nicht anwenden

- Stellen Sie sicher, dass Sie die richtige Asterisk-Version verwenden
- Überprüfen Sie, ob die Dateien bereits modifiziert wurden
- Verwenden Sie `patch -p1 --dry-run` für eine Test-Anwendung

### Fehler: "Could not find option suitable for category 'endpoint' named 'ms_signaling_address'"

**Problem:** Sie versuchen, `ms_signaling_address` in einem `[endpoint]` Abschnitt zu verwenden.

**Lösung:** 
- `ms_signaling_address` gehört **NUR** in einen `[transport]` Abschnitt
- Setzen Sie `ms_signaling_address` im Transport-Abschnitt
- Referenzieren Sie den Transport im Endpoint mit `transport=transport-name`

**Beispiel:**
```ini
; Transport mit ms_signaling_address
[transport-udp-teams]
type=transport
protocol=udp
ms_signaling_address=sbc.yourdomain.com

; Endpoint referenziert den Transport
[teams-endpoint]
type=endpoint
transport=transport-udp-teams  ; ← Wichtig: Transport referenzieren!
```

### Hostname wird nicht verwendet

- Überprüfen Sie die Konfiguration in `pjsip.conf`
- Stellen Sie sicher, dass `ms_signaling_address` im **Transport** (nicht Endpoint) gesetzt ist
- Überprüfen Sie, ob der Endpoint den richtigen Transport referenziert
- Überprüfen Sie die Asterisk-Logs auf Fehler

### Kompilierungsfehler

- Stellen Sie sicher, dass alle Abhängigkeiten installiert sind
- Führen Sie `make clean` aus und versuchen Sie es erneut
- Überprüfen Sie die Compiler-Ausgabe auf spezifische Fehler

## Support

Bei Problemen oder Fragen:

1. Überprüfen Sie die Asterisk-Logs (`/var/log/asterisk/full`)
2. Aktivieren Sie Debug-Logging: `asterisk -rx "core set verbose 5"`
3. Überprüfen Sie die SIP-Nachrichten mit `asterisk -rx "pjsip set logger on"`

## Lizenz

Dieser Patch basiert auf dem Asterisk-Quellcode und unterliegt derselben Lizenz wie Asterisk (GPL v2).

## Version

- **Patch-Version**: 1.0
- **Asterisk-Version**: 20.x
- **Erstellt**: 2025

## Changelog

### Version 1.0
- Initiale Portierung des MS Teams SBC Patches von Asterisk 18.3 nach Asterisk 20
- Anpassung an die geänderten APIs in Asterisk 20
- Aktualisierung der Dokumentation

