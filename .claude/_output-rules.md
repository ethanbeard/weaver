# Output rules

Cross-cutting quality standards for everything you write — chat responses,
files, memory entries, loop prompts, skill outputs. Read once, apply always.

## No slop

Claude Code chat output and workspace file writes are both subject to the
same rule: no filler.

- **No openers**: "Certainly!" / "Here's what I've done:" / "I've created a..."
- **No throat-clearing**: "It's worth noting that..." / "Interestingly..." / "That's a great question."
- **No unnecessary hedging**: "According to the source, X" not "X might possibly be the case."
- **No placeholder dates**: "in the near future" / "recently" / literal "YYYY-MM-DD".
- **No pre-narration**: don't announce what you're about to do ("I'll now X, then Y"). Just do it and report briefly if needed.

Short paragraphs. Concrete claims. Cite when appropriate.

## Exact phrasing preservation

When capturing the human's own words — in `IDENTITY.md`, `SOUL.md`,
`USER.md`, `memory/*.md`, or any file that records what they said — preserve
their phrasing. Don't paraphrase. Don't clean up grammar. The language IS
the insight.

- Direct quotes: verbatim, in quote blocks if appropriate
- Stated preferences: use their exact terminology  
- Ideas and frameworks: use the human's own phrasing for slugs, titles, and section headers

If someone says "I hate when my agent pre-narrates," don't record that as
"User dislikes verbose preambles from the assistant." Record the actual
sentence.

## Deterministic links

When a file produces a URL or path, use the actual value. Don't reconstruct
it from a guess.

- **External links**: use the URL the source provides. Never assume a pattern like "it's probably at example.com/{topic}".
- **Workspace file paths**: use paths you've verified (via `Glob`, `Read`, `ls`). Don't link to files you haven't confirmed exist.
- **Memory cross-references**: `[earlier note](memory/2026-04-15.md)` — spell the actual filename.

If you don't have the real URL or path, say so. Don't fabricate.

## Title and filename quality

Titles (for memory entries, loop descriptions, proposed automations):

- Descriptive enough to identify the content from a search result
- Under 60 characters
- Not full sentences — "Morning briefing" beats "A scheduled task that checks email every morning"
- Not generic — "Ethan's art collecting notes" beats "Notes"

Filenames (in `memory/`, `.claude/loops/`, new skill directories):

- **kebab-case**, no spaces, no special characters except `-` and `.`
- Describe content, not timing (unless the file is inherently date-based like `memory/YYYY-MM-DD.md`)
- Match the name in any frontmatter `name:` field — they shouldn't disagree

## When something's ambiguous, ask

If the human's input is terse and you need more context to write a
quality file or memory entry, ask one clarifying question rather than
filling in with plausible details.

You can produce lower-quality output fast, or higher-quality output after
one clarifying turn. The second option compounds; the first doesn't.

## Silence is an output

For automations, loops, and background work: producing nothing at all is
a valid, often correct response. Don't fill fires with "Nothing notable
since last check" when you could just end the turn silently.

The signal the human wants is the real finding, not a running commentary
on the absence of findings.
