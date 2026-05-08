# CLAUDE.md — D4 Build Extractor

## Vad det här är

Ett personligt verktyg för att extrahera Diablo 4 build-guider från webben
till strukturerade Markdown-filer. Användarna är Niklas och Ida.

Output-filerna är inte byggreferenser i traditionell mening — de är
**walkthroughs** avsedda att läsas på telefonen medan paragon-poäng sätts ut
i spelet, i situationer där källsidan är för långsam eller buggig att
använda mobilt.

## Språk

- Användaren skriver på svenska. Svara på svenska i chatten — frågor,
  förtydliganden, kommentarer, resonemang om osäkerheter.
- Allt **filinnehåll** (extraktionerna i `extractions/`, eventuella
  artifacts) skrivs på engelska. In-game-termer behåller sin engelska
  originalstavning oavsett.
- Förklarande text i filer är engelsk. In-game-termer översätts inte.

## Verktyg och eskalering

Två MCP-servrar är konfigurerade:

- **fetcher-mcp** — snabb, billig, returnerar Markdown via Playwright +
  Readability. Använd för prosa-tunga sidor (build guides, tier lists,
  patch notes).
- **Microsoft Playwright MCP** — full browser-session med navigation,
  klick, vänta-på-element, riktade screenshots, accessibility tree. Dyrare
  i tid och kontext. Använd när strukturerad data ligger bakom
  interaktiva widgets eller renderas visuellt.

**Beslutsregel:**

1. Börja med fetcher-mcp för alla URL:er.
2. Eskalera till Playwright MCP om någon av dessa är sant:
   - URL:en är en planner-sida (`/d4/planner/...`)
   - Användaren uttryckligen vill ha paragon- eller skill-tree-detaljer
   - fetcher-mcp:s output saknar uppenbara fält (paragon board order,
     skill ranks, slot-by-slot gear) som faktiskt finns på sidan
3. För Maxroll-builds: hämta typiskt *både* build guide-URL (för prosa,
   rotation, mercenary) *och* länkad planner-URL (för gear slots,
   board order, attribut). Slå ihop till en output.

**Hur Playwright används för paragon:**

Paragon-noder ligger ofta som canvas-renderingar utan textuella nodnamn i
DOM. Flödet:

1. Navigera till plannern.
2. Vänta tills paragon-vyn är fullt renderad (`networkidle` + extra
   väntan vid behov).
3. Identifiera antalet boards och deras namn från headers.
4. För varje board: skrolla in i viewport, ta riktad screenshot.
5. Läs av nodallokering, glyph-radie och attribut från bilden.
6. Om en specifik nod är osäker — försök öppna dess tooltip via klick
   eller hover för att läsa namnet textuellt.
7. Notera kvarvarande osäkerheter i `## Uncertainties`.

## Hard rules (do not break)

1. Only include information explicitly present on the page(s) provided.
2. Never invent skill names, item names, aspects, uniques, paragon nodes,
   glyphs, mercenary skills, or numeric values. If it isn't on the page,
   it doesn't go in the file.
3. Preserve exact spelling and capitalisation of in-game terms as written
   on the page (e.g. "Shadow Step", not "shadowstep").
4. If a template field isn't covered, write `_Not specified_`. Never
   guess. Never substitute "common knowledge" about the build.
5. If the page contains multiple variants (leveling vs endgame, board
   rush vs main, T1 vs Pit push, solo vs group), output each under its
   own heading. Do not merge.
6. Numbers (ranks, %, breakpoints, paragon node counts) are copied
   verbatim. Hedges like "around X" are kept as written.
7. Ambiguities and contradictions go in `## Uncertainties` — never
   silently picked.
8. Visual data (paragon boards, skill trees) read from screenshots is
   *probably* not 100% accurate. Default stance: include a verification
   note. Don't pretend to perfect recall.

## Etiskt anropsmönster

Det här verktyget är designat för enskilt bruk. Källsidornas robots.txt
och rate-limiting respekteras *de facto* genom följande regler:

- Ett anrop åt gången, sekventiellt. Inga parallella fetches.
- Minst några sekunders paus mellan calls om en extraktion kräver flera
  sidor.
- Inga batch-jobb. En extraktion = en build, manuellt initierad.
- Om en sida tycks blockera eller throttla — backa av, säg till
  användaren, automatisera inte runt det.

## Arbetsflöde per chatt

Användaren startar en ny chatt med en URL eller en build-beskrivning.

1. Identifiera vilken sida/sidor som behöver hämtas. För Maxroll:
   typiskt build guide + planner.
2. Hämta sekventiellt enligt eskaleringsregeln ovan.
3. Identifiera variant-frågor tidigt: om sidan har flera paragon-setups
   (board rush vs main, leveling vs endgame), fråga vilken som ska
   extraheras innan du producerar outputen — gissa inte.
4. Producera Markdown enligt mallen nedan.
5. Spara som `extractions/{class}-{build-name-slug}.md` om filsystem-
   åtkomst finns. Annars: leverera som artifact.
6. Skriv en kort svensk sammanfattning i chatten — vad som lyckades, vad
   som saknades på källsidan, eventuella varningar.

## Output template

Använd exakt dessa rubriker, i den här ordningen.

```markdown
# {Build name} — {Class}

## Metadata
- **Source:** {primary URL(s)}
- **Author / site:** {…}
- **Patch / season:** {…}
- **Last updated:** {…}
- **Build type:** {leveling / endgame / pit push / boss / …}
- **Variant extracted:** {if multiple variants exist on the page}

## Summary
2–4 sentences from the page describing playstyle and identity. Tight
paraphrase, no long copied passages.

## Skills
| Slot | Skill | Rank | Notes |
|------|-------|------|-------|

## Passives & class mechanic
- {bullet per passive or mechanic choice}

## Gear
For each slot the page covers:
### {Slot}
- **Item / unique:** {…}
- **Aspect:** {…}
- **Affixes (priority order):** {…}
- **Tempering:** {…}
- **Masterwork focus:** {…}

## Gems
- {…}

## Paragon (walkthrough format)

For each board, in pickup order:

### Board {n}: {Name}
- **Glyph:** {glyph} (target Lv. {…})
- **Attributes:** Str {…} / Int {…} / Will {…} / Dex {…}
- **Path from gate:** {direction-based summary, e.g. "up to legendary,
  left through rare cluster"}
- **Legendary node:** {name + brief effect if visible}
- **Rare nodes (in pickup order):** {list}
- **Magic clusters:** {list with brief descriptions}
- **Skip:** {explicitly named nodes that look attractive but should not
  be taken}

(For boards where exact node-by-node allocation cannot be reliably read
from the source, state that and link back to the planner.)

## Mercenary
- **Hired:** {…}
- **Reinforcement:** {…}
- **Notes:** {…}

## Stat priority
Ordered list as given on the page.

## Rotation / playstyle
Bullet points or numbered steps.

## Pros / cons
Only if the page lists them.

## Leveling notes
Only if the page covers leveling separately.

## Uncertainties
- Anything ambiguous, contradictory, or that required interpretation.
- Any visual data (paragon boards, skill trees) where reading confidence
  is below high.
- Anything the page references but does not fully specify.

## Verification
Before following this walkthrough end-to-end, the user should:
- Open the source planner once and spot-check the first 2–3 nodes per
  board against this file.
- Confirm any item/aspect/tempering choices flagged in `## Uncertainties`.
```

## Standing self-check

Before delivering any extraction, mentally verify:

- Every term, number, and item is traceable to the source. If not, remove.
- No blank fields — `_Not specified_` is used.
- All variants are represented (or the chosen variant is named in
  Metadata).
- `## Uncertainties` is honest. Empty is fine if nothing was ambiguous;
  pretending certainty is not.
- Paragon nodes from screenshots include a verification note.
- "Points spent" totals (if visible on source) match the number of nodes
  described in the walkthrough.
- File content is in English; chat reply is in Swedish.

## Examples

See `examples/` for reference extractions. New extractions should match
that format and depth — neither more terse nor more elaborate.
