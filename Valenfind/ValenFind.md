
## Overview


 The challenge consists of taking advantage of a path traversal vulnerability which results in arbitrary file read. This leads to the disclosure of application source code and credentials, unauthorized database retrieval, and account compromise.

**Platform**: TryHackMe
**Category**: Web
**Difficulty**: Medium
**Vulnerabilities**: Path Traversal / Arbitrary File Read

____


## Enumeration

We are provided with the URL where the vulnerable web app can be found: 
`http://MACHINE_IP:5000`

I began by adding the vulnerable application's IP to my attacking machine's `/etc/hosts` file as `valenfind.thm`.


### NMAP


``` Shell
nmap valenfind.thm -v -p- -sV -T4

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
5000/tcp open  upnp?


```

**Port 5000**:
* Although Nmap was unable to identify the service running on port 5000 reliably, we know that the vulnerable web app is running on it. I did some more research into what port 5000 is used for and discovered that the port is commonly used as Flask's development server port.

### Website

Navigating to `http://valenfind.thm:5000` opens a login/signup page:

![](Attachments/Pasted%20image%2020260921194621.png)

After creating an account on the page, I was greeted by a list of profiles.

![](Attachments/Pasted%20image%2020260921194640(1).png)


After opening a user profile and capturing the request with Burp Suite, I discovered that there is an API endpoint responsible for fetching user profile themes: `http://valenfind.thm:5000/api/fetch_layout?layout=`

![](Attachments/Pasted%20image%2020260921195442.png)

Submitting an empty request provided a rather verbose error message: `Error loading theme layout: [Errno 21] Is a directory: '/opt/Valenfind/templates/components/`

![](Attachments/Pasted%20image%2020260921195803.png)

I began to suspect LFI / Path traversal. To test my theory, I tried reading the contents of /etc/passwd: `http://valenfind.thm:5000/api/fetch_layout?layout=/etc/passwd`. It worked:

![](Attachments/Pasted%20image%2020260921200214.png)


____

## Gaining Access

### App.py

As I suspect the vulnerable application may use Flask, a Python web framework, I wanted to test if the vulnerable application is located in an `app.py` file.

Based on the error output, we know that the path `/opt/Valenfind` exists.  

I tried reading the contents of `/opt/Valenfind/app.py`: 
* `http://valenfind.thm:5000/api/fetch_layout?layout=/opt/Valenfind/app.py`

The output confirms the application uses Flask.

The file includes a reference to an SQL database and an API key: 

![](Attachments/Pasted%20image%2020260921200644.png)

The file also contained the API end point for fetching the database:

![](Attachments/Pasted%20image%2020260921201537.png)

### Database Exfiltration

``` Shell
curl -H "X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO" http://valenfind.thm:5000/api/admin/export_db -o valenfind.db

```

### Login Credentials

I used SQLite3 to connect to the database we just saved to the file `valenfind.db`:

```Shell
sqlite3 valenfind.db

sqlite> .tables
users

sqlite> SELECT usename, password FROM users;
romeo_montague|juliet123
casanova_official|secret123
cleopatra_queen|caesar_salad
sherlock_h|watson_is_cool
gatsby_great|green_light
jane_eyre|rochester_blind
count_dracula|sunlight_sucks
cupid|admin_root_x99
hecker|hecker123
sqlite>
```

### Flag

* I attempted to log in again with the credentials `cupid|admin_root_x99`.

![](Attachments/Pasted%20image%2020260214022842.png)
  


____

## Remediation

### Path Traversal / Arbitrary File Read

The `/api/fetch_layout` endpoint accepts user-controlled `layout` paramener and uses it to access files on the server. This allows an attacker to read files outside the intended template directory.

**Remediation Measures:**
* Use an allowlist of permitted templates.
* Avoid passing user input directly to filesystem operations.
* Canonicalize and validate file paths.
* Restrict filesystem permissions.
* Implement safe error handling.

### Sensitive Data Exposure & Database Access

The database export endpoint allowed access to a database containing user credentials when supplied with an API token.

**Remediation Measures:**
* Remove hard-coded secrets from the source code and retrieve them from a secure system.
* Rotate the exposed API token.
* Enforce authentication and authorization checks on administrative endpoints.
* Disable database export functionality in production environments if it's not required.

### Insecure Password Storage

The database contained plaintext passwords, allowing the credentials to be used directly after the database was obtained.

**Remediation measures:**
* Replace plaintext password storage with a modern password-hashing algorhitm.
* Require affected users to reset their passwords and invalidate existing sessions.
* Review authentication controls and consider MFA, particularly for administrative accounts.

____

## Key Takeaways

* A path traversal vulnerability in the theme-fetching endpoint enabled arbitrary file read, allowing access to sensitive files outside the intended template directory.

* A hard-coded API token was exposed through the file-read vulnerability and could subsequently be used to access the database export endpoint.

* The recovered credentials included apparently weak passwords, highlighting the importance of secure password storage and appropriate password-strength controls.

* The attack demonstrated how multiple vulnerabilities can be chained together: arbitrary file read exposed a secret, the secret enabled database access, and plaintext password storage increased the impact of the database disclosure.




