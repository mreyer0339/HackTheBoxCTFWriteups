# WRITEUP

HTB Challenge: SpookyPass

# STEPS

First, download `SpookyPass.zip`, unzip it with the posted PW (hackthebox), and add execution permissions to the binary:

![image](https://github.com/user-attachments/assets/1ded8b0d-2311-4579-9aa7-79512fd532ee)

Let's run the binary `pass`

![image](https://github.com/user-attachments/assets/8924e3be-aff4-4c71-9697-defa892b91c2)

Entering a blank password and attempting the same PW (hackthebox) doesn't work and won't run the binary.

Let's run `strings ./pass` to scan the file for printable characters (executables, libraries, object files, etc.)

![image](https://github.com/user-attachments/assets/63f9a5ea-3a6d-4188-b902-73355e023a4f)

Let's try to run the binary again, using the printed PW...

![image](https://github.com/user-attachments/assets/85606f9b-b417-4c7b-a935-a90822f67c3e)


We were returned an un-obfuscated string, the flag for this challenge.

FYI: This string has been altered to not reveal any full flags. If you want the flag, try it out yourself.
