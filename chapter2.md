
## Namespaces in operation,the namespaces API
> A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the resource. Namespaces are used for a variety of purposes, with the most notable being the implementation of containers, a technique for lightweight virtualization. This is the second part in a series of articles that looks in some detail at namespaces and the namespaces API. The first article in this series provided an overview of namespaces. This article looks at the namespaces API in some detail and shows the API in action in a number of example programs.

> The namespace API consists of three system calls—clone(), unshare(), and setns()—and a number of /proc files. In this article, we'll look at all of these system calls and some of the /proc files. In order to specify a namespace type on which to operate, the three system calls make use of the CLONE_NEW* constants listed in the previous article: CLONE_NEWIPC, CLONE_NEWNS, CLONE_NEWNET, CLONE_NEWPID, CLONE_NEWUSER, and CLONE_NEWUTS.

Creating a child in a new namespace: clone()

> One way of creating a namespace is via the use of clone(), a system call that creates a new process. For our purposes, clone() has the following prototype:

    int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
> Essentially, clone() is a more general version of the traditional UNIX fork() system call whose functionality can be controlled via the flags argument. In all, there are more than twenty different CLONE_* flags that control various aspects of the operation of clone(), including whether the parent and child process share resources such as virtual memory, open file descriptors, and signal dispositions. If one of the CLONE_NEW* bits is specified in the call, then a new namespace of the corresponding type is created, and the new process is made a member of that namespace; multiple CLONE_NEW* bits can be specified in flags.

> Our example program (demo_uts_namespace.c) uses clone() with the CLONE_NEWUTS flag to create a UTS namespace. As we saw last week, UTS namespaces isolate two system identifiers—the hostname and the NIS domain name—that are set using the sethostname() and setdomainname() system calls and returned by the uname() system call. You can find the full source of the program here. Below, we'll focus on just some of the key pieces of the program (and for brevity, we'll omit the error checking code that is present in the full version of the program).

> The example program takes one command-line argument. When run, it creates a child that executes in a new UTS namespace. Inside that namespace, the child changes the hostname to the string given as the program's command-line argument.

> The first significant piece of the main program is the clone() call that creates the child process:

    child_pid = clone(childFunc, 
                      child_stack + STACK_SIZE,   /* Points to start of 
                                                     downwardly growing stack */ 
                      CLONE_NEWUTS | SIGCHLD, argv[1]);

    printf("PID of child created by clone() is %ld\n", (long) child_pid);
> The new child will begin execution in the user-defined function childFunc(); that function will receive the final clone() argument (argv[1]) as its argument. Since CLONE_NEWUTS is specified as part of the flags argument, the child will execute in a newly created UTS namespace.

> The main program then sleeps for a moment. This is a (crude) way of giving the child time to change the hostname in its UTS namespace. The program then uses uname() to retrieve the host name in the parent's UTS namespace, and displays that hostname:

    sleep(1);           /* Give child time to change its hostname */

    uname(&uts);
    printf("uts.nodename in parent: %s\n", uts.nodename);
> Meanwhile, the childFunc() function executed by the child created by clone() first changes the hostname to the value supplied in its argument, and then retrieves and displays the modified hostname:

    sethostname(arg, strlen(arg);
    
    uname(&uts);
    printf("uts.nodename in child:  %s\n", uts.nodename);
> Before terminating, the child sleeps for a while. This has the effect of keeping the child's UTS namespace open, and gives us a chance to conduct some of the experiments that we show later.

> Running the program demonstrates that the parent and child processes have independent UTS namespaces:

    $ su                   # Need privilege to create a UTS namespace
    Password: 
    # uname -n
    antero
    # ./demo_uts_namespaces bizarro
    PID of child created by clone() is 27514
    uts.nodename in child:  bizarro
    uts.nodename in parent: antero
> As with most other namespaces (user namespaces are the exception), creating a UTS namespace requires privilege (specifically, CAP_SYS_ADMIN). This is necessary to avoid scenarios where set-user-ID applications could be fooled into doing the wrong thing because the system has an unexpected hostname.

> Another possibility is that a set-user-ID application might be using the hostname as part of the name of a lock file. If an unprivileged user could run the application in a UTS namespace with an arbitrary hostname, this would open the application to various attacks. Most simply, this would nullify the effect of the lock file, triggering misbehavior in instances of the application that run in different UTS namespaces. Alternatively, a malicious user could run a set-user-ID application in a UTS namespace with a hostname that causes creation of the lock file to overwrite an important file. (Hostname strings can contain arbitrary characters, including slashes.)

The /proc/PID/ns files

> Each process has a /proc/PID/ns directory that contains one file for each type of namespace. Starting in Linux 3.8, each of these files is a special symbolic link that provides a kind of handle for performing certain operations on the associated namespace for the process.

    $ ls -l /proc/$$/ns         # $$ is replaced by shell's PID
    total 0
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 ipc -> ipc:[4026531839]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user -> user:[4026531837]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]
> One use of these symbolic links is to discover whether two processes are in the same namespace. The kernel does some magic to ensure that if two processes are in the same namespace, then the inode numbers reported for the corresponding symbolic links in /proc/PID/ns will be the same. The inode numbers can be obtained using the stat() system call (in the st_ino field of the returned structure).

> However, the kernel also constructs each of the /proc/PID/ns symbolic links so that it points to a name consisting of a string that identifies the namespace type, followed by the inode number. We can examine this name using either the ls -l or the readlink command.

> Let's return to the shell session above where we ran the demo_uts_namespaces program. Looking at the /proc/PID/ns symbolic links for the parent and child process provides an alternative method of checking whether the two processes are in the same or different UTS namespaces:

    ^Z                                # Stop parent and child
    [1]+  Stopped          ./demo_uts_namespaces bizarro
    # jobs -l                         # Show PID of parent process
    [1]+ 27513 Stopped         ./demo_uts_namespaces bizarro
    # readlink /proc/27513/ns/uts     # Show parent UTS namespace
    uts:[4026531838]
    # readlink /proc/27514/ns/uts     # Show child UTS namespace
    uts:[4026532338]
> As can be seen, the content of the /proc/PID/ns/uts symbolic links differs, indicating that the two processes are in different UTS namespaces.

> The /proc/PID/ns symbolic links also serve other purposes. If we open one of these files, then the namespace will continue to exist as long as the file descriptor remains open, even if all processes in the namespace terminate. The same effect can also be obtained by bind mounting one of the symbolic links to another location in the file system:

    # touch ~/uts                            # Create mount point
    # mount --bind /proc/27514/ns/uts ~/uts
> Before Linux 3.8, the files in /proc/PID/ns were hard links rather than special symbolic links of the form described above. In addition, only the ipc, net, and uts files were present.

Joining an existing namespace: setns()

> Keeping a namespace open when it contains no processes is of course only useful if we intend to later add processes to it. That is the task of the setns() system call, which allows the calling process to join an existing namespace:

    int setns(int fd, int nstype);
> More precisely, setns() disassociates the calling process from one instance of a particular namespace type and reassociates the process with another instance of the same namespace type.

> The fd argument specifies the namespace to join; it is a file descriptor that refers to one of the symbolic links in a /proc/PID/ns directory. That file descriptor can be obtained either by opening one of those symbolic links directly or by opening a file that was bind mounted to one of the links.

> The nstype argument allows the caller to check the type of namespace that fd refers to. If this argument is specified as zero, no check is performed. This can be useful if the caller already knows the namespace type, or does not care about the type. The example program that we discuss in a moment (ns_exec.c) falls into the latter category: it is designed to work with any namespace type. Specifying nstype instead as one of the CLONE_NEW* constants causes the kernel to verify that fd is a file descriptor for the corresponding namespace type. This can be useful if, for example, the caller was passed the file descriptor via a UNIX domain socket and needs to verify what type of namespace it refers to.

> Using setns() and execve() (or one of the other exec() functions) allows us to construct a simple but useful tool: a program that joins a specified namespace and then executes a command in that namespace.

> Our program (ns_exec.c, whose full source can be found here) takes two or more command-line arguments. The first argument is the pathname of a /proc/PID/ns/* symbolic link (or a file that is bind mounted to one of those symbolic links). The remaining arguments are the name of a program to be executed inside the namespace that corresponds to that symbolic link and optional command-line arguments to be given to that program. The key steps in the program are the following:

    fd = open(argv[1], O_RDONLY);   /* Get descriptor for namespace */

    setns(fd, 0);                   /* Join that namespace */

    execvp(argv[2], &argv[2]);      /* Execute a command in namespace */
> An interesting program to execute inside a namespace is, of course, a shell. We can use the bind mount for the UTS namespace that we created earlier in conjunction with the ns_exec program to execute a shell in the new UTS namespace created by our invocation of demo_uts_namespaces:

    # ./ns_exec ~/uts /bin/bash     # ~/uts is bound to /proc/27514/ns/uts
    My PID is: 28788
> We can then verify that the shell is in the same UTS namespace as the child process created by demo_uts_namespaces, both by inspecting the hostname and by comparing the inode numbers of the /proc/PID/ns/uts files:

    # hostname
    bizarro
    # readlink /proc/27514/ns/uts
    uts:[4026532338]
    # readlink /proc/$$/ns/uts      # $$ is replaced by shell's PID
    uts:[4026532338]
> In earlier kernel versions, it was not possible to use setns() to join mount, PID, and user namespaces, but, starting with Linux 3.8, setns() now supports joining all namespace types.

Leaving a namespace: unshare()

> The final system call in the namespaces API is unshare():

    int unshare(int flags);
> The unshare() system call provides functionality similar to clone(), but operates on the calling process: it creates the new namespaces specified by the CLONE_NEW* bits in its flags argument and makes the caller a member of the namespaces. (As with clone(), unshare() provides functionality beyond working with namespaces that we'll ignore here.) The main purpose of unshare() is to isolate namespace (and other) side effects without having to create a new process or thread (as is done by clone()).

> Leaving aside the other effects of the clone() system call, a call of the form:

    clone(..., CLONE_NEWXXX, ....);
> is roughly equivalent, in namespace terms, to the sequence:

    if (fork() == 0)
        unshare(CLONE_NEWXXX);      /* Executed in the child process */
> One use of the unshare() system call is in the implementation of the unshare command, which allows the user to execute a command in a separate namespace from the shell. The general form of this command is:

>     unshare [options] program [arguments]
> The options are command-line flags that specify the namespaces to unshare before executing program with the specified arguments.

> The key steps in the implementation of the unshare command are straightforward:

     /* Code to initialize 'flags' according to command-line options
        omitted */

     unshare(flags);

     /* Now execute 'program' with 'arguments'; 'optind' is the index
        of the next command-line argument after options */

     execvp(argv[optind], &argv[optind]);
> A simple implementation of the unshare command (unshare.c) can be found here.

> In the following shell session, we use our unshare.c program to execute a shell in a separate mount namespace. As we noted in last week's article, mount namespaces isolate the set of filesystem mount points seen by a group of processes, allowing processes in different mount namespaces to have different views of the filesystem hierarchy.

    # echo $$                             # Show PID of shell
    8490
    # cat /proc/8490/mounts | grep mq     # Show one of the mounts in namespace
    mqueue /dev/mqueue mqueue rw,seclabel,relatime 0 0
    # readlink /proc/8490/ns/mnt          # Show mount namespace ID 
    mnt:[4026531840]
    # ./unshare -m /bin/bash              # Start new shell in separate mount namespace
    # readlink /proc/$$/ns/mnt            # Show mount namespace ID 
    mnt:[4026532325]
> Comparing the output of the two readlink commands shows that the two shells are in separate mount namespaces. Altering the set of mount points in one of the namespaces and checking whether that change is visible in the other namespace provides another way of demonstrating that the two programs are in separate namespaces:

    # umount /dev/mqueue                  # Remove a mount point in this shell
    # cat /proc/$$/mounts | grep mq       # Verify that mount point is gone
    # cat /proc/8490/mounts | grep mq     # Is it still present in the other namespace?
    mqueue /dev/mqueue mqueue rw,seclabel,relatime 0 0
> As can be seen from the output of the last two commands, the /dev/mqueue mount point has disappeared in one mount namespace, but continues to exist in the other.

Concluding remarks

> In this article we've looked at the fundamental pieces of the namespace API and how they are employed together. In the follow-on articles, we'll look in more depth at some other namespaces, in particular, the PID and user namespaces; user namespaces open up a range of new possibilities for applications to use kernel interfaces that were formerly restricted to privileged applications.


