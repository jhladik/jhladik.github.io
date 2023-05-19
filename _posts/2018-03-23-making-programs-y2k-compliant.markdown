---
layout: post
title:  "Making Programs Y2K Compliant: 1981 ASTCLOCK.COM DOS Program Disassembly"
date:   2018-03-23 23:23:00 -0700
categories: programming
---

![Before and after Y2K bug fix.](/assets/2018-03-23-making-programs-y2k-compliant/0.jpg)

I enjoy studying computer history and experiencing it. A program that sets time on my 1982 IBM PC had a Y2K bug in the date print out and I decided to solve it. After a very long night, I achieved success.

Recently, I became a proud owner of an IBM PC. The original PC did not come with an RTC chip (Real-Time Clock) on board. Every time you turn the computer on, the DOS prompt will welcome you with a prompt to enter the current date and time. An expansion card added the time and date capability to the PC along with other perks (serial port and 384 KB RAM expansion). To accommodate for the painful time and date setting prompt in DOS, the expansion card came with a floppy disk program that gets executed after every boot and syncs the DOS internal time and date to the hardware time of the RTC chip in the expansion card.

To my surprise, DOS 3.3 is Y2K compliant. If you are not familiar with the Y2K issue, you can [read about it on Wikipedia](https://en.wikipedia.org/wiki/Year_2000_problem).

The supplied program for automatic syncing of the hardware time of the RTC chip and DOS was working correctly; however, the program also provided a simple printout of the current time and date where the date was printed incorrectly. For example, on March 2, 2018, the program would print “Current date is 3/2/99”. It almost seems like the computer was forever stuck in the year 1999. I tried setting the date to 1998 and the program printed “/98” as expected.

I searched the internet for a newer version or a patch that would solve this issue but I was not successful. I found several new versions but none of them resolved the “stuck in 1999 printout issue”. I was ok with it for a while since it was actually setting the DOS system time correctly, it was only printing out the date wrong.

A couple of days later, I was thinking about the program. If it is setting the date in DOS correctly and only messing up the printout, it should not be too hard to fix. That being said, I had zero experience in disassembling a binary executable file.

One night, I decided to go ahead and try to disassemble and reverse-engineer the program. I used freeware called IDA (Interactive DisAssembler) by Hex-Rays. I had to tell the program which lines were data and which lines were the actual code. This was done by guessing.

After disassembling the code, I was looking for critical parts. I was looking for decimal 99 (hex 63) and found a suspicious-looking couple of lines in the code. Here is my take on the issue as I understood it from the program.

AST programmer(s) programmed the ASTCLOCK.COM to force anything higher than 99 (3 digits long) not to show in the printout of the program. This might somewhat make sense because it was 1982 and there were only two places for two decimal digits. Internally, the RTC chip onboard AST SIXPAK PLUS works with Y2K. ASTCLOCK.COM also does system calls to DOS with the Y2K date correctly. The only issue is the subroutine which converts the time from the chip into a two-digit printout.

I went through the code, disassembled it, and found the crucial place where the date gets checked against 99. AST decided to save the year in the RTC chip's memory as a one-byte quantity of the year minus 80. To make sure the year fits in a byte and to save some space for future years, 1996 becomes 96-80=16 and 2018 becomes 118(“rolls over 99”)-80=38.

The way the code maps integer values to ASCII digits is as follows. Assume our year is 1996. The value stored in the RTC chip is therefore 16. The code adds 80. When it gets to the printout subroutine, the code checks if it’s greater than 99, and then the value 96 gets divided by ten using the DIV instruction. The result of the division is stored in register AL and the remainder is stored in register AH. 96/10 = 9 remainder 6. Then the code adds ASCII offset for the character ‘0’ to these numbers and the year is now represented by ASCII digits. If we skipped the check for 99 and our year is 118 (2018), the following will happen: 118/10 = 11 with the remainder of 8. So 11 gets added with ASCII offset for ‘0’ to obtain the first character of the year and 8 gets added with ASCII offset for ‘0’ to obtain the second. This results in the following printout “;8”. We went over the ASCII ‘9’ to ‘;’ since the value is greater than 99.

This is why the programmer likely decided to force the value to 99 if greater than 99. A better solution is to subtract 100 from the value which will give us 118-100 = 18 and now 18/10 = 1 remainder 8 so the resulting characters in the printout are “18”. This way, the solution will work for a year from 1980-2099 instead of the original range of 1980-1999.

The original line of code at offset 0x017A

```
B0 63 ---> MOV AL, 63H ; force value to 99 if greater than 99
```

can be now replaced with

```
2C 64 ---> SUB AL, 64H ; subtract 100 from the value if greater than 99
```

which solves the Y2K issue.

You can use the DEBUG command which is part of DOS to patch the ASTCLOCK.COM by hand.

```
DEBUG ASTCLOCK.COM
-E017A
1088:017A B0.2C 63.64
-W
-Q
```