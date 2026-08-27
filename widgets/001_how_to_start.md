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

Why this approach? For starters, the underlying widget code doesn't need to assume layout, the layout pass is already completed at this point and all you're doing is literally *rendering* the thing you want to render at that position and size.

Most importantly, events are also typically handled by the layout engine as well. Most should provide utilities to determine which element is interacted with on any given frame.

Not all layout engines understand or have knowledge of "active" elements, so we may need to provide our own context for this. This is particularly true for text input where mouse input is used for auto-scrolling the widget when selecting outside the widget's bounds.

