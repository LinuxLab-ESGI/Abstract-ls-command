# Abstract of the `ls` command

- [Abstract of the `ls` command](#abstract-of-the-ls-command)
  - [Some useful options and tips](#some-useful-options-and-tips)
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

## How does ls work?

`ls` (**L**ist **D**irectory) est un programme du paquet coreutils developpe par la fondation GNU. C'est l'une des premieres commandes que l'on apprend lorsqu'on debute sur les distribs linux. Mais comment marche-t-elle reellement ?
### The shell

Premierement le shell est le premier programme que l'utilisateur va rencontrer en premier. C'est via lui qu'il pourra communiquer avec les autres programmes et par la suite ils se chargeront de communiquer avec le Kernel qui enfin accedera a la partie Hardware.

Ce flux d'echanges d'informations au sein du systeme d'exploitation peut etre represente succintement via cette ligne suivante :  

User > Shell > Other Programs > Kernel > Hardware

C'est donc via le shell que nous pouvons taper le programme ls pour ainsi l'appeler et l'executer par la suite.

### Checks (Aliases and PATH)

La premiere chose que va faire le shell lorsqu'on entre la commande ls est de verifier si un alias existe pour ce dernier. Si l'alias ls existe il remplacera cette valeur par ce qu'on a tape.

Si l'alias n'existe pas il va alors chercher la commande ls parmi les chemin contenu dans le PATH

Pour connaitre tous les chemins specifie dans le path on peut executer la commande suivante :  
`echo $PATH`

Pour savoir ou se trouve la commande en question nous pouvons executer ces commandes :

`which ls` donne le chemin d'ou vient la commande.
`whereis ls` cherche le chemin de la commande, code source ainsi que le manuel.
`type -a` cherche tous les chemins possible de la commande.

### Creation of the process

Une fois la commande ls trouvee. Le shell va pouvoir l'excuter pour se faire, trois etapes sont necessaires :  
**fork exec wait**

#### Fork

La permiere chose qui va etre faite lorsque le shell va exectuer la commande est un fork via la fonction fork().  
Notre shell va demander au kernel de creer un processus enfant a partir du shell qui sera le processus parent. Ainsi notre command ls pourra avoir son propre espace d'execution en quelque sort et pourra etre identifiable par un PID.

#### Exec

Desormais le shell va appeler la fonction exec**() qui s'occupe de charger le programme ls a ce nouveau process cree pour etre execute

#### Wait

Avant l'execution, le processus parent est bloque par la fonction wait() pour que le processus enfant puisse se terminer avant le parent.  
Le parent va alors attendre la fin de l'execution de l'enfant. On sera ainsi sur que l'enfant termine son execution pour eviter que le processus devienne un processus zombie.

### Execution of the ls command

Intervient desormais nore fameux programme ls.

il va principelament lire des fichiers et reperctoires grace a des fonctions du kernel space.

Principales fonctions pour la lecture des fichiers presents dans un repertoire :

#### `opendir()`

`opendir()` -> qui utilise `getdents()` derriere

Exemple d'un struct que recupere `opendir()` :

``` C

struct DIR
{
    struct dirent ent;
    struct _WDIR *wdirp;
};

```

Exemple d'un struct que recupere `getdents()`

``` C
struct linux_dirent {
               unsigned long  d_ino;     /* Inode number */
               unsigned long  d_off;     /* Offset to next linux_dirent */
               unsigned short d_reclen;  /* Length of this linux_dirent */
               char           d_name[];  /* Filename (null-terminated) */
           }
```

`opendir()` ouvre un flux du repertoire, et renvoie un pointeur sur ce flux. Le flux est positionnÃ© sur la premiÃ¨re entrÃ©e du rÃ©pertoire.
 Si un erreur interveint il renvoit null et renvoit un code error contenu dans errno.
 Le flux d'entree du reprtoire contient generalemnt le nom et le numero d'inode.

#### `readdir()`

Il renvoie un pointeur sur une structure dirent reprÃ©sentant l'entrÃ©e suivante du flux rÃ©pertoire pointÃ© par dir. Elle renvoie NULL Ã  la fin du rÃ©pertoire, ou en cas d'erreur.

Exemple d'un struct que recupere `readdir()` : 

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

Il lit la table d'inode du systeme de fichier.
Une table d'inodes contient des informations de ce repertoire mais egelment l'endroit de depart des autre fichiers (les inodes des fichiers contenus dans ce repertoires).  
Il rÃ©cupÃ¨re finalement l'Ã©tat du fichier pointÃ© (numero d'inode, uid, gid, heure de creation, odification, etc)

Exemple de struct que recupere `stat()` :

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

Puis selon les differents arguments `ls` va boucler ou non dans les autres repertoires.
Enfin il pourra afficher les fichiers et reperotires trouves dans la sortie standard vers le shell.

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

C'est toujours interesssant de pouvoir observer le code source original de la commande ls.
Pour cela nous pouvons l'obtenir directement a aprtir du site officiel GNU en telechraeant le code source du packet coreutils.

On peut alors oberserver toutes les subtilites qui ne sont pas prises en compte dans l'exemple du prgramme precedent.

https://www.gnu.org/software/coreutils/

Le code source de ls est situe dans le dossier src

## Sources

https://medium.com/meatandmachines/what-really-happens-when-you-type-ls-l-in-the-shell-a8914950fd73#:~:text=It%20is%20the%20way%20a,and%20created%20date%20and%20time.

https://stackoverflow.com/questions/20823371/what-is-the-difference-between-the-functions-of-the-exec-family-of-system-calls

https://www.percona.com/community-blog/2021/01/04/fork-exec-wait-and-exit/

https://gist.github.com/svdamani/cd9d2a5bb19228d912dc

https://mtodorovic.developpez.com/linux/programmation-avancee/

The C Programming Language Dennis Ritchie & Brian Kernighan

https://git.savannah.gnu.org/cgit/coreutils.git/refs/tags

https://www.gnu.org/software/coreutils/

https://gist.github.com/amitsaha/8169242

http://manpagesfr.free.fr/man/man3/opendir.3.html