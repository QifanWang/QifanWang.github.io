---
layout: post
title:  "CS:APP Malloc Lab"
categories: Labs
tags: [system programming, memory]
toc: true
--- 
Real generosity toward the future lies in giving all to the present
{: .message }

阅读 csapp 3e 第十、十一与十二章。完成 Proxy 实验，编写 proxy.c 代码，完成一个简易的代理服务器。

代码[见此](https://github.com/QifanWang/learning-csapp/tree/master/handout/proxylab-handout)。

终端 make 之后，通过下面命令调用评分程序，
```
./driver.sh
```

## 多线程实现
每 accept 一个连接请求就分配一个线程(detach)，在do_it函数中完成代理转发的工作。

{% highlight c %}
int main(int argc, char **argv)
{
    int listenfd, *connfdp;
    socklen_t clientlen;
    struct sockaddr_storage clinetaddr;
    pthread_t tid;

    if (argc != 2)
    {
        printf("Usage: %s <port_number>\n", argv[0]);
        exit(0);
    }

    init_cache();

    // argv[1] is listen port
    listenfd = Open_listenfd(argv[1]);

    clientlen = sizeof(struct sockaddr_storage);
    while (1)
    {
        connfdp = Malloc(sizeof(int));

        *connfdp = Accept(listenfd, (SA *)&clinetaddr, &clientlen);
        Pthread_create(&tid, NULL, thread, connfdp);
    }

    destroy_cache();

    return 0;
}

void *thread(void *vargp)
{
    Pthread_detach(pthread_self());

    int connfd = *((int *)vargp);
    Free(vargp);
    // do work
    do_it(connfd);

    Close(connfd);
    return NULL;
}
{% endhighlight %}

“代理”就是把客户端的请求进行一定的修改(修改请求头的相关字段)后，转发给服务器，再从服务器那里接收请求并返回给客户端。

完成这个lab 时，我的 proxy 代码并没有完全转发客户端请求，除了基本的请求头特殊处理以外，请求头部分字段由 proxy 自行填充(舍弃了客户端其他请求头的)。为了获取 response body 的长度，要额外读取 Content-length 字段。

{% highlight c %}
void do_it(int fd)
{

    char buf[MAXLINE], method[MAXLINE], oldUrl[MAXLINE], version[MAXLINE];
    char hostName[MAXLINE], port[MAXLINE], newUrl[MAXLINE];
    char req[MAXLINE], cacheBuf[MAXLINE];
    int fd2server;
    rio_t rio, rio2server;

    /* client -> proxy starts */
    Rio_readinitb(&rio, fd);
    Rio_readlineb(&rio, buf, MAXLINE);

    sscanf(buf, "%s %s %s", method, oldUrl, version);
    if (strcasecmp(method, "GET"))
    {
        proxyerror(fd, method, "501", "Not Implemented",
                   "Proxy does not implement this method");
        return;
    }

    // store original method
    strcpy(req, buf);

    int http_phead_len = strlen(http_phead);
    int content_len = 0;
    char *colon = strstr(oldUrl + http_phead_len, ":");
    char *forwardSlash = strstr(oldUrl + http_phead_len, "/");

    extract_hostname(hostName, oldUrl + http_phead_len);
    extract_portnum(port, colon);
    extract_newUri(newUrl, forwardSlash);

    // browser sends any additional req heads
    Rio_readlineb(&rio, buf, MAXLINE);
    while (strcmp(buf, "\r\n"))
    { // read until \r\n
        // actually ignore
        // Rio_writen(fd2server, buf, strlen(buf));
        Rio_readlineb(&rio, buf, MAXLINE);
    }

    /* client -> proxy ends */

    /* client <- proxy try to use cache*/
    int hashVal = getHash(req);
    if (use_cache(hashVal, req, fd))
    {
        return;
    }
    /* client <- proxy ends */

    /* proxy -> server starts */
    fd2server = Open_clientfd(hostName, port);
    Rio_readinitb(&rio2server, fd2server);

    sprintf(buf, "%s %s %s\r\n", method, newUrl, "HTTP/1.0");
    Rio_writen(fd2server, buf, strlen(buf));

    sprintf(buf, "Host: %s\r\n", hostName);
    Rio_writen(fd2server, buf, strlen(buf));

    sprintf(buf, "%s\r\n", user_agent_hdr);
    Rio_writen(fd2server, buf, strlen(buf));

    sprintf(buf, "%s\r\n", connection_hdr);
    Rio_writen(fd2server, buf, strlen(buf));

    sprintf(buf, "%s\r\n", proxy_conn_hdr);
    Rio_writen(fd2server, buf, strlen(buf));

    sprintf(buf, "\r\n");
    Rio_writen(fd2server, buf, strlen(buf));

    /* proxy -> server ends */

    // client <- proxy <- server Updates Cache
    int posOfRespHead = 0;
    Rio_readlineb(&rio2server, buf, MAXLINE);
    posOfRespHead = write2buf(cacheBuf, posOfRespHead, buf);
    while (strcmp(buf, "\r\n"))
    {
        if (strncmp(buf, contentlen_head, strlen(contentlen_head)) == 0)
        {
            sscanf(buf, "Content-length: %d", &content_len);
        }
        Rio_writen(fd, buf, strlen(buf));
        Rio_readlineb(&rio2server, buf, MAXLINE);
        posOfRespHead = write2buf(cacheBuf, posOfRespHead, buf);
    }
    Rio_writen(fd, buf, strlen(buf)); // \r\n

    // reponse content
    Rio_readnb(&rio2server, buf, content_len);
    Rio_writen(fd, buf, content_len);

    insert_cache(hashVal, req, strlen(req) + 1, cacheBuf, posOfRespHead + 1, buf, content_len);    

    close(fd2server);
}
{% endhighlight %}

## 并发读写缓存

用实现双链表加哈希，实现一个LRU的缓存。针对 Hash Collision 采取的策略是比较被哈希的"请求"字符串，即 Cache_Element 的 req 字段，如果冲突则覆盖(没有用线性探测、平方探测、重哈希或生成链表的方法)。
{% highlight c %}
#define MAX_ELEMENTS 10
#define HEAD 0
#define TAIL 11

struct Cache_Element
{
    u_int16_t prev, next;
    char *req;
    char *respHead;
    char *respBody;
    int respBodySz;
};

struct
{
    struct Cache_Element ele[MAX_ELEMENTS + 2];
    int cached_cnt;
    sem_t mutex;
} Cache;
{% endhighlight %}

每次读写命中缓存则要更新缓存单元在链表中的未知，无命中则添加新缓存。由于用全局变量 Cache 代表，所以必须要用 PV 原语进行临界区读写保护。

{% highlight c %}
// try to use the cached data at hashVal position
// if cache hit, write cached data into fd
int use_cache(int hashVal, char* key, int fd)
{
    int ret = 0;
    P(&Cache.mutex);
    if (Cache.ele[hashVal].req != NULL &&
        strcmp(Cache.ele[hashVal].req, key) == 0)
    {

        // use cache successfully
        ret = 1;

        toBeleast(hashVal);

        // send data to client
        char* cached = Cache.ele[hashVal].respHead;
        char* end = cached;
        while((end = strstr(cached, "\r\n")) != NULL) {
            int wrBytes = end - cached;
            wrBytes += 2;
            Rio_writen(fd, cached, wrBytes);

            cached = end + 2;
        }
        Rio_writen(fd, Cache.ele[hashVal].respBody, Cache.ele[hashVal].respBodySz);
    }
    V(&Cache.mutex);

    return ret;
}

void insert_cache(int hashVal, char* req, int reqLen, char* respHead, int headLen, char* respBody, int bodySz) {
    if(headLen <= 0 || bodySz > MAX_OBJECT_SIZE) {
        return;
    }

    P(&Cache.mutex);

    if(Cache.ele[hashVal].req != NULL || Cache.cached_cnt == MAX_ELEMENTS) { // overwrite cached elements
        free(Cache.ele[hashVal].req);
        Cache.ele[hashVal].req = malloc(reqLen);
        memcpy(Cache.ele[hashVal].req, req, reqLen);
        
        free(Cache.ele[hashVal].respHead);
        Cache.ele[hashVal].respHead = malloc(headLen);
        memcpy(Cache.ele[hashVal].respHead, respHead, headLen);

        free(Cache.ele[hashVal].respBody);
        Cache.ele[hashVal].respBody = malloc(bodySz);
        memcpy(Cache.ele[hashVal].respBody, respBody, bodySz);

        Cache.ele[hashVal].respBodySz = bodySz;

        toBeleast(hashVal);
    } else { // add new elements

        Cache.ele[hashVal].req = malloc(reqLen);
        memcpy(Cache.ele[hashVal].req, req, reqLen);
        
        Cache.ele[hashVal].respHead = malloc(headLen);
        memcpy(Cache.ele[hashVal].respHead, respHead, headLen);

        Cache.ele[hashVal].respBody = malloc(bodySz);
        memcpy(Cache.ele[hashVal].respBody, respBody, bodySz);
        
        Cache.ele[hashVal].respBodySz = bodySz;

        // add to head
        Cache.ele[hashVal].next = Cache.ele[HEAD].next;
        Cache.ele[hashVal].prev = HEAD;
        Cache.ele[HEAD].next = hashVal;
        Cache.ele[Cache.ele[hashVal].next].prev = hashVal;

        ++Cache.cached_cnt;
    }

    V(&Cache.mutex);
}
{% endhighlight %}

需要注意的是，用二进制数据读写函数接口操作字符串数据时，需要注意 \0 所占的空间，这在操作字符串数据时是尤其需要注意的。每次覆盖缓存时，我选择 free 再 malloc ，但其实可以固定分配空间大小，直接在空间上写覆盖即可。

## Conclusions

最后三章相当于把书本前面的内容串起来，进行应用上的总结，即如何完成程序间通信(本地的或远程的)。系统级 I/O 中文件的概念与 Descriptor/OpenFile/V-node table，网络编程中的套接字接口，并发编程三种模型以及同步问题(生产者消费者与读写问题等)，线程安全函数与可重入函数的概念，这些都可以进一步钻研(尤其是并发的同步问题)。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [malloc lab readme](http://csapp.cs.cmu.edu/3e/README-proxylab)
3. [malloc lab writeup](http://csapp.cs.cmu.edu/3e/proxylab.pdf)
4. [My Solution](https://github.com/QifanWang/learning-csapp/tree/master/handout/proxylab-handout)
