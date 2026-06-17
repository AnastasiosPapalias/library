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


