---
title: xv6-环境搭建
date: 2025-01-04
categories:
  - OS
tags:
  - xv6
  - Note
  - CPP
---

## xv6 启动！

万事开头难！
![PixPin_2024-12-27_21-38-12.png|675](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2024-12-27_21-38-12.png)
这是我所用的开发环境，Kubuntu+VsCode，这样的环境对我来说是比较舒服的，也是比较好用的在实际使用的过程中我没感觉ubuntu和kubuntu有任何的区别，反正我也不在乎桌面，我只在终端里用。
用debian系的有一个好处就是软件生态真的很不错，可别是最新的，版本什么的直接用apt都能直接获得，虽然不想archwiki那样完善，但是Google一下基本上问题也都能解决。


### 一个很有意思的工具 bear
我相信有很多同学在用vscode看项目代码的时候会发现，全是红线，项目变得根本就不可读，无法跳转，更无法获得依赖关系，只能把vscode当一个能用鼠标的阉割版的vim用。我们可以获取编译的命令行，然后让vscode知道然后把这些报错给消掉
**1.make -nB**
当然可以手动获取编译选项，但是这样也是比较复杂的而且十分的低效
**2.bear**
善于使用工具，君子性非异也，善假于物也。用bear把编译过程包起来就能自动获取编译选项。

### 如何在vscode中debug


==官方非GUI界面== 

```c++
make qemu-gdb
```
之后系统会在本地启动一个gdb，另起一个终端使用gdb连接。
![PixPin_2025-01-04_16-46-16.png600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-01-04_16-46-16.png)
到这就说明gdb启动成功了，但是后面会有一个小小的坑。

Type "apropos word" to search for commands related to "word". warning: File "/home/learn_code/xv6-labs-2024/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load:/home/learn_code/xv6-riscv/.gdbinit". To enable execution of this file add add-auto-load-safe-path /home/learn_code/xv6-labs-2024/.gdbinit line to your configuration file "/root/.config/gdb/gdbinit". To completely disable this security protection add set auto-load safe-path / line to your configuration file "/root/.config/gdb/gdbinit".

当你启动gdb的时候可能会出现以上错误
这个按照上面说明的路径添加一个文件即可
![PixPin_2025-01-04_17-04-54.png600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-01-04_17-04-54.png)
如果使用gdb会有如上报错。
![PixPin_2025-01-04_17-06-02.png600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-01-04_17-06-02.png)
因为xv6的架构是risc-v的所以要是用这个支持多架构的gdb成功启动。

==配置vscode使用图形化界面==

lanch.json
```json
{

    "version": "0.2.0",

    "configurations": [

        {

            "name": "xv6debug",

            "type": "cppdbg",

            "request": "launch",

            "program": "${workspaceFolder}/kernel/kernel",

            "stopAtEntry": true,

            "cwd": "${workspaceFolder}",

            "miDebuggerServerAddress": "127.0.0.1:25000", //见.gdbinit 中 target remote xxxx:xx 一定要一 一对应

            "miDebuggerPath": "/usr/bin/gdb-multiarch", // which gdb-multiarch

            "MIMode": "gdb",

            "preLaunchTask": "xv6build"

        }

    ]

}
``` 

task.json
```json
// xv6-riscv/.vscode/tasks.json

{

    "version": "2.0.0",

    "tasks": [

        {

            "label": "xv6build",

            "type": "shell",

            "isBackground": true,

            "command": "make qemu-gdb",

            "problemMatcher": [

                {

                    "pattern": [

                        {

                            "regexp": ".",

                            "file": 1,

                            "location": 2,

                            "message": 3

                        }

                    ],

                    "background": {

                        "beginsPattern": ".*Now run 'gdb' in another window.",

                        // 要对应编译成功后,一句echo的内容. 此处对应 Makefile Line:170

                        "endsPattern": "."

                    }

                }

            ]

        }

    ]

}
```

```txt
set confirm off
set architecture riscv:rv64
@REM target remote 127.0.0.1:25000
symbol-file kernel/kernel
set disassemble-next-line auto
set riscv use-compressed-breakpoints yes
```
要将gitinit里面的端口注释掉不然会发生冲突。
![PixPin_2025-01-04_17-13-08.png600](https://weijiale.oss-cn-shanghai.aliyuncs.com/picgo/PixPin_2025-01-04_17-13-08.png)
然后直接f5就可以调试了。