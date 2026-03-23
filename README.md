# wlzoom

**wlzoom** aims to make desktop zoom generally available in Wayland
compositors and desktop environments.

Zoom is an important accessibility tool for central vision loss and
other vision impairment conditions.

The set of tools described by **wlzoom** tries to cover different use
cases by being comprehensive and configurable, balancing simplicity
when it is achievable without compromising accessibility requirements.

## Project scope

**wlzoom** focus on the next general areas of interest.

- Feature set definition: trying to cover accessibility use cases as
completely as reasonably possible.
- Reference implementations: providing code for the feature set in a way
that facilitates adoption by compositor and DE developers.
- Configuration and control protocols: providing integration with
accessibility minded applications and drivers observing the Wayland
security design environment.

## Feature set definition

**wlzoom** defines three kinds of zoom. Note that for multiple displays,
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

* Regional zoom: zooms specific regions of the display, by taking the
space around them. They can be enabled independently. They can also be
enabled by checking if the cursor position collides with either their
original or their zoomed area, and disabled when it leaves one of them.

¹ Control methods that involve following the cursor have the option to
stop following on demand.

## Wayland control protocol

Waiting for reference implementations and user reports to validate data
model. Some details are already established:

* Pixel sizes are defined as floats. Values between 0 and 1 are treated
as a percentage of the display were the zoom is defined.

* Regions can be identified using a string.

## Reference implementations

### labwc

#### Data model

```c
enum wlpanmode {
  WLZ_CURSOR,
  WLZ_ABSOLUTE,
  WLZ_EDGE,
};

enum wlzoomshape {
  WLZ_RECTANGLE,
  WLZ_ELLIPSE,
};

struct wlzoomarea {
  unsigned float scale = 1.5;
  wlzoomshape shape = WLZ_RECTANGLE;
  float x = 0.0; // original rectangle top-left position
  float y = 0.0;
  float w = 0.0; // original rectangle size
  float h = 0.0;
  float dx = 0.0; // destination rectangle top-left position
  float dy = 0.0;
};

// WLZ_ELLIPSE
// x, y    original ellipse center position from display top-left
// h, w    radius in pixels, as total% ex. 270 pixels for 1080p
// dx, dy  destination ellipse center position from display top-left

enum wlzoomexpand {
  WLZ_NONE,
  WLZ_ORIGIN,
  WLZ_DEST,
  WLZ_ORIGIN_DEST, // enable on origin and keep until dest is left
};

struct wlzoomregion {
  char *id;
  wlzoomexpand expand = WLZ_ORIGIN;
  wlzoomarea area;
};

struct wlzoom {
  float pan_scale = 1.0;
  wlpanmode pan_mode = WLZ_EDGE;
  float pan_x = 0.0; // zoomed view top-left corner position
  float pan_y = 0.0;

  bool scope_fixed = false;
  wlzoomarea scope_area;

  wlzoomregion *regions;
};
```
#### Proposed implementation path

- Data model:
  - Add data model.
  - Persist xml configuration in new data model on boot.
  - Replace existing data model for the current magnifier.
- Expand xml configuration to allow region configuration.
- Add regional zoom support.
- Add actions to modify the state.

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
