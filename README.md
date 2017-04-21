# RTSP Server Build
****
## Prerequisite-TCP/UDP

#### 1. 网络模型

| ISO网络七层模型        | Linux网络模型           |
| :------------------: |:----------------------:|
| 应用层                |                        |
| 表示层                |应用层                   |
| 会话层                |                        |
| 传输层                |传输层                   |
| 网络层                |网络层                   |
| 数据链路层             |网络接口层               |
| 物理层                |                        |

#### 2. 网络模型各层中的协议

各层协议以Linux网络模型为例

| Linux网络模型         | 协议                    |
| :------------------: |:----------------------:|
| 应用层                | FTP, HTTP, DNS, RTSP等 |
| 传输层                |TCP, UDP                |
| 网络层                |IP, ICMP, ARP等         |
| 网络接口层             |                        |

其中TCP/UDP有时我们将之视为基本协议，很多众所周知的应用层协议都是基于此的
#### 3. socket

Linux的IPC通信方式有：
* 无名管道(pipe)
* 有名管道(FIFO)
* 信号(signal)
* 消息队列(massage)，
* 共享内存(share memory)
* 信号量(semaphore)
* 套接字(socket)

七种，其中socket作为进程间通信的一员主要用于网络上的进程通信
**你可以将它看作一个接口，一个标准，用于统一不同网络协议的具体操作**
#### 4. 网络字节序

不同的CPU有着不同的存储模式：大端模式和小端模式

*所谓的大端模式(Big-endian)，是指数据的高字节，保存在内存的低地址中，而数据的低字节，保存在内存的高地址中
所谓的小端模式(Little-endian)，是指数据的高字节保存在内存的高地址中，而数据的低字节保存在内存的低地址中*

而网络数据在传输时，如何适应不同的CPU将成为一个新的问题
TCP/IP和UDP/IP都规定其地址(address，4字节)和端口号(port，2字节)在通信发起时，都必须转换为网络字节序，**通信数据应该也要注意⚠️**
主机字节序和网络字节序转换接口：

    uint32_t ntohl(uint32_t netlong);
    uint16_t ntohs(uint16_t netshort);
    uint32_t htonl(uint32_t hostlong);
    uint16_t htons(uint16_t hostshort);
字符串和网络字节序转换接口：

    in_addr_t inet_addr(const char *cp);
    char * inet_ntoa(struct in_addr in);
#### 5. 地址和端口号结构

在通信前，地址和端口号必须转为网络字节序，那么必须了解它们存在于什么结构中
在Linux环境下，结构体`struct sockaddr`在/usr/include/linux/socket.h中定义

    typedef unsigned short sa_family_t;
    struct sockaddr {
        sa_family_t     sa_family;    /* address family, AF_xxx       */
        char            sa_data[14];    /* 14 bytes of protocol address */
    }
结构体`struct sockaddr_in`在/usr/include/netinet/in.h中定义

    /* Type to represent a port. */
    typedef uint16_t in_port_t; 
    /* Structure describing an Internet socket address. */
    struct sockaddr_in
    {
        __SOCKADDR_COMMON (sin_);
        in_port_t sin_port;                     /* Port number. */
        struct in_addr sin_addr;            /* Internet address. */

        /* Pad to size of `struct sockaddr'. */
        unsigned char sin_zero[sizeof (struct sockaddr) -
                           __SOCKADDR_COMMON_SIZE -
                           sizeof (in_port_t) -
                           sizeof (struct in_addr)];     
    };
这两个结构便是网络通信中的地址结构，其中结构体`struct sockaddr`是一个通用地址结构，所有用到地址的函数参数均为`struct sockaddr`类型的指针，如后面提到的绑定(`bind`)函数，而结构体`struct sockaddr_in`是一个用户具体操作的结构，其中`in_port_t sin_port`为端口号的定义，`struct in_addr sin_addr`为网络地址的定义

    struct in_addr {
            unsigned long s_addr;
    };
也印证了之前提到的地址为4字节，端口号为2字节的说明
#### 6. TCP编程模型
以一个TCP聊天的模型来说明
TCP服务端：

```c++
/**
 * Created by callon on 17-4-12.
 */

#include "sys/types.h"
#include "sys/socket.h"
#include "netinet/in.h"
#include "arpa/inet.h"
#include "stdio.h"
#include "unistd.h"
#include "strings.h"
#include "string.h"
#include "stdlib.h"
#include "errno.h"
#include "signal.h"
#include "sys/wait.h"
#include "sys/time.h"
#include "iostream"


void sig_child(int signo);
int main(int argc, char **argv)
{
	int listenfd,connectfd;
	//struct sockaddr is universal socket address
	//struct sockaddr_in is internet socket address
	//has same length
	//16 chars
	struct sockaddr_in server_addr,client_addr;
	pid_t childpid;
	char line[512],gline[512];
	ssize_t n;  
	int v=1;
	int on=1;
	//1.create socket
	listenfd = socket(AF_INET, SOCK_STREAM, 0);	
	if(listenfd == -1)
	{
		printf("create socket error: %s\n",(char*)strerror(errno));
		return -1;	
	}	

	memset(&server_addr,0,sizeof(server_addr));
	server_addr.sin_family = AF_INET;
	server_addr.sin_port = htons(1234);
	//htonl means host to net(32 bits)
	//INADDR_ANY is host address
	//you can use server_addr.sin_addr.s_addr = inet_addr("192.168.1.10");
	//function inet_addr returns net address
	server_addr.sin_addr.s_addr = htonl(INADDR_ANY); 
	
	setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, (char *)&v, sizeof(int));

	//2.bind address
	int bindc = bind(listenfd,(struct sockaddr *)&server_addr,sizeof(server_addr));
	if(bindc == -1)
	{
		printf("bind error: %s\n",strerror(errno));
		return -1;
	}	

	//3.listen socket
	//queue max size is 5
	listen(listenfd, 5);
	
	//avoid zombie process
	signal(SIGCHLD,sig_child);
	
	for(;;)
	{
		socklen_t len = sizeof(client_addr);
		//4.wait for connection
		//It  extracts  the  first   connection request  on  the queue of pending connections for the listening socket
		//attention:int accept(int, sockaddr*, socklen_t*)
		connectfd = accept(listenfd,(struct sockaddr *)&client_addr,&len);
		if(connectfd == -1){
			printf("accept client failed: %s\n",strerror(errno));
			return -1;		
		}
		//if child
		if((childpid = fork()) == 0)
		{
			//5.close
			close(listenfd);
			//attention:inet_ntoa(struct in_addr)
			printf("client from %s\n",inet_ntoa(client_addr.sin_addr));
			
			for(;;)
			{
				FILE *fp = stdin;
				fd_set rset;
				int maxfd;
				FD_ZERO(&rset);			
				FD_SET(fileno(fp),&rset);
				FD_SET(connectfd,&rset);
				maxfd = std::max(fileno(fp),connectfd);			
				select(maxfd+1,&rset,NULL,NULL,NULL);
				if(FD_ISSET(connectfd,&rset))
				{
					if((n = recv(connectfd,line,512,0)) > 0)
					{
						line[n] = '\0';
						printf("Client: %s",line);
						//reply to client						
						//char msgBack[512];
						//snprintf(msgBack,sizeof(msgBack),"recv: %s",line);
						//send(connectfd,msgBack,strlen(msgBack),0);
						memset(&line,0,sizeof(line));
					}
					else{
						printf("recv error: %s\n",strerror(errno));
						return -1;
					}				
				}
				if(FD_ISSET(fileno(fp),&rset))
				{
					if(fgets(gline,sizeof(gline),fp)==NULL)
					{
						printf("fgets error: %s\n",strerror(errno));
						return -1;
					}
					send(connectfd,gline,strlen(gline),0);
					memset(gline,0,512);
				}
			}	
			
			exit(0);
		}
		else if(childpid<0){
			printf("fork failed: %s\n",strerror(errno));
			return -1;		
		}	
		//5.close	
		close(connectfd);
	}
	return 0;
}

void sig_child(int signo)
{
	pid_t pid;
	int stat;

	while((pid = waitpid(-1,&stat,WNOHANG)) > 0);
	
	printf("child %d terminated.\n",pid);
	return;	
}
```
TCP客户端：

```c++
/**
 * Created by callon on 17-4-12.
 */

#include "sys/types.h"
#include "sys/socket.h"
#include "netinet/in.h"
#include "arpa/inet.h"
#include "stdio.h"
#include "unistd.h"
#include "strings.h"
#include "string.h"
#include "stdlib.h"
#include "iostream"
#include "errno.h"

int main(int argc, char **argv)
{
	int connectfd;
	struct sockaddr_in server_addr;
	//1.create socket
	connectfd = socket(AF_INET, SOCK_STREAM, 0);
	ssize_t n;
	if(connectfd == -1)
	{
		printf("create socket error: %s\n",strerror(errno));
		return -1;
	}
	//bzero(&server_addr,sizeof(server_addr));
	memset(&server_addr,0,sizeof(server_addr));
	server_addr.sin_family = AF_INET;
	//server bind port 0, server system will choose a port to bind
	//you should use command 'netstat -tupln' find the port number change as below
	server_addr.sin_port = htons(1234);
	//htonl means host to net(32 bits)
	//INADDR_ANY is host address
	//you can use server_addr.sin_addr.s_addr = inet_addr("192.168.1.10");
	//function inet_addr returns net address
	server_addr.sin_addr.s_addr = inet_addr("192.168.73.153");

	//2.connect to server
	int connectc = connect(connectfd,(struct sockaddr *)&server_addr,sizeof(server_addr));
	if(connectc == -1)
	{
		printf("connect failed: %s\n",strerror(errno));
		return -1;	
	}

	
	char recv1[512],send1[512];
	//strcpy(send1,"hello, tcp server!\n");
	//send(connectfd,send1,strlen(send1),0);
	//max transfer len: 100
	memset(send1,0,512);
	memset(recv1,0,512);
	for(;;)
	{
		FILE *fp = stdin;
		fd_set rset;
		int maxfd;
		FD_ZERO(&rset);
		FD_SET(fileno(fp),&rset);
		FD_SET(connectfd,&rset);
		maxfd = std::max(fileno(fp),connectfd)+1;
		select(maxfd,&rset,NULL,NULL,NULL);

		if(FD_ISSET(connectfd,&rset))
		{
			if((n = recv(connectfd,recv1,sizeof(recv1),0))<=0)
			{
				printf("recv error: %s\n",strerror(errno));
				return -1;
			}
			recv1[n] = '\0';
			printf("Server: %s",recv1);
			memset(recv1,0,strlen(recv1));
		}
		if(FD_ISSET(fileno(fp),&rset))
		{
			if(fgets(send1,sizeof(send1),fp)==NULL)
			{
				printf("fgets error: %s\n",strerror(errno));
				return -1;
			}
			send(connectfd,send1,strlen(send1),0);
			memset(send1,0,512);
		}
		 
	}
	//3.close
	close(connectfd);
	exit(0);
	return 0;
}
```
其中，`setsockopt`, `signal`, `select` 为最需要关注的三个函数，在基本编程模型中并不涉及，但它们都有着至关重要的作用：
* `setsockopt` 设置了地址复用功能，避免了服务端重新启动时出现 Address already used 错误
* `signal` 避免了服务端子进程在客户端退出后变为僵尸进程
* `select` 可使得 socket I/O 和终端 I/O 为非阻塞，并在事件到来时及时响应
#### 7. UDP编程模型
和TCP编程模型很相似，在此仅贴上相关函数：
UDP服务端：

```c++
//1.create socket
serverfd = socket(AF_INET, SOCK_DGRAM, 0);
//2.address&port initialize
memset(&server_addr,0,sizeof(server_addr));
server_addr.sin_family = AF_INET;
server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
server_addr.sin_port = htons(1234);
//3.bind address&port
int bindc = bind(serverfd, (struct sockaddr *)&server_addr, sizeof(server_addr));
//4.noblocking with socket I/O or others
select(maxfd+1,&mset,NULL,NULL,NULL);
//5.communication
if(FD_ISSET(serverfd,&mset))
  recvfrom(serverfd,buf,512,0,(struct sockaddr *)&client_addr, &len)
if(FD_ISSET(fileno(fp),&mset))
  sendto(serverfd,buf1,strlen(buf1),0,(struct sockaddr *)&client_addr, len)
//6.close
close(serverfd);
```
UDP客户端：

```c++
//1.create socket
clientfd = socket(AF_INET, SOCK_DGRAM, 0);
//2.address&port initialize
memset(&server_addr,0,sizeof(server_addr));
server_addr.sin_family = AF_INET;
server_addr.sin_addr.s_addr = inet_addr("192.168.73.153");
server_addr.sin_port = htons(1234);
//3.unlock server, communication later
sendto(clientfd,"hello, udp!\n",12,0,(struct sockaddr *)&server_addr, len)
//4. noblocking with socket I/O or others
select(maxfd+1,&mset,NULL,NULL,NULL);
//5.communication
if(FD_ISSET(clientfd,&mset))
	recvfrom(clientfd,buf,512,0,(struct sockaddr *)&server_addr, &len)
if(FD_ISSET(fileno(fp),&mset))
	sendto(clientfd,buf1,strlen(buf1),0,(struct sockaddr *)&server_addr, len)
//6.close
close(clientfd);
```
****
## RTSP Analysis

#### 1. 协议构成
RTSP 和 RTP 的关系：
* RTP 是实时传输协议，是传输层协议，建立于 UDP 之上，一般不作为单独应用层协议处理
* RTSP 是实时流传输协议，它是与 HTTP 同等级的应用层网络协议，它是由 realmedia 开发，用来传输流媒体影像文件

RTSP 可基于 RTP 之上，比如常见的视频流传输过程：
**视频压缩文件－>RTP打包－>基于UDP的RTSP网络传输**
也可以不做成RTP包，直接基于UDP传送，如：
**视频压缩文件－>基于UDP的RTSP网络传输**
#### 2. 重要数据结构
整个工程由
* main.c
* ringfifo.c
* ringfifo.h
* rtputils.c
* rtputils.h
* rtsputils.c
* rtsputils.h
* rtspservice.c
* rtspservice.h

组成，在分析各文件内容和关系前，需要分析其中的重要数据结构及意义
ringfifo.h 中定义了环形缓冲区结构`struct ringbuf`:

    struct ringbuf {
        unsigned char *buffer;
        int frame_type;
        int size;
    };
rtputils.h 中定义了 RTP 通信中的传输头`StRtpFixedHdr`：

    typedef struct
    {
        /**//* byte 0 */
        unsigned char u4CSrcLen:4;      /**//* expect 0 */
        unsigned char u1Externsion:1;   /**//* expect 1, see RTP_OP below */
        unsigned char u1Padding:1;      /**//* expect 0 */
        unsigned char u2Version:2;      /**//* expect 2 */
        /**//* byte 1 */
        unsigned char u7Payload:7;      /**//* RTP_PAYLOAD_RTSP */
        unsigned char u1Marker:1;       /**//* expect 1 */
        /**//* bytes 2, 3 */
        unsigned short u16SeqNum;
        /**//* bytes 4-7 */
        unsigned long u32TimeStamp;
        /**//* bytes 8-11 */
        unsigned long u32SSrc;          /**//* stream number is used here. */
    } StRtpFixedHdr;
定义了 RTP 通信结构`struct _tagStRtpHandle`：

    typedef struct _tagStRtpHandle
    {
        int                 s32Sock;
        struct sockaddr_in  stServAddr;
        unsigned short      u16SeqNum;
        unsigned long long        u32TimeStampInc;
        unsigned long long        u32TimeStampCurr;
        unsigned long long      u32CurrTime;
        unsigned long long      u32PrevTime;
        unsigned int        u32SSrc;
        StRtpFixedHdr       *pRtpFixedHdr;
        StNaluHdr           *pNaluHdr;
        StFuIndicator       *pFuInd;
        StFuHdr             *pFuHdr;
        EmRtpPayload        emPayload;
    #ifdef SAVE_NALU
        FILE                *pNaluFile;
    #endif
    } StRtpObj, *HndRtp;
rtsputils.h 中定义了 RTSP 通信中的端口`port_pair`：

    typedef struct
    {
            int RTP;
            int RTCP;
    } port_pair;
定义了 RTSP 通信中的传输参数`RTP_transport`：

    typedef struct _RTP_transport
    {
      rtp_type type;
      int rtp_fd;
      union{
        struct {
            port_pair cli_ports;
            port_pair ser_ports;
            unsigned char is_multicast;
          } udp;
        struct {
            port_pair interleaved;
            } tcp;
                // other trasports here
      } u;
    } RTP_transport;
定义了 RTSP 通信中的会话`RTP_session`和`RTSP_session`：

    typedef struct _RTP_session {
      struct _tagStRtpHandle *hndRtp;
      RTP_transport transport;
        unsigned char pause;
        unsigned char started;
        int sched_id;
      struct _RTP_session *next;
    }RTP_session;
    
    typedef struct _RTSP_session {
        int cur_state;   /*会话状态*/
        int session_id; /*会话的ID*/

        RTP_session *rtp_session; /*RTP会话*/

        struct _RTSP_session *next; /*下一个会话的指针，构成链表结构*/
    } RTSP_session;
定义了 RTSP 通信中的接收缓冲`RTSP_buffer`：

    typedef struct _RTSP_buffer {
        int fd;    /*socket文件描述符*/
        unsigned int port;/*端口号*/

        struct sockaddr stClientAddr;

        char in_buffer[RTSP_BUFFERSIZE];/*接收缓冲区*/
        unsigned int in_size;/*接收缓冲区的大小*/
        char out_buffer[RTSP_BUFFERSIZE+MAX_DESCR_LENGTH];/*发送缓冲区*/
        int out_size;/*发送缓冲区大小*/

        unsigned int rtsp_cseq;/*序列号*/
        char descr[MAX_DESCR_LENGTH];/*描述*/
        RTSP_session *session_list;/*会话链表*/
        struct _RTSP_buffer *next; /*指向下一个结构体，构成了链表结构*/
    } RTSP_buffer;
定义了 RTSP 通信中的播放参数`stPlayArgs`：

    typedef struct _play_args
    {
        struct tm playback_time;                    /*回放时间*/
        short playback_time_valid;                 /*回放时间是否合法*/
        float start_time;                                   /*开始时间*/
        short start_time_valid;                        /*开始时间是否合法*/
        float end_time;                                     /*结束时间*/
    } stPlayArgs;

定义了 RTSP 通信中的调度列表`stScheList`：

    typedef unsigned int (*RTP_play_action)(unsigned int u32Rtp, char *pData, int s32DataSize, unsigned int u32TimeStamp);

    typedef struct _schedule_list
    {
        int valid;/*合法性标识*/
      int BeginFrame;
        RTP_session *rtp_session;/*RTP会话*/
        RTP_play_action play_action;/*播放动作*/
    } stScheList;

定义了 RTSP 通信中的主机参数`StServPrefs`：

    typedef struct _StServPrefs {
        char hostname[256];
        char serv_root[256];
        char log[256];
        unsigned int port;
        unsigned int max_session;
    } StServPrefs;
#### 3. 各部分接口函数意义
ringfifo.h 中定义了对环形缓冲区的操作接口：
```c++
int addring (int i);//改变环形缓冲区取出和放入位置，并能够在大于缓冲区最大值时归0，形成环
int ringget(struct ringbuf *getinfo);//得到取出位置的缓冲结构
void ringput(unsigned char *buffer,int size,int encode_type);//将新的缓冲结构存入放入位置
void ringfree();//释放缓冲
void ringreset();//将取出位置、放入位置和总大小归0
```
rtputils.h 中定义了RTP服务端的创建、关闭和发送接口：
```c++
unsigned int RtpCreate(unsigned int u32IP, int s32Port, EmRtpPayload emPayload);//创建
void RtpDelete(unsigned int u32Rtp);//关闭
unsigned int RtpSend(unsigned int u32Rtp, char *pData, int s32DataSize, unsigned int u32TimeStamp);//发送
```









