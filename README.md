# Fast3890-exploit

This exploit uses the Cable Haunt vulnerability to open a shell for the Sagemcom F@ST 3890 (50_10_19-T1) cable modem.

The exploit serves a website that sends a malicious websocket request to the cable modem.
The request will overflow a return address in the spectrum analyzer of the cable modem and using a rop chain start listening for a tcp connection on port 1337.
The server will then send a payload over this tcp connection and the modem will start executing the payload.

The payload will listen for commands to be run in the eCos shell on the cable modem and redirect STDOUT to the tcp connection.

## Running the exploit

**Note: Windows 10 is not currently supported, you must use a Linux based OS**

install pwntools and flask for python3 and run `python exploit.py`.
now go to http://127.0.0.1:8080 to exploit the cable modem.

Now a interactive shell should pop in your terminal running the python script.
If you exit the shell the modem needs to be rebooted to start a new shell.

## building your own payload

if you want to compile your own payload you can grab the toolchain from [aeolus](https://github.com/Broadcom/aeolus) and run the following command:

```
/<toolchain Path>/gnutools/mipsisa32-elf/bin/mipsisa32-elf-gcc -O3 -c ./reverseshell.c -o ./reverseshell.o && /<toolchain Path>/toolchains/gnutools/mipsisa32-elf/bin/mipsisa32-elf-objcopy -O binary reverseshell.o exploit.raw
```

## building exploit for another cable modem

You can build and exploit any modem vulnerable to Cable Haunt using this technique. 
Go to [Cable Haunt](https://cablehaunt.com) for a list of known vulnerable cable modems.

### building the ROP chain

First you will need the firmware for the cable modem you're gonna exploit.
Then reverse engineer the firmware to find the addresses of relevant function for building the exploit such as socket(), bind(), accept(), listen(), recv() and connect() for a revers shell.
Then [Ropper](https://github.com/sashs/Ropper) can be used to find gadgets using the following command:

```
python Ropper.py --type all --all --badbytes 00c0c1f5f6f7f8f9fafbfcfdfeff2c -r -a MIPSBE -I 0x80004000 --console -f <firmware path>
```

### Unreachable gadgets

Not all gadgets can be reached as gadgets is just addresses send as raw bytes and not all bytes can be send as a websocket request with text frame.
fx the address 0x8080a864 can not directly be inserted in a websocket text frame as it is not a valid utf-8 character. but 0xf28080a864 can as 0xf2 means start new utf-8 symbol with 0x8080a864 as the trailing bytes. 
The following bytes cannot be insert no matter what because of the utf-8 specification: 0x00, 0xc0,0xc1,0xf5, 0xf6, 0xf7, 0xf8, 0xf9, 0xfa, 0xfb, 0xfc, 0xfd, 0xfe, 0xff, 0x2c.
You can use the `utf8TestScript.py` to test if a address if reachable.

### custom payload

You can use the process described above to compile your own payload for new cable modem but remember that the payload must not return! If it returns the process will look for a return address on the smashed stack and crass.

## Extracting firmware from Cable Modems

For most modems you will need a way to access the eCos shell to extract the firmware or extrating it directly from the flash chip.
These firmwares are usually packed in a format called [ProgramStore](https://github.com/jclehner/bcm2-utils/blob/master/FIRMWARE.md)
If the serial console is not working try running. (might only work if coax cable is disconnected or not povisioned)
```
snmpset -v2c -c private 192.168.0.1 1.3.6.1.4.1.4413.2.2.2.1.9.1.2.1.0 i 2
snmpset -v2c -c private 192.168.0.1 1.3.6.1.4.1.4413.2.2.2.1.9.1.2.1.0 i 0
snmpset -v2c -c private 192.168.0.1 1.3.6.1.4.1.4413.2.2.2.1.9.1.2.1.0 i 2
```
Then the shell should be accesable, and [bcm2-utils]( [https://github.com/jclehner/bcm2-utils) can make it easy to extract it using this shell.
When reversing the extracted firmware the load/base address is (as far as we have seen) always 0x80004000.
