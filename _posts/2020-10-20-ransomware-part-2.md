---
layout: post
title:  "Ransomware Part II"
summary: "Investigate a Ransomware executable and reverse the encryption"
author: trigleos
date: '2020-10-20 20:00:00 +0200'
category: ['Assembly','Reversing', 'Linux_Reversing','Ransomware']
tags: Windows Reversing, Reversing, C#, Ransomware
thumbnail: /assets/img/posts/ransomware-part-1/WannaCry.jpg
keywords: Ransomware, CTF, XOR encryption, Reverse Engineering, radare2
usemathjax: false
permalink: /blog/ransomware-part-2/
---
# castorsCTF2020
## TL;DR
Investigating a Ransomware attack and try to reverse the process
## Description
This is the second part of a two part series. In this part, we’ll apply the knowledge we got from analysing the executable in part 1 and reverse the encrypted image
## Recap
In part 1, we discovered that the executable contacts a server to get a seed and that that seed is then used to encrypt the flag using simple xor encryption. So let’s start by investigating the pcap that is part of the challenge
## Network analysis
We already discovered that the executable is contacting a server sitting at the IP address 192.168.0.2. The pcap file confirms this as well as the exact http call.

![http](/assets/img/posts/ransomware-part-2/packet1.png)

From the investigation of the executable, we know that the seed is only accepted if the server sends back an “ok” at the end of the exchange. Let’s see if we can find one.

![seed](/assets/img/posts/ransomware-part-2/packet2.png)

The capture contains 5 GET requests. However, only the last one is followed by POST sent from the client executable (the actual ransomware). If we take a look at that POST request, we can find the chosen seed

![seed_success](/assets/img/posts/ransomware-part-2/packet3.png)

Now that we know that the seed is 1337, we can finally start to write the programs that help us decrypt the image
## Generating the random sequence and decrypting the image
In order to get the random sequence that was used to encrypt the image, we need to write a Go program that generates it. You could easily write the entire solution in Go and let it also decrypt the file but before this challenge I had never used Go so I just limited myself to writing a function to generate and output the numbers. The last thing we need to find out is how long the image is so we know how many numbers we need to generate. A simple ls -l shows us that the image is 1441 bytes big (fairly small).With this info we can now write the Go program
```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	rand.Seed(1337) //the seed
	for i := 0; i < 1441; i++ {  //the size of the image
		fmt.Print(rand.Intn(254))  //the range of random numbers
		fmt.Print(" ")
	}	
}
```

Compiling this gives us a working program so let’s save the output to xor_key. I’ll write the main solution script in python because it’s the language I’m the most comfortable with.
```python
xor_key = open("xor_key","r")
xor_list = xor_key.readlines()[0] #read in key
xor_key.close()
xor_list = xor_list.split(" ")
with open("flag.png","rb") as encrypted:
	with open("output.png","wb") as output:
		byte = encrypted.read(1)
		index = 0
		while byte != b"":
			first = int.from_bytes(byte,"big") #transform byte of image to integer
			second = int(xor_list[index]) #take integer that was generated
			res = first ^ second #xor both
			print(res, first, second)
			output.write(res.to_bytes(1, byteorder="big")) #write result to output image
			byte = encrypted.read(1)
			index += 1
```
Running this gives us an output.png file so let’s try to open it.

![flag](/assets/img/posts/ransomware-part-2/final.png)

We managed to recover the encrypted file and defeat the ransomware. I hope you liked this detailed looked into reverse engineering. I’ll try to write some more posts about it because I’m very passionate about these types of challenges and I hope you’ll come back to read them as well.

-Trigleos

