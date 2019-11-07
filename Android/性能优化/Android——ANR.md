[toc]

# 概述
ANR（Application Not Responding）是指应用无响应。

Android系统对于一些事件都在在一定时间内完成，如果超过预订时间没有得到相应就会在成ANR。

ANR机制是对应用程序主线程的限制，要求主线程在限定的时间内处理完一些最常见的操作(启动服务、处理广播、处理输入)， 如果处理超时，则认为主线程已经失去了响应其他操作的能力。

# 场景
导致ANR的场景主要有以下：
-  Service Timeout(20 seconds) —— Service在特定的时间内无法处理完成
- Broadcast Timeout(10 seconds) ——BroadcastReceiver在特定时间内无法处理完成
- ContentProvider Timeout——内容提供者执行超时
- KeyDispatch Timeout(5 seconds) ——主要类型按键或触摸事件在特定时间内无响应


# 如何避免
将所有耗时操作，比如访问网络，Socket 通信，查询大量SQL 语句，复杂逻辑计算等都放在子线程中去，然后通过handler.sendMessage、runonUITread、AsyncTask 等方式更新UI，以确保用户界面操作的流畅度。

UI线程尽量只做跟UI相关的工作。

用Handler来处理UIThread和别的Thread之间的交互。