# 7z-kaboom

## 7zip extract bug explained

Ever wondered why 7zip is sometimes horribly slow to extract archives?
It has a problem with extracting into large directories with
lots of files/dirs in them.


## Why is this?

The 7z process is fully reading the filename listing of target directory
before each single file to be extracted from the archive. When the
number of files in the subdirectory is growing, 7z extraction process
will slow down linearly. After each new extracted file, it will
slow down more.

In this git repo, there is an example file 7z-kaboom.7z
that you may try to extract with 7zip if you want. It contains just
100 000 files with random filenames, each containing just 1 KB
of null bytes. It is a small 7z file that takes very long to extract,
given that there is only total 100 megabytes of data in the files.
This has been made just to demonstrate the extraction behaviour of 7z.

The reason for this behaviour is easy to reveal by running strace
on the 7z extract process, and we can see output similar to this:

```
umask(0)                                = 022
umask(022)                              = 0
readlink("hugedir/bddb7b5d4886b0cdb330f099498072f7", 0x1effd44, 1024) = -1 ENOENT (No such file or directory)
open("hugedir/bddb7b5d4886b0cdb330f099498072f7", O_WRONLY|O_CREAT, 0644) = 4
write(4, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
close(4)                                = 0
stat("hugedir/bddb7b5d4886b0cdb330f099498072f7", {st_mode=S_IFREG|0644, st_size=1024, ...}) = 0
utime("hugedir/bddb7b5d4886b0cdb330f099498072f7", [2017/04/25-21:20:23, 2017/04/25-20:30:53]) = 0
lstat("hugedir/bddb7b5d4886b0cdb330f099498072f7", {st_mode=S_IFREG|0644, st_size=1024, ...}) = 0
chmod("hugedir/bddb7b5d4886b0cdb330f099498072f7", 0644) = 0
write(1, "Extracting  hugedir/bddb7b5d4886"..., 53) = 53
mkdir("hugedir", 0700)                  = -1 EEXIST (File exists)
open("hugedir", O_RDONLY|O_NONBLOCK|O_DIRECTORY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFDIR|0700, st_size=1483880, ...}) = 0
getdents(4, /* 586 entries */, 32768)   = 32752
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
getdents(4, /* 585 entries */, 32768)   = 32760
...LOTS of similar lines deleted.
```

The first 10k to 20k files will be extracted at a relatively normal speed,
but after some 50k files have been extracted, the process is slowing
down to a crawl.

You may create your own hard-to-extract 7z archive
like this, for example:

    mkdir hugedir
    cd hugedir
    for i in `seq 100000`; do fn=`dd status=none if=/dev/urandom bs=1024 count=1 | md5sum | awk '{print $1}'`; dd status=none if=/dev/zero of=$fn bs=1024 count=1; done
    cd ..
    7z a 7z-kaboom.7z hugedir

Yes, I know it is clumsy and a small Python script would have done it much faster.


## TODO

I will soon include an ugly fix for the 7z source code as a workaround.


## Version data

Last updated: 2017-04-25

7zip version tested:

    7-Zip [64] 9.20  Copyright (c) 1999-2010 Igor Pavlov  2010-11-18
    p7zip Version 9.20 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,4 CPUs)


BR, -sjm
