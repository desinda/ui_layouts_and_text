# UI Layouts and Text
This repository is (hopefully) designed to assist in helping people understand how we can choose to build user interfaces and what I think is missing in the ecosystem, not necessarily just in Odin but in programming in general.

Most UI libraries take one of two forms:

 * A batteries-included, widgets-included and internalised UI ecosystem and library, normally built as a framework and expanded to include non-UI related content.
 * Or, layout-based libraries, some including widgets (Dear ImGui) and some without (Clay).

A user typically wants one of the following solutions:

 * The ability to build user interfaces with a visual aid to allow building UI fast and efficiently, without needing to think about internals.
 * And/or the ability the build custom widgets without fighting the library they work with.

Layout libraries come closest to *both* solutions but extensions typically require extensive knowledge of the underlying library code. Most people prefer to skip the nuances of the limitations most UI libraries provide (Dear ImGui is considered the most complete) and choose, instead, to either build a generic layout library or use an existing one (Clay and others) if suitable.

Moreover, the biggest challenge that no one dares to touch is text. Text rendering for most is one of the most difficult challenges to overcome and typically involves atlas packing, eviction policies to minimize memory usage, while also keeping font rendering dynamic and minimise as many GPU uploads as possible.

The typical strategy is to rasterise font glyphs in advance and upload the glyphs into a texture that's blit to a specific region on-screen. This is your "traditional" caching strategy. Libraries like Harfbuzz are playing around with GPU-rendered text, which is the idea of rasterising text on the GPU (through a shader in their case) rather than on the CPU. Since text is mostly geometry, handing off this complex render math to the GPU makes more sense than trying to rasterise on the CPU. If you used Harfbuzz, you typically pair it with FreeType, as FreeType is the loader and rasteriser. Harfbuzz is generally used to form glyphs from font data and pass that back to FreeType for rendering.

With GPU accelerated glyph transforms, the possibility arises that FreeType rasterisation can be skipped and letting the GPU with access to 64-bit and 128-bit floating point math make glyph rasterisation far more accurate and sub-pixel rendering significantly more efficient than prior caching strategies.

I realise I'm ranting here about text rendering, but it is important as most UI relies on it.

Other libraries exist that are not so heavy handed, like `stbtt`, that are more light-weight but for multi-lingual text it's less accurate than the heavier Harfbuzz library.

Or, another alternative which I've been looking into is Blend2d's text rendering pipeline, which uses its own custom pipeline and it's surprisingly *efficient*. The only issue is that Blend2d is purely CPU-side. That said, combining it with something like Clay shouldn't be too difficult.

The strategy would be that your beautiful UI aesthetic could be built with Blend2D and uploaded as textures to the GPU. Once ready, those textures just get blitted by whatever library you use using a combination of 9-slice rects and glyph information to render the final UI.

Combining this with a layout library means that most of the UI is technically done for you; the next question arises is "widgets". Since layout doesn't resolve widgets for you, you're building them yourself.

Widgets is where the time consuming portion of the UI building comes into play. If you want text input, you're generally building this yourself, and text input is a time consuming task.

Data-demanding UI wants tabular or grid layouts, and database records in a view want tables. Tables could be plain simple, but if you're building a custom database app with custom filtering, SQL queries, embedded tables, footers for sums/averages/counts, etc., this kind of widget is easily a few weeks worth of work.

Widgets is a user-defined area, but most people just want the widget, not necessarily having to consider all the boilerplate in between. Do you also rely on the layout library for tables for these complex widgets or define custom rects within the widget's known rect?

The former gives you layout for free, though one could argue that pixel precision is more suitable for these kinds of complex widgets. If you want immaculate pixel-precise rendering for tabular data, which for end consumer facing apps, you probably want this. Dear ImGui and libraries like it have shown their limitations, and they're not designed for user facing apps. That doesn't mean they can't be used, but end users are sold on aesthetic, not functionality, most of the time.

If you're target audience is developers, you can probably get away with using the free layout of the library you're using. But if you're looking to make something beautiful, you're probably spending the extra week or so polishing your widget and making it pixel perfect.

Over the next few weeks/months, I will probably expand this repo to show what I may consider for "beautiful" UI.