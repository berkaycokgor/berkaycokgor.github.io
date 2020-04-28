---
layout: post
title: DMV WRITE UP
---
Welcome all, hope you like this Walkthrough

Feedbacks are welcome

Let's start !

After I opened the machine, I scanned my network and found this

![nmap scan result 1](https://i.ibb.co/sKjZ5mF/dmv1.png)

I thought 10.56.8.6 is our target and looks like a web server with ssh enabled.
I scanned it deeper to not miss any open ports.

![nmap scan 2](https://i.ibb.co/mNrCW8B/dmv2.png)

-A parameter enables OS detection, version detection, script scanning and traceroute

-p- parameter scans all ports from 0 to 65535(max)

Well, it told us some other things than the previous scan.Web server is Apache 2.4.29 and SSH is OpenSSH 7.6p1 and server's OS is Ubuntu.

Then I moved on to the Web Page which is on port 80.

![web page](https://i.ibb.co/NmCRpy9/dmv3.png)

It looks like it converts a youtube video to mp3 (genius guess)
So, I tried a music video to convert.
First, I gave it a full youtube url like this
https://www.youtube.com/watch?v=XXXXXXX
and it says Oops! Something went wrong.
Then I noticed it says "Video ID:"
So I tried give the XXXXXXX part.

![enter image description here](https://i.ibb.co/X8rMC5W/dmv4.png)

It worked ! 
Link below(Download MP3) sent me to /tmp/downloads/somerandomnumbers.mp3

Checked /tmp/ and /tmp/downloads/ directories.

/tmp/ was 403 Forbidden and /tmp/downloads/ 404 Not Found(?)
So downloaded mp3 file and started to analyze it. Found nothing interesting.
I didn't know anything about converters or audio-video formats so I moved on things at least I know something about.

I went back to Web Page again.
I fired up burp and setted up proxy and intercepted the convert requests.
When I caught the request, i sent it to repeater to modify easily.

![enter image description here](https://i.ibb.co/MNN9BWH/dmv5.png)

Here is what it looks like.
This is a post request with one parameter named yt_url which is the full url of the video.

![enter image description here](https://i.ibb.co/wLZ7y4X/dmv6.png)

Response of that request is above.
It's a quite long json data.
I beautified it with some tools i found online.(There are also some burp extensions to do it btw)

![enter image description here](https://i.ibb.co/PxK6cCz/dmv7.png)
It looks much less scary now.

Key values are self-explanatory.

Since I know nothing about what generates this json output, I thought error part can be helpful to identify the application.
I copied the error value and paste it to google.

![enter image description here](https://i.ibb.co/vq7Jc2Z/dmv8.png)

Yay !
Looks like this is our guy.

https://github.com/ytdl-org/youtube-dl

**Command-line** program to download videos from YouTube.com and other video sites

I highlighted command-line part because it highly alerts me there can be OS command injection.
Of course  if our inputs aren't filtered well.
I installed youtube-dl to play with it and understand how it works.

![enter image description here](https://i.ibb.co/4SmmyL2/dmv9.png)

It's pretty simple, you give the url as a parameter and it downloads the video.
If you want to extract the audio you just give the -x parameter and it converts.
After I understood the basic usage the program I returned to burp.

I tested some basic command injection techniques like " | " , " & " , " ; ".

When I gave the " | " and  "&" there was an interesting thing in response.

![enter image description here](https://i.ibb.co/tZgmBpn/dmv11.png)

In errors part it says "sh: 1: -f: not found"

It is interesting because it is actually a sh shell error.

Let's look it in this way.

![enter image description here](https://i.ibb.co/xGR9d6C/dmv12.png)

See?  When you try to bind an unrecognized command with " | " it generates the same output with the error in the response.
Looks like we managed to make the application behave in an unexpected way,
Which is the way we want

After a few tries I couldn't get any output or signal that my payload works.
I tried "| whoami" , "| sleep 15" and things like that.
Nothing worked.

But I noticed that "url_original" key in the response looks like it contains our input.
Furthermore, I noticed that in the "url_original" it only contains the things until the whitespace.
Let me show you


![enter image description here](https://i.ibb.co/r2yRTnn/dmv14.png)
![enter image description here](https://i.ibb.co/w0Y3xTj/dmv13.png)

See the difference? 
Thus I started to give payloads without whitespaces.
Because it works the same.There's no difference for one word commands.
Let me show you this, too

![enter image description here](https://i.ibb.co/ZdN7TnP/dmv15.png)

As you can see there's nothing to do with whitespaces.

Now, We have to imagine how does our payload look like when web app run it.


![enter image description here](https://i.ibb.co/xGR9d6C/dmv12.png)

If we look at this scenario again(In response we saw that sh error when our payload was " | "), we know that in web app there's a "-f" parameter after payload automatically added. Moreover, we know that there's at least one command before our payload because it has to start with "youtube-dl" command.
Thus, we are pretty sure that our payload will be putted in the middle of the command.

Since we know it , we have to put command bind chars in the beginning of our payload as well as end of it.


![enter image description here](https://i.ibb.co/c6FjXTW/dmv28.png)

it must look like this.
Lets try it and see if our guess was true

![enter image description here](https://i.ibb.co/VNsgQyw/dmv16.png)

Yay! In output part we see www-data ! which means we managed to run whoami command on the server

The reason of you see %26(url encoded version of &) instead of & in our request is "&" has a special meaning in http post requests.
It is used to separate parameters so if you write "&" directly it doesn't behave like you want and not added to payload. You have to encode this character.

Yeah, even if we managed to run command on server, we still have a big problem with whitespaces. Since our goal is to get a nice shell from server, we cannot do this with one word commands.We have to figure it out.

At this point I immediately opened google and searched "how to bash script without whitespaces?" directly. :D

![enter image description here](https://i.ibb.co/ncLb0Mj/dmv17.png)

[The link I found the answer](https://unix.stackexchange.com/questions/351331/how-to-send-a-command-with-arguments-without-spaces)
Looked like there's a way to do it.
Let's try it then

![enter image description here](https://i.ibb.co/wcxsv0V/dmv18.png)


We successfully managed to run "uname -a" by a nice trick.

Now we can use it to get a shell !

After a few tries I couldn't manage to get a reverse shell via one liner reverse shells.
(bash etc.) because ${IFS} thing is problematic with output redirection things and has syntax errors.

Thus, I tried to upload a webshell with wget due to low complex syntax.
I set up a python web server on my kali machine with b374k shell named "b.php" in it.
[Web shell that I use](https://github.com/tennc/webshell/blob/master/php/b374k/b374k-3.2.2.php)

![enter image description here](https://i.ibb.co/TtP79jp/dmv19.png)

We see the "b.php saved" output.
Let's check our webshell.

![enter image description here](https://i.ibb.co/4t8K0Tb/dmv20.png)

Awesome ! 
We have a webshell on the server now !
If you're using b374k shell for the first time and you follow the steps by doing it with me there'll be a password prompt before accessing the shell. It's "b374k".

You can get a reverse shell with b374k from the Network prompt.

![enter image description here](https://i.ibb.co/3CQNJPv/dmv21.png)

Here is first flag in the admin folder.

For priv esc I run enum scripts  but found nothing interesting for a while.
I even tried Apache 2.4.29 Carpe Diem vuln but couldn't make it run.

At this point I asked for a hint on a telegram group and somebody said look at with [pspy](https://github.com/DominicBreuker/pspy) which is a tool to monitor processes in linux.It allows you to see commands run by other users, cron jobs etc which is a great utility.
I uploaded pspy to the server and  run it.

![enter image description here](https://i.ibb.co/94dTq08/dmv22.png)

Did you notice it?
(UID=0 means root)
Root is running a script on the web app folder.
There's a high chance we can modify it.
Lets go and check its permissions.

![enter image description here](https://i.ibb.co/mz4ccL7/dmv23.png)

Amazing !
File is even owned by www-data :D
Let's put a reverse shell in this script and hope it work nicely and give us a root shell.

![enter image description here](https://i.ibb.co/b7n91yr/dmv24.png)

Looks nice.


![enter image description here](https://i.ibb.co/WznXJFs/dmv25.png)


![enter image description here](https://media1.tenor.com/images/1ebc8fe1c139cdb48686b208dfcdf5d5/tenor.gif?itemid=12928789)

![enter image description here](https://i.ibb.co/JpCqK3k/dmv26.png)


![enter image description here](https://media.giphy.com/media/l2Sq8Oo9Pas7NoDoA/giphy.gif)

Yeah we did it !
Lets go and check the root flag.

![enter image description here](https://i.ibb.co/7Rt2Km2/dmv27.png)

:) hope you all liked this walkthrough.
Since I am not very experienced in writing blog posts, feedbacks are really important.

[@cokgorb](https://twitter.com/cokgorb)
