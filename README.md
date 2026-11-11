# claude-quiz

En `/quiz`-kommando til [Claude Code](https://claude.com/claude-code). Den laver studiequizzer for et vilkårligt fag ud fra dine læringsmål.

- Claude laver en ny statisk HTML-quiz med 10 spørgsmål pr. batch. Ca. en tredjedel er "for dummies", resten er scenarier.
- Svar bliver tjekket i browseren med det samme, med en forklaring. Svarmulighederne blandes.
- Med **Out of scope** (spørgsmål eller hele emnet) bliver spørgsmålet eller emnet udeladt fremover.
- Når du trykker "Færdig", åbner næste batch efter 5 minutter. Emner, du svarer forkert på, kommer igen i ny form.
- Ingen installation: kun Claude Code og en browser. Virker på Windows og Mac.

## Installation
I Claude Code:
```
/plugin marketplace add RasmusRosengaard/claude-quiz
/plugin install quiz@claude-quiz
```
Genstart derefter Claude Code.

Alternativt uden plugin-systemet:
```
curl -o ~/.claude/commands/quiz.md https://raw.githubusercontent.com/RasmusRosengaard/claude-quiz/main/commands/quiz.md
```

## Brug
```
/quiz databaser
```
Hvis faget er nyt, spørger Claude efter læringsmål, fokusemner, ønsker og (valgfrit) undervisningsmateriale: PowerPoints, PDF'er, Word-filer, mapper, links eller bare bognavne. Materialet læses én gang og opsummeres, så hver quiz forbliver billig. Ved hver start vælger du løbende quizzer eller én quiz nu, antal spørgsmål og interval. `/quiz list` viser alle fag, og `/quiz stop <fag>` stopper en quiz.

Data for hvert fag ligger i `~/tools/quiz/<fag>/`. Resultatfilerne downloades midlertidigt til `~/Downloads`. Browseren må derfor ikke være sat til "Spørg, hvor filer skal gemmes".
