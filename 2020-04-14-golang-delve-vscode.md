---
layout: post
date: 2020-04-14 01:00:00
author: iainnet
title: Golang Delve 调试工具和 VSCode 配置
---

[[blog]] [[golang]] [[vscode]]

## 介绍 Delve

Delve 是 go 语言的调试工具，允许我们给程序添加断点，查看运行时变量，提供线程和协程的状态等等。

### 安装

#### macOS

首先确保编译工具已经安装
`xcode-select --install`

使用 go get 安装 delve
`go get -u github.com/go-delve/delve/cmd/dlv`

开发者模式，可以避免每次都要输入密码来授权 delve
`sudo /usr/sbin/DevToolsSecurity -enable`

#### Linux

使用 go get 安装 delve
`go get -u github.com/go-delve/delve/cmd/dlv`

### 子命令（常用）

#### attach

语法： `dlv attach pid [executable]`

说明： 挂钩一个正在运行的进程开始调试。

一般可以用于常驻进程的调试，web服务、队列服务等等。挂钩上之后，会控制正在运行的进程，然后重启一个 debug session 。退出调式会话时，可以选择让程序继续运行或者直接杀死。

#### debug

语法： `dlv debug [package]`

说明： 编译当前目录下的 main 程序，启动，然后挂钩上去开启一个 debug 会话。

默认就是 ‘main’ 包，但是可以传入参数来指定要调试的包名。而且 debug 默认是禁用 optimizations（优化） 功能。

#### exec

语法： `dlv exec <path/to/binary>`

说明： 运行一个预先编译好的二进制程序，启动，然后挂钩上去开启一个 debug 会话。

如果是自己预先编译的话，注意带上编译参数：

go 1.10及其以上， -gcflags="all=-N -l"

go 1.10以前版本， -gcflags="-N -l"

这样才能禁用 optimizations（优化）功能。

之前测试效果，如果不加上面的编译参数，调试会有不少问题，比如看不到变量取值。

#### test

语法： `dlv test [package]`

说明： 编译测试用的二进制程序，并且关闭优化，开启一个 debug 会话。

### 调试命令(常用)

#### break

缩写： b

语法： `b [name] <linespec>`

设置断点

#### condition

语法： `condition <breakpoint name or id> <boolean expression>`

设置条件断点

#### continue

缩写： c

运行到下一个断点或者程序结束

#### next

缩写： n

跳到下一行

#### clear

语法： `clear <breakpoint name or id>`

#### step

缩写： s

进入调用的函数

#### step-instruction

缩写： si

进入当前cpu调用

#### stepout

缩写： so

跳出当前函数调用

#### restart

缩写： r

重启进程，重新开始调试。

#### config

可以查看、更改配置

`config <parameter> <value>`

比如修改，最大展示字符串长度： `config max-string-len 180`

## VSCode Debug 和 Delve 搭配

### 配置

#### 配置参数

```yaml
name: 名称，显示在 launch configuration 下拉面板
type: go
request: launch 、attach 两种。
  launch: 是直接加载启动二进制文件或者执行文件
  attach: 是挂钩一个正在运行的进程
mode: auto 、debug 、remote 、test 、exec 五种。
  auto: 是自动识别，判断到底是执行二进制文件，是直接执行文件，还是挂钩到正在运行进程id，
  debug: 等同于 dlv debug 。program字段要传入目录或者文件，会自动编译，然后启动一个 debug session 。
  remote: 等同于 dlv connect 。
  test: 等同于 dlv test 。program字段传入目录（test所在目录），然后启动一个 debug session 。
  exec: 等同于 dlv exec 。program字段传入已经编译好的可执行文件，然后启动一个 debug session 。
program: 不同 mode 下传入不同的数值。
env: 传入程序的环境变量
args: 传入程序的命令行变量，注意要按照空格切分的字符串数组。比如：-a /path/to/file -c 123 要写成 ["-a", "/path/to/file", "-c", 123]。
showLog: 展示日志，这个还是很有用的，出现执行错误，可以打开这个配置，可以方便查找问题。
```

#### 自动生成默认配置

在 debug 面板，点击创建 launch.json 。会在当前工程 .vscode 目录下，生成 launch.json 文件。默认内容如下：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${fileDirname}",
      "env": {},
      "args": []
    }
  ]
}
```

#### 增加调试测试用例的配置（dlv test）

```json
{
  "name": "Launch Test",
  "type": "go",
  "request": "launch",
  "mode": "test",
  "program": "${fileDirname}",
  "env": {},
  "args": []
}
```

${fileDirname} ： 当前打开文件所在目录。

所以上面配置执行起来就等同于在文件所在目录下执行： dlv test 。会开启 debug 会话，就可以加断点来调试测试用例。

 在 VSCode 中执行 `debug test` 的时候，可以切换到这个配置。

#### 增加关联执行进程的配置（dlv attach）

```json
{
  "name": "Attach to Process",
  "type": "go",
  "request": "attach",
  "mode": "local",
  "processId": 0
}
```

processId ： 传入当前正在运行的进程号，关联上去。

常用于调试常驻的服务进程。

#### 添加执行二进制文件的配置（dlv exec）

```json
{
  "name": "Launch Exec",
  "type": "go",
  "request": "launch",
  "mode": "exec",
  "program": "${workspaceFolder}/path/to/binaryfile",
  "env": {},
  "args": ["-c", "/path/to/file", "-a", "command"],
  "showLog": true
}
```

program ： 传入二进制文件

args ： 该命令执行的时候需要传入的命令行参数。

常用于调试脚本任务。上面的配置等同于： dlv exec path/to/binaryfile &#x2013; -c path/to/file -a command 。

### 基础用法

VSCode 启动 debug 会出现6个按钮，对应6个重要的操作。

![img](https://raw.githubusercontent.com/iAInNet/planeta/master/pics/golang_delve_vscode_1.png)

Play/Pause 按键： 相当于 dlv 中 continue 指令。跳出当前断点，执行代码，直到下一个断点停住。  
Forward 按键： 相当于 dlv 中的 next 指令。执行下一行，程序继续。  
Up 按键： 相当于 dlv 中的 step 指令。进入当前位置调用的函数。  
Down 按键：  相当于 dlv 中的 stepout 指令。跳出当前函数，回到之前进入的位置。  
Replay 按键： 相当于 dlv 中 restart 指令。 跳出所有断点，让程序从头开始执行。  
Stop 按键： 终止进程；如果是 attach 就是断开连接。  

## VSCode Debug 全局配置

在 settings.json 配置文件中，也有一些比较关键配置，能方便调试使用。比如增大字符串展示的长度。

```json
"go.delveConfig": {
        "dlvLoadConfig": {
            "maxStringLen": 512
        }
}
```

maxStringLen: 展示字符串的最大字节数。这个是比较常用的，避免出现错误信息显示不完全。

## 参考资料

[github地址](https://github.com/go-delve/delve)

[VSCode 配置变量](https://code.visualstudio.com/docs/editor/variables-reference)

[VSCode Debug 官方配置说明:](https://github.com/Microsoft/vscode-go/wiki/Debugging-Go-code-using-VS-Code)

[https://www.thegreatcodeadventure.com/debugging-a-go-web-app-with-vscode-and-delve/](https://www.thegreatcodeadventure.com/debugging-a-go-web-app-with-vscode-and-delve/)

[http://lallouslab.net/2018/07/02/3-steps-golang-debug-vscode-windows/](http://lallouslab.net/2018/07/02/3-steps-golang-debug-vscode-windows/)
