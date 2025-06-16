# WRITEUP

HTB Machine: Support

# Enumeration

Let's start by running OpenVPN and starting our machine.

Then ping the machine to ensure connectivity:

![image](https://github.com/user-attachments/assets/c5c0a7c9-6beb-42d1-91d4-c6d49677b527)

Now an Nmap scan to find open ports/services/versions

![image](https://github.com/user-attachments/assets/e7e8b469-5ce5-4d39-a876-f9b22b54c716)

A couple of open ports. Let's first check for open SMB shares (port 445)

I tried connecting and entered a blank password:

![image](https://github.com/user-attachments/assets/848015ee-1ecc-460d-abda-8c71f0e9b515)

In connecting with `support-tools` i get a smb shell

![image](https://github.com/user-attachments/assets/41108dc2-ee8c-4b55-bc8b-22a49e534df7)

There is an interesting file worth examining: `UserInfo.exe.zip`

Let's download the file and unzip it.

![image](https://github.com/user-attachments/assets/60301f57-ba02-4ab1-8504-1290dfc0279f)

![image](https://github.com/user-attachments/assets/1804bd8d-7fc4-430a-8cb1-8e869c3701ed)

It unzipped into lots of DLL files and an executable: `UserInfo.exe`

Let's use the `file` command to see what it is.

![image](https://github.com/user-attachments/assets/b725fdbd-5827-4e55-8484-6f4d7ccec33c)

Ah, it's a .Net executable.

`https://www.decompiler.com` is a web-based decompiler for executables. Let's upload `UserInfo.exe` to review the source code.

Upon review, we can see that there is a function called `LdapQuery`

![image](https://github.com/user-attachments/assets/f3dfa16a-cc44-4e28-a649-2a564e1f683a)

It looks like a binary is used to connect to an LDAP server to retrieve user info.

We will append `support.htb` (the LDAP server) to our hosts file.

![image](https://github.com/user-attachments/assets/1a4cfe7c-0ec2-4e2a-a69e-aeb539604d3f)

We can see from reviewing the `LdapQuery` function that the PW needed to authenticate with LDAP is parsed from the `Protected.getPassword()` function.

Here is the `Protected.getPassword()` function:

![image](https://github.com/user-attachments/assets/5380a301-f56c-4f06-95be-7fa6e0604733)

The function lays out the decryption process for the password (basically a double XOR) as such:
- `enc_password` string is Base64  decoded from Base64
- The string is placed into a byte array
- `array` is set to the same value as `array2`
- A loop walks through each character in `array` & XORS it with one letter of the key ("armando")   --> This would be the first XOR
- A loop walks through each character in `array` & XORS it with the hex byte `0xDFu` (value 223)
- The decrypted key is converted back to a readable string using the systems default encoding

Let's make a python script that will decode this: 

The script is saved as `HTB_Support_Decoder.py`

![image](https://github.com/user-attachments/assets/5207ff03-e461-40ab-8c16-5fe8847ab85d)

Let's run the script to decode the `enc_password` string.

![image](https://github.com/user-attachments/assets/74df44ce-a57e-4210-ac4a-5893ffe2ca68)

Now that we know the password, lets try connecting to the LDAP server with `ldapsearch`
- H (host URL)
- D (Bind DN) --> the user account to authenticate as 
- w (password)
- b (search base) --> starting point in the directory tree

![image](https://github.com/user-attachments/assets/4f08ddbf-3117-4a4d-838d-730390cb7ead)

Successfull connection.

# More to come...






