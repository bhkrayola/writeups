---
title: picoGym - Specialer 
date: 2024-02-20
slug: /writeups/picogym-specialer
excerpt: Reading files in a sandboxed environment
author: Brian Ho
---

Specialer is a challenge from picoCTF 2023. It is in the 'General Skills' category and worth 400 points. 

# Description
> Reception of Special has been cool to say the least. That's why we made an exclusive version of Special, called Secure Comprehensive Interface for Affecting Linux Empirically Rad, or just 'Specialer'. With Specialer, we really tried to remove the distractions from using a shell. Yes, we took out spell checker because of everybody's complaining. But we think you will be excited about our new, reduced feature set for keeping you focused on what needs it the most. Please start an instance to test your very own copy of Specialer.
Additional details will be available after launching your challenge instance.

Provided in my challenge instance:
- ssh -p 49322 ctf-player@saturn.picoctf.net
- password: 483e80d4

# Exploration 
After connecting to the challenge, I first ran some basic commands to see what was going on. 
```
Specialer$ ls
bash: ls: command not found
```
Command not found. Alright, let's run `compgen -c` to list all the commands available to us. I actually found out about this command from a Linux commands video. Never thought that would come in handy. 
```
Specialer$ compgen -c
if
then
else
elif
fi
case
esac
for
select
while
until
do
done
in
function
time
{
}
!
[[
]]
coproc
.
:
[
alias
bg
bind
break
builtin
caller
cd
command
compgen
complete
compopt
continue
declare
dirs
disown
echo
enable
eval
exec
exit
export
false
fc
fg
getopts
hash
help
history
jobs
kill
let
local
logout
mapfile
popd
printf
pushd
pwd
read
readarray
readonly
return
set
shift
shopt
source
suspend
test
times
trap
true
type
typeset
ulimit
umask
unalias
unset
wait
bash
```
There are quite a few commands available, but what is significant are the `echo` and `cd` comamnds. 
Let's run `echo *` to list the files, since we can't use `ls`.
```
Specialer$ echo *
abra ala sim
```

Ok, let's cd into abra. 
```
Specialer$ cd abra
Specialer$ pwd
/home/ctf-player/abra
```
Great. It worked.

Let's run `echo *` to list the files agian. 
```
Specialer$ echo *
cadabra.txt cadaniel.txt
```
Ok. Some txt files. Now to open the contents without the availability of `cat`, with some research I found about the structure `echo "$(<filename)"` which substitutes the `cat` command.
```
Specialer$ echo "$(<cadabra.txt)"
Nothing up my sleeve!
```
# Solution

After going through all of the directories and files, I found the flag in the `ala` directory.
```
Specialer$ cd ala
Specialer$ pwd
/home/ctf-player/ala
Specialer$ echo *
kazam.txt mode.txt
Specialer$ echo "$(<kazam.txt)"
return 0 picoCTF{y0u_d0n7_4ppr3c1473_wh47_w3r3_d01ng_h3r3_d5ef8b71}
```
Nice!

# Flag
```
picoCTF{y0u_d0n7_4ppr3c1473_wh47_w3r3_d01ng_h3r3_d5ef8b71}
```
