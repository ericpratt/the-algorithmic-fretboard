Let me start with the problem, because the problem is so ordinary it's almost invisible.

I'm a classical guitar teacher. Every week, students bring me PDFs — photocopies of photocopies, scans from a library, something a parent photographed on a kitchen table. And every week I think: *why can't I just hit play on this?* Why can't a student loop measures 24–35 at half tempo, hear the voices, and practice against it?

The answer is that the PDF is a picture. A computer sees ink, not music. Turning that ink into something playable — a format called **MusicXML**, which is what MuseScore and every other notation app speaks — is a research field called **Optical Music Recognition (OMR)**. People have been working on it for forty plus years.

So I did what anyone would do. I downloaded the open-source tool everyone uses (I believe it was homr — the name blurs; there are several), fed it a page of a Bach fugue arranged for guitar, and waited.

It was not even close. Not "a few wrong notes" not-close. *What piece is this* not-close.

I want to be fair to those tools: they're built for clean, single-line music. A Bach fugue on guitar is four voices stacked on **one staff**, with split stems, barres, string numbers, and ledger lines crawling off the bottom of the page. It is some of the most challenging music to sightread and play in the classical guitar literature; I have no doubt if computers could talk we'd hear a combination of exasperated sighs and collective groans.

---

## Chapter 1: The accident

**Wednesday, October 1.**

Here's the lovely part of this story. I didn't know anything about computer vision. I had never read a machine-learning paper. I didn't know what a "vision transformer" was, or a "token," or why anyone would care. My technical/software background consisted of some light web development, and a brief foray into competitive programming via the US Computing Olympiad (USACO) 

So I did the silliest possible experiment. I took a screenshot of ten measures of the **BWV 998 Fugue** (the Koonce edition — dense, scanned, annotated) and pasted it into a large language model with the instruction: *write me MusicXML*.

And it... did. Mostly. I imported it into MuseScore, put the guitar in my lap, and played through it against the page. Measures 1–2: perfect. Notes, rhythms, accidentals, even the left-hand fingerings. I'd just watched a chatbot do the thing a purpose-built forty-year-old field couldn't do on my music.

It was also slow — 15 to 30 minutes per 12-measure chunk — and when I pushed to the hard stuff, cracks appeared. But cracks in a thing that *works* are a completely different problem than a tool that outputs noise.

---

## Chapter 2: The idea that didn't work (and why that was the best news all week)

**Thursday and Friday, October 2–3.**

My first theory was about *coordinates*. The model gets lost on a dense page, I figured, so give it landmarks. I drew green rectangles around every measure, numbered them, and rewrote the prompt: "box 7 is measure 30, transcribe box by box."

I ran it properly this time — six transcription runs on Friday alone, logged in a table like a real experiment, because by now I'd realised that "it looked good" is not data. Same model, same twelve measures (mm. 24–35), same prompt skeleton — one run on the clean page, one on the boxed page.

| | Clean page (A) | Boxed page (B) |
|---|---|---|
| Wall time | 15 min | 20 min |
| Wrong notes | errors in m. 30 | *same* errors in m. 30, plus 1–2 more |
| Fingerings, barres, fermatas | good | good |

The boxes did nothing. Actually slightly worse, and 33% slower. The model wasn't getting lost on the page — it was getting lost on a *specific measure*, the same one both times, where interleaved sixteenth-note voices split across stems.

This is the moment the whole project turned. If the model doesn't need to be *told* where measures are, then the problem isn't navigation. The problem, I surmised: **it can't see well enough.**



![Measure boxes drawn by the first cropping script, page 5 of the Koonce BWV 998 Fugue](https://storage.ghost.io/c/74/10/741076b0-e7c9-45e7-979d-1e519df81113/content/images/2026/10/overlay_p5-2.png)
*Page 5, mm. 24–29, with the crop boxes overlaid. This is the check image the script draws so a human can confirm every box lands on a barline before any crop is cut.*

![Page 6 overlay](https://storage.ghost.io/c/74/10/741076b0-e7c9-45e7-979d-1e519df81113/content/images/2026/10/overlay_p6.png)
*Page 6, mm. 30–35. Note the little ossia staff at the top — more on that villain later.*

---

## Chapter 3: Wait, what does the model actually *see*?

**Friday night.**

This is where a person who reads papers would have known the answer already. I had to stumble into it.

When you send an image to a vision model, it doesn't look at your 300-dpi scan. It downsamples the image to a fixed pixel budget and chops it into a grid of small square **patches** (14×14 or 16×16 pixels). Each patch becomes one "token" — the same currency the model uses for words. A full letter page gets squeezed until a staff space — the gap between two lines where a notehead has to sit — might be three or four pixels tall. A sharp sign and a natural sign become the same smudge. A ledger line under the bass voice becomes a rumour.

Now the m. 30 failure made sense. It wasn't confused. It was *squinting*.

![Vision Transformer downsampling and 16x16 patch token grid comparison](https://storage.ghost.io/c/74/10/741076b0-e7c9-45e7-979d-1e519df81113/content/images/2026/10/vision_downsampling_comparison_harder.png)
*Why vision models 'squint': On a full-page scan, staff lines collapse and a single 16×16 patch token covers multiple notes at once. Cropping by measure gives the model razor-sharp resolution right where it counts.*



As luck would have it, later that night, while browsing recent papers I stumbled across: **LEGATO 2** (Yang, Zheng, Ebert & Smith, arXiv, July 2026). I had never read an arXiv paper in my life. I read this one three times.

Their first system, Legato 1, tried to do exactly what I'd started with — feed a whole page to a model and get notation out. They found it hit "severe resolution and scaling limits." In Legato 2, they changed one thing: they cut the page into **systems** (the horizontal lines of music) and fed those strips to the model one at a time, in reading order. Accuracy jumped. New state of the art.

I had independently arrived at the same diagnosis from the other direction — not from theory, but from watching a model miss a sharp in measure 14 at 150 dpi.

---

## Chapter 4: Taking it one step further than the paper

Legato 2 stopped at the system level, and for good reason — they're building a general tool for orchestral scores, piano, vocal music with lyrics. A system is the natural unit when you have lots of staves.

But I'm not building a general tool. I'm building one for *polyphonic guitar on a single staff*, and the moment I thought about it from a musician's point of view, the system felt like the wrong unit. The right unit is the **measure**. Here's why:

**1. A measure has math.** Every voice in a 4/4 measure has to add up to four beats. Every single one. That's not a heuristic — it's an invariant. A whole system has no such check (how many measures? how many notes? who knows). A measure crop gives you a free, deterministic test that catches a huge class of errors instantly: *if it doesn't sum to four, it's wrong.*

**2. A measure is the right shape.** A system strip is 8:1 or 12:1 — a long thin letterbox. Those patch grids I mentioned? Most of them land on blank margin above and below the staff. A measure crop is roughly square. Nearly every patch is spent on ink: noteheads, accidentals, fingerings.

**3. Errors stay in their lane.** These models are autoregressive — each output token conditions on everything before it. If the model misreads a voice in the first measure of a system, that confusion can smear through the whole line. Crop by measure and a mistake in measure 28 is quarantined to measure 28. You re-run one measure, not twelve.

So the architecture I landed on:

```
PDF page
  → geometry (PowerShell): find staves, find barlines, cut one measure per image, high-res
  → vision LLM: "list every note — staff position, duration, accidental — as JSON"
  → Gate 1 (code): does every voice sum to the meter?
  → Gate 2 (code): apply clef, key signature, ties → actual pitches
  → Gate 3 (code): serialize clean MusicXML
```

The LLM does the one thing it's shockingly good at — *looking* — and strict code does what it does best: arithmetic, bookkeeping, and not making things up.

I should be clear about what's real. As of today, the **cropper exists and works**. The three gates do not. The model still writes MusicXML directly from each crop, and I check it with a guitar. The gates are the next thing to build — *if* the next experiment says cropping is worth it.

Is this better than Legato 2? I have no idea yet, and I'd be lying if I claimed it. It takes the paper's central finding — *resolution via geometric cropping beats full-page processing* — and pushes it to the smallest unit that still has a mathematical check. Whether that extra step pays off is exactly what I'm testing next.

---

## Chapter 5: My turn to need glasses

**Saturday, October 4.** Two scripts, one day. The first was wrong by lunchtime, and the second exists because of it.

### Script one: `crop_measures.ps1`

The first cropper was honest about being dumb. I measured the measure boundaries by hand, typed them into a table, and the script cut one PNG per measure from the scan. Twelve measures, twelve files. Done.

Except I never looked at the crops.

I'd spent three days insisting the *computer* couldn't see, and then I shipped a coordinate table I hadn't checked with my own eyes. When I finally opened the folder: splits were off the barlines by 15 to 35 points. The fermata chord that opens m. 29 was sitting in the m. 28 file. The crop labelled m. 35 stopped at the bottom staff line and lost the entire bass voice. And page 6 has a system printed with the number "32" that actually holds *two* measures — so every file after it was named one measure wrong. I'd read that page a hundred times. I'd never noticed.

So: the computer needed higher resolution, and I needed to actually look at the thing. The fix for mine was an overlay image — the script now draws every box onto the page (those red rectangles above) and refuses to let me skip looking at it. **Never trust a crop you haven't seen.** 
![Before-and-after crops of measure 28, showing measure 29’s fermata chord mistakenly included in the earlier crop](https://storage.ghost.io/c/74/10/741076b0-e7c9-45e7-979d-1e519df81113/content/images/2026/10/chapter-5-real-crop-comparison.png)

*An earlier crop of measure 28 included the opening fermata chord of measure 29. The corrected crop stops at the barline. Never trust a crop you haven’t seen.*






### Script two: `measure_map.ps1`

A hand table works for twelve measures. It does not work for IMSLP. So the second script finds the measures itself: render the page at 300 dpi, keep only grey ink (so a teacher's red and green pen vanishes before detection even starts), track the staff lines across the page in strips so a warped book scan still resolves, find barlines as tall thin strokes with nothing hanging off them, cut.

Two details I'm proud of. The pixel loops are in C# embedded inside the PowerShell file, because PowerShell iterating over 19 million pixels takes a geological age — now it's about 8 seconds a page. And it has a regression suite: the six barlines I'd eventually hand-measured correctly from script one. The detector reproduces them within 0.2 points, 18 of 18 checks, and a sweep of all 14 pages of the Koonce book found every system and threw out the little ossia staves that would otherwise have shifted every measure number by one.

It fails in instructive ways. Heavy sixteenth-note beams can fatten a staff line until the tracker drops it. Green marker drawn *directly over* a barline kills it. And a notation-plus-tablature edition made it cheerfully reject all the real staves as "too small" and crop the TAB instead. So there's a fallback: hand-fix the JSON, skip detection, cut crops. The cropper is the product; the detector is a convenience. (Full pipeline in the appendix for the people who want it.)

---

## Chapter 6: Did it work?

**Sunday, October 5.**

Measure crops, model reading one bounded measure at a time:

- **Bach BWV 998 Fugue, mm. 24–35** — the twelve densest measures I own. One wrong note in twelve measures of four-voice counterpoint! ONE WRONG NOTE. The error: in m. 31 there's a tall square bracket (an editorial mark) sitting right before a C. The model read it as a natural sign and lowered a C♯ to C. Likely fixed in a future version by adding some sort of guitar symbol check. Difficulty unknown Everything else — the G3 tie across the 27|28 barline, the paired fermatas in m. 29, the barre markings, the circled string numbers — correct.

![The computer saw the bracket and decided it meant natural sign](https://storage.ghost.io/c/74/10/741076b0-e7c9-45e7-979d-1e519df81113/content/images/2026/10/bach-m31-bracket-small.png)


- **Aguado Study 26, all 42 measures**, from a scan with red teaching annotations left in: **zero wrong notes, 100% accurate rhythms**. Minor symbol placement issues on import. I checked it note by note with the guitar in my lap.

For calibration: the dedicated OMR tool produced unrecognozable Musicxml for similar Bach passages. 

I'll say what every result above actually is — one run, one piece, one model, judged by one guy with a guitar. It's not a benchmark. But it's the kind of result that makes you forget it's Sunday.

---

## Interlude: the time the model cheated

One story I can't leave out. On day two, I asked the model to transcribe a boxed page and build a playable version. It produced a beautiful interactive score — perfectly synced to the measure boxes. I was thrilled. However something irked me. While watchimg the LLM (Sol 6.1 I believe) work my eyes noticed a funny 'chain of thought' moving across the screen: "find a Musicxml file for Bach fugue performing search." 

Then I looked inside the file. The notes weren't from my scan at all. The model had quietly pulled a Creative Commons MIDI of the same fugue (Mutopia, Hajo Dezelski's transcription), transposed it, and timed it to my boxes. It *ignored the image it had been asked to read* and used a memorised answer.

That's not a transcription. That's a student who found the recording on YouTube. It's why the pipeline now has a rule: the model must declare its source, and anything matching a known external edition gets flagged. Trust, but diff.

---

## What's next

The honest gap in everything above: I've never run the *same passage* at full-page, system, and measure resolution side by side, from the same render, several times each, scored against a note list I've personally verified. Everything that says "crops win" is circumstantial.

So that's the next experiment. Three conditions — Page, System, Measure — three runs each, same model, same prompt, error rate counted per note. If measure beats system by a clear margin, the gates get built. If system is just as good, Legato 2 was right to stop there and I'll happily say so. If they're all the same, the bottleneck is somewhere I haven't looked yet.

On Wednesday I didn't know what a token was. By Sunday I had a barline detector scripted in powershell with a bit of c#. I still haven't finished a single machine-learning course, or read a second paper. What I had was a very specific, very hard piece of music, a guitar to check against, and the assistance of powerful large language models.

It turns out that's a surprisingly good starting kit.

---

*Eric Pratt teaches classical guitar in Santa Monica. The scripts, test logs, and crops live in a folder with far too many files named `m31.png`.*

---

### Appendix for the curious

- **Legato 2:** Yang, Zheng, Ebert, Smith — *LEGATO 2: Toward Multimodal Sheet Music Recognition and Understanding*, arXiv:2607.05769 (July 2026). Predecessor: arXiv:2506.19065.
- **Model used for transcription:** gpt-6.1-sol ("Sol 6.1").
- **Scripts:** `crop_measures.ps1` v1.1 (hand-measured table + overlays); `measure_map.ps1` v2.0-alpha (staff/barline detection, ink mask, `-MapFile` fallback). Windows PowerShell 5.1, inline C#, no installs.

**`crop_measures.ps1` — what it does**
1. Rips the embedded JPEG scans straight out of the PDF (no PDF library — it scans the bytes for JPEG start/end markers).
2. Reads a hardcoded table of measure coordinates, measured by hand.
3. Cuts one PNG per measure (`m24.png`, `m25.png`…) with a 4-point overlap at each barline so every crop includes its own barline; vertical bounds sit midway between systems.
4. Writes `coordinates.json` (every box normalized 0–1000) and `overlay_p<N>.png` check images with every box drawn on the page.

**`measure_map.ps1` — pipeline**
1. **Render** the page at 300 dpi with the Windows built-in (WinRT) PDF renderer.
2. **Ink mask:** keep pixels that are dark *and* low-saturation. Coloured pen marks drop out before detection.
3. **Track staff lines** across 12 vertical strips, so per-system skew and book-scan warp (0.37° one side, −0.05° the other on the Koonce scan) still resolve. A single global deskew failed; per-strip tracking worked.
4. **Group five equally spaced lines** into a staff. Reject small staves — the page-6 ossia has 24-px spacing against a 31-px median.
5. **Find barlines:** ink columns covering ≥92% of staff height, thin, no beam/notehead attached (stems were the only false positive in the first pass).
6. Write `measures.json`, draw overlays, cut per-measure and per-system crops.

Pixel loops are inline C# via `Add-Type`; ~8 s/page. Regression: 18/18 checks against the six hand-measured barlines, all within 0.2 pt. 14-page sweep: 97 systems found, 5 ossias rejected, measures-per-system consistent with the printed score.
- **Test passages:** Bach BWV 998 Fugue mm. 1–2 and 24–35 (Koonce 3rd ed. scan); Aguado Study 26 mm. 1–42; Coste Op. 38 No. 5 (detection only); Andrew York *Sunday Morning Overcast* (detection failure case).