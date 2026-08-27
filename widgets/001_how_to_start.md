# How to Start with Widgets
To begin with, I think it's worth discussing the basics of widget design, and we want to consider the data structure of each and how they should be formed in code.

Not all widgets will want the same shape, and it will ultimately depend on your needs.

For the most part, the immediate-mode (or IM) paradigm will fit most basic widgets. Things like buttons, checkboxes, radios, tabs, sliders, progress bars, and the likes, don't need structures. They can be simple values passed into a proc, like so:

```odin
if button("Click me!") {
    // do something on click
}

checked := &my_state.checked
if check_box("Is this checked?", checked) {
    // do something on check change (click)
}

if begin_tab_control() {
    // if this tab control is visible, render its content

    if begin_tab("Dashboard") {
        // if this tab is open and visible, render what's here

        end_tab()
    }

    end_tab_control()
}
```

This assumes automatic layout, so if incorporating in a layout system, you likely add a "rect" placement strategy. How would we do this?

An explicit parameter is likely most suitable:

```odin
if button("Click me!", &button_rect) {

}
```

You're probably wondering why I'm passing a rect as a pointer here. Invariably, you don't typically do this because the widget doesn't need to modify the rect itself especially if layout is already complete, though through habit I've come to accept that when working with animations and transitions, copying rectangles everywhere is not necessarily a good practice even if their size is negligible. Each animated widget will want the destination rect, not the rect it draws to, in order to control user input on mid-transitioning widgets.

Why this approach? For starters, the underlying widget code doesn't need to assume layout, the layout pass is already completed at this point and all you're doing is literally *rendering* the thing you want to render at that position and size.

Most importantly, events are also typically handled by the layout engine as well. Most should provide utilities to determine which element is interacted with on any given frame.

Not all layout engines understand or have knowledge of "active" elements, so we may need to provide our own context for this. This is particularly true for text input where mouse input is used for auto-scrolling the widget when selecting outside the widget's bounds.

Text input is a bit beyond the scope of this idea, but it does fall into the next strategy to discuss: retained state.

For state, a struct is likely necessary. How it's shaped and how it's allocated is up to the user, but a typical shape doesn't necessarily deviate from the original IM concept:

```odin
// init code
state.code_editor = init_code_editor()

// render
editor_rect := &state.code_editor_rect
code_editor_render(&state.code_editor, editor_rect)
```

The difference here is the separate initialisation. Build the initial struct, pass that into the render loop when you need it. For a code editor, especially, this would hold the text state, the render caching strategy, handle multi-threading (if necessary and via callbacks decoupled from the render loop), and whatever else you need. A good practice is to store internal state inside its own struct and document it clearly.

The final icing on the cake would be events and handling them. If you have your own custom layout library, then happy days. Modify it at will.

If you're using something like Clay with limited event exposure, it's possible you end up passing your own mouse, focus and key input handling state manually through the Clay pipeline. Because of the limited nature of Clay (by design), you may choose to preserve Clay IDs inside your own struct and handle focus independently.

This does mean some painful organisation, though, that's worth discussing in detail. I shall explore the Clay public API and write up one or more possible solutions to this.