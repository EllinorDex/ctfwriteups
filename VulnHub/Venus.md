## The Planets: Venus (user flag)

> Date release: 3 Jun 2021  
> Author: SirFlash  
> Series: The Planets  
> URL: [https://www.vulnhub.com/entry/the-planets-venus,705/](https://www.vulnhub.com/entry/the-planets-venus,705/)  
>  
> Difficulty: Medium
>
> Venus is a medium box requiring more knowledge than the previous box, "Mercury", in this series. There are two flags on the box: a user and root flag which include an md5 hash.

This writeup is so live I'll write all my failures and my success ! 

---------------
## User(WebApp) part:

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

![Pasted image 20210622192005](https://user-images.githubusercontent.com/84577967/127215596-7752e2db-db09-46f9-843c-cc2cbb9c5344.png)

* `Server: WSGIServer/0.2 CPython/3.9.5` => this is the webserver name 

![Pasted image 20210622192134](https://user-images.githubusercontent.com/84577967/127215599-f898ee03-4d0c-40a8-a97a-10e5014e6a95.png)

* However when we browse see, we came to this `Credentials guest:guest can be used to access the guest account.`

* In the login form let's log using the given creds ... After a successfull  that we have this sample HTML page

![Pasted image 20210622192255](https://user-images.githubusercontent.com/84577967/127215602-7d125706-e970-414b-a0f0-ded64c54764d.png)

* That has nothing interesting, I've download the image file `venus1.jpg`

![Pasted image 20210622192348](https://user-images.githubusercontent.com/84577967/127215605-f5b00a7e-8a27-43fc-bec0-11af115cd5e3.png)

* And tried some steganography tools on it, but nothing is special with it !

* Let's set our proxy on and re-login again ...

![Pasted image 20210622192624](https://user-images.githubusercontent.com/84577967/127215622-d7ee3dc9-3e9d-4205-8a69-eced7ab35047.png)

* What we notice is there a set cookie header in the response ! 

`Set-Cookie:  auth="Z3Vlc3Q6dGhyZmc="; Path=/`

* Great ! it seems a base64 encoded value let's send it to decoder

![Pasted image 20210622192705](https://user-images.githubusercontent.com/84577967/127215625-803f868c-9d7c-4bdd-be9e-0bc48db72f9f.png)

* Result: `guest:thrfg` the first part is clear, the second after the separator `:` isn't clear !  I assume it's a rot13 (we can confirm that using cyberchef or any other online websites) and the guess was correct the *rot13'ed* value is `guest`.

*  Anyway I tried to manipulate the cookies value (SQLInjection, Command Injection ... ) nothing is suspicious and nothing has worked !

*  Where is the bug ? The bug is if we remove the password value and the separator `:` its still working ! so the password isn't checked by the WebApp. 

![Pasted image 20210622193200](https://user-images.githubusercontent.com/84577967/127215628-698ee80c-b54f-48c6-bd3f-6f450ccf1ee1.png)

* And also if we send only the base64(user) value, the `auth` value will return in response both of `base64(user:rot13(pass))`.

* Great ! , We can now retrieve the password of any valid user !

 * Time to use some fuzzing and enumerating for other usernames using the login form and check if it says "Invalid password" and not "Invalid Username" .

```bash
wfuzz -c -u "http://192.168.1.37:8080/" -d "username=FUZZ&password=elonmusk" -w WordList/raft-large-words.txt --ss "Invalid password."
```

![Pasted image 20210622193332](https://user-images.githubusercontent.com/84577967/127215630-d482ba31-65db-454d-813c-ee6639ae322d.png)

* After it finished ! we got 3 valid users:

```bash
guest
venus
magellan
```

* So let's try to get the password of venus and magellan !

![Pasted image 20210622193516](https://user-images.githubusercontent.com/84577967/127215632-37832465-61cf-4599-a2ec-c7dda29ca378.png)

![Pasted image 20210622193537](https://user-images.githubusercontent.com/84577967/127215635-9e27f691-fe3a-4319-b649-9320e6df4ff4.png)

![Pasted image 20210622193611](https://user-images.githubusercontent.com/84577967/127215637-d38465db-623c-4499-a622-ae35a774a4fd.png)

![Pasted image 20210622194359](https://user-images.githubusercontent.com/84577967/127215640-040fc0b3-ed50-48ec-a208-9f48e52efb7e.png)

* The password for `magellan` is: `venusiangeology1989`

* The password for `venus` is: `venus`

* I tried the credentials on ssh and I was able to login with `magellan` . 

![Pasted image 20210622194522](https://user-images.githubusercontent.com/84577967/127215641-3a3563e3-13a8-4d4d-84cd-96ac900befc7.png)

* The user flag:

![Pasted image 20210622194552](https://user-images.githubusercontent.com/84577967/127215644-4994480c-6e71-4d9f-86db-2eb65818c20b.png)

----------------------------------------------

## Root (Privilege Escalation) Part:

* The root flag:

After some enumeration I came to this:

*Active Ports:*


![Pasted image 20210727202655](https://user-images.githubusercontent.com/84577967/127215674-6b40059e-d40d-4890-adf3-058eecaddad3.png)

![Pasted image 20210622200344](https://user-images.githubusercontent.com/84577967/127215653-defc2c6f-8019-452a-b8b2-9dc7525a96b5.png)

 * We have an expected port wich is 9080, let's dig more ...

![Pasted image 20210727202628](https://user-images.githubusercontent.com/84577967/127215670-97e4a79b-de27-45ac-ac94-9e4a09140acf.png)

* So this is a kinda of service executed by root !

 * We can confirm by looking at the process list.

![Pasted image 20210727202501](https://user-images.githubusercontent.com/84577967/127215661-8e8aabcd-e9cd-44ab-abdc-1e4687f774aa.png)

* Information about the binary file:

![Pasted image 20210727202604](https://user-images.githubusercontent.com/84577967/127215665-f5b39a4f-7199-4a09-bdf6-5de07a962203.png)

* The binary exploitation was done by datajerk.
* The full story and writeup => [Binary Exploitation writeup](https://github.com/datajerk/ctf-write-ups/tree/master/vulnhub/venus).

Hope you enjoyed the writeup !
