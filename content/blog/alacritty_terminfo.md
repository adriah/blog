+++
author = "cr4sh0v3rrid3"
title = "Alacritty terminfo"
date = "2024-04-05"
description = "Getting alacritty to work on remote hosts"
tags = [
    "linux",
    "howto"
]
+++


To get alacritty to work with remote hosts:

```
$ infocmp > al.txt
$ scp al.txt <user>@<host>:~/

# On the remote host:
$ tic -x al.txt
```

### What does "tic -x" do?
```
$ man tic
...
       -x   Treat unknown capabilities as user-defined (see user_caps(5)).  That is, if you supply a capability name which tic does not  recognize,  it  will  infer  its  type
            (Boolean,  number  or string) from the syntax and make an extended table entry for that.  User-defined capability strings whose name begins with “k” are treated as
            function keys.
...
```

## One liner
Note: this is stolen from the [archwiki](https://wiki.archlinux.org/title/Alacritty#Terminal_functionality_unavailable_in_remote_shells)
```
$ infocmp | ssh "$user@$host" 'tic -x /dev/stdin'
```
