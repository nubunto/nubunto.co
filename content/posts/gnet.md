+++
title = "About gnet"
date = "2022-06-06T13:00:00-03:00"
author = "Bruno Panuto"
authorTwitter = "panuto_" #do not include @
tags = []
keywords = []
description = "Fast and event-loop based? interesting..."
showFullContent = false
+++

Gnet advertises itself as a "fast and lightweight networking framework". By looking briefly at the source code, this seems to hold true. Gnet's responsibilities are:

- implementing an efficient networking connection poller based on epoll or kqueue
- implementing an event loop mechanism on top of this connection polling

Gnet does not offer parsers for protocols such as HTTP or websockets. As such, it is indeed, lightweight: the poller and the event loop is hidden behind a very simple interface called `gnet.EventHandler`. It's basically a bag of lifecycles defined as callbacks, which might be a weird idiom in Go but has it's place here.

## the websocket example, reworded

gnet has a [websocket example](https://github.com/gnet-io/gnet-examples/blob/v2/websocket/server/websocket.go). We'll be implementing this example, although changing a few things that the author judges necessary. Mostly personal preferences though!

We'll start in a single file called `websocket.go` that will be a main package. Let's define `wsServer`, which will be our websocket server:

```go
type wsServer struct {
	gnet.BuiltinEventEngine

	addr                      string
	atomicNumberOfConnections int64
}
```

Notice the embedding of `gnet.BuiltinEventEngine`. The gnet terminology might be a bit confusing, so let's unpack it.

`gnet.BuiltinEventEngine` is simply a default implementation of `gnet.EventHandler`. When you embed it, our server can now be used as a `gnet.EventHandler`, without us having to implement every single callback defined by the `gnet.EventHandler` interface.

These are the callbacks: 

```go
// OnBoot fires when the engine is ready for accepting connections.
// The parameter engine has information and various utilities.
OnBoot(eng Engine) (action Action)

// OnShutdown fires when the engine is being shut down, it is called right after
// all event-loops and connections are closed.
OnShutdown(eng Engine)

// OnOpen fires when a new connection has been opened.
//
// The Conn c has information about the connection such as its local and remote addresses.
// The parameter out is the return value which is going to be sent back to the peer.
// Sending large amounts of data back to the peer in OnOpen is usually not recommended.
OnOpen(c Conn) (out []byte, action Action)

// OnClose fires when a connection has been closed.
// The parameter err is the last known connection error.
OnClose(c Conn, err error) (action Action)

// OnTraffic fires when a socket receives data from the peer.
//
// Note that the []byte returned from Conn.Peek(int)/Conn.Next(int) is not allowed to be passed to a new goroutine,
// as this []byte will be reused within event-loop after OnTraffic() returns.
// If you have to use this []byte in a new goroutine, you should either make a copy of it or call Conn.Read([]byte)
// to read data into your own []byte, then pass the new []byte to the new goroutine.
OnTraffic(c Conn) (action Action)

// OnTick fires immediately after the engine starts and will fire again
// following the duration specified by the delay return value.
OnTick() (delay time.Duration, action Action)
```

So to recap: `gnet.Engine` is the actual mechanism that polls the network, and `gnet.EventHandler` is the type that contains the callbacks that are called whenever a certain event happens. You can think of them being analogous to `http.Server` and `http.Handler`. Not 100% accurate but it's a good enough comparison.

We're going to implement `OnBoot` just to log a message when the engine is ready:

```go
func (wss *wsServer) OnBoot(eng gnet.Engine) gnet.Action {
	logging.Infof("echo server with multi-core=true is listening on %s", wss.addr)

	return gnet.None
}
```

Next callbacks to implement are `OnOpen` and `OnClose`, because we need to track connections:

```go
func (wss *wsServer) OnOpen(conn gnet.Conn) ([]byte, gnet.Action) {
	conn.SetContext(new(wsCodec))

	atomic.AddInt64(&wss.atomicNumberOfConnections, 1)

	return nil, gnet.None
}

func (wss *wsServer) OnClose(conn gnet.Conn, err error) gnet.Action {
	if err != nil {
		logging.Warnf("error occurred on connection=%s, %v\n", conn.RemoteAddr().String(), err)
	}

	atomic.AddInt64(&wss.atomicNumberOfConnections, -1)
	logging.Infof("conn[%v] disconnected", conn.RemoteAddr().String())

	return gnet.None
}
```

And now, the heart of everything: `OnTraffic`. This callback is called every time there's new data in a connection. This is where we're doing the "echo" part of our server:

```go
func (wss *wsServer) OnTraffic(conn gnet.Conn) gnet.Action {
	codec, ok := conn.Context().(*wsCodec)
	if !ok {
		logging.Errorf("unexpected context type, shutting down connection")

		return gnet.Close
	}

	if !codec.upgradedWebsocketConnection {
		logging.Infof("conn[%v] upgrade websocket protocol", conn.RemoteAddr().String())

		_, err := ws.Upgrade(conn)
		if err != nil {
			logging.Warnf("conn[%v] [err=%v]", conn.RemoteAddr().String(), err.Error())

			return gnet.Close
		}

		codec.upgradedWebsocketConnection = true

		return gnet.None
	}

	msg, op, err := wsutil.ReadClientData(conn)
	if err != nil {
		if _, ok := err.(wsutil.ClosedError); !ok {
			logging.Warnf("conn[%v] [err=%v]", conn.RemoteAddr().String(), err.Error())
		}

		return gnet.Close
	}

	logging.Infof("conn[%v] receive [op=%v] [msg=%v]", conn.RemoteAddr().String(), op, string(msg))

	err = wsutil.WriteServerMessage(conn, op, msg)

    if err != nil {
		logging.Warnf("conn[%v] [err=%v]", conn.RemoteAddr().String(), err.Error())

		return gnet.Close
	}

	return gnet.None
}
```

There's just one final callback, implemented by the example, that is at least interesting: `OnTick`. OnTick fires once after the engine boots, and then fires at the specified interval. We're going to leverage it to print the number of connections to our websocket server:

```go
func (wss *wsServer) OnTick() (time.Duration, gnet.Action) {
	logging.Infof("[connected-count=%v]", atomic.LoadInt64(&wss.atomicNumberOfConnections))

	return 3 * time.Second, gnet.None
}
```

Let's wrap everything in a `main` function now!

```go
func main() {
	var port int

	flag.IntVar(&port, "port", 9000, "server port")
	flag.Parse()

	wss := &wsServer{
		addr: fmt.Sprintf("tcp://0.0.0.0:%d", port),
	}

	log.Println(
		"server exits:",
		gnet.Run(
			wss,
			wss.addr,
			gnet.WithMulticore(true),
			gnet.WithReusePort(true),
			gnet.WithTicker(true),
		),
	)
}
```

I like this example since it shows a non-trivial use of gnet, while being relatively simple. But let's take it a step further: what if we could broadcast messages across all connected clients?

## let's broadcast!

Let's create a new type that will hold a reference to connections. Let's give it two methods: one for tracking a connection, and one for untracking a connection. When a connection is open, we want to track it; when it is closed, we want to untrack it. We'll also have a method that will iterate over all connections and write a message to them.

We'll call it `broadcastService`:

```go
type broadcastService struct {
	connections map[gnet.Conn]struct{}
}
```

Now, we need a way to track and untrack connections:

```go
func (b *broadcastService) trackConnection(c gnet.Conn) {
	b.connections[c] = struct{}{}
}

func (b *broadcastService) untrackConnection(c gnet.Conn) {
	delete(b.connections, c)
}
```

And finally, we need a way to broadcast messages to tracked connections:

```go
func (b *broadcastService) broadcastMessage(op ws.OpCode, msg []byte) error {
	for c, _ := range b.connections {
		err := wsutil.WriteServerMessage(c, op, msg)
		if err != nil {
			return fmt.Errorf("writing server message: %w", err)
		}
	}
	return nil
}
```

We'll need `wsServer` to hold a reference to our `broadcastService`:

```go
type wsServer struct {
	gnet.BuiltinEventEngine

	addr                      string
	atomicNumberOfConnections int64

	bs *broadcastService // added!
}
```

Now, let's go back to our `OnOpen` and `OnClose` callbacks and track the connection accordingly:

```go
func (wss *wsServer) OnOpen(conn gnet.Conn) ([]byte, gnet.Action) {
	conn.SetContext(new(wsCodec))

	atomic.AddInt64(&wss.atomicNumberOfConnections, 1)

	wss.bs.trackConnection(conn)

	return nil, gnet.None
}

func (wss *wsServer) OnClose(conn gnet.Conn, err error) gnet.Action {
	if err != nil {
		logging.Warnf("error occurred on connection=%s, %v\n", conn.RemoteAddr().String(), err)
	}

	atomic.AddInt64(&wss.atomicNumberOfConnections, -1)
	logging.Infof("conn[%v] disconnected", conn.RemoteAddr().String())

	wss.bs.untrackConnection(conn)

	return gnet.None
}
```

We'll use the `OnTraffic` callback to broadcast messages across all connected clients:

```go
// inside `OnTraffic`, change:
err = wsutil.WriteServerMessage(conn, op, msg)
// to:
err = wss.bs.broadcastMessage(op, msg)
```

Finally, let's not forget to pass our `broadcastService` when instantiating `wsServer`:

```go
// inside main():
bs := &broadcastService{connections: make(map[gnet.Conn]struct{})}
wss := &wsServer{
	addr: fmt.Sprintf("tcp://0.0.0.0:%d", port),
    bs: bs,
}
```

Now, use `wscat` to connect multiple clients to your running program and see messages sent from one client broadcasted to every other client, _including_ yourself!

## differences from net/http

The basic difference is in two places: the way IO is handled, and the way concurrent connections are handled.

net/http servers tend to spawn one goroutine per HTTP request. In Go, given the nature of goroutines and the way they are multiplexed on top of threads by the runtime scheduler, this tends to work well by default. However, at a very large scale, it tends to be a bottleneck since goroutines start with a 2kb stack.

gnet however, takes a different approach. Instead of spawning a goroutine per request, gnet instantiates a poller: either epoll or kqueue depending on your OS (linux or bsd respectively). Then, instead of reading from a connection on a goroutine, gnet just delegates this work to the OS, who will notify when a given connection sends data on the wire.

This is why blocking in net/http is OK, while blocking in gnet is a crime. When you block in net/http, you are blocking an independent unit of computation. No other request will be impacted. In gnet, you are blocking the event loop. No other request can be served. This is analogous to how nodejs deals with concurrency.

If you want to see this in action, try putting a 10s delay in either `OnTraffic` or `broadcastService` with something like `<-time.After(10*time.Second)`. See that nothing else can happen, including `OnTick`.

## where is it useful

gnet is useful for applications where the network might be a bottleneck, or where you need to extract the most out of each concurrent connection. Real time applications that need a low memory footprint might benefit more from gnet than other types of web apps.

If you need to serve hundreds of thousands or millions of request with commodity hardware, and you need to do so with as little latency as possible, then gnet is for you. If you don't need it, though, you don't need it: gnet is a networking framework, not an HTTP or application framework. Of course, you can build HTTP or web apps on top of it, but the effort to do so is still necessary, whereas using e.g. net/http would allow one to move faster for certain kinds of applications, particularly HTTP, since it's a battle tested, stable and spec-compliant.

That being said, I feel gnet shines brighter when paired with Redis. I'm planning to start a low latency, highly scalable pub/sub broker with a websocket interface leveraging Redis PubSub and gnet's networking capabilities and see how it fares versus other alternatives, such as Elixir's Phoenix or Erlang's EMQ.

Since gnet is event-loop based, I wonder if a technology like LiveView built on top of these would be scalable as well. Just thinking out loud, though.