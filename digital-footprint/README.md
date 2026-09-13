# Digital Footprint

- **Room:** [Digital Footprint](https://tryhackme.com/room/osintchallengeiv)
- **Category:** OSINT
- **Difficulty:** Easy

## Summary

A four-task OSINT chain built around a fictional company, ACME Jet Solutions. Each task uses a different technique: reverse image searching and EXIF data, the Archive.org, landmark identification via Google Maps, and metadata extraction from a leaked office document.

## Task 1 - The Leaked Photo

The photo shows a house partly hidden by trees, with a fence reading "The Rectory" and a security sign showing "ADT - Armed Response". Googling that combination pointed to Fidelity ADT, a South African home and commercial security company - narrowing the search to South Africa.

Checking the photo's properties revealed EXIF data with GPS coordinates. Plugging those into a map (with help from Gemini) placed the location in central Johannesburg, near Von Wielligh Street in the business district.

**Flag:** `THM{Johannesburg}`

## Task 2 - Archived Company Website

ACME Jet Solutions claims on social media to have been founded in 2025. To check that, I looked the site up on archive.org.

There was no page preview available, but the item's metadata included:

```
Publication date: 2016
Firstfiledate: 20160210224602
Lastfiledate: 20160212160442
```

The `Firstfiledate` field is the first time the site was archived - proving it existed years before the claimed 2025 founding date.

**Flag:** `THM{20160210224602}`

## Task 3 - Mysterious Landmark

Flags visible on the left of the image read "Dublin One". Dublin 1 (D1/D01) is a postal district on Dublin's northside, and searching "Dublin 1 landmarks" pointed to the tall spike-shaped monument in the middle of the photo: The Spire.

Using Google Maps to find The Spire's exact position, I lined up the view to match the building on the right of the image, above a Pandora shop. The pin there was labeled "Post Direct Mail Services", which corresponds to the General Post Office - a building with a well-documented role in Ireland's fight for independence.

**Flag:** `THM{General Post Office}`

## Task 4 - Internal Documents

The task provided a leaked `.odt` file - an internal memo from "Mark" to "Robin" about system updates, mentioning an upcoming video.

An `.odt` file is really just a zip archive. Renaming it to `.zip` and extracting it exposed the internal file structure, including `meta.xml`, which listed an internal username.

Since the memo mentioned a video, I searched that username on YouTube. No videos were posted, but there was a single note containing the flag.

**Flag:** `THM{Y0u_f0und_7h3_fin4l_fl4g!}`

## Lessons Learned

- EXIF metadata and the Wayback Machine are often the fastest way to verify claims a company makes about itself.
- Office file formats like `.odt` are zip containers - always worth extracting and checking `meta.xml` for leftover metadata.
- Landmark identification works best by cross-referencing multiple visual clues (flags, signage, architecture) rather than relying on one detail.