# locker

Easily add lock-file behaviour (so as to prevent concurrent running) to commands (particularly cronjobs).

    locker -n [name] [command]

# Usage

At simplest, add a lockfile (`/tmp/website` here) to a command:

    locker -n website php5 ~/public_html/cron.php

More complicatedly, send a mail on failure:

    locker -n website --on-failure '~/bin/mail_failure website' php5 ~/public_html/cron.php

Redirect stderr/stdout:

    locker -n website -e 'php5 ~/public_html/cron.php' --stdout ~/logs/webcron.log --stderr ~/logs/webcron.err

See `locker --help` (or `./help.txt`) for a more thorough explanation of all options, and `locker -h` 
for a quick listing.

# Command Execution

The command is executed using Perl's `IPC::Open3`; this means it is executed in a `sh` subshell and 
no sanitising or artificial re-creation of an enviornment is carried out. This ought to cause no 
surprises when simply adding this to a cronjob execution.

The command may be passed as a string to `-e` or `--execute`. If neither of these switches is used, 
any not-otherwise-processed arguments are joined together to create the command. A `--` may be used
to signal the end of locker's arguments: 

    locker -n dulog --stdout /home/du.log -- du -sh /home/*

# Output

Output is treated in three streams; `locker`'s own output, the stdout of the passed command and the 
stderr of the command. No distinction is made between the command given with `--execute` and any given
with `--on-failure` or `--on-success`.

For each, the command is executed and then its stdout is read and processed, and then its stderr. 

By default, everything is printed to stdout/stderr. Each line is prefixed with one of 'DEBUG', 'INFO'
or 'ERROR' for `locker`'s own output, and with 'STDOUT' for the command's standard out and 'STDERR'
for the command's standard error. 

This can be directed to a single file with `--log [path]`, or treated separately with `--stdout` and
`--stderr`:

    locker -n web -e 'bash php ~/cron.sh' --stdout ./cron.stdout --stderr ./cron.stderr --logfile ./locker.out

By default, each of those files (`./cron.stdout`, `./cron.stderr` and `./locker.out`) will be created
anew (as if output were redirected with `>`), but the `--append-stdout`, `--append-stderr` and `--append-log`
switches will cause them instead to be appended-to (`>>`).

Stdout and stderr can each be discarded, with `--no-stdout` and `--no-stderr` respectively. The respective
handle is still opened in this instance, it's just never read-from.

# Logging

Locker logs very tersley to syslog; the intention is just to provide evidence that the command ran, 
rather than to replace the need to direct output to sensible places. The ident used is 'locker/<name>'
in the case where <name> is a simple word, and simply 'locker' when it isn't. 

# Locking

By default, a lockfile named `name` is created in a 'lockfile directory', which defaults to `/tmp/`
and may be overridden with `--lockfiledir`. The expectation is that every cronjob is simply given
a different name to cause them to use their own lockfiles. 

If the name given begins with a `.` or a `/`, then it is used as a path verbatim and the 'lockfile
directory' is ignored.

The locking algorithm is relatively simple and will not work accross NFS. *Any* problem with locking
results in `locker` writing an error message and exiting in an error state, including failure to 
unlock. 

Care is taken to ensure that the lockfile is removed however the program exits; the presence of a
lock file should always indicate a currently-running instance, rather than an earlier failure.
