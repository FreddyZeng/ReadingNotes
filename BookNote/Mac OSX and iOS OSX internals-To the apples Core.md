# Mac OSX and iOS OSX internals-To the apples Core（深入解析Mac OS X & iOS 操作系统）笔记

## OSX 架构描述

**分用户态和内核态**

**用户态：**

- 操作系统的GUI体验层和第三方应用APP
- 常用的一些OC，C，C++库。系统的C库，如GCD

**内核态：**

- BSD内核
- Mach内核和IOKit（硬件驱动开发）
- 硬件

## 用户体验层

为了支持MAC电脑或者iOS设备的GUI显示

### Aqua

**登录过程的支持**

**登录过程**

用户态的launchd(引导进程)->CoreGraphics的WindowServer触发启动（-daemon参数）->CGXServer函数（确定是后台运行还是前台运行）

->LoginWindow（根据**登录配置**，显示GUI?）

**登录配置**

- /etc/ttys中把 getty配置注释代码打开，就可以直接进入控制台模式
- 或者修改登录窗口信息，**Name and password**，在用户名输入>console，密码留空就可以进入控制台模式。 CTRL+D（可以退出控制台模式），或者reboot重启

### QuickLook

QuickLook是一个给窗口Finder提供按下空格键就可以显示缩略图功能。它提供类似插件的机制加载功能。把一个没有main入口的特殊编译程序，放到系统级别的路径（/System/Library/QuickLook）或者用户级别(~/Library/QuickLook)的路径进行加载功能。

quickLook是后台进程

**通过配置加载插件**

/System/Library/LaunchAgents/com.apple.quicklook.plist。按路径加载插件

**通过qlmanage管理插件**

qlmanage -m查看插件列表，在plugins：下面

**插件和文件内容的绑定关系**

UTI标识mac上的文件格式，插件应该定义每一个UTI格式的支持


### Spotlight

Spotlight是一个搜索技术。和Alfred相似，不过第三方的总比apple自己的要好用。。

Spotlight是GUI进程

mds是后台进程

Spotlight是基于mds做功能管理，由一些命令命令行可以使用

- 查询功能，mdfind
- 管理数据功能，mdutil
- 测试Spotlight导入器（Spotlight import）工具，mdimport
- 文件元数据（查询功能），mdls

支持mds的功能由**fsevent**和**Spotlight import**

- fsevent的作用是，监听系统的文件增删改。通知给mds
- Spotlight import的作用是，提供对文件的信息进行提取。
  系统级别路径在/System/Library/Spotlignt
  用户级别路径在~/Library/Spotlignt
  可以在Xcode中创建**Spotlight import**

**iOS Spotlight**

iOS的高版本是可以容Siri Kit中使用Spotlight功能，低版本好像由其它一个框架提供。

### Accessibility

这个书中没有，有空搜索回来再不错



## Darwin

### shell

sh

bash

zsh

一般都是用bash或者zsh

**开启ssh**

使用root权限，把/System/Library/LaunchDaemons中的ssh.plist的disable改为false

估计修改后，就会通过fsevent发送文件变化通知，然后系统就会启动ssh进程。相反，也会把ssh进程杀死。

**类似这样的的操作，以后可以通过搜/System/Library/目录下的一些内容来进行功能开启，使用grep或者rg**

**iOS ssh**

默认root密码为alpine，密码来源是第一个iOS称号是alpine

### 文件系统
HFS+默认是保持文字是区分大小写，访问文件时是不区分大小写。也就是写入区分大小写信息，访问文件名字时不区分。

HFS+有2个功能可以开启

- HFSX，开启大小写敏感，这个在iOS上是开启的，为了性能？
- JHFS，开启日志记录。如果有事务操作（读写硬盘，存储数据行为？）被中断（突然没电？突然无辜重启？），可以通过日志恢复操作继续完成，或者丢弃一个操作，可能会造成数据丢失。



## UNIX系统目录

- /bin，常用的命令
- /sbin，system bin，系统管理工具，网络配置等
- /usr，用户安装的程序
- /usr的bin,sbin，应用程序级别的工具。
  lib是应用程序级别的共享库。
  include，是应用程序级别的头文件。
- /etc->/private/etc。大量系统级别的配置
- /dev，BSD设备，文件描述符。
- /tmp->/private/tmp软连接，临时目录，所有人都可以访问
- /var->/private/var软连接，保持日志等

ls -lO，可以显示文件是否显示在Finder，有一个hidden属性就不会显示在Finder程序中。

### OS X特有目录

- /Applications，应用程序目录
- /Developer，开发者工具目录，如Xcode
- /Library，系统应用的数据，
- /Nerwork，发现网络邻居，虚拟目录
- /System, 系统目录，包含框架和一些内核模块
- /Users, 所有的用户目录都在这里
- /volumes，挂载的硬盘都在这里
- /Cores，核心转储（core dump）？核心虚拟内存镜像？

### iOS 文件系统的区别

- iOS是HFSX大小写。OSX 默认是不敏感可以配置
- iOS内核以kernelcache形式放在/System/Library/Caches/com. apple. kernelcaches，而OSX是压缩的。可能是因为iOS最强速度，所以不压缩。而且Mac电脑驱动很多，所以需要压缩
- /Applications 指向/var/stash/Applications，越狱特征
- 没有/Users，只有/User
- 没有/volumes，无法加载磁盘
- /Developer只有和Xcode调试的时候才会有，使用iOS设备进行开发调试

## bundle

bundle可以理解为一个多文件载体格式。

- 可以是目录结构
- 可以是一个共享库的文件格式，动态库，动态加载

## 应用程序和app

- OSX 应用程序的bundle相对整洁
- iOS应用程序bundle不太整洁
  系统应用在，/Applications日录下
  越狱应用在，/var/ stash/ Applications
  AppStore购买的应用，/var/mobileApplications，并且有GUID格式作为目录（沙盒机制）.

iOS程序运行时访问的文件路径，都是沙盒环境。也就是在GUID的目录下。

ipa是一个压缩文件。类似java的jar包

### info.plist

bundle的info.plist保持了程序的元信息。

info.plist的格式

- XML
- 二进制格式
- JSON

**使用plutil进行格式转换为XML**

- file命令查看文件格式
- `plutil -convert xml --< Info plist converted Info plist`

**info.plist文件属性**

一些应用程序的属性配置，如区域，应用程序名字，app bundleid，类型，处理外部了解等。

info.plist也定义了一些特殊的功能访问和是否开启它

CF能被iOS和OSX使用，NS一般时OSX

### Resources目录

资源文件不编译进二进制文件，减少二进制文件大小. 有什么用?app启动速度和二进制的大小由关系..

### NIB文件

.nib是二进制的plist文件
.xib->.nib

plutil可以让.nib->.xib反编译.

### 通过.lproj文件实现国际化

bundle中的有个.lproj专门来处理国际化,根据文件的命名确定多语言

zh_TW繁体, zh_CN简体, en英语

### 图标文件.icns

应用的程序icon,封装在appname.icns中

### CodeResources

CodeResources是用来计算bundle的所有文件的内容hash. 为了防止篡改,或者判断文件是否损坏
files属性,key是文件名的bash64转码.value是hash



**应用程序的默认配置保持**

- defaults, NSUserDefault. 保持在~/Library/Preferences中
- /Library/Preferences是全局的保持位置.
- 还有一个NSGlobalDomain,这个保持在隐藏文件.GlobalPreferences.plist中.会存在用户目录下,也存在全局目录下.可能和NSUserDefault Suite有关系

通过命令行编辑plist

`defaults`命令可以操作NSUserDefault的信息

**使用默认配置加载程序**

- 点击文件(launch)
- 右键文件,open with
- 终端, open命令

LaunchServices是Core Services的功能.

LaunchServices操作

- lsregister,重置 LaunchServices数据库
  ![image-20210103141345395](C:%5CUsers%5CAdministrator%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20210103141345395.png)

## 框架

框架是bundle格式。**框架基于库。**框架是apple特有的格式，而库是Unix的格式

苹果系统的框架是基于库组成的。

### 框架bundle格式

- CodeResources/	指向 Code Signature/ Coderesources plist文件的符号链接
- Headers/		向这个框架提供的。h文件所在目录的符号链接
- Resources/	指向这个框架所需要的，nib文件(用于GUI)、,1proj文件和其他文件所在目录的符号链接
- Versions/		在这个子目录下实现版本控制
- A/ 	字母名称的目录表示这个框架的版本
- Current/		指向这个框架首选版本的符号链接
- framework-name	指向这个框架首选版本的二进制文件的符号链接

框架的格式，都是使用了类似软连接的格式。

- current可以进行版本切换，并且允许向前和向后兼容
- 框架的GCC支持-framework选项，GCC编译参数是-I。而库的参数是-l，有相似之处

#### 查找框架

**查找框架有什么用呢？**

查找当前框架的功能，或者查找它的二进制来使用。上面的lsregister就是

也可以了解有没有更多的功能吧

- /System/Library/Frameworks包含苹果提供的框架，iOS和OSX中都是如此。
- /Network/ Library/Frameworks可能会用于（罕见）网络上安装的公共框架。
- /Library/Frameworks保存第三方框架(不出预料，在iOS上这个目录为空)。
- ~/Library/ Frameworks保存用户提供的框架（如果有的话）

**框架搜索路径**

在编译的时候，会修改下面的路径环境变量，再编译。

-  DYLD FRAMEWORK PATH
-  DYLD LIBRARY PATH
-  DYLD FALLBACK FRAMEWORK PATH
-  DYLD FALLBACK LIBRARY PATH

**私有框架**

私有框架是在框架内部使用，或者苹果自己的app应用使用。不公开

**私有框架位置**

/System/Library/PrivateFrameworks

#### 顶层框架

**Cocoa框架**

Cocoa框架是应用编程环境，OC和Swift主要语言。

Cocoa本质很小，它主要是基于苹果其它框架构建的。--将Cocoa依赖的框架的符号重新导出为自己的符号。

Cocoa框架使用了一种叫做**保护伞（umbrella）框架**的术语。意思是Cocoa框架按支持应用层程序员功能开发来来划分框架。如UIKit是需要UI功能，而Foundation是基础功能。UIKit它们是import很多框架，所以成为**保护伞（umbrella）框架**。而且，**UIKit引入，都是一些编译好的框架。这样可以提升编译速度，开发速度。组件化的时候，可以参考这个点。**

### OS X 和iOS公共框架列表

## 库

**库的位置**

- 库保存在 /usr/lib目录中(不存在/lib目录)。

**库的后缀**

- .dylib后缀动态库
- 而Unix的ELF格式是.so动态库

**兼容**

- .dylib不兼容.so

**库的显示**

- 在MAC电脑上的iOS SDK中容易查找到
- 在iOS文件系统查看不到。因为采用了库缓存优化机制，混肴机制。？？为什么

**核心库的结构**

- 一些核心库在libSystem.B.dylib中，然后使用软连接指向libSystem.B.dylib
- libSystem.B.dylib也引入一些内部库，位置在/usr/ib/system。

也就是说，libSystem.B.dylib的结构依赖，一部分在自身内部，一部分是在Mac电脑/usr/ib/system路径。

然后，由一些库，会使用软链接到libSystem.B.dylib中。

**为什么要这样？**

**猜想**

可能libSystem.B.dylib是公用的，维护一份。而libSystem.B.dylib依赖环境路径的，可能是不同平台，它的行为是不一样的。**也就是说依赖环境路径的，是一个抽象的行为。可能环境是Mac，也可能是iOS。而libSystem.B.dylib内部的，都是必须在一起通用稳定的逻辑**

**查询库的依赖关系**

`otool -L libSystem.B.dylib `

**库的加载**

dyld称为Mach-O加载器

## 其它应用程序类型

### JAVA(仅限mac OS)

### Widget

**保持位置**

/Library/Widgets

**组成**

- widgetname. html
- widgetname. js
- widgetname.css
- 本地化语言
- Images
- 二进制插件，支持JS无法访问的原生功能

### BSD/Mach原生程序

支持内核的应用程序编写。并且有MacPorts和fink，类似linux的apt。

OSX 支持POSIX

## 系统调用

程序的功能简单功能运算是可以操作通用寄存器完成。

如果是打开文件等（访问硬件），需要操作系统管理的，都是基于/usr/lib/libsystem. B dylib。所以程序需要链接/usr/lib/libsystem. B dylib才能访问这些系统调用。**这也是操作系统的意义所在，操作系统来帮我们管理硬件资源，我们仅仅是使用API，告诉操作系统应用程序需要什么功能和操作**

OS X组成

- Mach调用
- POSIX调用

### POSIX

POSIX是一套C语言API接口，提供抽象的API。由XNU中的BSD内核提供，BSD层本身有一些设计功能，依赖Mach内核的基础功能

使用它

- 通过抽象的API使用
  移植很好，都是C函数和C++API调用
- 通过抽象的API函数编号使用。函数编号是函数地址
  不能移植，因为mac OS的二进制格式是Mach-O，而Unix是ELF

### Mach系统调用

BSD层是对Mach内核包装

**32位机器上**

由于POSIX只定义了非负的系统调用，所以负数空间还没有使用，因此Mach就使用负数

**64位机器上**

Mach系统调用是正数，但是以0x200000开头，而 POSIX调用编号以0100000。函数编号是函数地址



应用程序函数或者命令->/usr/lib/libsystem. B dylib->Mach或BSD的POSIX接口。函数编号是函数地址



**使用otool查看Mach-O文件的函数调用**

可以看到函数的调用二进制，就是把函数地址放进eax寄存器，再设置一些参数到寄存器，再调用函数。

根据上面的规则，可以知道这个系统函数是Mach内核还是BSD内核

## XNU概述

组成

- Mach微内核
- BSD封装层
- libKern
- IOKit，内核驱动开发。支持驱动硬件。

### Mach

**功能**

- 进程和线程抽象
- 虚拟内存管理
- 任务调度
- 进程间通信和消息传递机制

是一些基础功能API

### BSD层

组成

- UNIX进程模型
- POSX线程模型( Pthread)及其相关的同步原语
-  UNX用户和组
-  网络协议栈( BSD Socket API)
-  文件系统访问
-  设备访问(通过/dev目录访问)

提供用户，线程和进程模型。属于提供针对某些用户需求设计的功能。由Mach内核基础功能来支持。

### libkern

为了支持内核的编写，支持C++语法。感觉是对内核驱动开发的封装，为开发者加速。

### IOKit

基于libkern，形成一个类似应用程序（Cocoa框架开发）的内核开发环境。



# OS X和iOS使用的技术

 BSD和XNU的差异

- fsevent和apple event
- 沙盒机制
- HFS+文件系统，日志系统

## BSD相关的特征

### sysctl

sysctl是用来获取内核值，或者修改内核值得函数。

sysctl结构

- sysctl命令
- sysctl库
- __sydctl系统编号（202）

内核变量通过内核变量得管理信息库名称访问（MIB）.意思就是，通过一些编号来分类获取和修改属性。

sysctl -A可以获取所有变量。但是不能遍历变量。

sysctl函数参数在头文件<sys/sysctl.h>

可以运行时，在特殊的命名空间和appleprofile名称空间注册额外的sysctl变量值和增加命名空间。

有一些kdebug函数，获取值的命令ps都是基于sysctl获取的。



### kqueue

kqueue是一个内核事件通知机制。一个kqueue是描述符，这个描述符会阻塞等待知道一个事件发生才运行。**所以可以用来在用户态或者内核态进行进程通信。**

**用户态使用过程**

- 创建一个kqueue队列
- 创建一个kevent事件结构体，并且使用EV_SET初始化事件，传入进程id或者端口信息或者文件路径以及事件的类型。
- 调用kevent函数进行设置事件监听，需要传入kqueue队列和kevent结构体。
- 掉哦那个事件函数kevent进行阻塞。如果当前进程满足事件过滤器则返回，否则阻塞。

**事件过滤器参数，事件类型常量<sys/event.h>**

- 端口接收消息
- proc--进程操作活动+被发送信号等
- 审计活动？？
- 监视发给进程特定的信号
- 定时器
- 文件内容读写
- 文件非内容信息变更

**猜想**

**所以sysctl修改内核环境变量导致触发新的行为，可能是通过kqueue文件内容变化+进程端口+进程活动事件通知描述符来实现的。**

### 审计（OS X）

审计就是实时获取内核事件日志，打log？？

#### 审计命令

- praudit命令记录内核日志log
- auditreduce用来搜索日志记录
- `praudit /dev/auditpipe`获取实时访问事件

#### 过滤审计

-   `audit_class`日志的事件掩码转为可读的名称
- `audit_control`日志策略，日志管理数据
- `audit_event`日志的事件标识符转为可读的名称
- `audit_user`针对用户来过滤事件类别
- `audit_warn`对audit后台服务程序产生警告信息，设置条件通知，或者打log

#### 审计操作函数

<bsm/audit.h>文件

- auditctl在一个文件路径创建审计日志文件
- audit记录审计
- auditon配置审计
- getaudit获取审计会话状态
- setaudit设置审计会话状态
- getauid/setauid审计会话ID

**苹果额外的审计**

- mach_port_name_t
  返回一个Mach端口，可以通过这个端口发送消息到审计
- audit_session_join
  把Mach端口添加到审计会话

- audit_session_port
  返回一个Mach端口，可以通过这个端口发送消息到审计

### 强制访问控制

**Mandatory Access Control**

通过给进程，PID等设置标签，而标签归纳为要一个集合。没有相同的集合就不能访问。

这样比按照用户来设置权限更加强，控制性更好。

MAC是一种架构框架，提供一种机制是插件的形式对系统安全性进行控制。可以在MAC中注册特殊的内核拓展。**每一个系统调用首先通过MAC的验证，然后才能处理来自用户态的请求。**

MAC本身不会限制系统调用，而是注册的模块会有一定的权限处理逻辑。这类似**依赖注入，控制反转**，策略模块控制的逻辑是在外部传入，灵活控制变化。

MAC模块的注册可以通过sysctl进行动态注册，`sysctl security`进行列举出所有标签集合

**MAC函数**

mac函数都是__mac_ 开头

著名的iOS越狱，把proc_enforce和 vnode_enforce 设置为0，禁用iOS上的代码签名。

**权限控制**

MAC是OS X的隔离机制，也是iOS entitlement的机制。

## 3.2OS X和iOS特有的技术

### 3.2.1用户和组的管理

### 3.2.2系统配置

删除etc，使用configd

### 3.2.3记录日志

### 3.2.4 Apple事件和 Apple Script

### 3.2.5 Fsevents

**猜想**

底层是用Mach的api做的？还是kqueue这套机制

### 3.2.6通知

IPC机制，notifyd守护程序和分布式通知服务器distnoted。

<notify.h>头文件有说明，也有UNIX信号和Mach端口以及文件描述符相关通知的函数。

#### 通知测试工具notifyutil

- notifyutil -w等待通知
- notifyutil -p发布通知

### 3.2.7其他重要的API

#### Grand Central Dispatch(第4章)

#### Launch Daemon(第7章)

#### XPC(第7章)

#### kdebug(第5章)

#### 系统套接字（第17章）

#### Mach AP(第9章第11章）

#### I/O Kit AP(第19章)

## OS X和iOS的安全机制

### 3.3.1代码签名

### 3.3.2隔离机制（沙盒化）

### 3.3.3 Entitlement:更严格的沙盒
dynamic-codesigning,允许动态生成代码创建具有rwx权限的内存页）

entitlement本身内嵌在开发者自己的证书中。上传到 App Store的应用程序内嵌了 entitlement

### 3.3.4沙盒机制的实施

AppleMobileFileIntegrity I内核扩展(被人们亲切地称为AMFD)是iOS特有的内核扩展模块

系统调用，触发MAC权限获取加载额外的权限内核，解析sanbox.kext更新权限，再确定系统调用是否执行。

## 4 庖丁解进程：Mach-0格式、进程以及线程内幕

### 4.1.1进程和线程

进程组，把进程添加到进程组setpgrp

进程有父子结构，fork创建子进程。子进程结束后可以通过main的return返回值或者exit参数返回。

线程当做基本单元。线程只不过是一组寄存器的状态，一个进程中可以存在多个线程。
一个进程内的所有线程都共享虚拟内存空间
文件描述符和各种句柄。进程的抽象依然以一个或多个线程的容器的形式保存下来。

### 4.1.2进程生命周期

<sys/proc. h>中可以找到进程的状态常量

SIDL->RUN->SZOMB->Dead

RUN的切换状态有可运行和正在运行,原因是时间片切换.

RUN可能切换到SLEEP,因为访问硬件资源或者锁的时候,避免占用CPU计算,可以释放CPU等待唤醒
RUN可能切换到STOP,是程序代码主动停止执行,可能是程序用户有暂停需求.

线程依赖进程.如果进程被杀死,线程会结束.

#### 1.僵尸状态

正常来说,子进程完成任务,就会返回一个值给父进程.

如果父进程先结束,子进程就会被pid为1的进程收养.

如果父进程创建了一个子进程,子进程完成了任务释放了所有资源,但是占用这pid变成了僵尸进程,需父进程使用wait来获取子进程的返回值,子进程才会释放pid.或者父进程结束,pid为1的进程使用wait处理这些已经完成任务,但是还没有处理返回值的子进程.

#### 2. pid_suspend 和 pid_resume

<sys/syscall.h>头文件可以找到函数定义

它是Mach内核级别的休眠.而且它们可以调用多次,但是次数抵消才会让进程执行.

OS X调用已经变了.iOS还在使用.

pid_shutdown_sockets在进入后台的时候会被调用, 应该是用来关闭进程的套字节

### 4.1.3UNX信号

<sys/signal.h>

软中断,标识进程发生了异常

有来源,和没有信号捕抓时默认的处理

POSIX允许向线程发送信号

#### 进程的基本安全性

UID用户ID
GID进程组ID

OSX 由其它方式维护UID和GID

OSX 没有Capability，Capability可以将root权限分解为小权限，然后再委托给非root用户。不过OSX 和iOS提供了沙盒机制和entitlements，只要开发者申请，就可以拥有合适的权限。

### 4.2可执行文件

| 可执行格式         | 魔术                                               | 用途                                                         |
| ------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| ELF                | \x7FELF                                            | linux和UNIX原生格式，OS X不支持                              |
| 脚本               | #！                                                | 把跟在#！后面的当成命令，然后后面的文本内容作为stdin传递给命令 |
| 通用二进制格式（胖二进制） | 0xcafebabe（小尾顺序）<br />0xbebafeca（大尾顺序） | 包含多种架构支持的二进制格式，只在OSX上支持                  |
| Mach-O             | 0xfeedface(32位)<br/> 0xfeedfacf(64位)             | OSX的原生二进制格式                                          |

### 4.3通用二进制格式

通用二进制文件格式定义在<mach-o/fat.h>上

工具

- lipo命令提取和删除或者合并二进制格式文件
- otool

fat_header

- macgic标识Mach-O文件类型
- nfat_arch有多少种格式

fat_arch

- offset，这个架构的二进制代码在整个通用二进制文件中的偏移
- size，内层二进制代码的大小
- align内存对齐，4K对齐。

##### 加载过程

当内核加载胖Maco-O的时候，仅仅会读取对应的架构，然后加载对应架构的二进制内容到内存。

事实上，内核加载二进制文件的第一个页面就可以读取文件头，而这个文件头实际上可以做一个目录，再根据这个目录继续加载合适的镜像。








