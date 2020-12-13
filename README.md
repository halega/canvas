# canvas

This Go module uses WebSockets to communicate with a
[2D canvas graphics context](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D)
in a web browser.
It offers a portable way to create interactive 2D graphics from within
a Go program.

The Go program (server) sends draw commands to the web browser (client) via
WebSockets using a binary format.
The client in return sends keyboard and mouse events to the server.

The module does not require operating system specific backends or Cgo bindings.
It does not use WebAssembly, which means the Go code runs on the server side,
not in the browser.
The client-server design means the canvas can be displayed on a different
machine over the network.

The WebSocket communication imposes some overhead, but it is good enough for
many use cases. Different browsers also have different performance
characteristics.

## Examples

### Draw a static image

The `ListenAndServe` function initializes the canvas server and takes the
following arguments: the network address with the port number to bind to, a
run function, and zero or more options, such as the canvas size in pixels,
or a title for the browser tab.

The run function is executed when a client connects to the server.
This is the entry point for drawing.

```
package main

import (
	"image/color"

	"github.com/fzipp/canvas"
)

func main() {
	canvas.ListenAndServe(":8080", run,
		canvas.Size(100, 80),
		canvas.Title("Example 1: Drawing"),
	)
}

func run(ctx *canvas.Context) {
	ctx.SetFillStyle(color.RGBA{R: 200, A: 255})
	ctx.FillRect(10, 10, 50, 50)
	// ...
	ctx.Flush()
}
```

After the program has been started, the canvas can be accessed by
opening http://localhost:8080 in a web browser.

The server does not immediately send each drawing operation to the client,
but buffers them until the `Flush` method gets called.
The flush should happen once the image, or an animation frame is complete.
Without a flush nothing gets displayed.

Each client connection starts its own run function as a goroutine. Access to
shared state between client connections must be synchronized. If you do not
want to share state between connections you should keep it local to the run
function and pass the state to other functions called by the run function.

### An animation loop

You can create an animation by putting a `for` loop in the `run` function.
Within this loop the `ctx.Quit()` channel should be observed. 

```
package main

import (
	"time"

	"github.com/fzipp/canvas"
)

func main() {
	canvas.ListenAndServe(":8080", run,
		canvas.Size(800, 600),
		canvas.Title("Example 2: Animation"),
	)
}

func run(ctx *canvas.Context) {
	d := &Demo{}
	for {
		select {
		case <-ctx.Quit():
			return
		default:
			d.update()
			d.draw(ctx)
			ctx.Flush()
			time.Sleep(time.Second / 2)
		}
	}
}

type Demo struct {
	X, Y int
	// ...
}

func (d *Demo) update() {
	d.X += 1
	d.Y += 1
	// ...
}

func (d *Demo) draw(ctx *canvas.Context) {
	// ...
}
```

### Keyboard and mouse events

In order to handle keyboard and mouse events you have to specify which events
the client should observe and send to the server.
This is done by passing an `EnableEvents` option to the `ListenAndServe`
function. 
Mouse move events typically create more WebSocket communication than the
others.
So you may want to enable them only if you actually use them.

The `ctx.Events()` channel receives the observed events, and a type switch
determines the specific event type.
A useful pattern is a `handle` method dedicated to event handling:

```
package main

import "github.com/fzipp/canvas"

func main() {
	canvas.ListenAndServe(":8080", run,
		canvas.Size(800, 600),
		canvas.Title("Example 3: Events"),
		canvas.EnableEvents(
			canvas.SendMouseDown,
			canvas.SendMouseMove,
			canvas.SendKeyDown,
		),
	)
}

func run(ctx *canvas.Context) {
	d := &Demo{}
	for {
		d.update()
		d.draw(ctx)
		ctx.Flush()
		select {
		case <-ctx.Quit():
			return
		case event := <-ctx.Events():
			d.handle(event)
		}
	}
}

type Demo struct {
	// ...
}

func (d *Demo) handle(event canvas.Event) {
	switch e := event.(type) {
	case canvas.MouseDownEvent:
		// ...
	case canvas.MouseMoveEvent:
		// ...
   	case canvas.KeyDownEvent:
		// ...
	}
}

func (d *Demo) update() {
	// ...
}

func (d *Demo) draw(ctx *canvas.Context) {
	// ...
}
```

## Alternatives

If this module is not the right thing for you, and if you are looking for a
more direct solution for 2D graphics have a look at these alternatives:

* [github.com/tfriedel6/canvas](https://github.com/tfriedel6/canvas) -
  A canvas implementation for Go with OpenGL backends for various
  operating systems.
* [github.com/llgcode/draw2d](https://github.com/llgcode/draw2d) -
  A 2D vector graphics library for Go with support for multiple outputs
  such as images, PDF documents, OpenGL and SVG.
* [github.com/ajstarks/svgo](https://github.com/ajstarks/svgo) -
  A Go library for SVG generation.
* [github.com/tdewolff/canvas](https://github.com/tdewolff/canvas) -
  A common vector drawing target that can output SVG, PDF, EPS,
  raster images (PNG, JPG, GIF, ...), HTML Canvas through WASM, and OpenGL.

2D game engines:

* [github.com/faiface/pixel](https://github.com/faiface/pixel) - Pixel
* [github.com/hajimehoshi/ebiten](https://github.com/hajimehoshi/ebiten) - Ebiten
* [github.com/oakmound/oak](https://github.com/oakmound/oak) - Oak

## License

This project is free and open source software licensed under the
[BSD 3-Clause License](LICENSE).
