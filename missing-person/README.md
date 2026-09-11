# Missing Person - TryHackMe Writeup

- **Room:** [Missing Person](https://tryhackme.com/room/missingperson)
- **Category:** OSINT
- **Difficulty:** Easy
- **Date completed:** 11th of September 2026

## Summary

A friend went on holiday in 2025, shared a couple of photos, and then went quiet. The task was to track him down using nothing but the pictures and some OSINT skills - reverse image search, googling, and checking photo metadata.
<img width="960" height="540" alt="MotoGP" src="https://github.com/user-attachments/assets/5285e8e8-d7cf-4ab5-98c2-f8f7dcc09ec8" />
<img width="1360" height="765" alt="food" src="https://github.com/user-attachments/assets/5cc12fe0-763d-4142-92ac-4285de9a6f61" />



## Walkthrough

### 1. The circuit

Reverse image search on the first photo pointed to a race track, and the big "PETRAMINA" sign in the background confirmed it - this was the **Pertamina Mandalika International Street Circuit**.

### 2. The date of the event

Since the task mentioned "holiday in 2025", I searched for the event at that circuit in 2025. It came back as taking place from the **3rd to the 5th of October 2025**.

### 3. The restaurant

Another reverse image search, this time on a photo of the food, led me to a restaurant called **Cantina Mexicana**.

### 4. Time the photo was taken

I checked the photo's metadata. Windows Properties only showed hours and minutes, no seconds, so I ran it through an online metadata viewer (pics.io) instead. That gave the full timestamp: **19:55:30**.

### 5. The bar's address

The story continues: he went to a MotoGP afterparty with a local DJ. Knowing where the MotoGP took place, I searched for "Pertamina Mandalika International Street Circuit 2025 After Party" and found two candidates - Mandalika Beach Club and Surfers Bar (Kuta Lombok). Digging into Surfers Bar's Instagram confirmed it was the official MotoGP 2025 afterparty venue. Address: **Jl. Raya Kuta, Kuta, Kec. Pujut, Kabupaten Lombok Tengah, Nusa Tenggara Bar**.

### 6. The DJ's stage name

Surfers Bar had posted an [Instagram reel](https://www.instagram.com/reels/DOcVqWAEznb/) promoting the event, naming the DJ who played that night: **Bong Leleh**.

### 7. The cave

Searching for the DJ's stage name led to his Facebook profile ([facebook.com/bongleleh](https://www.facebook.com/bongleleh/)). The profile name itself gave it away - **Gua Sumur Lombok**, a natural cave in Indonesia he takes tourists to.

### 8. The phone number

Still on the DJ's Facebook page, the contact number for his tour business was listed: **853-3313-7345**.

And voila, there you have it!

## Answers

| # | Question | Answer |
|---|---|---|
| 1 | Circuit name | Pertamina Mandalika International Street Circuit |
| 2 | Event date | 03-05/10/2025 |
| 3 | Restaurant | Cantina Mexicana |
| 4 | Photo time | 19:55:30 |
| 5 | Bar address | Jl. Raya Kuta, Kuta, Kec. Pujut, Kabupaten Lombok Tengah, Nusa Tenggara Bar |
| 6 | DJ stage name | Bong Leleh |
| 7 | Cave | Gua Sumur Lombok |
| 8 | Tour business number | 853-3313-7345 |

## Lessons Learned

Reverse image search and basic photo metadata checks can go a very long way in OSINT - most of this room was solved just by following the visual and social media trail one clue at a time.
