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
<img width="945" height="610" alt="image" src="https://github.com/user-attachments/assets/ff7b4954-e77f-4982-bfa2-066b278a9215" />


### 2. Komoot profile

The [Komoot link](https://www.komoot.com/pl-pl/user/5667624959835) led straight to a profile for **Jim Lee** - answer to question 1. His bio mentioned turning his life around, getting into running, starting his own company, and linked his GitHub: `github.com/jiml33t`.

Logging into my own Komoot account to dig further was a dead end.

At this point I noted he'd mentioned his own company - figured that might be worth connecting to a company name later, but decided to focus on the GitHub link first since it was the most direct lead.

### 3. GitHub -> email address

His GitHub bio named the company "Jim Lee Security Consulting" - that confirmed the company lead from the Komoot bio. His profile repo had a README with 1 commit. From my recruiter days I remembered a trick for pulling the email address behind a commit: I opened the commit, clicked "View Commit Details", then added `.patch` to the end of the URL to get the raw patch source. That dumped his email in plain text: `jimleepro1@gmail.com`, answering question 2.

### 4. Instagram - a hard one

The phone number gave me the most trouble, so I went back to basic googling. Searching "jiml33t" turned up an Instagram account - no pictures, not tagged in anything, and not following anyone useful. But the bio linked a Threads account, so I followed that next.

### 5. Threads -> shopfront -> city

A photo on his Threads showed a shopfront reading **IRIGATII.RO**, a Romanian shop selling irrigation gear (sprinklers, pumps).
<img width="945" height="749" alt="image" src="https://github.com/user-attachments/assets/12ccb513-ce20-4f1b-a764-ab33cede235f" />


Googling that shop name led straight to their website. Since they only had a single physical location in Romania, their contact page gave a full address with no ambiguity: Calea Buziașului 13, 300701 Timișoara, Romania - answering question 3, the city - Timișoara.

### 6. Phone number - going back to the start

With the phone number still unsolved, I circled back to the "Jim Lee Security Consulting" company name, but nothing new turned up there. So I decided to try my luck and just emailed Jim Lee directly at the address from step 3, with the email only saying `test`. It paid off - I got an automatic reply back with his digital business card attached, which had his phone number right on it, answering question 4.
<img width="661" height="316" alt="image" src="https://github.com/user-attachments/assets/7e5cfdff-97a1-4a9b-86ba-ff7b12185c79" />


### 7. Tram station

Checking the shop's street, Calea Buziașului, on Google Maps and then searching "calea buziasului timisoara tram station" surfaced the nearest stop, which matched the mapped location: **Piața Gheorghe Domășneanu**.
<img width="945" height="860" alt="image" src="https://github.com/user-attachments/assets/648eea3d-5f8c-43db-8531-2a77bc8ca0b4" />


And voila, there you have it - Jim Lee, found.

## Answers

| Question | Answer |
|---|---|
| Full name | Jim Lee |
| Email address | jimleepro1@gmail.com |
| Phone number | +40 743 321 239 |
| City | Timișoara, Romania |
| Tram station | Piața Gheorghe Domășneanu |

## Lessons Learned

- A single leaked link (Komoot, in this case) can unravel an entire identity if the person reused it across other platforms.
- Git commit history is a classic way to leak a real email address, even from an otherwise "clean" GitHub profile.
- Spotting a legible shopfront sign in a photo and googling the business name can pin down a real-world location fast - no reverse image search needed.
- Sending an email to a target and getting an auto-reply is itself a valid, low-effort OSINT technique - people forget their auto-replies leak info.
