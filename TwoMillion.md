Hello,

  My name is mreyer0339 and this is my first CTF writeup using retired machines on (https://app.hackthebox.com/). I am using OpenVPN to connect to HTB virtual machines on my Kali Linux machine. 

  I am currently certified with the following:
  -CompTIA A+
  -CompTIA Network+
  -CompTIA Security+
  -CompTIA PenTest+
  -CompTIA SecurityX (Formerly CASP+)
  -AWS Certified Cloud Practitioner

  I hope to continue my red-team process learning through HTB and doing more writeups, as well as eventually working towards completing OCSP certification.

  

############################################################## WRITEUP ############################################################## 

HTB Machine: TwoMillion

############################################################## ENUMERATION PHASE ##############################################################

Let's begin by pinging the machine to ensure connectivity.
![image](https://github.com/user-attachments/assets/a6aede84-cb42-4e54-a071-0052116a3b92)

Success!

Now, lets add the IP to our /etc/hosts file (2million.htb)

![image](https://github.com/user-attachments/assets/7abc96a0-3dc3-4503-a534-6b0f79e85d88)

Now lets perform an nmap scan (versioning, OS detection, top 100 ports)

![image](https://github.com/user-attachments/assets/283ba6b7-5a4f-4d60-b3f1-28766ed51d7d)

Ports 22 (SSH) and 80 (HTTP) are open on a Linux device.
I'll start by trying to see if I can SSH using root or admin without a password.

![image](https://github.com/user-attachments/assets/a7786c71-ac18-4754-8cac-3f18144f7a9a)

Fail.

Let's try and access the web server on port 80 by navigating to http://10.10.11.221 on my browser.

![image](https://github.com/user-attachments/assets/a5998660-2fc8-4a15-91ed-32b1516ebacb)

We have access to the website. I see a "Join" & "Login" page. Let's start by navigating to the "Join" page and inspecting the source code.

![image](https://github.com/user-attachments/assets/d5dabba3-d179-4ce3-a0be-46a9144fe9b7)

It looks like there is a function that submits a POST request to /api/v1/invite/verify which first checks if the request was successful and then stores it locally.
I also see a script called inviteapi.min.js. Let's look at what it does.

![image](https://github.com/user-attachments/assets/306c2c20-1a75-4151-810c-31952a4cf3de)

This is all garbled...  Lets run it through a javascript unpacker. I used one at (https://matthewfl.com/unPacker.html)

![image](https://github.com/user-attachments/assets/9d713346-6bc0-4fb4-82ec-3f93d20fcc76)

Great. It looks like inviteapi.min.js has 2 functions: The first one looks almost the same as the previous one that verifies an invite code. The second one makes a POST request to /api/v1/invite/how/to/generate.
Lets dig further into this by using the curl command.

![image](https://github.com/user-attachments/assets/a42dcd1f-3278-4563-87ac-6b9272a88fc3)

I'm seeing some encrypted-looking data with a mentioned encryption type: ROT13 (as well as a nice little hint that it needs decrypting).
I googled ROT13 because it was an encryption type i'm not familiar with and found this website: (https://rot13.com) which seems to allow for decryption. I pasted the encrypted string and decrypted.

![image](https://github.com/user-attachments/assets/a91c3ee0-a817-443a-9954-6cd94a127c45)

Following the instructions, I made a POST request to the server

![image](https://github.com/user-attachments/assets/d330ada2-f455-4391-ba24-c3636d22f8b2)

This looks very similar to the previous output, but isn't encrypted anymore. It may be Base64 encoded (the giveaway is that its only ASCII characters and it ends with a "=")
The next step would be to decode it.

![image](https://github.com/user-attachments/assets/f688c32e-3ef9-467e-9e5c-396a7118c439)

Is that an invite code??? Looks like one. Let's test it.

![image](https://github.com/user-attachments/assets/5eaf038b-8cc1-4f96-9363-08c5bce296ea)

Huzzah! I'll add some random credentials (U: random, E: random@2million.htb, P: random)
Logging in with those credentials worked. I am brought to a home page with an interesting page: "Access"

![image](https://github.com/user-attachments/assets/2996cac5-95e9-41a1-8ea6-e2a2b4f8d889)

![image](https://github.com/user-attachments/assets/38333781-fa25-4c9c-bf52-8adfad8148d6)

The page looks like it lets you regenerate a VPN file for HTB. There is also a button to download "Connection Pack". Let's initiate BurpSuite to see what it does. Here is the output:

![image](https://github.com/user-attachments/assets/66446107-4f68-4527-b516-e2fff59f2f09)

It looks like it sends a GET request to /api/v1/users/vpn/generate and downloads the VPN.

I want to test out the URL /api using curl.

![image](https://github.com/user-attachments/assets/f58ac72e-744f-46f2-a67f-383e23aec9af)

Unauthorized (401). If I authenticate with my PHP session cookie from my browser it may work.

![image](https://github.com/user-attachments/assets/7d3e4203-e7bc-4bb8-a1c2-be99ae24661b)

Yep. That did it. OK (200)

Lets go a step further and request /api/v1. 
I start by trying curl but realize it needs to be cleaned up or else I'll get an aneurysm. I found that piping to jq makes it all really pretty :)

![image](https://github.com/user-attachments/assets/0870920e-f7b1-4f5d-b4ee-367eebe573f6)

Lots of endpoints shown here. Interested in the admin ones.
I'll check if im an admin first. I doubt it...

![image](https://github.com/user-attachments/assets/dfc4686a-70ac-433b-a825-fbe515bd212c)

Negative. 

Let's next try the /vpn/generate endpoint.

![image](https://github.com/user-attachments/assets/0dbbf4d9-ad30-47d4-b690-2a39ef6ca47a)

Unauthorized (401)

On to the last one: /settings/update

![image](https://github.com/user-attachments/assets/61e6de69-20e4-45ed-9a2d-3a4a9d184f8c)

Looks good. OK (200). Weird that it says invalid content type... 

I know from my SecurityX studies that APIs typically talk using JSON, and this is confirmed by previous server response syntax.

I'm going to retry this, setting the Content-Type header to JSON

![image](https://github.com/user-attachments/assets/71672c2d-e6bf-4037-8dcc-82d16d6f6a16)

Missing parameter: email
Let's add it and retry.

![image](https://github.com/user-attachments/assets/1b473969-bbf1-4d62-b419-498f920432e9)

Still missing a parameter: is_admin
One more time...

![image](https://github.com/user-attachments/assets/347840c8-b90d-46fe-ba43-cb4782b5c1b2)

One more time...
Gotta get my variables right.

![image](https://github.com/user-attachments/assets/1f3a3f45-6f60-4aa1-a228-2880d2424e75)

Rechecking admin status by curl'ing to the /admin/auth endpoint

![image](https://github.com/user-attachments/assets/95f4bcbb-de36-4b49-8b31-815595c632a9)

I am an admin. 

############################################################## FOOTHOLD PHASE ##############################################################







