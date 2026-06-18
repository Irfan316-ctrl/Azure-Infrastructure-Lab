# File Permissions Learning

## What I Learned

### File Permissions Basics

Every file and folder has permissions. They control who can read, write, and execute it.

The format looks like: -rwxrwxrwx

The first character is either - (file) or d (directory).

The next 9 characters are in 3 groups of 3:
- First 3: owner permissions
- Next 3: group permissions
- Last 3: others permissions

### The Three Permissions

r = read (can view the file)
w = write (can change the file)
x = execute (can run the file, or access a directory)

### Owner, Group, Others

Owner = the user who created it
Group = a group of users
Others = everyone else on the system

### Understanding Permissions

Example: -rw-r--r--

Breaking it down:
- First character: - means it's a file
- Owner: rw- means read and write
- Group: r-- means read only
- Others: r-- means read only

### The chmod Number System

r (read) = 4
w (write) = 2
x (execute) = 1

You add them together:

7 = 4+2+1 = rwx (full permissions)
6 = 4+2 = rw- (read and write, no execute)
5 = 4+1 = r-x (read and execute, no write)
4 = r-- (read only)
0 = --- (no permissions at all)

### Common chmod Examples

chmod 755 = rwxr-xr-x
Owner gets full access. Group and others can read and execute.

chmod 700 = rwx------
Only owner can access. Group and others get nothing.

chmod 644 = rw-r--r--
Owner can read and write. Group and others can only read.

chmod 600 = rw-------
Only owner can read and write. Everyone else is blocked.

### Directories Are Different

For directories, x doesn't mean "run it."

x on a directory = you can ENTER and ACCESS the directory

r on a directory = you can LIST files inside

w on a directory = you can CREATE and DELETE files inside

So rwx on a folder means you can list, enter, and create/delete files.

### What I Practiced

Created testfile.txt with permissions -rw-r--r--

Used chmod 600 to change it to -rw------- (only I can read/write)

Used chmod 755 to change it to -rwxr-xr-x (everyone can read/execute)

Created myproject folder with permissions drwxr-xr-x

Used chmod 700 to change it to drwx------ (only I can access)

### Why This Matters

File permissions are essential for:
- Security: keeping sensitive files and folders private
- Shared work: allowing some people to read but not modify
- Scripts: making files executable so they can run
- System administration: controlling access to important system files

In a real company, permissions protect production databases, configuration files, and private data.