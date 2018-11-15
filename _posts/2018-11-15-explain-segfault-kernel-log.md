---
layout: post
title: Segfault内核日志解析
---

# {{ page.title }}

## 起因
测试服务器跑飞了，由于设置问题，没产生coredump文件，只在```/var/log/kern.log```里面找到如下两行日志：
```
Nov 14 21:29:43 test kernel: [467301.907292] skynet[6125]: segfault at 10 ip 00007f9f50d8291b sp 00007f9f55829470 error 4 in pvp.so[7f9f508e2000+899000]
Nov 14 21:29:43 test kernel: [467301.907302] skynet[6126]: segfault at 20 ip 00007f9f50e3b92c sp 00007f9f54e28600 error 4 in pvp.so[7f9f508e2000+899000]
```
这个问题之前没有出现过，应该是一个低几率出现的bug，不应该放过。这里记录一下这个问题的定位过程。

## 日志解析
根据产生该日志的[linux 内核代码][1]，及网络上搜索到的相关信息，如[Interpreting segfault messages][2]，可以确定该日志是由`SIGSEGV`信号的默认`handler`产生的。

去除掉干扰信息，选取第一条日志进行分析。
```
[467301.907292] skynet[6125]: segfault at 10 ip 00007f9f50d8291b sp 00007f9f55829470 error 4 in pvp.so[7f9f508e2000+899000]
```

日志中各个部分的含义如下:

* \[467301.907292\]
	产生日志时，系统运行时间。
	
* skynet\[6125\]
	进程名，线程id

* segfault
	[段错误][3]，可能由写只读内存，空指针引用，栈溢出等原因引起。

* at 10
	发生错误的虚拟内存地址。
	
	这里的地址值是0x10，明显是一个非法地址。
	
	这个非法地址是如何产生的呢？
	
	我们知道，对于c/c++的结构体、类等复合结构，访问其字段时，一般通过结构体指针加上成员偏移量，计算出实际的地址。一个合理的猜想是，访问空指针结构体的0x10偏移量位置的字段，就会产生访问地址0x10值的效果。我们看到下一条segfault日志的地址是0x20，同样可以作类似猜想。后面我们在问题定位时验证这一猜想。
	
* ip 00007f9f50d8291b
	发生错误时正在执行的指令地址。

* sp 00007f9f55829470
	发生错误时的栈指针。

* error 4
	错误码。
	
	追踪到错误码在内核中的[定义位置][4]。
```c
/*
 * Page fault error code bits:
 *
 *   bit 0 ==	 0: no page found	1: protection fault
 *   bit 1 ==	 0: read access		1: write access
 *   bit 2 ==	 0: kernel-mode access	1: user-mode access
 *   bit 3 ==				1: use of reserved bit detected
 *   bit 4 ==				1: fault was an instruction fetch
 *   bit 5 ==				1: protection keys block access
 */
enum x86_pf_error_code {
	X86_PF_PROT	=		1 << 0,
	X86_PF_WRITE	=		1 << 1,
	X86_PF_USER	=		1 << 2,
	X86_PF_RSVD	=		1 << 3,
	X86_PF_INSTR	=		1 << 4,
	X86_PF_PK	=		1 << 5,
};
```
	这条日志里面的`4`表示用户态内存访问错误。

* pvp.so\[7f9f508e2000+899000\]
正在执行的指令，所处的模块，及改模块加载的基址。0x899000表示改模块占用的内存大小。

## 第一现场
通过解析日志，我们得到如下信息。

进程`skynet`的线程`6125`在执行位于`pvp.so`模块偏移地址`4A091B`处指令时，访问了内存地址0x10，导致了段错误。

12微秒后，该进程的线程`6126`在执行位于`pvp.so`模块偏移地址`55992C`(00007f9f50e3b92c-7f9f508e2000)处指令时，访问了内存地址0x20，导致了段错误。

> 指令在模块内的偏移地址 = 指令虚拟地址 - 模块基址
> 4A091B = 00007f9f50d8291b - 7f9f508e2000
> 55992C = 00007f9f50e3b92c - 7f9f508e2000

这里有一个小问题，为什么同一个进程会产生两次段错误记录？按理说一个段错误，就会导致进程被杀死。这里两次段错误日志之间，仅相差12微秒，考虑到误差，猜想这两个错误是并发执行时，同时发生的。是不是太巧了？

借助`addr2line`工具，可以分析出出错时执行的代码。如下：

```shell
$ addr2line -e pvp.so -fCi 4A091B
pvp::PvpShootData::set_has_iweaponownerpvpid()
****/TGame_PVP.pb.h:28858

$ cat -n TGame_PVP.pb.h | head -28863 | tail -10
 28854  inline bool PvpShootData::has_iweaponownerpvpid() const {
 28855    return (_has_bits_[0] & 0x00000080u) != 0;
 28856  }
 28857  inline void PvpShootData::set_has_iweaponownerpvpid() {
 28858    _has_bits_[0] |= 0x00000080u;
 28859  }
 28860  inline void PvpShootData::clear_has_iweaponownerpvpid() {
 28861    _has_bits_[0] &= ~0x00000080u;
 28862  }
 28863  inline void PvpShootData::clear_iweaponownerpvpid() {
```

```shell
$ addr2line -e pvp.so -fCi 55992c
pvp::PvpShootData::iid() const
****/TGame_PVP.pb.h:28679

$ cat -n TGame_PVP.pb.h | head -28684 | tail -10
 28675    clear_has_iid();
 28676  }
 28677  inline ::google::protobuf::int32 PvpShootData::iid() const {
 28678    // @@protoc_insertion_point(field_get:pvp.PvpShootData.iid)
 28679    return iid_;
 28680  }
 28681  inline void PvpShootData::set_iid(::google::protobuf::int32 value) {
 28682    set_has_iid();
 28683    iid_ = value;
 28684    // @@protoc_insertion_point(field_set:pvp.PvpShootData.iid)
```

第一条日志表示，***读取***类`pvp:PvpShootData`的成员`_has_bits_`时，产生段错误。
第二条日志表示，***读取***类`pvp:PvpShootData`的成员`iid_`时，产生段错误。

使用`gdb`工具，加载`pvp.so`，查看`pvp::PvpShootData`的内存布局，可以清楚的看到`_has_bits_`和`iid_`的偏移量分别为0x10, 0x20。上面的猜想可以验证，也说明

```shell
(gdb) ptype/o pvp::PvpShootData
/* offset    |  size */  type = class pvp::PvpShootData : public google::protobuf::Message {
                         public:
                           static const int kIndexInFileMessages;
                           static const int kSttarPosFieldNumber;
                           static const int kIidFieldNumber;
                           static const int kDwtimeFieldNumber;
                           static const int kItarIdFieldNumber;
                           static const int kIdamageFieldNumber;
                           static const int kIdamagePartFieldNumber;
                           static const int kIweaponidFieldNumber;
                           static const int kIweaponOwnerPvpidFieldNumber;
                         private:
/*    8      |     8 */    class google::protobuf::internal::InternalMetadataWithArena 
        : public google::protobuf::internal::InternalMetadataWithArenaBase<google::protobuf::UnknownFieldSet, google::protobuf::internal::InternalMetadataWithArena> {

                               /* total size (bytes):    8 */
                           } _internal_metadata_;
/*   16      |     4 */    class google::protobuf::internal::HasBits<1ul> {
                             private:
/*   16      |     4 */        google::protobuf::uint32 has_bits_[1];

                               /* total size (bytes):    4 */
                           } _has_bits_;
/*   20      |     4 */    int _cached_size_;
/*   24      |     8 */    pvp::IntPosition *sttarpos_;
/*   32      |     4 */    google::protobuf::int32 iid_;
/*   36      |     4 */    google::protobuf::uint32 dwtime_;
/*   40      |     4 */    google::protobuf::int32 itarid_;
/*   44      |     4 */    google::protobuf::int32 idamage_;
/*   48      |     4 */    google::protobuf::int32 idamagepart_;
/*   52      |     4 */    google::protobuf::int32 iweaponid_;
/*   56      |     4 */    google::protobuf::int32 iweaponownerpvpid_;

                           /* total size (bytes):   64 */
                         }
```

根据以上分析，日志信息（指令地址、错误代号、内存地址）和实际代码，可以很好的匹配起来。确认第一现场是以下两个函数。

```cpp
 // 第二条日志第一现场
 28677  inline ::google::protobuf::int32 PvpShootData::iid() const {
 28678    // @@protoc_insertion_point(field_get:pvp.PvpShootData.iid)
 28679    return iid_;
 28680  }
 ...
 // 第一条日志第一现场
 28857  inline void PvpShootData::set_has_iweaponownerpvpid() {
 28858    _has_bits_[0] |= 0x00000080u;
 28859  }
```

## 追踪
由于没有堆栈信息，确认现场后，需要阅读源代码，确认bug位置。

以第二条日志为例，简单描述下过程。

第一现场的代码，是由`protoc`编译生成的解析协议的代码，暂不怀疑其有bug。

向上回溯2层，都是各只有一处调用：
```cpp
  1626  void static PvpShootData2Struct(const PvpShootData *stRes, PbStruct::PvpShootData &stRet)
  1627  {
  1628      stRet.iid = stRes->iid();
  1629      stRet.dwtime = stRes->dwtime();
  1630      stRet.itarid = stRes->itarid();
  1631      stRet.idamage = stRes->idamage();
  1632      if (stRes->has_sttarpos())
  1633      {
  1634          IntPosition2Struct(&stRes->sttarpos(),stRet.sttarpos);
  1635      }
  1636      stRet.idamagepart = stRes->idamagepart();
  1637      stRet.iweaponid = stRes->iweaponid();
  1638      stRet.iweaponownerpvpid = stRes->iweaponownerpvpid();
  1639
  1640  }
...
  1656  void static PvpShootDatas2Struct(const PvpShootDatas *stRes, PbStruct::PvpShootDatas &stRet)
  1657  {
  1658      stRet.iid = stRes->iid();
  1659      for (size_t nIndex = 0;nIndex < stRes->astdatas_size();++nIndex)
  1660      {
  1661          PbStruct::PvpShootData stPvpShootData;
  1662          PvpShootData2Struct(&stRes->astdatas(nIndex), stPvpShootData);
  1663          stRet.astdatas.push_back(stPvpShootData);
  1664      }
  1665
  1666  }
```

执行`1628`行时，`stRes`为空指针。`stRes`来源于`&stRes->astdatas(nIndex)`，是一个引用值，不太可能是这一层引起的空指针。

再向上回溯几层，可以追踪到`const PvpShootDatas *stRes`来源于一个`static`变量。

源代码如下：
```cpp
template<typename T>
static google::protobuf::Message* ParseMsg(const char* pData, int nLength)
{
   static T value;
   value.Clear();
   bool bRet = value.ParseFromArray((const void*)(pData), nLength);
   return (bRet == true) ? &value : NULL;
}
```

解析`protobuf`消息时，每种类型消息共用同一个结构体。在单线程模型下，同一时间只处理同一条消息，不会有问题。但在多线程模型下，该共享变量会被几个线程同时使用，就会出现上面的场景：正在遍历处理消息体中的一个数组时，数组内容被其他线程改变了。而`skynet`恰好是多线程模型。

## 结论
`kernel`打印的日志信息看似简单，仔细挖掘一下，能获得很多确定性信息，辅助问题定位。

## 参考
[1]:https://github.com/torvalds/linux/blob/9bb9d4fdce9e6b351b7b905f150745a0fccccc06/arch/um/kernel/trap.c#L153
[2]:https://stackoverflow.com/questions/2549214/interpreting-segfault-messages
[3]:https://en.wikipedia.org/wiki/Segmentation_fault
[4]:https://github.com/torvalds/linux/blob/6f0d349d922ba44e4348a17a78ea51b7135965b1/arch/x86/include/asm/traps.h#L148
