# Noob - HackMyVM (Easy)

![Noob.png](Noob.png)

## Übersicht

*   **VM:** Noob
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Noob)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 14. Juli 2021
*   **Original-Writeup:** https://alientec1908.github.io/Noob_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, die User- und Root-Flags der Maschine "Noob" zu erlangen. Der Weg dorthin führte über einen Golang-Webserver, der auf einem ungewöhnlichen Port (65530) lief. Durch Enumeration wurde entdeckt, dass dieser Webserver ein Verzeichnis (`/nt4share/.ssh/`) exponierte, das den öffentlichen und – kritischerweise – den privaten SSH-Schlüssel für den Benutzer `adela` enthielt. Dies ermöglichte einen direkten SSH-Login als `adela`. Die finale "Privilegieneskalation" zum Lesen der Root-Flag erfolgte durch Ausnutzung einer Local File Inclusion (LFI)-ähnlichen Schwachstelle, die durch das Erstellen eines symbolischen Links im Home-Verzeichnis von `adela` (auf das der Webserver Zugriff hatte) auf das `/root`-Verzeichnis ermöglicht wurde. Über den Webserver konnte so der Inhalt von `/root`, einschließlich der Root-Flag, ausgelesen werden.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `curl`
*   `vi`
*   `chmod`
*   `ssh`
*   `sudo` (versucht)
*   `find`
*   `ln`
*   Standard Linux-Befehle (`ls`, `id`, `cat`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Noob" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.118) mit `arp-scan` identifiziert. Hostname `noob.hmv` in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 7.9p1) und einen ungewöhnlichen Port 65530 (HTTP, Golang net/http server).
    *   `gobuster` auf `http://192.168.2.118:65530/` fand nur einen Endpunkt `/index`.
    *   Durch (nicht explizit im Log gezeigte) weitere Enumeration oder Raten wurde der Pfad `/nt4share/.ssh/` auf dem Go-Webserver entdeckt.

2.  **Information Disclosure & Initial Access (SSH als `adela`):**
    *   Über den Webserver wurde auf `http://192.168.2.118:65530/nt4share/.ssh/id_rsa.pub` zugegriffen, was den öffentlichen Schlüssel und den Benutzernamen `adela@noob` enthüllte.
    *   Anschließend wurde über `http://192.168.2.118:65530/nt4share/.ssh/id_rsa` der private SSH-Schlüssel für `adela` heruntergeladen.
    *   Der private Schlüssel wurde gespeichert (`idroot`), die Berechtigungen gesetzt (`chmod 600 idroot`).
    *   Erfolgreicher SSH-Login als `adela` mit dem gefundenen privaten Schlüssel (`ssh -i idroot adela@noob.hmv`).

3.  **Privilege Escalation (LFI via Symlink & Root-Dateien lesen):**
    *   Als `adela` wurde festgestellt, dass `sudo` nicht verfügbar war und die SUID-Suche keine einfachen Vektoren lieferte.
    *   Ein symbolischer Link wurde im Home-Verzeichnis von `adela` erstellt (z.B. `ln -s /root /home/adela/nt4share/root` oder `ln -s /root /home/adela/rootlink`). Der Go-Webserver hatte Zugriff auf dieses Verzeichnis.
    *   Durch Aufrufen des entsprechenden Pfades über den Webserver (z.B. `http://192.168.2.118:65530/nt4share/root/`) konnte der Inhalt des `/root`-Verzeichnisses aufgelistet werden.
    *   Die User-Flag (`HMVVCXUIUHFDSUIYREWDS432`) wurde über `http://192.168.2.118:65530/nt4share/root/user.txt` gelesen.
    *   Die Root-Flag (`HMVA97DSA8732HJGDSA78623`) wurde über `http://192.168.2.118:65530/nt4share/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Exponierte SSH-Schlüssel:** Der private SSH-Schlüssel des Benutzers `adela` war über den Webserver öffentlich zugänglich.
*   **Unsichere Webserver-Konfiguration (Path Traversal / LFI via Symlink):** Der Go-Webserver erlaubte das Folgen von symbolischen Links, die von einem Benutzer mit geringeren Rechten (`adela`) erstellt wurden und auf sensible Verzeichnisse (`/root`) außerhalb des vorgesehenen Web-Roots zeigten. Dies ermöglichte das Auslesen von Dateien mit den Rechten des Webserver-Prozesses (der implizit Leserechte auf `/root` hatte oder der Symlink-Trick dies umging).
*   **Dienste auf Nicht-Standard-Ports:** Der Webserver lief auf dem unüblichen Port 65530.

## Flags

*   **User Flag (gelesen aus `/root/user.txt` via LFI):** `HMVVCXUIUHFDSUIYREWDS432`
*   **Root Flag (gelesen aus `/root/root.txt` via LFI):** `HMVA97DSA8732HJGDSA78623`

## Tags

`HackMyVM`, `Noob`, `Easy`, `SSH Key Leak`, `Information Disclosure`, `Path Traversal`, `LFI`, `Symlink Abuse`, `Golang Webserver`, `Linux`, `Web`, `Privilege Escalation` (File Read)
