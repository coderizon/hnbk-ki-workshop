# Fragenkatalog zum Ausprobieren der Wissensdatenbank

Mit diesen Fragen kannst du das Retrieval testen, nachdem du die vier Dokumente in
Open WebUI hochgeladen hast. Zu jeder Frage steht die erwartete Antwort, damit du
selbst prüfen kannst, ob das System richtig aus den Dokumenten antwortet.

## So gehst du vor

1. Lege in Open WebUI unter **Arbeitsbereich → Wissen** eine Sammlung an,
   z. B. „Beispieldokumente".
2. Lade die **vier** Dokumente hoch:
   `it-ordnung.md`, `brandschutzordnung.md`,
   `produktdatenblatt-packesel-x2.pdf`, `exkursion-zeche-hannibal.pdf`.
   (Diese Datei und die `README.md` werden **nicht** hochgeladen.)
3. Starte einen Chat, hänge die Wissenssammlung an deine Frage an und stelle die
   Fragen unten.
4. Achte unter jeder Antwort auf die **angezeigten Quellen**: Aus welchem Dokument
   und welchem Abschnitt stammt die Antwort?

## Fragen zur IT-Ordnung

1. **Wie heißt das unterrichtliche WLAN und wo erfährt man das Passwort?**
   *Erwartet:* HNBK-Lab; das Passwort hängt im Sekretariat in Raum A101 aus.
2. **Wie viele Seiten darf eine Schülerin pro Halbjahr drucken, und wo ist Farbdruck möglich?**
   *Erwartet:* 500 Seiten pro Halbjahr; Farbdruck nur in der Bibliothek.
3. **Wie heißen die beiden lokalen Server und in welchem Raum stehen sie?**
   *Erwartet:* Spark-Nord und Spark-Süd, im Serverraum B114.

## Fragen zur Brandschutzordnung

1. **Wo ist der Sammelplatz im Brandfall?**
   *Erwartet:* Parkplatz Nord gegenüber Halle 3.
2. **Wer ist die Brandschutzbeauftragte und wo ist sie zu finden?**
   *Erwartet:* Frau Keller, Raum A004 (Durchwahl 204).
3. **Wie oft finden Evakuierungsübungen statt?**
   *Erwartet:* Zweimal jährlich, im Herbst und im Frühjahr.

## Fragen zum Produktdatenblatt (Lastenrad Packesel X2)

1. **Wie hoch ist die maximale Zuladung und wie weit reicht der Akku?**
   *Erwartet:* 180 kg Zuladung; Reichweite bis zu 90 km.
2. **Was kostet das Packesel X2 und wie lange gilt die Garantie auf den Rahmen?**
   *Erwartet:* 4.299 €; 5 Jahre Garantie auf den Rahmen (2 Jahre auf den Akku).
3. **In welchen Farben ist das Lastenrad erhältlich?**
   *Erwartet:* Tannengrün, Anthrazit und Signalrot.

## Fragen zur Exkursion (Zeche Hannibal)

1. **Wann und wo ist der Treffpunkt für die Exkursion?**
   *Erwartet:* Freitag, 25. September 2026, am Haupteingang der Schule um 08:15 Uhr.
2. **Was kostet die Exkursion und bis wann muss bezahlt werden?**
   *Erwartet:* 12 € pro Person; bar bis zum 18. September 2026 bei Frau Vogt.
3. **Was muss man mitbringen?**
   *Erwartet:* Festes Schuhwerk, Regenjacke, eigene Verpflegung und den
   Schülerausweis. Der Helm wird vor Ort gestellt.

## Dokumentübergreifende Fragen

Diese Fragen lassen sich nur beantworten, wenn das System aus **zwei** Dokumenten
zusammenträgt.

1. **Welche Durchwahl hat der IT-Support, und welche der interne Sicherheitsdienst?**
   *Erwartet:* IT-Support 230 (aus der IT-Ordnung); Sicherheitsdienst 555 (aus der
   Brandschutzordnung).
2. **In welchen Räumen stehen CO2-Feuerlöscher, und was steht laut IT-Ordnung in einem dieser Räume?**
   *Erwartet:* CO2-Löscher in B112 und B114; in B114 stehen die Server Spark-Nord
   und Spark-Süd, in B112 sitzt der IT-Support.

## Erdungs-Test (Antwort steht NICHT in den Dokumenten)

Diese Fragen prüfen, ob das System ehrlich bleibt, statt zu raten. Eine gute
Antwort sagt sinngemäß, dass die Information nicht in den Unterlagen steht.

1. **Wie lautet das aktuelle WLAN-Passwort für HNBK-Lab?**
   *Erwartet:* Das Passwort steht in keinem Dokument (es hängt nur im Sekretariat
   aus). Das System sollte das so benennen und keine Zahl erfinden.
2. **Wie viele Mitarbeitende hat die Ruhrrad Manufaktur GmbH?**
   *Erwartet:* Diese Angabe ist in den Dokumenten nicht enthalten.

## Tipp

Stelle dieselbe Frage einmal **mit** und einmal **ohne** angehängte
Wissenssammlung. Ohne die Dokumente kann das Modell die schulspezifischen Fakten
nicht kennen, mit den Dokumenten schon. Das macht den Nutzen von RAG am
deutlichsten sichtbar.
