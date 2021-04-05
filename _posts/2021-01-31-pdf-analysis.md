---
layout: post
title:  "PDF analysis"
summary: "Analyze broken pdf file and extract several hints that lead to the solution"
author: trigleos
date: '2021-01-31 20:00:00 +0200'
category: ['Forensics']
tags: PDF, Forensics
thumbnail: /assets/img/posts/pdf-analysis/pdf.png
keywords: PDF, Forensics, CTF
usemathjax: false
permalink: /blog/pdf-analysis/
---
# PDF is broken and so is this file

## TL;DR
Analyze broken pdf file and extract several hints that lead to the solution

## Description
This PDF contains the flag, but you’ll probably need to fix it first to figure out how it’s embedded. Fortunately, the file contains everything you need to render it. Follow the clues to find the flag, and hopefully learn something about the PDF format in the process.
The challenge provides us with a challenge.pdf file

## The ruby script
When we try to open the pdf, we just get a white page with nothing on it. Let’s run strings on it and see if we can find something:

![ruby](/assets/img/posts/pdf-analysis/pdf1.png)

This line reveals that the pdf file can also be interpreted as a ruby script. Here’s the entire script:
```ruby
port = 8080
if ARGV.length > 0 then
  port = ARGV[0].to_i
html=DATA.read().encode('UTF-8', 'binary', :invalid => :replace, :undef => :replace).split(/<\/html>/)[0]+"</html>\n"
v=TCPServer.new('',port)
print "Server running at http://localhost:#{port}/\nTo listen on a different port, re-run with the desired port as a command-line argument.\n\n"
loop do
  s=v.accept
  ip = Socket.unpack_sockaddr_in(s.getpeername)[1]
  print "Got a connection from #{ip}\n"
  request=s.gets
  if request != nil then
    request = request.split(' ')
  end
  if request == nil or request.length < 2 or request[0].upcase != "GET" then
    s.print "HTTP/1.1 400 Bad Request\r\nContent-Length: 0\r\nContent-Type: text/html\r\nConnection: close\r\n\r\n"
    s.close
    next
  end
  req_filename = CGI.unescape(request[1].sub(/^\//,""))
  print "#{ip} GET /#{req_filename}\n"
  if req_filename == "favicon.ico" then
      s.print "HTTP/1.1 404 Not Found\r\nContent-Length: 0\r\nContent-Type: text/html\r\nConnection: close\r\n\r\n"
      s.close
      next
  elsif req_filename.downcase.end_with? ".zip" then
    c="application/zip"
    d=File.open(__FILE__).read
    n=File.size(__FILE__)
  else
    c="text/html"
    d=html
    n=html.length
  end
  begin
    s.print "HTTP/1.1 200 OK\r\nContent-Type: #{c}\r\nContent-Length: #{n}\r\nConnection: close\r\n\r\n"+d
    s.close
  rescue Errno::EPIPE
    print "Connection from #{ip} closed; broken pipe\n"
  end
__END__
<html>
  <head>
    <title>A PDF that is also a Ruby Script?</title>
  </head>
  <body>
    <center>
      <a href="/flag.zip"><h1>Download</h1></a>
    </center>
    <!-- this is not the flag -->
  </body>
</html>
<!--
```
Without going too much into detail, the script basically just starts a webserver on port 8080. So let’s run this script and visit the website. We can simply do this by typing ruby challenge.pdf and then visit localhost:8080 in our browser

![website](/assets/img/posts/pdf-analysis/pdf2.png)

The webpage is quite bare but we can download something, a flag.zip file. (Are we close?).

## The zip file
When we unzip flag.zip, we get a folder called feelies containing two files, false_flag.md (shit) and mutool. Here’s what false_flag.md reads :
```md
You didn't think it would be this easy, did you?

https://www.youtube.com/watch?v=VVdmmN0su6E#t=11m32s

Maybe try running `./mutool draw -r 300 -o rendered.png` on this PDF


$ docker run -ti --rm -w /workdir/ --mount type=bind,source="$PWD",target=/workdir ubuntu:bionic ./mutool 
```
The youtube link leads to a video by LiveOverflow (great youtuber) in which he talks about file formats and how to not design CTF challenges. So this is a dead end. However, now that we have mutool, we can run the specified command and see what we get:

![pdf](/assets/img/posts/pdf-analysis/pdf3.png)

So apparently we’re still not at the end. This image contains several interesting hints but I focussed mostly on the last line:
**LMGTFY: 2642 didier “42 bytes” object**
LMGTFY stands for Let me google that for you so let’s google “2642 didier “42 bytes” object”.
This leads us to the following website: https://blog.didierstevens.com/2008/05/19/pdf-stream-objects/
The author talks about pdf streams and how they can be interpreted in different ways using Filter cascades. With this in mind, let’s search for streams that employ this technique in our pdf.(Just use ghex and search for the keyword “stream”).

## PDF streams

Doing this, we soon come across the following two streams:
```
4919 0 obj
<<
/Length 100
/Filter /FlateDecode
\>>stream
.
.
.
endstream
endobj
4919 1 obj
<<
/Length 89827
/Filter [/FlateDecode /ASCIIHexDecode /DCTDecode]
\>>stream
```

Now first of all, every object in a pdf file has something called an object reference that are indexed in the xref table at the end of each pdf. Streams in particular have an indirect object reference. In our case that object reference is 4919. We also notice that the second stream overwrites the first, because of its higher version number (1 instead of 0). These two streams are at the offsets 0xB5F31 and 0xB5FFD respectively. This is important for when we extract the objects with binwalk later. We can see that the first filter is FlateDecode. This filter basically extracts zlib compressed objects. The second stream also has the ASCIIHexDecode filter which transforms hex numbers into a binary file and the DCTDecode filter that decodes the binary content to a jpg file and displays it. So let’s run binwalk and recuperate the objects.

![binwalk](/assets/img/posts/pdf-analysis/pdf4.png)

We can see our two streams. Binwalk automatically extracts the files from the compressed streams. Now we can look what the first stream contains:

```bash
cat B5F31

pip3 install polyfile
Also check out the `--html` option!
But you'll need to "fix" this PDF first!
```

*I actually didn’t use this hint for anything, I still don’t know the intended way to get the flag so if anyone did it using another technique, drop a comment.*
Now let’s look at the other stream in file B5FFD:

![encoded flag](/assets/img/posts/pdf-analysis/pdf5.png)

Those are the hex numbers that are going to get transformed into bytes by the second filter. We can write a script to do this or we can use the help of a website. I did the latter. Doing this we get a jpg file that contains the following:

![flag](/assets/img/posts/pdf-analysis/pdf6.png)

So the flag is justCTF{BytesAreNotRealWakeUpSheeple}.

## Conclusion

This was a great challenge that taught me a lot about the inner workings of the pdf file format. The author certainly stuck to the advice that LiveOverflow gave in his video. The file however contained a lot more hints that didn’t directly apply to my solution such as a hint to use readelf -p .note and the hint to use polyfile. If anyone has found another way to solve this challenge, I’d be very interested to hear your solution. Until next time
-Trigleos

