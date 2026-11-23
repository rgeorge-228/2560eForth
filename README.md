              2560eForth 
    A Port of C. H. Ting 328eForth 
    R. D. George November 2026 
    
INTRODUCTION: 

I am indebted to C. H. Ting and his colleagues at The Silicon Valley Forth Interest Group 
(SVFIG) that brought FORTH to Microcontrollers. I started working with the Atmega 
328P several years ago and found the work of Ting’s 328eForth on the internet. His 
work with various Microcontrollers and Microprocessors was my inspiration to pursue 
the interest in digital electronics and Amateur Radio. 
After several years of projects on the 328P Microcontroller I started to add a simple file 
system using a 64K byte I2C EEPROM and then found the work of Igor Voytenko who 
had developed AVR assembly code for basic IEEE 745 floating point operations for the 
Atmega 328P Microcontroller. Adding both to the 328eForth used up over half of the 
Microcontrollers 32K of Flash memory. I had an SainSmart MEGA 2560 board and said, 
let’s port the assembler code to the larger Microcontroller (256K Flash & 8K SRAM). 
That was the start of this system. 

PORTING 328eFORTH: 

The biggest issue with the port to the 2560 was the larger memory range (256K vs. 
32K). The 328P uses 16 bit addressing and can only use 32K of that address range. 
The 2560 uses 18 bit addressing and can use the full 256K range. To accomplish this, 
the 2560 hardware uses 2 bits of a RAMPZ register to extend the addressing ELPM, 
SPM instructions. Additionally, the Call/Return instruction push/pop three registers on 
the system stack (RAMPZ, HI, LO) for a 24 bit address. All Call and Return require 3 
bytes to and from the system stack (or return stack). The 328eForth system (as all 
FORTH code) uses the return stack to store temporaries such as loop counters and 
jump addresses for flow control logic. The 328eForth assembly code was changed to 
Push-Pop 3 bytes for every Call/Return and words that save and retrieve 16 bit values 
from the return stack process 3 bytes to 2 bytes if it’s an address on the stack. 
The Ping/Pong flash buffers used by Ting’s code were increased to 256 bytes each as 
this is the page size of the 2560’s flash page. The rest of the Ping/Pong buffer code was 
changed to accommodate the larger buffers. 
The next issue was where to put the Forth dictionary. Since the 2560 is basically 4 flash 
blocks of 64K each and special attention for instruction coding is required when reading 
and writing to memory above the Block 0 flash, the Forth dictionary is placed in the first 
64K of flash memory. This puts a limit on the size of the Forth system but is much larger 
than the 32K of the 328P Foth system. All addressing can be limited to 16 bits in the first 
64K memory block which helps keep the Primitive and Basic Forth words simple. 
The highest memory block (Block 3) was set aside for booting the 2560 and assembly 
code for floating point and file handling code. The Reset Vector was reset to 1F000 and 
the Interrupt Table added there. 

BOOTLOADER: 

The bootloader for the 2560 is 8K in the block 3 memory. Because we are using that 
area for our code, the bootloader for the Microcontroller is not available. There may be a 
work around for that in a further version. All assembler programming uses a AVR ISP 
programmer and the Microchip Studio Version 7 IDE. Binary versions can be created, 
saved, and loaded with this system. 

SYSTEM REQUIREMENTS: 

The basic hardware requirements are: 
1) Atmega 2560 board with serial USB     - $10 - $15 Chinese suppliers
2) AVR ISP programmer                    - $25 - $30 Amazon 
3) Computer to run Microchip Studio and Terminal Emulator 
4) Microchip Studio 7                   - Free download 
5) Tera Term 5                          - Free download     
6) USB cables B to A                    - $5 Amazon

Figure 1 below is a example system with a computer running TeraTerm 5 and connected 
to the SainSmart 2560 board via USB with the AVR ISP programmer via USB plugged 
into the 2560 ISP header. Programming the 2560 from the Microchip Studio assembler 
Hex file will erase all the 2560’s flash, so any Forth words or disk files will be erased. In 
fact, you must run the Format command after a new Hex file is loaded unless that file 
was saved with an image that has the file structure included.
<img width="967" height="975" alt="image" src="https://github.com/user-attachments/assets/ba4f70b5-bd07-4626-8600-649b715b7b28" />
Figure 1

MINIMAL FILE SYSTEM (MFS): 

To take advantage of the extra flash memory in the 2560 Microcontroller, I created a 
virtual Disk Drive using two of the 64K memory blocks. The two middle blocks totaling 
128K were available for use. I thought that 128K is about the size of the original 5 ¼ 
floppy disks of the 1980’s. At that time this was considered a good bit of disk storage. 
The code started out as Forth words with some of the longer functions in assembler 
loaded in Hi memory. With the help of Claude AI, we recreated most of the Forth code to 
assembly thereby saving space in the Lo memory Forth dictionary. 
The MFS supports basic file I/O (Reads, Writes, Copy, Rename, Delete, Execute).  
There are utilities to print a Help screen, to print a directory of the disk and to directly 
read any sector of flash memory on the 2560. There is a Format command to create a 
new virtual disk with just one file (Hello.dat). The Format command must be run if new 
Hex code is loaded to the 2560. 
After the initial Format, Open the terminal emulator (prefer TeraTerm 5) and copy the 
Forth code file MFSforth3.txt and paste into the TeraTerm screen. This will add all the 
Forth words used by MFS code. The MFSforth3.txt ends with these lines: 
: OK1 ( -- ) $" Hello.dat" Show ; 
Flush 
SaveState 
The SaveState command will save all pointers in flash that will be reloaded to SRAM 
when the 2560 reboots. This guarantees all Forth words are available. 
The OK1 command should print the Hello.dat file from the disk drive to the terminal. 

FLOATING POINT: 

Thanks to Igor Voytenko for his work in creating AVR assembler code for IEEE 745 
floating point operations. His assembly code (GitHub - igor-240340/Float32AVR:) was 
added to the Hi memory block along with my code changes to interface to the 328eforth 
code I ported. Specifically, we added direct conversion from the terminal input of real 
decimal numbers, including exponential notation to floating point values on the data 
stack as 4 byte values. Added floating point literals in compiled words. Added floating 
point constants (i.e. 3.14597 FCON PI) and additional Forth words to make it easy to 
use floating point numbers in FORTH. Some examples are F@, F!, 2SWAP, F+, F-. F*, 
F/, F2AE, F2I16, A2F, I2F, F>. Variable and arrays are treated the same as regular Forth 
words with the exception that all FP values take 4 bytes each. Here is a quick math 
example.

\  Print Pi r^2 for a given r – to the terminal screen
3.14597 FCON Pi			\ constant Pi
: Pir^2		( fp,r – )		\ print Pi r^2
2Dup F* Pi F*		\ double r multiply then by constant Pi
   	Cr 9 F2AE Cr		\ print ASCII on new line
;
Flush					\ flush buffer to flash directory
2.0 Pir^2  will print
1.258E+01


\  Print Pi r^2 for a given r – to the terminal screen
3.14597 FCON Pi                      \ constant Pi
: Pir^2     ( fp,r – )               \ print Pi r^2
    2Dup F* Pi F*                    \ double r multiply then by constant Pi
    Cr 9 F2AE Cr                     \ print ASCII on new line
;
Flush                                \ flush buffer to flash directory
 
2.0 Pir^2        will print the value
1.258E+01  

Changing the 9 F2AE to C F2AE will give more digits in the output number. 
Most of the floating point code is in Hi memory with only “stubs” in the Forth dictionary. 
This saves lots of space for user words in the dictionary. 

INSTALLING TEST SYSTEM: 

I will illustrate installing the assembly test code and the Forth MFS code on an Atmega 
2560 board with a laptop computer and software. The assembly file “FileTest7.txt” is 
copied to the Microchip Studio 7 New Project screen and gets the name “main.asm”. 
Select Build Solution from the drop down as in Figure 2.
<img width="975" height="471" alt="image" src="https://github.com/user-attachments/assets/a1c67531-f0a5-4361-95de-9972b2c122aa" />
Figure 2

Fix any assemble errors, then ReBuild the main.asm code. When there are no errors, 
you can browse the filename.lss file to see the work of the assembler. 
Next click the little icon of an IC chip with a lighting bolt to bring up the programming 
dialog screen and change the Tool to the AVR ISP programmer Figure 3. 
<img width="975" height="568" alt="image" src="https://github.com/user-attachments/assets/a41283be-441a-45ea-8e27-1d4fe632a153" />
Figure 3 

Click the “Apply” then “Read” then “Memories” buttons to see Figure 4 below. 
<img width="975" height="566" alt="image" src="https://github.com/user-attachments/assets/a3d1f830-f630-4fe5-8725-8e4103136d3b" />
Figure 4 

If it appears that the binary Hex file is correct, click “Program” and you should see: 
Erasing device... OK 
Programming Flash...OK 
Verifying Flash...OK 
Open the Tera Term terminal emulator program and check the Com Port from Windows 
Device Manager and set up the serial port as in Figure 5. 
<img width="975" height="984" alt="image" src="https://github.com/user-attachments/assets/a069ab2f-f712-4db1-a3b6-bade38ca768c" />
Figure 5 

Be sure to set a line delay to at least 600 ms to allow for 2560eForth to process 
incoming text lines from the terminal. 
<img width="975" height="602" alt="image" src="https://github.com/user-attachments/assets/39503177-f9cd-49dd-b19b-3527e32c2fc8" />
<img width="975" height="570" alt="image" src="https://github.com/user-attachments/assets/025a74a6-e852-40ff-89a4-759787fc0d73" />
Figure 6 

Figure 6 is the data for programming the 2560 flash (Fuses & Lock bits). 
Once the Tera Term program connects to the 2560 board, use the Words command to 
get something like Figure 7 
 <img width="975" height="521" alt="image" src="https://github.com/user-attachments/assets/2889e66d-d41b-4782-86fb-d41627340476" />
Figure 7 
 
The floating point words and the assembly part of the MFS are now loaded on the 2560. 
The next step is to copy the test file MFSforth3.txt to the Tera Term program as Figure 8. 
 
After the load completes, try OK1 word. As you can see, I forgot the Format command. 
 
<img width="975" height="406" alt="image" src="https://github.com/user-attachments/assets/239fa060-41f4-4611-9a91-26b676a397ad" />
Figure 8 

Once all the code is loaded, you has a complete Beta Test System. I say Beta with a 
grain of salt. I have partially tested all new words and have tried a FFT word with CW 
data from my Ham radio. At this point we have:        
dir                FLASH DRIVE 001 
        File Name       Date Changed        Size 
        Hello.dat       09032026              512
        
Total bytes used                              512                               
Total disk size                             128512                                  
Total bytes free                            128000

?mem 
Flash  53597 
Ram     6395 
ok 

So, there are 53K of flash for Forth words and 6.4K of SRAM for variables This is plenty 
of room for developing microprocessor programs.  

ADDITIONAL CODE: 

As part of the development of this 2560eForth I used some utilities to simplify the 
pointer used in the MFS system. 
\  -------------------------- WFR 2010-10-22 ------------------------ 
\ Modified for 328eForth, 23mar11cht 
\ Extend eForth first, 22mar11cht 
: DOES R> LAST @ NAME> 2+ I! ;  
\ specific for subroutine thread code.  see VALUE and MARKER. 
: VALUE ( n --, interpret ) VARIABLE DP @ 2- ! DOES R> 2* I@ @ ; 
\ compile a pointer in flash, access data in RAM 
: [TO] ( n --, interpret ) ' 4 + I@ ! ;  
\ change value while interpreting 
: TO ( n --, compile ) R> DUP 1+ 2* I@ 2+ 2* I@ SWAP 2+ >R ! ; 
\ compiled in colon word to change value at run time 
\ marker is easier for 328eForth with only one vocabulary, 23mar11cht 
: Marker ( <new_name> -- ) 
last @ dp @ cp @ context @ 
create -2 cp +! , , , , 
does r> 2* dup i@ context ! 
2+ dup i@ cp ! 
2+ dup i@ dp ! 
2+ i@ last !  
; 
\ chop-to  
\ marker chop-to   
\ error here if first time, next cuts back 
\ create a forget point to chop back to 
\ -------------------------------------------------------------------------- 
I am not sure who WFR is, but I give credit for his utilities. 

OBSERVATIONS: 

The Atmega 2560 board gives the hobbyist more I/O than anyone could want and with a 
small Forth system it provides an excellent platform for digital project development. I set 
out to use this system to help restoring an MMD-1 8080 microcomputer system from the 
1970’s. I ended up with a more capable system, with floating point and a simple file 
system that I am presenting here. I will try to document what has been done to create 
this 2560eForth project. I want to dedicate this work to Chen Hanson Ting whose initial 
work with micro-forth created the groundwork for small systems with large capabilities. 
As is said “we stand on the shoulders of great men” and C. H. Ting is one of them. I 
wish to share the results of this work that was based  on his 328eForth code, and ported 
to a novel microcontroller system. The system presented here is far from complete at 
this time and probably contains logic bugs that I have not discovered. 
The assembler and FORTH code are presented “as is” with no guarantee as to function. 
The intent is to give others an opportunity to try this system. I have included one 
example of Fast Fourier Transform in the Appendix A. The example uses an ASCII CSV 
file loaded from the disk and converted to floating point arrays. Then the arrays are 
processed by a 256 point FFT algorithm using the floating point words defined in 
FORTH with their assembly code stored in the 2560 Hi memory. The output is a 64 point 
bin number and floating point frequency (Y values).  
I have mixed FORTH and assembler for speed and size and the ability to move words 
from the Forth dictionary in Lo memory to Hi memory to save space for new user when 
they develop applications. Additional optimizing is possible if more space in needed. 
Programming in FORTH is a different animal because of the reverse polish notation and 
the heavy use of the data stack. On the plus side, code can be tested in small bites as 
tasks are broken down into discrete pieces that become FORTH words. Defining words 
can be used to create new FORTH words/objects as with the VALUE word used 
extensively in the MFS code to track file pointers. This is basically a structure object 
collection for the file I/O. I am assuming that an interested user has a little knowledge of 
FORTH or would like to learn it. Also, AVR assembler coding would be an asset when 
looking at how the system was created. 
A very good reference for the basis of this FORTH system is the work of C. H. Ting at 
Offete Store and at https:Silicon Vally Forth Interest Group //www.forth.org/svfig/ . The 
documentation on the 328eForth by C. H. Ting covers most of the basic Forth words 
ported to the 2560eForth system. I recommend looking at the archive at Offete Store as 
it contains a great deal of Ting’s document work with microcomputers and 
microcontrollers. Work has been started on documentation of the new Forth words 
added for File access and Floating Point math. 
Along with this Word document are additional files that I used in the example code in 
Appendix A. 

Separate files are:  

    TestFile7.txt    main.asm 
    MFSforth3.txt    forth source 
    DataFile.dat     FFT data CW signal 
    FFT256.txt       FFTforth routine 
 
 
APPENDIX A: 
 
Simple Forth test code for a FFT algorithm from Claude AI: 
 
Start by loading the data file (DataFile.dat) from a CSV file with FWRITE: 
 
fwrite DataFile.dat 
 Enter New Date 09012026 or CR > 11212026 
Time,Value 
4464,0 
4619,0 
4774,0 
4913,0 
5068,18 
5223,37 
5378,51 
5533,55      ….. to end and close with ESC 
 
Next load the FFT256.txt code file to the virtual disk: 
 
fwrite 
        SYNTAX: COMMAND FileName.Ext [-A] 
ok 
fwrite FFT256.f 
 Enter New Date 11212026 or CR > 
\ Test of FFT code October 30 2026 
\ Use FFT DataFile.dat -- comma seperated data file X,Y to 256 lines 
\ 
Hex 
Marker chopfft 
100 Constant Pts 
Variable XR  3FE Allot   \ 256 floats 
Variable XI  3FE Allot 
Variable WR  1FE Allot   \ twiddles, 128 floats 
Variable WI  1FE Allot 
Variable Wc 2 Allot  Variable Ws 2 Allot 
Variable Tr 2 Allot  Variable Ti 2 Allot  Variable Tf 2 Allot 
Variable Cbuf 1E Allot 
Variable Mn 2 Allot 
0.9996988 FCON Cw   \ cos(2 pi/256) 
0.0245412 FCON Sw   \ sin(2 pi/256) 
00 value Np  00 value Lg  00 value Ns 
00 value Half  00 value Strd  00 value St  00 value Kk 
00 value Ix  00 value Jx  00 value Pa  00 value Pb 
00 value Lend  00 value Fs 
: FNEG ( fp -- -fp ) Swap 8000 Xor Swap ; 
: El ( base i -- a ) 2* 2* + ; 
: Fx ( a b -- )   \ swap the floats at a and b 
to Pb to Pa  Pa F@ Tf F!  Pb F@ Pa F!  Tf F@ Pb F! ; 
: WStep ( k -- )   \ W[k] = W[k-1] * W1 
>R 
WR R@ 1- El F@ Cw F*  WI R@ 1- El F@ Sw FNEG F*  F- 
WR R@ El F! 
WR R@ 1- El F@ Sw FNEG F*  WI R@ 1- El F@ Cw F*  F+ 
WI R> El F! ; 
: MkTw ( -- ) 
1 I2F WR F!  0 0 WI F! 
7E For 7F R@ - WStep Next ; 
: Rev ( i -- j )   \ reverse the low Lg bits 
0 Swap 
Lg 1- For 
Dup 1 And Rot 2* Or Swap 2/ 
Next 
Drop ;   ……….. to end and close with ESC 
Notice my error where the file name was not entered for the FWRITE Command. 
Check the DIR command to see files loaded. 
dir         FLASH DRIVE 001 
     File Name      Date Changed      Size 
     Hello.dat       09032026            512 
     DataFile.dat    11212026           1024 
     FFT256.f        11212026           4096 
 
Total bytes used                        5632 
Total disk size                       128512 
Total bytes free                      122880 
Ok 
 
 
Now Compile the FFT256.f using the FEVAL command:  
Feval FFT256.f 
ok This took several seconds to compile. If you want to watch the load and compile 
line by line, use the -E switch after the FEVAL command. 
 
Now use the WORDS command to see new words defined: 
words 
 FFT ANAL DCOFF SPEC .BIN PWR SIZE LOAD VAL NUM? LASTFLD XFORM 
STAGE BFLY BITREV REV MKTW WSTEP FX EL FNEG FS LEND PB PA JX IX KK ST 
STRD HALF NS LG NP SW CW MN CBUF TF TI TR WS WC WI WR XI XR PTS 
CHOPFFT OK1 PDUMP FRENAME FCOPY GETNAME FDELETE FWRITE RUN 
SHOW FEVAL FREAD GO NEWFILE XFER PUTLINE PUTEVAL GETLINE GETTXT 
PUTTTY PAUSE >FNAME IMOVE >FILE >DATE ?MEM ?OPTION DEL.NUM 
MFS.CNT MFS.NUM MFS.ATT DIR.E 
 
Now run the FFT word and it prints the 64 item bin data array X and Y: 
fft DataFile.dat 
 0  5.820765E-11 
 1  7.284131E+03 
 2  7.235551E+02 
 3  1.163712E+04 
 4  3.062994E+03 
 5  2.629431E+02 
 6  1.657620E+04 
 7  1.348385E+04 
 8  4.534468E+01 
 9  7.736566E+04 
 10  2.704343E+05 
 11  1.608946E+06 
 12  2.540606E+05 
 13  6.709603E+04   …… to  63  2.751140E+02 
 
Peak is in Bin 11 @ 1,608,946 which is a frequency of 1120 Hz from my received CW 
audio. You can capture the output with the Tera Term Log options if desired. 
 
The FFT256 Forth code uses lots of SRAM for floating point arrays. So, the utility 
Marker ChopFFT if run at the start of the compile can be used to reset DP and CP and 
recover all this space as below. 
 
?mem 
Flash  50723 
Ram     3241 
ok 
chopfftok 
?mem 
Flash  53597 
Ram     6395 
ok 
 
The compiled words from FFT256.f can be added back by just using the FEVAL 
FFT256.f command again. 
 
APPENDIX B: New 2560eForth Commands 
 
Stack Abbreviations of Number Types 
flag     Boolean, 0 or -1 (false or true) 
char     ASCII character or 8 bit byte 
n       a 16 bit number 
fp      A 4 byte Floating Point number 
d       a 32 bit number 
addr    a 16 bit address 

 
Floating Point Operations 
F!           (fp addr -- )     Save fp at addr (4 bytes) 
F@           (addr – fp)       Replace addr with fp at addr (4 bytes) 
FCON <name>  (fp -- )          Defines FP constant, at run time fp 4bytes is 
                               Left on the stack 
2SWAP        (fp1, fp2 – fp2, fp1)   Swap floating point numbers 4 bytes each 
F+           (fp1, fp2 – fp3)   Adds two FP numbers for FP result 
f_           (fp1, fp2 – fp3)   Subtracts fp2 from fp1 for FP result 
F*           {fp1, fp2 – fp3)   Multiplies fp1 by fp2 for FP result 
F/           (fp1, fp2 – fp3)   Divides fp1 by fp2 for FP result 
F>           (fp1, fp2 – flag)  Flag is true (-1) If fp2>fp1 
F2AE         (fp, n -- )        Prints ASCII string length n as N.NNNE+-NN 
F2I16        (fp – n)           Truncates fp to Integer n (16 bits) 
A2F          (addr – fp)        Converts ASCII at addr to FP 
I2F          (char – fp)        Converts char (8 bits) to FP 


File Operations 
FWRITE        ( -- ) 
<name>|<-A>                     Takes terminal input and stores in <name> 
                                with optional -A append  
FREAD         ( -- ) 
<name>|<-P>                     Outputs to terminal file <name> with optional 
                                P for pause 
FEVAL         ( -- )
<name>|<-E>                     Executes (Eval) file with Forth compile or 
                                interpret, optional -E echo process
FCOPY <file1> ( -- ) 
<file2>|<-A>                    Copies <file1> to <file2> with optional -A for 
                                append <file1> to <file2> 
FRENAME       ( -- ) 
<file1> <file2>                 Rename <file1> to <file2> just changes names 
                                in the directory
FDELETE       ( -- )
<name>                          Will delete <name> file data from virtual disk 
                                drive 
DIR           ( -- )            Print a directory of files on the disk drive 

HELP          ( -- )            Print a help message of file commands 


Utilities 
IDUMP         (addr -- )        Dumps 256 bytes from Flash at addr adds the 
                                block address to the dump 
FDUMP          (n -- )          Dumps 256 bytes from Flash at sector n 
RDUMP          ( -- )           Formatted dump of all 32 registers 
FORMAT         ( -- )           Format the virtual disk, used only once 
?MEM           ( -- )           Prints Flash and SRAM memory available  
“S <file>”     ( -- )   
RUN                             Compiled <file name> at run time RUN calls 
                                FEVAL to execute the code in <file> 
“S <file>”     ( -- ) 
SHOW                            Compiled <file name> at run time SHOW 
                                prints <file> to terminal  
 
END:      
                                Robert George   November 2026 
