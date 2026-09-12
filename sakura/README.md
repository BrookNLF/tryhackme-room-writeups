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

That file path gave up the attacker's username straight away.

**Answer:** `SakuraSnowAngelAiko`

## Part 2 - Reconnaissance

Searched the username and found a Twitter account, [@SakuraLoverAiko](https://x.com/SakuraLoverAiko), which tagged someone called Aiko Abe in an early tweet - that name matched the pattern. From there, found a GitHub account, `sakurasnowangelaiko`, with a repo called PGP. Decrypted the PGP block using [cirw.in/gpg-decoder](https://cirw.in/gpg-decoder/) and got the attacker's email.

**Answers:**
- Full email: `SakuraSnowAngel83@protonmail.com`
- Real name: `Aiko Abe`

## Part 3 - Unveil

The attacker had started deleting things from GitHub, so the goal was to dig up what they were trying to hide, then use it to trace their crypto activity.

Found a repo called ETH with a file named `miningscript`. Googling the contents pointed to Ethereum mining. One of the repo's commits also contained a full stratum connection string with the wallet address baked in:

```
stratum://0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef.Aiko:pswd@eu1.ethermine.org:4444
```

Plugged the wallet address into [etherscan.io](https://etherscan.io) to pull the transaction history. The payment on 23rd of January 2021 UTC came from the Ethermine pool. Two other transactions showed 0 ETH exchanged with a Tether-labelled address.

**Answers:**
- Cryptocurrency: `Ethereum`
- Wallet address: `0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef`
- Mining pool on 23rd of January 2021 UTC: `Ethermine`
- Other cryptocurrency exchanged: `Tether (USDT)`

**Why Tether:** Tether (USDT) is a stablecoin - a cryptocurrency pegged 1:1 to the US dollar, so its value doesn't swing the way ETH's does. The two 0 ETH transactions on the wallet were interacting with a USDT contract address, which is the on-chain signature of someone moving value out of ETH and into a stablecoin. That's usually done to cash out or park funds without being exposed to crypto price volatility, which is why Tether was the second currency on the wallet.

## Part 4 - Taunt

The attacker messaged the OSINT Dojo directly on Twitter, taunting them, using the handle `SakuraLoverAiko` (found earlier, so that answered the first question here for free).

The tricky part was the BSSID for the attacker's home WiFi:

1. A tweet contained a hash. Ran it through an MD5 lookup: `b2b37b3c106eb3f86e2340a3050968e2` decoded to `8f89316d960182709dea3300e70ff11e`.
2. Another tweet, styled like a terminal (green text, black background), read: *"Not too concerned about someone else finding them on the Dark Web. Anyone who wants them will have to do a real DEEP search to find where I PASTEd them."* That pointed to a Tor-hosted paste site called DeepPaste.
3. DeepPaste no longer exists (the room dates to 2021, and I'm doing this in 2026), so the .onion lead was a dead end. Had to lean on an older writeup to bridge that gap.
4. From there, found a list of WiFi SSIDs in the leaked data. Tested them on [wigle.net](https://wigle.net) (made a free account for the advanced search), and the one that mattered was the Home WiFi, `DK1F-G`.

**Answers:**
- Current Twitter handle: `SakuraLoverAiko`
- Home WiFi BSSID: `84:AF:EC:34:FC:F8`

## Part 5 - Homebound

Last stretch: piece together the attacker's route home from photos posted on Twitter.

1. **Airport before the flight:** A tweet had a photo of a pink-blossomed tree. Reverse image search placed it at Long Bridge Park in Arlington, Virginia. The nearest airport is Washington National, code `DCA`.
2. **Last layover:** Another photo showed the JAL First Class Lounge (Sakura Lounge) - JAL is just short for Japan Airlines, not an airport code. Googling the lounge's location placed it at Haneda Airport, Tokyo, code `HND`.
3. **Lake on the final flight map:** A tweet had a screenshot of an unlabelled map. The coastline curve and a small anvil-shaped island (Sado Island) pointed to Japan. Lining the map up against that landmark put a lake in the frame: Lake Inawashiro.
4. **Home city:** With Tokyo confirmed as a layover (so it couldn't be the final stop) and the room named after sakura trees, I searched for Japanese cities known for them. That gave three candidates: Tokyo, Kyoto, and Hirosaki. Kyoto has no airport hub of its own, so a direct Washington-Kyoto route wouldn't go through Haneda the way the layover clue required. Hirosaki fit: a Washington-Hirosaki route does layover in Haneda.

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
- Reverse image search plus general geography knowledge (Geoguessr-style pattern spotting) can pin down a location even with no labels or text to go on.
- Crypto trails are followable with just a block explorer (Etherscan) and no prior crypto experience.