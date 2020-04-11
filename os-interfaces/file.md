# File

The xv6 file system provides data files, which are uninterpreted byte arrays, and directories, which contain named references to data files and other directories.

Paths that don’t begin with / are evaluated relative to the calling process’s current directory,

A file, called an `inode`, can have multiple names, called `links`. The `link` system call creates another file system name referring to the same `inode` as an existing file.

The file’s `inode` and the disk space holding its content are only **freed** when the **file’s link count is zero and no file descriptors refer to it.**

`cd` changes current process working directory. When shell running this cmd, it does not fork child and change dir. Instead, it change the current working directory of the shell itself. The reason behind is: if forking a child to do `cd`, we are changing the child process’s working directory, not our current shell process.

