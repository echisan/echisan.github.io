---
title: 操作系统-经典进程同步问题
date: 2019-10-15 14:27:32
tags:
  - 操作系统
  - 进程
---

# 经典进程同步问题
关于进程同步有一系列经典问题，比较代表性的有
- 生产者——消费者
- 读者——写者
- 哲学家进餐问题

## 生产者——消费者问题

1. 利用记录型信号量解决生产者-消费者问题

假定生产者消费者之间的公用缓冲池具有n个缓冲区，互斥信号量mutex实现诸进程对缓冲池的互斥使用，empty、full表示缓冲池中空缓冲区和满缓冲区的数量。

假设：缓冲池未满，生产者可将消息送入缓冲池；缓冲池未空，消费者则可以从缓冲池取走一个消息。

```c
// in为进队列的下标，out为消费队列的下标
int in=0,out=0;
// 循环队列
item buffer[n];
// mutex互斥锁，empty：剩余可用的队列空间，full当前已有的产品数量
semaphore mutex=1,empty=n,full=0;
void proceducer(){
    do{
        // 生产了一个产品nextp
        producer an item nextp;
        ...
        // 如果是empty为0，则进程会被放进等待的队列中
        wait(empty);
        // 获得一个锁的东西，比如此时有消费者在buffer中获取产品，则也会被放进等待队列中，直到被释放
        wait(mutex);
        // 已取得锁，可以将产品放进in的位置
        buffer[in]=nextp;   
        // 将该buffer当循环队列使用，计算下次放进去队列的下标
        in:=(in+1)%n;
        // 释放该锁，使消费者可以获得该互斥锁
        signal(mutex);
        // full表示有多少个产品，signal表示对full++，此时多了一个产品，则唤醒(通知)在等待队列中的消费者可以进行消费了
        signal(full);
    }while(true);
}

void consumer(){
    do {
        // 判断队列是否为空，为空则将进程放进阻塞队列中，直到proceducer()有新的产品被生产了，然后通知该阻塞的队列的进程，也就本进程了
        wait(full);
        // 获取操作队列的锁
        wait(mutex);
        // 获取产品
        nextc=buffer[out];
        // 循环队列，获取下一个出队列的下标
        out=(out+1)%n;
        // 释放队列的锁，通知生产者可以操作队列了
        signal(mutex);
        // empty++，队列增加了一个位置，通知生产在阻塞队列的进程可以生产了 
        signal(empty);
        // 消费。。
        consumer the item in nextc;
    } while(true);
}

// main 
void main(){
    cobegin
        proceducer(); consumer();
    coend
}
```

2. 利用AND信号量解决生存者-消费者问题  

几乎与上述描述一致，这里用AND信号量解决

```c
// in为进队列的下标，out为消费队列的下标
int in=0,out=0;
// 循环队列
item buffer[n];
// mutex互斥锁，empty：剩余可用的队列空间，full当前已有的产品数量
semaphore mutex=1,empty=n,full=0;
void proceducer(){
    do{
        // 创建新产品
        producer an item nextp;
        // AND信号量，这里复习一下Swait是什么
        // 一个死循环，然后判断是否所有条件都满足(大于等于1)，只要有其中一个条件不满足，就会将该进程放进等待或者阻塞队列中
        // 当满足条件了，会对swait中的参数进行减一操作，对empty减一，就是空闲数-1
        Swait(empty,mutex);
        // 把新生产的产品放进去队列
        buffer[in]=nextp;
        // 计算下标
        in:=(in+1)%n;
        // AND信号量的操作，也复习一下
        // 对Ssignal中的所有参数进行加一操作
        // 并唤醒等待或阻塞的进程
        Ssignal(mutex,full);
    } while(true);
}

void consumer(){
    do {
        // 上述一样，当full(未消费的产品数)大于等于1
        // 且mutex互斥锁没有被占用时
        // 取得资源的使用权，否则阻塞
        Swait(full,mutex);
        nextc=buffer[out];
        out=(out+1)%n;
        // 对mutex跟empty进行加一操作，表示释放资源
        // 并且唤醒在等待empty跟mutex的进程
        Ssignal(mutex,empty);
        consumer the item in nextc;
    }while(true);
}
```

3. 利用管程

在利用管程方法来解决生产者—消费者的问题时，首先为他们建立一个管程，并命名为producerconsumer，简称pc，其中包含两个过程
1. put(x)过程。将产品放进去缓冲池中，需要count<=N
2. get(x)过程。从缓冲池中取出一个产品，需要count>1

对于条件变量notfull和notempty，分别有两个过程cwait和csignal对它们进行操作
1. cwait(condition)过程：当管程被一个进程占用时，其它进程调用该过程时阻塞，并挂在condition的队列上
2. csignal(condition)过程：唤醒在cwait执行后阻塞的条件condition队列上的进程，如果进程不止一个，则选择其中一个唤醒，队列为空则无操作返回

```c
Monitor producerconsumer{
    item buffer[N];
    int in,out;
    // 条件变量
    // notfull，没有空余的区域
    // notempty，队列为空
    condition notfull,notempty;
    int count;
    public:
    void put(item x){
        // 当队列中的数据以达到最大值
        // 则将本进程放进等待队列中阻塞
        if(count>=N)cwait(notfull);
        buffer[in]=x;
        in=(in+1)%N;
        count++;
        // 唤醒队列其中一个进程可以进行消费了
        csignal(notempty);
    }
    void get(item x){
        // 当队列没有数据时
        // 将进程放进等待数据的队列中阻塞
        if(count<=0)cwait(notempty);
        x=buffer[out];
        out=(out+1)%N;
        count--;
        // 当数据被消费了
        // 通知被阻塞的生产者可以继续生产了
        csignal(notfull);
    }
    // init
    {   in=0;out=0;count=0; }
}PC;
```
在利用管程解决生产者-消费者时，其中的生产者消费者可描述为：
```c
void producer(){
    item x;
    while(true){
        produce an item in nextp;
        PC.put(x);
    }
}

void consumer(){
    item x;
    while(true){
        PC.get(x);
        consume the item in nextc;
    }
}

void main(){
    cobegin;
    producer(); consumer();
    coend;
}

```

## 哲学家进餐问题

### 利用记录型信号量解决问题

可以用一个信号量表示一只筷子，五个信号量构成一个数组
```
semaphore chopstick[5]={1,1,1,1,1}
```

```c
do{
    // 拿左手边的筷子，如果没有则阻塞
    wait(chopstick[i]);
    // 拿右手边的筷子，如果没有则阻塞
    wait(chopstick[(i+1)%5];
    
    // 拿到筷子了，吃饭。。
    
    // 释放左手边的筷子
    signal(chopstick[i]);
    // 释放右手边的筷子
    signal(chopstick[(i+1)%5];
}
```
上述解法可保证不会有两个相邻的哲学家同时进餐，但却有可能引起死锁。假如五位哲学家同时极饿而各自拿起左边的筷子时，都将因为无筷子可拿而无限期地等待。对于这样的死锁问题，可采取一下几种解决方法：
1. 至多只允许有四位哲学家同时去拿左边的筷子，最终能保证至少有一位哲学家能够进餐，并在用完时能释放他用过的两只筷子，从而使更多的哲学家能够进餐
2. 仅当哲学家的左、右两只筷子均可用时，才允许他拿起筷子进餐
3. 规定奇数号哲学家先拿他左边的筷子，然后再去拿右边的筷子，而偶数号哲学家则相反。按照这规定，将是1、2号哲学家竞争1号筷子，3、4号竞争3号筷子。即五位哲学家都先竞争奇数号筷子，获得后再去竞争偶数号筷子，最后总有一位哲学家能获得两只筷子而进餐。


### 利用AND信号量机制
从上述可以知道，第二个解决方案
> 2. 仅当哲学家的左、右两只筷子均可用时，才允许他拿起筷子进餐

```c
semaphore chopstick chopstick[5]={1,1,1,1,1}
do {
    ...
    // think
    ...
    Sswait(chopstick[(i+1)%5],chopstick[i]);
    // eat
    ...
    Ssignal(chopstick[(i+1)%5],chopstick[i]);
} while(true)
```

## 读者-写者问题
也就是对某个文件进行read,write操作，read操作可以多个进程同时执行，但是write只能允许一个进程执行并且此时不允许read操作以及write操作，否则回引起混乱

### 利用记录型信号量解决问题
为实现reader与writer进程间在读或写时的互斥信号量`Wmutex`。另外，再设置一个整形变量`Readcount`表示正在读的进程数目，由于只要有一个reader进程在读，就不允许writer进程去写。因此，仅当readcount=0,reader进程才需要执行wait(wmutex)操作。当reader执行了readcount减1的操作后其值为0时，才需要执行signal(Wmutex)，以便让writer进程写操作。因为readcount是一个临界值，所以需要设置一个互斥锁rmutex

> 注意⚠️：这里注意两个信号量，`rmutex`表示对`readcount`临界资源的互斥锁，而`wmutex`表示对被读写的资源的一个互斥锁

```c
// 两个信号量
semaphore rmutex=1,wmutex=1;
// 记录获得读锁的进程数
int readcount=0;
void reader(){
    do{
        // 阻塞等待readcount的互斥锁
        wait(rmutex);
        // 当readcount为0时，代表没有进程在进行读操作
        // 那么有可能此时该资源正在被写
        // 所以需要等待该资源的锁
        if(readcount==0) wait(wmutex);
        // 获得资源的锁后，说明写操作已经结束了
        // 可以进行一些操作了
        // 对readcount++操作，表示此时多了一个进程进行读操作
        readcount++;
        // 此时可以将rmutex锁释放了
        // 因为同一时间可以有多个进程进行读操作
        // 若没有释放rmutex锁的话
        // 别的进程则无法获取到该锁
        // 进而退化成，读操作也是互斥了的
        signal(rmutex);
        // ..读操作
        
        // ..读完
        // 读完之后获取readcount锁
        // 因为需要对readcount进行减1的操作
        wait(rmutex);
        readcount--;
        // 如果当没有进程在读时，那么可以释放该资源的互斥锁了
        // 因为只要有1个进程在读，那么就无法进行写操作
        // 所以当readcount为0时，则可以释放该锁
        // 便可以唤醒writer进程，进行写操作
        if(readcount==0) signal(wmutex);
        // 对readcount--完后，就可以释放该锁了
        signal(rmutex);
    }while(true);
}
// writer进程
void writer(){
    do{
        // 等待该资源的互斥锁
        wait(wmutex);
        // .. 写操作
        // 释放
        signal(wmutex);
    }while(true);
}
```

### 利用信号量集机制解决读者-写者问题
这里与上述的问题不同，它增加了一个限制，最多只允许RN个读者同时读，为此又引入一个信号量L，并赋予其初值RN，通过执行`wait(L,1,1)`操作来控制读者的数目，每当有一个读者进入时，要先执行`wait(L,1,1)`操作，使L的值减1。当有RN个读者进入读后，L便减为0，当第RN+1个读者想要进入读时，则会阻塞。

> RN: 表示限制的读进程数  
> mux: 表示对资源的写锁

```c
int RN;
semaphore L=RN,mx=1;
void reader(){
    do{
        // 获取读锁
        // 读锁资源总共L个，一次最少获取资源1个，本次获取1个资源
        Swait(L,1,1);
        // 这里表示所有进程都可以获取锁
        // mx--
        Swait(mx,1,0);
        // 读操作...
        // 释放资源
        Ssignal(L,1);
    }while(true);
}
void writer(){
    do{
        // 等待读锁，获取全部的读资源锁
        Swait(mx,1,1,L,RN,0);
        // 写操作...
        // 写完释放写锁
        Ssignal(mx,1);
    }while(true);
}
```
> `Swait(S,1,0)`当S>=1时，允许多个进程进入某特定区，当S变为0后，则阻止任何进程进入特定区

