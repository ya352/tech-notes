# Images in a .pptx get scrambled on iPhone while desktop is perfect — and why the old .ppt fixes it

A real-world finding: a programmatically generated PowerPoint looked flawless in desktop PowerPoint, but turned into a mess when opened on an iPhone — and the solution that finally worked after a long chain of almost-fixes.

## TL;DR

A multi-image .pptx (OOXML) generated programmatically (in our case with python-pptx) renders perfectly on desktop, but iOS QuickLook — the system preview engine used by Telegram, WhatsApp, Files and Mail on iPhone — non-deterministically shows the wrong photos: images duplicate and "jump" between slides on every open/scroll.

This is not file corruption and not a structural error: the file is valid, desktop and LibreOffice read it correctly. It is a bug/limitation of the iOS QuickLook renderer on slides with multiple images.

No in-.pptx trick fixed it reliably (globally unique shape ids, globally unique relationship ids, dropping srcRect cropping, separate media parts per image, merging photos into a single image).

The fix that worked: save/convert to the binary .ppt (PowerPoint 97-2003) format. There images are embedded directly in the stream, with no relationship resolution — it renders consistently everywhere, while staying editable and keeping hyperlinks.

Important caveat: do the conversion with headless LibreOffice (`soffice --convert-to ppt`), and the LibreOffice version matters — an old one (7.4) produces a .ppt with broken text on iOS; use a recent build (≥ 25.2).

## Context

We programmatically build a casting deck: one slide per role, up to 4 "cards" per slide, each card containing a person's photo plus text (name, measurements, a clickable link, a note). 16:9, generated with python-pptx. On desktop (PowerPoint, Keynote, LibreOffice) it was always flawless.

## Symptom

Open the file on an iPhone (typically forwarded via Telegram/WhatsApp, or opened from Files), and:

- photos end up on the wrong people;
- the same photo gets duplicated across cards/slides;
- re-opening or scrolling changes the arrangement randomly (non-deterministic);
- the text (names, captions) stays in place — it's specifically the images that move.

All of these iOS apps use the same system preview engine — QuickLook — which is why the bug reproduces identically in Telegram, WhatsApp, Files and Mail.

## What we tried — and why it did NOT help

Every hypothesis was "structural" (something must be off in the OOXML). Each one seemed reasonable, and we implemented and verified each in the file:

1. **Globally unique shape ids (p:cNvPr/@id).** python-pptx renumbers ids per slide, so shapes on different slides share ids. Hypothesis: QuickLook treats ids globally and confuses shapes. We made ids unique across the whole file. → Still fine on desktop, slightly better in iOS Files, but did not cure it.

2. **Globally unique relationship ids (rId).** Per spec, rId is local to its part (slideN.xml.rels), and repeating rId2 across slides is valid. But some mobile readers seem to keep a "global" table. We renumbered rIds with a running counter (in sync across .rels and the r:embed references). → Did not cure it.

3. **Dropped srcRect cropping.** iOS QuickLook ignored srcRect (the image got stretched). We started physically pre-cropping images and inserting them without srcRect. → Stretching gone, but the scrambling remained.

4. **A separate media part per image (disabled dedup).** python-pptx collapses identical images into a single part. We made every blob byte-unique so each picture had its own part. → Did not cure it.

5. **Merged the photo row into ONE image per slide** (keeping the text as separate, editable objects). Logic: with a single `<p:pic>` per slide there is nothing to scramble. → Within a slide, true — but images kept jumping between slides. This is what finally killed the hypothesis: it's not about the number of images.

The conclusion from the whole series: the file is valid, desktop is flawless, yet iOS QuickLook still non-deterministically scrambles the images. So the problem is in the renderer, not in our markup.

## The test that pointed at the root cause

The decisive test: open the file in desktop PowerPoint and "Save As…" again.

- Re-saving to modern .pptx — still broke on iOS.
- Re-saving to .ppt (PowerPoint 97-2003) — opened correctly on iOS.

The difference is not OOXML "cleanliness" — it's the format itself.

## Root cause

.pptx is a ZIP container (OOXML) where every image lives as a separate part and is bound to a shape via a relationship (a:blip r:embed="rId…", resolved through slideN.xml.rels). It appears that iOS QuickLook, on slides with multiple images, resolves those relationships unreliably (a race / wrong key), so images get matched to the wrong shapes — non-deterministically.

The binary .ppt (PowerPoint 97-2003) is an OLE document where images are embedded directly in the stream, with no separate parts and no relationship resolution. There is simply nothing to scramble → stable rendering across all viewers.

## The solution

Generate as usual (we like python-pptx, and the cards stay editable), then convert the output to .ppt with headless LibreOffice:

```
soffice --headless --convert-to ppt --outdir /tmp deck.pptx
```

Result:

- renders correctly on iPhone (Telegram/WhatsApp/Files/Mail) — photos in place;
- the file is editable in PowerPoint/Keynote;
- hyperlinks are preserved (verified — clicking a link inside a card works);
- the .ppt is roughly 1.5× larger than the .pptx (images stored inline) — acceptable.

## The pitfall: LibreOffice version

The conversion is sensitive to the converter version:

- old LibreOffice 7.4 (the default in Debian 12 apt) produced a .ppt whose text broke on iOS (names wrapped mid-word, captions overlapped), even though the photos were already correct;
- LibreOffice ≥ 25.2 produces a fully correct .ppt.

If you install the converter on a server, use a recent build. The official standalone tarball installs without root: extract the .debs with `dpkg-deb -x` into your home directory and point your pipeline at that soffice.

## Takeaways

- If a programmatically generated PowerPoint will travel to phones, don't trust .pptx in mobile previews, especially on slides with multiple images.
- For sending/viewing, the most reliable choice is the binary .ppt (or PDF, if you don't need editability).
- "Structural" OOXML tweaks (unique ids/rIds, dropping srcRect, etc.) do not fix the renderer bug — it's wasted effort.
- When converting via LibreOffice, use a recent version.

If you've hit the same case, or have a better explanation of iOS QuickLook's behavior, I'd love to compare notes.

