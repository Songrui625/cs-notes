---
title: CS144 Lab0
date: 2021-03-20 23:01:25
tags: networking  

---

这个lab0需要我们实现两个东西：1、HTTP Client。2、一个in-memory的Byte Stream。<!--more-->

## Writing Webget

lab讲义已经提示的很完善了，直接上代码：

```c++
void get_URL(const string &host, const string &path) {
    // Your code here.
    TCPSocket sock{};
    sock.connect(Address(host, "http"));
    sock.write("GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\n\r\n");
    sock.shutdown((SHUT_WR)); //不再向服务器socket写入数据，关闭写端。
    while (!sock.eof()) {
        cout << sock.read();
    }
```

发送的HTTP报文格式一定要正确，如果不确定请先打印在console进行debug。  

提一下在客户端socket读取服务器返回的字节时已经不再向服务器socket写入字节，所以可以关闭写端。

```bash
$ ./apps/webget cs144.keithw.org /hello
HTTP/1.1 200 OK
Date: Sat, 20 Mar 2021 15:30:29 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Content-Type: text/plain

Hello, CS144!
```

## An in-memory reliable byte stream

要想正确的实现字节流，得站在用户的角度理解，为什么要设计这些接口，比如`end_input()`, 当你不再写入字节的时候，就需要调用这个接口，调用这个接口就得需要一个flag位，我们得声明一个bool _input_ended。再比如你要读取字节流里的内容，什么时候结束呢？有过经验的同学都知道当遇到`end of file`的时候就会结束，所以在实现`pop_output`的时候，要考虑什么时候将`is_eof`置为true？单线程会怎样，多线程会怎样？  

```c++
class ByteStream {
  private:
    // Your code here -- add private members as necessary.
    size_t _capacity;
    std::string _data;
    size_t _bytes_read;
    size_t _bytes_write;
    bool _input_ended;
    bool _is_eof;
    ...
  public:
```

```c++

ByteStream::ByteStream(const size_t capacity):
    _capacity(capacity),
    _data(),
    _bytes_read(0),
    _bytes_write(0),
    _input_ended(false),
    _is_eof(false) {}

size_t ByteStream::write(const string &data) {
    if (_input_ended) return 0;
    int ret = 0;
    size_t n = data.size();

    ret = remaining_capacity() >= n ? n : remaining_capacity();

    _data.append(data.substr(0, ret));
    _bytes_write += ret; 
    return static_cast<size_t>(ret);
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const {
    if (buffer_empty()) return _data;
    size_t n = len > buffer_size() ? len: buffer_size();
    return _data.substr(0, n);
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) {
    size_t n = len > buffer_size() ? buffer_size() : len;
    _bytes_read += n;
    _data.erase(0, n);
    // if (buffer_empty()) _is_eof = true; 为什么不可以只值判断data是否为空。
    // 如果data为空就宣告结束的话，那么就宣告后面无法再读取字节流中的数据了，因为已经读完了
    // 但事实上，writer仍有可能继续向socket写数据，因为input_ended标志没有被置位。
    // 所以会发生矛盾, 必须是writer不再向socket写数据时，并且data为空，才能宣告到达了end of file。
    if (buffer_empty() && input_ended()) _is_eof = true;
}

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
    string ret = peek_output(len);
    pop_output(len);
    return ret;
}

void ByteStream::end_input() { 
    _input_ended = true;
    if (buffer_empty()) _is_eof = true;
}

bool ByteStream::input_ended() const { return _input_ended; }

size_t ByteStream::buffer_size() const { return _data.size(); }

bool ByteStream::buffer_empty() const { return _data.empty(); }

bool ByteStream::eof() const { return _is_eof; }

size_t ByteStream::bytes_written() const { return _bytes_write; }

size_t ByteStream::bytes_read() const { return _bytes_read; }

size_t ByteStream::remaining_capacity() const { return _capacity - buffer_size(); }

```

最后贴个测试结果：

```bash
$ make check_lab0
[100%] Testing Lab 0...
Test project /home/linux/codespace/sponge/build
    Start 23: t_byte_stream_construction
1/9 Test #23: t_byte_stream_construction .......   Passed    0.00 sec
    Start 24: t_byte_stream_one_write
2/9 Test #24: t_byte_stream_one_write ..........   Passed    0.00 sec
    Start 25: t_byte_stream_two_writes
3/9 Test #25: t_byte_stream_two_writes .........   Passed    0.00 sec
    Start 26: t_byte_stream_capacity
4/9 Test #26: t_byte_stream_capacity ...........   Passed    0.30 sec
    Start 27: t_byte_stream_many_writes
5/9 Test #27: t_byte_stream_many_writes ........   Passed    0.00 sec
    Start 28: t_webget
6/9 Test #28: t_webget .........................   Passed    1.32 sec
    Start 50: t_address_dt
7/9 Test #50: t_address_dt .....................   Passed    0.00 sec
    Start 51: t_parser_dt
8/9 Test #51: t_parser_dt ......................   Passed    0.00 sec
    Start 52: t_socket_dt
9/9 Test #52: t_socket_dt ......................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 9

Total Test time (real) =   1.65 sec
[100%] Built target check_lab0
```

