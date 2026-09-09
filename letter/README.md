# Letter

- **Room:** Letter (TryHackMe)
- **Link:** https://tryhackme.com/room/letter
- **Category:** OSINT
- **Difficulty:** Easy
- **Date completed:** 9th of September 2026

## Summary

A pure OSINT room, no exploitation involved. You're a postal worker who finds a damaged, undeliverable letter - a torn newspaper clipping and a handwritten note. The goal is to piece together enough clues to figure out the postal code on the envelope, then dig further to identify the full name and age of the person mentioned in the note.

<img width="1021" height="397" alt="Newspaper_clipping" src="https://github.com/user-attachments/assets/d0fd6450-2bec-41a3-abe4-23d02944cdee" />
<img width="1448" height="1086" alt="letter" src="https://github.com/user-attachments/assets/240cb09b-10bf-4877-97d1-bc7126e8e3e7" />

### Contents of the letter
```
Mon cher Édouard,

Aujourd'hui, en rangeant le grenier chez mes grands-parents, je suis tombée sur cette vieille coupure de journal. Ton arrière-grand-père n'avait même pas l'âge de passer le permis quand il s'est distingué ce jour-là. Le benjamin de l'équipe, et certainement pas le moins courageux.

Il serait si fier de te voir sur l'eau à ton tour.

Avec toute mon affection,
Audette
```



## Walkthrough

### Step 1: Reading the note

The provided `Note.txt` file contained a short handwritten note in French, addressed to someone called Édouard. Translated, it read as a woman named Audette telling Édouard she'd found an old newspaper clipping while cleaning her grandparents' attic, and that his great-grandfather - too young to even have a driver's license at the time - had distinguished himself and was the youngest ("le benjamin") of his team, and certainly not the least brave.

This set up the real target: some historical event involving a very young hero, tied to Édouard's family.

### Step 2: Finding the postal code on the envelope

The envelope itself was heavily water-damaged, with the actual address illegible. In the bottom right corner though, there was a row of small orange marks - a mix of dots and vertical bars.

At first this looked like it could be binary or Morse code, but a search turned up something more specific: this is a real French postal barcode system ("code-barres postal"), historically used by La Poste's sorting machines to encode postal codes directly on envelopes.

Using the decode table from the French Wikipedia page on the topic, and reading the groups right to left, the code decoded out to:

```
29760
```

One group in the image showed 5 tall bars instead of a valid 4-bar combination - that turned out to be a distortion from the water damage, and reading it as 4 bars gave the correct result.

### Step 3: Establishing the year from the clipping

The newspaper clipping was from **L'Ouest-Éclair**, a real regional French newspaper. Most of the main headline was torn away, but the surrounding smaller headlines were still partly readable, mentioning:

- Whether Amundsen had reached the North Pole
- A statement from Herriot, referencing him as involved in French politics at the time

Both of these pointed to the year **1925**.

### Step 4: Connecting the postal code to a real event

Searching what happened in the Finistère region of France in 1925 turned up two notable events:

1. The Penmarc'h Lifeboat Disaster (23rd of May 1925) - two rescue lifeboats capsized during a storm while trying to save a fishing crew, resulting in 27 deaths.
2. The end of the Penn Sardin cannery workers' strike in Douarnenez (6th of January 1925).

Given the tone of the note - a story of youthful courage during a rescue - the Penmarc'h Lifeboat Disaster was the clear match. Looking up the postal code for Penmarc'h confirmed it: **29760**, matching the code decoded from the envelope barcode.

### Step 5: Identifying the person in the note

With the event confirmed, the note's reference to "le benjamin de l'équipe" (the youngest of the team) pointed to a specific person. Searching for the Penmarc'h Lifeboat Disaster alongside that phrase turned up the answer: **Yves-Marie Gourlaouen**, a 15-year-old cabin boy aboard the fishing boat Arche d'Alliance, who went back out into the storm with the crew to help rescue survivors and was later awarded a Silver Medal for it.

And voila, there you have it!

## Flag

```
THM{Yves-Marie_Gourlaouen_15}
```

## Lessons Learned

- Not every "code" is binary or ASCII - sometimes it's a real-world, country-specific system (like a postal barcode), and a quick search for what the marks physically look like beats trying to force-fit a generic encoding onto it.
- Damaged or unclear evidence (like the extra bar from water damage) doesn't always mean the puzzle is broken - cross-checking the result against a second, independent source (postal code lookup vs. historical event location) is what confirms you got it right.
- Piecing together a date from partial, unrelated headlines (Amundsen, Herriot) is a solid OSINT technique - historical news events can be dated even when the main headline itself is destroyed.
- Searching in the target's own language (French, in this case) surfaces far more relevant historical results than searching in English.
