# 7z-kaboom

## 7zip extract bug explained

Ever wondered why 7zip is sometimes horribly slow to extract archives?
It has a problem with extracting into large directories with
lots of files/dirs in them.


## Why is this?

The 7z process is always reading the full listing of target directory
before each single file to be extracted from the archive. When the
number of files in the subdirectory is growing, 7z extraction process
will gradually but quickly slow down. And after each new extracted file,
it will slow down a bit more.

In this git repo, there is an example file 7z-kaboom.7z
that you may try to extract with 7zip if you want. It contains just
100 000 files  each containing just 1 KB of null bytes.
It is a small 7z file that takes a veeery long time to extract.

This has been made just to demonstrate the extraction behaviour of 7z.

The reason for this behaviour is easy to reveal by running strace
on the 7z extract process, and we can see output similar to this:

```
mkdir("hugedir", 0700)                  = -1 EEXIST (File exists)
open("hugedir", O_RDONLY|O_NONBLOCK|O_DIRECTORY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFDIR|0700, st_size=295980, ...}) = 0
getdents(4, /* 1027 entries */, 32768)  = 32760
getdents(4, /* 1026 entries */, 32768)  = 32752
getdents(4, /* 1026 entries */, 32768)  = 32752
getdents(4, /* 1026 entries */, 32768)  = 32744
getdents(4, /* 1026 entries */, 32768)  = 32744
getdents(4, /* 1026 entries */, 32768)  = 32752
getdents(4, /* 1026 entries */, 32768)  = 32752
getdents(4, /* 1026 entries */, 32768)  = 32752
getdents(4, /* 1026 entries */, 32768)  = 32760
...
...LOTS of similar lines deleted.
...
close(4)                                = 0
umask(0)                                = 02
umask(02)                               = 0
readlink("hugedir/f23314", 0x11df214, 1024) = -1 ENOENT (No such file or directory)
open("hugedir/f23314", O_WRONLY|O_CREAT, 0664) = 4
write(4, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
close(4)                                = 0
stat("hugedir/f23314", {st_mode=S_IFREG|0664, st_size=1024, ...}) = 0
utime("hugedir/f23314", [2017/04/26-14:58:49, 2017/04/26-14:54:36]) = 0
lstat("hugedir/f23314", {st_mode=S_IFREG|0664, st_size=1024, ...}) = 0
chmod("hugedir/f23314", 0664)           = 0

```

The first 10k to 20k files will be extracted at a relatively normal speed,
but after some 50k files have been extracted, the process is slowed
down to a crawl.

You may create your own hard-to-extract 7z archive
like this, for example:

    mkdir hugedir
    cd hugedir
    for i in `seq 100000`; do dd status=none if=/dev/zero of=f$i bs=1024 count=1; done
    cd ..
    7z a 7z-kaboom.7z hugedir


## The root cause

The 7z internal function:

    bool CFindFile::FindFirst(CFSTR cfWildcard, CFileInfo &fi, bool ignoreLink)

This function is apparently called from many points in p7zip code.
It takes a "wildcard" as an argument, and it is also often called
with just a filename without any wildcard characters. Nevertheless,
this function is always making a full directory scan until a filename
matching ```cfWildcard``` is found.

When extracting an archive with a very large flat directory
with at least tens of thousands of files, this process of
making full directory filename scans becomes very costly
and makes it practically impossible to extract archives
with huge subdirs with millions of files. Using an SSD does not help.
Using a ramdisk does not help either. 7zip is just burning cpu cycles
in a very useless way. **This is not an IO bottleneck problem at all!**


## The fix

A patch for  **p7zip 16.02** source code is included in this git repo.

Basically, in the function ```CFindFile::FindFirst()``` we are
checking if the given *wildcard* parameter actually contains any
wildcard characters that are ```?``` and ```*``` as usual.

If we have just a filename that is not a wildcard,
the potentially costly full directory scan is avoided.
Just a normal stat() call is made to check the existence
of the given filename.


## Version data

Last updated: 2017-04-26

7zip versions tested:

    7-Zip [64] 9.20  Copyright (c) 1999-2010 Igor Pavlov  2010-11-18
    p7zip Version 9.20 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,4 CPUs)

    7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
    p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,56 CPUs Intel(R) Xeon(R) CPU E5-2658 v4 @ 2.30GHz (406F1),ASM,AES-NI)


BR, -sjm
