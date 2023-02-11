# Revisiting a "Bad" CPU.

In my [last blog](https://dbalsom.github.io/blog/2022/12/25/Chasing-a-bad-CPU.html), I discussed encountering a bad CPU during my testing with it [connected to an Arduino](https://github.com/dbalsom/arduino_8088/).

Something was nagging me though, so I took this "Bad" CPU and put it into my IBM 5150.

It booted up straight away, and ran through all of 8088mph with no issues. That sort of puts the kibosh and the whole failed CPU theory, certainly.  But what explains the failure to calculcate effective addressing modes that I was seeing on the Arduino?

## Dynamic Logic

Just like DRAM must be refreshed constantly, sometimes we have dynamic logic circuits on a CPU that must be refreshed periodically. One reference to this I found online is https://web.archive.org/web/20201129040032/https://jamiestarling.com/project-8088-the-8088-cpu-pinout/ which states, 
>the 8088 registers are made from dynamic memory cells – they have to be refreshed. The minimum clock speed for any 8088 is 2Mhz. The max depends on the chip you are using.

On the Arduino, we are clocking the 8088 much slower than 2Mhz, certainly. Much lower, probably on the order of some kilohertz, although I haven't taken an exact measurement. Okay, so some of the registers on the 8088 used for address calculation were losing their state.
The real question then is why switching to the AMD version of the 8088 seems to work fine at those lower speeds.  The AMD part is a 2nd-source fabricator, from what I understand the masks should be identical, just a different manufacturing line. The part is D8088 and there's no indiciation it is a CMOS part. Its [datasheet](https://richierhombus.space/happytrees/datasheets/datasheet-AMD--8088.pdf) indicates a maximum cycle time of 500ns or 2Mhz. The chip dates are 1982 vs 1978, however, so there's the possibility of any number of design changes, possibly ones that rely on dynamic logic less. Who knows.

I have put the AMD 8088 through significant futher testing against my emulator, cycle by cycle, and have not encountered any further abberations that would make me suspsect it was having issues being clocked extremely slowly.  If you do not clock it at all, however, it will hang up and need to be reset, but this does not happen for a signficant time in cpu terms, probably several milliseconds.

The difference to me is quite interesting, and if anyone has any suggestions I'd love to hear what might explain this. You can avoid the issue entirely by using a fully CMOS version of the 8088, such as the 80c88 made by Harris, Intersil and others. From what I understand, CPUs manufactured on a fully CMOS process do not have any dynamic logic gates; in theory, you could single-step them forever. There is an open question whether they have the same exact cycle timings; I have ordered one off eBay for testing. If I find any differences I will publish an entry on it.
