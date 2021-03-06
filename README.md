# GO-NETTY

[![GoDoc][1]][2] [![license-Apache 2][3]][4] [![Go Report Card][5]][6] [![Build Status][9]][10] [![Coverage Status][11]][12]

<!--[![Downloads][7]][8]-->

[1]: https://godoc.org/github.com/go-netty/go-netty?status.svg
[2]: https://godoc.org/github.com/go-netty/go-netty
[3]: https://img.shields.io/badge/license-Apache%202-blue.svg
[4]: LICENSE
[5]: https://goreportcard.com/badge/github.com/go-netty/go-netty
[6]: https://goreportcard.com/report/github.com/go-netty/go-netty
[7]: https://img.shields.io/github/downloads/go-netty/go-netty/total.svg?maxAge=1800
[8]: https://github.com/go-netty/go-netty/releases
[9]: https://travis-ci.org/go-netty/go-netty.svg?branch=master
[10]: https://travis-ci.org/go-netty/go-netty
[11]: https://codecov.io/gh/go-netty/go-netty/branch/master/graph/badge.svg
[12]: https://codecov.io/gh/go-netty/go-netty

##
[中文介绍](./README-zh.md)

## Introduction

go-netty is heavily inspired by [netty](https://github.com/netty/netty)

## Feature

* Extensible transport support, default support TCP, [UDP, QUIC, KCP, Websocket](https://github.com/go-netty/go-netty-transport)
* Extensible codec support
* Based on responsibility chain model
* Zero-dependency

## Documentation
* [GoDoc](https://godoc.org/github.com/go-netty/go-netty)
* [Go-Netty.com](https://go-netty.com)

## Examples

* [chat_server](https://github.com/go-netty/go-netty-samples/blob/master/chat_server/main.go)  
* [file_server](https://github.com/go-netty/go-netty-samples/blob/master/file_server/main.go)  
* [tcp_server](https://github.com/go-netty/go-netty-samples/blob/master/tcp_server/main.go)  
* [redis_cli](https://github.com/go-netty/go-netty-samples/blob/master/redis_cli/main.go)
* [go-netty-samples](https://github.com/go-netty/go-netty-samples)  

## Quick Start
```go
package main

import (
	"fmt"
	"os"
	"strings"

	"github.com/go-netty/go-netty"
	"github.com/go-netty/go-netty/codec/format"
	"github.com/go-netty/go-netty/codec/frame"
	"github.com/go-netty/go-netty/transport/tcp"
)

func main() {

    // child pipeline initializer
    var childPipelineInitializer = func(channel netty.Channel) {
        channel.Pipeline().
            // the maximum allowable packet length is 128 bytes，use \n to splite, strip delimiter
            AddLast(frame.DelimiterCodec(128, "\n", true)).
            // convert to string
            AddLast(format.TextCodec()).
            // LoggerHandler, print connected/disconnected event and received messages
            AddLast(LoggerHandler{}).
            // UpperHandler (string to upper case)
            AddLast(UpperHandler{})
    }

    // new go-netty bootstrap
    netty.NewBootstrap().
        // configure the child pipeline initializer
        ChildInitializer(childPipelineInitializer).
        // configure the transport protocol
        Transport(tcp.New()).
        // configure the listening address
        Listen("0.0.0.0:9527").
        // waiting for exit signal
        Action(netty.WaitSignal(os.Interrupt)).
        // print exit message
        Action(func(bs netty.Bootstrap) {
            fmt.Println("server exited")
        })
}

type LoggerHandler struct {}

func (LoggerHandler) HandleActive(ctx netty.ActiveContext) {
    fmt.Println("go-netty:", "->", "active:", ctx.Channel().RemoteAddr())
    // write welcome message
    ctx.Write("Hello I'm " + "go-netty")
}

func (LoggerHandler) HandleRead(ctx netty.InboundContext, message netty.Message) {
    fmt.Println("go-netty:", "->", "handle read:", message)
    // leave it to the next handler(UpperHandler)
    ctx.HandleRead(message)
}

func (LoggerHandler) HandleInactive(ctx netty.InactiveContext, ex netty.Exception) {
    fmt.Println("go-netty:", "->", "inactive:", ctx.Channel().RemoteAddr(), ex)
    // disconnected，the default processing is to close the connection
    ctx.HandleInactive(ex)
}

type UpperHandler struct {}

func (UpperHandler) HandleRead(ctx netty.InboundContext, message netty.Message) {
    // text to upper case
    text := message.(string)
    upText := strings.ToUpper(text)
    // write the result to the client
    ctx.Write(text + " -> " + upText)
}
```

using <code>Netcat</code> to send message  
```bash
$ echo -n -e "Hello Go-Netty\nhttps://go-netty.com\n" | nc 127.0.0.1 9527
Hello I'm go-netty
Hello Go-Netty -> HELLO GO-NETTY
https://go-netty.com -> HTTPS://GO-NETTY.COM
```