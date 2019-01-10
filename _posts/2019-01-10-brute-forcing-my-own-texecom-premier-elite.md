![Wintex running on Linux](/assets/alarm/texecom.png){:width="100%"}

I live in a new build house, which came with a Texecom Elite 48 alarm system fitted. After a couple of years of use I got an error code I couldn't reset with my "master user" code. A few emails to the company who installed the alarm let me know that they had helpfully not handed over the alarm when they commissioned the system, and then deleted the code when I didn't pay for extended service. Not impressed!

They let me know they could come out (for a big fee) and reset the entire alarm system, including the codes. By this point though I was grumpy with them, and determined to show them there was an alternative. What I really wanted to do was email them and tell them the code _they_ had set on _my_ system.

My suspicion was the device wouldn't have any effective protection against brute-forcing at the serial/UDL layer - and that even if it did I was in no worse a position that I had been already so I started work.

First I needed to get access to the device itself, so having removed the case I tested with a FTDI cable I had lying around using the excellent instructions from [this blog](https://cybergibbons.com/alarms-2/programming-a-texecom-premier-elite-12-w-using-a-ftdi-cable/).

Almost immediate success, but I quickly found it difficult to use a laptop leaning into the cupboard under the stairs where the alarm is. I wanted to control it from my Linux PC upstairs.

Another [excellent blog](https://gw0udm.wordpress.com/2015/04/16/texecom-com-ip-controller-diy-and-cheap-too/) suggested a raspberry pi zero, so I dug one out the cupboard and I thought I'd be ready to start. However around this time my FTDI cable stopped working (probably because of [FTDIgate](https://hackaday.com/2014/10/22/watch-that-windows-update-ftdi-drivers-are-killing-fake-chips/) so I was temporarily set-back. I had spare 3.3v cables but not the 5v that the alarm demands so I ordered a couple of multi-voltage ones from Ebay (I reckoned there was a decent chance each wouldn't work) and looked for temporary alternatives.

I knew the GPIO pins on the pi were meant to be 3.3v but I thought there was a chance that they'd work on the alarm. I went through the usual fuss to disable the serial console and enable the serial cable and figure out which of the two ports it was running and, and started getting some promising but slightly intermittent results. Because of what I wanted to do I decided to wait for the 5v FTDI cable which would be more reliable.

When it arrived I set up the PI with `ser2net` running as discussed and was quickly able to connect from Wintex running on my Windows device over my home wifi network. However I needed to brute force the connection I would need to figure out the protocol, so I enabled `ser2net` logging and watched what happened as I tried to log on with a bunch of guessed UDL codes. For example the code 1801 generated the following.

> Wintex: `03 5A A2 07 5A 31 38 30 31 D4 07 4F 00 16 78 01 1A`

> Alarm: `0B 5A 05 01 00 00 01 05 09 00 85 03 06 F6 03 0F ED`

Unable to see in what order these characters were being sent and received I dropped into Java on my Linux box and replayed the sent codes in order, with a 50ms delay after each. The let me see the composition of the requests and responses

> Send 1: `03 5A A2`

> Response 1: `0B 5A 05 01 00 00 01 05 09 00 85`

> Send 2: `07 5A 31 38 30 31 D4`

> Response 2: `03 06 F6`

> Send 3: `07 4F 00 16 78 01 1A`

> Response: 3 `03 0F ED`

With a little experimentation I could see that 'Send 2' contained the UDL code in ASCII with two bytes leading and one trailing. But what were they? With enough examples I figured out that the trailing digit was a checksum (the sum of each of the digits in the UDL code, subtracted from 222 for four digits codes and 124 for six digit codes). The first two digits were simply the length of the UDL + 3, and then `5A`. At this stage I could see that Send 1 and 2 followed the same pattern of total length of string followed by `5A` so I felt confident I was on the right track.

I knocked up some ugly but effective code, which let me take arbitrary 4 or 6 digit guesses and generate the codes for 'Send 2'

```java
private String makeHexString(int num, int len) {
  Integer[] array = getIntAsIntArrayOfDigit(num);
  int sum = 0;
  for (int i = 0; i < array.length; i++)
    sum += array[i];
  int checksum = 222 - sum;
  if (len == 6)
    checksum -= 98;

  return String.format("%02d", len + 3) + " 5a "
      + toHexString(("" + String.format("%0" + len + "d", num)).getBytes()) + Integer.toHexString(checksum);

}

private Integer[] getIntAsIntArrayOfDigit(int test) {
  int temp = test;
  ArrayList<Integer> array = new ArrayList<Integer>();
  do {
    array.add(temp % 10);
    temp /= 10;
  } while (temp > 0);
  return (Integer[]) array.toArray(new Integer[array.size()]);
}
```

From here it was easy, iterate through all 4 and 6 digit codes (I was pretty sure it was one of these) and wait to see if any of the responses changed.

I ran it for a few hours and started getting different results. I couldn't log on directly with the code I had found, and suspected I may have hit a defence against brute-forcing, but at over 3000 codes tried at that stage I thought it unlikely. I left the alarm for 24 hours to see if it recovered but I didn't so I removed the batter and powered it down. I re-ran my code from a few codes before where it stopped and it quickly halted. I tried the new number it had stopped on and boom I was in. :-)

I must admit I logged in and out a few times to prove I had it, before taking great pleasure in emailing the alarm company to let them know I didn't need their expensive call-out and that I knew their code, in case they had used it for other premises.

![Pi connected to texecom alarm picture](/assets/alarm/pi.jpg){:width="100%"}

It was just a matter now if tidying up. I changed the codes and a few settings. Ordered some unofficial proximity tags (now I had full control of my own system) and reset the warnings. I also decided that I'd like to keep remote control of my alarm so decided to keep the raspberry pi connected. I gently closed the case over the FTDI cable (which was tricky) and left it hanging. I also installed Wintex on Linux using wine and playonlinux (no problems).

That's pretty much it. I plan to go back and test the GPIO pins again and see if i can liberate my FTDI cable, and ideally I'd draw power from the 12v inside the alarm so I can leave the pi inside rather than outside the case. (I've ordered a 12v to 5v adapter for a couple of quid to do that with)

Ideally I'd like to remotely monitor the alarm from my phone, but that's a whole new challenge.
