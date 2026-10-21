---
title: "Timelines That Don't Invent Dates"
date: 2026-10-21 12:00:00 +0530
categories: [AI, Projects]
tags: [claude, agent-skills, python, timeline, calendars, panchang]
image:
  path: /assets/img/posts/chronology/timeline-conflict.jpg
  alt: A chronology merged from three sources, with a conflicting meeting time flagged
---

Ask a chatbot to put a document's events in order and you'll get a neat,
confident timeline. Look closer and three things usually go wrong:

- **Fake precision.** A date known only as "July 1969" becomes *16 July 1969*.
- **Silent choices.** Three sources give three times for the same meeting; the
  timeline shows one of them and doesn't mention the other two.
- **Calendar blindness.** Shakespeare and Cervantes both "died on 23 April 1616".
  They died **eleven days apart**: England still used the Julian calendar, and
  Spain had moved to the Gregorian.

I built **[CHRONOLOGY](https://github.com/gauravnavgire/chronology)** to fix
exactly these problems. It's an open-source [Agent
Skill](https://github.com/gauravnavgire/chronology/blob/main/chronology/SKILL.md)
for Claude, plus a standalone Python command-line tool that needs nothing beyond
the standard library.

> **Try it first:** the [interactive demo](https://gauravnavgire.github.io/chronology/)
> merges a witness statement, a police log and a newspaper report into one
> timeline. Click any issue in the side panel.
{: .prompt-tip }

## The idea: let the model read, let code do the dates

Language models are good at *reading*: working out that "on Tuesday" in an
article dated Friday 14 March means 11 March, or that two descriptions refer to
the same meeting. They are much less reliable at the mechanical parts: date
arithmetic, BCE years, time zones, calendar conversion, and consistent sorting.

So the skill splits the work:

1. **Claude extracts** each event with its date as precise as the source allows,
   and **the source's exact words** for it.
2. **A deterministic script** (`chronology.py`) does everything involving dates:
   ordering, grouping, checks, calendar conversion and rendering.
3. **A self-check** looks every quote up in the original text before Claude
   answers.

## Every date is a window

Instead of a single point, every date is a range of possible moments:
"late 1970s" is 1975–1979, and "c. 753 BCE" is 754–752 BCE. That sounds like a
small change, but it lets the tool be honest about **order**. If two events'
windows overlap, the source doesn't actually say which came first, so the
event is marked **≈** rather than placed with false certainty. And a date known
only to the month is shown as *date known only to month*, never quietly
pinned to a day.

## When sources disagree, you see all of them

Give it several documents, and events that describe the same happening are
merged. The output shows how many sources back each event. When the sources
disagree, the event is marked **⚠** and each source keeps its own date. In
"lanes by source" view the disagreement is impossible to miss:

![Lanes by source: the same meeting at 14:02, 14:10 and the next day](/assets/img/posts/chronology/lanes-by-source.jpg)
_The same meeting, according to a witness, the police log and a newspaper._

A consistency report then lists what can't all be true:

```text
ERROR   [order]       'Report filed' (11 Jun 1998) should be after 'Smith meets Jones' (12 Jun 1998 14:02) but is entirely before it
ERROR   [offset]      'Phone call' (15 Jun 1998) is stated, but +2 days from 'Smith meets Jones' gives 14 Jun 1998
ERROR   [anachronism] 'Smith tweets' (1990) is before Mar 2006, the earliest it could be: Twitter launched March 2006
WARNING [lifespan]    'Letter signed by Smith' (2003) involves Smith after 'Smith dies' (10 May 2001)
```
{: .nolineno }

Contradictions are **reported, never fixed**. Changing a date to make a story
consistent is exactly the kind of quiet "help" a timeline shouldn't give.

## Catching invented details

Each event carries a short quote from the source, and `--verify` looks every
one up in the original text. To test it, I planted four errors in a set of
letters: an invented quote, a misquote ("pay rates" where the source says
"piece-rates"), a wrong year and a wrong page. All four were caught:

```text
ERROR   [verify] 'Hale meets the strikers': quote not found in letters.txt
WARNING [verify] 'Carders' strike begins': quote matches only approximately (97%). Source has: "…over the new piece-rates."
WARNING [verify] 'Mill destroyed by fire': the quote mentions 1888 but the event is dated 12 Nov 1887
WARNING [verify] 'Rebuilding planned': quote is on page 3 of letters.txt, but cited as p. 1
```
{: .nolineno }

## Eight calendars, including the Hindu panchang

Dates are written the way the source writes them (`julian:1616-04-23`,
`hijri:1445-ramadan`, `hebrew:5784-adar2-14`, `japanese:Showa 20-08-15`,
regnal and fiscal years) and converted exactly, keeping the original wording.

The one I enjoyed most was the **Hindu panchang**. A date like *Vikram Samvat
2081, Chaitra shukla 9* can't be converted with a formula. The tithi depends on
where the Sun and Moon actually are at local sunrise, and the month names
depend on which sidereal sign the Sun occupies at each new moon. So the tool
computes the astronomy itself, in pure Python: Meeus's lunar and solar series,
Lahiri ayanamsa and a sunrise calculation. The checks:

- new moons agree with published times to **within a minute**;
- Ugadi, Ram Navami and Holi land on the right days, with tithi start and end
  times **within about two minutes** of published panchangs;
- the leap months (*adhika masa*) of 2020, 2023 and 2026 come out right.

Then two historical records. Shivaji's coronation, *Jyeshtha shukla 13, Shaka
1596*, computes to **16 June 1674**, which is exactly the conventional "6 June
1674" read as an Old Style date. And Gandhi's birth tithi, *Bhadrapada vad 12*,
turned out to be current at sunrise only on 3 October 1869, yet he was born on
2 October. The answer: the tithi **began at 09:47 on 2 October**, and a birth is
dated by the tithi at that moment, not at sunrise. The tool now reports when
each tithi actually ran, for exactly this reason.

## Install

**Claude.ai / Claude Desktop:** download `chronology.zip` from the
[latest release](https://github.com/gauravnavgire/chronology/releases/latest) and
upload it under *Customize → Skills*.

**Claude Code:**

```bash
git clone https://github.com/gauravnavgire/chronology
cp -r chronology/chronology ~/.claude/skills/
```
{: .nolineno }

Then just ask: *"Here are two witness statements and the dispatch log: merge
them into one chronology and tell me where they disagree."*

**Without Claude:** the script works on its own. Feed it event JSON, CSV or
SQLite and ask for a tree, a table, swimlanes, an interactive HTML page, a
calendar (`.ics`) file or a Gantt chart.

## Limits, honestly

- **Claude still does the reading.** A date the model misses is still missed.
  The self-check catches invented and misquoted events, not missing ones.
- **The evaluation is still running.** The repo includes a kit that runs real
  Claude sessions on 14 test documents with answer keys, scoring recall,
  precision and date accuracy. I'll publish the numbers when it's done.
- Hindu solar calendars (Tamil, Malayalam, Bengali) and the Chinese calendar
  aren't supported yet.

## Contribute

The project is [MIT licensed on GitHub](https://github.com/gauravnavgire/chronology).
The engine has 45 tests that check against known answers, and new calendars and
more date languages are good first issues. If you try it on a real document,
I'd love to hear what it got right and wrong.
