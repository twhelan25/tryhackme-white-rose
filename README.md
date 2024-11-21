![intro](https://github.com/user-attachments/assets/ea44f140-2536-41c9-966b-07d1befd83bd)
![intro 2](https://github.com/user-attachments/assets/8a38ad71-ae4f-438d-a15d-559edd19ef4a)

# tryhackme-white-rose

This is a walkthrough for the tryhackme CTF White Rose. I will not provide any flags or passwords as this is intended to be used as a guide. 

## Scanning/Reconnaissance

First off, let's store the target IP as a variable for easy access.

Command: export ip=xx.xx.xx.xx

Next, let's run an nmap scan on the target IP:
```bash
nmap -sV -sC -A -v $ip -oN
```

Command break down:

-sV

Service Version Detection: This option enables version detection, which attempts to determine the version of the software running on open ports. For example, it might identify an HTTP server as Apache with a specific version.
-sC

Default Scripts: This option runs a collection of default NSE (Nmap Scripting Engine) scripts that are commonly useful. These scripts perform various functions like checking for vulnerabilities, gathering additional information, and identifying open services. They’re a good starting point for gathering basic information about a host.
-A

Aggressive Scan: This option enables several scans at once. It combines OS detection (-O), version detection (-sV), script scanning (-sC), and traceroute (--traceroute). It’s useful for a comprehensive scan but can be intrusive and time-consuming.
-v

Verbose Mode: Enables verbose output, which provides more detailed information about the scan’s progress and results.
$ip

Target IP: This is a placeholder for the target IP address you want to scan. In practice, replace $ip with the actual IP of the machine you are targeting.
-oN

Output in Normal Format: This option saves the scan results in a plain text file format. After -oN, specify a filename where you want to store the output.

![nmap](https://github.com/user-attachments/assets/eb5784e0-9dd7-4851-82e7-5b1a4fdfe474)

The scan reveals two open port, ssh and a web server on port 80. Let's check out the web sever.

When I tried to navigate to the ip, the url changed form the ip to cyprusbank.thm and the browser displayed site cannot be reached. After adding cyprusbank.thm to our /etc/hosts file, the site is displayed:

![hosts](https://github.com/user-attachments/assets/0e8f9346-30a4-49c4-ab3d-20d925bda498)

This site loads this message about maintainence:

![national_bank](https://github.com/user-attachments/assets/492324cc-41a3-4b72-8f05-e09051a08ce6)

I also ran a gobuster and nikto scan on the target but it didn't turn up anything so I ran a scan for sub domains with ffuf:

![ffuf](https://github.com/user-attachments/assets/a0bff6f0-c01b-426d-a5fd-ccfa61abebb6)

This reveals an admin sub domain.

We will repeat the same process of adding this to our /etc/hosts:

![hosts2](https://github.com/user-attachments/assets/178466b4-3c6f-44b2-9290-8926e9d6c14c)

This brings us to an admin login panel and we'll use Olivia's creds that were provided in the ctf introduction:

![cortez_admin](https://github.com/user-attachments/assets/886b2892-b2c1-4ff2-ba4b-82dce9648b2c)

Once logged in the home page displays a list banking information for different bank customers, including Tyrell. However, the phone numbers are not yet shwon, so we can't answer the first question yet. 

When navigating to messages we see a chat box that displays some previous messages, which Olivia can add to.

![messages1](https://github.com/user-attachments/assets/f17fceb5-5902-4fb0-82e2-54326f87b592)

After examining the url, we see the ?c=5. I tested this for an IDOR vulnerability and by changing the 5 to 0:

This reveals previous messages, including one where admin Gayle Bev shares her credentails!

![gale creds](https://github.com/user-attachments/assets/6f54571b-855c-4b13-bb6c-6bbfb1ce34f4)

We'll log out of Olivia's account and back into Gayles:

![gayle_login](https://github.com/user-attachments/assets/b8f3b491-25d2-4585-9524-6082a18eae79)

And now we can see Tyrell's number.

![tyrell](https://github.com/user-attachments/assets/8d515b4e-798c-4b8e-94c8-f7ad3caec65e)

We can also access the settings menus for the site. I updated Tyrell's password to password:

![settings](https://github.com/user-attachments/assets/2f4a61ca-e72d-4faa-b1d3-1509b0d699aa)

The import thing to notice here is the password change is fully displayed on the site which suggests that the site is vulnerable to SSTI.

We will update the password again and capture the request in burp suite.

Once we have incepted the request, we'll sent it to repeater:

![intercept](https://github.com/user-attachments/assets/ebc78c75-e6d1-4215-86cd-d2462392d58b)

As a test, I added a 0 to the end of password-password0 and click send. In the response we see an error "settings.ejs". 

![intercept2](https://github.com/user-attachments/assets/26e4f783-d04f-492b-85b1-8df8a9898632)

Here's a brief explaination:

![gpt](https://github.com/user-attachments/assets/c6e09053-7926-4e51-87ee-ef252c4f10f6)

I then did a google search of "ejs ssti payloads" and here's the site I used to base my payload off of: https://eslam.io/posts/ejs-server-side-template-injection-rce/

The first payload that I used is:
``` bash
&settings[view options][outputFunctionName]=x;return global.process.mainModule.require('child_process').execSync('id');//
```
And here is the response:

![payload1](https://github.com/user-attachments/assets/995b96f5-08e6-43ea-a756-d74b69b4503f)

Then I changed to get /etc/passwd:

![passwd](https://github.com/user-attachments/assets/a7ae1d45-df60-4ab5-a670-c02114230a5a)

Now, let's get RCE with a reverse shell. I used nc mkfifo with url encoded:

![shell](https://github.com/user-attachments/assets/1f1ab7de-a9dd-4199-a1b7-87b8ba0f403e)

![web](https://github.com/user-attachments/assets/2baad4ae-5458-4602-8609-9a8ebad924b4)

And now we can grab user.txt:

![user txt](https://github.com/user-attachments/assets/b4838ffa-4879-4f59-9add-302aa0a62db4)

And sudo -l reveals that we can run sudoedit with this path and I checked the sudoedit version:

![sudo -l](https://github.com/user-attachments/assets/effdb339-77ed-4c15-9875-e30735104510)

I did a google search and found this exploit:

![exploitdb](https://github.com/user-attachments/assets/6d688218-e4a8-40e8-ae09-2178d125c980)




