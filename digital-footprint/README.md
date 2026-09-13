# Digital Footprint

- **Room:** [Digital Footprint](https://tryhackme.com/room/osintchallengeiv)
- **Category:** OSINT
- **Difficulty:** Easy

## Summary

A four-task OSINT chain built around a fictional company, ACME Jet Solutions. Each task uses a different technique: reverse image searching and EXIF data, the Wayback Machine, landmark identification via Google Maps, and metadata extraction from a leaked office document.

## Task 1 - The Leaked Photo

<img width="945" height="606" alt="image" src="https://github.com/user-attachments/assets/14f6eb5d-7242-4056-ae74-30efa79d810a" />


The photo shows a house partly hidden by trees, with a fence reading "The Rectory" and a security sign showing "ADT - Armed Response" - suggesting a security company. Googling that combination pointed to Fidelity ADT, a South African home and commercial security company, which narrowed the search down to South Africa.

I also checked the photo's properties for EXIF data and found GPS coordinates in the details.

<img width="285" height="65" alt="image" src="https://github.com/user-attachments/assets/c4cb6dc8-8930-44ca-b633-da219bb855aa" />

Plugging those coordinates into a map, with help from Gemini, placed the location in central Johannesburg, near Von Wielligh Street in the business district.

<img width="845" height="291" alt="image" src="https://github.com/user-attachments/assets/f7533667-f34f-4c58-8c14-6f0c0bc0c3e7" />

**Flag:** `THM{Johannesburg}`

## Task 2 - Archived Company Website

Since the website wasn't reachable directly, I checked archive.org for a copy.

There was no page preview available, but the item's metadata included:

```
https://archive.org/details/warc-acme.com-jef
Publication date 2016
Topics warcarchives
Item Size 9.5G
Access-restricted-item true
Addeddate 2016-02-13 00:40:30
Firstfiledate 20160210224602
Identifier warc-acme.com-jef
Lastfiledate 20160212160442
Pages 183762
Scandate 20160210224602
Scanner Internet Archive Python library 0.9.8
```
<img width="945" height="481" alt="image" src="https://github.com/user-attachments/assets/9883c29a-492a-485a-87a1-cbcd83d4655e" />

The `Firstfiledate` field was the date we were looking for.

**Flag:** `THM{20160210224602}`

## Task 3 - Mysterious Landmark

<img width="945" height="1260" alt="image" src="https://github.com/user-attachments/assets/480c8c5d-4d5c-4c51-972c-aabe29100aaa" />

The first thing I did was look at the flags on the left of the picture. They read "Dublin One", and I googled that. Dublin 1, also rendered as D1 or D01, is a postal district on the northside of Dublin, Ireland. Searching "Dublin 1 landmarks" pointed to the tall structure in the middle of the picture, called The Spire.

I then went to Google Maps to check The Spire's location and see what historic landmarks sit right next to it. I found the exact spot the picture was taken from, to line up the building on the right.

<img width="945" height="444" alt="image" src="https://github.com/user-attachments/assets/0a73dc73-e93e-4ee9-a187-8ad81d1acc9d" />

On the right of the picture, above what is now a Pandora shop, a map pin read "Post Direct Mail Services". The official name for that address turned out to be the General Post Office. To confirm it, I looked up whether the General Post Office had any role in Ireland's independence - and it did.

<img width="752" height="314" alt="Zrzut ekranu 2026-09-13 133641" src="https://github.com/user-attachments/assets/fc59da46-7715-42bd-bc97-1ebc1ca0c031" />

The answer:

**Flag:** `THM{General Post Office}`

## Task 4 - Internal Documents

The leaked document was an `.odt` file containing an internal memo from "Mark" to "Robin" about system updates, mentioning an upcoming video.

<img width="504" height="223" alt="image" src="https://github.com/user-attachments/assets/37a2ebe5-5d11-46a7-9cf4-bbc36865f2d9" />

I didn't know this beforehand and had to look it up, but `.odt` files are essentially zip files and may contain more files within them. Renaming the file to `.zip` and unpacking it exposed what was hidden inside.

<img width="649" height="341" alt="image" src="https://github.com/user-attachments/assets/17b9bcfb-e890-419e-bb92-8c18a5277ffa" />

I checked all the files, and the one that caught my attention was `meta.xml`.

<img width="2559" height="371" alt="image" src="https://github.com/user-attachments/assets/2c0a18b8-2793-4821-bf78-27fa4f85b982" />

It had an Internal Username listed. Since the memo mentioned an upcoming video, I went to YouTube to check if that username was connected to any account. It was - there were no videos, but one note, which had the flag inside: `THM{Y0u_f0und_7h3_fin4l_fl4g!}`.

<img width="789" height="629" alt="Zrzut ekranu 2026-09-13 134704" src="https://github.com/user-attachments/assets/5ae7d55d-2bee-4d19-81f2-e6ce88faf0fc" />

And voila, there you have it!

**Flag:** `THM{Y0u_f0und_7h3_fin4l_fl4g!}`

## Lessons Learned

- EXIF metadata and the Wayback Machine / Archive.org are often the fastest way to verify claims a company makes about itself.
- Office file formats like `.odt` are zip containers - always worth extracting and checking `meta.xml` for leftover metadata.
- Landmark identification works best by cross-referencing multiple visual clues (flags, signage, architecture) rather than relying on one detail.
