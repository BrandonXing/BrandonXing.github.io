---
layout: post
title: UNIX网络编程 epoll
date: 2023-03-30 14:18 +0800
last_modified_at: 2023-03-30 14:18 +0800
tags: [UNIX, 计算机网络]
categories: [UNIX, 学习笔记]
toc:  true
---

## epoll

epoll是Linux下多路复用IO接口，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。
它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合；在获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。

### epoll修改单进程fd为1024的限制

修改配置文件

{% highlight js %}
sudo vi /etc/security/limits.conf
	在文件尾部写入以下配置,soft软限制，hard硬限制。
	*		soft	nofile	65536
	* 		hard	nofile	100000
{% endhighlight %}

通过 ulimit -a 可以查看软限制参数。

### epoll常用API

epoll实际上将所有监听的文件描述符放在一棵红黑树上，监听事件只返回触发事件的文件描述符来进行处理，头文件为：
{% highlight js %}
#include <sys/epoll.h>
{% endhighlight js %}

####　int epoll_create(int size)

返回红黑树根节点句柄，size是监听的文件描述符数量，可以扩容。

#### int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)

控制某个epoll监控的文件描述符上的事件：注册，修改，删除
{% highlight js %}
epfd:	为epoll_create返回的句柄。
op:		表示为执行的动作，用3个宏表述
		EPOLL_CTL_ADD(注册新的fd到epfd，即加到红黑树上)，
		EPOLL_CTL_MOD(修改已经注册的fd的监听事件)
		EPOLL_CTL_DEL(从epfd删除一个fd，即从红黑树上摘取下来)
fd：	即为要控制的文件描述符
struct epoll_event{
	__uint32_t  events; /*表示为监听事件的类型*/
	epoll_data_t data;	/使用者数据变量，是一个union/
}；
1. 关于events常用的有以下几种宏定义：
EPOLLIN：表示监听对应描述符的读事件
EPOLLOUT：表示监听对应描述符的写事件
EPOLLERR：表示监听对应描述符的错误事件
EPOLLET：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)而言的，在高效并发中常常会用到。
/*以上四个是最常用的监听事件类型，后面几个了解下*/
EPOLLPRI：表示对于的文件描述符有紧急的数据可读（表示有带外来数据到来）
EPOLLHUP：表示对应的文件描述符被挂断
EPOLLONESHOT：只监听一次事件，当监听完成这次事件之后，如果还要继续监听这个socket话，需要重新吧这个socket放入树中
2. 关于epoll_data_t data重点了解下前两个;
		typedef union epoll_data {
			void *ptr;				//用于epoll反应堆，之后再详细描述
			int fd;					//对应的描述符
			uint32_t u32;
			uint64_t u64;
		} epoll_data_t;
{% endhighlight js %}

#### int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)

等待被监控的文件描述符上有事件产生

{% highlight js %}
	epfd:	为epoll_create返回的句柄。
	events：用来存内核得到的事件的数组，由用户自己定义，规定大小
	maxevents：告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size
	timeout：	是超时时间
			-1：	阻塞
			 0：	立即返回，非阻塞
		   >0：	指定毫秒
	返回值：	成功返回有多少文件描述符就绪，时间到时返回0，出错返回-1

{% endhighlight js %}

### epoll事件模型

EPOLL事件有两种模型：
Edge Triggered (ET) 边缘触发只有数据到来才触发，不管缓存区中是否还有数据。
Level Triggered (LT) 水平触发只要缓存区有数据都会触发。

#### ET模式

epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。最好以下面的方式调用ET模式的epoll接口，在后面会介绍避免可能的缺陷。

基于非阻塞文件句柄
只有当read或者write返回EAGAIN(非阻塞读，暂时无数据)时才需要挂起、等待。但这并不是说每次read时都需要循环读，直到读到产生一个EAGAIN才认为此次事件处理完成，当read返回的读到的数据长度小于请求的数据长度时，就可以确定此时缓冲中已没有数据了（舍去剩余的数据），也就可以认为此事读事件已处理完成。
ps：所谓的EAGAIN，就是当read读取到足够的字符或者文件末尾时产生的信号。write也是写入足够的字符才能产生的信号。
ET(edge-triggered)：ET是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知。请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once).

修改一个文件描述符为非阻塞的方式
{% highlight js %} 
	int flag;
	int connfd;			//用于文件描述符
	...
    flag = fcntl(connfd, F_GETFL);          			   /* 修改connfd为非阻塞读 */
    flag |= O_NONBLOCK;
    fcntl(connfd, F_SETFL, flag);
    或者
    fcntl(connfd, F_SETFL, O_NONBLOCK);     /*将connfd设为非阻塞,这种方式会将原先的flag属性覆盖*/
{% endhighlight js %} 

#### LT模式

LT模式即Level Triggered工作模式。
LT(level triggered)：LT是缺省的工作方式，并且同时支持block和no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表。

#### 通过libevent中的epoll反应堆模型看epoll的ET模式

{% highlight js %} 
/*
 *epoll基于非阻塞I/O事件驱动
 */
#include <stdio.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>

#define MAX_EVENTS  1024                                    //监听上限数
#define BUFLEN 4096
#define SERV_PORT   8080

void recvdata(int fd, int events, void *arg);
void senddata(int fd, int events, void *arg);

/* 描述就绪文件描述符相关信息 */

struct myevent_s {
    int fd;                                                 //要监听的文件描述符
    int events;                                             //对应的监听事件
    void *arg;                                              //泛型参数
    void (*call_back)(int fd, int events, void *arg);       //回调函数
    int status;                                             //是否在监听:1->在红黑树上(监听), 0->不在(不监听)
    char buf[BUFLEN];
    int len;
    long last_active;                                       //记录每次加入红黑树 g_efd 的时间值
};

int g_efd;                                                  //全局变量, 保存epoll_create返回的文件描述符
struct myevent_s g_events[MAX_EVENTS+1];                    //自定义结构体类型数组. +1-->listen fd


/*将结构体 myevent_s 成员变量 初始化*/

void eventset(struct myevent_s *ev, int fd, void (*call_back)(int, int, void *), void *arg)
{
    ev->fd = fd;
    ev->call_back = call_back;
    ev->events = 0;
    ev->arg = arg;
    ev->status = 0;
    //memset(ev->buf, 0, sizeof(ev->buf));
    //ev->len = 0;
    ev->last_active = time(NULL);                       //调用eventset函数的时间

    return;
}

/* 向 epoll监听的红黑树 添加一个 文件描述符 */

void eventadd(int efd, int events, struct myevent_s *ev)
{
    struct epoll_event epv = {0, {0}};
    int op;
    epv.data.ptr = ev;
    epv.events = ev->events = events;       //EPOLLIN 或 EPOLLOUT

    if (ev->status == 1) {                                          //已经在红黑树 g_efd 里
        op = EPOLL_CTL_MOD;                                         //修改其属性
    } else {                                //不在红黑树里
        op = EPOLL_CTL_ADD;                 //将其加入红黑树 g_efd, 并将status置1
        ev->status = 1;
    }
    
    if (epoll_ctl(efd, op, ev->fd, &epv) < 0)                       //实际添加/修改
        printf("event add failed [fd=%d], events[%d]\n", ev->fd, events);
    else
        printf("event add OK [fd=%d], op=%d, events[%0X]\n", ev->fd, op, events);
    
    return ;
}

/* 从epoll 监听的 红黑树中删除一个 文件描述符*/

void eventdel(int efd, struct myevent_s *ev)
{
    struct epoll_event epv = {0, {0}};

    if (ev->status != 1)                                        //不在红黑树上
        return ;
    
    epv.data.ptr = ev;
    ev->status = 0;                                             //修改状态
    epoll_ctl(efd, EPOLL_CTL_DEL, ev->fd, &epv);                //从红黑树 efd 上将 ev->fd 摘除
    
    return ;
}

/*  当有文件描述符就绪, epoll返回, 调用该函数 与客户端建立链接 */

void acceptconn(int lfd, int events, void *arg)
{
    struct sockaddr_in cin;
    socklen_t len = sizeof(cin);
    int cfd, i;

    if ((cfd = accept(lfd, (struct sockaddr *)&cin, &len)) == -1) {
        if (errno != EAGAIN && errno != EINTR) {
            /* 暂时不做出错处理 */
        }
        printf("%s: accept, %s\n", __func__, strerror(errno));
        return ;
    }
    
    do {
        for (i = 0; i < MAX_EVENTS; i++)                                //从全局数组g_events中找一个空闲元素
            if (g_events[i].status == 0)                                //类似于select中找值为-1的元素
                break;                                                  //跳出 for
    
        if (i == MAX_EVENTS) {
            printf("%s: max connect limit[%d]\n", __func__, MAX_EVENTS);
            break;                                                      //跳出do while(0) 不执行后续代码
        }
    
        int flag = 0;
        if ((flag = fcntl(cfd, F_SETFL, O_NONBLOCK)) < 0) {             //将cfd也设置为非阻塞
            printf("%s: fcntl nonblocking failed, %s\n", __func__, strerror(errno));
            break;
        }
    
        /* 给cfd设置一个 myevent_s 结构体, 回调函数 设置为 recvdata */
    
        eventset(&g_events[i], cfd, recvdata, &g_events[i]);   
        eventadd(g_efd, EPOLLIN, &g_events[i]);                         //将cfd添加到红黑树g_efd中,监听读事件
    
    } while(0);
    
    printf("new connect [%s:%d][time:%ld], pos[%d]\n", 
            inet_ntoa(cin.sin_addr), ntohs(cin.sin_port), g_events[i].last_active, i);
    return ;
}

void recvdata(int fd, int events, void *arg)
{
    struct myevent_s *ev = (struct myevent_s *)arg;
    int len;

    len = recv(fd, ev->buf, sizeof(ev->buf), 0);            //读文件描述符, 数据存入myevent_s成员buf中
    
    eventdel(g_efd, ev);        //将该节点从红黑树上摘除
    
    if (len > 0) {
    
        ev->len = len;
        ev->buf[len] = '\0';                                //手动添加字符串结束标记
        printf("C[%d]:%s\n", fd, ev->buf);
    
        eventset(ev, fd, senddata, ev);                     //设置该 fd 对应的回调函数为 senddata
        eventadd(g_efd, EPOLLOUT, ev);                      //将fd加入红黑树g_efd中,监听其写事件
    
    } else if (len == 0) {
        close(ev->fd);
        /* ev-g_events 地址相减得到偏移元素位置 */
        printf("[fd=%d] pos[%ld], closed\n", fd, ev-g_events);
    } else {
        close(ev->fd);
        printf("recv[fd=%d] error[%d]:%s\n", fd, errno, strerror(errno));
    }
    
    return;
}

void senddata(int fd, int events, void *arg)
{
    struct myevent_s *ev = (struct myevent_s *)arg;
    int len;

    len = send(fd, ev->buf, ev->len, 0);                    //直接将数据 回写给客户端。未作处理
    /*
    printf("fd=%d\tev->buf=%s\ttev->len=%d\n", fd, ev->buf, ev->len);
    printf("send len = %d\n", len);
    */
    
    if (len > 0) {
    
        printf("send[fd=%d], [%d]%s\n", fd, len, ev->buf);
        eventdel(g_efd, ev);                                //从红黑树g_efd中移除
        eventset(ev, fd, recvdata, ev);                     //将该fd的 回调函数改为 recvdata
        eventadd(g_efd, EPOLLIN, ev);                       //从新添加到红黑树上， 设为监听读事件
    
    } else {
        close(ev->fd);                                      //关闭链接
        eventdel(g_efd, ev);                                //从红黑树g_efd中移除
        printf("send[fd=%d] error %s\n", fd, strerror(errno));
    }
    
    return ;
}

/*创建 socket, 初始化lfd */

void initlistensocket(int efd, short port)
{
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(lfd, F_SETFL, O_NONBLOCK);                                            //将socket设为非阻塞

    /* void eventset(struct myevent_s *ev, int fd, void (*call_back)(int, int, void *), void *arg);  */
    eventset(&g_events[MAX_EVENTS], lfd, acceptconn, &g_events[MAX_EVENTS]);
    
    /* void eventadd(int efd, int events, struct myevent_s *ev) */
    eventadd(efd, EPOLLIN, &g_events[MAX_EVENTS]);
    
    struct sockaddr_in sin;
    memset(&sin, 0, sizeof(sin));                                               //bzero(&sin, sizeof(sin))
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = INADDR_ANY;
    sin.sin_port = htons(port);
    
    bind(lfd, (struct sockaddr *)&sin, sizeof(sin));
    
    listen(lfd, 20);
    
    return ;
}

int main(int argc, char *argv[])
{
    unsigned short port = SERV_PORT;

    if (argc == 2)
        port = atoi(argv[1]);                           //使用用户指定端口.如未指定,用默认端口
    
    g_efd = epoll_create(MAX_EVENTS+1);                 //创建红黑树,返回给全局 g_efd 
    if (g_efd <= 0)
        printf("create efd in %s err %s\n", __func__, strerror(errno));
    
    initlistensocket(g_efd, port);                      //初始化监听socket
    
    struct epoll_event events[MAX_EVENTS+1];            //保存已经满足就绪事件的文件描述符数组 
    printf("server running:port[%d]\n", port);
    
    int checkpos = 0, i;
    while (1) {
        /* 超时验证，每次测试100个链接，不测试listenfd 当客户端60秒内没有和服务器通信，则关闭此客户端链接 */
    
        long now = time(NULL);                          //当前时间
        for (i = 0; i < 100; i++, checkpos++) {         //一次循环检测100个。 使用checkpos控制检测对象
            if (checkpos == MAX_EVENTS)
                checkpos = 0;
            if (g_events[checkpos].status != 1)         //不在红黑树 g_efd 上
                continue;
    
            long duration = now - g_events[checkpos].last_active;       //客户端不活跃的世间
    
            if (duration >= 60) {
                close(g_events[checkpos].fd);                           //关闭与该客户端链接
                printf("[fd=%d] timeout\n", g_events[checkpos].fd);
                eventdel(g_efd, &g_events[checkpos]);                   //将该客户端 从红黑树 g_efd移除
            }
        }
    
        /*监听红黑树g_efd, 将满足的事件的文件描述符加至events数组中, 1秒没有事件满足, 返回 0*/
        int nfd = epoll_wait(g_efd, events, MAX_EVENTS+1, 1000);
        if (nfd < 0) {
            printf("epoll_wait error, exit\n");
            break;
        }
    
        for (i = 0; i < nfd; i++) {
            /*使用自定义结构体myevent_s类型指针, 接收 联合体data的void *ptr成员*/
            struct myevent_s *ev = (struct myevent_s *)events[i].data.ptr;  
    
            if ((events[i].events & EPOLLIN) && (ev->events & EPOLLIN)) {           //读就绪事件
                ev->call_back(ev->fd, events[i].events, ev->arg);
            }
            
            if ((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT)) {         //写就绪事件
                ev->call_back(ev->fd, events[i].events, ev->arg);
            }
        }
    }
    
    /* 退出前释放所有资源 */
    return 0;
}
{% endhighlight js %} 

### 总结

对比于原来的：
socket、bind、listen – epoll_create 创建监听红黑树 – 返回 epfd – epoll_ctl() 向树上添加一个监听fd
while(1){-- epoll_wait监听 – 对应监听fd有事件产生 – 返回监听满足数组。-- 判断返回数组元素 – lfd满足 – accept - cfd 满足 – read() — 完成对应任务 --write回去}
对于反应堆模型：
socket、bind、listen – epoll_create 创建监听红黑树 – 返回 epfd – epoll_ctl() 向树上添加一个监听fd
while(1){-- epoll_wait监听 – 对应监听fd有事件产生 – 返回监听满足数组。-- 判断返回数组元素 – lfd满足 – accept - cfd 满足 – read() — 完成对应任务 – cfd从监听红黑树上摘下–epoll_ctl(监听)cfd的写事件 – EPOLLOUT – 回调函数 – epoll_ctr() --EPOLL_CTL_ADD重新放大红黑树上监听写事件 – 等待epoll_wait返回 – 说明cfd可写 – write回去 – cfd从监听红黑树上摘下–epoll_ctl(监听)cfd的写事件 – EPOLLIN – epoll_ctr() – EPOLL_CTL_ADD重新放到红黑树上监听写事件}

对于写事件在滑动窗口满以及半关闭状态也要调查
对于读的触发条件可能很好理解
EPOLLIN触发条件：
1. 从不可读状态变为可读状态。
2. 内核接收到新发来的数据。
而对于写来说

EPOLLOUT触发条件：
1. 从不可写状态变为可写状态。
2. 实践发现，只要同时注册了EPOLLIN和EPOLLOUT事件，当对端发数据来的时候，如果此时是可写状态，epoll会同时触发EPOLLIN和EPOLLOUT事件。(可以说这种情况写事件触发多亏了读事件触发，如果没有同时注册读写事件，那么即使当前是可写的也不会被epoll_wait通知，因为epoll此时没有关注写事件）
3. 接受连接后，只要注册了EPOLLOUT事件，那么就会马上触发EPOLLOUT事件。
可读意味着缓存区有数据可读出来。
可写意味着缓存区有空间能让你写进去。
