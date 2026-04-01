# dezoom

**dezoom** aims to make desktop zoom generally available in X11, Wayland
and desktop environments in general.

Zoom is an important accessibility tool for central vision loss and
other vision impairment conditions.

The set of tools described by **dezoom** tries to cover different use
cases by being comprehensive and configurable, balancing simplicity
when it is achievable without compromising accessibility requirements.

## Project scope

**dezoom** focus on the next general areas of interest.

- Feature set definition: trying to cover accessibility use cases as
completely as reasonably possible.
- Configuration and control protocols: providing integration with
accessibility minded applications and hardware.
- Reference implementations: providing code for the feature set in a way
that facilitates adoption by developers.

## Feature set

**dezoom** defines three kinds of zoom. Note that for multiple displays,
each display its considered independent.

* Panoramic zoom: zooms-in the entire display and allows panning across
the original content. It snaps to the edges of the display.
  * Cursor panning: The zoom center follows the cursor¹.
  * Absolute panning: Moves the zoom center explicitly, using keys o
  other input methods.
  * Edge panning: The cursor displaces the zoom center when its
  positioned against the borders of the display.

* Scope zoom: zooms-in a region of the display, taking the space around
it and following the cursor¹.

* Regional zoom: zooms specific regions in the display by taking the
space around them. The zooms in this regions can be enabled (expanded)
independently from each other:
  * Regions define two areas: the inner and the outer area. The inner
  area surrounds the content that needs the zoom and the outer area is
  the space where the zoomed content is going to be displayed.
  * Regions can be configured to expand depending on the cursor entering
  or leaving the inner and outer areas.
  * If the outer area cannot contain the zoomed content, its content
  will pan following the cursor.
  * Regions can also be ephemeral. In this case, they will self-destroy
  after the next event that contracts them.

¹ Control methods that involve following the cursor have the option to
stop following on demand.

The different zooms are combined in a reverse order.

First, the regional zooms are applied. If multiple regional zoomed areas
overlap, the one that was defined the last shows on top of the other
ones, replacing the content without extra amplification.

Then, the scope zoom is applied over the existing zoomed regions,
providing further amplification.

Finally, the panoramic zoom acts over the whole display.

## Control protocols

Waiting for reference implementations and user reports to validate data
model before designing the protocols.

Some details already established:

* Different regions are identified using a string.

* The protocol supports configuring the positions and scale factors
using values relative to the total display size.

### Focus zoom case

A special region, the "focus" region, can be used by applications and UI
toolkit libraries to zoom over widgets and other UI elements when they
have the interface focus or the user moves the cursor over them. This is
just a proposed name for the concept, the actual implementation will
use a regular ephemeral region, but the protocol will have into
consideration this use case.

### Protocol independent behaviors

Some implementations can add additional behaviors to **dezoom**, like
adapting cursor dynamics (speed, acceleration...) to the zoom
definitions. The protocol will allow for a simple way to control them,
but will not be strict about their definitions.

## Reference implementations

### Data model

```c
enum dezoompanmode {
  DEZOOM_CURSOR,
  DEZOOM_ABSOLUTE,
  DEZOOM_EDGE,
};

enum dezoomshape {
  DEZOOM_RECTANGLE,
  DEZOOM_ELLIPSE,
};

struct dezoomarea {
  unsigned float scale = 1.5;
  dezoomshape shape = DEZOOM_RECTANGLE;
  float x = 0.0; // original rectangle top-left position
  float y = 0.0;
  float w = 0.0; // original rectangle size
  float h = 0.0;
  float dx = 0.0; // destination rectangle top-left position
  float dy = 0.0;
};

// DEZOOM_ELLIPSE
// x, y    original ellipse center position from display top-left
// h, w    radius in pixels, as total% ex. 270 pixels for 1080p
// dx, dy  destination ellipse center position from display top-left

enum dezoomexpand {
  DEZOOM_NONE,
  DEZOOM_CREATION,
  DEZOOM_ORIGIN,
  DEZOOM_DEST,
  DEZOOM_ORIGIN_DEST, // enable on origin and keep until dest is left
};

struct dezoomregion {
  char *id;
  bool enabled = false;
  bool ephemeral = false;
  dezoomexpand expand = DEZOOM_ORIGIN;
  dezoomarea area;
  float dw = 0.0; // override for the destination area size
  float dh = 0.0;
};

struct dezoom {
  float pan_scale = 1.0;
  dezoompanmode pan_mode = DEZOOM_EDGE;
  float pan_x = 0.0; // zoomed view top-left corner position
  float pan_y = 0.0;

  bool scope_fixed = false;
  dezoomarea scope_area;

  dezoomregion *regions;
};
```
#### Shadertoy shape data example

Because code is better than words...

##### Rectangle

Note: In Shadertoy, (0,0) is bottom left.

```
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
  float scale = 1.5;
  float x = iMouse.x;
  float y = iMouse.y;
  float w = 200.0;
  float h = 150.0;
  float dx = 400.0;
  float dy = 300.0;

  vec2 srcPos  = vec2(x, y);
  vec2 srcSize = vec2(w, h);
  vec2 dstPos  = vec2(dx, dy);
  vec2 dstSize = srcSize * scale;

  vec2 local = fragCoord - dstPos;
  if (local.x >= 0.0 && local.y >= 0.0 &&
    local.x <= dstSize.x && local.y <= dstSize.y)
  {
    vec2 srcPixel = srcPos + (local / scale);
    vec2 srcUV = srcPixel / iResolution.xy;
    fragColor = texture(iChannel0, srcUV);
  } else {
    fragColor = texture(iChannel0, fragCoord / iResolution.xy);
  }
}
```

##### Ellipse

```
void mainImage(out vec4 fragColor, in vec2 fragCoord)
{
  float scale = 1.5;
  float x = iMouse.x;
  float y = iMouse.y;
  float w = 70.0;
  float h = 60.0;
  float dx = 200.0;
  float dy = 200.0;

  vec2 radius = vec2(w, h);
  vec2 mg_center = vec2(x, y);

  vec2 pixelCoord = fragCoord;
  vec2 uv = pixelCoord / iResolution.xy;

  vec2 d = pixelCoord - mg_center;
  float ellipseDist = dot(d * d, 1.0 / (radius * radius));
  if (ellipseDist < 1.0)
  {
    vec2 centered_coords = pixelCoord - mg_center;
    vec2 magnified_coords = centered_coords / scale;
    vec2 new_pixelCoord = magnified_coords + mg_center;
    vec2 new_uv = new_pixelCoord / iResolution.xy;
    fragColor = texture(iChannel0, new_uv);
  } else {
    fragColor = texture(iChannel0, uv);
  }
}
```

### labwc

Lightweight Wayland compositor ideal for experimentation.

### Proposed implementation path

- Data model:
  - Add data model.
  - Persist xml configuration in new data model on boot.
  - Replace existing data model for the current magnifier.
- Expand xml configuration to allow region configuration.
- Add regional zoom support.
- Add actions to modify the state.
- Test the implementation of a Wayland control protocol.

### picom

X11 compositor with support for shaders.

### Proposed implementation path

- Data model:
  - Add data model.
  - Persist configuration in new data model on boot.
- Expand configuration to allow region configuration.
- Build a shader to apply the zooms.
- Validate performance and coverage implications.
- Test the implementation of a protocol using a control socket.

## Origins and acknowledgments

This project merges the features of various applications I was using
before moving to Wayland. They worked fine in isolation but for complex
interfaces (some videogames and dense UIs like Godot's) I needed a more
integrated and comprehensive approach that was not available.

Panoramic zoom: MacOS zoom and
[boomer](https://github.com/tsoding/boomer).

Scope zoom: [xzoom](https://tracker.debian.org/pkg/xzoom) (or
[xzoom ftp](ftp://sunsite.unc.edu/pub/linux/libs/X/), and later
[kmag](https://apps.kde.org/en-gb/kmag/).

Many Wayland compositors and DE already offer some zoom capabilities,
but they are not as integrated and comprehensive as I need.
