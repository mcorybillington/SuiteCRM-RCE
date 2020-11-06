# CVE-2020-28328 SuiteCRM Remote Code Execution via Log File System Setting and Log File Poisioning
#### Overview
I recently discovered two vulnerabilities in [SuiteCRM](https://github.com/salesagility/SuiteCRM) that provides an attack chain for a low privileged user to achieve code execution on the underlying operating system. The attack chain is Cross-Site Scripting, which can be used to perform Cross-Site Request Forgery, which leads to Remote Code Execution by tampering with the application configuration and poisioning a log file. This is all achieved via a file upload that contains malicious JavaScript that a low privileged user can trick a user with administrative privileges into running. The Proof-Of-Concept files and video I have attached demonstrates a low privileged user performing this attack and obtaining areverse shell on the system that is hosting SuiteCRM.  
[This was patched in version 7.11.17 of SuiteCRM.](https://suitecrm.com/suitecrm-7-11-17-7-10-28-lts-versions-released/)
#### Version details
SuiteCRM Version 7.11.15
## Cross-Site Scripting (XSS)
The stored Cross-Site Scripting exists in the 'Create Documents' file upload. A low privileged user is able to upload a file with any contents. The user can then examine the link provided to download this document and ascertain the file's location on the filesystem by locating the `id` parameter. This long, random value is the name of the fileinside the /uploads/ directory. The user can place arbitrary JavaScript in this file and then send the link to another user. This can be used to hijack another user's session and/or perform actions on that user's behalf, which will be shown in the PoC video.
## Remote Code Execution
After discovering that I could become the admin through session hijacking via the Cross-Site Scripting, I then discovered that I could control the system properties under 'Admin → System Settings', namely the log file property. Log file extensions were pretty well blocked, but I was able to use BurpSuite to update the 'Log File Name' value to be any arbitrary value, including .php extensions. I did this by submitting a request without changing anything and capturing the POST request that actually updates the values. I changed the filename via the `logger_file_name` parameter to `shell.php` and simply made the 'Extension' field blank. That provided a php file that I could access in browser at the webroot, but I needed some php code insidethe file to execute.  
Next, I examined the output in the file and noticed that I could control input into the file via user properties if I updated a user (such as the user's first or last name), if the logging was set to info (which I believe was default...). So, I captured a request in burp and inserted some php code ``<?php $id =`id`; echo $id; ?>`` in the `last_name` form field. This resulted in the output of the `id` command on Linux in the context of the web server user, `www-data`. The only characters that I could tell were escaped based on the log file are single quotes, double quotes, and back slashes. You can verify this by `tail`'ing the sql log file on the backend.
## Chaining the two for Cross-Site Request Forgery (One click -> shell)
I was able to perform all this as the admin user because I was able to obtain session cookies, however to have a working chain, I needed to have this execute via JavaScript in the context of the admin user. I was able to script this using a few fetch requests to perform each of these POST requests. The first one updated the system properties, the second updated the admin user's 'Last Name' field, and the last performed a GET request on the newly created log file with malicious php code. The malicious php code would perform a curl request against my machine, pull down a bash reverse shell, and then pipe the output of that curl request directly to bash, executing the code. I then uploaded this using the same method that I discussed in the XSS section and revisited the new link as the admin user.  With a web server hosting my bash file and a netcat listener running, I was able to get a reverse shell.
## Mitigation
#### Cross-Site Scripting
Ensure you have `AllowOveride All` set in Apache. nginx does not have this setting and I did not test it on nginx.
#### Remote Code Execution
Update to the latest release of SuiteCRM, or at least version 7.11.17.  
This is the specific fix. Commit [1618af16eaa494c4551bac961e5ac8fc3d87ab8c](https://github.com/salesagility/SuiteCRM/commit/1618af16eaa494c4551bac961e5ac8fc3d87ab8c#diff-e9704a2002d127cd455e1eb0507042080bb79d362091e770803ff69a31139d0f)
## Reporting to SuiteCRM
SuiteCRM was very responsive throughout the reporting process. They acknowledged the RCE, which was patched. The XSS was the result of a web server configuration so they did not acknowlede it as a vulnerability. They did, however, note that they would be updating the documentation in light of this.
#### Timeline
06 AUG 2020 -> Both issues reported to security@suitecrm.com  
07 AUG 2020 <- SuiteCRM confirms receipt of report and raises issue with internal security team  
21 AUG 2020 -> I contacted security@suitecrm.com for a follow up  
25 AUG 2020 <- SuiteCRM replies regarding web server config/XSS  
26 AUG 2020 -> I reply to say that the suggestion of `AllowOveride All` mitigates XSS  
16 SEP 2020 -> I contacted security@suitecrm.com for a follow up  
17 SEP 2020 <- SuiteCRM replies to confirm the issue as a partial issue  
29 OCT 2020 Update released  
03 NOV 2020 -> I contact security@suitecrm.com to ensure nothing else is needed on their end before releasing writeup  
05 NOV 2020 <- SuiteCRM replies 
> Now we have released a patch for this issue and it is in the pubic domain, there is no problem with you doing a blog post on the vulnerabilities from our perspective. 

05 NOV 2020 -> CVE requested by me  
06 NOV 2020 <- [CVE-2020-28328](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-28328) issued  
## POC
These aren't too tough to figure out, so I'll leave scripting it as an exercise for the reader... ;) They are also pretty easy to just perform through the web interface...  
![](SuiteCRM-PoC.gif)
## Thanks to SuiteCRM!
They were very easy to work with and I definitely plan to continue searching for and reporting vulnerabilities in this software!
