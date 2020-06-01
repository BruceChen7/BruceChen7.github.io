---
title: Euid Ruid And Suid的概念
layout: post
tags: Linux
date: 2014-11-14
toc: true
post_list: "date"
home_btn: true
toc_level: 4
footer: true
---


##  数值范围

在不同的系统中，UID的值的范围也有所不同，但一般来说UID都是由一个15位的整数表示，其范围在0～32767之内，且有如下限制

* 超级用户的UID总为0
* 数值于1～100内的UID约定预留给系统使用，有些手册则推荐在此基础上再预留101～499（如RHEL）甚至是101～999（如Debian[1]）的UID以作备用；而相对应的，在Linux中用useradd命令创建第一个用户时,默认为之分配的UID则为1000


## 分类

**有效用户ID (effective-user-id)**


有效用户ID（Effective UID，即EUID）与有效用户组ID（Effective Group ID，即EGID）在`创建`与`访问文件`的时候发挥作用；具体来说，创建文件时，系统内核将根据`创建文件`的进程的`EUID`与`EGID`设定文件的所有者/组属性， 而在访问文件时，内核亦根据访问进程的`EUID`与`EGID`决定其能否访问文件。

> The effective user id is the user ID that the process is currently wielding. Initially,this ID is equal to the real user ID,because when
a process forks ,the effective user id  of the parent is inherited by the child.Furthermore,when the process issues an `exec` call,the effective user is usually unchanged.

> But,it is during the exec call that the key difference between real user id and the effective id emerges: by executing a setuid(suid) binary, the process can change its effective user id. To be exact,the effective user ID is set to the user ID of the owner of the progame file For instance,because the /usr/bin/passwd file  is a setuid file,and root is its owner ,when a normal users shell spawns a process  to exec  this file, the process takes on the effective user id of root regardless of who the executing user is


linux通常都不建议用户使用root权限去进行一般的处理，但是普通用户在做很多很多services相关的操作的时候，可能需要一个特殊的权限。 为了满足这样的要求，许多services相关的executable有一个标志，这就是set-user-id bit。当这个set-user-id bit=ON的时候，这个executable被用exec启动之后的进程的effective user id就是这个executable的`owner id`， 而并非parent process real user id。如果`set-user-id bit=OFF`的时候，这个被exec起来的进程的effective user id应该是等于进程的user id的。一个文件如果没有SUID或SGID位，则euid=uid egid=gid，分别是运行这个程序的用户的uid和gid。如果普通文件myfile是属于foo用户的，是可执行的，现在没设SUID位。例如kevin用户的uid和gid分别为204和202，foo用户的uid和gid为200,201，kevin运行myfile程序形成的进程的euid=uid=204，egid=gid=202，内核根据这些值来判断进程对资源访问的限制，其实就是kevin用户对资源访问的权限，和foo没关系

【真实用户ID (real-user-id)】

真实用户ID（Real UID,即RUID）与真实用户组ID（Real GID，即RGID）用于`辨识`进程的`真正所有者`，且会影响到进程发送信号的权限。没有超级用户权限的进程仅在其RUID与目标进程的RUID相匹配时才能向目标进程发送信号， 例如在父子进程间，子进程从父进程处继承了认证信息，使得父子进程间可以互相发送信号。

> The real user id is the uid of the the user who originally ran the process. It is set to  the real user ID of the process parent, and does not change during an exec call.Normally,the login process sets the reall user ID of the user login shell to that of the user,and all of the user's processes continue carry this user ID

【暂存用户ID (saved-user-id)】

暂存用户ID（Saved UID，即SUID）用于`以提升权限运行的进程暂时需要做一些不需特权的操作时使用`，这种情况下进程会暂时将自己的`有效用户ID`从特权用户（常为root）对应的UID变为某个`非特权用户`对应的UID，而后将`原有的特权用户UID`复制为`SUID暂存`；之后当进程完成不需特权的操作后，进程使用SUID的值重置EUID以重新获得特权。在这里需要说明的是，无特权进程的EUID值只能设为与`RUID`、`SUID`与`EUID`（也即不改变）之一相同的值。

> `the saved user id` is the process' originally effective user id. when a process forks, the child inherits the saved user. upon an exec call,however,the kernel sets the saved user id to the effective user id,thereby making a record of the effective user id at the time of the exec


##  说明
首先需要明确一点,这几个概念都是和进程相关的.` real user ID`表示的是实际上进程的执行者是谁,`effective user ID`主要用于校验该进程在执行时所获得的文件访问权限,也就是说当进程访问文件时检查权限时实际上检查的该进程的"effective user ID",saved set-user-ID 仅在`effective user ID`发生改变时保存。

## 杂项
* 在遵循POSIX的环境中，id这一命令可以给出当前用户的用户名、所属组及对应的UID、GID的值
* setuid file

Unix下的可执行文件可以设定sticky位，比如用chmod u+s some_exec，此时这个some_exec是一个`SetUID`程序，无论你的uid是什么，当你运行这个程序时，你的euid会变成这个some_exec的属主的uid，一般把它叫suid，此时你的这个进程的权限就变成了这个属主的权限，但uid依然保持不变


