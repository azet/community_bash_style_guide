## hitchhikers guide to writing useful and modern bash scripts

### introduction

This is intended to be a community driven bash style and best practice guide. There are a lot of blog posts and articles out there, but they do not always agree on certain issues, and mostly lack hints and best practices to achieve a certain goal.  It's not that difficult to figure out a common strategy. so here it is. **please participate** (fork, open a pull request,..).

here's how you write bash code that somebody else will actually understand, is unit testable and will work in different environments no matter what. please read the mentioned articles, you will not regret it. furthermore people that will have to work with or maintain your scripts will not hate you in the future. 


##### general documentation, style guides, tutorials and articles:
* https://www.gnu.org/software/bash/manual/bashref.html
* http://wiki.bash-hackers.org/doku.php
* http://mywiki.wooledge.org/BashFAQ
* https://google-styleguide.googlecode.com/svn/trunk/shell.xml
* http://www.kfirlavi.com/blog/2012/11/14/defensive-bash-programming/
* http://mywiki.wooledge.org/BashWeaknesses 
* http://warewulf.lbl.gov/trac/wiki/Node%20Health%20Check#TipsandBestPracticesforChecks


##### linting and static analysis:
* http://www.shellcheck.net
* https://github.com/koalaman/shellcheck
* https://www.npmjs.org/package/grunt-lint-bash
* https://github.com/duggan/shlint
* http://manpages.ubuntu.com/manpages/natty/man1/checkbashisms.1.html

##### unit testing:
* https://code.google.com/p/shunit2/

##### debugging:
* `set -evx` and `bash -evx script.sh`
* http://bashdb.sourceforge.net/

### common mistakes and useful tricks

#### never use backticks
wrong:
```bash
`call_command_in_subshell`
```
correct:
```bash
$(call_command_in_subshell)
```

backticks are not POSIX compliant. they also cannot be nested without being escaped (which looks just insane):
```bash
$(call_command_in_subshell $(different_command $(yetanother_as_parameter)))
```
#### multiline pipe

instead of:
```bash
ls ${long_list_of_parameters} | grep ${foo} | grep -v grep | pgrep | wc -l | sort | uniq
```
do:
```bash
ls ${long_list_of_parameters} 
    | grep ${foo} 
    | grep -v grep 
    | pgrep 
    | wc -l 
    | sort 
    | uniq
```

#### overusing grep and `grep -v`
please never do that. there's almost certainly a better way to express this.


for example:
```bash
ps ax | grep ${processname} | grep -v grep 
```
versus:
```bash
pgrep ${processname}
```
#### using awk to print an element
stackexchange is full of this behavoir:

```bash
${listofthings} | awk '{ print $3 }' # get the third item
```
use bashisms instead:
```bash
${listofthings:3}

```

#### don't use `seq` for ranges
use `{x..y}` instead!

e.g.:
```bash
for k in {1..100}; do 
    $(do_awesome_stuff_with_input ${k})
done
```

the built-in range expression can do much more, see: http://wiki.bash-hackers.org/syntax/expansion/brace#ranges

#### use `printf` instead of `echo`
the bash builtin `printf` should be prefered to `echo` where possible. it does work like `printf` in C or any other high-level language, for reference see: http://wiki.bash-hackers.org/commands/builtin/printf

#### bash arithmetic instead of `expr`
bash offers the whole nine yards of arithmetic expressions directly as built-in bashisms.   

 **DO NOT USE `expr`**


for reference see:   
* http://wiki.bash-hackers.org/syntax/arith_expr
* http://www.softpanorama.org/Scripting/Shellorama/arithmetic_expressions.shtml


#### never use `bc(1)` for modulo operations
it will come to hurt you, trust me.

`bc(1)` does not properly handle modulo operations most of the time: https://superuser.com/questions/31445/gnu-bc-modulo-with-scale-other-than-0

#### using sockets with bash
although i do not really recommend it, it's possible to do simple (or even complex) socket operations in bash using the `/dev/tcp` and `/dev/udp` pseudo-devices: http://wiki.bash-hackers.org/syntax/redirection

example:
```bash
function recv() {
   local proto=${1} # tcp or udp
   local host=${2}  # hostname
   local port=${3}  # port number
   exec 3<>/dev/${proto}/${host}/${port}
   cat <&3
}

function send() {
   local msg=${1}
   echo -e ${msg} >&3
}

[...]
```

you may consider using `nc` (netcat) or even the far more advanced program `socat`: 
* http://www.dest-unreach.org/socat/doc/socat.html
* http://stuff.mit.edu/afs/sipb/machine/penguin-lust/src/socat-1.7.1.2/EXAMPLES


#### FIFO/named pipes
if you do not know what a named pipe is, please read this: http://wiki.bash-hackers.org/howto/redirection_tutorial


#### disown
`disown` is a bash built-in that can be used to remove a job from the job table of a bash script. for example, if you spawn a lot of sub processes, you can remove one or multiple of these processes with `disown` and the script will not care about it anymore.

see: https://www.gnu.org/software/bash/manual/bashref.html#index-disown

#### basic parallelism with `coproc`
usually people use `&` to send a process to the background and `wait` to wait for the process to finish. people then often use named pipes, files and global variables to communicate between the parent and sub programs. `coproc` can be used instead to have parallel jobs that can easily communicate with each other: http://wiki.bash-hackers.org/syntax/keywords/coproc

another excellent way to parallelize things in bash is by using GNU parallel: https://www.gnu.org/software/parallel/parallel_tutorial.html 
 
#### trapping signals and failing gracefully:
`trap` is used for signal handling in bash, a generic error handling function may be used like this:

```bash
readonly banner="my first bash project >>"
function fail() {
        # generic fail function for bash scripts
        # arg: 1 - custom error message
        # arg: 2 - file
        # arg: 3 - line number
        # arg: 4 - exit status
        echo "${banner} ERROR: ${1}."
        [[ ${2+defined} && ${3+defined} && ${4+defined} ]] && \
        echo "${banner} file: ${2}, line number: ${3}, exit code: ${4}. exiting!"
        
        # generic clean up code goes here (tempfiles, forked processes,..)

        exit 1
} ; trap 'fail "caught signal"' HUP KILL QUIT
```

```bash
do_stuff ${withinput} || fail "did not do stuff correctly" ${FILENAME} ${LINENO} $?
```

### final remarks
i will extend this over time. input very welcome.
