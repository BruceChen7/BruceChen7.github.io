title: Simple Http Server Code Learning
date: 2014-9-7
tags: Source Code Reading
---

首先来看看源码的目录

```

    |__ src - All .c source files in this directory <br/>
    |__ include - All header files in this directory
    |_Makefile, config.txt - These files at the same level <br/>
	tokenize.c             -utility function to separate a line into tokens
	fileparser.c           -utility function to parse a file line by line
	http_config.c          -Handles reading of config file, error checking
		               -and populating in-memory config object.
	http_server.c          -Main http server engine
	listener.c             -Connection handler routine
```

<!--more -->
# 源码中的数据结构关系

[![Data structure](/Images/SourceCode/shs.png)](/Images/SourceCode/shs.png)

其中，

* `http_server_config` 表示的是服务器配置的数据结构，其包括 `listen_port`，表示服务器监听的端口，`document_root`表示的是根目录的位置。 `valid_file_type`，表示的是 支持的文件类型,`directory_index_file`,是默认的入口(index)文件的名字。`f_type_cnt`表示的是支持的文件类型的数量
* http_server_data 包含了`http_server_config`这种数据结构，还包含了一个套接字`sockfd`，不过作者在这个程序中并没有用到这个数据结构，可能这是为了以后的方便。
* valid_file_type 包含了两个成员：一个是扩展名称`extendsion`,一个是类型名称`type`
* directory_index_file 就包含了一个数据成员，也就是一个默认的入口文件。

# main 函数

在`http_server.c`中，我们可以找到该程序的入口函数，`main`函数，`main`函数主要干了这么几件事:

[![Main](/Images/SourceCode/SNS_procedures.png)](/Images/SourceCode/SNS_procedures.png)

其中主要的源码为

```cpp
int main(int argc, char *argv[])
{
    struct http_server_config cfg ;
        //initialzied the http server file
    memset(&cfg, 0, sizeof(cfg));
    char filename[MAX_FILENAME];
    if(argc == 2)
    {
                // read the user-defined http server Configuration
            if(strlen(argv[1]) < MAX_FILENAME)
                    strcpy(filename, argv[1]);
    }
    else {
        // use the default http server Configuration
        strcpy(filename, "config.txt");
    }

    // read the server Configuration
    if(file_parser(filename, cfg_reader, &cfg)== -1)
    {
            LOG(stdout, "Configuration Error.Exiting...\n");
            exit(1);

    }
    /*Ignore child death and avoid zombies!*/
    signal(SIGCLD, SIG_IGN);
    LOG(stdout, "Starting primitive HTTP web server implemented for CSC573\n");
    if(connection_handler(&cfg) == -1)
    {
            LOG(stdout, "Error!Exiting..\n");
            exit(1);
    }
    return 0;
}
```


# file_parser 函数
该函数传入的是配置文件名`filename`，和一个函数指针,这个函数是用来判断该服务器配置文件是否配置正确。

```cpp
int file_parser(char *filename, int (*reader)(void *, char *line), void *c)
{

    FILE *fp;
    char line[MAX_LINE];
    assert(filename != NULL);
    assert(c != NULL);
    if((fp = fopen(filename, "r")) == NULL)
    {
            LOG(stdout, "Failed to open config file\n");
            return -1;
    }
    while((fgets(line, MAX_LINE, fp) != NULL))
    {
            if(reader(c, line) == -1)
            {
                    LOG(stdout, "file parser: reader() failed\n");
                    return -1;
            }
    }
    fclose(fp);
    return 0;
}
```

程序非常简单，首先是检查文件名的值，然后打开该配置文件,然后逐行的检查配置文是否书写正确,最主要的事情是调入的函数来完成，也就是`reader`来完成。

考虑作者设计这个函数的原型

* int file_parser(char *filename, int (*reader)(void *, char *line), void *c),这里面最主要的是函数指针的原型和 `void *` 空指针的运用，在 `main` 函数中，作者将`http_server_config`这个服务器配置文件类型，来传给了这个`file_parser`函数，这点比较巧妙。注意这里的经验总结。接下来，就是传入给这个函数的函数指针的研究(cfg_reader)

# cfg_reader(void *c,char *line)

这个函数主要的作用是读入配置文件，然后根据配置文件的内容中的值写入到`http_server_config`类型的c中，其中包括`listen_port`,`document_root	`, `valid_file_type filetypes`, `dire_index`,`f_type_cnt`,具体的做法是:

* 首先这个函数判断传入的 `http_server_conig` 类型 c 与 配置文件的每一行`line` 是否为空,然后就将配置文件以空格为分界符，将里面的关键词如:`Listen`, `DocumentRoot`等 通过 `token` 这个指针来获取，然后将其值写入到c中。这个过程主要是`tokenize`这个函数来完成。下面看一看这个主要的`tokenize`函数。


# tokenize 函数

```cpp
/* This function separates tokens from the given string. Tokens can be of
 * maximum MAX_TOKEN_SIZE . After a call to tokenize, str points to the first
 * character after the current token; the string is consumed by the tokenize
 * routine.
 * RETURNS: length of current token
 *          -1 if error
 */
int tokenize(char **str, char *dest)
{
    int count = 0;

        // test if it's whitespace character
    while(isspace(**str))
            (*str)++;
    while(!isspace(**str) && (**str != '\0'))
    {
            *dest++ = *(*str)++;
            count++;
            if(count >= MAX_TOKEN_SIZE)
            {
                    count = -1;
                    break;
            }
    }
    *dest = '\0';
    return count;
}
```


* 这个函数是处理配置文件中一行的标志。`str` 是读入到内存缓冲区的配置文件中的一行,`dest`是提取出来的标志
* 注意到`tokenize`这个函数的首参数是二级指针，也就是说传入的指针每次改变，而这里传入的实参是`char line[MAX_LINE]`的 `line`这个缓冲区数组，也就是说每次`tokenize`，line的值是改变的.
*  该函数首先判断当前读入的配置文件的这一行`str`所指的字符是否是 `空字符`，如果是，则str所指的`地址值`增加。然后在不是空字符和字符串结尾的情况下，将`str`所指的值复制到`dest`所指的缓冲区中。例如:如果读入的配置文件的一行是`Listen 8080 `，则传到`dest`所指的缓冲区的内容是`Listen`，而传入的指针str 已经将原来指向缓冲区内容`Listen 8080`的首地址，改成指向`8080`前面空字符的地址。
