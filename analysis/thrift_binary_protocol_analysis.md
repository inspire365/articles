Thrift Binary Protocol Analysis

如下面就是一个非常简单的thrift定义文件，并以这个作为例子进行分析：
struct FooBar
{
1: i64 foo;
2: optional string bar;
}
service FbService
{
i32 CheckFooBar(1: FooBar fb, 2: i32 status);
} // end service

通过用：
> thrift -r -strict --gen cpp -o . fb.thrift
自动生成cpp文件
我们来关注函数：
int32_t FbServiceClient::CheckFooBar(const FooBar& fb, const int32_t status)
{
send_CheckFooBar(fb, status);
return recv_CheckFooBar();
}
自动生成的发送和接收代码，我们来关注发送基本可以知道协议是怎样的了
void FbServiceClient::send_CheckFooBar(const FooBar& fb, const int32_t status)
{
int32_t cseqid = 0;
oprot_->writeMessageBegin("CheckFooBar", ::apache::thrift::protocol::T_CALL, cseqid);
FbService_CheckFooBar_pargs args;
args.fb = &fb;
args.status = &status;
args.write(oprot_);
oprot_->writeMessageEnd();
oprot_->getTransport()->writeEnd();
oprot_->getTransport()->flush();
}
对于发送，首先写一个Message的头，然后再将一个参数构造成的结构体序列化，后面是消息的结束标记

template <class Transport_>
uint32_t TBinaryProtocolT<Transport_>::writeMessageBegin(const std::string& name,
const TMessageType messageType,
const int32_t seqid) {
if (this->strict_write_) {
// VERSION_1 (0x80010000) is taken by TBinaryProtocol
int32_t version = (VERSION_1) | ((int32_t)messageType);
uint32_t wsize = 0;
wsize += writeI32(version);
wsize += writeString(name);
wsize += writeI32(seqid);
return wsize;
} else {
uint32_t wsize = 0;
wsize += writeString(name);
wsize += writeByte((int8_t)messageType);
wsize += writeI32(seqid);
return wsize;
}
}
对于Message头，如果是严格写类型的(这里只讨论这种情况)，首先写一个32位的版本+消息类型，再写一个函数名，然后是一个32位的seq. 下面再看一下结构体的序列化

uint32_t FbService_CheckFooBar_pargs::write(::apache::thrift::protocol::TProtocol* oprot) const {
uint32_t xfer = 0;
xfer += oprot->writeStructBegin("FbService_CheckFooBar_pargs");
xfer += oprot->writeFieldBegin("fb", ::apache::thrift::protocol::T_STRUCT, 1); 
xfer += (*(this->fb)).write(oprot);
xfer += oprot->writeFieldEnd();
xfer += oprot->writeFieldBegin("status", ::apache::thrift::protocol::T_I32, 2); 
xfer += oprot->writeI32((*(this->status)));
xfer += oprot->writeFieldEnd();
xfer += oprot->writeFieldStop();
xfer += oprot->writeStructEnd();
return xfer;
}
template <class Transport_>
uint32_t TBinaryProtocolT<Transport_>::writeStructBegin(const char* name) {
(void) name;
return 0;
}
template <class Transport_>
uint32_t TBinaryProtocolT<Transport_>::writeFieldBegin(const char* name,
const TType fieldType,
const int16_t fieldId) {
(void) name;
uint32_t wsize = 0;
wsize += writeByte((int8_t)fieldType);
wsize += writeI16(fieldId);
return wsize;
}

template <class Transport_>
uint32_t TBinaryProtocolT<Transport_>::writeFieldEnd() {
return 0;
}
template <class Transport_>
uint32_t TBinaryProtocolT<Transport_>::writeFieldStop() {
return
writeByte((int8_t)T_STOP); // src/protocol/TProtocol.h: T_STOP = 0
}
对于为了传参数自动生成的结构体，会将每个字段的类型和fieldid写下来，结构体的最后会写一个停止符，即0
同样的方法可以查看其他类型的字段和回复消息的序列化。
例如，对于32位整数的序列化是这样的：
template <class Transport_>
uint32_t TBinaryProtocolT<Transport_>::writeI32(const int32_t i32) {
int32_t net = (int32_t)htonl(i32);
this->trans_->write((uint8_t*)&net, 4);
return 4;
}
为了不同机器的兼容性，先转化为网络字节序，再写入这32位数据。
 

综上，可以大致了解到，thrift TBinaryProtocolT 大约是这样的：


