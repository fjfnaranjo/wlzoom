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
    WLZ_SQUARE,
    WLZ_CIRCLE,
};

struct wlzoom {
    float pan_scale = 1.0;
    // distance between original and zoomed top-left corners
    float pan_x = 0.0; 
    float pan_y = 0.0;

    bool scope_enabled = false;
    bool scope_stuck = false;
    unsigned float scope_scale = 1.5;
    wlzoomshape scope_shape = WLZ_CIRCLE;
    float scope_x = 0.0; // from screen TL corner to circle center
    float scope_y = 0.0;
    float scope_size_x = 0.25; // pixels for radius, as total%
    float scope_size_y = 0.25; // ex. 270 pixel radius for 1080p

    wlzoomregion *regions;
};

enum wlzoomexpand {
    WLZ_NONE,
    WLZ_INNER,
    WLZ_OUTER,
};

struct wlzoomregion {
    bool region_enabled = false;
    wlzoomexpand region_expand = WLZ_INNER;
    unsigned float region_scale = 1.5;
    wlzoomshape region_shape = WLZ_SQUARE;
    float region_x = 0.0; // from screen to square TL corners
    float region_y = 0.0;
    float region_size_x = 0.25; // pixels for side, as total%
    float region_size_y = 0.25; // ex. 270 pixel side for 1080p
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
