# Write FS simulator in python

## Implement a file system simulator

```text
// Disk layout:
// [ boot block | sb block | log | inode blocks | free bit map | data blocks ]
```

## Design a crash recovery system in Python

### A system call pattern:

Begin ops Get a buffer Write data to buffer Write actions to log \(Log\_write\) End ops

### Flow

Begin ops waits until enough space in log blocks. Mark current ops as outstanding. Log\_write adds a new block to the log list. End\_op commits if no other outstanding system calls. Call commit. Commit does 4 steps: 1. Write modified blocks from cache to log blocks in disk 2. Write header to disk 3. Install writes from log blocks into home locations 4. Clear log 5. Clear head

### Recover from log:

Triggers when the system restarts. 1. Read log head from disk. 2. Install writes into home locations. 3. Clear log. 4. Clear head.

## Design a simple file system

The possible operations are:

* mkdir\(\) - creates a new directory
* creat\(\) - creates a new \(empty\) file
* open\(\), write\(\), close\(\) - appends a block to a file
* link\(\)   - creates a hard link to a file
* unlink\(\) - unlinks a file \(removing it if linkcnt==0\)

The state of the file system is shown by printing the contents of four different data structures:

```text
inode bitmap - indicates which inodes are allocated
inodes             - table of inodes and their contents
data bitmap   - indicates which data blocks are allocated
data                 - indicates contents of data blocks
```

Initial plan: 1. Read very simple fs impl. 2. Draw a PoC about how it works in the end 3. Write logging system 4. Design fs data structure 5. Implement my version of fs 6. Use mock for testing

### Support transaction

Q: In actual system calls impl, each fs system call has a pattern, used for logging purpose. How do we design our simulator to work the same? A: To make it simple, let’s have batch updates: Begin Mkdir a Open a/test.txt Write a/test.txt ‘A Msg’ Link a/test.txt a/twins.txt Close a/test.txt End

With these sequences, we are making a transaction. All operations will be logged to log system, then perform the disk updates.

