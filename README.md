# wlzoom

A comprehensive set of tools to magnify areas in a display. Mainly
intended as an accessibility tool for central vision loss and other
vision impairment conditions.

## Scope

wlzoom supports three kinds of zooms:

* Panoramic zoom: zooms-in the entire screen and allows panning across
the original content.
* Scope zoom: zooms-in a region of the screen, taking the space around
it and following the cursor.
* Regional zoom: zooms specific regions of the screen, magnifying them
by taking the space around them.

Scope zoom can be made "sticky" to fix its position, allowing cursor
movement without moving the scoped area.

Regions can be expanded automatically when the cursor enters the
original or zoomed area.

## Data model

wlzoom supports dynamic alterations of the properties of this zooms. A
protocol will support modifications of this properties to allow other
clients to implement vision related accessibility functionality.

Reference of the internal data structure:

```c
enum wlzoomshape {
  WLZ_RECTANGLE,
  WLZ_ELLIPSE,
};

enum wlzoomarea {
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
// x, y    original ellipse center position from screen top-left
// h, w    radius in pixels, as total% ex. 270 pixels for 1080p
// dx, dy  destination ellipse center position from screen top-left

struct wlzoom {
  float pan_scale = 1.0;
  float pan_x = 0.0; // zoomed view top-left corner position
  float pan_y = 0.0;

  bool scope_enabled = false;
  bool scope_stuck = false;
  wlzoomarea scope_area;

  wlzoomregion *regions;
};

enum wlzoomexpand {
  WLZ_NONE,
  WLZ_ORIGIN,
  WLZ_DEST,
};

struct wlzoomregion {
  bool enabled = false;
  wlzoomexpand expand = WLZ_ORIGIN;
  wlzoomarea area;
}
```

## Reference implementation

### labwc

Ideal candidate because already has a magnifier tool that supports both
panoramic and scope zooms.

Proposed implementation path:

- Data model:
  - Add data model.
  - Persist xml configuration in new data model on boot.
  - Replace existing data model for the current magnifier.
- Expand xml configuration to allow region configuration.
- Add regional zoom support.

## Wayland control protocol

Waiting for reference implementations and user reports to validate data
model.

## Shadertoy versions

Because code is better than words...

### Rectangle

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

### Ellipse

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

This design merges the features of various applications I was using
before moving to Wayland. They worked fine in isolation but the feature
sets were needed for complex interfaces (videogames and dense UIs like
Godot).

Panoramic zoom: MacOS zoom and
[boomer](https://github.com/tsoding/boomer).

Scope zoom: [xzoom](https://tracker.debian.org/pkg/xzoom) (or
[xzoom ftp](ftp://sunsite.unc.edu/pub/linux/libs/X/), and later
[kmag](https://apps.kde.org/en-gb/kmag/).
