Vor dem Test wurde eine neue Sicherung erstellt:

```bash
sudo cp -a /etc/ssh/sshd_config.d/00-lab-hardening.conf /root/00-lab-hardening.conf.avant-test-auditd
```

Hinzufügen eines Kommentars und Syntaxprüfung:

```bash
echo '# Test auditd : modification controlee' | sudo tee -a /etc/ssh/sshd_config.d/00-lab-hardening.conf
sudo /usr/sbin/sshd -t
```

Suche nach den Ereignissen:

```bash
sudo ausearch -k ssh_config -ts recent -i
```

### 17.2 Analyse des Audit-Ereignisses

Die Ausgabe enthielt ein Ereignis zum Laden einer Regel mit der Kennung **112** sowie das Testereignis mit der Kennung **140**, das am **22. Oktober 2026 um 10:11:25.299 UTC** aufgezeichnet wurde.

Relevante Auszüge aus Ereignis 140:

```text
type=PROCTITLE ... proctitle=tee -a /etc/ssh/sshd_config.d/00-lab-hardening.conf
type=PATH ... name=/etc/ssh/sshd_config.d/00-lab-hardening.conf
type=SYSCALL ... syscall=openat success=yes
... a2=O_WRONLY|O_CREAT|O_APPEND
... auid=vagrant uid=root euid=root
... comm=tee exe=/usr/bin/tee key=ssh_config
```

| Feld | Ergebnis | Interpretation |
|---|---|---|
| Ereigniskennung | `140` | Zusammenfassung der Einträge desselben Vorgangs |
| `name` | Pfad der SSH-Datei | Betroffene Datei |
| `exe` / `comm` | `/usr/bin/tee` / `tee` | Ausgeführtes Programm |
| `auid` | `vagrant` | Im Audit beibehaltene ursprüngliche Anmeldeidentität |
| `uid` / `euid` | `root` / `root` | Identitäten des mit `sudo` gestarteten Prozesses |
| Systemaufruf | `openat` | Öffnen der Datei |
| Flags | `O_WRONLY`, `O_CREAT`, `O_APPEND` | Öffnen zum Schreiben mit Anhängen |
| `success` | `yes` | Erfolg des beobachteten Systemaufrufs |
| `key` | `ssh_config` | Zuordnung zur SSH-Regel |

Das Ereignis zeigt das erfolgreiche Öffnen zum Schreiben mit Anhängen durch `tee`, aus einer `vagrant` zugeordneten Sitzung und mit root-Rechten. Dieser Protokolleintrag gibt den hinzugefügten Inhalt nicht wieder; die tatsächliche Inhaltsänderung wird separat mit AIDE überprüft.

### 17.3 Erkennung derselben Änderung mit AIDE

```bash
sudo aide --config=/etc/aide/aide-lab.conf --check
```

Die Prüfung vom **22. Oktober um 10:17:15 UTC** erkannte einen geänderten Eintrag unter 188 Einträgen, ohne hinzugefügte oder gelöschte Einträge.

| Attribut der SSH-Datei | Referenz | Nach dem auditd-Kommentar |
|---|---|---|
| Grösse | 134 Byte | 173 Byte |
| `mtime` | 20.10.2026 um 12:11:54 UTC | 22.10.2026 um 10:11:25 UTC |
| `ctime` | 22.10.2026 um 09:53:35 UTC | 22.10.2026 um 10:11:25 UTC |
| SHA-256 | Referenzhashwert | Abweichender Hashwert |

Der `mtime` stimmt auf die Sekunde genau mit dem Zeitpunkt des auditd-Ereignisses überein. Die beiden Werkzeuge liefern ergänzende Nachweise: AIDE erkennt die Abweichung von der Referenz, und auditd ermöglicht die Einordnung des Vorgangs in seinen Kontext.

### 17.4 Wiederherstellung eines sauberen Zustands

```bash
sudo cp -a /root/00-lab-hardening.conf.avant-test-auditd /etc/ssh/sshd_config.d/00-lab-hardening.conf
sudo cmp /root/00-lab-hardening.conf.avant-test-auditd /etc/ssh/sshd_config.d/00-lab-hardening.conf
sudo /usr/sbin/sshd -t
sudo aide --config=/etc/aide/aide-lab.conf --check
```

Das Vorgehen sieht vor, vor der Übernahme des neuen Zustands zu prüfen, ob nur der erwartete Unterschied beim `ctime` verbleibt. Anschliessend wurde die Referenz nach einer vorgängigen Sicherung aktualisiert:

```bash
sudo cp -a /var/lib/aide/aide-lab.db /root/aide-lab.db.avant-actualisation-auditd
sudo aide --config=/etc/aide/aide-lab.conf --update
```

Nach Prüfung des Berichts:

```bash
sudo cp /var/lib/aide/aide-lab.db.new /var/lib/aide/aide-lab.db
sudo chmod 600 /var/lib/aide/aide-lab.db
sudo aide --config=/etc/aide/aide-lab.conf --check
```

Das um **10:22:51 UTC** tatsächlich übermittelte Endergebnis lautete:

```text
AIDE found NO differences between database and filesystem. Looks okay!!
Number of entries: 188
```

Für die Zwischenschritte dieser zweiten Wiederherstellung liegt in den übermittelten Unterlagen keine detaillierte Ausgabe vor; die abschliessende Prüfung bestätigt die Rückkehr zu einem Zustand, der der aktivierten Referenz entspricht.

## 19. Bilanz der zusätzlichen Tests

Die Tests T01 bis T17 sind in den Abschnitten 9 und 13 aufgeführt. Die ergänzenden Überprüfungen lauten wie folgt:

| ID | Überprüfung | Beobachtetes Ergebnis | Status |
|---|---|---|---|
| T18 | Anfängliche AIDE-Referenz | Keine Unterschiede, 188 Einträge um 09:50:26 UTC | Bestätigt |
| T19 | Kontrollierte Änderung der SSH-Datei | Ein geänderter Eintrag; Grösse von 134 auf 171 Byte und abweichender SHA-256-Hashwert | Bestätigt |
| T20 | Wiederherstellung nach dem ersten Test | Nur der `ctime` weicht ab; Inhalt gemäss den überwachten Attributen wiederhergestellt | Bestätigt |
| T21 | Kontrollierte Aktualisierung der Referenz | Keine Unterschiede um 09:55:49 UTC | Bestätigt |
| T22 | Laden der auditd-Regeln | Drei Regeln vorhanden, Audit aktiviert, `lost 0` | Bestätigt |
| T23 | Nachvollziehbarkeit des Vorgangs | Ereignis 140: `tee`, SSH-Datei, `auid=vagrant`, `euid=root`, `success=yes` | Bestätigt |
| T24 | Korrelation zwischen AIDE und auditd | Eine Änderung erkannt; `mtime` stimmt mit dem Zeitpunkt des Ereignisses überein | Bestätigt |
| T25 | Abschluss des kombinierten Tests | Keine AIDE-Unterschiede um 10:22:51 UTC nach Wiederherstellung und Aktualisierung | Bestätigt |
