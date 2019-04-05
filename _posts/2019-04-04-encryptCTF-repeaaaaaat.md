---
layout: post
title: repeaaaaaat - encryptCTF
author: devploit
item_image: https://i.imgur.com/RRpCsP5.png
tags: ctf encryptctf web jinja2 injection
---

### Challenge:

**Level:** Easy

**Description:**

>**repeaaaaaat**
>
>Can you repeaaaaaat?  
>http://104.154.106.182:5050  
>author: codacker

### Write-up:

We start by visiting the URL where we can see a static website with the same image repeated infinitely.

![Branching](https://i.imgur.com/sDCkxld.png)

If we check the source code we can see an interesting base64 code that is changing randomly after every website refresh.

![Octocat](https://i.imgur.com/raZODRq.png)

After many refreshes we have these base64 codes.

```
<!-- d2hhdF9hcmVfeW91X3NlYXJjaGluZ19mb3IK -->
<!-- L2xvbF9ub19vbmVfd2lsbF9zZWVfd2hhdHNfaGVyZQ== -->
<!-- Lz9zZWNyZXQ9ZmxhZw== -->
```

We try to decode them to see what each one gives us.

```
<!-- d2hhdF9hcmVfeW91X3NlYXJjaGluZ19mb3IK -->
[+] Decoded from Base64 : what_are_you_searching_for
```

We visit the website with the specified route and find another b64 code.

```
aHR0cHM6Ly93d3cueW91dHViZS5jb20vd2F0Y2g/dj01ckFPeWg3WW1FYwo=
[+] Decoded from Base64 : https://www.youtube.com/watch?v=5rAOyh7YmEc
```

It seems to be a YouTube video with nothing important... RABBIT HOLE!

Proceed with the next base64 achieved on the home page.

```
<!-- L2xvbF9ub19vbmVfd2lsbF9zZWVfd2hhdHNfaGVyZQ== -->
[+] Decoded from Base64 : /lol_no_one_will_see_whats_here
```

Once again we visit the URL and we get a new code.

```
aHR0cHM6Ly93d3cueW91dHViZS5jb20vd2F0Y2hcP3ZcPVBHakxoT2hNTFhjCg==
[+] Decoded from Base64 : https://www.youtube.com/watch\?v\=PGjLhOhMLXc
```

We get another no sense YouTube URL... ANOTHER RABBIT HOLE?! Let's proceed with the last code obtained...

```
<!-- Lz9zZWNyZXQ9ZmxhZw== -->
[+] Decoded from Base64 : /?secret=flag
```

This code does seem to have something interesting... But there's nothing special about the web code we visited... at least at first glance.

Searching and although the name and the statement do not seem to talk about injection, we start to try different types until we see that it is an **injectable Jinja2**.

Example of injection check in Jinja2:

![Branching](https://i.imgur.com/HibuAGI.png)

Knowing the way forward, exploitation is easily found with a Google search about this python template.

To see all the files that are in the directory we must use the following injection.

![Branching](https://i.imgur.com/3MjB0c9.png)

Finally, knowing the file that we want to list, we just have to do one more injection for it

![Branching](https://i.imgur.com/jDVN3Cx.png)

**Flag: encryptCTF{!nj3c7!0n5_4r3_b4D}**
