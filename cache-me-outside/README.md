# Cache Me Outside

- **Room:** [Cache Me Outside](https://tryhackme.com/room/cachemeoutside)
- **Category:** OSINT
- **Difficulty:** Medium
- **Date completed:** 11th of September 2026

## Summary

A retired hacker left pieces of his identity scattered across the internet. Starting from a leaked Discord screenshot, the goal was to identify him and trace his online trail down to a real-world location and date.

## Walkthrough

### 1. Reading the Discord conversation

Two usernames showed up: WKM1137 and JJ. There was mention of a forum that got taken down. JJ was clearly the target - he said he'd been laying low, was done with "the scene", and had gotten into hiking and cycling. He mentioned using Komoot to log and plan routes, and dropped a link to his profile.

### 2. Komoot profile

The [Komoot link](https://www.komoot.com/pl-pl/user/5667624959835) led straight to a profile for **Jim Lee** - answer to question 1. His bio mentioned turning his life around, getting into running, starting his own company, and linked his GitHub: `github.com/jiml33t`.

Logging into my own Komoot account to dig further was a dead end.

### 3. GitHub -> email address

His GitHub bio named the company "Jim Lee Security Consulting". His profile repo had a README with 1 commit. Opening that commit, then adding `.patch` to the end of the commit URL, dumped the raw patch - which exposed his email in plain text: `jimleepro1@gmail.com`, answering question 2.

### 4. Instagram -> shopfront -> city

Googling the GitHub handle "jiml33t" turned up an Instagram account with no photos and no useful follows, but the bio linked a Threads account. A photo on Threads showed a shopfront reading **IRIGATII.RO**, a Romanian shop selling irrigation gear (sprinklers, pumps).

Googling that shop name led straight to their website. Since they only had a single physical location in Romania, their contact page gave a full address with no ambiguity: Calea Buziașului 13, 300701 Timișoara, Romania - answering question 3, the city.

### 5. Phone number

Emailing Jim Lee at the address found in step 3 triggered an auto-reply containing his digital business card, which included his phone number - answering question 4.

### 6. Tram station

Checking the shop's street, Calea Buziașului, on Google Maps and then searching "calea buziasului timisoara tram station" surfaced the nearest stop, which matched the mapped location: **Piața Gheorghe Domășneanu**.

## Answers

| Question | Answer |
|---|---|
| Full name | Jim Lee |
| Email address | jimleepro1@gmail.com |
| Phone number | +40 743 321 239 |
| City | Timișoara |
| Tram station | Piața Gheorghe Domășneanu |

## Lessons Learned

- A single leaked link (Komoot, in this case) can unravel an entire identity if the person reused it across other platforms.
- Git commit history is a classic way to leak a real email address, even from an otherwise "clean" GitHub profile.
- Spotting a legible shopfront sign in a photo and googling the business name can pin down a real-world location fast - no reverse image search needed.
- Sending an email to a target and getting an auto-reply is itself a valid, low-effort OSINT technique - people forget their auto-replies leak info.