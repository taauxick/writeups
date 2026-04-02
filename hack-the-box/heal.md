# 🩹Heal

Welcome to another fun-filled journey into the world of insecure enterprise apps. Today’s patient is “Heal” — a misconfigured resume builder So, scrub in — we’re operating without anesthesia.

## 👋 Hi, I’m Shivam (taauxick)

A curious mind with a keyboard, currently diving deep into the world of ethical hacking and CTFs. I enjoy poking around where I shouldn’t (legally, of course), and turning misconfigurations into opportunities.\
This writeup walks through my process of breaking into the “Heal” lab — from login to root — with some sarcasm, a bit of bash, and a whole lot of fun.

Check out my portfolio showcasing projects, write-ups, and my journey in cybersecurity:

{% embed url="https://taauxick.github.io/portfolio" %}

&#x20;

## &#x20;2. RustScan — Opening the Case

`rustscan -a heal.htb -r 1-65535 -- -sV -sT -A`

🩻 Output:

* **22/tcp** — SSH
* **80/tcp** — HTTP

Port 80 is always where dreams begin. And where misconfigurations live rent-free.

![](<../.gitbook/assets/11437c995d3388405a47f6b99d0d4e95 (3).png>)

## 3. Port 80 — Yet Another Login Page

A  Resume-builder-themed web app... with a login form on the main page..

> "Security starts with a login box," they said.\
> "_We don’t need brute-force protection,_" they probably also said.

We respectfully ignored it and moved to something juicier.

![](<../.gitbook/assets/264eee1af5334e613bd9957e00aadc6b (3).png>)

## 4. Subdomain Enumeration — DNS: The Gift That Keeps Giving

Fuzzed some subdomains and struck gold:

`api.heal.thm`

![](<../.gitbook/assets/ed9dbedf67ed44bf991f064e86fae054 (3).png>)

## 5. Rails + Ruby Version Disclosure — Thanks for the Patch Notes

`api.heal.thm` was leaking:

* `X-Rails-Version: 7.1.4`
* `X-Powered-By: Ruby 3.3.5`

That’s not a header. That’s an **exploit invitation**.

![](<../.gitbook/assets/30f75fe7c7356df3f00a3b38a470d087 (3).png>)

## 6. Survey Time!&#x20;

<figure><img src="../.gitbook/assets/17f687e830002a577d591ea651e210c6 (3).png" alt=""><figcaption></figcaption></figure>

`takesurvey.heal.thm` had a friendly UI, and a not-so-friendly **Export as PDF** button.

Clicked it → captured request in **Burp Suite** → found file path usage.

One thing led to another, and boom...

![](<../.gitbook/assets/0869af1484afd02eb7426be45c43682e (3).png>)

## 7. Local File Inclusion (LFI) — Cut Deeper

Tried:

`GET /download?filenaem=../../../../../../etc/passwd`

&#x20;

![](<../.gitbook/assets/c8f4d1091b37f5bbb1e3f139c9e76273 (3).png>)

✅ Success.

&#x20;Digging Rails File Structure

<figure><img src="../.gitbook/assets/dcb81db99ad6ffe2997c31845ecfeb2e (3).png" alt=""><figcaption></figcaption></figure>

Then found a SQLite database file exposed — possibly

If you listen closely, you can hear the Rails server cry.

![](<../.gitbook/assets/da6b8f981bcdec9ef1bc2da049def0bf (3).png>)

Got:

* **Username:** ralph
* **Hash:** stored in bcrypt

Dumped the hash into a `.hash` file, ran:

`hashcat -m 3200 ralph.hash /usr/share/wordlists/rockyou.txt --force`

⛏️ Result:

`ralph:147258369`

They say users are the weakest link. Turns out, so are their passwords.

![](<../.gitbook/assets/6b598c99bbb0e1e48433ef4e4e0ed11a (3).png>)

## 🧑‍💼 8. Admin Panel Access

Logged in as `ralph` — now suddenly we’re in the **Admin Panel**.

Oh, the power. Oh, the exposed forms.&#x20;

<figure><img src="../.gitbook/assets/2ae87816d11654cc637b3ba54de70ae3 (3).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/6b30a4d6ea069651d07d9bcfa5b450ad (3).png" alt=""><figcaption></figcaption></figure>

## 9. LimeSurvey RCE — GitHub Magic

Found this little gem:

{% embed url="https://github.com/N4s1rl1/Limesurvey-6.6.4-RCE" %}

Ran the exploit with our creds

![](<../.gitbook/assets/309d73280a1b089a87664b9465c1c15e (3).png>)

💥 Shell as `www-data` achieved. Patient’s heart rate dropping

![](<../.gitbook/assets/4e330f831f36a5776ce370d6a80ef91a (3).png>)

## 10. Looting Files & SSH Login to ron

Started poking around the web directory like a nosey intern.

![](<../.gitbook/assets/21c2c55a390cc8935aefdf96d97648e2 (3).png>)

A plain text password, just _lying there_. No encryption, no secrets manager, not even a `.bak` extension to pretend like it was protected.\
<br>

## Only One Patient Left — Ron

There was just **one user** we hadn’t poked yet. The mysterious, the quiet, the final boss of bad practices...

🧔 `ron`

So naturally, I did what anyone would do:

`ssh ron@heal.thm`

🩺 Using the password I found discarded like last week's prescription.

✅ Logged in.

![](<../.gitbook/assets/069d2b61a5dac23c0a244f6139189c07 (3).png>)

![](<../.gitbook/assets/1de3a2164ce60d4159e794c14fe8ff07 (3).png>)

Captured user.txt.

## 11. Internal Service Hunting with netstat

`netstat -tulnp`

Too many services to count. But one stood out like a tumor:

![](<../.gitbook/assets/8cbba359b7871daa50c6044404742d30 (3).png>)

So we did what any ethical hacker would do — **tunneled it through SSH**.

`ssh -L 8500:127.0.0.1:8500 ron@heal.thm`

![](<../.gitbook/assets/8e32c072b1f4bd53841d7794b9713293 (3).png>)

Visited: `http://localhost:8500`\
Saw Consul dashboard

![](<../.gitbook/assets/f3244a566729e306009d64e32c317246 (3).png>)

## 12. Consul Exploitation for Root — Straight from ExploitDB

Found the perfect exploit:

{% embed url="https://www.exploit-db.com/exploits/51117" %}

![](<../.gitbook/assets/a0430172c9e884d57a45edb61c39f83f (3).png>)

🐚 Got root shell.

Mission: **successfully violated HIPAA**, metaphorically speaking.

## 13. Final Thoughts — Heal Thyself

This box had more vulnerabilities than a reality TV show contestant.

From LFI to hash leaks, from lazy admin panels to Consul misconfigurations — this was a **CTF pentester’s Disneyland**.
