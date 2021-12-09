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

1. **void fill_window()**  
   `TCP Sender`需要尽可能填满窗口:只要字节流中可以读取新的字节**并且**窗口有足够的空间，那就读取
   尽可能多的字节然后形成Segment，发送到对等方。

我们必须确保Segment尽可能的大，尽可能填满窗口，但是**payload大小不能超过TCPConfig::MAX PAYLOAD SIZE (1452 bytes)**。
，这里有个坑点就是payload大小最大不能超过1452B,但如果流已经end了，并且窗口还有大小，那么`fin`也需要置位，这是因为从这

个常量的名字可以看出来，它只影响payload，而不影响`fin`的占位大小。

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

2. **void ack received( const WrappingInt32 ackno, const uint16 t window_size)**

当收到`Receiver`的`ackno`的`window size`时，意味着`Sender`知道了`Recevier`感兴趣的字节窗口，左边界就是`ackno`，右边界就是`ackno + window size - 1`，即区间[`ackno`, `ackno` + `window size`)。知道了`ackno`意味着，`ackno`之前的所有字节都已经被确认了，我们就可以把这些已经被接收（确认）的字节从`outstanding segment`中`pop`掉，就是简单判断一下`outstanding segment`中的每个`segment`最后一个字节的序列号`seqno`是否小于`ackno`即可。一旦成功接收了Segment，意味着缓冲区窗口可能有新的空间，得继续调用`fill window()`，这是为了最大化网络带宽的利用。

**算法设计**

1. 首先我们要忽略那些无效的`ackno`，即确认了那些根本没有发送过的序列号`ackno`，或者是小于最近一次收到的确认号`_ackno_rcvd`。还需要注意的是如果`ackno`与最近一次收到的`_ackno_rcvd`相同，我们不能忽略这个确认，因为可能之前`window size`是比较小的，而我们还有Segment在网络上传输，说明缓冲区窗口是没有空间支持`Sender`发送新的Segment的，所以`Sender`需要等待一个`window size`更大的确认，让`Sender`直到可以发送新的`Segment`了。
2. 把`segment_outstading`中那些已经被完全确认的segment全部出队，调用`fill_window()`，尽可能地发送更多的字节，最大化网络带宽的利用。
3. 如果成功确认了一个segment，成功确认指的是我们确实从`segment_outstanding`中有pop掉任何一个segment，或者说新收到的确认号`ackno`大于最近一次收到的确认号`_ackno_rcvd`，那`Sender`就得把重传计时器重置(包括重置超时时间为`initial_RTO`)，如果所有的segment都已经确认，即`segment_outstanding`空了，那就可以关闭重传计时器了。

```c++
//! \param ackno The remote receiver's ackno (acknowledgment number)
//! \param window_size The remote receiver's advertised window size
void TCPSender::ack_received(const WrappingInt32 ackno, const uint16_t window_size) {
    uint64_t abs_ackno = unwrap(ackno, _isn, _ackno_rcvd);
    // invalid ackno should return
    if (abs_ackno < _ackno_rcvd || abs_ackno > _next_seqno) return;
    // use for update retx timer metadata
    bool success_receipt = abs_ackno > _ackno_rcvd;
    // update ackno and window size metadata
    _ackno_rcvd = abs_ackno;
    _window_size = window_size;
    // remove any that have now been fully acknowledged 
    // (the ackno is greater than all of the sequence numbers in the segment).
    TCPSegment seg;
    while (!_segments_outstanding.empty()) {
        seg = _segments_outstanding.front();
        // next seqno of last byte's seqno of seg
        WrappingInt32 end_seqno = seg.header().seqno + seg.length_in_sequence_space();
        if (end_seqno.raw_value() <= ackno.raw_value()) {
            _bytes_in_flight -= seg.length_in_sequence_space();
            _segments_outstanding.pop();
        } else {
            break;
        }
    }

    fill_window();

    if (success_receipt) {
        _timer.reset_RTO(_initial_retransmission_timeout);
        // When all outstanding data has been acknowledged, stop the retransmission timer.
        if (_segments_outstanding.empty()) {
            _timer.stop();
        } else {
            // If the sender has any outstanding data, restart the retransmission timer
            // so that it will expire after RTO milliseconds (for the current value of RTO).
            _timer.reset();
        }
        _retransmission_num = 0;
    }
}
```

3. **void tick( const size t ms_since_last_tick )**

每当过了一段时间——如果发送出去的segment没有及时被确认，那么就需要重传它们。每当这个`tick`被调用，指意味着自从上次调用`tick`函数经历了多长时间。我单独设计了一个类来实现超市重传计时器, 代码非常简单, 但为了不贴一大段代码，只发一些关键实现。

```c++
class RetransmissionTimer {
  private:
    size_t _timer{0};
    size_t _RTO;
    bool _running{false};

   public:
	...
    void tick (const size_t ms_since_last_tick) {
      if (!_running) {
        return;
      }
      _timer += ms_since_last_tick;
    }

    inline void double_RTO() { _RTO *= 2; }
};
```

**算法设计**

1. 如果经历了`ms_since_last_tick`之后，超过了超时重传时间`RTO`并且还有发送的segment没有被确认，就真正触发了重传，重传第一个超时的segment。
2. 在1的基础上，只有当通告窗口大于0时，需要对计时器进行指数退避。并且重传次数加一重传后，让计时器重新计时。

```c++
//! \param[in] ms_since_last_tick the number of milliseconds since the last call to this method
void TCPSender::tick(const size_t ms_since_last_tick) {
    _timer.tick(ms_since_last_tick);
    if (_timer.timer() >= _timer.RTO() && !_segments_outstanding.empty()) {
        TCPSegment seg = _segments_outstanding.front();
        _segments_out.push(seg);
        _retransmission_num += 1;
        if (_window_size > 0) {
            // exp backoff
            _timer.double_RTO();
        }
        _timer.reset();
    }
}
```

4. **void send_empty_segment()**

发送一个空的segment，segment不占用任何的字节空间，但是需要设置正确的需序列号。这在我们即将实现的`TCPConnection`中可能会非常有用，我们只想发送一个空的确认。

```c++
void TCPSender::send_empty_segment() {
    TCPSegment seg;
    seg.header().seqno = wrap(_next_seqno, _isn);
    _segments_out.push(seg);
}
```

5. **void send_segment(TCPSegment& *seg*) **

发送一个真正占用字节空间的segment，需要注意的是当成功发送了之后，如果没开启超时重传计时器，那就启动，并重置超时重传计时器的初始化超时重传时间。

```c++
void TCPSender::send_segment(TCPSegment& seg) {
    if (seg.length_in_sequence_space() == 0) return;

    seg.header().seqno = next_seqno();
    // update metadata
    _bytes_in_flight += seg.length_in_sequence_space();
    _next_seqno += seg.length_in_sequence_space();
    // push seg into sending queue
    _segments_out.push(seg);
    _segments_outstanding.push(seg);
    // if the timer is not running, start it running
    if (!_timer.running()) {
        _timer.run();
    }
    _timer.reset_RTO(_initial_retransmission_timeout);
}
```

到这里，这个lab就已经完成了，下面贴一下测试。

```bash
$ make check_lab3
[100%] Testing the TCP sender...
Test project /home/linux/codespace/sponge/build
      Start  1: t_wrapping_ints_cmp
 1/33 Test  #1: t_wrapping_ints_cmp ..............   Passed    0.00 sec
      Start  2: t_wrapping_ints_unwrap
 2/33 Test  #2: t_wrapping_ints_unwrap ...........   Passed    0.00 sec
      Start  3: t_wrapping_ints_wrap
 3/33 Test  #3: t_wrapping_ints_wrap .............   Passed    0.00 sec
      Start  4: t_wrapping_ints_roundtrip
 4/33 Test  #4: t_wrapping_ints_roundtrip ........   Passed    0.18 sec
      Start  5: t_recv_connect
 5/33 Test  #5: t_recv_connect ...................   Passed    0.00 sec
      Start  6: t_recv_transmit
 6/33 Test  #6: t_recv_transmit ..................   Passed    0.04 sec
      Start  7: t_recv_window
 7/33 Test  #7: t_recv_window ....................   Passed    0.00 sec
      Start  8: t_recv_reorder
 8/33 Test  #8: t_recv_reorder ...................   Passed    0.00 sec
      Start  9: t_recv_close
 9/33 Test  #9: t_recv_close .....................   Passed    0.00 sec
      Start 10: t_recv_special
10/33 Test #10: t_recv_special ...................   Passed    0.00 sec
      Start 11: t_send_connect
11/33 Test #11: t_send_connect ...................   Passed    0.00 sec
      Start 12: t_send_transmit
12/33 Test #12: t_send_transmit ..................   Passed    0.04 sec
      Start 13: t_send_retx
13/33 Test #13: t_send_retx ......................   Passed    0.00 sec
      Start 14: t_send_window
14/33 Test #14: t_send_window ....................   Passed    0.01 sec
      Start 15: t_send_ack
15/33 Test #15: t_send_ack .......................   Passed    0.00 sec
      Start 16: t_send_close
16/33 Test #16: t_send_close .....................   Passed    0.00 sec
      Start 17: t_send_extra
17/33 Test #17: t_send_extra .....................   Passed    0.01 sec
      Start 18: t_strm_reassem_single
18/33 Test #18: t_strm_reassem_single ............   Passed    0.00 sec
      Start 19: t_strm_reassem_seq
19/33 Test #19: t_strm_reassem_seq ...............   Passed    0.00 sec
      Start 20: t_strm_reassem_dup
20/33 Test #20: t_strm_reassem_dup ...............   Passed    0.01 sec
      Start 21: t_strm_reassem_holes
21/33 Test #21: t_strm_reassem_holes .............   Passed    0.00 sec
      Start 22: t_strm_reassem_many
22/33 Test #22: t_strm_reassem_many ..............   Passed    0.05 sec
      Start 23: t_strm_reassem_overlapping
23/33 Test #23: t_strm_reassem_overlapping .......   Passed    0.00 sec
      Start 24: t_strm_reassem_win
24/33 Test #24: t_strm_reassem_win ...............   Passed    0.05 sec
      Start 25: t_strm_reassem_cap
25/33 Test #25: t_strm_reassem_cap ...............   Passed    0.06 sec
      Start 26: t_byte_stream_construction
26/33 Test #26: t_byte_stream_construction .......   Passed    0.00 sec
      Start 27: t_byte_stream_one_write
27/33 Test #27: t_byte_stream_one_write ..........   Passed    0.00 sec
      Start 28: t_byte_stream_two_writes
28/33 Test #28: t_byte_stream_two_writes .........   Passed    0.00 sec
      Start 29: t_byte_stream_capacity
29/33 Test #29: t_byte_stream_capacity ...........   Passed    0.37 sec
      Start 30: t_byte_stream_many_writes
30/33 Test #30: t_byte_stream_many_writes ........   Passed    0.01 sec
      Start 48: t_address_dt
31/33 Test #48: t_address_dt .....................   Passed    0.02 sec
      Start 49: t_parser_dt
32/33 Test #49: t_parser_dt ......................   Passed    0.00 sec
      Start 50: t_socket_dt
33/33 Test #50: t_socket_dt ......................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 33

Total Test time (real) =   0.91 sec
[100%] Built target check_lab3
```

现在我们能回答一些常见面试题了。

#### TCP如何保证可靠传输的？
TCP可靠传输解决的是`Segment`在网络中会出现的四种问题：`损坏`、`重复`、`乱序`和`丢失`。
* 损坏：TCP通过对Segment进行校验和校验来处理。
* 重复和乱序：TCP通过维护一对字节流来处理，每个字节都会有一个唯一的索引，将乱序的字节合并为有序的字节,
  同时忽略那些重复的字节。
* 丢失：TCP会维护一个超时重传计时器，有一个默认初始值`initial_RTO`，每发一个新的Segment，计时器会启动，
  如果经过了`RTO`时间还没有被确认，这时候就会重传Segment，同时执行指数退避，`RTO`变为原来的两倍，但TCP也不会
  一直重传，当重传次数超过某一固定的数值时，TCP认为这个连接已经无望了，出问题了，需要中止。

#### 什么是ARQ(auto repeat request)?
ARQ为自动重传请求，这是TCP的指责之一，只要发送的Segment没有在一定时间内及时确认，那TCP就要重传它，
因为TCP必须保证对等方无论如何至少要接收到一次Segment。

