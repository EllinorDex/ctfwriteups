This writeup is so live I'll write all my failures and my success ! And I'll try to be clear in all the steps ...

---------------
## Enumeration part:

* Nmap scan log:

```bash
❯ sudo nmap -sC -sV -p- -r 192.168.1.37 -vvv
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-22 18:58 CEST
Reason: 65410 no-responses and 123 admin-prohibiteds
PORT     STATE SERVICE    REASON         VERSION
22/tcp   open  ssh        syn-ack ttl 63 OpenSSH 8.5 (protocol 2.0)
| ssh-hostkey:
|   256 b0:3e:1c:68:4a:31:32:77:53:e3:10:89:d6:29:78:50 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBB+dV9A80/dgYSig2NEBJYcoRe6VFus7DqjGWjNYjN4FH4e8scrM8P9zuw8EYJTdIjDVeJbersbscUbJTTH3C+w=
|   256 fd:b4:20:d0:d8:da:02:67:a4:a5:48:f3:46:e2:b9:0f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEG7ONqJEC7HEEiTZaI+MemunphhJ23BhWM0eLlcL/BJ
8080/tcp open  http-proxy syn-ack ttl 63 WSGIServer/0.2 CPython/3.9.5
| fingerprint-strings:
|   GetRequest, HTTPOptions:
|     HTTP/1.1 200 OK
|     Date: Mon, 21 Jun 2021 05:05:21 GMT
|     Server: WSGIServer/0.2 CPython/3.9.5
|     Content-Type: text/html; charset=utf-8
|     X-Frame-Options: DENY
|     Content-Length: 626
|     X-Content-Type-Options: nosniff
|     Referrer-Policy: same-origin
|     <html>
|     <head>
|     <title>Venus Monitoring Login</title>
|     <style>
|     .aligncenter {
|     text-align: center;
|     label {
|     display:block;
|     position:relative;
|     </style>
|     </head>
|     <body>
|     <h1> Venus Monitoring Login </h1>
|     <h2>Please login: </h2>
|     Credentials guest:guest can be used to access the guest account.
|     <form action="/" method="post">
|     <label for="username">Username:</label>
|     <input id="username" type="text" name="username">
|     <label for="password">Password:</label>
|     <input id="username" type="text" name="password">
|     <input type="submit" value="Login">
|     </form>
|     </body>
|_    </html>
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: WSGIServer/0.2 CPython/3.9.5
|_http-title: Venus Monitoring Login

Nmap done: 1 IP address (1 host up) scanned in 243.85 seconds
```

* We have only 2 opened ports (22 for ssh) and (a webserver on port 8080)

![[Pasted image 20210622192005.png]]

* `Server: WSGIServer/0.2 CPython/3.9.5` => this is the webserver name 

![[Pasted image 20210622192134.png]]

* However when we browse see, we came to this `Credentials guest:guest can be used to access the guest account.`

* And also a login form, so I had to try logging in using the creds above ... After that we have this sample HTML page

![[Pasted image 20210622192255.png]]

* That has nothing interesting, I've download the image file `venus1.jpg`

![[Pasted image 20210622192348.png]]

* And tried some steganography tools on it, but nothing is special with it !

![[Pasted image 20210622192516.png]]

* Let's set our proxy on and re-login again ...

![[Pasted image 20210622192624.png]]

* What we notice is there a set cookie header in the response ! 

`Set-Cookie:  auth="Z3Vlc3Q6dGhyZmc="; Path=/`

* Great ! it seems a base64 encoded value let's send it to decoder

![[Pasted image 20210622192705.png]]

* Result: `guest:thrfg` the first part is clear, the second after the separator `:` isn't clear !  I assume it's a rot13 (we can confirm that using cyberchef or any other online websites) and the guess was correct the *rot13'ed* value is `guest`.

*  Anyway I tried to manipulate the cookies value (SQLInjection, Command Injection ... ) nothing is suspicious and nothing has worked !

*  Where is the bug ? The bug is if we remove the password value and the separator `:` its still working ! so the password isn't checked by the WebApp. 

![[Pasted image 20210622193200.png]]

* And also if we send only the base64(user) value, the `auth` value will return in response both of `base64(user:rot13(pass))`.

* Great ! , We can now retrieve the password of any valid user !

 * Time to use some fuzzing and enumerating for other usernames using the login form and check if it says "Invalid password" and not "Invalid Username" .

```bash
wfuzz -c -u "http://192.168.1.37:8080/" -d "username=FUZZ&password=elonmusk" -w WordList/raft-large-words.txt --ss "Invalid password."
```


![[Pasted image 20210622193332.png]]

* After it finished ! we got 3 valid users:

```bash
guest
venus
magellan
```

* So let's try to get the password of venus and magellan !

![[Pasted image 20210622193516.png]]

![[Pasted image 20210622193537.png]]

![[Pasted image 20210622193611.png]]

![[Pasted image 20210622194359.png]]

* The password for `magellan` is: `venusiangeology1989`

* The password for `venus` is: `venus`
----------------------------------------------

## Privilege Escalation Part:
* I tried the credentials on ssh and I was able to login with `magellan` . 

![[Pasted image 20210622194522.png]]

* The user flag:

![[Pasted image 20210622194552.png]]

* The root flag:

After some enumeration I came to this:

*Active Ports:*

![[Pasted image 20210622200344.png]]

 * We have an expected port wich is 9080, let's dig more ...

![[Pasted image 20210727202628.png]]

* So this is a kinda of service executed by root !

 * We can confirm by looking at the process list.

![[Pasted image 20210727202501.png]]

* Information about the binary file:

![[Pasted image 20210727202604.png]]

* I'll leave you with this writeup from a friend that details the root part from the binary exploitation to the flag => [Binary Exploitation WriteUP](https://github.com/datajerk/ctf-write-ups/tree/master/vulnhub/venus).
*  By datajerk.

Hope you enjoyed the writeup !
