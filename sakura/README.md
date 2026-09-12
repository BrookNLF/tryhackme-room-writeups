# Sakura - TryHackMe OSINT Room Writeup

- **Room:** [Sakura](https://tryhackme.com/room/sakura)
- **Category:** OSINT
- **Difficulty:** Easy
- **Date completed:** 12th of September 2026

## Summary

The OSINT Dojo got hit by a cyberattack. No real damage was done, but the attacker left an SVG image behind. The whole room is about following the trail from that one image, through social media, GitHub, a crypto wallet, and finally a set of travel photos, to figure out who the attacker is and where they are headed.

## Part 1 - Tip-Off

Pulled up the SVG's page source instead of waiting on an EXIF tool, and found this in the Inkscape metadata:

```
inkscape:export-filename="/home/SakuraSnowAngelAiko/Desktop/pwnedletter.png"
```

<p align="center">
  <img width="659" height="1014" alt="image" src="https://github.com/user-attachments/assets/7124e96e-06b8-4283-9319-1133d6023aeb" />
</p>

That file path gave up the attacker's username straight away.

**Answer:** `SakuraSnowAngelAiko`

## Part 2 - Reconnaissance

Searched the username and found a Twitter account, [@SakuraLoverAiko](https://x.com/SakuraLoverAiko), which tagged someone called Aiko Abe in an early tweet - that name matched the pattern. From there, found a GitHub account, `sakurasnowangelaiko`, with a repo called PGP. Decrypted the PGP block using [cirw.in/gpg-decoder](https://cirw.in/gpg-decoder/) and got the attacker's email.

<p align="center">
  <img width="45%" alt="image" src="https://github.com/user-attachments/assets/e9b767cd-0ab9-4b1a-b42d-453cb8b53cde" />
  <img width="45%" alt="image" src="https://github.com/user-attachments/assets/b287c491-1d6e-4a4f-b95f-c1b09f03104a" />
</p>

**Answers:**
- Full email: `SakuraSnowAngel83@protonmail.com`
- Real name: `Aiko Abe`

## Part 3 - Unveil

The attacker had started deleting things from GitHub, so the goal was to dig up what they were trying to hide, then use it to trace their crypto activity.

Found a repo called ETH with a file named `miningscript`. Googling the contents pointed to Ethereum mining. One of the repo's commits also contained a full stratum connection string with the wallet address baked in:

```
stratum://0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef.Aiko:pswd@eu1.ethermine.org:4444
```

I'd never touched cryptocurrency before this room, so I copy-pasted the wallet address straight into Google and stumbled onto [etherscan.io](https://etherscan.io), a block explorer that showed the full transaction history for that address - no prior crypto knowledge needed to figure that part out. The payment on 23rd of January 2021 UTC came from the Ethermine pool. Two other transactions showed 0 ETH exchanged with a Tether-labelled address, which is what led to the next question.

<p align="center">
  <img width="945" height="1069" alt="image" src="https://github.com/user-attachments/assets/5a02b6fb-e24f-4dda-a3f2-e518de47ca9b" />
</p>

**Answers:**
- Cryptocurrency: `Ethereum`
- Wallet address: `0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef`
- Mining pool on 23rd of January 2021 UTC: `Ethermine`
- Other cryptocurrency exchanged: `Tether (USDT)`

**Why Tether:** since I had no cryptocurrency background going in, I googled what Tether/USDT actually was rather than guessing. Tether (USDT) is a stablecoin - a cryptocurrency pegged 1:1 to the US dollar, so its value doesn't swing the way ETH's does. The two 0 ETH transactions on the wallet were interacting with a USDT contract address, which is the on-chain signature of someone moving value out of ETH and into a stablecoin. That's usually done to cash out or park funds without being exposed to crypto price volatility, which is why Tether was the second currency on the wallet.

## Part 4 - Taunt

The attacker messaged the OSINT Dojo directly on Twitter, taunting them, using the handle `SakuraLoverAiko` (found earlier, so that answered the first question here for free).

<p align="center">
  <img width="444" height="309" alt="image" src="https://github.com/user-attachments/assets/83194fa6-7e52-4a50-aac4-c64bb5e27b7c" />
</p>

The tricky part was the BSSID for the attacker's home WiFi:

1. A tweet contained a hash. Decoding an MD5 hash was the first thing that came to mind when I saw it, so I ran it through a lookup anyway: `b2b37b3c106eb3f86e2340a3050968e2` decoded to `8f89316d960182709dea3300e70ff11e`. In the end this turned out to be unnecessary - it didn't actually help find the BSSID.
2. Another tweet, styled like a terminal (green text, black background), read: *"Not too concerned about someone else finding them on the Dark Web. Anyone who wants them will have to do a real DEEP search to find where I PASTEd them."* That pointed to a Tor-hosted paste site called DeepPaste.
3. DeepPaste no longer exists (the room dates to 2021, and I'm doing this in 2026), so I couldn't visit the .onion site myself to grab the paste, and Wayback Machine doesn't work on the Dark Web. I had to find someone else's writeup of this same CTF room and use it to get the body of the paste the tweet was pointing to.

<p align="center">
  <img width="945" height="536" alt="image" src="https://github.com/user-attachments/assets/344934ff-5497-4e29-bbce-1b217131becb" />
</p>

4. From there, found a list of WiFi SSIDs in the leaked data. Tested them on [wigle.net](https://wigle.net) (made a free account for the advanced search), and the one that mattered was the Home WiFi, `DK1F-G`.

<p align="center">
  <img width="945" height="272" alt="image" src="https://github.com/user-attachments/assets/184dc48d-4d31-4307-9c2f-7874c01b1058" />
</p>

**Answers:**
- Current Twitter handle: `SakuraLoverAiko`
- Home WiFi BSSID: `84:AF:EC:34:FC:F8`

## Part 5 - Homebound

Last stretch: piece together the attacker's route home from photos posted on Twitter.

1. **Airport before the flight:** A tweet had a photo of a pink-blossomed tree. Reverse image search placed it at Long Bridge Park in Arlington, Virginia. The nearest airport is Washington National, code `DCA`.

<p align="center">
  <img width="945" height="709" alt="image" src="https://github.com/user-attachments/assets/25ed46d4-3ee0-4dc6-be97-6a1a8761d0ba" />
</p>

2. **Last layover:** Another photo showed the JAL First Class Lounge (Sakura Lounge) - JAL is just short for Japan Airlines, not an airport code. Googling the lounge's location placed it at Haneda Airport, Tokyo, code `HND`.

<p align="center">
  <img width="693" height="816" alt="image" src="https://github.com/user-attachments/assets/1ccd40f6-5096-4b08-a1f1-333a0d94adbc" />
</p>

3. **Lake on the final flight map:** A tweet had a screenshot of an unlabelled Google Maps view. My past experience playing Geoguessr came in handy here - I noticed the coastline curving to the left, which is a shape typical of Japan (and the Haneda layover already pointed that way too). Scanning through Japanese coastline shapes, I spotted a small anvil-shaped island on the left, which turned out to be Sado Island. Lining the map up against that landmark put a lake in the frame: `Lake Inawashiro`.

<p align="center">
  <img width="45%" alt="image" src="https://github.com/user-attachments/assets/557db39f-2abe-4bc3-be54-d54abea9a7c5" />
  <img width="45%" alt="image" src="https://github.com/user-attachments/assets/b837e930-211f-430d-8e0b-cfb07cb472d2" />
</p>

4. **Home city:** With Tokyo confirmed as a layover (so it couldn't be the final stop) and the room named after sakura trees, I searched for Japanese cities known for them. That gave three candidates: Tokyo, Kyoto, and Hirosaki.

<p align="center">
  <img width="945" height="840" alt="image" src="https://github.com/user-attachments/assets/a2de58e1-806e-4967-bf7e-d1668c726371" />
</p>

Kyoto has no airport hub of its own, so a direct Washington-Kyoto route wouldn't go through Haneda the way the layover clue required.

<p align="center">
  <img width="945" height="491" alt="image" src="https://github.com/user-attachments/assets/23379e0c-3d63-4500-82b5-1338e0139c1f" />
</p>

`Hirosaki` fit: a Washington-Hirosaki route does layover in Haneda.

<p align="center">
  <img width="945" height="851" alt="image" src="https://github.com/user-attachments/assets/8489ed05-8440-4ac9-a92d-6eda220d795c" />
</p>

**Answers:**
- Airport before flight: `DCA` (Washington National)
- Last layover: `HND` (Haneda)
- Lake: `Lake Inawashiro`
- Home city: `Hirosaki`

And voila, there you have it!

## Lessons Learned

- Right-clicking to check page source can beat waiting on a dedicated metadata tool, especially for SVGs.
- Deleted content isn't always gone - GitHub commit history can hold onto things the user thought they removed.
- Not every lead survives five years. When a site (like DeepPaste) has gone offline, it's fine to lean on other writeups to bridge that one gap rather than getting stuck.
- Wayback Machine is no use for .onion sites - it only crawls the regular web, so a dead Tor hidden service like DeepPaste can't be recovered through it at all.
- Reverse image search plus general geography knowledge (Geoguessr-style pattern spotting) can pin down a location even with no labels or text to go on.
- Crypto trails are followable with just a block explorer (Etherscan) and no prior crypto experience.
