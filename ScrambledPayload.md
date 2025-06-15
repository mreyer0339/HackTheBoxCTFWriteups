# WRITEUP

HTB Challenge: ScrambledPayload

# STEPS

Let's download the provided .zip file and unzip it with the given PW (hackthebox).

![image](https://github.com/user-attachments/assets/069dd444-0e7c-44e7-9eb5-c12bdc42d6ff)

We are given  `payload.vbs`, indicating a Visual Basic file.

Let's try and run it.

![image](https://github.com/user-attachments/assets/90808ebe-f3bf-4567-9a71-e287594e88f7)

Permission denied.

Using `chmod +X payload.vbs` and retrying we get the following:

![image](https://github.com/user-attachments/assets/ea9a2690-124b-4fa1-a7b9-745ea3461fbc)

Let's take a look at the file in a text editor.

![image](https://github.com/user-attachments/assets/faa99890-0e3f-4ee5-96d8-2f54b8d7b0c5)

Given clues such as certain character usage, length of the string, and common padding at the end of the string "= or ==", we can probably infer this is Base64 encoded. Let's test it.

First, we'll copy the string inside the quotes, then run it through a Base64 decode command. `echo "<string>" | base64 -d` and get the output printed to screen.

The output has the following at the beggining and end:

![image](https://github.com/user-attachments/assets/c3a78ad1-b430-4648-8c5f-2cf3e8c43a5f)

![image](https://github.com/user-attachments/assets/0ab721bd-d0aa-4f4e-8913-e3aa0732d85e)

Let's break down what this means:
- d= ""
  - Sets an empty string 
- for i = 0 to length of array
  - Loops over each part of array
- Chr(...)
  - Converts # to ASCII
- Array(...)(i)*23) mod 256
  - Takes value from array and applies an offset (*23) before wrapping into byte space (mod 256)
  - Byte space is used to normalize values to the ASCII range
- d = d + Chr(...)
  - Append each decoded character to the string "d"
- Execute d
  - Run the decoded string "d"
 
We need to intercept the generated code "d" before `Execute` runs it.

Let's change the code with `nano`. I'll replace `Execute d` with the following
- `Set fso = CreateObjected("Scripting.FileSystemObject")`
  - Create a FileSystemObject (giving us access to file methods like `.CreateTextfile`, `.OpenTextFile`, `.DeleteFile`, etc.
- `Set f = fso.CreateTextFile("readable.txt", True)`
  - Create new text file "readable.txt"
  - `true` indicates to overwrite the file if it already exists
  - The resulting object is stored in variable `f`
- `f.write d`
  - Writes contents of `d` to the file
- `f.close`
  -close file 
