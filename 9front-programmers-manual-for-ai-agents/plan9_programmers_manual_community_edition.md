---
title: "9front Programmer's Manual for AI Agents — Community Edition"
author: "Anastasios Papalias"
publisher: "Anastasios Papalias"
year: "2026"
rights: "Free — published on GitHub"
cover-image: "cover.png"
lang: en
---

::: {.copyright-page}
**9front Programmer's Manual for AI Agents — Community Edition**

Author: Anastasios Papalias
Published: 2026
License: Free — published on GitHub
Repository: github.com/AnastasiosPapalias/Plan9

*The structured AI training dataset for 9front and Plan 9 C programming.*

*Cover: visual concept developed with generative tools.*
:::


# Contents of This Edition


| Section | Description |
| --- | --- |
| Part 1 — Structured API Reference | 20 concept blocks: signature, description, resource model, examples for every key syscall and library function |
| Part 2 — Code Patterns | 10 complete compilable 9front C programs covering all required pattern types |
| Part 3 — Anti-POSIX Training | 16 WRONG→CORRECT→WHY blocks teaching AI to reject POSIX hallucinations |
| Part 4 — Symbol Map | 107 typed symbols — all syscalls, library functions, types, macros, constants |
| Part 5 — Integration Guide | How to use this dataset: RAG pipelines, SFT fine-tuning, symbol-aware code completion |
| Part 6 — Supplementary Reference | AI Reasoning Protocol, dirread/dirreadall, ioproc, factotum/auth, tokenize/getfields |
| symbol_map.jsonl | 107 typed symbols with full metadata for code completion tooling (JSONL) |
| plan9_dataset.jsonl | 77 pre-chunked RAG records with topic, embedding tags, and importance field (JSONL) |
| plan9_programmers_manual_community_edition.md | This document in Markdown format |
| plan9_programmers_manual_community_edition.docx | This document in Word format |
| README.md | GitHub landing page with quick-start guide |


## About the Community Edition


This edition is primarily intended for AI consumption, enabling AI agents to learn, understand, and perform effectively on the subject matter of Plan 9 and 9front C programming. Content is structured, clear, and optimised for machine readability. The document you are reading contains the five structured dataset layers. The master reference document (plan9_programmers_manual_community_edition.md) contains the full 1.5 MB corpus including kernel source commentary, textbook chapters, and Bell Labs papers.


## Dataset Architecture — 7 Layers


| Layer | File | Purpose | Records |
| --- | --- | --- | --- |
| L1 — Core Knowledge | Master .md | 24 man pages verbatim + Bell Labs papers | — |
| L2 — Structured MD | plan9_dataset.md | 20 concept blocks, all required fields | 20 |
| L3 — Symbol Map | symbol_map.jsonl | Typed index of every symbol | 107 |
| L4 — Code Patterns | code_patterns.md | 10 complete example programs | 10 |
| L5 — Anti-POSIX | anti_patterns.md | 16 WRONG→CORRECT→WHY blocks | 16 |
| L6 — Reasoning | plan9_dataset.jsonl | Explicit 4-step decision protocol | 1 |
| L7 — RAG Chunks | plan9_dataset.jsonl | Vector-database ready, tagged, importanced | 77 |


# Part 1 — Structured API Reference


**System Calls, Library Functions & Core Concepts**


> Each concept block contains mandatory fields: SIGNATURE, DESCRIPTION, RESOURCE MODEL, WHEN TO USE, WHEN NOT TO USE, RELATED, SOURCE, and EXAMPLE. All signatures were sourced live from man.aiju.de — the authoritative 9front man pages. These blocks form Layer 2 (Structured Markdown) of the dataset architecture.


# 9front/Plan 9 C Programmer's Dataset


## Structured Reference — Every concept as a typed, relational block


> **Format:** Each concept block contains mandatory fields from the spec. Layer 2 of the dataset architecture. Source: man.aiju.de (live 9front man pages) + Bell Labs papers + Ballesteros.


## open (SYSTEM CALL)


#### CATEGORY


file I/O


#### SIGNATURE


```c
int open(char *file, int omode)
int create(char *file, int omode, ulong perm)
int close(int fd)
```


#### DESCRIPTION


Open opens the file for I/O and returns an associated file descriptor.


Omode is one of OREAD (0), OWRITE (1), ORDWR (2), or OEXEC (3).


Additional flags ORed into omode: OTRUNC (truncate to zero), OCEXEC (close on exec),


ORCLOSE (remove on last close).


Create creates a new file or resets an existing one. If the file already exists, it


is truncated to 0 length. perm sets permissions for new files. DMDIR|perm creates a


directory. DMAPPEND creates an append-only file. DMEXCL creates an exclusive-use file.


OEXCL in omode causes create to fail if the file already exists.


Close closes the file descriptor. Always succeeds on a valid fd; the fd is freed


for reuse. Files are closed automatically on process exit.


#### RESOURCE MODEL

- namespace: evaluated at time of call
- fd table: allocates lowest available fd; shared or private depending on rfork flags
- error: sets errstr; returns -1


#### WHEN TO USE


Any time you need to read, write, or create a file or directory.


Use create instead of open for new files; they are separate calls in Plan 9


(unlike UNIX which merges them with O_CREAT).


#### WHEN NOT TO USE


Do not use open on a nonexistent file expecting it to create one — it will fail.


Use create for that.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>
```


```c
void
main(void)
{
int fd, n;
char buf[512];
```


```c
fd = open("/env/path", OREAD);
if(fd < 0)
sysfatal("open: %r");
n = read(fd, buf, sizeof buf - 1);
if(n < 0)
sysfatal("read: %r");
buf[n] = 0;
print("path=%s\n", buf);
close(fd);
exits(nil);
}
```


#### RELATED

- read, write, seek, close
- create, remove
- bind, mount (for device files)
- stat (for file metadata)


#### SOURCE

- /sys/man/2/open (man.aiju.de/2/open)
- /sys/src/libc/9syscall


## read / write (SYSTEM CALLS)


#### CATEGORY


file I/O


#### SIGNATURE


```c
long read(int fd, void *buf, long nbytes)
long readn(int fd, void *buf, long nbytes)
long write(int fd, void *buf, long nbytes)
long pread(int fd, void *buf, long nbytes, vlong offset)
long pwrite(int fd, void *buf, long nbytes, vlong offset)
```


#### DESCRIPTION


Read reads up to nbytes bytes from the current file offset into buf. Returns the


number of bytes actually read. Returns 0 at end of file. A short read is normal


and must not be treated as an error — for example, one read from a console returns


at most one line.


Readn loops calling read until exactly nbytes bytes have been read or EOF/error.


Write writes nbytes bytes from buf at the current file offset. Returns bytes written.


If write returns fewer bytes than requested, that IS an error.


Pread and pwrite are positional: they atomically seek to offset and do the I/O,


without changing the file's current offset. Essential for multiprocess I/O on a


shared fd.


#### RESOURCE MODEL

- fd: must be open with compatible mode (OREAD/OWRITE/ORDWR)
- offset: advanced by the number of bytes transferred (not for pread/pwrite)
- error: sets errstr; returns -1


#### WHEN TO USE

- read: sequential reads from a file or stream
- readn: when you need exactly N bytes (e.g., fixed-size records)
- pread/pwrite: concurrent multiprocess access to same fd without races


#### WHEN NOT TO USE


Do not assume read fills the buffer completely. Always check the return value.


Use readn when you need exactly N bytes.


#### EXAMPLE


```c
/* Read entire file into a buffer */
char *buf;
long n, size;
Dir *d;
```


```c
d = dirstat(path);
if(d == nil) sysfatal("dirstat: %r");
size = d->length;
free(d);
```


```c
buf = malloc(size + 1);
if(buf == nil) sysfatal("malloc: %r");
```


```c
fd = open(path, OREAD);
if(fd < 0) sysfatal("open: %r");
n = readn(fd, buf, size);
close(fd);
buf[n] = 0;
```


#### RELATED

- open, create, seek, dup
- pread, pwrite (positional variants)
- readn (loop variant)


#### SOURCE

- /sys/man/2/read (man.aiju.de/2/read)
- /sys/src/libc/9syscall
- /sys/src/libc/port/readn.c


## rfork (SYSTEM CALL)


#### CATEGORY


process control


#### SIGNATURE


```c
int rfork(int flags)
int fork(void)    /* = rfork(RFFDG|RFREND|RFPROC) */
```


#### DESCRIPTION


Rfork is the only way to create new processes in Plan 9. The flags argument


provides bit-level control over exactly which resources the new process shares,


copies, or initialises fresh. This is fundamentally different from POSIX fork,


which has a fixed resource sharing model.


**Complete flag reference:**


| Flag | Effect |
| --- | --- |
| RFPROC | Create a new process (required to actually fork; without it, changes apply to current process) |
| RFNOWAIT | Child is dissociated from parent: exits without creating a Waitmsg |
| RFNAMEG | Child inherits a copy of the parent's name space (mutually exclusive with RFCNAMEG) |
| RFCNAMEG | Child starts with a clean name space; must build from scratch via mount |
| RFNOMNT | Disallow mount/bind and # device paths in child |
| RFENVG | Child gets a copy of the parent's environment (mutually exclusive with RFCENVG) |
| RFCENVG | Child starts with an empty environment |
| RFNOTEG | Child starts a new note group; notes to parent don't reach child and vice versa |
| RFFDG | Child gets a copy of the file descriptor table (mutually exclusive with RFCFDG) |
| RFCFDG | Child starts with an empty file descriptor table (no stdin/stdout/stderr) |
| RFREND | Child cannot rendezvous with parent or parent's ancestors |
| RFMEM | Child shares data and bss segments with parent (only valid with RFPROC; stack is always separate) |


Return values: parent gets child PID, child gets 0, error returns -1.


Rfork will sleep if resources are temporarily unavailable.


#### RESOURCE MODEL


```c
Resource       RFFDG        RFCFDG       (neither)
──────────────────────────────────────────────────
fd table:      copied       empty        shared
```


```c
Resource       RFNAMEG      RFCNAMEG     (neither)
──────────────────────────────────────────────────
name space:    copied       empty        shared
```


```c
Resource       RFENVG       RFCENVG      (neither)
──────────────────────────────────────────────────
environment:   copied       empty        shared
```


```c
Resource       RFMEM        (not set)
──────────────────────────────────────
data/bss:      shared       copied (CoW)
stack:         always separate (CoW)
```


#### WHEN TO USE

- Standard fork: rfork(RFPROC|RFFDG|RFNOTEG|RFENVG) — isolated child with own fds
- Thread-like (shared memory): rfork(RFPROC|RFMEM|RFFDG) — libthread uses this
- New name space: rfork(RFPROC|RFFDG|RFNAMEG|RFNOTEG) then rebuild with bind/mount
- Daemon child: add RFNOWAIT to prevent zombie accumulation
- Sandbox: rfork(RFPROC|RFCFDG|RFCNAMEG|RFCENVG|RFNOMNT) — maximum isolation


#### WHEN NOT TO USE


Do not use bare fork() when you need specific resource sharing control.


Do not use RFMEM alone (without RFPROC) expecting it to create a new process —


without RFPROC the change applies to the calling process only.


Do not use RFMEM + RFFDG together expecting full isolation — they are contradictory.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>
```


```c
void
main(void)
{
int pid;
Waitmsg *w;
```


```c
switch(pid = rfork(RFPROC|RFFDG|RFNOTEG|RFENVG)){
case -1:
sysfatal("rfork: %r");
case 0:
/* child */
print("child pid %d\n", getpid());
exits(nil);
default:
/* parent */
w = wait();
if(w == nil) sysfatal("wait: %r");
if(w->msg[0])
fprint(2, "child failed: %s\n", w->msg);
free(w);
exits(nil);
}
}
```


#### RELATED

- exec, execl
- wait, await, waitpid
- exits
- notify, postnote
- bind, mount (for name space manipulation after fork)


#### SOURCE

- /sys/man/2/fork (man.aiju.de/2/fork)
- /sys/src/libc/9syscall
- /sys/src/libc/9sys/fork.c


## exec / execl (SYSTEM CALLS)


#### CATEGORY


process control


#### SIGNATURE


```c
int exec(char *name, char *argv[])
int execl(char *name, ...)
```


#### DESCRIPTION


Exec overlays the calling process with the named file. On success, exec does not


return — the process continues as the new program. On failure, returns -1 and


sets errstr.


argv is a nil-terminated array of NUL-terminated strings. By convention, argv[0]


is the program name. Execl accepts arguments directly as variadic arguments,


terminated with a nil pointer.


Files remain open across exec unless opened with OCEXEC.


Notification handlers are cleared by exec.


#### RESOURCE MODEL

- process image: completely replaced
- pid: unchanged
- open files: preserved (unless OCEXEC was set)
- name space: preserved
- note handlers: cleared


#### WHEN TO USE


After rfork(RFPROC|...) in the child, to run a different program.


The idiom is always rfork then exec — never exec without forking first unless


you want to replace the current process permanently.


#### WHEN NOT TO USE


Do not exec without first forking if you want the parent to continue.


#### EXAMPLE


```c
void
runcmd(char *cmd, char **args)
{
int pid;
Waitmsg *w;
```


```c
switch(pid = rfork(RFPROC|RFFDG|RFENVG|RFNOTEG)){
case -1:
sysfatal("rfork: %r");
case 0:
exec(cmd, args);
sysfatal("exec %s: %r", cmd);   /* only reached on error */
}
w = wait();
if(w->msg[0])
fprint(2, "%s: %s\n", cmd, w->msg);
free(w);
}
```


#### RELATED

- rfork (must fork first)
- wait, exits
- open, dup (fd management before exec)


#### SOURCE

- /sys/man/2/exec (man.aiju.de/2/exec)
- /sys/src/libc/9syscall


## exits / wait (SYSTEM CALLS)


#### CATEGORY


process control


#### SIGNATURE


```c
void exits(char *msg)
void _exits(char *msg)
Waitmsg* wait(void)
int waitpid(void)
int await(char *s, int n)
```


#### DESCRIPTION


Exits terminates the calling process. msg is passed to the parent via wait.


A nil or empty string means success; any non-empty string means failure.


Before calling _exits, exits calls registered atexit functions in reverse order.


Wait blocks until any child process exits. Returns a malloc'd Waitmsg (caller


must free). Waitmsg.pid is the child's pid; Waitmsg.msg is the exit string


(empty string for success); Waitmsg.time[3] contains user time, sys time, and


real time in milliseconds. Returns nil if no children remain.


_Exits is the raw system call; it does not call atexit functions.


#### RESOURCE MODEL

- Returns: never (exits, _exits)
- wait return: malloc'd Waitmsg* that caller must free()
- On no children: wait returns nil immediately


#### WHEN TO USE

- Always call exits(nil) at the end of main (not return 0)
- Call exits("reason") on error with a brief message
- Call wait() after rfork(RFPROC|...) to collect child status


#### WHEN NOT TO USE


Do not use exit() (POSIX) — it does not exist in the native Plan 9 API.


Do not return from main without calling exits — behaviour is undefined.


#### EXAMPLE


```c
/* Clean exit */
exits(nil);         /* success */
exits("usage");     /* failure */
exits("open failed"); /* failure with message */
```


```c
/* Wait for child */
Waitmsg *w = wait();
if(w == nil) sysfatal("wait: %r");
if(w->msg[0] != '\0')
fprint(2, "child failed: %s\n", w->msg);
free(w);
```


#### RELATED

- rfork (creates children)
- atexit (cleanup on exit)
- _exits (raw syscall, no atexit)


#### SOURCE

- /sys/man/2/exits (man.aiju.de/2/exits)
- /sys/man/2/wait  (man.aiju.de/2/wait)
- /sys/src/libc/port/exits.c


## errstr / werrstr (SYSTEM CALL + LIBRARY)


#### CATEGORY


error handling


#### SIGNATURE


```c
int   errstr(char *err, uint nerr)
void  rerrstr(char *err, uint nerr)
void  werrstr(char *fmt, ...)
#define ERRMAX 128
```


#### DESCRIPTION


Plan 9 replaces errno with a per-process error string. Every failed system call


sets this string before returning -1.


errstr swaps the per-process error string with the contents of err (exchange


semantics — you give your string, you get the current error string back).


The most common idiom is to call it with an initially-empty buffer.


rerrstr reads the error string without modifying it (read-only peek).


werrstr formats a string using printf-style format and sets it as the error


string. Used in library functions to annotate errors.


The format verb %r in print/fprint/snprint/seprint expands to the current


error string automatically — no need to call errstr explicitly.


#### RESOURCE MODEL

- Per-process: each process has its own error string buffer
- Length limit: ERRMAX bytes (128). Strings longer than ERRMAX-1 are truncated.
- Exchange: errstr swaps, not reads


#### WHEN TO USE

- After any failed system call to report the reason
- In library functions, use werrstr to annotate errors with context
- Use %r in format strings rather than calling errstr manually


#### WHEN NOT TO USE


Do not use errno (POSIX) — it does not apply to native Plan 9 programs.


Do not forget to check return values; error strings are only meaningful after


a failing call.


#### EXAMPLE


```c
/* Pattern 1: %r auto-expansion */
fd = open(path, OREAD);
if(fd < 0)
sysfatal("open %s: %r", path);
```


```c
/* Pattern 2: Explicit errstr */
char err[ERRMAX];
if(write(fd, buf, n) != n){
errstr(err, sizeof err);
fprint(2, "write failed: %s\n", err);
exits(err);
}
```


```c
/* Pattern 3: Library annotation */
int
myopen(char *path)
{
int fd = open(path, OREAD);
if(fd < 0){
werrstr("myopen %s: %r", path);
return -1;
}
return fd;
}
```


#### RELATED

- sysfatal (print + exits in one call)
- %r format verb (expands errstr)
- perror (POSIX — do not use)


#### SOURCE

- /sys/man/2/errstr (man.aiju.de/2/errstr)
- /sys/src/libc/9sys/werrstr.c


## bind / mount / unmount (SYSTEM CALLS)


#### CATEGORY


name space manipulation


#### SIGNATURE


```c
int bind(char *name, char *old, int flag)
int mount(int fd, int afd, char *old, int flag, char *aname)
int unmount(char *name, char *old)
```


#### DESCRIPTION


Bind and mount modify the file name space of the current process and all processes


in its name space group (controlled by rfork RFNAMEG/RFCNAMEG).


**bind(name, old, flag):** Makes old an alias for name. After a successful bind,


accessing old actually accesses name's target.


**mount(fd, afd, old, flag, aname):** Attaches the 9P file server on fd to the


name space at old. afd is the authentication file descriptor from fauth(2),


or -1 if no authentication is needed. aname selects which file tree the server


offers (pass "" for default). The fd is automatically closed on successful mount.


**Flag values:**


| Flag | Meaning |
| --- | --- |
| MREPL | Replace old with the new file/tree |
| MBEFORE | Add to union directory; new contents searched first |
| MAFTER | Add to union directory; new contents searched last |
| MCREATE | Allow file creation in this union member |
| MCACHE | (mount only) Enable client-side caching |


Union directories (via MBEFORE/MAFTER) allow multiple directory trees to be


overlaid at one path. When looking up a file, each member is searched in order


until a match is found.


**unmount(name, old):** If name is nil, unmounts everything at old. If name is


given, unmounts only that specific binding.


#### RESOURCE MODEL

- Affects: current process and all processes sharing its name space group
- Isolation: rfork(RFNAMEG) gives a private copy; rfork(RFCNAMEG) gives a clean slate
- Persistence: bind/mount effects last until unmounted or process exits


#### WHEN TO USE

- Overlay /bin with architecture-specific binaries: bind /386/bin /bin MAFTER
- Import a remote file system: mount(fd, -1, "/n/remote", MREPL, "")
- Create a union /bin: bind /rc/bin /bin MAFTER
- Sandbox: rfork(RFNAMEG) then rebuild name space with minimal bind/mount calls
- Plumb a device into /dev: bind "#p" /proc MREPL


#### WHEN NOT TO USE


Do not bind without first ensuring you are in a private name space if you don't


want to affect sibling processes. The effect is shared unless rfork(RFNAMEG) was used.


#### EXAMPLE


```c
/* Mount a pipe-connected 9P server */
int fd[2];
pipe(fd);
/* parent serves on fd[0]; child mounts fd[1] */
switch(rfork(RFPROC|RFFDG|RFNAMEG)){
case 0:
close(fd[0]);
if(mount(fd[1], -1, "/mnt/myfs", MREPL, "") < 0)
sysfatal("mount: %r");
close(fd[1]);
/* now /mnt/myfs is the server's tree */
break;
default:
close(fd[1]);
serve(fd[0]);  /* serve 9P on fd[0] */
break;
}
```


#### RELATED

- rfork (RFNAMEG, RFCNAMEG for name space isolation)
- fauth (authentication before mount)
- unmount
- namespace(4) (conventions for name space layout)


#### SOURCE

- /sys/man/2/bind (man.aiju.de/2/bind)
- /sys/src/libc/9syscall


## dial / announce / listen / accept (LIBRARY)


#### CATEGORY


networking


#### SIGNATURE


```c
int  dial(char *addr, char *local, char *dir, int *cfdp)
int  announce(char *addr, char *dir)
int  listen(char *dir, char *newdir)
int  accept(int ctl, char *dir)
int  hangup(int ctl)
char* netmkaddr(char *addr, char *defnet, char *defservice)
```


#### DESCRIPTION


These functions implement network connections by manipulating /net.


Address format: network!host!service or network!host or just host.


Network is any protocol listed in /net (tcp, udp, etc.) or the meta-token net


which tries all available networks.


**dial(addr, local, dir, cfdp):** Connect to addr. Returns the data fd. If dir is


non-nil, copies the line directory path (<40 bytes) into it. If cfdp is non-nil,


sets *cfdp to the ctl fd. The connection is closed when both data and ctl fds


are closed. local sets the local address (nil = any).


**announce(addr, dir):** Register a service at addr (use tcp!*!portnum for all


interfaces). Returns a ctl fd. Copies line directory into dir.


**listen(dir, newdir):** Wait for an incoming call on an announced address.


Blocks until connection arrives. Returns a ctl fd; copies new line directory


into newdir.


**accept(ctl, dir):** Accept the connection represented by the ctl fd from listen.


Returns data fd opened ORDWR.


**hangup(ctl):** Force-close a connection without closing fds.


**netmkaddr(addr, defnet, defservice):** Fill in defaults for network and service


if not present in addr. Returns pointer to static data.


#### RESOURCE MODEL

- All connections are files under /net
- Closing the data fd (or ctl fd if no data fd) closes the connection
- Line directory path: always < 40 bytes
- Error: returns -1 and sets errstr


#### WHEN TO USE

- Client TCP connection: fd = dial("tcp!host!80", nil, nil, nil)
- Server: announce + loop of listen + accept + rfork to handle
- Any protocol (TCP, UDP, etc.) with the same API


#### WHEN NOT TO USE


Do not use socket()/connect()/bind() — POSIX sockets do not exist in native Plan 9.


#### EXAMPLE


```c
/* TCP client */
int fd = dial("tcp!example.com!80", nil, nil, nil);
if(fd < 0) sysfatal("dial: %r");
fprint(fd, "GET / HTTP/1.0\r\n\r\n");
/* read response... */
close(fd);
```


```c
/* TCP server */
char adir[40], ldir[40];
int afd = announce("tcp!*!9999", adir);
if(afd < 0) sysfatal("announce: %r");
for(;;){
int lfd = listen(adir, ldir);
if(lfd < 0) sysfatal("listen: %r");
int dfd = accept(lfd, ldir);
close(lfd);
switch(rfork(RFPROC|RFFDG|RFNOTEG)){
case 0: handle(dfd); exits(nil);
default: close(dfd);
}
}
```


#### RELATED

- ip(3) (the /net device)
- cs (connection server — dial uses it automatically)
- fauth, mount (for authenticated 9P mounts)


#### SOURCE

- /sys/man/2/dial (man.aiju.de/2/dial / man.9front.org/2/dial)
- /sys/src/libc/9sys/dial.c


## errstr model vs errno (CRITICAL CONCEPT)


#### CATEGORY


error handling / anti-POSIX


#### DESCRIPTION


Plan 9 uses a per-process string for error reporting. POSIX uses an integer (errno).


These are completely incompatible. The Plan 9 model is strictly superior: error


messages are human-readable and can carry context.


**Key differences:**


| Aspect | POSIX (errno) | Plan 9 (errstr) |
| --- | --- | --- |
| Type | int | char[ERRMAX] |
| Max info | EINVAL = 22 | "open /foo: file does not exist" |
| How to read | strerror(errno) | errstr(buf, sizeof buf) or %r |
| How to set | errno = EINVAL | werrstr("...") |
| Thread-safe | TLS-based | per-process buffer |
| Standard include | <errno.h> | <libc.h> only |


**errno does not exist** in native Plan 9 C. Including <errno.h> is APE/POSIX only.


#### RELATED

- errstr, werrstr, rerrstr
- sysfatal (convenience: print + exits)


#### SOURCE

- /sys/man/2/errstr


## stat / Dir (SYSTEM CALL + STRUCTURE)


#### CATEGORY


file metadata


#### SIGNATURE


```c
Dir* dirstat(char *name)
Dir* dirfstat(int fd)
int  dirwstat(char *name, Dir *dir)
int  dirfwstat(int fd, Dir *dir)
void nulldir(Dir *d)
```


```c
typedef struct Dir {
uint  type;     /* server type */
uint  dev;      /* server subtype */
Qid   qid;      /* unique id from server */
ulong mode;     /* permissions */
ulong atime;    /* last read time (sec since epoch) */
ulong mtime;    /* last write time */
vlong length;   /* file length in bytes */
char  *name;    /* last element of path */
char  *uid;     /* owner name */
char  *gid;     /* group name */
char  *muid;    /* last modifier name */
} Dir;
```


```c
typedef struct Qid {
uvlong path;    /* unique path id (64 bits) */
ulong  vers;    /* incremented on each write */
uchar  type;    /* QTDIR, QTAPPEND, QTEXCL, QTAUTH */
} Qid;
```


#### DESCRIPTION


dirstat and dirfstat return a malloc'd Dir* or nil on error. The caller must


free() the returned pointer (which also frees the embedded strings).


dirwstat and dirfwstat write modified metadata back. Use nulldir to zero-fill


a Dir before setting only the fields you want to change — this avoids needing


to stat first.


Mode bits:


```c
0x80000000  DMDIR     directory
0x40000000  DMAPPEND  append-only
0x20000000  DMEXCL    exclusive use
0400   owner read
0200   owner write
0100   owner execute/search
0070   group rwx
0007   other rwx
```


Qid.vers changes every time the file is modified — use this for cache validation.


#### RESOURCE MODEL

- Return: malloc'd Dir* (caller must free)
- Strings: name, uid, gid, muid are in the same malloc'd block — free once


#### WHEN TO USE

- Check if file exists: dirstat returns nil on missing file
- Get file size: d->length
- Check file type: d->mode & DMDIR
- Change permissions: nulldir + set mode + dirwstat


#### EXAMPLE


```c
Dir *d = dirstat(path);
if(d == nil){
/* file does not exist or error */
char err[ERRMAX];
errstr(err, sizeof err);
fprint(2, "%s: %s\n", path, err);
exits(err);
}
print("size=%lld mode=%ulo owner=%s\n", d->length, d->mode, d->uid);
free(d);
```


```c
/* Change mode without stat first */
Dir nd;
nulldir(&nd);
nd.mode = 0644;
if(dirwstat(path, &nd) < 0)
sysfatal("chmod: %r");
```


#### RELATED

- dirread, dirreadall (directory listing)
- open, create
- stat(5) (wire format)


#### SOURCE

- /sys/man/2/stat (man.aiju.de/2/stat)


## notify / noted / atnotify (SYSTEM CALLS + LIBRARY)


#### CATEGORY


process control / events


#### SIGNATURE


```c
int notify(void (*f)(void*, char*))
int noted(int v)
int atnotify(int (*f)(void*, char*), int in)
ulong alarm(ulong ms)
int postnote(int group, int pid, char *note)
```


#### DESCRIPTION


Notes are Plan 9's signal analog. They are human-readable strings, not integers.


Common notes:

- "interrupt" — user hit DEL key
- "hangup" — I/O connection closed
- "alarm" — timer expired (from alarm(2))
- "kill" — unconditional termination
- "sys: write on closed pipe" — written to a closed pipe


**atnotify (preferred):** Register handler f(ureg, notestring). Return non-zero to


consume the note; return zero to pass to next handler or default. Use in=1 to


register, in=0 to unregister.


**notify (lower level):** Registers a single handler function f. The handler MUST


call noted(NCONT) or noted(NDFLT) to resume or take default action — it cannot


return normally. An argument of nil cancels the handler.


**noted(v):** Called from within a notify handler. NCONT resumes execution; NDFLT


takes the default action (usually termination). Does not return to the handler.


**alarm(ms):** Schedule an "alarm" note after ms milliseconds. Returns remaining


time of previous alarm. Zero cancels.


**postnote(group, pid, note):** Send a note. PNPROC sends to one process;


PNGROUP sends to the entire note group.


#### RESOURCE MODEL

- Note handlers are cleared by exec
- Only one notify handler at a time; atnotify manages a list internally
- Notes are asynchronous: delivered when the process is scheduled
- Handlers may not do floating-point operations


#### WHEN TO USE


Use atnotify for clean signal-like handling (interrupt, hangup, alarm).


Use alarm + atnotify for timeouts.


Use postnote to kill or signal other processes.


#### WHEN NOT TO USE


Do not use signal() (POSIX) — it does not exist in native Plan 9.


Do not return from a notify handler — call noted() instead.


#### EXAMPLE


```c
int interrupted = 0;
```


```c
static int
inthndlr(void *ureg, char *msg)
{
USED(ureg);
if(strcmp(msg, "interrupt") == 0){
interrupted = 1;
return 1;   /* consumed */
}
return 0;       /* pass to next handler */
}
```


```c
void
main(void)
{
atnotify(inthndlr, 1);
/* ... do work, check interrupted periodically ... */
exits(nil);
}
```


```c
/* Timeout example */
static void
alarmhndlr(void *u, char *msg)
{
USED(u);
if(strcmp(msg, "alarm") == 0){
noted(NCONT);
return;
}
noted(NDFLT);
}
```


#### RELATED

- postnote (send note to another process)
- alarm (schedule automatic "alarm" note)
- fork (RFNOTEG for note group isolation)


#### SOURCE

- /sys/man/2/notify (man.aiju.de/2/notify)
- /sys/src/libc/port/atnotify.c


## print / fprint / seprint / smprint (LIBRARY)


#### CATEGORY


I/O formatting


#### SIGNATURE


```c
int   print(char *fmt, ...)
int   fprint(int fd, char *fmt, ...)
int   snprint(char *s, int len, char *fmt, ...)
char* seprint(char *s, char *e, char *fmt, ...)
char* smprint(char *fmt, ...)
int   vfprint(int fd, char *fmt, va_list v)
char* vseprint(char *s, char *e, char *fmt, va_list v)
char* vsmprint(char *fmt, va_list v)
```


#### DESCRIPTION


Plan 9's print family replaces stdio's printf. Do not mix them.


**print:** write to fd 1 (standard output).


**fprint:** write to any fd.


**snprint:** write to buffer s of length len; always NUL-terminates; returns bytes written.


**seprint:** write to buffer [s, e); NUL-terminates; returns pointer to terminating NUL.


Preferred for building strings incrementally without overflow risk.


**smprint:** malloc a string large enough and fill it; caller must free(). Never truncates.


**Format verbs:**


| Verb | Meaning |
| --- | --- |
| %d | decimal int |
| %ud | unsigned decimal |
| %ld | long decimal |
| %lld | vlong decimal |
| %x | hex lowercase |
| %X | hex uppercase |
| %llx | vlong hex |
| %o | octal |
| %b | binary (Plan 9 extension) |
| %f | float decimal |
| %g | float shortest |
| %e | float scientific |
| %s | char* string |
| %S | Rune* string |
| %c | char |
| %C | Rune |
| %p | pointer hex |
| %r | current errstr (no argument) |
| %q | quoted string (rc-safe) |


sprint (without n) is **deprecated** — unsafe, no bounds checking.


#### WHEN TO USE

- Preferred for error messages: fprint(2, "error: %r\n")
- Building strings: seprint for fixed buffers, smprint when size unknown
- %r verb saves calling errstr explicitly after failures


#### EXAMPLE


```c
/* seprint for incremental string building */
char buf[256];
char *p = buf, *e = buf + sizeof buf;
p = seprint(p, e, "user=%s ", user);
p = seprint(p, e, "pid=%d ", pid);
p = seprint(p, e, "err=%r");
print("%s\n", buf);
```


```c
/* smprint when size unknown */
char *s = smprint("file: %s size: %lld", path, size);
/* use s ... */
free(s);
```


#### RELATED

- fmtinstall (custom verbs)
- errstr (%r verb)


#### SOURCE

- /sys/man/2/print (man.aiju.de/2/print)


## Biobuf / bio(2) (LIBRARY)


#### CATEGORY


buffered I/O


#### SIGNATURE


```c
#include <bio.h>
Biobuf* Bopen(char *name, int mode)
int     Binit(Biobuf *bp, int fd, int mode)
int     Bterm(Biobufhdr *bp)
void*   Brdline(Biobufhdr *bp, int delim)
char*   Brdstr(Biobufhdr *bp, int delim, int nulldelim)
int     Blinelen(Biobufhdr *bp)
int     Bgetc(Biobufhdr *bp)
long    Bgetrune(Biobufhdr *bp)
int     Bungetc(Biobufhdr *bp)
long    Bwrite(Biobufhdr *bp, void *buf, long n)
int     Bputc(Biobufhdr *bp, int c)
int     Bputrune(Biobufhdr *bp, long r)
int     Bprint(Biobufhdr *bp, char *fmt, ...)
int     Bflush(Biobufhdr *bp)
vlong   Boffset(Biobufhdr *bp)
#define Beof (-1)
```


#### DESCRIPTION


libbio is Plan 9's buffered I/O library. It buffers reads and writes for


performance and provides line-oriented and Rune-aware reading.


**Bopen:** opens a file by name for OREAD or OWRITE and returns a malloc'd Biobuf*.


**Binit:** wraps an existing fd (e.g., 0 for stdin). Does not close fd on Bterm.


**Bterm:** flushes and frees the Biobuf. For Binit buffers, leaves the fd open.


**Brdline(bp, delim):** reads until delim (typically '\n'), returns pointer into


internal buffer (not malloc'd, not NUL-terminated). Use Blinelen to get length.


Returns nil at EOF.


**Brdstr(bp, delim, nulldelim):** like Brdline but returns a malloc'd NUL-terminated


string. If nulldelim is nonzero, the delimiter is removed. Caller must free().


**Bgetrune:** reads one Unicode rune (handles UTF-8 automatically). Returns Beof at EOF.


**Bgetc:** reads one byte.


libbio is far faster than raw read() for character-at-a-time or line-at-a-time access.


#### WHEN TO USE

- Reading text files line by line
- Any character-at-a-time or line-at-a-time access
- Unicode/Rune-aware input
- Wrapping stdin: Binit(&bin, 0, OREAD) (Plan 9 has no scanf)


#### WHEN NOT TO USE


Do not use FILE* (stdio) for new Plan 9 code. Use Biobuf instead.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>
#include <bio.h>
```


```c
void
main(int argc, char *argv[])
{
Biobuf *b;
char *line;
```


```c
if(argc < 2) sysfatal("usage: prog file");
b = Bopen(argv[1], OREAD);
if(b == nil) sysfatal("Bopen: %r");
while((line = Brdline(b, '\n')) != nil){
line[Blinelen(b)-1] = 0;  /* strip newline */
print("%s\n", line);
}
Bterm(b);
exits(nil);
}
```


#### RELATED

- open, read (raw I/O)
- chartorune, runetochar (UTF-8 handling)


#### SOURCE

- /sys/man/2/bio (man.aiju.de/2/bio)
- /sys/src/libc/bio


## thread / Channel (LIBRARY)


#### CATEGORY


concurrency


#### SIGNATURE


```c
#include <thread.h>
/* Entry point: */
void threadmain(int argc, char *argv[])
int  threadcreate(void (*f)(void*), void *arg, uint stacksize)
int  proccreate(void (*f)(void*), void *arg, uint stacksize)
void threadexits(char *msg)
void threadexitsall(char *msg)
```


```c
Channel* chancreate(int elemsize, int bufsize)
void     chanfree(Channel *c)
int  send(Channel *c, void *v)
int  recv(Channel *c, void *v)
int  nbsend(Channel *c, void *v)
int  nbrecv(Channel *c, void *v)
void* recvp(Channel *c)
int   sendp(Channel *c, void *p)
ulong recvul(Channel *c)
int   sendul(Channel *c, ulong v)
int   alt(Alt *alts)
```


#### DESCRIPTION


libthread provides CSP-style concurrency. Programs use threadmain instead of main.


**threadcreate:** creates a new thread within the current OS process (shared memory).


**proccreate:** creates a new OS process (separate memory by default, uses rfork internally).


Both return a thread id. Typical stacksize: 8192; increase for recursive functions.


**chancreate(elemsize, bufsize):** creates a channel. bufsize=0 means synchronous


(rendezvous: send blocks until recv, and vice versa). bufsize>0 means asynchronous


with capacity bufsize.


**send/recv:** block until the operation can proceed. nbsend/nbrecv are non-blocking


and return 1 on success, 0 if would block.


**alt:** selects over multiple channels simultaneously. Fill Alt array, terminate with


{nil, nil, CHANEND}, call alt() which blocks until one is ready and returns its index.


#### RESOURCE MODEL

- Threads in the same process share memory (stack is unique per thread)
- Channels are typed; elemsize must match the actual element type
- Since Plan 9 I/O blocks an OS process, blocking reads in a thread stall all

threads in that process — use proccreate or ioproc(2) for I/O


#### WHEN TO USE


libthread is the preferred way to write concurrent Plan 9 programs.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>
#include <thread.h>
```


```c
Channel *results;
```


```c
void
worker(void *arg)
{
char *msg = arg;
/* do work */
sendp(results, smprint("done: %s", msg));
}
```


```c
void
threadmain(int argc, char *argv[])
{
results = chancreate(sizeof(void*), 0);
threadcreate(worker, "task1", 8192);
threadcreate(worker, "task2", 8192);
char *r1 = recvp(results);
char *r2 = recvp(results);
print("%s\n%s\n", r1, r2);
free(r1); free(r2);
threadexitsall(nil);
}
```


#### RELATED

- lock, qlock, rlock (synchronisation)
- ioproc(2) (non-blocking I/O helper)
- alt (select over multiple channels)
- Channel


#### SOURCE

- /sys/man/2/thread (man.aiju.de/2/thread)
- /sys/src/libthread


## lock / qlock / rwlock / rendezvous (LIBRARY + SYSCALL)


#### CATEGORY


synchronisation


#### SIGNATURE


```c
/* Spin lock (use rarely) */
void lock(Lock *l); void unlock(Lock *l); int canlock(Lock *l);
```


```c
/* Queuing mutex (preferred in threaded programs) */
void qlock(QLock *l); void qunlock(QLock *l); int canqlock(QLock *l);
```


```c
/* Reader-writer lock */
void rlock(RWLock *l); void runlock(RWLock *l);
void wlock(RWLock *l); void wunlock(RWLock *l);
```


```c
/* Condition variable (requires QLock) */
void rsleep(Rendez *r);
int  rwakeup(Rendez *r);
int  rwakeupall(Rendez *r);
```


```c
/* Kernel-level two-value exchange */
void* rendezvous(void *tag, void *val);
```


```c
/* Reference counting */
void incref(Ref *r); long decref(Ref *r);
```


#### DESCRIPTION


**Lock:** spin lock. Fast but wastes CPU while waiting. Blocks context switches between


libthread threads — use QLock instead in threaded programs.


**QLock:** queuing lock. Processes/threads block (sleep) rather than spin. At most one


holder at a time. This is the right default choice.


**RWLock:** multiple concurrent readers or one exclusive writer.


**Rendez:** condition variable, always used with a QLock. The QLock must be held when


calling rsleep; it is atomically released and re-acquired around the sleep.


**rendezvous:** kernel-level synchronization. Two processes present the same tag; values


are exchanged and both are awakened. libthread uses this internally for all its locking.


#### RESOURCE MODEL

- Lock, QLock, RWLock, Rendez are value types: embed in structs, zero-initialize
- Ref: embed, zero-initialize; decref returns new count (0 = last reference freed)
- rendezvous: returns (void*)~0 if interrupted by a note


#### WHEN TO USE

- QLock for mutual exclusion in most programs
- Lock only for very short critical sections where spin cost is negligible
- RWLock when reads far outnumber writes
- Rendez for condition variables (producer/consumer, etc.)


#### EXAMPLE


```c
typedef struct Cache Cache;
struct Cache {
QLock lk;
int   count;
char  *data;
};
```


```c
void
insert(Cache *c, char *item)
{
qlock(&c->lk);
c->count++;
/* update c->data */
qunlock(&c->lk);
}
```


#### RELATED

- thread (libthread uses these internally)
- rendezvous (kernel primitive all locks are built on)


#### SOURCE

- /sys/man/2/lock (man.9front.org/2/lock)
- /sys/src/libc/port/lock.c


## dup / pipe / seek (SYSTEM CALLS)


#### CATEGORY


file I/O / process setup


#### SIGNATURE


```c
int dup(int oldfd, int newfd)
int pipe(int fd[2])
vlong seek(int fd, vlong n, int type)
```


#### DESCRIPTION


**dup(oldfd, newfd):** returns a new fd referring to the same file as oldfd.


If newfd is -1, returns the lowest available fd. If newfd >= 0, closes newfd


first (if open) then duplicates. Used to redirect stdin/stdout/stderr before exec.


**pipe(fd[2]):** creates a bidirectional pipe. fd[0] for reading, fd[1] for writing.


Bidirectional: data written to fd[1] is readable from fd[0] and vice versa.


**seek(fd, n, type):** repositions the file offset. type 0 = absolute, 1 = relative,


2 = from end. Returns new position or -1 on error. Fails on directories; no-op on pipes.


#### EXAMPLE


```c
/* Redirect stdout to a file */
int fd = create("out.txt", OWRITE, 0644);
dup(fd, 1);   /* fd 1 (stdout) now refers to the file */
close(fd);
/* now print() writes to out.txt */
```


```c
/* Pipe between parent and child */
int pfd[2];
pipe(pfd);
if(rfork(RFPROC|RFFDG) == 0){
close(pfd[0]);
dup(pfd[1], 1);  /* child stdout → pipe write end */
close(pfd[1]);
exec("/bin/ls", argv);
}
close(pfd[1]);
/* parent reads from pfd[0] */
```


#### RELATED

- open, create, close
- rfork (RFFDG for fd table sharing)


#### SOURCE

- /sys/man/2/dup, /sys/man/2/pipe, /sys/man/2/seek (man.aiju.de)


## Rune / UTF-8 (TYPE + LIBRARY)


#### CATEGORY


Unicode / strings


#### SIGNATURE


```c
typedef unsigned int Rune;
```


```c
int   runetochar(char *buf, Rune *r)
int   chartorune(Rune *r, char *s)
int   runelen(long r)
int   fullrune(char *s, int n)
int   utflen(char *s)
char* utfrune(char *s, long c)
int   isalpharune(Rune r)
Rune  toupperrune(Rune r)
Rune  tolowerrune(Rune r)
#define UTFmax    4
#define Runeself  0x80
#define Runeerror 0xFFFD
```


#### DESCRIPTION


Plan 9 invented UTF-8. All string data in the system is UTF-8. char* strings


are always UTF-8. Rune is a 32-bit Unicode code point.


runetochar: encodes one Rune to UTF-8; returns bytes written (1–4).


chartorune: decodes one Rune from UTF-8 at s; returns bytes consumed.


runelen: returns the number of bytes needed to encode rune r.


fullrune: returns true if s[0..n-1] is a complete, valid UTF-8 rune.


utflen: counts the number of Runes in a UTF-8 string.


utfrune: find a Rune in a UTF-8 string (like strchr but Rune-aware).


Runes below Runeself (0x80) are identical to ASCII and encode as a single byte.


Runeerror (0xFFFD) is the Unicode replacement character — returned by chartorune


on invalid input.


#### WHEN TO USE

- Process text character by character, not byte by byte
- Use Bgetrune for rune-by-rune reading from a Biobuf
- Use utflen when you need character count, not byte count
- Use toupperrune/tolowerrune for case conversion


#### EXAMPLE


```c
/* Walk a UTF-8 string rune by rune */
void
walkrunes(char *s)
{
Rune r;
int n;
while(*s){
n = chartorune(&r, s);
s += n;
if(r == Runeerror && n == 1)
continue; /* invalid UTF-8 byte */
print("U+%04X\n", r);
}
}
```


#### RELATED

- bio (Bgetrune, Bputrune for buffered Rune I/O)
- print (%C, %S for Rune output)
- utf(6) (encoding specification)


#### SOURCE

- /sys/man/2/runestrdup (man.aiju.de)
- /sys/src/libc/port/rune.c


## ARGBEGIN / ARGEND (MACRO)


#### CATEGORY


argument parsing


#### SIGNATURE


```c
/* Used inside main(int argc, char *argv[]) */
ARGBEGIN{
case 'f':  flag = 1; break;
case 'n':  n = atoi(ARGF()); break;
default:   usage();
}ARGEND
```


#### DESCRIPTION


ARGBEGIN/ARGEND is a macro pair that parses command-line flags in Plan 9 style.


It modifies argc and argv in place to consume flags, leaving only non-flag arguments.


ARGF() consumes and returns the next argument (either the rest of the current


flag option, or the next argv element). It returns nil if no argument follows.


After ARGEND, argc/argv point to the remaining non-option arguments.


The macro also sets argv0 to the program name (argv[0]).


#### EXAMPLE


```c
void
main(int argc, char *argv[])
{
char *output = nil;
int verbose = 0;
```


```c
ARGBEGIN{
case 'v':  verbose++;  break;
case 'o':  output = ARGF(); if(!output) usage(); break;
default:   usage();
}ARGEND
```


```c
if(argc == 0) usage();
/* argv[0..argc-1] are now non-flag args */
exits(nil);
}
```


```c
static void usage(void){
fprint(2, "usage: prog [-v] [-o output] file...\n");
exits("usage");
}
```


#### RELATED

- exits (call from usage())


#### SOURCE

- /sys/include/libc.h


## sysfatal (LIBRARY)


#### CATEGORY


error handling


#### SIGNATURE


```c
void sysfatal(char *fmt, ...)
```


#### DESCRIPTION


Sysfatal prints a formatted error message to stderr (fd 2) and calls exits with


that message as the status. If argv0 is set (which ARGBEGIN does automatically),


it is printed before the message as "progname: message".


It is the idiomatic way to abort on fatal errors in Plan 9 programs.


#### EXAMPLE


```c
fd = open(path, OREAD);
if(fd < 0)
sysfatal("open %s: %r", path);   /* print + exit */
```


```c
/* Equivalent manual version: */
if(fd < 0){
fprint(2, "%s: open %s: %r\n", argv0, path);
exits("open failed");
}
```


#### RELATED

- exits, werrstr, errstr


#### SOURCE

- /sys/src/libc/port/sysfatal.c


# Part 2 — Code Patterns


**Ten Complete 9front C Programs**


> Each pattern is a complete, compilable program with explanatory notes. Patterns cover: minimal program, raw file I/O, buffered I/O with libbio, rune-safe Unicode processing, process spawning with rfork, namespace manipulation, service interaction (/net /proc /env), error propagation, full skeleton with ARGBEGIN/ARGEND, and a 9P file server. These form Layer 4 (Code Patterns) of the dataset.


# 9front Code Patterns


## Layer 4 — All required patterns with explanation


## Pattern 1: Minimal Program


Every 9front C program has this skeleton:


```c
#include <u.h>
#include <libc.h>
```


```c
void
main(int argc, char *argv[])
{
/* u.h MUST be first, libc.h MUST be second */
/* main returns void — not int */
exits(nil);     /* nil or "" = success */
}
```


**Rules:**

- #include <u.h> first — always
- #include <libc.h> second — always
- void main(...) — never int main
- exits(nil) at end — never return 0
- Compile: 8c -FVw prog.c && 8l -o 8.prog prog.8


## Pattern 2: File I/O (raw syscalls)


```c
#include <u.h>
#include <libc.h>
```


```c
void
main(int argc, char *argv[])
{
int in, out;
long n;
char buf[8192];
```


```c
if(argc != 3){
fprint(2, "usage: copy src dst\n");
exits("usage");
}
```


```c
in = open(argv[1], OREAD);
if(in < 0)
sysfatal("open %s: %r", argv[1]);
```


```c
out = create(argv[2], OWRITE, 0644);
if(out < 0)
sysfatal("create %s: %r", argv[2]);
```


```c
while((n = read(in, buf, sizeof buf)) > 0){
if(write(out, buf, n) != n)
sysfatal("write: %r");
}
if(n < 0)
sysfatal("read: %r");
```


```c
close(in);
close(out);
exits(nil);
}
```


**Key points:**

- open(path, OREAD) — open existing, read only
- create(path, OWRITE, perm) — create or truncate
- read returns 0 at EOF, -1 on error; short reads are normal
- write should return exactly n — if not, it's an error
- close — always close what you open


## Pattern 3: Buffered I/O with libbio


```c
#include <u.h>
#include <libc.h>
#include <bio.h>
```


```c
/* Count lines in a file — showing Biobuf usage */
static long
countlines(Biobuf *b)
{
long n = 0;
char *line;
while((line = Brdline(b, '\n')) != nil)
n++;
return n;
}
```


```c
void
main(int argc, char *argv[])
{
Biobuf *b;
long n;
int i;
```


```c
if(argc == 1){
/* wrap stdin */
Biobuf bin;
if(Binit(&bin, 0, OREAD) == Beof)
sysfatal("Binit: %r");
n = countlines(&bin);
print("%ld\n", n);
Bterm(&bin);
} else {
for(i = 1; i < argc; i++){
b = Bopen(argv[i], OREAD);
if(b == nil){
fprint(2, "%s: %r\n", argv[i]);
continue;
}
n = countlines(b);
print("%ld\t%s\n", n, argv[i]);
Bterm(b);
}
}
exits(nil);
}
```


**Key points:**

- Bopen opens by name; Binit wraps an existing fd
- Brdline(b, '\n') returns pointer into internal buffer (not malloc'd)
- Use Blinelen(b) to get line length; line[Blinelen(b)-1] = 0 strips newline
- Bterm flushes and frees (doesn't close fd for Binit buffers)
- Bgetrune for Unicode-aware character reading
- Bgetc returns int, Bgetrune returns long (both return Beof = -1 at EOF)


## Pattern 4: Rune-safe text processing


```c
#include <u.h>
#include <libc.h>
#include <bio.h>
```


```c
/* Count Unicode characters (not bytes) in a file */
static uvlong
countRunes(Biobuf *b)
{
uvlong n = 0;
long r;
while((r = Bgetrune(b)) != (long)Beof)
n++;
return n;
}
```


```c
/* Walk a UTF-8 string rune by rune */
static void
processstring(char *s)
{
Rune r;
int n;
while(*s){
n = chartorune(&r, s);
s += n;
if(r == Runeerror && n == 1)
continue;   /* invalid UTF-8 byte — skip */
/* process rune r */
if(isalpharune(r))
r = toupperrune(r);
/* encode back */
char buf[UTFmax+1];
buf[runetochar(buf, &r)] = 0;
print("%s", buf);
}
print("\n");
}
```


```c
void
main(void)
{
processstring("héllo wörld");
exits(nil);
}
```


**Key points:**

- Bgetrune reads one Unicode code point from a Biobuf (handles UTF-8 automatically)
- chartorune(&r, s) decodes one rune and returns bytes consumed
- runetochar(buf, &r) encodes one rune and returns bytes written
- UTFmax = 4 (max bytes per rune); always size rune buffers as char buf[UTFmax+1]
- Runeerror (0xFFFD) = invalid input; check n == 1 to distinguish genuine error
- isalpharune, toupperrune, tolowerrune — Unicode-aware character classification


## Pattern 5: Process spawning with rfork


```c
#include <u.h>
#include <libc.h>
```


```c
/* Run a command and wait for it */
static void
runcmd(char *cmd, char **args)
{
int pid;
Waitmsg *w;
```


```c
switch(pid = rfork(RFPROC|RFFDG|RFENVG|RFNOTEG)){
case -1:
sysfatal("rfork: %r");
case 0:
/* child */
exec(cmd, args);
sysfatal("exec %s: %r", cmd);   /* only reached on error */
}
/* parent: wait for child */
w = wait();
if(w == nil)
sysfatal("wait: %r");
if(w->msg[0] != '\0')
fprint(2, "%s exited: %s\n", cmd, w->msg);
free(w);
}
```


```c
/* Run command with stdout captured to a pipe */
static char*
cmdoutput(char *cmd, char **args)
{
int pfd[2], n, tot;
char buf[4096];
Waitmsg *w;
```


```c
pipe(pfd);
switch(rfork(RFPROC|RFFDG|RFENVG|RFNOTEG)){
case -1:
sysfatal("rfork: %r");
case 0:
close(pfd[0]);
dup(pfd[1], 1);   /* child stdout → write end */
close(pfd[1]);
exec(cmd, args);
sysfatal("exec: %r");
}
close(pfd[1]);
tot = 0;
while((n = read(pfd[0], buf+tot, sizeof buf-1-tot)) > 0)
tot += n;
close(pfd[0]);
buf[tot] = 0;
w = wait();
if(w && w->msg[0])
fprint(2, "%s: %s\n", cmd, w->msg);
free(w);
return strdup(buf);
}
```


```c
void
main(void)
{
char *args[] = {"date", nil};
runcmd("/bin/date", args);
```


```c
char *out = cmdoutput("/bin/date", args);
print("date said: %s", out);
free(out);
exits(nil);
}
```


**Key points:**

- rfork(RFPROC|RFFDG|RFENVG|RFNOTEG) is the standard "isolated child" fork
- exec(path, argv) does not return on success; always follow with sysfatal
- wait() returns malloc'd Waitmsg* — always free it
- w->msg[0] == '\0' means success; non-empty string means failure
- dup(fd, 1) redirects stdout; always close the original fd after dup
- RFNOTEG isolates the child's note group (keyboard interrupt won't kill parent)


## Pattern 6: Namespace manipulation


```c
#include <u.h>
#include <libc.h>
```


```c
/* Demonstrate name space manipulation */
void
main(void)
{
int fd;
```


```c
/*
* Example 1: Bind a device into /dev
* This makes the proc device visible in /dev
* (it's normally at /proc, but this shows the mechanism)
*/
if(bind("#p", "/proc", MREPL) < 0)
fprint(2, "bind: %r\n");
```


```c
/*
* Example 2: Union directory
* Add /rc/bin to /bin so rc scripts are found in $path
*/
if(bind("/rc/bin", "/bin", MAFTER) < 0)
sysfatal("bind: %r");
```


```c
/*
* Example 3: Private name space
* rfork(RFNAMEG) gives a private copy of the name space
* Changes don't affect other processes
*/
if(rfork(RFNAMEG) < 0)
sysfatal("rfork: %r");
/* now bind/mount won't affect other processes */
if(bind("/tmp", "/usr", MREPL) < 0)
sysfatal("bind: %r");
/* /usr is now /tmp in this process only */
```


```c
/*
* Example 4: Mount a 9P server from a pipe
*/
int pfd[2];
pipe(pfd);
switch(rfork(RFPROC|RFFDG)){
case 0:
close(pfd[1]);
/* child serves 9P on pfd[0] */
/* serve(pfd[0]); */
exits(nil);
default:
close(pfd[0]);
if(mount(pfd[1], -1, "/mnt/srv", MREPL, "") < 0)
sysfatal("mount: %r");
/* pfd[1] is closed by mount on success */
}
```


```c
exits(nil);
}
```


**Key points:**

- bind(new, old, flag): make old an alias for new
- mount(fd, afd, old, flag, aname): attach 9P server fd at old
- Flags: MREPL replace, MBEFORE prepend to union, MAFTER append to union
- MCREATE: OR with above — allow file creation in this union member
- rfork(RFNAMEG) copies the name space (private copy, changes don't propagate)
- rfork(RFCNAMEG) gives a clean name space (must build from scratch)
- unmount(nil, "/mnt/foo") unmounts everything at /mnt/foo


## Pattern 7: Service interaction (/net, /proc, /env)


```c
#include <u.h>
#include <libc.h>
```


```c
/* Read environment variable via /env */
static char*
envget(char *name)
{
char path[256];
int fd;
long n;
char *buf;
Dir *d;
```


```c
snprint(path, sizeof path, "/env/%s", name);
d = dirstat(path);
if(d == nil) return nil;
buf = malloc(d->length + 1);
free(d);
if(buf == nil) return nil;
fd = open(path, OREAD);
if(fd < 0){ free(buf); return nil; }
n = readn(fd, buf, d->length);
close(fd);
if(n < 0){ free(buf); return nil; }
buf[n] = 0;
return buf;  /* caller must free */
}
```


```c
/* Kill a process by writing to /proc/pid/note */
static int
killproc(int pid)
{
char path[64];
int fd;
snprint(path, sizeof path, "/proc/%d/note", pid);
fd = open(path, OWRITE);
if(fd < 0) return -1;
fprint(fd, "kill");
close(fd);
return 0;
}
```


```c
/* Read process name from /proc */
static char*
procname(int pid)
{
char path[64];
char buf[64];
int fd, n;
snprint(path, sizeof path, "/proc/%d/status", pid);
fd = open(path, OREAD);
if(fd < 0) return nil;
n = read(fd, buf, sizeof buf - 1);
close(fd);
if(n <= 0) return nil;
buf[n] = 0;
/* first field of status is the process name */
char *sp = strchr(buf, ' ');
if(sp) *sp = 0;
return strdup(buf);
}
```


```c
/* Make a TCP connection and exchange data */
static void
tcpexample(void)
{
int fd;
char buf[4096];
long n;
```


```c
fd = dial("tcp!example.com!80", nil, nil, nil);
if(fd < 0)
sysfatal("dial: %r");
fprint(fd, "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n");
while((n = read(fd, buf, sizeof buf)) > 0)
write(1, buf, n);
close(fd);
}
```


```c
void
main(void)
{
char *user = envget("user");
if(user){
print("user: %s\n", user);
free(user);
}
tcpexample();
exits(nil);
}
```


**Key points:**

- Everything is a file: environment in /env, processes in /proc, network in /net
- Write text commands to ctl files to control devices
- /proc/<pid>/note — write a note string to signal a process
- /proc/<pid>/status — text status (name, pid, ppid, state, etc.)
- /env/<name> — read/write environment variables directly
- dial("tcp!host!port", ...) — make TCP connections
- announce("tcp!*!port", dir) + listen + accept — serve TCP connections
- No special syscalls needed for these — just open/read/write/close


## Pattern 8: Complete program with all idioms


```c
#include <u.h>
#include <libc.h>
#include <bio.h>
```


```c
/* A complete program demonstrating all Plan 9 C idioms:
- ARGBEGIN/ARGEND option parsing
- error handling with %r and sysfatal
- Biobuf for I/O
- process spawning
- proper exits
*/
```


```c
static int verbose;
static int linenum;
```


```c
static void
usage(void)
{
fprint(2, "usage: %s [-v] [-n] file...\n", argv0);
exits("usage");
}
```


```c
static void
processfile(char *path)
{
Biobuf *b;
char *line;
long lno = 0;
```


```c
b = Bopen(path, OREAD);
if(b == nil){
fprint(2, "%s: %r\n", path);
return;
}
while((line = Brdline(b, '\n')) != nil){
lno++;
line[Blinelen(b)-1] = 0;   /* strip newline */
if(verbose)
fprint(2, "[debug] line %ld: %s\n", lno, line);
if(linenum)
print("%ld\t%s\n", lno, line);
else
print("%s\n", line);
}
Bterm(b);
if(verbose)
fprint(2, "[debug] %s: %ld lines\n", path, lno);
}
```


```c
void
main(int argc, char *argv[])
{
ARGBEGIN{
case 'v':   verbose++;  break;
case 'n':   linenum++;  break;
default:    usage();
}ARGEND
```


```c
if(argc == 0){
Biobuf bin;
char *line;
long lno = 0;
if(Binit(&bin, 0, OREAD) == Beof)
sysfatal("Binit stdin: %r");
while((line = Brdline(&bin, '\n')) != nil){
lno++;
line[Blinelen(&bin)-1] = 0;
if(linenum)
print("%ld\t%s\n", lno, line);
else
print("%s\n", line);
}
Bterm(&bin);
} else {
for(int i = 0; i < argc; i++)
processfile(argv[i]);
}
exits(nil);
}
```


## Pattern 9: Error propagation across call depth


```c
#include <u.h>
#include <libc.h>
```


```c
/*
* Rule: library functions set errstr and return -1 (or nil).
* Callers check the return value and can annotate the error
* with werrstr before propagating.
*/
```


```c
static int
readconfig(char *path, char **out)
{
int fd;
long n;
char *buf;
Dir *d;
```


```c
d = dirstat(path);
if(d == nil){
werrstr("readconfig %s: %r", path);
return -1;
}
buf = malloc(d->length + 1);
free(d);
if(buf == nil){
werrstr("readconfig: malloc: %r");
return -1;
}
fd = open(path, OREAD);
if(fd < 0){
werrstr("readconfig open %s: %r", path);
free(buf);
return -1;
}
n = readn(fd, buf, d->length);
close(fd);
if(n < 0){
werrstr("readconfig read %s: %r", path);
free(buf);
return -1;
}
buf[n] = 0;
*out = buf;
return 0;
}
```


```c
void
main(void)
{
char *config;
if(readconfig("/etc/myapp/config", &config) < 0)
sysfatal("%r");     /* %r expands the werrstr message */
print("config: %s\n", config);
free(config);
exits(nil);
}
```


**Key points:**

- Library functions: return -1/nil on error, set errstr with werrstr
- Use werrstr("context: %r") to annotate with context while preserving the original error
- %r in print verbs reads the current errstr automatically
- sysfatal("%r") prints the annotated error and exits
- Never use errno; never use perror; always use the errstr chain


## Pattern 10: 9P file server (user-level service)


```c
#include <u.h>
#include <libc.h>
#include <thread.h>
#include <9p.h>
```


```c
/*
* A minimal 9P file server exposing one file "data".
* Mounts at /mnt/demo when run.
*/
```


```c
static char *filedata = "hello from 9P\n";
```


```c
static void
fsread(Req *r)
{
readstr(r, filedata);   /* handles offset/count arithmetic */
respond(r, nil);        /* nil = success */
}
```


```c
static void
fswrite(Req *r)
{
char *newdata = mallocz(r->ifcall.count + 1, 1);
if(newdata == nil){
respond(r, "out of memory");
return;
}
memmove(newdata, r->ifcall.data, r->ifcall.count);
free(filedata);
filedata = newdata;
r->ofcall.count = r->ifcall.count;
respond(r, nil);
}
```


```c
Srv fs = {
.read  = fsread,
.write = fswrite,
};
```


```c
void
threadmain(int argc, char *argv[])
{
char *srvname = "demo";
char *mtpt = "/mnt/demo";
```


```c
ARGBEGIN{
case 's': srvname = ARGF(); break;
case 'm': mtpt    = ARGF(); break;
default:  fprint(2, "usage: demo [-s srvname] [-m mtpt]\n"); exits("usage");
}ARGEND
```


```c
fs.tree = alloctree(nil, nil, DMDIR|0777, nil);
createfile(fs.tree->root, "data", "none", 0666, nil);
```


```c
postmountsrv(&fs, srvname, mtpt, MREPL);
threadexitsall(nil);
}
```


**Key points:**

- lib9p handles the 9P protocol; you implement the file operations
- readstr(r, s) handles offset/count for string files automatically
- respond(r, nil) = success; respond(r, "error string") = failure
- alloctree creates an in-memory file tree; createfile adds files
- postmountsrv posts the server on /srv/srvname and mounts at mtpt
- threadmain is the entry point (lib9p uses libthread internally)
- Access from rc: cat /mnt/demo/data, echo hello >/mnt/demo/data


# Part 3 — Anti-POSIX Training


**16 WRONG → CORRECT → WHY Blocks**


> Each block teaches an AI agent to recognise and reject a POSIX or Linux hallucination and generate the correct Plan 9 idiom instead. Misconceptions covered: stdio/printf, fork(), signal(), errno, pthread, select/poll, exit(), getenv/setenv, unsafe string functions, malloc handling, open flags, struct stat, BSD sockets, ARGBEGIN, device files, and void main. This is Layer 5 (Anti-POSIX Training) of the dataset.


# 9front Anti-Patterns: POSIX Misconceptions Corrected


## Layer 5 — Anti-POSIX Training


## Every block teaches the AI to REJECT the wrong pattern and generate the correct one.


## POSIX MISCONCEPTION: #include <stdio.h> / printf / FILE*


### WRONG


```c
#include <stdio.h>
int main(void) {
printf("hello\n");
FILE *f = fopen("file.txt", "r");
char buf[256];
fgets(buf, sizeof buf, f);
fclose(f);
return 0;
}
```


### CORRECT


```c
#include <u.h>
#include <libc.h>
#include <bio.h>
void
main(void)
{
int fd;
Biobuf *b;
char *line;
```


```c
print("hello\n");
b = Bopen("file.txt", OREAD);
if(b == nil) sysfatal("Bopen: %r");
while((line = Brdline(b, '\n')) != nil){
line[Blinelen(b)-1] = 0;
print("%s\n", line);
}
Bterm(b);
exits(nil);
}
```


### WHY

- <stdio.h>, FILE*, printf, fopen, fgets, fclose, return 0 are POSIX/ANSI C
- Plan 9 does NOT use stdio for new code
- print goes to fd 1; fprint(fd, ...) to any fd; Biobuf for buffered I/O
- exits(nil) replaces return 0; exits("error") replaces return 1
- void main(void) not int main(void) — main returns void in Plan 9


## POSIX MISCONCEPTION: fork() for process creation


### WRONG


```c
#include <unistd.h>
pid_t pid = fork();
if(pid == 0){
/* child */
execv("/bin/ls", args);
exit(1);
}
/* parent: waitpid */
int status;
waitpid(pid, &status, 0);
if(WEXITSTATUS(status) != 0) ...
```


### CORRECT


```c
#include <u.h>
#include <libc.h>
int pid;
Waitmsg *w;
```


```c
switch(pid = rfork(RFPROC|RFFDG|RFENVG|RFNOTEG)){
case -1:
sysfatal("rfork: %r");
case 0:
/* child */
exec("/bin/ls", args);
sysfatal("exec: %r");
/* NOTREACHED */
}
/* parent */
w = wait();
if(w == nil) sysfatal("wait: %r");
if(w->msg[0])
fprint(2, "child failed: %s\n", w->msg);
free(w);
```


### WHY

- fork() exists in Plan 9 as rfork(RFFDG|RFREND|RFPROC) but gives no resource control
- rfork(flags) lets you specify exactly what is shared/copied/clean
- waitpid and WEXITSTATUS do not exist; use wait() which returns a Waitmsg*
- Exit status is a string, not an integer; w->msg[0] == 0 means success
- execv becomes exec(path, argv) — no PATH search, no v/p/l variants needed
- Children that exit with a non-empty string put it in w->msg


## POSIX MISCONCEPTION: signal() for event handling


### WRONG


```c
#include <signal.h>
static volatile int quit = 0;
void handler(int sig){ quit = 1; }
int main(void){
signal(SIGINT, handler);
signal(SIGTERM, handler);
while(!quit) do_work();
return 0;
}
```


### CORRECT


```c
#include <u.h>
#include <libc.h>
static int quit = 0;
```


```c
static int
notehndlr(void *ureg, char *msg)
{
USED(ureg);
if(strcmp(msg, "interrupt") == 0 || strcmp(msg, "hangup") == 0){
quit = 1;
return 1;   /* note consumed */
}
return 0;       /* pass to default */
}
```


```c
void
main(void)
{
atnotify(notehndlr, 1);
while(!quit)
do_work();
exits(nil);
}
```


### WHY

- signal(), SIGINT, SIGTERM, SIGPIPE etc. do not exist in native Plan 9
- Notes are strings, not integers: "interrupt", "hangup", "alarm", "kill"
- atnotify(f, 1) registers a handler; atnotify(f, 0) unregisters
- Handler returns 1 = handled, 0 = pass to next / default
- The lower-level notify(f) + noted(NCONT/NDFLT) also exists but atnotify is simpler


## POSIX MISCONCEPTION: errno for error checking


### WRONG


```c
#include <errno.h>
fd = open(path, O_RDONLY);
if(fd == -1){
if(errno == ENOENT)
fprintf(stderr, "not found\n");
else
perror("open");
exit(1);
}
```


### CORRECT


```c
fd = open(path, OREAD);
if(fd < 0){
char err[ERRMAX];
errstr(err, sizeof err);
if(strstr(err, "does not exist") != nil)
fprint(2, "not found\n");
else
fprint(2, "open %s: %s\n", path, err);
exits(err);
}
/* or more concisely: */
if(fd < 0)
sysfatal("open %s: %r", path);
```


### WHY

- errno, ENOENT, EINVAL, perror(), strerror() are POSIX only
- Plan 9 uses a per-process error string (max ERRMAX=128 bytes)
- Failed system calls set the error string; read it with errstr(buf, sizeof buf)
- %r in print verbs expands the error string automatically
- Error strings are human-readable: "file does not exist", "permission denied", etc.
- Do NOT compare against integer codes — compare against substrings if needed


## POSIX MISCONCEPTION: pthread for concurrency


### WRONG


```c
#include <pthread.h>
pthread_mutex_t mu = PTHREAD_MUTEX_INITIALIZER;
void *worker(void *arg){ pthread_mutex_lock(&mu); ... }
int main(void){
pthread_t t;
pthread_create(&t, nil, worker, arg);
pthread_join(t, nil);
}
```


### CORRECT


```c
#include <u.h>
#include <libc.h>
#include <thread.h>
QLock mu;
```


```c
void
worker(void *arg)
{
qlock(&mu);
/* critical section */
qunlock(&mu);
threadexits(nil);
}
```


```c
void
threadmain(int argc, char *argv[])
{
threadcreate(worker, arg, 8192);
/* channels are idiomatic for communication: */
Channel *c = chancreate(sizeof(void*), 0);
/* ... */
threadexitsall(nil);
}
```


### WHY

- pthread_* does not exist in native Plan 9
- libthread provides threads (threadcreate) and processes (proccreate)
- Mutex → QLock (qlock/qunlock); RW lock → RWLock (rlock/wlock)
- CSP channels (Channel) are idiomatic for communication between threads
- Entry point is threadmain, not main, when using libthread
- #include <thread.h> links libthread automatically via #pragma lib


## POSIX MISCONCEPTION: select()/poll() for multiplexing


### WRONG


```c
fd_set rfds;
FD_ZERO(&rfds);
FD_SET(fd1, &rfds);
FD_SET(fd2, &rfds);
select(maxfd+1, &rfds, nil, nil, nil);
if(FD_ISSET(fd1, &rfds)) handle1();
```


### CORRECT


```c
#include <u.h>
#include <libc.h>
#include <thread.h>
```


```c
/* In Plan 9: use a thread per blocking fd, communicate via channels */
Channel *ch = chancreate(sizeof(int), 8);
```


```c
void
reader1(void *arg)
{
int *fdp = arg, n, fd = *fdp;
char buf[512];
while((n = read(fd, buf, sizeof buf)) > 0)
/* send to channel or process inline */;
sendul(ch, -1);  /* signal EOF */
}
```


```c
/* Or use alt() to select over multiple channels: */
Alt alts[3];
int v1, v2;
alts[0] = (Alt){chan1, &v1, CHANRCV};
alts[1] = (Alt){chan2, &v2, CHANRCV};
alts[2] = (Alt){nil,  nil,  CHANEND};
switch(alt(alts)){
case 0: /* chan1 received v1 */ break;
case 1: /* chan2 received v2 */ break;
}
```


### WHY

- select(), poll(), epoll() are POSIX and do not exist in native Plan 9
- The Plan 9 idiom is: one thread/process per blocking resource
- Threads communicate results over typed Channels
- alt() in libthread selects over multiple channels (like select but typed)
- This model is cleaner: each concern is isolated, no fd-set management


## POSIX MISCONCEPTION: exit(0) / exit(1)


### WRONG


```c
int main(int argc, char *argv[]){
if(argc < 2){ fprintf(stderr, "usage\n"); exit(1); }
/* ... */
exit(0);
}
```


### CORRECT


```c
void
main(int argc, char *argv[])
{
ARGBEGIN{ ... }ARGEND
if(argc < 1){ fprint(2, "usage: prog file\n"); exits("usage"); }
/* ... */
exits(nil);
}
```


### WHY

- exit(int) does not exist in native Plan 9; use exits(char *)
- exits(nil) or exits("") = success
- exits("usage"), exits("error") = failure with message
- The exit string is passed to the parent via wait()
- main is void not int; you cannot return a value from it


## POSIX MISCONCEPTION: getenv("HOME") / setenv


### WRONG


```c
#include <stdlib.h>
char *home = getenv("HOME");
setenv("DEBUG", "1", 1);
```


### CORRECT


```c
#include <u.h>
#include <libc.h>
char *home = getenv("home");      /* lowercase in Plan 9 */
if(home == nil) home = "/tmp";
free(home);                       /* getenv returns malloc'd string */
```


```c
putenv("debug", "1");             /* putenv, not setenv */
```


### WHY

- Plan 9 environment variable names are conventionally lowercase
- getenv returns a **malloc'd** string (caller must free) or nil
- setenv does not exist; use putenv(name, value)
- The environment is stored as files in /env; any write to /env/foo sets $foo in rc
- Common names: home, user, path, objtype, cputype, service, sysname


## POSIX MISCONCEPTION: strlen/strcmp/strcpy are safe


### WRONG


```c
char buf[64];
strcpy(buf, input);                 /* no bounds check */
char *s = malloc(strlen(a)+strlen(b)+1);
sprintf(s, "%s%s", a, b);          /* sprintf */
```


### CORRECT


```c
char buf[64];
strncpy(buf, input, sizeof buf - 1);
buf[sizeof buf - 1] = 0;
```


```c
/* Or use smprint for dynamic allocation: */
char *s = smprint("%s%s", a, b);    /* malloc, exact size, free() after */
free(s);
```


```c
/* Or seprint for fixed buffers: */
char buf[256];
char *p = seprint(buf, buf+sizeof buf, "%s%s", a, b);
/* p points to NUL terminator; buf is always NUL-terminated */
```


### WHY

- strcpy has no bounds checking — buffer overflow
- sprintf has no bounds checking — use snprint, seprint, or smprint
- Plan 9's smprint malloc's exactly the right size (and the caller must free it)
- seprint(buf, end, fmt, ...) writes to [buf, end), always NUL-terminates
- These are safe by construction, not by convention


## POSIX MISCONCEPTION: malloc failure is rare / ignore it


### WRONG


```c
char *buf = malloc(1024);
strcpy(buf, data);   /* no nil check */
```


### CORRECT


```c
char *buf = mallocz(1024, 1);   /* zero-fills */
if(buf == nil)
sysfatal("malloc: %r");
/* or use smprint which handles it: */
char *s = smprint("...");
/* smprint never returns nil — calls sysfatal internally on OOM */
```


### WHY

- In Plan 9, malloc returns nil on failure (like POSIX) but the convention is to

check it

- mallocz(size, 1) zero-fills (second arg = clear flag)
- smprint, strdup, etc. handle allocation internally and abort on OOM
- sysfatal prints the error and exits — no recovery needed for OOM in typical programs


## POSIX MISCONCEPTION: O_RDONLY | O_CREAT | O_TRUNC flags


### WRONG


```c
#include <fcntl.h>
int fd = open(path, O_RDWR | O_CREAT | O_TRUNC, 0644);
```


### CORRECT


```c
/* In Plan 9, open and create are separate system calls */
/* Open existing file: */
int fd = open(path, ORDWR);
```


```c
/* Create new (or truncate existing): */
int fd = create(path, ORDWR, 0644);
```


```c
/* Create only if not exists (atomic): */
int fd = create(path, ORDWR|OEXCL, 0644);
```


```c
/* Create a directory: */
int fd = create(dirpath, OREAD, DMDIR|0755);
```


### WHY

- O_RDONLY, O_WRONLY, O_RDWR, O_CREAT, O_TRUNC are POSIX flags — not Plan 9
- Plan 9 uses OREAD(0), OWRITE(1), ORDWR(2), OEXEC(3) for open modes
- OTRUNC, OCEXEC, ORCLOSE are OR'd modifiers (no O_ prefix)
- open only opens existing files; create creates new or truncates existing
- For atomic exclusive creation, use create(path, mode|OEXCL, perm) which fails if

the file already exists


## POSIX MISCONCEPTION: stat() returns struct stat


### WRONG


```c
#include <sys/stat.h>
struct stat st;
if(stat(path, &st) == 0){
printf("size=%ld\n", st.st_size);
if(S_ISDIR(st.st_mode)) ...;
}
```


### CORRECT


```c
Dir *d = dirstat(path);
if(d == nil){
/* file does not exist or error */
sysfatal("dirstat %s: %r", path);
}
print("size=%lld\n", d->length);
if(d->mode & DMDIR) { /* is a directory */ }
free(d);   /* d is malloc'd — must free! */
```


### WHY

- struct stat, st_size, st_mode, S_ISDIR etc. are POSIX — not Plan 9
- Plan 9 uses Dir* from dirstat(path) or dirfstat(fd)
- Dir is a malloc'd struct with char* fields (name, uid, gid, muid) in the same block
- free(d) frees both the struct and all embedded strings — one free call
- File length is vlong (64-bit), not off_t; use %lld to print it


## POSIX MISCONCEPTION: struct sockaddr / socket / connect


### WRONG


```c
#include <sys/socket.h>
#include <netinet/in.h>
int fd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in addr = {AF_INET, htons(80), ...};
connect(fd, (struct sockaddr*)&addr, sizeof addr);
```


### CORRECT


```c
/* Plan 9: all networking through /net file system */
int fd = dial("tcp!example.com!80", nil, nil, nil);
if(fd < 0) sysfatal("dial: %r");
/* fd is the data file — read/write directly */
```


### WHY

- BSD sockets (socket, connect, bind, sockaddr, htons, etc.) are POSIX — not Plan 9
- Plan 9 uses the /net file system and the dial library function
- Address format: "network!host!service" e.g. "tcp!192.168.1.1!80"
- Use net as network to try all available networks automatically
- dial internally uses /net/tcp/clone and writes connect addr to the ctl file
- Server side: announce("tcp!*!port", dir) → listen(dir, newdir) → accept(ctl, dir)


## POSIX MISCONCEPTION: argc/argv handling without ARGBEGIN


### WRONG


```c
int main(int argc, char *argv[]) {
int i, verbose = 0;
char *output = NULL;
for(i = 1; i < argc; i++){
if(strcmp(argv[i], "-v") == 0) verbose = 1;
else if(strcmp(argv[i], "-o") == 0 && i+1 < argc)
output = argv[++i];
else if(argv[i][0] == '-'){
fprintf(stderr, "unknown flag\n"); exit(1);
}
}
}
```


### CORRECT


```c
void
main(int argc, char *argv[])
{
int verbose = 0;
char *output = nil;
```


```c
ARGBEGIN{
case 'v': verbose++; break;
case 'o': output = ARGF(); if(!output) usage(); break;
default:  usage();
}ARGEND
/* argv[0..argc-1] are now the non-flag arguments */
exits(nil);
}
```


### WHY

- Plan 9 provides ARGBEGIN/ARGEND macros specifically for option parsing
- ARGF() gets the option argument (next argv or rest of current flag)
- argv0 is automatically set to the program name
- After ARGEND, argc/argv skip past all flags to non-option arguments
- This is the idiomatic Plan 9 way — always use it


## POSIX MISCONCEPTION: /dev/null, /dev/zero, /dev/random and device files


### WRONG


```c
/* Assuming device files behave identically to Linux */
int fd = open("/dev/tty", ORDWR);     /* does not exist */
int fd = open("/dev/urandom", OREAD); /* may not exist */
/* Writing ioctl commands to control a device: */
ioctl(fd, TIOCSWINSZ, &ws);
```


### CORRECT


```c
/* Plan 9 device files that DO exist: */
int fd = open("/dev/null",   ORDWR);   /* reads EOF, discards writes */
int fd = open("/dev/zero",   OREAD);   /* reads zeros */
int fd = open("/dev/random", OREAD);   /* random bytes (non-blocking) */
int fd = open("/dev/cons",   ORDWR);   /* current process's console */
int fd = open("/dev/mouse",  OREAD);   /* mouse events (in rio) */
```


```c
/* Control devices by writing TEXT to ctl files — no ioctl: */
int ctl = open("/dev/eia0ctl", OWRITE);
fprint(ctl, "b9600");   /* set 9600 baud */
close(ctl);
```


```c
/* Kernel devices accessed as #X special paths: */
bind("#c", "/dev", MAFTER);   /* bind console device into /dev */
bind("#p", "/proc", MREPL);   /* bind proc device */
```


### WHY

- /dev/tty does not exist — use /dev/cons (always refers to YOUR console)
- ioctl() does not exist in Plan 9 — all device control is via text written to ctl files
- The /dev namespace is **per-process**: bound by the window system at process start
- rio binds /dev/cons, /dev/mouse, /dev/draw per window automatically
- Kernel devices are the kernel's own file systems; access via #X syntax or after binding
- /dev/random is non-blocking in 9front (no distinction between random and urandom)


## POSIX MISCONCEPTION: Returning from main without exits


### WRONG


```c
int main(int argc, char *argv[]){
/* ... */
return 0;         /* or return 1; */
}
```


### CORRECT


```c
void
main(int argc, char *argv[])
{
/* ... */
exits(nil);       /* success */
}
/* or on error: */
exits("some error message");
```


### WHY

- main must be declared void in Plan 9 (not int)
- You cannot return a string from a function by returning it — hence void main
- The exit status is the string argument to exits, not a return value
- exits(nil) or exits("") = success (equivalent to exit(0))
- exits("anything") = failure; the string goes into w->msg in the parent's wait()


# Part 4 — Symbol Map Quick Reference


**107 Typed Symbols**


> Every system call, library function, type, macro and constant in the dataset. Critical symbols are those an AI agent must know to write any Plan 9 C program. Full metadata including related functions, flags, and source paths is in symbol_map.jsonl.


### File Io


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| readn | library function | long readn(int fd, void *buf, long nbytes) | high |
| pread | system call | long pread(int fd, void *buf, long nbytes, vlong offset) | high |
| pwrite | system call | long pwrite(int fd, void *buf, long nbytes, vlong offset) | high |
| dup | system call | int dup(int oldfd, int newfd) | high |
| pipe | system call | int pipe(int fd[2]) | high |
| seek | system call | vlong seek(int fd, vlong n, int type) | medium |
| remove | system call | int remove(char *file) | medium |
| open | system call | int open(char *file, int omode) | critical |
| create | system call | int create(char *file, int omode, ulong perm) | critical |
| close | system call | int close(int fd) | critical |
| read | system call | long read(int fd, void *buf, long nbytes) | critical |
| write | system call | long write(int fd, void *buf, long nbytes) | critical |


### Process


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| fork | library function | int fork(void) | high |
| execl | library function | int execl(char *name, ...) | medium |
| _exits | system call | void _exits(char *msg) | medium |
| await | system call | int await(char *s, int n) | medium |
| waitpid | library function | int waitpid(void) | medium |
| rfork | system call | int rfork(int flags) | critical |
| exec | system call | int exec(char name, char argv[]) | critical |
| exits | system call | void exits(char *msg) | critical |
| wait | library function | Waitmsg* wait(void) | critical |


### Namespace


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| unmount | system call | int unmount(char name, char old) | medium |
| fauth | system call | int fauth(int fd, char *aname) | medium |
| bind | system call | int bind(char name, char old, int flag) | critical |
| mount | system call | int mount(int fd, int afd, char old, int flag, char aname) | critical |


### Error Handling


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| ERRMAX | constant | maximum error string length in bytes | high |
| rerrstr | library function | void rerrstr(char *err, uint nerr) | medium |
| errstr | system call | int errstr(char *err, uint nerr) | critical |
| werrstr | library function | void werrstr(char *fmt, ...) | critical |
| sysfatal | library function | void sysfatal(char *fmt, ...) | critical |


### Notification


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| notify | system call | int notify(void (f)(void, char*)) | high |
| noted | system call | int noted(int v) | high |
| postnote | library function | int postnote(int group, int pid, char *note) | medium |
| alarm | system call | ulong alarm(ulong millisecs) | medium |
| atnotify | library function | int atnotify(int (f)(void, char*), int in) | critical |


### Networking


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| announce | library function | int announce(char addr, char dir) | high |
| listen | library function | int listen(char dir, char newdir) | high |
| accept | library function | int accept(int ctl, char *dir) | high |
| netmkaddr | library function | char netmkaddr(char addr, char defnet, char defservice) | medium |
| hangup | library function | int hangup(int ctl) | low |
| dial | library function | int dial(char addr, char local, char dir, int cfdp) | critical |


### Buffered Io


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| Brdstr | library function | char Brdstr(Biobufhdr bp, int delim, int nulldelim) | high |
| Bgetc | library function | int Bgetc(Biobufhdr *bp) | high |
| Bprint | library function | int Bprint(Biobufhdr bp, char fmt, ...) | high |
| Bflush | library function | int Bflush(Biobufhdr *bp) | medium |
| Bopen | library function | Biobuf Bopen(char name, int mode) | critical |
| Binit | library function | int Binit(Biobuf *bp, int fd, int mode) | critical |
| Brdline | library function | void Brdline(Biobufhdr bp, int delim) | critical |
| Bgetrune | library function | long Bgetrune(Biobufhdr *bp) | critical |
| Bterm | library function | int Bterm(Biobufhdr *bp) | critical |
| Beof | constant | returned by Bgetc, Bgetrune, Binit on EOF or error | critical |


### Thread


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| proccreate | library function | int proccreate(void (f)(void), void *arg, uint stacksize) | high |
| alt | library function | int alt(Alt *alts) | high |
| threadmain | entry point | void threadmain(int argc, char *argv[]) | critical |
| threadcreate | library function | int threadcreate(void (f)(void), void *arg, uint stacksize) | critical |
| threadexits | library function | void threadexits(char *msg) | critical |
| threadexitsall | library function | void threadexitsall(char *msg) | critical |
| chancreate | library function | Channel* chancreate(int elemsize, int bufsize) | critical |
| send | library function | int send(Channel c, void v) | critical |
| recv | library function | int recv(Channel c, void v) | critical |


### Synchronisation


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| rendezvous | system call | void rendezvous(void tag, void *val) | high |
| rlock | library function | void rlock(RWLock *l) | high |
| wlock | library function | void wlock(RWLock *l) | high |
| lock | library function | void lock(Lock *l) | medium |
| rsleep | library function | void rsleep(Rendez *r) | medium |
| incref | library function | void incref(Ref *r) | medium |
| decref | library function | long decref(Ref *r) | medium |
| qlock | library function | void qlock(QLock *l) | critical |
| qunlock | library function | void qunlock(QLock *l) | critical |


### File Metadata


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| dirfstat | library function | Dir* dirfstat(int fd) | high |
| dirread | library function | int dirread(int fd, Dir **buf) | high |
| dirreadall | library function | long dirreadall(int fd, Dir **buf) | high |
| Qid | struct | path unique per server; vers increments on each write | high |
| dirwstat | library function | int dirwstat(char name, Dir dir) | medium |
| nulldir | library function | void nulldir(Dir *d) | medium |
| dirstat | library function | Dir dirstat(char name) | critical |
| Dir | struct | from dirstat(); free() once frees struct and all embedded strings | critical |


### Formatting


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| snprint | library function | int snprint(char s, int len, char fmt, ...) | high |
| seprint | library function | char seprint(char s, char e, char fmt, ...) | high |
| smprint | library function | char smprint(char fmt, ...) | high |
| sprint | library function | int sprint(char s, char fmt, ...) | low |
| print | library function | int print(char *fmt, ...) | critical |
| fprint | library function | int fprint(int fd, char *fmt, ...) | critical |


### Unicode


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| utflen | library function | int utflen(char *s) | high |
| Runeself | constant | Runes < Runeself are single-byte ASCII-compatible | high |
| Runeerror | constant | Unicode replacement character; returned on invalid input | high |
| fullrune | library function | int fullrune(char *s, int n) | medium |
| Rune | typedef | one Unicode code point (32-bit) | critical |
| chartorune | library function | int chartorune(Rune r, char s) | critical |
| runetochar | library function | int runetochar(char buf, Rune r) | critical |
| UTFmax | constant | maximum bytes needed to encode any Rune | critical |


### Argument Parsing


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| argv0 | global variable | program name (set by ARGBEGIN); used by sysfatal in error messages | high |
| ARGBEGIN | macro | starts option parsing loop in main(); modifies argc/argv in place; se… | critical |
| ARGEND | macro | ends ARGBEGIN block; after it, argc/argv point to non-flag arguments | critical |
| ARGF | macro | returns next argument as string (rest of flag or next argv); nil if m… | critical |


### Memory


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| mallocz | library function | void* mallocz(ulong size, int clr) | high |
| malloc | library function | void* malloc(ulong size) | critical |
| free | library function | void free(void *p) | critical |


### Types


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| uvlong | typedef | 64-bit unsigned integer; print with %ulld | high |
| ulong | typedef | unsigned long; print with %lud | high |
| nelem | macro | nelem(array) | high |
| USED | macro | USED(x) | medium |
| vlong | typedef | 64-bit signed integer; print with %lld | critical |
| nil | constant | Plan 9's NULL; defined as ((void*)0) | critical |


### Environment


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| getenv | library function | char getenv(char name) | high |
| putenv | library function | int putenv(char name, char val) | medium |


### Strings


| Symbol | Type | Signature / Description | Importance |
| --- | --- | --- | --- |
| tokenize | library function | int tokenize(char str, char *args, int maxargs) | high |


# Part 5 — Integration Guide


**Using the Dataset with RAG, SFT & Code Completion**


> Complete instructions for integrating this dataset into three use modes: RAG pipeline (vector database embedding and retrieval), SFT fine-tuning (generating instruction pairs from the structured files), and symbol-aware code completion. Includes the full topic taxonomy and live man page URL list for keeping the dataset current.


# 9front Dataset — Integration Guide


## How to use all five files to train or power an AI agent


## What Each File Is For


| File | Format | Use case |
| --- | --- | --- |
| plan9_dataset.md | Markdown | Human-readable reference; system prompt injection; fine-tuning source |
| plan9_dataset.jsonl | JSONL | RAG vector database; chunk retrieval; semantic search |
| symbol_map.jsonl | JSONL | Type-safe API lookup; symbol resolution; code completion hints |
| code_patterns.md | Markdown | Few-shot examples; code generation prompts |
| anti_patterns.md | Markdown | Negative examples; POSIX rejection training |


## Option A: RAG Pipeline (Recommended for Production)


### Step 1 — Embed the chunks


```c
import json
```


```c
chunks = []
with open('plan9_dataset.jsonl') as f:
for line in f:
if line.strip():
chunks.append(json.loads(line))
```


```c
# Each chunk has: id, topic, content, source, embedding_tags, importance
# Embed 'content' field with your embedding model
# Store id, topic, embedding_tags, importance as metadata
```


```c
for chunk in chunks:
embedding = embed(chunk['content'])   # your embedding function
vector_db.upsert(
id=chunk['id'],
vector=embedding,
metadata={
'topic':          chunk['topic'],
'source':         chunk['source'],
'importance':     chunk['importance'],
'embedding_tags': chunk['embedding_tags'],
}
)
```


### Step 2 — Query strategy


```c
def query_9front(user_question, top_k=5):
# Boost critical chunks
results = vector_db.query(
vector=embed(user_question),
top_k=top_k * 2,
filter={'importance': {'$in': ['critical', 'high']}}
)
# Always include the reasoning protocol chunk
reasoning = vector_db.fetch('ai_reasoning_...')  # always prepend this
return [reasoning] + results[:top_k]
```


### Step 3 — System prompt construction


```c
SYSTEM = """You are an expert 9front/Plan 9 C programmer.
You write ONLY native Plan 9 C code — never POSIX, never Linux, never BSD.
```


```c
MANDATORY RULES:
1. Always #include <u.h> first, then #include <libc.h>
2. main() returns void; exit with exits(nil) for success, exits("msg") for failure
3. Error handling: check return values; use %r in format strings for errstr
4. Process creation: rfork(flags), not fork()/pthread
5. Signals: atnotify(), not signal()
6. I/O: open/read/write or Biobuf, never FILE* or stdio
7. Networking: dial()/announce()/listen()/accept(), never socket()/connect()
```


```c
Retrieved context:
{retrieved_chunks}
"""
```


## Option B: Fine-Tuning (SFT)


### Generating instruction pairs from the dataset


```c
import json, re
```


```c
# Extract WRONG/CORRECT pairs from anti_patterns.md
anti = open('anti_patterns.md').read()
blocks = re.split(r'(?=## POSIX MISCONCEPTION)', anti)
```


```c
pairs = []
for block in blocks:
if '## POSIX MISCONCEPTION' not in block:
continue
title = re.search(r'POSIX MISCONCEPTION: (.+)', block).group(1).strip()
wrong_m   = re.search(r'### WRONG\n```c(.+?)```', block, re.DOTALL)
correct_m = re.search(r'### CORRECT\n```c(.+?)```', block, re.DOTALL)
why_m     = re.search(r'### WHY\n(.+?)(?=\n###|\n---|\Z)', block, re.DOTALL)
```


```c
if wrong_m and correct_m and why_m:
pairs.append({
"instruction": f"Write Plan 9 C code to: {title}",
"input": f"Wrong POSIX approach:\n```c{wrong_m.group(1)}```",
"output": f"Correct Plan 9 approach:\n```c{correct_m.group(1)}```\n\nExplanation:\n{why_m.group(1).strip()}",
"source": "anti_patterns.md"
})
```


```c
# Extract examples from code_patterns.md
code = open('code_patterns.md').read()
patterns = re.split(r'(?=## Pattern \d+)', code)
for p in patterns[1:]:
title_m = re.match(r'## Pattern \d+: (.+)', p)
if not title_m:
continue
title = title_m.group(1).strip()
code_m = re.search(r'```c(.+?)```', p, re.DOTALL)
if code_m:
pairs.append({
"instruction": f"Write a complete 9front C program that demonstrates: {title}",
"input": "",
"output": f"```c{code_m.group(1)}```",
"source": "code_patterns.md"
})
```


```c
# Extract from structured blocks in plan9_dataset.md
md = open('plan9_dataset.md').read()
sections = re.split(r'\n(?=## )', md)
for sec in sections[1:]:
lines = sec.split('\n')
title = lines[0].lstrip('#').strip()
sig_m    = re.search(r'### SIGNATURE\n```c(.+?)```', sec, re.DOTALL)
desc_m   = re.search(r'### DESCRIPTION\n(.+?)(?=\n###)', sec, re.DOTALL)
when_m   = re.search(r'### WHEN TO USE\n(.+?)(?=\n###)', sec, re.DOTALL)
example_m= re.search(r'### EXAMPLE\n```c(.+?)```', sec, re.DOTALL)
```


```c
if sig_m and desc_m:
pairs.append({
"instruction": f"Explain the Plan 9 {title} API and show how to use it correctly",
"input": "",
"output": (
f"## {title}\n\n"
f"**Signature:**\n```c{sig_m.group(1)}```\n\n"
f"**Description:**\n{desc_m.group(1).strip()}\n\n"
+ (f"**When to use:**\n{when_m.group(1).strip()}\n\n" if when_m else "")
+ (f"**Example:**\n```c{example_m.group(1)}```" if example_m else "")
),
"source": "plan9_dataset.md"
})
```


```c
print(f"Generated {len(pairs)} instruction pairs")
with open('plan9_sft_pairs.jsonl', 'w') as f:
for p in pairs:
f.write(json.dumps(p, ensure_ascii=False) + '\n')
```


## Option C: Symbol-Aware Code Completion


```c
import json
```


```c
symbols = {}
with open('symbol_map.jsonl') as f:
for line in f:
if line.strip():
s = json.loads(line)
symbols[s['symbol']] = s
```


```c
def get_signature(name):
s = symbols.get(name)
if s:
return s.get('signature', f"/* see man.aiju.de/2/{name} */")
return None
```


```c
def get_related(name):
s = symbols.get(name)
if s:
return s.get('related', [])
return []
```


```c
def get_header(name):
s = symbols.get(name)
if s:
return s.get('header', '<libc.h>')
return '<libc.h>'
```


```c
# Query: what do I need to make a TCP connection?
sym = symbols.get('dial')
print(f"dial: {sym['signature']}")
print(f"related: {sym['related']}")
print(f"header: {sym['header']}")
print(f"importance: {sym['importance']}")
print()
```


```c
# Check if something sets errstr (for error handling code gen)
for name in ['open', 'rfork', 'exits', 'rendezvous']:
s = symbols.get(name)
print(f"{name}: sets_errstr={s.get('sets_errstr', '?')}")
```


## Chunk Importance Tiers


Use importance field to prioritise retrieval:


| Importance | Meaning | Count |
| --- | --- | --- |
| critical | Always retrieve for any 9front question | 42 chunks |
| high | Retrieve for relevant questions | 34 chunks |
| medium | Retrieve for specific queries | 1 chunk |


For a system with limited context window, always include critical chunks first.


## Topic Taxonomy


Query by topic field for targeted retrieval:


| Topic | What it covers |
| --- | --- |
| process | rfork, exec, exits, wait, notes |
| file_io | open, create, read, write, seek, dup, pipe |
| namespace | bind, mount, unmount, rfork namespace flags |
| error_handling | errstr, werrstr, sysfatal, ERRMAX |
| networking | dial, announce, listen, accept |
| buffered_io | Biobuf, Bopen, Brdline, Bgetrune |
| concurrency | libthread, Channel, threadcreate |
| synchronisation | QLock, RWLock, rendezvous, Rendez |
| file_metadata | Dir, Qid, dirstat, dirwstat |
| unicode | Rune, chartorune, utflen, UTFmax |
| formatting | print, fprint, seprint, smprint, %r |
| anti_posix | WRONG→CORRECT misconception blocks |
| code_pattern | Complete working example programs |
| ai_reasoning | Step-by-step reasoning protocol |
| 9p | 9P protocol, file server programming |
| kernel | Dev interface, Chan, kernel driver structure |
| security | factotum, auth, TLS, secstore |


## Man Page URLs (live source)


All man page content was fetched from https://man.aiju.de/. To refresh:


```c
man.aiju.de/2/open       man.aiju.de/2/read      man.aiju.de/2/fork
man.aiju.de/2/stat       man.aiju.de/2/bind       man.aiju.de/2/errstr
man.aiju.de/2/exits      man.aiju.de/2/wait       man.aiju.de/2/notify
man.aiju.de/2/print      man.aiju.de/2/dial       man.aiju.de/2/lock
man.aiju.de/2/rendezvous man.aiju.de/2/bio        man.aiju.de/2/thread
man.aiju.de/2/exec       man.aiju.de/2/dup        man.aiju.de/2/pipe
man.aiju.de/2/seek       man.aiju.de/2/postnote   man.aiju.de/2/getfields
man.aiju.de/2/dirread    man.aiju.de/2/remove     man.aiju.de/2/alarm
man.aiju.de/2/fauth      man.aiju.de/3/proc       man.aiju.de/3/ip
man.aiju.de/4/namespace  man.aiju.de/5/intro
```


## Coverage Summary


| Layer (spec) | Status | Detail |
| --- | --- | --- |
| L1: Core Knowledge | ✅ Complete | 24 man pages verbatim from man.aiju.de |
| L2: Structured Markdown | ✅ Complete | 20 concept blocks, all fields present |
| L3: Symbol Map | ✅ Complete | 107 symbols, all required fields |
| L4: Code Patterns | ✅ Complete | 10 patterns × all 7 required types |
| L5: Anti-POSIX | ✅ Complete | 16 balanced WRONG/CORRECT/WHY blocks |
| L6: AI Reasoning | ✅ Complete | Explicit 4-step protocol in dedicated chunk |
| L7: RAG Chunks | ✅ Complete | 77 chunks, normalised topics, all fields |


All 47 spec requirements: **47/47 ✅**


# Part 6 — Supplementary Reference


> This section fills gaps identified in the topic taxonomy: the AI reasoning protocol, directory reading, non-blocking I/O, authentication/security basics, and string tokenisation. All content follows the same structured format as Part 1.


## AI Reasoning Protocol (CONCEPT)


#### CATEGORY


ai_reasoning


#### DESCRIPTION


When an AI agent is asked to write or review Plan 9 C code, it must apply this 4-step protocol before producing any output. Skipping any step causes POSIX hallucinations.


**Step 1 — Identify the target system**

Ask: Is this native Plan 9 / 9front, APE (ANSI/POSIX Environment), or hosted (Linux/macOS)?

- Native: use ONLY libc.h, bio.h, thread.h, 9p.h — no POSIX headers ever
- APE: POSIX is available but explicitly opted-in via `/bin/ape/psh` or `#pragma lib "..."`
- Hosted: standard POSIX rules apply — this manual does not apply

**Step 2 — Reject POSIX reflexes**

For every API you are about to write, check against this list of forbidden POSIX symbols:

| Forbidden | Plan 9 replacement |
| --- | --- |
| stdio.h, printf, FILE*, fopen | print, fprint, Biobuf, Bopen |
| fork() | rfork(RFPROC\|RFFDG\|RFENVG\|RFNOTEG) |
| signal(), SIGINT | atnotify(), "interrupt" note string |
| errno, ENOENT, perror | errstr(), werrstr(), %r verb |
| pthread_create | threadcreate() or proccreate() |
| select(), poll(), epoll() | alt() or one thread per blocking fd |
| exit(0), exit(1) | exits(nil), exits("message") |
| getenv / setenv | getenv (malloc'd, must free) / putenv |
| socket(), connect() | dial() |
| struct stat, stat() | Dir*, dirstat() |
| int main() | void main() |
| return 0 from main | exits(nil) |

**Step 3 — Select the correct include chain**

Every Plan 9 C file starts with exactly this order:

```c
#include <u.h>      /* ALWAYS first — defines base types */
#include <libc.h>   /* ALWAYS second — syscalls and library */
/* then optional, in any order: */
#include <bio.h>    /* buffered I/O */
#include <thread.h> /* libthread — also changes entry to threadmain */
#include <9p.h>     /* 9P file server library */
```

Never include POSIX headers (`<stdio.h>`, `<unistd.h>`, `<stdlib.h>`, `<string.h>`, etc.) in native Plan 9 code. All standard functionality is in `<libc.h>`.

**Step 4 — Verify the skeleton**

Every program must have this exact skeleton:

```c
#include <u.h>
#include <libc.h>

void
main(int argc, char *argv[])
{
    ARGBEGIN{ ... }ARGEND
    /* work */
    exits(nil);
}
```

If using libthread, replace `main` with `threadmain` and `exits` with `threadexitsall`.


#### RELATED

- Anti-POSIX training (Part 3) — 16 WRONG→CORRECT→WHY blocks
- ARGBEGIN/ARGEND — argument parsing
- errstr, werrstr — error handling


#### SOURCE

- This document (ai_reasoning topic, Layer 6)


## dirread / dirreadall (LIBRARY)


#### CATEGORY


file metadata / directory listing


#### SIGNATURE


```c
#include <u.h>
#include <libc.h>

int   dirread(int fd, Dir **buf)
long  dirreadall(int fd, Dir **buf)
```


#### DESCRIPTION


dirread reads one "chunk" of directory entries from an open directory fd (opened with `open(path, OREAD)`). It allocates a buffer via malloc, sets `*buf` to point to it, and returns the number of `Dir` entries in that chunk. Returns 0 at end of directory. Returns -1 on error.

dirreadall reads the **entire** directory in one call. Allocates a single malloc'd array of `Dir` structs, sets `*buf`, returns the total count. Returns -1 on error. This is the preferred function for small to medium directories.

The returned `*buf` must be freed with a single `free(*buf)` call — all Dir structs and their embedded strings (name, uid, gid, muid) are packed into one allocation.

Entries are returned in the order the server sends them (usually inode order). There is no guarantee of alphabetical order — sort if needed.


#### RESOURCE MODEL

- `*buf`: single malloc'd block; one `free(*buf)` frees everything
- Entries: `Dir` structs packed sequentially; strings point into the same block
- Error: returns -1, sets errstr; `*buf` is nil


#### WHEN TO USE

- List the contents of a directory
- Find files matching a pattern in a directory
- Get metadata for all entries without calling dirstat per entry
- dirreadall: when you want everything at once (simpler)
- dirread in a loop: when the directory may be very large


#### WHEN NOT TO USE

Do not call dirstat in a loop when you want all entries — use dirreadall instead.

Do not assume alphabetical order — sort the returned array if you need it.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>

/* List a directory — like a minimal ls */
void
main(int argc, char *argv[])
{
    Dir *buf;
    long n, i;
    char *dir;
    int fd;

    dir = argc > 1 ? argv[1] : ".";

    fd = open(dir, OREAD);
    if(fd < 0)
        sysfatal("open %s: %r", dir);

    n = dirreadall(fd, &buf);
    close(fd);
    if(n < 0)
        sysfatal("dirreadall: %r");

    for(i = 0; i < n; i++){
        if(buf[i].mode & DMDIR)
            print("%s/\n", buf[i].name);
        else
            print("%-20s %lld bytes\n", buf[i].name, buf[i].length);
    }
    free(buf);
    exits(nil);
}
```


```c
/* dirread in a loop (for large directories) */
void
scandir(char *path)
{
    Dir *buf;
    int fd, n, i;

    fd = open(path, OREAD);
    if(fd < 0)
        sysfatal("open %s: %r", path);

    while((n = dirread(fd, &buf)) > 0){
        for(i = 0; i < n; i++)
            print("%s\n", buf[i].name);
        free(buf);
    }
    if(n < 0)
        sysfatal("dirread: %r");
    close(fd);
}
```


#### RELATED

- dirstat, dirfstat (single file metadata)
- Dir, Qid (the structures returned)
- open (must open directory with OREAD before calling)


#### SOURCE

- /sys/man/2/dirread (man.aiju.de/2/dirread)
- /sys/src/libc/9sys/dirread.c


## ioproc (LIBRARY)


#### CATEGORY


concurrency / non-blocking I/O


#### SIGNATURE


```c
#include <u.h>
#include <libc.h>
#include <thread.h>

typedef struct Ioproc Ioproc;

Ioproc* ioproc(void)
void     closeioproc(Ioproc *io)

long     ioread(Ioproc *io, int fd, void *buf, long n)
long     iowrite(Ioproc *io, int fd, void *buf, long n)
int      ioopen(Ioproc *io, char *path, int mode)
int      iocreate(Ioproc *io, char *path, int mode, ulong perm)
int      ioclose(Ioproc *io, int fd)
int      iodial(Ioproc *io, char *addr, char *local, char *dir, int *cfdp)
long     iosleep(Ioproc *io, long ms)
```


#### DESCRIPTION


libthread threads within the same OS process share that process's kernel thread. A blocking system call (read, dial, etc.) in one thread blocks **all** threads in that process — not just the calling one.

`ioproc` solves this by wrapping each blocking call in a dedicated OS process (via `proccreate` internally). The calling thread sends the I/O request over a channel to the ioproc, which performs the blocking call in its own process, then sends the result back. This allows other threads to continue running while the I/O is pending.

**ioproc():** spawns a new I/O helper process. Returns an `Ioproc*` handle.

**closeioproc(io):** shuts down the helper process and frees the handle.

**ioread, iowrite, etc.:** drop-in replacements for the corresponding system calls. They submit the request to the ioproc, block the calling *thread* (not the process), and return the result.

**iosleep(io, ms):** sleep for ms milliseconds without blocking other threads.


#### RESOURCE MODEL

- Each `Ioproc*` is a separate OS process — do not share across threads without synchronisation
- One ioproc per thread is the typical pattern
- Closing the ioproc kills its helper process
- Error: same as underlying syscall (returns -1, sets errstr)


#### WHEN TO USE

- Any blocking I/O inside a `threadcreate` thread that must not stall other threads
- Network connections inside threads: use `iodial` instead of `dial`
- Timed operations: `iosleep` instead of `sleep` in threads


#### WHEN NOT TO USE

Do not use ioproc when the entire program is single-threaded — use plain syscalls.

Do not share one `Ioproc*` between multiple threads without a mutex — each call uses internal state.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>
#include <thread.h>

typedef struct Arg Arg;
struct Arg {
    char *addr;
    Channel *done;
};

void
fetcher(void *v)
{
    Arg *a = v;
    Ioproc *io;
    int fd;
    char buf[512];
    long n;

    io = ioproc();

    fd = iodial(io, a->addr, nil, nil, nil);
    if(fd < 0){
        fprint(2, "dial %s: %r\n", a->addr);
        closeioproc(io);
        sendp(a->done, nil);
        return;
    }

    /* ioread doesn't block other threads */
    n = ioread(io, fd, buf, sizeof buf - 1);
    ioclose(io, fd);
    closeioproc(io);

    if(n > 0){
        buf[n] = 0;
        sendp(a->done, strdup(buf));
    } else {
        sendp(a->done, nil);
    }
}

void
threadmain(int argc, char *argv[])
{
    Channel *done = chancreate(sizeof(void*), 0);
    Arg a = { "tcp!example.com!80", done };

    threadcreate(fetcher, &a, 16384);

    char *result = recvp(done);
    if(result){
        print("%s\n", result);
        free(result);
    }
    threadexitsall(nil);
}
```


#### RELATED

- thread, Channel (libthread concurrency)
- dial, read, write (the underlying syscalls wrapped by ioproc)
- proccreate (what ioproc uses internally)


#### SOURCE

- /sys/man/2/ioproc (man.aiju.de/2/ioproc)
- /sys/src/libthread/ioproc.c


## factotum / auth (LIBRARY + DAEMON)


#### CATEGORY


security / authentication


#### SIGNATURE


```c
#include <u.h>
#include <libc.h>
#include <auth.h>

AuthInfo* auth_proxy(AuthGetkey *getkey, char *fmt, ...)
AuthInfo* auth_userpasswd(char *user, char *passwd)
void      auth_freeAI(AuthInfo *ai)

int       fauth(int fd, char *aname)
int       mount(int fd, int afd, char *old, int flag, char *aname)
```


#### DESCRIPTION


Plan 9 security is built on **factotum**, a per-user authentication agent that runs as a separate file server at `/mnt/factotum`. Programs never handle raw passwords or keys directly — they delegate all credential operations to factotum via the auth library.

**factotum** holds credentials (passwords, keys, certificates) for the current user. It speaks authentication protocols on behalf of the program. If a credential is needed and not cached, factotum prompts the user or the caller supplies a getkey function.

**auth_proxy(getkey, fmt, ...):** the primary authentication function. Performs mutual authentication using the protocol and parameters described by fmt (a factotum key template string). Returns a malloc'd `AuthInfo*` on success, nil on failure. The caller must free it with `auth_freeAI`.

AuthInfo contains:
```c
typedef struct AuthInfo {
    char    *cuid;      /* caller's identity */
    char    *suid;      /* server's identity */
    char    *cap;       /* capability (optional) */
    uchar   *secret;    /* shared secret (optional) */
    int     nsecret;
} AuthInfo;
```

**auth_userpasswd(user, passwd):** simplified password check — authenticates user/password against the auth server. Returns AuthInfo* or nil. Useful for simple password-based programs.

**fauth(fd, aname):** low-level — opens an authentication file descriptor for a mount. Pass the result as `afd` to `mount()`. Most programs use the auth library instead.

**Factotum key template format:**
```
proto=p9sk1 dom=yourdomain user=username
proto=dp9ik dom=yourdomain user=username
proto=pass  user=username
```


#### RESOURCE MODEL

- factotum runs as a per-user process; its file tree is at `/mnt/factotum`
- `AuthInfo*` is malloc'd — call `auth_freeAI(ai)` when done
- Credentials are never stored in the program — factotum owns them
- Link with: `#pragma lib "<auth.h>"` is implicit when you include auth.h with 9front


#### WHEN TO USE

- Any program that needs to verify a user's identity
- 9P servers that require authenticated mounts
- Programs that need to pass authentication to a remote service
- Reading `/mnt/factotum/ctl` to check or add keys programmatically


#### WHEN NOT TO USE

Do not store passwords in environment variables or files — use factotum.

Do not implement authentication protocols yourself — factotum handles them.

Do not use PAM, NSS, or `/etc/passwd` — they do not exist in Plan 9.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>
#include <auth.h>

/* Authenticate the current user with a password */
void
main(void)
{
    AuthInfo *ai;
    char *user;

    user = getenv("user");
    if(user == nil)
        sysfatal("no $user set");

    /* Ask factotum to authenticate user with a password */
    ai = auth_userpasswd(user, "mysecretpassword");
    if(ai == nil)
        sysfatal("authentication failed: %r");

    print("authenticated: cuid=%s suid=%s\n", ai->cuid, ai->suid);
    auth_freeAI(ai);
    free(user);
    exits(nil);
}
```


```c
/* Check factotum is running */
void
checkfactotum(void)
{
    int fd = open("/mnt/factotum/ctl", OREAD);
    if(fd < 0)
        sysfatal("factotum not running: %r");
    close(fd);
}
```


#### RELATED

- secstore (encrypted key storage for factotum)
- fauth, mount (authenticated 9P mounts)
- /mnt/factotum/ctl (factotum control file)
- auth(2) man page


#### SOURCE

- /sys/man/2/auth (man.aiju.de/2/auth)
- /sys/src/libauth
- factotum(4) — the factotum daemon


## tokenize / getfields (LIBRARY)


#### CATEGORY


strings / argument parsing


#### SIGNATURE


```c
#include <u.h>
#include <libc.h>

int tokenize(char *str, char **args, int maxargs)
int getfields(char *str, char **args, int maxargs, int multisep, char *sep)
```


#### DESCRIPTION


**tokenize(str, args, maxargs):** splits str in-place on whitespace (spaces and tabs). Modifies str by inserting NUL terminators. Fills `args[0..n-1]` with pointers into str. Returns the number of tokens found, up to maxargs. Handles quoted strings: `'single quoted'` and `"double quoted"` spans are treated as single tokens with the quotes stripped. This is the standard Plan 9 way to parse simple configuration lines or command output.

**getfields(str, args, maxargs, multisep, sep):** more general version. sep is a string of separator characters. If multisep is non-zero, consecutive separators are treated as one (like awk's default). If multisep is zero, each separator character produces a field boundary (like `cut`). Does NOT handle quoting. Returns field count.


#### RESOURCE MODEL

- Both functions modify `str` in-place — pass a copy if you need the original
- `args[]` point into `str` — do not free them separately
- Both return the number of tokens/fields found (may be less than maxargs)


#### WHEN TO USE

- tokenize: parsing configuration files, splitting shell-like command strings
- getfields: parsing CSV-like data, colon-separated paths, tab-delimited tables
- Both: splitting the output of a command captured via a pipe


#### WHEN NOT TO USE

Do not use strtok (POSIX) — tokenize and getfields are the Plan 9 replacements.

Do not pass a string literal to either function — they modify the string in-place.


#### EXAMPLE


```c
#include <u.h>
#include <libc.h>

void
main(void)
{
    char line[] = "hello world  'foo bar'  baz";
    char *fields[16];
    int n, i;

    /* tokenize: whitespace split, handles quoting */
    n = tokenize(line, fields, nelem(fields));
    print("tokenize found %d tokens:\n", n);
    for(i = 0; i < n; i++)
        print("  [%d] = %s\n", i, fields[i]);
    /* output:
       [0] = hello
       [1] = world
       [2] = foo bar     <- quotes stripped, space preserved
       [3] = baz
    */

    /* getfields: split on colon (like $path) */
    char path[] = "/bin:/rc/bin:/usr/bin";
    char *parts[32];
    int m;

    m = getfields(path, parts, nelem(parts), 1, ":");
    print("path has %d components:\n", m);
    for(i = 0; i < m; i++)
        print("  %s\n", parts[i]);

    exits(nil);
}
```


```c
/* Parse /proc/<pid>/status first field (process name) */
char *
procname(int pid)
{
    char path[64], buf[256];
    char *fields[4];
    int fd, n;

    snprint(path, sizeof path, "/proc/%d/status", pid);
    fd = open(path, OREAD);
    if(fd < 0) return nil;
    n = read(fd, buf, sizeof buf - 1);
    close(fd);
    if(n <= 0) return nil;
    buf[n] = 0;

    /* status line: "name pid ppid group ..." */
    if(getfields(buf, fields, nelem(fields), 1, " \t") < 1)
        return nil;
    return strdup(fields[0]);
}
```


#### RELATED

- ARGBEGIN/ARGEND (command-line flag parsing)
- seprint, smprint (string building)
- Brdline (line reading from files to feed into tokenize)


#### SOURCE

- /sys/man/2/getfields (man.aiju.de/2/getfields)
- /sys/src/libc/port/tokenize.c

