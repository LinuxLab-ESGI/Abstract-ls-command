# Abstract of the `ls` command

- [Abstract of the `ls` command](#abstract-of-the-ls-command)
  - [Some useful options and tips](#some-useful-options-and-tips)
  - [History of `ls`](#history-of-ls)
  - [How does ls work?](#how-does-ls-work)
    - [The shell](#the-shell)
    - [Checks (Aliases and PATH)](#checks-aliases-and-path)
    - [Creation of the process](#creation-of-the-process)
      - [Fork](#fork)
      - [Exec](#exec)
      - [Wait](#wait)
    - [Execution of the ls command](#execution-of-the-ls-command)
      - [`opendir()`](#opendir)
      - [`readdir()`](#readdir)
      - [`stat()`](#stat)
  - [Example with our own ls program](#example-with-our-own-ls-program)
  - [ls source code](#ls-source-code)
  - [Sources](#sources)

## Some useful options and tips

`ls` with these arguments :  

| Arguments | Description                               |
| --------- | ----------------------------------------- |
| `-l`      | More information in list format.          |
| `-a`      | Hidden files.                             |
| `-R`      | Recursive, list files in sub-directories. |
| `-lSh`    | Sort by size with human readable.         |
| `-i`      | Inodes.                                   |
| `-n`      | Numeric UID and GID.                      |

## History of `ls`

The `ls` command is one of the oldest and most important commands for Unix/Linux environments.

In its first versions it was named `listf` and was included in the Compatible Time Sharing System (CTSS, 1961) developed at MIT, whose successor was Multics (1964).
A few years later `listf` was renamed by `list` and finally by `ls` in 1969 in front of the beginning of Unix development.
It was in 1971 that the `ls` command was officially documented.

Nowadays the `ls` command we use comes from the GNU foundation.

## How does ls work?

`ls` (**L**i**S**t Directory) is a program from the coreutils package developed by the GNU foundation. It is one of the first commands we learn when we start working with Linux distributions. But how does it actually work?

### The shell

First of all, the shell is the first program that the user will encounter. It is through it that he will be able to communicate with the other programs and then they will communicate with the Kernel which will finally access the Hardware part.  
This flow of information exchange within the operating system can be represented succinctly via the following line:  

User > Shell > Other Programs > Kernel > Hardware

It is therefore via the shell that we can type the `ls` program to call and execute it afterwards.

### Checks (Aliases and PATH)

The first thing the shell will do when we enter the `ls` command is to check if an alias exists for it.  
If the alias ls exists it will replace this value with what we typed.  
If the alias does not exist then it will look for the `ls` command among the paths contained in the `$PATH` variable.  

To know all the paths specified in the path we can execute the following command:  
`echo $PATH`

To find out where the command in question is located we can run these commands:  

- `which ls` gives the path where the command comes from.  
- `whereis ls` search for the command path, source code and manual.  
- `type -a` searches all possible paths of the command.  

### Creation of the process

Once the `ls` command is found. The shell will be able to execute it. To do this, three steps are necessary:  
**fork, exec and wait**

#### Fork

The first thing that will be done when the shell executes the command is a fork via the `fork()` function.  
Our shell will ask the kernel to create a child process from the shell which will be the parent process. So our `ls` command will have its own execution space in some way and can be identified by a PID.

#### Exec

Now the shell will call the `exec**()` function which will load the ls program to this new process created to be executed.

#### Wait

Before execution, the parent process is blocked by the `wait()` function so that the child process can finish before the parent.  
The parent will then wait for the end of the child's execution. We will be sure that the child finishes its execution to avoid it becoming a zombie process.

### Execution of the ls command

Now comes our famous ls program.

It will mainly read files and folders through kernel space functions.  
Main functions for reading files in a directory :

#### `opendir()`

`opendir()` -> which uses `getdents()` behind.

`opendir()` returns a *DIR* struct :

``` C

struct DIR
{
    struct dirent ent;
    struct _WDIR *wdirp;
};

```

`getdents()` returns a *linux_dirent* :

``` C
struct linux_dirent {
               unsigned long  d_ino;     /* Inode number */
               unsigned long  d_off;     /* Offset to next linux_dirent */
               unsigned short d_reclen;  /* Length of this linux_dirent */
               char           d_name[];  /* Filename (null-terminated) */
           }
```

`opendir()` opens a stream in the directory, and returns a pointer to that stream. The feed is positioned on the first entry of the directory.  
If an error occurs it returns null and returns an error code contained in errno.  
The input stream of the directory usually contains the name and number of the inode.

#### `readdir()`

It returns a pointer to a dirent structure representing the next entry in the directory stream pointed to by dir. It returns NULL at the end of the directory, or on error.

`readdir()` returns a *dirent* struct :

```C
struct dirent
{
    ino_t d_ino;             /* inode number */
    off_t d_off;             /* offset to the next dirent */
    unsigned short d_reclen; /* length of this record */
    unsigned char d_type;    /* type of file; not supported
                                by all file system types */
    char d_name[256];        /* filename */
};
```

#### `stat()`

It reads the inode table of the file system.
An inode table contains information about this directory but also the starting location of other files (the inodes of the files contained in this directory).  
It finally retrieves the state of the pointed file (inode number, uid, gid, creation time, odification, etc)

`stat()` returns a *stat* struct :

``` C

struct stat {
    dev_t     st_dev;     /* ID of device containing file */
    ino_t     st_ino;     /* inode number */
    mode_t    st_mode;    /* protection */
    nlink_t   st_nlink;   /* number of hard links */
    uid_t     st_uid;     /* user ID of owner */
    gid_t     st_gid;     /* group ID of owner */
    dev_t     st_rdev;    /* device ID (if special file) */
    off_t     st_size;    /* total size, in bytes */
    blksize_t st_blksize; /* blocksize for file system I/O */
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
    time_t    st_atime;   /* time of last access */
    time_t    st_mtime;   /* time of last modification */
    time_t    st_ctime;   /* time of last status change */
};
```

Then according to the different arguments `ls` will loop or not in the other directories.
Finally it will be able to display the files and references found in the standard output to the shell.

## Example with our own ls program

``` C
/**
 * File : lc.c
 * Description : Simple ls with path in argument
*/

/* Basic i/o stream */
#include <stdio.h>
/* Directories entry stream format */
#include <dirent.h>
/* Exit codes */
#include <stdlib.h>

void myLs(const char *dir)
{
    /* Entry stream of the files in directory */
    struct dirent *file;
    /* Stream of the directory */
    DIR *dirStream = opendir(dir);

    /* Check any error given by opendir */
    if (!dirStream)
    {
        perror(dir);
        exit(EXIT_FAILURE);
    }

    /* While the next entry directory is not readable, print the file's name */
    while ((file = readdir(dirStream)) != NULL)
    {
        /* Don't print hidden files */
        if (file->d_name[0] == '.')
            continue;
        printf("%s ", file->d_name);
    }
    printf("\n");
}

int main(int argc, char const *argv[])
{
    if (!argv[1])
        /* Print files in the current path if there aren't any arguments */
        myLs(".");
    else
        /* Print files in the path given in argument */
        myLs(argv[1]);
    return 0;
}

/**
 * Compiling : gcc ls.c -o ls
 * Execution : ./ls or ./ls dirPath/
*/
```

## ls source code

It is always interesting to be able to see the original source code of the ls command.  
For this we can get it directly from the official GNU site by downloading the source code of the coreutils package.

We can then observe all the subtleties that are not taken into account in the example of the previous program.

[Coreutils](https://www.gnu.org/software/coreutils/)

The source code of ls is located in the src folder.

## Sources

The C Programming Language Dennis Ritchie & Brian Kernighan

[Family of exec](https://stackoverflow.com/questions/20823371/what-is-the-difference-between-the-functions-of-the-exec-family-of-system-calls)

[fork, exec, wait and exit](https://www.percona.com/community-blog/2021/01/04/fork-exec-wait-and-exit/)

[What happens behind ls command](https://medium.com/meatandmachines/what-really-happens-when-you-type-ls-l-in-the-shell-a8914950fd73#:~:text=It%20is%20the%20way%20a,and%20created%20date%20and%20time.)

[ls example](https://gist.github.com/svdamani/cd9d2a5bb19228d912dc)

[Programmation avancée](https://mtodorovic.developpez.com/linux/programmation-avancee/)

[Old ls](https://git.savannah.gnu.org/cgit/coreutils.git/refs/tags)

[Coreutils](https://www.gnu.org/software/coreutils/)

[How does ls work? by Amitasha](https://gist.github.com/amitsaha/8169242)

[opendir](http://manpagesfr.free.fr/man/man3/opendir.3.html)

[History of ls](https://linuxgazette.net/issue48/fischer.html)

___
|          |            |
| -------- | ---------- |
| Author   | AnthonyF   |
| Created  | 24/04/2021 |
| Modified | 28/04/2022 |