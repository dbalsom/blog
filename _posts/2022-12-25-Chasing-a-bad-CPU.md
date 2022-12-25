---
title: "Chasing a bad CPU"
date: 2022-12-25
---

I've been working on a project involving controlling an Intel 8088 CPU via an Arduino Mega, which I eventually was going to make a blog post about on its own. For now, see https://github.com/dbalsom/arduino_8088 for a background.

However during development of this project I encountered an interesting situation I thought was worth writing some notes about.

Currently, my Arduino sketch can execute extremely simple programs on the 8088 and print instruction traces. It looks like this:

```
## Changing to state: Execute. Spent (92) us in previous state.
00000265 E A:[FCCCC]    M:... I:... CODE t1 \         |  0 [        ]
00000266 E   [FCCCC] CS M:R.. I:... CODE t2  | <-r 02 |  0 [        ]
00000267 E   [FCCCC] CS M:R.. I:... PASV t3  | <-r 02 |  0 [        ]
00000268 E   [FCCCC] CS M:... I:... PASV t4 /         |  0 [        ]
00000269 E A:[FCCCD]    M:... I:... CODE t1 \         |  1 [02      ]
00000270 E   [FCCCD] CS M:R.. I:... CODE t2  | <-r 01 | F0 [        ] <-q 02 ADD
00000271 E   [FCCCD] CS M:R.. I:... PASV t3  | <-r 01 |  0 [        ]
00000272 E   [FCCCD] CS M:... I:... PASV t4 /         |  0 [        ]
00000273 E A:[FCCCE]    M:... I:... CODE t1 \         |  1 [01      ]
00000274 E   [FCCCE] CS M:R.. I:... CODE t2  | <-r 90 | S0 [        ] <-q 01
00000275 E   [FCCCE] CS M:R.. I:... PASV t3  | <-r 90 |  0 [        ]
00000276 E   [FCCCE] CS M:... I:... PASV t4 /         |  0 [        ]
00000277 E A:[FCCCF]    M:... I:... CODE t1 \         |  1 [90      ]
00000278 E   [FCCCF] CS M:R.. I:... CODE t2  | <-r 90 |  1 [90      ]
00000279 E   [FCCCF] CS M:R.. I:... PASV t3  | <-r 90 |  1 [90      ]
00000280 E   [FCCCF] CS M:... I:... PASV t4 /         |  1 [90      ]
00000281 E   [FCCCF]    M:... I:... PASV t1           |  2 [9090    ]
00000282 E   [FCCCF]    M:... I:... PASV t1           |  2 [9090    ]
00000283 E A:[25000]    M:... I:... MEMR t1 \         |  2 [9090    ]
00000284 E   [25000] DS M:R.. I:... MEMR t2  | <-r 90 |  2 [9090    ]
00000285 E   [25000] DS M:R.. I:... PASV t3  | <-r 90 |  2 [9090    ]
00000286 E   [25000] DS M:... I:... PASV t4 /         |  2 [9090    ]
00000287 E A:[FCCD0]    M:... I:... CODE t1 \         |  2 [9090    ]
00000288 E   [FCCD0] CS M:R.. I:... CODE t2  | <-r 90 |  2 [9090    ]
00000289 E   [FCCD0] CS M:R.. I:... PASV t3  | <-r 90 |  2 [9090    ]
00000290 E   [FCCD0] CS M:... I:... PASV t4 /         | F1 [90      ] <-q 90 NOP
00000291 E A:[FCCD1]    M:... I:... CODE t1 \         |  2 [9090    ]
00000292 E   [FCCD1] CS M:R.. I:... CODE t2  | <-r 90 |  2 [9090    ]
## Changing to state: Store. Spent (90812) us in previous state.
## Changing to state: Done. Spent (14292) us in previous state.
```

The trace format is a bit verbose, but from left to right we have the following fields:
* Cycle number
* Sketch state (E indicates it is Executing the specified program, there are other values only important internally)
* 'A:' indicates the Address Latch Enable (ALE) signal
* [00000] are the contents of the address latch.
* CS/DS/SS/ES - active segment decoded from S3-S4
* M: and I: are outputs from the Intel 8288 bus controller, specifying Read, Write or Advanced Write (Basically Write but one cycle earlier), for Memory or IO, respectively.
* CODE/MEMR/PASV/etc: Bus transfer mode
* t1/t2/t3/t4: Bus cycle state
* <-r w-> Data bus value read or written
* F/S/E - Processor instruction queue operation: 
  * First byte of instruction read
  * Subsequent byte of instruction read
  * Queue flushed
* Length of instruction queue
* Contents of instruction queue (Tracked by the Arduino)
* Finally, queue reads are noted and instruction opcodes decoded to mnemonic.

In the trace above, we are executing the program bytes
**0x0A, 0x01, 0x90** or 
```
add al, [bx+si]
nop
```

You can see the cycle number begins at 265. This is because a load routine is run before the specified program is executed to set up the register state to specified values.
Of relevance to the instructions above, **DS==0x2000**, **BX==0x5000** and **SI==0x1000**, just arbitrary values that make it easier to see when they are added together.

What should happen this 'add' instruction executes, is the cpu should calculate a memory address by adding BX+SI, retrieve the byte at that offset from segment DS, and add it to the value in AL.

But we can see the first read operation:
```
A:[25000]    M:... I:... MEMR t1 \         |  2 [9090    ]
  [25000] DS M:R.. I:... MEMR t2  | <-r 90 |  2 [9090    ]
  [25000] DS M:R.. I:... PASV t3  | <-r 90 |  2 [9090    ]
  [25000] DS M:... I:... PASV t4 /         |  2 [9090    ]
```

Right now, there's no memory to read or write to on the Arduino, so the last value of the data bus is used, in this case it remains 0x90.

Where do we get the address of **25000**?
Let's calculate what we should see. To calculate the effective address we first take the base segment register DS, and shift over four bits, then add our address calculation offset:
```
Base(DS)<<4: 20000
        +BX:  5000
        +SI:  1000
        ----------
             26000
```
  
Hmm, that doesn't match the address of 25000 actually read. Where did the value of SI go? Even more curiously, in addressing modes that fetch an 8 or 16 bit displacement, I could see the CPU fetching the displacement bytes, and then failing to add them. Eventually I built a small table comparing the expected addressing modes and the addressing modes the CPU was actually appearing to use:

```
modrm  expected   observed
   00: ds:[bx+si] ds:[bx]
   01: ds:[bx+di] ds:[bx]
   02: ss:[bp+si] ss:[bp]
   03: ss:[bp+di] ss:[bp]
   04: ds:[si]    ds:[si]
   05: ds:[di]    ds:[di]
   06: ds:[disp]  ds:[disp]
   07: ds:[bx]    ds:[bx]

modrm  expected        observed
   40: ds:[bx+si+disp] ds:[bx+0]
   41: ds:[bx+di+disp] ds:[bx+0]
   42: ss:[bp+si+disp] ss:[bp+0]
   43: ss:[bp+di+disp] ss:[bp+0]
   44: ds:[si+disp]    ds:[si+0]
   45: ds:[di+disp]    ds:[di+0]
   46: ss:[bp+disp]    ss:[bp+0]
   47: ds:[bx+disp]    ds:[bx+0]
```

We can toss out the idea that the modrm byte is not being read; there's definitely a pattern here, and modes that specify a BP register are using the SS segment register as we would expect. It's not an issue with SI or DI being set correctly either - in modrm 0x04,0x05,0x44 and 0x45 the SI and DI values are used correctly.

It seems like the only thing that isn't quite working is that neither the index or displacement is being added properly. Very odd.

I was scratching my head for quite some time, wondering where the bug in my code was, but nothing was adding up.  Eventually, I took the CPU out of my working IBM PC 5150 and got the following memory read:

```
[0x02, 0x00] add al, [bx+si]
A:[26000]    M:... I:... MEMR t1 \         |  2 [9090    ]
  [26000] DS M:R.. I:... MEMR t2  | <-r 90 |  2 [9090    ]
  [26000] DS M:R.. I:... PASV t3  | <-r 90 |  2 [9090    ]
  [26000] DS M:... I:... PASV t4 /         |  2 [9090    ]
```

A known good CPU produces the correct result, very interesting. Can we add an 8-bit displacement of +5 as well?

```
 [0x02, 0x40, 0x40] add al, [bx+si+05]
 A:[26005]    M:... I:... MEMR t1 \         |  2 [9090    ]
   [26005] DS M:R.. I:... MEMR t2  | <-r 90 |  2 [9090    ]
   [26005] DS M:R.. I:... PASV t3  | <-r 90 |  2 [9090    ]
   [26005] DS M:... I:... PASV t4 /         |  2 [9090    ]
```

So it appears that I have a bad 8088 CPU (ordered from eBay over 90 days ago, alas) that has failed in a very particular manner. Of course, plugged into an actual computer, the effect this sort of malfunction has would almost certainly lock up the machine somewhere during BIOS initialization, so you'd never quite know your CPU was in fact *partially* working. 

One tends to think of CPU failures as an all-or-nothing affair, so this was a pretty interesting situation to encounter. When dealing with unexpected results, I almost always suspect a bug in my code rather than a hardware failure - I had a third CPU that failed to reset itself entirely, so that sort of failure mode is more expected.

Hope this writeup is interesting to someone!

