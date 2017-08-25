---
title: Thrift源码分析--Protocol
date: 2017-08-25 15:45:45
tags:
  - RPC
  - 原理
  - 源码
  - Thrift
---

## 简介    

在介绍完IO流以后，基本上就知道了服务端和客户端是如何建立链接以及进行数据传输的，但是对于如何进行数据的序列化和反序列化，将在本文进行详 细记介绍。  
协议层的抽象类方法叫做：TProtocol,在包org.apache.thrift.protocol下。其中定义了协议的所有方法  

<!--more-->  

##### TProtocol的UML图：  
![TProtocol](/images/TProtocol.jpg "TProtocol结构")  
分别定义了简版json协议、json传输协议、二进制传输协议、密集型（对int和long进行了长度压缩，采用了zigzag 编码）详情我会在后面继续介绍。  

子类是我司封装的方法和官方的简版传输协议（对消息体进行了深度压缩）  

*先介绍下协议层需要完成的内容，定义了一个序列化和反序列化的协议:*  

```java
/**
 * Transport
 */
protected TTransport trans_;

/**
 * Writing methods.
 */

public abstract void writeMessageBegin(TMessage message) throws TException;

public abstract void writeMessageEnd() throws TException;

public abstract void writeStructBegin(TStruct struct) throws TException;

public abstract void writeStructEnd() throws TException;

public abstract void writeFieldBegin(TField field) throws TException;

public abstract void writeFieldEnd() throws TException;

public abstract void writeFieldStop() throws TException;

public abstract void writeMapBegin(TMap map) throws TException;

public abstract void writeMapEnd() throws TException;

public abstract void writeListBegin(TList list) throws TException;

public abstract void writeListEnd() throws TException;

public abstract void writeSetBegin(TSet set) throws TException;

public abstract void writeSetEnd() throws TException;

public abstract void writeBool(boolean b) throws TException;

public abstract void writeByte(byte b) throws TException;

public abstract void writeI16(short i16) throws TException;

public abstract void writeI32(int i32) throws TException;

public abstract void writeI64(long i64) throws TException;

public abstract void writeDouble(double dub) throws TException;

public abstract void writeString(String str) throws TException;

public abstract void writeBinary(ByteBuffer buf) throws TException;

/**
 * Reading methods.
 */

public abstract TMessage readMessageBegin() throws TException;

public abstract void readMessageEnd() throws TException;

public abstract TStruct readStructBegin() throws TException;

public abstract void readStructEnd() throws TException;

public abstract TField readFieldBegin() throws TException;

public abstract void readFieldEnd() throws TException;

public abstract TMap readMapBegin() throws TException;

public abstract void readMapEnd() throws TException;

public abstract TList readListBegin() throws TException;

public abstract void readListEnd() throws TException;

public abstract TSet readSetBegin() throws TException;

public abstract void readSetEnd() throws TException;

public abstract boolean readBool() throws TException;

public abstract byte readByte() throws TException;

public abstract short readI16() throws TException;

public abstract int readI32() throws TException;

public abstract long readI64() throws TException;

public abstract double readDouble() throws TException;

public abstract String readString() throws TException;

public abstract ByteBuffer readBinary() throws TException;

/**
 * Scheme accessor 用于选择标准传输协议还是简版协议
 */
public Class<? extends IScheme> getScheme() {
  return StandardScheme.class;
}
```

___需要注意的是在write方法里面，并没有wirte i32、bool、long、i16、double等并没有writeEnd方法，因为这些都是定长的___

在这个协议里面定义了一个请求是如何被请求端打包，并且被服务端解包的。

在这个协议中有一个类叫做TMessage。可以简单来看一下源码：  
```java
public final class TMessage {
  public TMessage() {
    this("", TType.STOP, 0);
  }

  public TMessage(String n, byte t, int s) {
    name = n;
    type = t;
    seqid = s;
  }

//方法名字
  public final String name;
//类型，使用的是 org.apache.thrift.protocol.TMessageType中的枚举，目前使用的都是其中的CALL
  public final byte type;
//目前都是0
  public final int seqid;

  @Override
  public String toString() {
    return "<TMessage name:'" + name + "' type: " + type + " seqid:" + seqid + ">";
  }

  @Override
  public boolean equals(Object other) {
    if (other instanceof TMessage) {
      return equals((TMessage) other);
    }
    return false;
  }

  public boolean equals(TMessage other) {
    return name.equals(other.name) && type == other.type && seqid == other.seqid;
  }
}
```

下面来详细解释下协议层的使用。  
首先来看其中定义的方法。  
TTransport:这个是传输层，Thrift传输层---I/O操作  
剩下的就是一个如何来构造一次传输的内容。  
先看分类，整体上分为两个大类别，每一个方法都有写和读的方法在对应着。先来说说写方法。  
##### __A__  
首先是定义了一个方法调用的开始writeMessageBegin，这个方法中写入TMessage这个对象，其中包含了调用的方法名字，调用类型（目前看到的都是CALL），以及当前消息的序列（可能与传输层有关）。  

##### __B__
根据函数中定义的参数顺序，依次调用相应的写方法和写结束方法（只有不可知长度的类型才需要结束方法）。写入字段的消息类型和消息长度，以及消息顺序号。  

*__比如如何写入string这个类型__*  

```java
public void writeString(String str) throws TException {
  try {
    byte[] dat = str.getBytes("UTF-8");
	//写入数组长度是i32，也就是说超长会出问题－ －
    writeI32(dat.length);
	//在传输层里面写入相应长度的数据
    trans_.write(dat, 0, dat.length);
  } catch (UnsupportedEncodingException uex) {
    throw new TException("JVM DOES NOT SUPPORT UTF-8");
  }
}
```

*__再比如如何写入map这个类型__*  
在写入之前用到的一些类型：  
```java
//首先是定义了TMap这个对象
public final class TMap {
  public TMap() {
    this(TType.STOP, TType.STOP, 0);
  }

  public TMap(byte k, byte v, int s) {
    keyType = k;
    valueType = v;
    size = s;
  }
//key类型
  public final byte  keyType;
//value类型
  public final byte  valueType;
  public final int   size;
}
```

其中引用了一个新的对象教TType。  
```java
//其中对key和value的定义是在 org.apache.thrift.protocol.TType
public final class TType {
  public static final byte STOP   = 0;
  public static final byte VOID   = 1;
  public static final byte BOOL   = 2;
  public static final byte BYTE   = 3;
  public static final byte DOUBLE = 4;
  public static final byte I16    = 6;
  public static final byte I32    = 8;
  public static final byte I64    = 10;
  public static final byte STRING = 11;
  public static final byte STRUCT = 12;
  public static final byte MAP    = 13;
  public static final byte SET    = 14;
  public static final byte LIST   = 15;
  public static final byte ENUM   = 16;
}
```

下面是正式的写入map这个对象,一共有两种写入方法，分别为标准写入和Tuple写入（简版）？  

__下面对比两种的写入的不同点:__   

*标准写入*
```java
if (struct.testMap != null) {
  if (struct.isSetTestMap()) {
	//这个参数是指TField其中表名了参数类型，名字，和id
    oprot.writeFieldBegin(TEST_MAP_FIELD_DESC);
    {
      oprot.writeMapBegin(new org.apache.thrift.protocol.TMap(org.apache.thrift.protocol.TType.STRING, org.apache.thrift.protocol.TType.I32, struct.testMap.size()));
      for (Map.Entry<String, Integer> _iter4 : struct.testMap.entrySet())
      {
        oprot.writeString(_iter4.getKey());
        oprot.writeI32(_iter4.getValue());
      }
      oprot.writeMapEnd();
    }
    oprot.writeFieldEnd();
  }
}
```

下面是其中的一段field的代码
```java
public class TField {
  public TField() {
    this("", TType.STOP, (short)0);
  }

  public TField(String n, byte t, short i) {
    name = n;
    type = t;
    id = i;
  }

//字段名字
  public final String name;
//字段类型详情在下面
  public final byte   type;
//id，会增长
  public final short  id;

  public String toString() {
    return "<TField name:'" + name + "' type:" + type + " field-id:" + id + ">";
  }

  public boolean equals(TField otherField) {
    return type == otherField.type && id == otherField.id;
  }
}
```

*简单版本*
```java
if (struct.isSetTestMap()) {
  {
    oprot.writeI32(struct.testMap.size());
    for (Map.Entry<String, Integer> _iter5 : struct.testMap.entrySet())
    {
      oprot.writeString(_iter5.getKey());
      oprot.writeI32(_iter5.getValue());
    }
  }
}
```


可见简版的写入相对于标准更节省空间。因此传输效率更高。我司采用的是标准传输协议。
在读取的时候采用的是对应的方法  
*标准协议：*
```java

case 4: // TEST_MAP
  if (schemeField.type == org.apache.thrift.protocol.TType.MAP) {
    {
      org.apache.thrift.protocol.TMap _map0 = iprot.readMapBegin();
      struct.testMap = new HashMap<String,Integer>(2*_map0.size);
      for (int _i1 = 0; _i1 < _map0.size; ++_i1)
      {
        String _key2; // required
        int _val3; // required
        _key2 = iprot.readString();
        _val3 = iprot.readI32();
        struct.testMap.put(_key2, _val3);
      }
      iprot.readMapEnd();
    }
    struct.setTestMapIsSet(true);
  } else {
    org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
  }
  break;
```

*简单版本*
```java
if (incoming.get(1)) { //incoming标示了有哪些optional被赋值
  {
    org.apache.thrift.protocol.TMap _map6 = new org.apache.thrift.protocol.TMap(org.apache.thrift.protocol.TType.STRING, org.apache.thrift.protocol.TType.I32, iprot.readI32());
    struct.testMap = new HashMap<String,Integer>(2*_map6.size);
    for (int _i7 = 0; _i7 < _map6.size; ++_i7)
    {
      String _key8; // required
      int _val9; // required
      _key8 = iprot.readString();
      _val9 = iprot.readI32();
      struct.testMap.put(_key8, _val9);
    }
  }
  struct.setTestMapIsSet(true);
}
```

##### __C__
然后writeMessageEnd，完成了消息的写入  


整体上来说，如果想要对代码进行缩减或者是更改，需要注意的地方很多，比如读写的顺序。而且在不断的发版过程中的变量顺序调整也会影响数据的序列化和反序列化。
因此借用官方的一句话：
```java
/**
 * Autogenerated by Thrift Compiler (0.8.0)
 *
 * DO NOT EDIT UNLESS YOU ARE SURE THAT YOU KNOW WHAT YOU ARE DOING
 *  @generated
 */
 ```
