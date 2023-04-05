---
title: "Creating a MPRIS server in Go"
date: 2023-03-22T17:48:47+01:00
draft: true
---

# Why?

The use case of creating a MPRIS server is basically when you're making a music player and want to broadcast metadata to the system. Since I was creating a webradio player in Go, I needed this feature.

There are a lot of documentation online ([and even lib](https://github.com/leberKleber/go-mpris)) on how to create a MPRIS client to retrieve currently playing metadata but a lot less when it comes to creating an actual server. Since creating a MPRIS server is an absolute pain and is poorly documented I thought I might tell my journey of archiving it using the great godbus library.

# What is MPRIS

MPRIS is a D-Bus interface. That's when we need to explain what's a D-Bus interface. D-Bus is a protocol to communicate data between applications on Linux. It's main usage on the Linux desktop ecosystem. If you need a comparison, if you wanted your web application to communicate with other services you would use an API and communicate with it using HTTP. On Linux, if you need your desktop applications to communicate with each others, you might use D-Bus. udev is a good example of thing that works using D-Bus; it allows communication with devices (block devices) in userspace.

A common use case is also mine, sharing metadata of currently playing song. The music player (Firefox, mpd, mpv, whatever) works as a D-Bus server that exports currently playing song metadata and the desktop environment act as a client that picks up this information to display it in the taskbar for example.

To communicate data using D-Bus you write and read data to something called a "message bus", we'll talk about this later on.

# Implementation

First of all, I owe great thanks to [mpd-mpris](https://github.com/natsukagami/mpd-mpris) for the reference implementation, without this project nothing would have been possible.
And of course, [godbus](https://github.com/godbus/dbus), without it, communicating with dbus in Go would be close to impossible. godbus exposes proper interfaces to make it easier to work with D-BUS and well... if you thought you could just go the hard way and implement D-BUS bindings yourself, the answer is basically: no you can't. You will understand by reading this article that implementing even the most basic thing that communicate with D-BUS is an absolute pain.

## Creating the D-BUS instance

The first thing to do is to create a D-BUS instance. By instance I mean something that will connect to a D-BUS... bus!

Our instance needs 2 things: some props that we will make available to the bus and something to store the connection to the bus.

godbus offers types to do this:

```go
import(
  "github.com/godbus/dbus/v5"
  "github.com/godbus/dbus/v5/prop"
)

type Instance struct {
	props *prop.Properties
	conn  *dbus.Conn
}
```

Once we have the instance we can connect to the session bus.

## Connect to the session bus

They are 2 types of message bus on D-Bus: the session bus, one for each user session and the system bus that is shared across the whole system.

Here is how we connect to the session bus and store the connection in our instance:

```go
func main() {
  conn, err := dbus.ConnectSessionBus()
  if err != nil {
    log.Fatalln(err)
  }

  ins := &Instance{
    conn: conn,
  }
}
```

## Export props on the bus

Now, for MPRIS to actually detect our media player, there are a few things we need to export on the bus. The list is available on the official specification**s**.

MPRIS specifications is called a "D-Bus interface" and contain a list of properties to implement. We have 2 interfaces to implement:

* [MediaPlayer2](https://specifications.freedesktop.org/mpris-spec/latest/Media_Player.html)
* [MediaPlayer2.Player](https://specifications.freedesktop.org/mpris-spec/latest/Player_Interface.html)

Giving a quick glance you'll see that almost all properties are mandatory.

A D-Bus property is composed of 4 things: a value, a function that will be call when the value is changed, a property to define if the value is writable or not and a property that tells how to emit `org.freedesktop.DBus.Properties.PropertiesChanged` when the value is changed.

Here is how to implement the simple property "[CanPlay](https://specifications.freedesktop.org/mpris-spec/latest/Player_Interface.html#Property:CanPlay)" of the MPRIS interface:

```go
var canPlay = prop.Prop{
  Value: true,
  Writable: true,
  Emit: prop.EmitTrue,
  Callback: nil,
}

func main() {
  // Connect to the session bus

  ins.props, err = prop.Export(conn, "/org/mpris/MediaPlayer2", map[string]map[string]*prop.Prop{
    "org.mpris.MediaPlayer2": {"CanPlay": &canPlay}
  })
}
```

prop.Export is a "magic" function of the godbus lib that will do the job of actually exporting the props to D-Bus.

It takes a connection (that we've setup earlier), the object path you want your export on and the list of properties to export.

The object path is the designated path for the interface we're implementing. Any application can declare a path that it communicate on. You can see it like an URL+path of a web app. In the case of MPRIS the path they choose is `/org/mpris/MediaPlayer2`.

The list of properties is a `map[string]map[string]*prop.Prop` with the first level of the map being the name of the interface and the second one the name of the property.

Good, we've implemented one property, 18 more to go! 😂

## Implement export functions on bus

Now, using this connection, what we want is to export some functions to be used by D-Bus and actually start implementing things.

We'll go with a basic use case: start and stop playback by clicking the button in your desktop environment / window manager.

Let's start by creating a new type with a few public functions:

```go
type MediaPlayer2 struct{}

func (m *MediaPlayer2) Pause() *dbus.Error {
  // do something to pause playback
}

func (m *MediaPlayer2) Play() *dbus.Error {
  // do something to start playback
}
```

