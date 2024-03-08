## Boot xv6
没什么好说的

## sleep
用atoi函数转下类型调用sleep函数即可
```cpp
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main (int argc, char *argv[])
{
	if(argc <= 1) {
	fprintf(2, "Usage: sleep...\n");
	exit(1);
	}
	sleep(atoi(argv[1]));
	exit(0);
	// 可以添加一个范围判断，但我懒得写了
}
```

## pingpong
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main (int argc, char *argv[])
{
	char ch;
	int p[2];
	pipe(p); // 创建管道

	int pid = fork(); //创建子进程
	if(pid == 0) { // 如果为子进程则尝试从管道读取字符
		if(read(p[0], &ch, 1)) {
			printf("%d: received ping\n", pid);
			write(p[1], "0", 1); // 读到字符后向管道写入字符
			exit(0);
		}
	} else {
		write(p[1], "1", 1); // 父进程先向管道写入一个字符
		if(read(p[0], &ch, 1)) { // 尝试从管道读取字符
			printf("%d: received pong\n", pid);
			exit(0);
		}
	}
	exit(-1);
}
```

### primes
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void
primes(void) {
    int p[2];
    int num, pid, n;

    if(read(0, &num, sizeof(num)) == 0) {
        return; // Important: To terminate the recursion
        exit(0);
    }
    
    printf("prime %d\n", num);
    
    pipe(p);
    pid = fork();
    
    if(pid == 0) {
        // set the read-port's file description as 0: 
        close(0);

        dup(p[0]);
        close(p[0]);
        close(p[1]);

        primes();
        exit(0);
    } else {
        while(read(0, &n, sizeof(n))) {
            // check if the n can be devided by the first number
            if(n % num != 0) {
                write(p[1], &n, sizeof(n));
            }
        }
        close(p[1]);

        wait(0);
        exit(0);
    }
}

int
main (int argc, char *argv[]) 
{
    int p[2];
    pipe(p);
    int pid = fork();

    if(pid == 0) {
        // set the read-port's file description as 0: 
        close(0);

        dup(p[0]);
        close(p[0]);
        close(p[1]);

        primes();
    } else {
        for(int i = 2; i <= 35; i++) {
            write(p[1], &i, sizeof(i));
        }
        close(p[1]);
        
        wait(0);
        exit(0);
    }
    exit(-1);
}
```

### find

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"

int 
find(char *path, char *filename)
{
	char buf[512], *p, *name;
	int fd;
	struct dirent de;
	struct stat st;
	
	if ((fd = open(path, O_RDONLY)) < 0)
	{
		fprintf(2, "ls: cannot open %s\n", path);
		return -1;
	}
	
	if (fstat(fd, &st) < 0)
	{
		fprintf(2, "ls: cannot stat %s\n", path);
		close(fd);
		return -1;
	}
	
	switch (st.type)
	{
		case T_DEVICE:
		case T_FILE:
		// get the filename of the current file
			name = path;
			for (int i = strlen(path) - 1; i >= 0; i--)
			{
				if (path[i] == '/')
				{
					name = &path[i + 1];
					break;
				}
			}
			if (!strcmp(name, filename))
			{
				printf("%s\n", path);
			}
		break;
		
	case T_DIR:
		if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf)
		{
			printf("ls: path too long\n");
			break;
			return -1;
		}
		strcpy(buf, path);
		p = buf + strlen(buf);
		*p++ = '/';
		while (read(fd, &de, sizeof(de)) == sizeof(de))
		{
			if (de.inum == 0 || !strcmp(de.name, ".") || !strcmp(de.name, ".."))
				continue;
			memmove(p, de.name, DIRSIZ);
			p[DIRSIZ] = 0;
			find(buf, filename);
		}
		break;
	}
	close(fd);
	return 0;
}

  

int main(int argc, char *argv[])
{
	if (argc != 3)
	{
		printf("Invalid arguments.\n");
		exit(-1);
	}
	int stat = find(argv[1], argv[2]);
	exit(stat);
}
```
### xargs
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main (int argc, char *argv[]) 
{
    char *new_args[argc];
    int i;
    
    for(i=1; i < argc; i++) {
        new_args[i - 1] = argv[i];
    }

    new_args[argc - 1] = malloc(512);
    new_args[argc] = 0;

    while(gets(new_args[argc - 1],512)) {
        if(new_args[argc - 1][0] == 0)
            break;
        if(new_args[argc - 1][strlen(new_args[argc - 1]) - 1]=='\n')
            new_args[argc - 1][strlen(new_args[argc - 1]) - 1] = 0;
        if(fork()==0)
            exec(argv[1],new_args);
        else
            wait(0);
    }
    
    exit(0);
}
```