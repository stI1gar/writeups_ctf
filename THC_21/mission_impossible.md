# Writeup Mission Impossible

Creator of the challenge : Cryptax

Writeup author : stI1gar

## Presentation
This is a writeup on the Android reverse challenge "Mission Impossible", designed by Cryptax.
I mostly used static analysis tools to solve this challenge.

## Launching the application
The first step was to launch the app in an Android x86 VM, in Virtualbox.
We come across a music player, which plays the famous main theme of "Mission Impossible" !
There are 3 buttons to respectively : start, pause and stop the music.

## Decompilation and analysis
To understand better how this application works, I used Jadx, a decompilation tool.
As we can see, the code present in the application achieves exactly what is expected : a MediaPlayer object allows us to play audio files, and there are three buttons that control the latter.
I decided to study the app's resources, ie : Mission Impossible's famous theme song !

## MP3 analysis
I used two tools (on Linux based distros, eg Debian) which are particularly useful for steganography challenges : 
- first : 'binwalk' (firmware analysis, etc), but it does not detect any hidden files,
- second : the 'strings' command : good news, the output looks like Dalvik (smali) bytecode !
    
       $ strings MissionImpossibleTheme.mp3

So I'm now assuming that a dex file is hidden within this mp3. But how is the audio file still playable ?
Using the "hexdump" command, we notice that there is a "TAG" string, which precedes the words "Mission Impossible": these are the metadata of the file (here, the title), which are normally at the end of the file according to the standard. We can therefore hide some data after the metadata of the mp3 file, while keeping the audio file readable !This additional data is simply ignored.

However, right after the metadata : we spot the beginning of a dex file header, which is simply the "dex" string. The corresponding offset in the file is : 0x32d770.

    $ hexdump -C MissionImpossibleTheme.mp3 | grep dex
    0032d770 64 65 78 0a 30 33 35 00 dc 8f a7 54 32 37 50 a3 | dex.035 .... T27P. |

However, the total file size is: 3,391,556 bytes.

    $ wc -c MissionImpossibleTheme.mp3
    3391556

With this information, we can calculate the number of bytes to extract from the file starting from the end, i.e.: 59604.

    $ echo $ ((3391556 - 0x32d770))
    59604

We can finally use the tail command to extract the dex file :

    $ tail -c 59604 MissionImpossibleTheme.mp3> extracted.dex

## Analysis of the extracted dex
We decompile the extracted dex using Jadx. Note that there are two Java classes:

- **IMRead**: allows AES 128 encryption / decryption in GCM mode, with constant keys and IV, and base64 encoding.
- **smalldex**: contains information on the flag (the analysis is more difficumt due to obfuscation / redundant code).

By studying the smalldex class, we come across the following string, which is most likely the encrypted flag also encoded in base 64:
    
    IkUegPuai + gfBce7nTfCkMZzZSwne3X3mnyrc5oBcD2yGHUXyMMcjCaXX2AAY20H

## Python script
I wrote a small Python script for this, which gives us the flag:

    from Crypto.Cipher import AES
    import base64

    # encrypted and encoded flag
    ciphertext = "IkUegPuai+gfBce7nTfCkMZzZSwne3X3mnyrc5oBcD2yGHUXyMMcjCaXX2AAY20H"

    # key and iv
    key = b'd0_you_acc3pt_it'
    iv = b'your_m1ssi0n'

    def decrypt():
        # decode the ciphertext from base64
        enc = base64.b64decode(ciphertext)

        # decrypt the ciphertext
        cipher = AES.new(key, AES.MODE_GCM, iv)
        dec = cipher.decrypt(enc)

        # print the decrypted secret
        print("flag : ", dec)

    decrypt()
    
## Dynamic analysis
A great trick from Cryptax herself : you can directly execute the dex file with the following command ;

    adb shell dalvikvm -cp /sdcard/simple.zip thcon21.ctf.payload.smalldex

By putting "MissionImpossible" as an argument, it directly gives you the flag !

## Flag
We finally find : THCon21{Th1s-Was-Poss1ble-For-U} !
