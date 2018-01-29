---
title: Thrift源码分析--原生调用例子
tags:
  - RPC
  - 原理
  - 源码
  - Thrift
date: 2017-08-15 00:10:20
---

## 简介    

在所有的内容开始之前，我们来使用一下，最原始的Thrift用法。并且附上源码～

<!--more-->  

#### Thrift文件(idl)
```java
namespace java com.kris.thrift.demo.service

service DemoService {
    //hello world
    string helloWorld(1:string name)

}
```

#### Server端(java)
```java
package com.kris.thrift.demo.service;

import org.apache.thrift.TProcessor;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.server.TServer;
import org.apache.thrift.server.TSimpleServer;
import org.apache.thrift.transport.TServerSocket;

public class Server{
    public static final int SERVER_PORT = 9001;

    public void startServer() {
        try {
            System.out.println("TSimpleServer start ....");

            TProcessor tprocessor = new DemoService.Processor<DemoService.Iface>(
                    new DemoServiceImpl());

            TServerSocket serverTransport = new TServerSocket(SERVER_PORT);
            TServer.Args tArgs = new TServer.Args(serverTransport);
            tArgs.processor(tprocessor);
            tArgs.protocolFactory(new TBinaryProtocol.Factory());
            TServer server = new TSimpleServer(tArgs);
            server.serve();

        } catch (Exception e) {
            System.out.println("Server start error!!!");
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        Server server = new Server();
        server.startServer();
    }

}
```

#### Client(java)
```java
package com.kris.thrift.demo.service;

import org.apache.thrift.TException;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;
import org.apache.thrift.transport.TTransportException;
import org.apache.thrift.transport.TFramedTransport;


import java.net.Inet4Address;
import java.net.InetAddress;
import java.net.NetworkInterface;
import java.net.SocketException;
import java.util.Enumeration;


public class Client {

    public static final int SERVER_PORT = 9001;
    public static final int TIMEOUT = 30000;
    public static TTransport transport = null;

    public static DemoService.Client client;

    public static void initClient() {
        try {
            transport = new TSocket(getLocalIp(), SERVER_PORT, TIMEOUT);

            transport.open();

            TProtocol protocol = new TBinaryProtocol(transport);

            client = new DemoService.Client(protocol);


        }
        catch(Exception e){
            e.printStackTrace();
        }

    }

    public static void destroyClient(){
        if (null != transport)
            transport.close();
    }


    public static String getLocalIp(){
        String ipAddress = null;
        Enumeration<NetworkInterface> net = null;
        try {
            net = NetworkInterface.getNetworkInterfaces();
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }

        while(net.hasMoreElements()){
            NetworkInterface element = net.nextElement();
            Enumeration<InetAddress> addresses = element.getInetAddresses();
            while (addresses.hasMoreElements()){
                InetAddress ip = addresses.nextElement();
                if (ip instanceof Inet4Address){

                    if (ip.isSiteLocalAddress()){

                        ipAddress = ip.getHostAddress();
                    }

                }

            }
        }
        return ipAddress;
    }
}
```


#### 代码下载地址
[Demo](/uploads/thrift例子.zip)
