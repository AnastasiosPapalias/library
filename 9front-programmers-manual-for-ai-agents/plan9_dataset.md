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


