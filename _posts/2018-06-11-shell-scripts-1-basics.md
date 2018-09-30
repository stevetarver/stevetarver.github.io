---
layout: post
title:  "Shell Scripts 1: Bourne back again"
date:   2018-06-12 11:11:11
tags:   bourne sh alpine linux devops docker kubernetes k8s
---

**OR...** _bullet proof Bourne scripting basics for developers and SREs in the cloud._

<img style="float: left; margin: 10px 30px 20px 0px;" src="/images/logo/sh.png">

Many developers struggle with shell commands and scripting because they don't do it enough to learn through repitition, and it isn't sexy enough to devote precious spare time to learning thoroughly. 

With a "particular set of skills", you can easily write, maintain, and debug scripts: ones without intermittent failures, that are easy to understand, and that allow SREs focus on actual problems instead of debugging your script.

If you don't already have the shell basics, and realistically won't spend the time to learn, this multi-part series will provide just enough information to confidently write production quality scripts for your containers, pods, and general operational tasks. 

The source backing this series is [here](https://github.com/stevetarver/shell-scripts), including a script to set up a targeted test environment - one line to set up and play along as you read.

**btw:** I commonly refer to business developers and operational developers (aka SREs) as just _developers_ cause I'm all about the DevOps ;)

## What kind of scripts?

Developers create docker images. Robust docker images make it easier to reach high SLOs and keep PagerDuty silent. Most directly, you can use these shell scripting skills to create:

* Docker container ENTRYPOINT scripts - aka `docker_entrypoint.sh`, container initialization scripts
* Docker container CMD scripts - application run scripts, internal container diagnostics, etc.
* Build or Makefile scripts
* Deploy scripts
* Operational scripts to manage and debug k8s nodes, underlying images and containers, etc.

**Buuuttt:** we're not going to talk much about Docker today. We are going to lay the foundation for writing rock-solid scripts by:

* Developing good software engineering practices.
* Scoping our toolchain and environment to limit knowledge required to be effective.
* Developing a test environment.
* Developing some debugging strategies.

The goal is not to be a bad-ass shell scripter; instead it is to know how to do a few things very well; a toolbox we can use to build scripts for all daily chores.

Most operators I've worked with can write amazingly complex scripts, but time pressures allow for getting the script to work for their use case only. This leads to one-off "tricky" constructs, toggling comments to enable different actions, leaving in dead code in case it is needed later; generally creating a mess that only they can deal with, for up to a month after original writing. No disrespect intended, I have lived with that time pressure - it murders quality. These articles aim to make a solid approach the most familiar, the one you fall back on instead of Stack Overflow.

## Which shell?

In the bare metal and VM world, `bash` is ubiquitous; in my container world, not so much. I try to make and deploy tiny infrastructure images derived from Alpine Linux or BusyBox. When other providers try to create a truly tiny docker image, they might try to save bash's 380KB footprint by omitting it. You can't count on `bash` being available.

I have also come to appreciate Bourne shell's lack of functionality. In Bash, there are many more options and equivalent constructs which means more for me to know and always some one-off way of doing things that I have to debug when sleepy - excess functionality == time sink. The Bourne shell provides enough functionality to do everything I want to do, AND pick a single best way of doing it - kinda similar to golang implementing `while` as `for boolean`.

When typing on the command line, you don't really care which shell you are in because you are simply launching other programs like `kubectl`, `docker`, `ls`, `ps`, `netstat`, `dig`, etc. But when scripting, the nuances between shells can make creation and debugging a time sink. Instead of trying to work in all shells, I pick the one I know is available everywhere - `sh`.

## Which Docker image?

Alpine Linux is beautiful! The base image is 4MB and provides `sh` and all of my shell scripting fundamentals: `grep`, `tr`, `cut`, `sed`, `awk`, `vi`, `yada` `yada++`. BusyBox provides these as well, but the Alpine package manager feels more familiar and has a robust package set should I need more advanced functionality.

Tiny is also beautiful because less features means less security attack surface. 

There are several Bourne shell variants, and some OS flavor differences between them. Because we are focusing on mainstream Bourne shell features, everything we do should work equally well in more robust images like Ubuntu.

Our first **good engineering practices** are:

* _Create a test environment that replicates production._
* _Create, evolve and test scripts in that test environment to avoid copy/paste/omission errors possible when your repo is not in the test environment._
* _Edit scripts in your favorite IDE for speed, accuracy, and developer joy._


Fortunately, in cloud environments, all three are trivial.

This docker command creates and starts an Alpine container and dumps us into `sh` on that container. It also mounts the current directory at `/tmp` and will remove the container when we exit the shell.

```
ᐅ docker run -it --rm   \
    --name alpine-test  \
    -v $(pwd):/tmp      \
    alpine:3.6          \
    sh -c 'cd /tmp; sh'

/tmp #
```
**NOTE**: You can easily test your scripts in other linux flavors by changing the image line.

If you are playing along, after you clone [the repo](https://github.com/stevetarver/shell-scripts), you can start the test environment from repo root with `./start_test_env.sh`.

## Rules of the game

To become a rock-solid script developer, we'll adopt the following strategy:

* Focus on single patterns that work everywhere.
* Don't be clever. Strive to write simple, clear code that anyone can understand.
* Be resilient and catch all edge cases.
* Test robustly.
* Do code katas to make fundamental tasks familiar, then incrementally add skills as needed.

## The base script template

Our first "single pattern" is the shell template itself - one template that I copy and paste to create each new script:

```bash
#!/bin/sh -e
#
# This line tells what I do
#
# This paragraph provides more details if necessary
#
# USE:
#   Show calling conventions here
#
# NOTES:
#   Note any assumptions, preconditions, side effects
#
# EXAMPLES:
#   Provide example use here
#
# EXIT CODES:
#   Describe all custom exit codes here
#
# CAVEATS:
#   Any warnings for script use?
#
THIS_SCRIPT_DIR=$(dirname $(readlink -f "${0}"))
(
    cd ${THIS_SCRIPT_DIR}
    # Do stuff in the dir the script lives in without changing pwd
    # Now that we know our dir, constructing relative paths is easy
    # NOTE: the parens isolate the change by shifting into a subshell
)

# You could do directory insensitive things here, but have a good
# reason for not just putting them into a single paren block

(
    cd ${THIS_SCRIPT_DIR}/..
    # Do some stuff in my parent dir without changing pwd
    # Each directory change should be scoped with ( )
)
```
See the [source code](https://github.com/stevetarver/shell-scripts/blob/master/script_template.sh).

Let's tear this apart... First note that we are not using the portable `#!/usr/bin/env` shebang common with `bash`, `ksh`, etc. That indirection is important for different flavors of unix as well as other operating systems to insulate scripts from different install locations. Anything that looks like a *nix shell should have `/usr/bin/env`. BUT, it will also have `/bin/sh`, so that indirection is unnecessary.

Next, note that our shebang has a `-e` flag - default error handling is turned on. In this mode, the script exits on the first command that fails and returns that exit code. If you need to turn off this default error handling, you should wrap only the statements that need error handling off with `set +e` and `set -e`, and document why. Examples are shown below.

The file comment block provides all the information a user needs to understand the script - what you would expect out of a man page. Having a standard layout makes the doc easier to scan. If a section has no comments, consider leaving it in to show that you considered it and that section is intentionally blank.

`THIS_SCRIPT_DIR=$(dirname $(readlink -f "${0}"))` provides the script's directory - the directory locator. Any script that works with relative paths needs to know its own location so it can reliably locate those other files or directories. 

There are many ways to do this, but they may be shell sensitive or fail in edge cases. This method will accurately identify the script directory when called:

* from the local directory
* from any other directory
* through a symbolic link
* and when the script is on your `${PATH}`

I include this construct for scripts that don't use relative paths, including a comment. Inevitably, someone will add a relative path command to one script in your fleet and only test it from `pwd`. That one script will be the one that generates a late night call 'cause sleepy people can't figure why things stopped working. Including the directory locator and scope blocks provides a reminder for maintainers to help avoid introducing the error, and simplifies fixing the script should the error be introduced.

Note how I treat commands and variables. I always wrap commands with `$( )` when I need their output. `$(pwd)` is functionally equivalent to `` `pwd` ``, but more clearly identifies the command. Imagine scanning a complex script for errors and trying to tell the difference between `` ` `` and `` ' `` - especially without syntax highlighting.

I always wrap variables with `${ }`. `${VAR}` returns the value of a variable, equivalent to `$VAR`, but more clearly indicates the variable. Quick typing may turn `$awhile` into `$a while`. In contrast, `${a while}` is easily identified as an error. Also note that I always use upper case variable names so it is easy to tell what is "supposed" to be a variable. Even without `${ }`, `$A WHILE` is easy to spot as an error.

The parens in the fragment below scope the directory change to only commands in the block - it creates a sub-shell. This adds a lot of clarity to the intended scope of the directory change, preserves the `pwd` the script was called from, and avoids a variety of maintenance injection errors.

```bash
(
   cd ${THIS_SCRIPT_DIR}
   # Do stuff in the dir the script lives in without changing pwd
)
```
The nice thing about having this standard is that once you get used to it, you notice every deviation, and any deviation is suspect.

## Scoping error checks

Our default is to have error checking on, but there are some cases where you don't care if the command fails. Let's invent a scenario: 

> Our build process requires project build scripts to remove container and image artifacts. Our build server has run out of disk space because individual projects don't always do this, or don't cleanup after build failures. Create a script to include in each build that ensures these artifacts are removed.

We expect the project build scripts to clean up after themselves, and we know that `docker rm -fv container-name` will error if the container does not exist. We want to include our script in every build job, but don't want our script to fail the build, especially for the common case. We also don't want to turn off default error checking for all other commands in the script.

The solution: turn off error checking for only the commands that can generate errors we don't care about.

```bash
# Intentionally disable error checking
set +e
docker rm -fv ${CONTAINER_NAME}
docker rmi -f ${IMAGE_NAME}
set -e
```

Software requirements met, but we can do better. 

**Good engineering practice**: _Tell users what you are doing, trap common errors, and then, describe the errors and possible solutions._

Below, we check if there are any containers related to this image, and then tell the user if we are, or are not removing any.

```bash
# Does the container exist?
CONTAINERS=$(docker ps -a -f "ancestor=${IMAGE_NAME}" | grep "${CONTAINER_NAME}" | cut -f1 -d' ')

if [ -n "${CONTAINERS}" ]; then
    echo "${ECHO_PREFIX} Stopping and removing ${CONTAINER_NAME} containers"
    docker rm -fv ${CONTAINERS}
else
    echo "${ECHO_PREFIX} No ${CONTAINER_NAME} containers running"
fi
```

See the [source code](https://github.com/stevetarver/shell-scripts/blob/master/fancy/docker_image_cleanup.sh) but note that it won't run in our test environment as configured - we didn't install docker or build any images there. I keep it around to clean up ad hoc builds on my mac.

## Exit codes

**Good engineering practices**: 

* _Return different exit codes for different types of errors._
* _Don't use reserved exit codes._

[Bash and other shells have many reserved exit codes](http://tldp.org/LDP/abs/html/exitcodes.html); Bourne is mercifully simple:

* `0`: success
* `1`: shell built-in failure
* `2`: shell syntax error
* `3`: untrapped signal received 

Exit code meanings vary widely by *nix flavor, and probably Bourne shell variant as well. The range `4 - 125` seems unreserved in all and is a good range for your custom exit codes. 

It is pretty common to `exit 1` on any failure. Resist! Exit code `1` is reserved for shell built-in failures. If `exit 1` always means "shell built-in" failure, you have a good hint at where to look in the failing script. Otherwise, the problem could be a shell built-in, any program that returns exit code 1, or any place the script calls `exit 1` - much harder to debug.

You might think that defining a dictionary of common error codes for your fleet is a great idea - would really help debugging if `100` always meant `file not found` for example. In practice, it won't be used pervasively and an inconsistently followed rule is not a rule - it is wasted effort because debuggers will always have to look at script documentation anyway.


## Catching exit codes

When your script calls another program with default error checking on, the other program may fail and the exit code is returned to the caller. The caller will not know the exit code origin and will have a hard time debugging the root cause.

**Good engineering practice**: 

* _Make it easy for callers to understand errors:_
    * _Prefer printing what you are doing, and how it failed._
    * _Separate data from informational output._
    * _Otherwise, document and return custom exit codes._

The clearest error documentation is printing what you are doing and why it failed. Scripts that produce output that is consumed or processed by other scripts should separate data from informational output so they can both easily process output and see errors inline with processing. There are some edge cases where this is not possible; those scripts must rely on communication through exit code. In all cases, trap exit codes from other programs and exit with your own documented custom code. It can be very difficult to identify the source of nested exit codes and find the doc describing it - your users will thank you for the additional effort.

There are three basic strategies:

1. Print data to `stdout` and information and failures to `stderr` - generally best.
1. Print data, information, and errors to `stdout` - acceptable if users will only view data, but that is usually short-sighted. If your script is useful, someone will want to automate it.
1. Print only data, document and return custom error codes - edge cases, requires extra effort debugging.

Let's take a look at implementations of each, in order.

We'll use a test script that simply exits with the code we provide: [`return_exit_code.sh`](https://github.com/stevetarver/shell-scripts/blob/master/examples/return_exit_code.sh). Because the script we call returns different error codes for different types of errors, we can separate errors we can tolerate from those we cannot.

```bash
THIS_SCRIPT_DIR=$(dirname $(readlink -f "${0}"))
(
    cd ${THIS_SCRIPT_DIR}

    # Default exit code for the script we call is 4
    INTERNAL_CODE=4
    # Set called script exit code if user supplied one
    [ -n "${1}" ] && INTERNAL_CODE=${1}

    echo "Calling: ./return_exit_code.sh ${INTERNAL_CODE}" >&2

    # Intentionally turn off default error checking for this script call
    set +e
    ./return_exit_code.sh ${INTERNAL_CODE}
    # capture the exit code - any operation will overwrite it - including set, test & echo
    EXIT_CODE=$?
    set -e
    # continue script execution with error checking on

    # provide blocks to handle each error code
    # Note: We *KNOW* we are working with integers so we do not wrap ${EXIT_CODE} with
    #       double quotes.
    if [ ${EXIT_CODE} -eq 0 ]; then
        # Do what ever with script success
        echo "return_exit_code.sh succeeded. Exit code: ${EXIT_CODE}" >&2
    elif [ ${EXIT_CODE} -eq 4 ]; then
        # Do what ever with script failure returning exit code 4
        echo "return_exit_code.sh failed, but that's safe to ignore. Continuing processing. Exit code: ${EXIT_CODE}" >&2
    else
        echo "return_exit_code.sh had an unrecoverable error. Aborting. Exit code: ${EXIT_CODE}" >&2
        exit 4
    fi
)
```
See the [source code](https://github.com/stevetarver/shell-scripts/blob/master/exit_code_check_1.sh).

Note that all informational and error msgs are redirected to `stderr` with `>&2`.

Since `stdout` and `stderr` are clearly separated, the caller has a lot of control over what to do with each.

They can ignore all errors, sending them to `/dev/null`:

```
./exit_code_check_1.sh 2>/dev/null
```

They can send `stderr` to an `errors.txt` file:

```
./exit_code_check_1.sh 2>errors.txt
```

Or they can send `stdout` to a `output.txt` and `stderr` to `errors.txt`:

```
./exit_code_check_1.sh 1>ouput.txt 2>errors.txt
```

Next up is a script that sends everything to `stdout`. Let's look at the parts of the script that change:

```bash
    # ...
    echo "Calling: ./return_exit_code.sh ${INTERNAL_CODE}"
    # ...

    if [ ${EXIT_CODE} -eq 0 ]; then
        # Do what ever with script success
        echo "return_exit_code.sh succeeded. Exit code: ${EXIT_CODE}"
    elif [ ${EXIT_CODE} -eq 4 ]; then
        # Do what ever with script failure returning exit code 4
        echo "return_exit_code.sh failed, but that's safe to ignore. Continuing processing. Exit code: ${EXIT_CODE}"
    else
        echo "return_exit_code.sh had an unrecoverable error. Aborting. Exit code: ${EXIT_CODE}"
        exit 4
    fi
```

See the [source code](https://github.com/stevetarver/shell-scripts/blob/master/exit_code_check_2.sh).

As expected, the only change is to remove `>&2` from each echo line.

In the final case, the only change is that all `echo` commands are removed.

See the [source code](https://github.com/stevetarver/shell-scripts/blob/master/exit_code_check_3.sh).

## Command logging (simple debugging)

The more complicated the script, the more difficult it is to tell what is actually going on. The `set -x` command turns on a command logger, printing each command executed to the terminal. There are two strategies:

* `#!/bin/sh -ex`: log every command executed in the script
* `set -x`: tightly scope command logging to the block bounded by `set -x` and `set +x`

This facility is rarely suitable for more than local debugging; `set -x` prints to stdout, so if your script reads and parses stdout, it will probably choke. Furthermore, the shebang version will print every command executed in the script. For other than trivial scripts, this will be overwhelmning. Your best use will probably be to tightly scope command logging around an interesting block - complex while/case blocks or if ladders.

Let's take it for a spin using [`./examples/return_exit_code.sh`](https://github.com/stevetarver/shell-scripts/blob/master/examples/return_exit_code.sh). This script takes an exit code as an argument and, optionally, lets you specify `set` args:

```bash
THIS_SCRIPT_DIR=$(dirname $(readlink -f "${0}"))
(
    # Set default values
    EXIT_CODE=4
    [ -n "${1}" ] && EXIT_CODE="${1}"

    exit ${EXIT_CODE}
)
```

It is pretty easy to match up script commands and args with command logging:

```bash
# ./return_exit_code.sh 2 -ex
+ readlink -f ./return_exit_code.sh
+ dirname /tmp/examples/return_exit_code.sh
+ THIS_SCRIPT_DIR=/tmp/examples
+ EXIT_CODE=4
+ [ -n 2 ]
+ EXIT_CODE=2
+ exit 2
```

Each command logged is preceded by `+ `. You probably recognize the first three lines as our standard directory identification. 

Wanna debug some of my code? Follow along with the [`./examples/set_x.sh` script](https://github.com/stevetarver/shell-scripts/blob/master/examples/set_x.sh).

I wrote a little script that simply prints the first argument if provided, does nothing if that first arg is missing or is the empty string:

```bash
if [ -n ${1} ]; then
    echo ${1}
fi
```
Take a minute - do you see any bugs? Any suspect code?

This works as expected:

```bash
# ./set_x.sh hello
hello
```
But, if I call it with no args or an empty string, it prints a newline - I don't want it to print anything with empty args:

```
# ./set_x.sh

# ./set_x.sh ''
```

And, if I try to print a command flag, it just prints a newline:

```
# ./set_x.sh -e
```
Let's focus on the last call and wrap the code with `set -x` and `set +x`:

```
+ [ -n -e ]
+ echo -e

+ set +x
```

We can see that `test` is identifying a string of non-zero length, and that `echo` is trying to print `-e`, but it is just not working. Time to google! You might eventually land on [Rich’s sh (POSIX shell) tricks](http://www.etalabs.net/sh_tricks.html) which describes how using `echo` introduce once-in-a-long-while bugs.  See more about this in the **TIPS ->** _Use printf on variables_ section.

If I look closely, I notice that I did not wrap the `test` variable in double quotes and I am using a string test. Given that, and our new knowledge of `echo`, I can improve the script by replacing it with:

```bash
if [ -n "${1}" ]; then
    printf "%s\\n" "${1}"
fi
```
OK, nice trivial example. But what would I do if I have a script that calls another script, that calls another script and there is a problem in calling conventions between the scripts? One strategy is to conditionally enable command logging in each script based on an environment variable:

```bash
[ -n "${DEBUG}" ] && set -x

# suspect code is here

[ -n "${DEBUG}" ] && set +x
```

Then I can easily compare output with and without command logging:

```
# Execute the script
./suspect_script.sh

# Execute the script with command logging
export DEBUG=1; ./suspect_script.sh
```

That's a decent start for debugging, but wait, there's more...

# Other debugging options

The Alpine Linux Bourne shell (ash) provides command switches we can use to learn about the script but these vary by flavor - to get the correct ones, see the `ash` [man page](https://linux.die.net/man/1/ash).

The interesting options are:

```
-n noexec   If not interactive, read commands but do not execute them.
            This is useful for checking the syntax of shell scripts.
-v verbose  The shell writes its input to standard error as it is read. 
            Useful for debugging.
-x xtrace   Write each command to standard error (preceded by a '+ ') 
            before it is executed. Useful for debugging.
```

**TODO** test and doc -u	Treat unset variables as an error when substituting.

Let's try them out on an obviously flawed script: [`bad_syntax.sh`](https://github.com/stevetarver/shell-scripts/blob/master/examples/bad_syntax.sh):

```bash
22 THIS_SCRIPT_DIR=$(dirname $(readlink -f "${0}"))
23 (
24    # Not directory dependent
25
26    if [ -eq "foo" ]then
27        echo "echo"
28    else
29        echo "bar"
30 )
```

First, the syntax check:

```
# sh -n bad_syntax.sh
bad_syntax.sh: line 28: syntax error: unexpected "else" (expecting "then")
# echo $?
2
```

Not bad, the shell told us where it got stuck and why. But it also bailed on the first error. Obviously this would require an iterative approach to solving syntax problems. This would probably be a good addition to a `git` pre-commit hook.

Next, command logging:

```
# sh -x bad_syntax.sh
+ readlink -f bad_syntax.sh
+ dirname /tmp/examples/bad_syntax.sh
+ THIS_SCRIPT_DIR=/tmp/examples
bad_syntax.sh: line 28: syntax error: unexpected "else" (expecting "then")
```

This can add a little value - it lists all commands that executed successfully, and then printed the syntax error on the `if` block. If you had a complex `if-then` ladder, this would help identify variables used when entering the problem code.

And finally, verbose:

```bash
# sh -v bad_syntax.sh
# ... file header comments omitted for clarity/brevity.
THIS_SCRIPT_DIR=$(dirname $(readlink -f "${0}"))
(
    # Not directory dependent

    if [ -eq "foo" ]then
        echo "echo"
    else
bad_syntax.sh: line 28: syntax error: unexpected "else" (expecting "then")
```

This is another interesting view: it shows every line executed successfully until the problem commands. You can readily see from the output that the `test` block is missing a command terminator `;` and there is no space between `]` and `then`.

The Bourne shell does a good job of identifying error location, until you source a script. Then line numbers reflect the merged script: all sourced scripts inserted at the source command line. You can make short work of finding the error location with the `-v` option with both stdout and stderr redirected to file:

```bash
sh -v ./script.sh &>out.txt
```

If you `vi` `out.txt`, and `<line num> gg>`, you can get right to that problem area.


## Tips

### Use printf on variables

Many use `echo` to print output consumed by other commands or pipe variables to other commands. This is fine until an argument starts with `\` or you have a newline sensitive command. Across Unix, POSIX, and various shells, echo has inconsistent and/or unspecified behavior. This is what happens in Alpine `sh`:

```bash
# echo "-e"

# echo "\\n"
\n
# echo "\n"
\n
# echo hello
hello
# echo -n hello
hello/ #
# echo -ne hello
hello/ #
```

Imagine trying to find a bug that once in a while includes one of these edge cases! The edge cases could be more common than you think. Consider a script that generates a Basic Auth header:

```
/ # echo steve.tarver:mypassword | base64
c3RldmUudGFydmVyOm15cGFzc3dvcmQK
/ # echo -n steve.tarver:mypassword | base64
c3RldmUudGFydmVyOm15cGFzc3dvcmQ=
```
Clearly these are different and one will likely not work:

```
# echo steve.tarver:mypassword | base64 | base64 -d
steve.tarver:mypassword
# echo -n steve.tarver:mypassword | base64 | base64 -d
steve.tarver:mypassword/ #
```

The first includes the newline in the string sent to `base64`, and the second will actually work. This error is more apparent if you use `printf`:

```
# printf "%s" steve.tarver:mypassword | base64
c3RldmUudGFydmVyOm15cGFzc3dvcmQ=
# printf "%s\\n" steve.tarver:mypassword | base64
c3RldmUudGFydmVyOm15cGFzc3dvcmQK
```
It is very clear that the second example includes a newline, and the display doesn't concatenate the output and the command prompt.

**TIP**: _Only use `echo` for informational displays where you can be certain of the arguments; use `printf` for variable manipulation._

## Epilog

That is a pretty solid base for writing simple command scripts. Stay tuned for part two where we look at more robust utility scripts including arg parsing, validation, and colored printing - managing complexity and creating a great UX.
