# Lab 3: The TCP Sender

在这个lab中，我们会实现一个TCP Sender

## Introduction
TCP是在不可靠的数据报上用可靠地方式传递一对流量控制的字节流，每个方向维护一个。TCP连接中
有两方，每一方同时充当`sender`和`receiver`，这两方被称为`端点`或者`对等方`。

`TCP Sender`会在Segment中写入所有和Lab2中`TCP Receiver`有关的字段：即序列号`seqno`、`SYN`、
`payload`以及`fin`。但是`TCP Sender`只感兴趣收到Segment中的确认号`ackno`和窗口大小`window size`。

TCP Sender主要负责以下部分。
* 跟踪对等方`Tcp Receiver`的接收窗口`window size`和确认号`ackno`
* 当对等方`window size`没有满以及字节流`ByteStream`中有数据时，**尽可能多的填满窗口**，即发送
Segment(必要时将`SYN`或者`FIN`置位)。
* 跟踪那些已经发送但还没有被确认的Segment, 我们称为 `outstanding segments`
* 重发那些已经发送但没有被确认并且已经`超时`的Segment

## Implementing the TCP sender
`TCP Sender`的基本思想：给定一个即将要发送的字节流，将它们分为许多Segment，发送到对等方
的`TCP Receiver`，如果对等方没有及时确认，重发Segment。

接下来看一下要实现的函数，以下每一个函数可能**最后都要发送Segment**。

1.`void fill window()`  
`TCP Sender`需要尽可能填满窗口:只要字节流中可以读取新的字节**并且**窗口有足够的空间，那就读取
尽可能多的字节然后形成Segment，发送到对等方。

我们必须确保Segment尽可能的大，尽可能填满窗口，但是**payload大小不能超过TCPConfig::MAX PAYLOAD SIZE (1452 bytes)**。
，这里有个坑点就是payload大小最大不能超过1452B,但如果流已经end了，并且窗口还有大小，那么`fin`也需要置位。  

**反复强调**：`syn`和`fin`也占一个序列号，也算在window size中。

**算法设计**
1. 首先需要发送`SYN`包，即三次握手的第一次，此时不允许携带payload。
2. 如果已经发送了`FIN`包，那就不需要再填充窗口了，直接返回。
3. 计算缓冲区窗口大小，缓冲区窗口 = 接收方通告窗口大小 - 正在网络中传输的字节
数(`FIN`和`SYN`也算在内)。这里有个坑，当window size为0时，我们需要把它当作1来处理，
但是即使把它当作1，现在网络中正在传输的字节大于1，这样我们的缓冲区窗口仍然没有剩余的空间
来发送Segment。
4. 还有一个坑点是，当我们缓冲区窗口还有大小的时候，一旦发送过了`FIN`包或者Segment不占用
任何一个序列空间就得直接返回了。
```c++
void TCPSender::fill_window() {
    TCPSegment seg;
    // state: closed --> SYN_SENT
    // first handshake of three handshakes, segment can not carry payload.
    if (0 == _next_seqno) {
        _syn_sent = true;
        seg.header().syn = true;
        send_segment(seg);
        return;
    }
    if (_fin_sent) {
        return;
    }
    // calculate the remaining buffer window size
    // ack like it was one if window size is zero and
    // buffer window size still have space
    int32_t remaining_win_cap = !_window_size ? 1 - bytes_in_flight() : _window_size - bytes_in_flight();
    if (remaining_win_cap)
    while (remaining_win_cap > 0 && !_fin_sent) {
        size_t payload_size = min(static_cast<size_t>(remaining_win_cap), TCPConfig::MAX_PAYLOAD_SIZE);
        seg.payload() = std::move(_stream.read(payload_size));
        remaining_win_cap -= seg.length_in_sequence_space(); 
        // check if we still have enough space to fit fin flag
        if (remaining_win_cap > 0 && _stream.eof()) {
            seg.header().fin = true;
            _fin_sent = true;
            remaining_win_cap -= 1;
        }
        if (0 == seg.length_in_sequence_space()) {
            return;
        }
        send_segment(seg); 
    }
}
```

