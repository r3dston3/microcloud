# Antworten – Projekt „Stack unter Versionskontrolle mit CI“

Serhii K. · 11ITS2 · LF10 · KW47

---

## Phase 1 – SSH-Schlüssel (Kontrollfragen)

1. Bei GitHub hochgeladen wird nur `id_ed25519.pub`, der **öffentliche** Schlüssel. Der private Schlüssel `id_ed25519` bleibt auf dem Rechner: Wer ihn hat, kann sich als ich ausgeben und in meine Repos pushen.
2. GitHub schickt eine Challenge, die nur mit dem passenden privaten Schlüssel gelöst werden kann. Mein Rechner löst sie lokal, GitHub prüft die Antwort mit dem hinterlegten öffentlichen Schlüssel. Ein Passwort wird nie übertragen.

## Phase 2 – Der erste rote Lauf

1. Lauf 2 scheitert in Schritt **4 · Port 8085 prüfen**.
2. Fehlermeldung (Annotation „Port 8085 wird nicht veröffentlicht“):
   > Kein Service in deiner compose.yml veröffentlicht den Host-Port 8085. Aktuell veröffentlicht: kuma → 3001:3001, web → 8080:80.
3. Der Stack ist technisch in Ordnung und lief in Modul 4 auch. Er erfüllt aber die **neue Vorgabe** der Firma nicht: Die Webdienste müssen auf dem Host-Port **8085** erreichbar sein, mein Stack veröffentlicht noch 8080. Der automatische Test prüft genau diese vereinbarte Anforderung.
4. Übersprungen wurden die Schritte **5–7** (Stack starten, Erreichbarkeit, Inhalt), weil ein Job nach dem ersten fehlgeschlagenen Schritt abbricht. Trotzdem ausgeführt wurden **8** (Steckbrief, `if: always()`), **9** (Logs, `if: failure()`) und **10** (Aufräumen, `if: always()`).

## Phase 3 – Port 8085

Änderung in der `compose.yml`: `"8080:80"` → `"8085:80"` (links Host-Port, rechts Container-Port). Lokal getestet mit `docker compose up -d` und `curl http://localhost:8085`, danach committet und gepusht → CI grün.

---

## Phase 4 – Den Workflow verstehen

### Teil A: Aufbau der Workflow-Datei

1. **Events:** `push` (auf jeden Branch), `pull_request` (mit Ziel-Branch `main`) und `workflow_dispatch` (manuell per Knopf im Tab Actions).
2. **Runner:** `ubuntu-latest` – eine frische virtuelle Ubuntu-Maschine, die **GitHub** für jeden Lauf bereitstellt (GitHub-hosted Runner) und danach wieder verwirft. Docker und Docker Compose sind vorinstalliert.
3. **Job:** `stack-test` mit dem Anzeigenamen „Compose-Stack testen (Port 8085)“. Er hat **10 Steps**.
4. **Action vs. run:** Nur Step 1 nutzt eine fertige Action (`uses: actions/checkout@v7`, lädt den Code auf den Runner). Die Steps 2–10 führen eigene Shell-Befehle mit `run:` aus.
5. **Rechte:** `permissions: contents: read` – der Workflow darf den Code nur lesen, nichts ändern oder veröffentlichen. Das ist das **Prinzip der minimalen Rechte**: Wird ein Schritt (z. B. eine fremde Action) kompromittiert, kann er kaum Schaden anrichten.
6. **`if: !github.event.repository.is_template`:** In der Vorlage (Template-Repository) gibt es noch keinen Stack, dort wird der Job **übersprungen**, damit die Vorlage nicht dauerhaft rot ist. In meinem Repo (kein Template) läuft der Job immer.

### Teil B: Was wird geprüft?

7. Zuerst schlägt fehl …
   1. Tabulator statt Leerzeichen → Step **3 · YAML-Syntax prüfen**
   2. Ordner `html/` nie committet → Step **7 · Inhalt prüfen** (`html/index.html` fehlt)
   3. `ports: - "8085:8080"` → Step **6 · Erreichbarkeit**: Docker leitet auf Container-Port 8080 weiter, nginx lauscht aber auf 80, also antwortet niemand.
   4. `compose.yml` in einem Unterordner → Step **2 · Compose-Datei suchen**
8. Ein Webserver antwortet auch dann auf Port 8085, wenn er **die falsche Seite** ausliefert – z. B. seine eingebaute Standardseite, weil das Volume an der falschen Stelle eingebunden ist. Genau das passiert in Phase 5 beim Wechsel auf Apache („It works!“). Ein guter Test prüft das **Ergebnis, das der Kunde erwartet**, nicht nur, ob „irgendwas läuft“.
9. Step 10 sorgt für ein sauberes Ende: Container, Netzwerke und Volumes werden entfernt. Auf **selbst gehosteten Runnern**, die wiederverwendet werden, würden sonst Container liegen bleiben und Ports blockieren – der nächste Lauf würde dann z. B. mit „port is already allocated“ scheitern. Aufräumen gehört zu jedem sauberen Test.

### Teil C: Einordnen

10. Das ist **Continuous Integration**: Jede Änderung wird automatisch geprüft und getestet. **Continuous Delivery** (fertiges Artefakt, z. B. Image in einer Registry bereitstellen) und **Continuous Deployment** (automatisches Ausrollen auf den Server) finden nicht statt – den LXC aktualisiere ich selbst mit `git pull` und `docker compose up -d`.
11. Vorteile gegenüber „mal kurz mit `curl` testen“:
    - Der Test läuft **bei jeder Änderung automatisch** und wird nie vergessen.
    - Er läuft **immer gleich** in einer sauberen Umgebung („bei mir geht's“ zählt nicht).
    - Fehler fallen **vor** dem Betrieb auf, und das Ergebnis ist **dokumentiert und für alle sichtbar** (Haken am Commit, Abzeichen im README).

### Teil D: Die DoD-Abzeichen

12. Ein Abzeichen zeigt immer den letzten Lauf auf **einem** Branch. DoD 3 prüft, ob es den Branch `dev` mit eigener Änderung gibt – also muss die Prüfung laufen, wenn auf `dev` gepusht wird, und das Abzeichen muss die Läufe auf `dev` zeigen (`?branch=dev`). Apache auf `main`, Repo-Inhalt und Pull Request betreffen dagegen den Betriebsstand `main`.
13. `dod-5-pull-request.sh` braucht: den zuletzt gemergten Pull Request von `dev` nach `main` mit seinem Merge-Zeitpunkt (`merged_at`) und seinem letzten Commit (`head.sha`), ob er überhaupt Dateien ändert (`changed_files`), und das Ergebnis des Checks „Compose-Stack testen (Port 8085)“ für genau diesen Commit mit Abschlusszeitpunkt (`completed_at`). Grün ist DoD 5 nur, wenn der Check **erfolgreich** war **und vor dem Merge** fertig war.
