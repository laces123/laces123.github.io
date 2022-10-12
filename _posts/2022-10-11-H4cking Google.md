---
title: H4cking Google CTF Eposide000 challenge 1
date: 2022-10-11 6:00:00 
categories: [H4cking Google, CTF]
tags: [ctf, sql injection, lfi]
---
# H4ck1ng Google CTF

The h4cking google ctf is part of the new h4cking google youtube video series which is really good and I suggest that you give it a watch.  Here is a link to the [playlist](https://www.youtube.com/playlist?list=PL590L5WQmH8dsxxz7ooJAgmijwOz0lh2H) If you want to jump straight to the ctf you can go to <https://h4ck1ng.google>

## Eposide000 Challenge 1
---
Start Url for this challenge is <https://hackerchess-web.h4ck.ctfcompetition.com/>

In this challenge this challenge you need to be able to checkmate the computer or do you?

Lets start off with a port scan. Now I know that nmap is considered the GOAT of port scanners but I think ProjectDiscovery port scanner [naabu](https://github.com/projectdiscovery/naabu) is great if you want something fast and reliable but don't need all the extra information the nmap gives you

```bash

sudo naabu -host hackerchess-web.h4ck.ctfcompetition.com

                  __
  ___  ___  ___ _/ /  __ __
 / _ \/ _ \/ _ \/ _ \/ // /
/_//_/\_,_/\_,_/_.__/\_,_/ v2.1.0

                projectdiscovery.io

Use with caution. You are responsible for your actions
Developers assume no liability and are not responsible for any misuse or damage.
[INF] Running SYN scan with CAP_NET_RAW privileges
[INF] Found 2 ports on host hackerchess-web.h4ck.ctfcompetition.com (34.160.239.58)
hackerchess-web.h4ck.ctfcompetition.com:80
hackerchess-web.h4ck.ctfcompetition.com:443
```
Ok, looks like nothing but your basic web ports

So you get a chess board when you visit the web page with some other links.  
![Hacker Chess website](/assets/images/2022-10-11_18-20.png)

With your web browser proxying all your traffic through burp click on all the links and play with the functionality.  You will notice that if you try and play a game of chess the computer cheats and you can't win.  Well this is hacker chess so I am sure that there is some way to beat this game.
## Method 1
I sure you have already notice that there is a link called Master Login, bet that is intresting lets take a look. 

![Master Login](/assets/images/2022-10-11_21-49.png)

Well what do we have here.  Its a very basic login panel, you can try the usually subjects for you username and password

* admin/admin
* root/root
* guest/guest
* test/test
* chess/chess

Usually on login panels for ctf's brute forcing or guessing the username and password isn't the way forward.  SQL injection is a common thing to try on login panels. So I figure I try a basic payload

```
username: admin
password: ' or 1=1-- 

Make sure you add a space after the -- or it won't work
```
Bam! we get in and it redirects to a page like this 
![Admin panel/dashboard](/assets/images/2022-10-11_22-07.png)

Alright we can stop that pesky AI from cheating, and I wonder what happens if we make the ai think negative time.

Now lets go back to the homepage and see if we can beat the AI

## Method 2

I think that this way of getting the flag is more non gimicky/ realistic way of hacking a server. 

On the main page if you go to view the source and scroll down towards the bottom you should see some javascript code that looks like this.

```javascript

<script>
function load_baseboard() {
  const url = "load_board.php"
  let xhr = new XMLHttpRequest()
  const formData = new FormData();
  formData.append('filename', 'baseboard.fen')

  xhr.open('POST', url, true)
  xhr.send(formData);
  window.location.href = "index.php";
}
</script>
```
What this script does is make a http post request to load_board.php form data filename=baseboard.fen.  Now if you have been proxing all your request to Burp/Zap in your Http History you should see a post request to load_board.php send it to repeater and you should have something like so.
![Repeater Tab](/assets/images/2022-10-11_22-59.png)

It appears to take a file name baseboard.fen show you the contents of that file.
Filename parmaters are great places to try for lfi or Local File Inclusions. So lets change baseboard.fen to ../../../../../../../etc/passwd which is a world readable file on all linux systems.  
![LFI etc/passwd](/assets/images/2022-10-11_23-07.png)

Yes we see the contents of passwd we have proven that we have lfi.  So what can we do with this?  Well we don't know where the flag is saved or the name of the file its saved has so we need to do a bit more.  Normally when you do a view source on a html page you don't see any of the server side code (php in this case).  Lets see if we can see the php code for index.php. Change are payload to from ../../../../../../../etc/passwd to index.php and we get back the php code.  Here is a snippet of the code that is relevent for us.
```php
if ($chess->inCheckmate()) {
        if ($chess->turn != "b") {
            echo '<h1>You lost! Game Over!</h1>';
        } else {
            echo "<h1>ZOMG How did you defeat my AI :(. You definitely cheated. Here's your flag: ". getenv('REDIRECT_FLAG') . "</h1>";
        }
}
```

It essiantally says if we checkmate the AI for it to get the flag from the enviroment variable __REDIRECT_FLAG__.  We have now found where the flag is stored but how to access it cause its not in a file like flag.txt.  It is store in a enviroment variable.  Luckly on linux there is a file that list or shows all the eniroment variables

That file location is 
```bash
/proc/self/environ
```
So now all we need to do is change are lfi payload to something like this

```bash
../../../../../../../proc/self/environ
```
Search for the __REDIRECT_FLAG__ variable and you will get the flag.