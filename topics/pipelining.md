## 使用 pipelining 提升 Redis 性能

### 请求 / 响应协议与 RTT（往返时间）

Redis 是一款基于请求 / 响应协议的采用客户端-服务器架构模型设计的 TCP 服务。

即一个请求处理完成通常需要经历如下步骤：

- 客户端向服务器发送查询请求，以进程阻塞的方式，监听 socket 端口读取服务端响应。
- 服务端处理 redis 命令，并将响应发送回客户端。

下面给出 4 个命令演示这个处理过程：

- Client: INCR X
- Server: 1
- Client: INCR X
- Server: 2
- Client: INCR X
- Server: 3
- Client: INCR X
- Server: 4

客户端和服务器通过网络进行连接。这个连接处理过程可能很快完成（通过 loopback 接口）也可能非常慢（建立了一个多次跳转的网络连接）。无论网络延迟情况如何，数据包都需要经历从客户端到服务端，然后再由服务端返回回复信息给客户端。

这个时间称之为 RTT （往返时间）。当客户端在一个逻辑处理中需要发送多个请求时（如向某个链表中同时加入多个元素，或向单个数据库中添加多个键），显而易见 RTT 对性能影响有多大。

如果实现了 loopback 接口，RTT 会显著缩短（比如，在我的本机 127.0.0.1 ping 得到的结果介于 0 到 44 毫秒之间），然而如果你需要在单词逻辑处理中多次写入是这个响应时间依然无法接受。

幸运的是有一种解决方案能够提升这种场景的性能。

### Redis 管道

一个请求 / 响应服务器可以实现即使客户端未接收到旧的请求的响应数据，服务器依然能够处理新的请求。通过这种方式，客户端能够一次发送多个命令，并且只需在最后一次性读取所有命令的响应，而不必等待命令都接收到服务端响应。

这种已经被广泛运用几十年的技术称为管道（pipelining）。比如，POP3 协议早已实现对改功能的支持，以提升从邮件服务器下载新邮件的速度。

Redis 在其早期本本中就已经支持了管道技术，所以无论你使用哪个版本的 Redis ，都能够使用其管道功能。下面是一个使用的例子：

```shell
$ (printf "PING\r\nPING\r\nPING\r\n"; sleep 1) | nc localhost 6379
+PONG
+PONG
+PONG
```

这一次我们没有为每个命令花费 RTT 开销，而是只用了一个命令的开销时间。

非常明确的，用管道顺序操作的第一个例子如下：
- Client: INCR X
- Client: INCR X
- Client: INCR X
- Client: INCR X
- Server: 1
- Server: 2
- Server: 3
- Server: 4

**重要提示：** 当客户端使用管道发送命令时，服务端在内存中以类似队列形式处理所有命令并响应。所以当你需要使用管道发送大量命令时，最好以合理的命令数量分批次提交，比如先发送一个 10k 的命令，读取响应回复后，再次发送 10k 命令，依次列推。这样他们的速度几乎一样，但是对于 10k 的命令来讲，能够额外用于回复队列的内存会达最大值。

### 不仅仅是 RTT

管道不仅降低了 RTT 延迟响应，还能够提升单个 Redis 服务能够处理的命令数量。这是因为，从数据结构和回复响应的角度讲在不使用管道时，单个命令处理时的资源开销很小，但是对于套接字的 I/O 消耗却非常大。这里涉及到了 read() 和 write() 的系统调用，太意味着从用户态到内核态的转换。上下文切换会导致速度严重下降。

当使用管道，通常只要单个 read() 系统调用读取多个命令，然后通过一个 write() 系统调用投递回复响应。介于此，每秒执行的请求总数将随着管道命令长度的增加几乎呈线性增长，最终达到不使用管道的基线的10倍，如下图所示:

![iops](https://redis.io/images/redisdoc/pipeline_iops.png)

### 测试用例

下面我们使用支持管道功能的 Ruby Redis 客户端进行基准测试，来测试管道技术对性能的提升：

```ruby
require 'rubygems'
require 'redis'

def bench(descr)
    start = Time.now
    yield
    puts "#{descr} #{Time.now-start} seconds"
end

def without_pipelining
    r = Redis.new
    10000.times {
        r.ping
    }
end

def with_pipelining
    r = Redis.new
    r.pipelined {
        10000.times {
            r.ping
        }
    }
end

bench("without pipelining") {
    without_pipelining
}
bench("with pipelining") {
    with_pipelining
}
```

在我的 Mac OS X 系统上运行基于 loopback 接口的脚本结果如下，即使 RTT 以足够小也能提升不少性能：

```
without pipelining 1.185238 seconds
with pipelining 0.250783 seconds
```

如你所见，通过管道我们提升了 5 倍的性能。

### 管道 VS 脚本

使用 [Redis 脚本](https://redis.io/commands/eval)（2.6 及以上版本）编程在服务端去处理大量管道操作能够更好提升性能。脚本的最大优势之一是能够同事降低读写数据的延迟，让读取、计算和写入操作变得非常快（由于客户端需要接收到 read 命令的响应后才能调用 write 命令，这种场景下，管道技术则无能为力）。

有时应用程序可能会在管道中使用 [EVAL] 或 [EVALSHA] 命令。这点可能是 Redis 通过 [SCRIPT LOAD] 命令支持该功能的又一力证（它保证了调用 [EVALSHA] 命令不会失败的风险）。

@TODO 

[原文](https://redis.io/topics/pipelining)
