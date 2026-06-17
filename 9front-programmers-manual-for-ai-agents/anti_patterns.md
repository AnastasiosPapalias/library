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


