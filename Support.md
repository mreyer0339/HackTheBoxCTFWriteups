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

ILSpy is a popular decompiler for .Net executables
