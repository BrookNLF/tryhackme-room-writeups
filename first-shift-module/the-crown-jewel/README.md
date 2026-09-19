# The Crown Jewel - TryHackMe Writeup

Room: https://tryhackme.com/room/thecrownjewel
Type: Splunk / PCAP analysis
Difficulty: Easy
Completed: 19th of September 2026

## Scenario

I am on a shift, looking at a new alert from Imperium Labs, a company monitored by an MSSP. The company has a global presence and keeps proprietary source code and project data on heavily secured GitLab and Jira servers.

The alert is called **Reverse Shell Outbound Connection Detected**. I have the raw PCAPs and the Splunk logs for this event. The goal is to analyse the traffic and logs, reconstruct the attack and find out how the "Crown Jewel" data was stolen.

---

## Q1. From which internal IP did the suspicious connection originate?

This question needed me to follow the tip from the room description and filter for the event called Reverse Shell Outbound Connection Detected.

```
index=network_logs log_type=ids "event.msg"="Reverse Shell Outbound Connection Detected"
```

[insert screenshot]

The source IP of the alert is in the `src_ip` field.

Answer: `10.10.10.100`

---

## Q2. What outbound connection was detected as a C2 channel? (Answer example: 1.2.3.4:9996)

The answer is visible in the Q1 screenshot. The alert event also has the `dest_ip` and `dest_port` fields. I joined the two together in the `ip:port` format. A reverse shell alert is about a machine connecting out to an attacker, so the destination is the C2 server.

Answer: `1.1.1.1:8080`

---

## Q3. Which MAC address is impersonating the gateway 10.10.10.1?

The question sounded like ARP spoofing, where an attacker's machine claims to be the gateway so traffic goes through it. I first checked which log types exist:

```
index=network_logs
| stats count by log_type
```

There was an `arp` type with 90 events.

[insert screenshot]

Then I looked at a few ARP events and expanded them to see the fields:

```
index=network_logs log_type=arp
| head 3
```

In the 2nd event, `sender_ip` was `10.10.10.1` with `sender_mac` `00:0c:29:11:22:33`. That was a bit lucky, because `head 3` only shows the first 3 events.

[insert screenshot]

To check it properly, I filtered on the gateway IP and counted the MACs:

```
index=network_logs log_type=arp event.sender_ip="10.10.10.1"
| stats count by event.sender_mac
```

I expected two MACs (the real gateway and the fake one), but only one came back, with 45 events. The event was also an unsolicited broadcast (`is-at` to `ff:ff:ff:ff:ff:ff`), which is typical for ARP spoofing.

[insert screenshot]

Answer: `00:0c:29:11:22:33`

---

## Q4. What is the non-standard User-Agent hitting the Jira instance?

I went back to the base query `index=network_logs`. In the second event I saw `host: jira`, so I used "Add to search" on it and Splunk added it to my query (with an `spath`). I then expanded a random event to find the name of the User-Agent field. It was called `agent`.

[insert screenshot]

My first idea was to make a `table` of the agents, but that would list every single event. I also wrongly assumed the field only existed in `log_type=http`. My query returned 0 results at first, because Splunk's own `host` field is `lab`, and the `host: jira` from the log is a different field. Splunk had already extracted it as `extracted_host`.

The final query groups the events by User-Agent and puts the rarest at the top:

```
index=network_logs extracted_host=jira
| stats count by event.agent
| sort count
```

Six agents came back. Five were common browsers or tools with 335-393 requests each. One had a single request and an obvious exploit name.

[insert screenshot]

Answer: `CVE-202X-EXPLOIT`

---

## Q5. How many ARP spoofing attacks were observed in the PCAP?

This was already answered in Q3. Back there I googled MAC address impersonation and found that it goes hand in hand with ARP spoofing. The `arp` log type had 90 events, so that is the number.

Answer: `90`

---

## Q6. What's the payload containing the plaintext creds found in the POST request?

Credentials in a POST request meant HTTP traffic, so I started in Splunk and expanded an event:

```
index=network_logs POST
```

It was a normal POST to `/static/js/app.js`, with no field for the request body.

[insert screenshot]

Next I grouped the POSTs by URI:

```
index=network_logs POST
| stats count by event.uri
| sort count
```

The rarest one was `/vulnerable_endpoint?cmd=RCE`, which is an exploit attempt, not the creds. `/login` looked like a good suspect, but Splunk only had the metadata for it, not the body.

I then checked if the creds were in the logs at all:

```
index=network_logs password
```

It returned 0 results, so they were only in the PCAP. I switched to Wireshark and used this filter:

```
http.request.method == "POST"
```

Only one packet came back: a POST to `/login.php`. In the packet details (and also in Follow TCP Stream), under **HTML Form URL Encoded**, I could see the username and password. The raw payload is also visible in the bytes pane.

[insert screenshot]

The destination MAC of this packet was `00:0c:29:11:22:33`, the same MAC that impersonated the gateway in Q3. So the victim sent its creds straight to the attacker.

Answer: `username=dev_user&password=SecretPassword!`

---

## Q7. What domain, owned by the attacker, was used for data exfiltration?

Data exfiltration often hides in DNS. Attackers put the stolen data into long subdomains, like `abc123.evil.com`. The `dns` log type had 1733 events, so I started there. I expanded one event and found that the queried domain is in the `query` field, so in Splunk it is `event.query`. This one was normal (`www.google.com`).

[insert screenshot]

Next I counted every queried domain and sorted from rarest to most common:

```
index=network_logs log_type=dns
| stats count by event.query
| sort count
```

There were 510 different queries. The top rows were all long, random-looking subdomains of the same domain, each seen only once. Normal domains get looked up many times, so this stood out.

[insert screenshot]

Answer: `exfil-domain.xyz`

---

## Q8. After examining the logs, which protocol was used for data exfiltration?

I found this one while solving Q7. The attacker's domain `exfil-domain.xyz` showed up in the `dns` log type, in lots of long, random subdomains that each appeared only once. That is a typical sign of DNS tunnelling, where stolen data is encoded into the domain name and sent out with normal-looking DNS queries.

[insert screenshot]

Answer: `DNS`

---

## Summary

- Before writing a new query, expand the event and read all its fields. In Q2 the answer was already in the alert from Q1.
- Filter in the query instead of scrolling through events. Using head 3 in Q3 worked only because I got lucky.
- Check the exact field names. Splunk's own host field was lab, and the host: jira from the log was a different field (extracted_host).
- stats count by <field> with sort count is a fast way to find odd things. It found the exploit User-Agent in Q4 and the exfil domain in Q7.
- Splunk had only the request metadata. The plaintext creds were only in the PCAP, so I needed Wireshark for Q6.
- Different clues can point to the same attack. The MAC from the ARP spoofing in Q3 was also the destination of the credentials in Q6.