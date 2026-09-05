# Incorporating Clay
If we were to incorporate Clay as an example, we would need to know what API surface it exposes. As this is an example, use this as more of a guide rather than a "how to". This is not specifically designed for use with Clay, it is a "guide" in widget design for UI development, so don't think that Clay is the only layout library that exists, I'm just using it for reference as it is the most popular.

Let's begin by studying what exists in Clay for use.

The most notable procedures that exist appear to be the following:

```odin
clay.GetElementId(idString : clay.String) -> clay.ElementId
clay.GetElementData(id : clay.ElementId) -> clay.ElementData
```

Clay's own documentation is a fantastic resource for all things related to [Clay](https://github.com/nicbarker/clay), so I'm not going to reinvent how to use the procedures, but I will describe what we could do with these options for custom widget design that doesn't rely on Clay's *specific* API, notably text, text input and custom rendering, possibly with pre-baked Blend2D primitives.

Let's discuss how we might build a Numerical widget with Up/Down control buttons. We need some states and a structure for this widget:

```odin
Widget :: struct {
    base_id : clay.ElementId,

    variant : union{
        ^NumericUpDown,
    }
}

NumericUpDown :: struct {
    using base : Widget,

    input_bounds_id : clay.ElementId,
    up_button_id : clay.ElementId,
    down_button_id : clay.ElementId,

    up_pressed : bool,
    down_pressed : bool,
}
```

This is a complex enough example to need some structure.

This is divided into three individual components within the area of the widget:

 * Area for text input if we desire it
 * Two areas for buttons for Up and Down.

The `base_id` is used within a base structure, `Widget`, to be stored in an array to iterate and determine event validity at the beginning of the next frame, after events have been passed. We do this because in most IM paradigms, there is a small one-frame delay between when widgets are constructed and when they receive user input.

We could clear this list before each call to `clay.BeginLayout` if we desire, or preserve it for retained state. In most cases, it's probably the former.

One important caveat with Clay is that it doesn't have a mechanism or facility for *focused* elements. To build a state incorporating this, we may want to include one:

```odin
WidgetState :: struct {
    widgets : [dynamic]Widget,
    focused_index : int,
}
```

We could determine the last focused element by checking where the mouse was last *pressed down* (NOT clicked). Down is the activator for focus change, not a click, in most modern UI libraries.

Then, when checking the widgets *before* `BeginLayout`, after the first frame:

```odin
for &w in widget_state.widgets {
    #partial switch &v in w.variant {
        case NumericUpDown:
            v.up_pressed = false
            v.down_pressed = false

            if clay.PointerOver(v.up_button_id) && rl.IsMouseButtonDown(.LEFT) {
                v.up_pressed = true
            }

            if clay.PointerOver(v.down_button_id) && rl.IsMouseButtonDown(.LEFT) {
                v.down_pressed = true
            }
    }
}
```

Unsurprisingly, widget management with Clay is not the easiest. Layout & widgets decoupled makes codebase architecture naturally more complex than it should be, and is where retained UIs get some benefit from a layout/widget coupling.

Although the above code example is a simple example, it hopefully presents the kind of work necessary to build "widgets" with a layout system. The idea is that your widgets are stored independently from the layout system and manage it separately.

You will notice I use Raylib events directly here. Clay is useful for layouts and keeping pointer states within the realms of Clay is the correct way to determine which element is hovered, but from then on events should pass back over to the user.

You will wonder why I have architected the above the way I have, so let me explain:

 * Any widget that's designed to modify user state needs an internal architecture that allows modifying that state. In the above example, we separate user state (min, max, value) from the components of the widget so that the following frame knows what is "pressed". 
 * When the frame's layout measures *this* widget, check `up_pressed` and `down_pressed`, and change the state accordingly.

Here is an example of how we might construct this widget at the call-site during clay layout (this assumes a global widget state of the variable `g_widgets`):

```odin
numeric :: proc(id : string, min, max : int, value : ^int) {
    clay_id := clay.ID(id)
    widget : ^Widget
    if index, ok := has_widget(&g_widgets, clay_id); ok { // checking `base_id`
        widget = &g_widgets.widgets[index]
    }
    else {
        widget = create_widget(&g_widgets, clay_id, NumericUpDown{}) // create the widget if it doesn't exist
    }

    if clay.UI(clay_id)({
        layout= {
            sizing= {
                width= clay.SizingFit({ min= 80, max= 150 }),
                height= clay.SizingFixed(30), // should be font size + padding, fixed for simplicity
            },
            childAlignment= {
                x= .Right,
                y= .Top
            }
        }
    }) {
        // ... the rest of the widget construction
    }
}
```

This relies on Clay where every part of the widget is designed from Clay's own layout system. There are pros and cons to this, so let's discuss:

 * By using Clay directly, you use Clay's layout engine and mouse pointer events, so Clay knows what part of the layout is being hovered over, etc. and you can just query it.
 * By contrast, having Clay do all the mouse pointer work means less control over events and requires user-defined solutions (like the `focused_index` field) to actually build interactive widgets that require complex interactivity.

In my opinion, widgets and the widget loop itself should be its own event pass. It's additional work for the trade of a less ergonomic pattern. The result, however, is that each widget can perform its own layout pass not heavily reliant on Clay or any event system found in a Layout library. It also means layout libraries that do *not* supply event systems of their own can also be used, and you can iterate across the layout library and check where your mouse is.

In Clay, there is one more caveat to draw on, and that's its draw command list system. This uses `rawptr` and can be type matched in a switch statement with an enum whose backing type is `uintptr` and custom rendering performed accordingly. This I say is a caveat because the event ergonomics you *actually* want per-widget is no longer in Clay territory, so you need separate tracking as well, not just references to the bounding box of each widget on the screen. That means manually tracking if the mouse is currently held down on the widget, whether the mouse remains down as it moves outside the bounding box, feeding text input characters, and other UI-related input.

Another question that this raises is "drag and drop". Most layout libraries do not provide a "drag-and-drop" API, so once again, you're on your own. The work involved in building a layout library is "just for layout", yet libraries like Clay go beyond what is necessary, and in the real world where widgets need to change their internal state according to user input, Clay performing the update for you is no longer beneficial and a waste of computation power.

A better approach to the widget system is to pass the bounding box *to the widget*:

```odin
numeric :: proc(id : Id, min, max : int, value : ^int, rect : ^Rect) {
    // code for numeric
}
```

The procedure simply handles constructing the widget at the call-site and adding it to a temporary list, freed at the end of each frame. Each widget knows it's rect (a pointer in this case - see [How to Start](001_how_to_start.md) for why) and rendering picks up what was constructed as *widgets* rather than pure layout.

It is an extra pass in the frame loop:

 1. New Frame -> Input Polling
 2. Layout
 3. Widgets (*new*)
 4. Render (*not commands, the widgets themselves*)

Widgets hold their rendering state, not the layout. Layout is "just layout", after all.

Most Layout libraries delay things for one frame, though, so to minimise the risk of complexity, we could change the order slightly:

 1. Layout
 2. New Frame -> Input Polling
 3. Widgets (input polled, so can perform event checks here)
 4. Render

This means no frame delays for most cases, particularly input, and the rendered state is whatever is the current widget state on the current frame.

In most IM-style libraries, layout is delayed mainly because events can change layout, so layout *after* input polling rather than *before*, and poll against the last frame's layout state. Clay's [own documentation](https://github.com/nicbarker/clay/tree/main#mouse-touch-and-pointer-interactions) recommends laying out twice in the following:

> Note that the bounding box queried by Clay_PointerOver is from the last frame. This generally shouldn't make a difference except in the case of animations that move at high speed. If this is an issue for you, performing layout twice per frame with the same data will give you the correct interaction the second time.

This complexity is a shortcoming of the IM paradigm, whereas traditional retained-mode UI doesn't have this issue. The argument said of this is that computers are fast and the one frame delay is a fair trade.

Where layout and widgets are coupled like in the Dear ImGui case, this argument holds. Where layout and widgets are decoupled, the argument becomes harder to justify since the complexity introduced by needing to define your own widgets makes the one frame delay more difficult to rationalise. The reordering above makes the "define your widgets" motive more ergonomically friendly and easier to maintain over time.

In the next article, I will discuss how one could define their own widgets, taking from a recent post I made on the Odin forums where this very repo is illustrated.