---
title: "Creating a MPRIS server in Go"
date: 2023-03-22T17:48:47+01:00
---

# Why?

The use case of creating a MPRIS server is basically when you're making a music player and want to broadcast metadata to the system. Since I was creating a webradio player in Go, I needed this feature.

There are a lot of documentation online ([and even lib](https://github.com/leberKleber/go-mpris)) on how to create a MPRIS client to retrieve currently playing metadata but a lot less when it comes to creating an actual server. Since creating a MPRIS server is an absolute pain and is poorly documented I thought I might tell my journey of archiving it using the great godbus library.

# What is MPRIS

MPRIS is a D-Bus interface. That's when we need to explain what's a D-Bus interface. D-Bus is a protocol to communicate data between applications on Linux. It's main usage is on the Linux desktop ecosystem. If you need a comparison; if you wanted your web application to communicate with other services you would use an API and communicate with it using HTTP. On Linux, if you need your desktop applications to communicate with each others, you might use D-Bus. udev is a good example of thing that works using D-Bus; it allows communication with devices (block devices) in userspace.

A common use case is also mine, sharing metadata of currently playing song. The music player (Firefox, mpd, mpv, whatever) works as a D-Bus server that exports currently playing song metadata and the desktop environment act as a client that picks up this information to display it in the taskbar for example.

To communicate data using D-Bus you write and read data to something called a "message bus", we'll talk about this later on.

# Implementation

First of all, I owe great thanks to [mpd-mpris](https://github.com/natsukagami/mpd-mpris) for the reference implementation, without this project nothing would have been possible.
And of course, [godbus](https://github.com/godbus/dbus), without it, communicating with dbus in Go would be close to impossible. godbus exposes proper interfaces to make it easier to work with D-Bus and well... if you thought you could just go the hard way and implement D-Bus bindings yourself, the answer is basically: no you can't. I hope that by reading this article you'll get a glance of why implementing even the most basic thing that communicate with D-Bus is an absolute pain.

## Creating the D-Bus instance

The first thing to do is to create a D-Bus instance. By instance I mean something that will connect to a D-Bus... bus!

Our instance needs 2 things: some properties that we will make available to the bus and something to store the connection to the bus.

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

They are 2 types of message bus on D-Bus: the session bus, one for each user session and the system bus that is shared across the whole system. For MPRIS we want the session bus.

Here is how we connect to the session bus and store the connection in our instance:

```go
func main() {
  conn, err := dbus.ConnectSessionBus()
  if err != nil {
    log.Fatalln(err)
  }
  defer conn.Close()

  ins := &Instance{
    conn: conn,
  }
}
```

## Export props on the bus

Now, for MPRIS to actually detect our media player, there are a few things we need to export on the bus. The list is available on the official specification**s**.

MPRIS specifications is called a "D-Bus interface" and contain a list of properties and functions to implement. We have 2 interfaces to implement:

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

  ins.props, err = prop.Export(
    conn,
    "/org/mpris/MediaPlayer2",
    map[string]map[string]*prop.Prop{
      "org.mpris.MediaPlayer2": {"CanPlay": &canPlay}
    }
  )
}
```

`prop.Export` is a "magic" function of the godbus lib that will do the job of actually exporting the props to D-Bus.

It takes a connection (that we've setup earlier), the object path you want your export on and the list of properties to export.

The object path is the designated path for the interface we're implementing. Any application can declare a path that it communicate on. You can see it like an URL+endpoint of a web app. In the case of MPRIS the path they choose is `/org/mpris/MediaPlayer2`.

The list of properties is a `map[string]map[string]*prop.Prop` with the first level of the map being the name of the interface and the second one the name of the property.

Good, we've implemented one property, 18 more to go! 😂

## Taking shortcuts

When working with D-Bus, you'll quickly start to realize that most of the stuff is overengineer for your use case. That's when as a good lazy developer you'll start taking shortcuts. We need to implement 19 mandatory properties just for MPRIS to detect our player even if you just actually need one property for you use case.

A good shortcut I stole from [mpd-mpris](https://github.com/natsukagami/mpd-mpris) is the newProp function:

```go
func newProp(value interface{}, cb func(*prop.Change) *dbus.Error) *prop.Prop {
	return &prop.Prop{
		Value:    value,
		Writable: true,
		Emit:     prop.EmitTrue,
		Callback: cb,
	}
}
```

This will allow us to create a property with just the parameters that are actually useful: the value and the callback function.

Using the newProp function we can now implement the 19 properties:

```go
var player = map[string]*prop.Prop{
  "PlaybackStatus": newProp("Playing", nil),
  "Rate":           newProp(1.0, nil),
  "Metadata":       newProp(map[string]interface{}{}, nil),
  "Volume":         newProp(float64(100), nil),
  "Position":       newProp(int64(0), nil),
  "MinimumRate":    newProp(1.0, nil),
  "MaximumRate":    newProp(1.0, nil),
  "CanGoNext":      newProp(false, nil),
  "CanGoPrevious":  newProp(false, nil),
  "CanPlay":        newProp(true, nil),
  "CanPause":       newProp(true, nil),
  "CanSeek":        newProp(false, nil),
  "CanControl":     newProp(false, nil),
}

var mediaPlayer2 = map[string]*prop.Prop{
	"CanQuit":             newProp(false, nil),
	"CanRaise":            newProp(false, nil),
	"HasTrackList":        newProp(false, nil),
	"Identity":            newProp("myPlayer", nil),
	"SupportedUriSchemes": newProp([]string{}, nil),
	"SupportedMimeTypes":  newProp([]string{}, nil),
}

func main() {
  // Connect to the session bus

  ins.props, err = prop.Export(
    conn,
    "/org/mpris/MediaPlayer2",
    map[string]map[string]*prop.Prop{
      "org.mpris.MediaPlayer2":        mediaPlayer2,
      "org.mpris.MediaPlayer2.Player": playerProp,
    },
  )
}
```

All there is left to do now, is to request a name for our player (and check that it's available):

```go
main() {
  // Connect to the session bus
  // Export properties

  reply, err := conn.RequestName("org.mpris.MediaPlayer2.fip-player", dbus.NameFlagReplaceExisting)
  if err != nil {
    log.Fatalln(err)
  }
  if reply != dbus.RequestNameReplyPrimaryOwner {
    log.Fatalln("Name already taken")
  }

  select {}
}
```

With this, we should be able to detect our player using a MPRIS client.

[playerctl](https://github.com/altdesktop/playerctl) is the ideal candidate to test this:

```shell
$ playerctl -l
myPlayer
```

You can test this yourself using [my example](https://gist.github.com/DucNg/4b60437832f2d007324eba200599b212)!

## Implement and export functions on the bus

Now, that we have our D-Bus hello world, we can start actually implementing things.

We'll go with a basic use case: start and stop playback by clicking the button in your desktop environment / window manager (or using playerctl).

Let's start by creating a new type with a couple of public functions:

```go
type MediaPlayer2 struct{}

func (m *MediaPlayer2) Pause() *dbus.Error {
  // do something to pause playback
}

func (m *MediaPlayer2) Play() *dbus.Error {
  // do something to start playback
}
```

We can now export this type to D-Bus:

```go
func main() {
  // Connect to the session bus
  // Export properties

  mp2 := &MediaPlayer2{}

  err = conn.Export(mp2, "/org/mpris/MediaPlayer2", "org.mpris.MediaPlayer2.Player")
	if err != nil {
		log.Fatalln(err)
	}

  select{}
}
```

Again the `conn.Export` is a "magic" function of godbus that will export our interface to the message bus. When a call to the path is made, it will look for public functions of the same name in the interface and call it. Functions need to return `*dbus.Error` in order to be used by godbus.

You can test this using playerctl:

```shell
playerctl pause
playerctl play
```

Using this, something should happen on respective functions of your program.

## Updating the metadata

Something we might want to do now is to change the props we've set initially. To do this we can use the `props.Set()` function:

```go
var metadata = &map[string]interface{}{
	"mpris:trackid": "/org/mpris/MediaPlayer2/coolSong",
	"mpris:length":  0,

	"xesam:title": "my cool song",
}

func main() {
  // Connect to the session bus
  // Export properties

  dbusErr = ins.props.Set(
    "org.mpris.MediaPlayer2.Player",
    "Metadata",
    dbus.MakeVariant(metadata),
  )
	if dbusErr != nil {
		log.Println(dbusErr, metadata)
	})
}
```

Thanks to godbus we have an easy way to make a variant of our metadata which mandatory for the change to be effective.

To know which fields you can set on metadata and their type, you can refer to the [MPRIS metadata spec](https://www.freedesktop.org/wiki/Specifications/mpris-spec/metadata/)

# Conclusion

With this you should have a quick overview of how to make a MPRIS server and how to interact with it's main features.

If you want to go more in depth with a proper implementation, you can look at the source code of [mpd-mpris](https://github.com/natsukagami/mpd-mpris) or my [fip-player](https://github.com/DucNg/fip-player) using a mpv backend.
