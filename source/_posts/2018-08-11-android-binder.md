---
title:      "Android Binder"
date:       2018-08-11 09:38:43
author:     "lixiang"
categories: 工具
tags:
    - Android
---

> 在Binder库中，Service组件和Client组件分别使用模板类BnInterface和BpInterface来描述，前者称为Binder本地对象，后者称为Binder代理对象。Binder库中的Binder本地对象和Binder代理对象分别对应于Binder驱动程序中的Binder实体对象和Binder引用对象。<br><br>
> 下面将简要介绍Binder进程间的通信，代码包含Server进程和Client进程，Server进程实现了一个Service组件，负责管理前面介绍的虚拟硬件设备mydev的寄存器val，并且向Client进程提供访问服务。以下将实例划分为common、server和client三个模块，其中common实现了硬件访问服务接口IMydevService，以及Binder本地对象类BnMydevService和Binder代理对象类BpMydevService；server实现了一个Server进程，里面包含一个Service组件MydevService；client实现了一个Client进程，它通过一个BpMydevService代理对象去访问运行在Server进程中的Service组件MydevService所提供的服务。<br><br>

---

## 程序目录结构
```
~/Android/external/binder
----common
    ----IMydevgService.h
    ----IMydevService.cpp
----server
    ----MydevServer.cpp
    ----Android.mk
----client
    ----MydevClient.cpp
    ----Android.mk
```

## Server进程

- common/IMydevService.h

``` cpp
#ifndef IMYDEVSERVICE_H_
#define IMYDEVSERVICE_H_

#include <utils/RefBase.h>
#include <binder/IInterface.h>
#include <binder/Parcel.h>

#define MYDEV_SERVICE "com.eightplus.MydevService" //MYDEV_SERVICE用来描述Service组件MydevService注册到Service Manager的名称

using namespace android;

//硬件访问服务接口
class IMydevService: public IInterface
{
public:
    DECLARE_META_INTERFACE(MydevService);
    virtual int32_t getVal() = 0;//读取虚拟硬件设备mydev中寄存器val的值
    virtual void setVal(int32_t val) = 0;//向虚拟硬件设备mydev中寄存器val中写入值
};

//Binder本地对象类BnMydevService
class BnMydevService: public BnInterface<IMydevService>
{
public:
    virtual status_t onTransact(uint32_t code, const Parcel &data, Parcel *reply, uint32_t flags= 0);
};
#endif
```

- common/IMydevService.cpp

``` cpp
#define LOG_TAG "IMydevService"
#include <utils/Log.h>
#include "IMydevService.h"

using namespace android;

enum
{
    GET_VAL = IBinder::First_CALL_TRANSACTION;
    SET_VAL = IBinder::First_CALL_TRANSACTION + 1;
};

class BpMydevService: public BpInterface<IMydevService>
{
public:
    BpMydevService(const sp<IBinder>& impl): BpInterface<IMydevService>(impl)
    {
    }

public:
    int32_t getVal()
    {
        Parcel data;
        data.writeInterfaceToken(IMydevService::getInterfaceDescriptor());

        Parcel reply;
        remote()->transact(GET_VAL, data, &reply);

        int32_t val = reply.readInt32();
        return val;
    }

    void setVal(int32_t val)
    {
        Parcel data;
        data.writeInterfaceToken(IMydevService::getInterfaceDescriptor());
        data.writeInt32(val);

        Parcel reply;
        remote->transact(SET_VAL, data, &reply);
    }
};

IMPLEMENT_META_INTERFACE(MydevService, "com.eightplus.IMydevService");

status_t BnMydevService::onTransact(uint32_t code, const Parcel &data, Parcel *reply, uint32_t flags)
{
    switch(code)
    {
        case GET_VAL:
        {
            CHECK_INTERFACE(IMydevService, data, reply);
            int32_t val = getVal();
            reply->writeInt32(val);
            return NO_ERROR;
        }
        case SET_VAL:
        {
            CHECK_INTERFACE(IMydevService, data, reply);
            int32_t val = data.readInt32();
            setVal(val);
            return NO_ERROR;
        }
        default:
        {
            return BBinder::onTransact(code, data, reply, flags);
        }
    }
}
```

- server/MydevServer.cpp

``` cpp
#define LOG_TAG "MydevServer"
#include <stdlib.h>
#include <fcntl.h>
#include <utils/Log.h>
#include <binder/IServiceManager.h>
#include <binder/IPCThreadState.h>
#include "../common/IMydevService.h"

#define MYDEV_DEVICE_NAME "/dev/mydev"

//Service组件类MydevService
class MydevService: public BnMydevService
{
public:
    MydevService()
    {
        fd = open(MYDEV_DEVICE_NAME, O_RDWR);
        if(fd == -1)
            LOGE("Failed to open device %s.\n", MYDEV_DEVICE_NAME);
    }

    virtual ~MydevService()
    {
        if(fd != -1)
            close(fd);
    }

public:
    static void instantiate()
    {
        defaultServiceManager()->addService(String16(MYDEV_SERVICE), new MydevService());
    }

    int32_t getVal()
    {
        int32_t val = 0;
        if(fd != -1)
            read(fd, &val, sizeof(val));
        rerurn val;
    }

    void setVal(int32_t val)
    {
        if(fd != -1)
            write(fd, &val, sizeof(val));
    }

private:
    int fd;
};

int main(int argc, char** argv)
{
    MydevService::instantiate();//将MydevService组件注册到Service Manager中

    ProcessState::self()->startThreadPool();//启动一个Binder线程池
    IPCThreadState::self()->joinThreadPool();//将主线程添加到进程的Binder线程池中，用来处理来自Client进程的通信请求

    return 0;
}
```

- server/Android.mk
```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := ../common/IMydevService.cpp \
		MydevServer.cpp
LOCAL_SHARED_LIBRARIES := libcutils libutils libbinder
LOCAL_PACKAGE_NAME := MydevServer
include $(BUILD_EXECUTABLE)
```

## Client进程

 - client/MydevClient.cpp

 ``` cpp
 #define LOG_TAG "MydevClient"
 #include <utils/Log.h>
 #include <binder/IServiceManager.h>
 #include "../common/IMydevService.h"

 int main()
 {
     sp<IBinder> binder = defaultServiceManager()->getService(String16(MYDEV_SERVICE));
     if (binder == NULL) {
         LOGE("Failed to get mydev service: %s.\n", MYDEV_SERVICE);
         return -1;
     }

     sp<IMydevService> service = IMydevService::asInterface(binder);
     if (service == NULL) {
         LOGE("Failed to get mydev service interface.\n");
         return -2;
     }

     printf("Read original value from MydevService:\n");

     int32_t val = service->getVal();
     printf(" %d.\n", val);//0
     printf("Add value 1 to MydevService.\n");

     val += 1;
     service->setVal(val);

     printf("Read the value from MydevService again:\n");

     val = service->getVal();
     printf(" %d.\n", val);//1

     return 0;
 }
 ```

- client/Android.mk
```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := ../common/IMydevService.cpp \
		MydevClient.cpp
LOCAL_SHARED_LIBRARIES := libcutils libutils libbinder
LOCAL_PACKAGE_NAME := MydevClient
include $(BUILD_EXECUTABLE)
```

## 单独编译server和client

* ~/Android$ mmm ./external/binder/server/
* ~/Android$ mmm ./external/binder/client/

编译完成之后，就可以在out/target/product/generic/system/bin目录下看到编译结果的输出文件MydevServer和MydevClient了

## 重新打包system.img文件
* ~/Android$make snod

## 运行模拟器：
* ~/Android$emulator -kernel kernel/goldfish/arch/arm/boot/zImage &
* ~/Android$ adb shell
* root@android:/ # cd system/bin
* root@android:/system/bin # ./MydevServer &
* root@android:/system/bin # ./MydevXClient
```
Read original value from MydevService:
0.
Add value 1 to MydevService.
Read the value from MydevService again:
1.
```
