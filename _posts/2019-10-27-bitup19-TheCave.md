---
layout: post
title: TheCave - bitup19
author: devploit
item_image: https://i.imgur.com/SiolaB9.png
tags: ctf bitup19 php rce
---

### 0x01 Challenge:

**Level:** Easy/Medium

**Description:**

>**The Cave**
>
>Perro malo!! Este año te quedas en casa!! DONDE VAS?! El perro de Bitup es muy travieso, se ha llevado la flag de este reto, y se ha ido corriendo al fondo de "la cueva". Le gusta enterrar cosas en "hoyos" donde nadie pueda verlas y luego meterse el dentro.
>Puedes tocarlo, tranqui, no hace nada. https://cave.bitupalicante.com
>Agradecimiento a JoMoZa por su colaboración con este reto.

### 0x02 Write-up:

We start by visiting the URL where we can see a static website with psychedelic background.

![Branching](https://i.imgur.com/SiolaB9.png)

If we check the source code we can see an interesting path commented.

![Branching](https://i.imgur.com/A97mhOt.png)

This new path has a series of parameters that are based on the options we click on.

![Branching](https://i.imgur.com/Dw0KkRM.png)

To correctly verify everything we can do about these parameters, we will use Burp Suite <3.

As soon as we start playing with the repeater we see that we are going to have a WAF stopping our attempts.

![Branching](https://i.imgur.com/rHQiaX6.png)

From here, I discovered two ways to solve them, depending on whether you are normal or as stubborn as me :)

### 0x02-1 Concat linux commands:

Investigating for a while we could have realized that commands can be concatenated without the WAF getting in the way we see.

![Branching](https://i.imgur.com/Qh4koKr.png)

![Branching](https://i.imgur.com/BaOJ2tL.png)

`Flag: bitup19{Unexp3ct3d_eval(uat3)}`


### 0x02-2 Bypassing WAF with XOR operations:

As I said, I am a stubborn so I focused on bypassing the WAF and getting PHP functions on the server. For this, after a few hours of attempts, I saw that I could send the functions in XOR operations using "^" operator to evade the filter and use strings that are not allowed.

Example: system(ls) --> ("zznrno"^"9EXCya"^"0FEErc")(("zd"^"qN"^"gY"))

Executing system(ls):

![Branching](https://i.imgur.com/zlzQxiS.png)

Executing our XOR bypass:

![Branching](https://i.imgur.com/H8BXZV8.png)

How PHP treats our payload:

![Branching](https://i.imgur.com/7KeH5Td.png)

--

**Why does PHP treat our payload as a string?**

The ^ is the exclusive or operator, which means that we're in reality working with binary values. So lets break down what happens.

The XOR operator on binary values will return 1 where just one of the bits were 1, otherwise it returns 0 (0^0 = 0, 0^1 = 1, 1^0 = 1, 1^1 = 0). When you use XOR on characters, you're using their ASCII values. These ASCII values are integers, so we need to convert those to binary to see what's actually going on.

```
A = 65 = 1000001
S = 83 = 1010011
B = 66 = 1000010

A       1000001
        ^
S       1010011
        ^
B       1000010
----------------
result  0010010 = 80 = P

A^S^B = P
```

If we do an 'echo "A"^"S"^"B";' PHP will return us a P as we see.

<img src="https://i.imgur.com/7IAD6ZY.png" width="250" height="100">

--

Curiously I have not found any tool to exploit this type of bypass (which I think could be useful in front of several WAFs), so I created a github repo with this tool so you can use it as you want. It is very simple but it can help in more than one situation.

[XORpass](https://github.com/devploit/XORpass)

Finally, with the use of this tool, getting the flag would be as easy as we see in the following image.

![Branching](https://i.imgur.com/XDZkcgm.png)

`Flag: bitup19{Unexp3ct3d_eval(uat3)}`


