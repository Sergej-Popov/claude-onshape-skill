---
name: claude-onshape-skill
description: Field notes for driving Onshape (cad.onshape.com) through the Claude in Chrome browser tools - keyboard shortcuts, sketch/dimension/fillet click recipes, feature dialogs, assemblies, and read-only REST API calls from the page that replace slow clicking. Use whenever working in an Onshape document in Chrome.
---

# Onshape via Claude in Chrome

Practical notes for working in Onshape fast. Generic Onshape knowledge only, no project specifics.

## Golden rules

- **Read before clicking.** The REST API works from inside the page with the user's session (same origin, cookies), so a `fetch('/api/v6/...')` in `javascript_tool` returns exact feature parameters, sketch coordinates and assembly placements in one call. Use it to learn the model, then use the UI to change it. See "REST API from the page".
- **Never trust the screenshot for numbers.** Read dimensions from edit boxes or the API.
- **Onshape disconnects when its window isn't focused/visible** (the user is working in other windows). Run this focus shim right after every page load or navigation (it doesn't survive a reload):
  ```js
  Object.defineProperty(Document.prototype,'hidden',{configurable:true,get(){return false;}});
  Object.defineProperty(Document.prototype,'visibilityState',{configurable:true,get(){return 'visible';}});
  Document.prototype.hasFocus = () => true;
  if (!window.__focusKeep) { const stop = e => e.stopImmediatePropagation();
    ['visibilitychange','webkitvisibilitychange','blur','pagehide','freeze'].forEach(t => { window.addEventListener(t,stop,true); document.addEventListener(t,stop,true); });
    window.__focusKeep = setInterval(() => { window.dispatchEvent(new Event('focus')); document.dispatchEvent(new Event('mousemove')); }, 20000); }
  ```
  Keeping the Onshape tab in its own browser window also helps.
- If `javascript_tool` still answers but every screenshot times out ("Page.captureScreenshot timed out"), the Chrome window is probably minimised or fully covered, so Chrome stops painting it. The focus shim can't fix this. Ask the user to un-minimise or uncover the window. Smaller screenshots (`scale` 0.6) cut the load during heavy redraws.
- A grey overlay with "Onshape is not connected. Your document is saved. Attempting to reconnect" means the server connection dropped. Open sketches close, hovers and screenshots time out. Wait for the reconnect (or reload the element URL) before doing anything else. Reopen the sketch afterwards.
- Onshape tabs can freeze the renderer for 30 s or more (screenshots time out with "Script injection timed out"). Wait about 10 s and retry once. If it's still frozen, stop and ask the user to check the tab (often a stuck dialog). An open sketch that isn't accepted is lost on reload.
- Sketch edits are saved only when the sketch is accepted (green check, top of the sketch dialog). Accept often.

## Document layout

- URL: `/documents/{did}/w/{wid}/e/{eid}`. Each tab at the bottom (Part Studio or Assembly) is an element with its own `eid`.
- Bottom tab right-click menu: Rename, Duplicate (creates "X Copy 1" next to it), Delete, Export...
- Left panel: feature tree (Part Studio) or instance list (Assembly). The parts list is below the tree.
- Folders in the tree expand with the small arrow at about x 38. Expanding a folder shifts every row below it, so re-read positions after expanding (expand bottom-up to keep the upper positions valid).

## Keyboard shortcuts (canvas must have focus: click empty canvas first)

| Key | Action |
|---|---|
| `n` | View normal to the sketch plane (may come out flipped, e.g. "Bottom") |
| `shift+5` / `shift+6` | Top / Bottom view |
| `shift+7` | Isometric |
| `f` | Zoom to fit |
| `g` | Corner rectangle (sketch) |
| `d` | Dimension tool (sketch). Toggles: pressing it while active turns it OFF |
| `shift+f` | Sketch fillet |
| `Escape` | Exit current tool / clear selection (press twice to be sure) |
| `l` | Line (sketch) |

Sketch constraints (select the entities first, then press the key; much safer than toolbar clicks):
`i` Coincident (two lines â†’ collinear) Â· `shift+o` Concentric Â· `b` Parallel Â· `t` Tangent Â· `h` Horizontal Â· `v` Vertical Â· `shift+l` Perpendicular Â· `e` Equal Â· `shift+m` Midpoint Â· `shift+k` Normal Â· `shift+g` Pierce Â· `shift+q` Symmetric Â· **`shift+j` Fix** Â· `shift+u` Curvature.
In a narrow window the constraint icons collapse into a dropdown at the right end of the sketch toolbar, which lists these shortcuts.

Zoom: mouse wheel over a point keeps that point fixed. 10 ticks is only a small step. 6 to 9 batches of 10 ticks takes a full-part view to about 7 to 12 px/mm. To move to a different area: `f`, then zoom again at the new point (there's no easy pan).

## Coordinates: screenshot vs page (important for any JS event)

Screenshot coordinates are NOT CSS pixels. The screenshot can be smaller than the viewport (e.g. a 1355 px screenshot of an 1835 px `innerWidth`). For `document.elementFromPoint` or synthetic events, multiply screenshot coordinates by `k = innerWidth / screenshotWidth`. Check with `innerWidth` and `devicePixelRatio` once per session.

Page helpers used throughout these notes (define once per tab; they survive switching elements but not a reload). Set `W` to the width of your screenshots:

```js
const W = 1288;                                        // screenshot width in px
window.__k = innerWidth / W;                           // screenshot px â†’ CSS px
const vis = l => [...document.querySelectorAll('*')].filter(e => e.offsetParent && e.children.length < 3 && e.textContent.trim() === l);
window.__clickText = l => { const e = vis(l)[0]; if (!e) return 'not found';
  const r = e.getBoundingClientRect(), o = {bubbles:true, clientX:r.x+r.width/2, clientY:r.y+r.height/2, view:window};
  ['mousedown','mouseup','click'].forEach(t => e.dispatchEvent(new MouseEvent(t, o))); return 'ok'; };
window.__treePos = l => vis(l).map(e => { const r = e.getBoundingClientRect(); return ((r.x+r.width/2)/__k|0) + ',' + ((r.y+r.height/2)/__k|0); }).join(' | ');  // screenshot coords
```

## Selecting many sketch entities

- `ctrl+a` from the computer tool does NOT reach Onshape. The browser selects page text instead (the document title turns highlighted). Don't rely on it.
- The computer tool's `left_click_drag` does NOT make a box selection (no intermediate moves). Use synthetic pointer events instead. Window selection drags left to right (fully enclosed only). Start on bare canvas: check `document.elementFromPoint(x,y).tagName === 'CANVAS'`, because invisible dialog wrappers (`os-param-container`) and dimension labels (HTML overlays) cover parts of the canvas.

```js
window.__drag = async (x1,y1,x2,y2) => {            // CSS px
  const el = document.elementFromPoint(x1,y1);
  const o = (x,y,b) => ({bubbles:true,cancelable:true,clientX:x,clientY:y,button:0,buttons:b,pointerId:1,pointerType:'mouse',isPrimary:true,view:window});
  el.dispatchEvent(new PointerEvent('pointerdown',o(x1,y1,1))); el.dispatchEvent(new MouseEvent('mousedown',o(x1,y1,1)));
  for (let i=1;i<=12;i++){const x=x1+(x2-x1)*i/12,y=y1+(y2-y1)*i/12,t=document.elementFromPoint(x,y)||el;
    t.dispatchEvent(new PointerEvent('pointermove',o(x,y,1))); t.dispatchEvent(new MouseEvent('mousemove',o(x,y,1))); await new Promise(r=>setTimeout(r,30));}
  const t=document.elementFromPoint(x2,y2)||el; t.dispatchEvent(new PointerEvent('pointerup',o(x2,y2,0))); t.dispatchEvent(new MouseEvent('mouseup',o(x2,y2,0)));
};
```
Selected sketch entities turn orange. `Delete` removes them.

## Copy a whole sketch into a new sketch

1. Open the source sketch, right-click empty canvas â†’ **Copy sketch**. Close the sketch with the red X (no changes).
2. Toolbar **Sketch** â†’ pick the plane. The tree is often scrolled, so clicking the plane's outline in the viewport is more reliable than the tree.
3. Right-click empty canvas â†’ **Paste sketch entities**. The geometry sticks to the cursor (orange). Click to place it.
4. Internal dimensions and constraints come along. Delete unwanted parts with the box selection above.

## Sketch editing recipes

- **Open a sketch:** double-click it in the tree. **Accept:** the green check in the sketch dialog (about x 312, y 68). **Cancel:** the red X next to it.
- **Change an existing dimension:** double-click the dimension text â†’ edit box with the value selected â†’ `ctrl+a`, type `8 mm`, `Return`. Confirm with Return or the green check. Pressing Escape right after typing can discard the value.
- **Add a dimension:** `Escape` twice, click empty canvas, `d`, click entity 1, click entity 2 (or one line for its length), click empty space to place it â†’ edit box opens with the current measured value (a good sanity check of scale) â†’ `ctrl+a`, type the value, `Return`.
  - If the clicks highlight a *region* (orange fill plus an "Area: ..." readout bottom right), the dimension tool wasn't active: the `d` toggled it off. Reset and retry.
  - Lines closer than about 8 px apart are unreliable to click. Zoom in until targets are 10 px or more apart.
  - Dimension extension lines can lie on top of geometry and steal clicks. Pick a different edge or a spot along the line away from labels.
- **Rectangle:** `g`, click corner, click opposite corner, `Escape`. It starts blue (under-defined) and turns black when fully dimensioned.
- **Sketch fillet** (`shift+f`): clicking a vertex often hits a dimension label instead. Click the **two lines** that meet (line A, then line B) instead â†’ preview arc with the last-used radius â†’ `Return` opens the radius box (first time shows the default 5 mm; `ctrl+a`, type `1 mm`) â†’ `Return` again commits. `Escape` before the next corner. The radius is remembered between corners.
- Adding a closed shape inside an existing region splits it (the region becomes "region with hole"). Extrudes that referenced the region usually keep it, so cutting a window into a plate can be done purely in the sketch. Verify by viewing the face after rebuilding.
- Blue geometry = under-defined, black = fully defined.
- **Wait about 1 s after the `Return` that commits a dimension before accepting the sketch or pressing Escape.** Accepting straight away silently kept the old measured value (the dimension stayed, e.g. 39.893 instead of 40). Verify every sketch with the API after accepting.
- **Draw away from existing geometry, then dimension it into place.** Rough shapes drawn in empty space don't snap to the wrong things; dimensions from the origin (`d`, origin, edge or centre point, place) move them. Each dimension moves the shape, so re-read positions from a screenshot before the next click on it.
- **Circle** (`c`): click centre, click a point on the rim. Its first dimension is a diameter. Hover a centre point for 1 s before clicking to snap concentric.
- **Polygon** (toolbar pentagon icon, "Inscribed polygon"): click centre, click size. The shape then stays live with a "â—€ 6 â–¶" side-count widget; the next click commits it (clicking an arrow changes the count *and* commits). After that the count shows as an "Nx" label: double-click it, type `6`, `Return`. For a hex, dimension across flats between two parallel edges. Concentric polygons drawn the same way come out with the same rotation.
- A point on an axis: select the point, then the origin, press `v` (or `h`); then add one origin dimension for the other coordinate.
- **Unsigned dimensions keep the current side.** A shape drawn below the origin stays below when you type a positive distance. Put it on the right side first: dragging a sketch point with the synthetic `__drag` helper (pointer events with an initial hover move, CSS px = screenshot px Ã— `innerWidth/screenshotWidth`) moves it, solver and all.
- A point-to-circle dimension measures to the rim nearest the placement (e.g. origin to top of a Ã˜26 circle = centre + 13). Click the centre point instead when it isn't crowded, or type centre + radius.
- **Slot:** Offset tool dropdown (toolbar, next to Offset) â†’ Slot. Draw a centre line first, pick it, click once to commit (default Ã˜20), then double-click the Ã˜ label to set the width. The centre line becomes construction, so the slot is one region.
- External references: snapping a centre to a part edge (e.g. a hole) is lost when that part is hidden; the circle floats. Constrain to the origin with dimensions instead.
- **Sketching on a face, regions:** the face outside your shape is a region too. A click just outside a small shape selects that big region and extrudes the whole face. Click well inside the shape and check the part's bounding box after.

## Features

- **Extrude** `shift+e`. Dialog: Solid/Surface/Thin, then New/Add/Remove/Intersect, the region list, Blind + Depth field (about x 300, y 241 when 3 regions are listed; it moves down as regions are added), Merge scope (required for Add, red when empty). Pick sketch regions by clicking inside them (hover first). Depth: triple-click the field, `ctrl+a`, type `7.5 mm`, `Return`. Accept with the green check.
- **A sketch is auto-hidden once a feature consumes it.** To reuse its regions: right-click the sketch in the tree â†’ Show. If a part's face sits over the sketch in the current view, clicks pick the face: hide that part (Parts list â†’ right-click â†’ Hide), select the regions, and pick the hidden part for Merge scope by clicking its name in the Parts list. Show it again afterwards.
- **Mirror** (no shortcut): "Search tools" box in the toolbar â†’ type `Mirror` â†’ `Return`. Part mirror: entities (click a part in the Parts list) + Mirror plane (click the field, then a plane in the tree, e.g. Default geometry â†’ Right). **It can switch itself to "Add" with some merge scope when the result overlaps another part.** Re-click New before accepting.
- **Part colour:** Parts list â†’ right-click â†’ Edit appearance â†’ hex field (about x 218, y 813 at a 1256-wide view) â†’ `ctrl+a`, type the hex without `#`, `Return`. Read an existing part's exact colour from the API: `/parts/...` â†’ `appearance.color.{red,green,blue}`.
- **Changing Part Studio geometry moves the parts in every assembly that uses them** (instance transforms are relative to the studio geometry). After moving or re-mirroring a part, compute the studio delta from the API bounding boxes and move the assembly instance by the opposite amount (in its local axes).
- Multi-select parts: click, then ctrl+click in the Parts list, then right-click â†’ "Edit appearance for N parts" colours them all at once.
- Hidden parts and shown sketches: when the only part covering a sketch is hidden, the sketch stays visible after an extrude consumes it, so further extrudes can reuse its regions right away.
- Pick the mirror plane at the part's true outermost face (check the bounding box). A wall face that isn't the extreme makes the mirrored copy overlap the original. The result can be checked with `/boundingboxes`: the touching faces should share a coordinate.
- Quick check of which dimension box opened: `document.activeElement.value` in javascript_tool returns e.g. "8.6 mm", cheaper than a zoom screenshot.
- In a crowded sketch, double-clicking a label can open a neighbouring dimension (a radius, or `#variable`). Read the value in the edit box before typing; cancel with the red X if it's the wrong one. Zoom in until the labels separate.
- To mirror a part next to its source, use the source's outer side face as the mirror plane rather than a default plane. Picking a face in Mirror: clear the plane field, hover the face until it highlights, click. A coincident face on another part may get picked instead, which is the same plane and fine.
- **Transform** (Part Studio, via Search tools â†’ `Transform`): select parts in the Parts list (click, scroll, click more; no ctrl needed while the field is active), set the dropdown to "Translate by XYZ", type X/Y/Z (negatives work here), accept. A clean way to tidy the studio layout without touching sketches. Remember this moves those parts in every assembly too.
- A part derived from a hidden "source" part by Mirror is a separate part, so moving it with Transform afterwards is fine.
- Rename a part or feature: right-click it in the tree or Parts list â†’ Rename â†’ `ctrl+a`, type, `Return`.
- Hidden parts return all-zero bounding boxes from the API. Show them before measuring.
- API `/parts/.../partid/{pid}/bodydetails` â†’ `bodies[0].vertices[].point{x,y,z}` (metres): an easy way to verify z levels (extrude depths) and feature positions after building.

- Double-click an extrude in the tree â†’ dialog showing selections ("Face of Sketch 1" x N), depth, and merge scope. The selected regions highlight orange, which is a quick way to see which sketch regions a feature uses. Cancel with the red X.
- Double-clicking a **Transform** feature sometimes only highlights it and sometimes opens it (rolling the model back to that point). Read its parameters from the API, and if it opened by accident, cancel with the red X.
- The feature tree scrolls by itself while you work. Before double-clicking a tree entry by coordinates, take a screenshot or use `find` and `scroll_to` with a ref; a stale position opened the wrong feature.
- After accepting a sketch, check that no tree entry turned red and that the Parts (N) count is unchanged.
- **Extrude options worth knowing:** "Symmetric" checkbox (the depth is the total, so 52 gives Â±26), end-type dropdown (Blind, Up to next/face/part/vertex, Through all; Through all plus Symmetric cuts both walls of a box), "Starting offset". Like Mirror, Extrude switches itself to Add when the new body touches a part; re-click New after picking regions.
- **Shell** (Search tools â†’ `Shell`): click the face to remove (the bottom view `shift+6` makes it easy), set thickness (the field moves down once a face is listed), accept. It hollows inward.
- **Feature mirror:** Mirror â†’ dropdown "Part mirror" â†’ "Feature mirror", click the feature in the tree, then the plane. Good for a cut (socket, hole) that must appear on both sides.
- Picking a default plane edge-on: in Front view the Right plane is the vertical line through the origin; hover it, then click. That works when a click on the tree entry doesn't register.
- Parts list right-click â†’ "Hide all parts" (or "Hide other parts") clears the view to work on sketch regions inside a body.
- **Extrude direction:** the "Opposite direction" arrow is the icon right of the end-type dropdown (about x 318). The small icon right of the Depth field is "Tolerance options", not a flip; toggling it gives "Consumed end creates an invalid tolerant depth". "Boolean resulted in no geometry change" or "tools and targets do not intersect" on Add/Remove usually means the direction is wrong.
- **Plane** (Search tools â†’ `Plane`): pick a plane or face, type the offset. Handy for a feature that starts a gap away from a wall (sketch there, then extrude both ways from it).
- **Check interference** (Part Studio): select parts in the Parts list (click, shift+click), right-click â†’ "Check interference for N partsâ€¦". The dialog lists interfering pairs or says "No interferences". A quick static clash test before any assembly.
- Renaming many parts: a right-click on a list row can be synthesised from JS (`mousedown`/`mouseup`/`contextmenu` with `button: 2` on the row element), then click the menu item whose text is "Rename" via dispatched `mousedown`/`mouseup`/`click`. The inline input gets focus; type the name with the computer tool and press Return. List order stays in creation order after renaming.
- Context-menu items in general can be clicked by finding the element by its exact text (`offsetParent` set, few children) and dispatching mouse events, which avoids a `find` call and position drift (menus near the bottom of the list open upwards).
- **Never type into the page unless an input has focus.** Check `document.activeElement.tagName` first. Letters typed at the canvas are shortcuts: a hex colour like "1F1E1D" opened Extrude via shift+E, and `ctrl+a` selected the page text. In the appearance dialog, triple-click the hex field (find it via JS as the visible input whose value is 6 hex digits) before typing.
- Parts-list row positions drift by a row when the list scrolls after the first click (the header collapses). Re-read positions from the DOM (`getBoundingClientRect` of the row text Ã· `__k`) after the first click, before ctrl/shift-clicks. Rows near the bottom edge (y > ~850) sit under the tab bar; `scrollIntoView` them first.
- `p` toggles plane visibility; good before screenshots.
- Build each part in its assembled position in the Part Studio (sketch on mid planes, symmetric extrudes). Inserting each part into an assembly then lands it in place, and checking is just bounding boxes.

## Assemblies

- Instance right-click menu: Replace instances..., Hide, Fix, Edit in context, Switch to <part studio>, Delete, etc.
- **Replace instances:** keeps the instance's placement. Flow: dialog "Replace instance" â†’ click "Select replacement component" â†’ picker (Current document / Part Studios tab) â†’ expand the Part Studio with the `>` arrow â†’ click the part â†’ preview â†’ green check. Tick "Replace all instances" first to swap every instance of the same part at once.
- **Insert:** toolbar Insert â†’ Part Studios tab â†’ expand the studio with `>` â†’ click a part (each click inserts one more instance at its Part Studio position, i.e. identity transform) â†’ green check.
- **Exact positioning without mates (triad typed values):** click the instance in the viewport so the triad appears (it sits where you clicked and is aligned with the instance's own axes). A real click on an arrow only highlights it. A **synthetic press-drag-release of at least about 5 px** on an arrow cone (translate) or on the small ring handle at an arrow's tip (rotate) opens a value box. Type a value (`-38.448 mm` / `90 deg`), `Return`, `Escape`. The value is the displacement from the drag start, along or about the instance's own axis, so convert global deltas into part axes with the instance rotation. The tip handle on the X arrow rotated about Z and the one on Y about X (cyclic). Rotations pivot about the triad point, so do rotations first, then read the translation back from the API and correct it.
  - **Negative angles typed in the rotate box are silently ignored** (only the small drag rotation applies). Type `270 deg` instead of `-90 deg`. Negative distances in the translate box work fine.
  - Do the synthetic drag and the typing in the same batch. A value box left over after an undo or Escape is stale, and typing into it does nothing. Start a fresh drag.
  - Verify every step with the assembly API (`occurrences[].transform`) rather than the screenshot. Toolbar Undo (top left) cleanly reverts a bad transform.
  - Multi-select in the instance list: click, then shift+click. `Delete` removes the selected instances (no confirmation).
- **Fast way to add a part with an existing orientation:** instance list â†’ right-click an already-oriented instance â†’ "Copy <name>" â†’ right-click canvas â†’ "Paste <name>" (the paste keeps the rotation, lands at an offset) â†’ right-click the copy â†’ Replace instances â†’ pick the new part. The orientation is kept, so only translations are left to type. Much faster than rotating fresh inserts.
- Replace dialog flow: "Select replacement component" field (about x 340, y 165 at a 1036-wide view) â†’ picker â†’ expand the Part Studio with the `>` arrow â†’ pick the part via a `find` ref (list order shifts) â†’ green check.
- **Assembly selection is additive.** Clicking an instance (in the list or the viewport) adds to the current selection, and the triad stays on the first item. To select one instance: click blank canvas, wait 1 s, `Escape` twice, wait 1 s, then click the instance in the viewport (the triad appears at the click point). Check the list highlight before dragging.
- **Reliable clear selection:** right-click blank canvas â†’ "Clear selection" (near the bottom of the long menu; the menu is anchored so its bottom sits at the click point). With nothing selected the short canvas menu shows instead (Show all / Paste / Zoom to fit), which confirms the selection is empty.
- Selecting from the instance list (one click after Clear selection) puts the triad at the instance origin; more predictable than clicking in the viewport.
- **Paste lands at the right-click point** (the cursor position of the canvas menu). Two pastes from the same spot overlap exactly, which is fine: each is then moved by typed triad values computed from the API transform.
- **Assembly linear pattern** (Search tools â†’ "linear pattern"): pre-select the seed instance, open the tool, click the Direction field, hover then click a straight edge of any part (a silhouette edge against the background picks reliably; clicking a face gives the face normal). Set Distance and Instance count, green check. Pattern copies follow the seed. Watch out: clicking an edge while the Instances field is active adds that part as an instance; remove it with its Ã—.
  - Field values: focus the input via JS (`input.focus(); input.select()`), type, then press **Return**. Triple-click plus Tab once left the default 25 mm in place even though the box showed the typed value; verify spacing from the API afterwards.
- Collapsing the overlay: a 1-tick wheel zoom out after `f` keeps parts clear of the panels for clean screenshots; hover over the tab bar to keep the cursor out of the shot.
- Export several parts: ctrl-click them in the Parts list (modifier clicks work), right-click â†’ Export â†’ STL with "Export unique parts as individual files" downloads one zip.
- If an instance moves off-screen or behind the instance panel: right-click it in the list â†’ "Zoom to selection", then zoom out a few ticks.
- Instance context menu (right-click on the triad): Move to origin, Hide, Fix, Suppress, Replace, ... There is no typed-transform entry, and no Transform tool in search.
- Duplicating an assembly tab, then using Replace to point it at a duplicated Part Studio, gives a parallel test assembly with identical placements.

### Mechanisms: mates that move, and verifying them

- **Insert many parts:** in the Insert dialog the part rows are DOM elements; `scrollIntoView` the row by its text and dispatch `mousedown`/`mouseup`/`click` at its centre. Each click inserts one instance at the studio position. A plain click right after opening the dialog once inserted a part twice; trust the "Instances (N)" count in the list over an immediate API read, which can lag.
- **Group** (Search tools â†’ `Group`): click instances in the list, accept. Put everything that moves as one rigid set (shaft, cams, knobs) in one group and the frame in another; then Fix the frame (instance right-click â†’ Fix).
- **Implicit mate connectors, what you actually get:**
  - Circular edge â†’ the exact circle centre. The most reliable pick.
  - Cylindrical face â†’ a point on the axis, at the face's middle or at one end depending on where you hover. You can't tell which from the glyph, so check the result.
  - **Planar face with a hole â†’ the face's area centroid**, not the circle centre. A flange face with an off-centre hex hole put the connector 0.41 mm off-axis.
  - A point on a rim edge (triad sitting on the edge, not at the centre) is an edge point; move the cursor off the edge.
- **On creation, every mate snaps the connector origins together**, cylindrical included: the part jumps along the axis by the connector gap. Fix it with the mate's Offset (Z) field and work out the sign by trial (a wrong sign doubles the jump). Verify each instance's transform via the API after every mate; translation should be 0 for parts designed in place.
- Hiding an instance while a mate dialog is open works (instance list â†’ right-click â†’ Hide), and selections already made stay. Use this to reach a face that is covered (an eccentric disc inside a ring, a slot behind another leg), then Show it again for the second pick.
- Cylindrical leaves axial sliding free and parts drift during drags (a whole group slid 0.9 mm and showed up as interference). Use Revolute + Offset for shafts.
- **Tangent mate** (plane â†” cylinder, e.g. slot wall â†” guide pin) has several solutions (pin on either side of the wall). The solver keeps whichever is nearest **and will move any free part to get there, including the driving shaft**. Recipe:
  1. Fix the driving group while solving (instance â†’ Fix; the shaft revolute shows an error meanwhile, which is expected), and Unfix it afterwards.
  2. Put the free part near the intended pose first: a synthetic pointer drag on the part in the canvas rotates it about its revolute.
  3. Check the pose against an independent calculation (expected angle from the kinematics vs the transform's rotation `atan2(R[9],R[10])`).
  4. If it's on the wrong side, toggle "Flip primary axis" once. Read `flipAlignment` from `/assemblies/.../features` and remember that two toggles cancel. If flipping alone doesn't move it, suppress the mate, drag the part close to the right pose, unsuppress, then flip.
- **Check interference** (assembly): select instances in the list (click the first, shift+click the last), right-click â†’ "Check interferenceâ€¦". The result is a snapshot; the dialog's refresh icon (bottom left) re-runs it. The answer is a list of pairs or "No interferences".
- **Stepping a mechanism through exact angles:** temporarily switch the driving Revolute to **Fastened** (same connectors), then Offset â†’ "Rotate about Z" â†’ Rotation angle. Step at most 45Â° at a time; a 270Â°â†’0Â° jump left every mate in ERROR (toolbar Undo recovered it). At each step, read transforms and run Check interference. Finally set the angle to 0, switch back to Revolute and set the rotate dropdown back to "Rotate about X" (revolute refuses rotation about Z).
- Dragging instances in the canvas with the synthetic drag only works when the start point is on the part's shaded surface, not on a mate-connector glyph or dimension overlay, and not while all instances are selected.
- Mate status via the API: `/assemblies/.../features` â†’ `featureStates[id].featureStatus` (OK / ERROR) for every mate.

## REST API from the page (read-only GETs need no extra headers)

Run in `javascript_tool` on any cad.onshape.com tab of the document. Units are metres. Stash big results on `window.__x` and print slices.

```js
const base = '/api/v6/partstudios/d/DID/w/WID/e/EID';
await (await fetch(base + '/features', {headers:{accept:'application/json'}})).json();  // feature list with parameters
await (await fetch(base + '/sketches', {headers:{accept:'application/json'}})).json();  // sketch geometry: sketches[].geomEntities (lineSegment startPoint/endPoint, arc center/radius/startPoint/endPoint, many 'point' entries)
await (await fetch('/api/v6/assemblies/d/DID/w/WID/e/AEID', {headers:{accept:'application/json'}})).json(); // rootAssembly.instances (name, elementId, partId) + occurrences[].transform (4x4 row-major, translation in [3],[7],[11])
```

- **The API only sees the last accepted state of a sketch.** In-progress edits don't appear until the green check. To verify geometry: accept the sketch, GET `/sketches`, reopen. Great for catching bad snaps, e.g. a line endpoint that snapped to an arc's centre point or landed 0.2 mm off an endpoint.
- **HOVER BEFORE EVERY SKETCH CLICK.** Onshape's snapping and automatic coincident constraints only happen if the pointer hovered over the target first. A bare `left_click` (no prior mouse move) places a FREE point even when it lands exactly on an existing endpoint. Pattern: `hover` near the target, `hover` on the target, wait 1 s (a yellow square or highlight shows the snap), then `left_click` the same spot.
- Joining existing geometry without snapping: select point + line (or point + point) and press `i`. Point-on-line moves the point onto the line's infinite extension; the less-constrained entity moves. Check the result and use toolbar Undo (top-left arrow at about x 44) if the wrong thing moved. Undo through the toolbar works; `ctrl+z` may not reach Onshape.
- Line-tool snapping grabs any nearby point, including arc centres. Zoom in until the target point is 20 px or more from any other point, or draw roughly and then fix with constraints (`i` coincident, `v`/`h`).
- Feature parameters: `parameterId` plus `expression`/`value`, and `queries` for selections. Useful IDs: `operationType` (NEW/ADD/REMOVE), `depth`, `oppositeDirection`, `booleanScope`, `entities`, `startOffset*`, transform `transformType` / `dx` / `dy` / `dz` / `distance`, mirror `mirrorPlane`.
- Sketch feature JSON (`BTMSketch-151`): `entities[]` are `BTMSketchCurveSegment-155` with `geometry` `BTCurveGeometryLine-117` (pntX, pntY, dirX, dirY; startParam/endParam = signed distances along dir) or `BTCurveGeometryCircle-115` (radius, xCenter, yCenter, xDir, yDir, clockwise; params are angles in radians). The sketch plane is `parameters[sketchPlane].queries[0]`.
- **Writes (POST/DELETE) return 401 without the XSRF header.** GETs work with cookies alone. Writing needs Onshape's `X-XSRF-TOKEN` header, whose value comes from a cookie. Reading it means handling a session credential, so ask the user before using it, or create geometry another way (UI, or DXF import into a sketch).
- **Output filter gotcha:** the Chrome tool blocks results that look like cookie/query-string data (`key=value` runs, JSON with long IDs). Format output as `key -> value` or `key value | ...` text, and don't print raw JSON or entity IDs.
- Keep printed output under about 1000 to 1500 characters or it's truncated. Page through with `.slice()`.
- Per-part bounding boxes: `/api/v6/parts/d/DID/w/WID/e/EID/partid/{pid}/boundingboxes`. The Part Studio `/boundingboxes?partId=` form ignores the part and returns the whole studio. A small helper that lists every part with its box (`name [pid] x lo..hi y .. z ..` in mm) after each feature catches wrong regions and directions immediately.
- Sketch check helper: GET `/sketches`, filter `geomEntities` by `entityType` (`lineSegment` has startPoint/endPoint, `circle`/`arc` have center/radius) and print in mm. Coordinates are in the sketch plane's 2D frame.

## Browser-tool gotchas specific to Onshape

- **NEVER click sketch toolbar buttons (constraints like Fix, Coincident, ...) via `find` refs, even fresh ones.** The ref resolves to an element whose click lands on the Onshape logo (top left) and navigates to the documents list. This happened twice. Locate toolbar icons by hovering coordinates and reading the tooltip (e.g. "Sketch fillet (shift+f)"), then click by coordinates. Refs are fine for context-menu items and tree entries.
- **`find` refs go stale fast.** Onshape re-renders the toolbar when the mode changes (sketch â†” part studio â†” dialog). Clicking an old `ref_N` can hit something else entirely; one stale ref hit the Onshape logo and left the document. Run `find` again right before clicking, or click by coordinates from a fresh screenshot.
- If you leave the document by accident: re-`navigate` to the element URL. Open sketch edits survived a navigation away (Onshape saves sketch changes as you go), but don't count on it. Accept sketches often.

- `get_page_text` is useless (canvas app). Use screenshots plus `zoom` on regions.
- Every `scroll` inside `browser_batch` returns a full screenshot. Zoom with one standalone scroll call, or accept the image cost. Wheel zoom also drifts the view, so re-screenshot before clicking.
- `find` works well for menus: "Rename menu item", "Replace instances... menu item".
- A screenshot at `scale: 0.6` to `0.7` is enough to navigate. Multiply coordinates back by 1/scale.
- After a context-menu click, wait about 1 to 2 s before the next action. Dialogs animate in.

### Animating a mechanism
- Mate list â†’ right-click the driving revolute â†’ **Animateâ€¦**. Dialog: Start/End value (default 0â€“360 deg), Steps (default 300; 120 runs quicker), Playback type (Single / Reciprocate / Loop; a custom dropdown, not a `<select>`), â–¶ / â– . The dialog resets to defaults each time it opens.
- Clean view first: right-click any mate â†’ **Hide all mates** (hides the mate glyphs), then expand a mate, right-click one of its "Mate connector" rows â†’ **Hide all mate connectors** (hides the connector triads). The canvas context menu and view-cube dropdown have no such toggle.
- To restore: right-click a mate â†’ Show all mates; right-click a mate connector row â†’ Show all mate connectors.
- gif_creator captured only 1 frame across a whole animation (a click, 2 standalone screenshots and 12 batched screenshots). Don't promise the user a GIF from it; if one is needed, step the mate angle and save a screenshot per step (`save_to_disk`), then assemble the frames locally.

## Revolve, helix, sweep, threads, variables, configurations (learned on a screw-top jar)

### Tool search and dialogs
- **Open tools via JS, never by clicking the search box by coordinates.** The "Search tools" button moves when the toolbar changes (sketch vs part mode). A miss sends the typed tool name to the canvas as shortcuts. `document.querySelector('button.command-search-trigger').click()` focuses the search input; then type the tool name and press Return.
- **Wait about 2 s after a tool opens before the first selection click.** A region click straight after Return was silently dropped (the feature then failed with empty selections).
- **Escape cancels the whole feature dialog** (helix, mateâ€¦), not just a dropdown or popup. Close dropdowns by picking an item.
- Dropdown items (Turns / Turns and pitch, Start angle / Start point, Lock profile direction, Hide, Show, Rename, Delete, Configure, Animateâ€¦) click reliably with a JS text match (`__clickText('Turns and pitch')`: find visible elements whose trimmed text equals the label, dispatch mousedown/mouseup/click).

### Variables and expressions
- Variable feature (search `Variable`): Name and Value fields, accept. Dimensions then take expressions: `#r`, `#h - 12 mm`, `#r - 2.2 mm`, `2 * #r - 4 mm`. Feature fields (Plane offset etc.) take them too.
- **The first dimension in a fresh sketch scales the whole sketch** about the origin. Make the first dimension a big, meaningful one (an overall length or the origin-to-shape distance), not a tiny one: a first dimension of 2.8 mm shrank a triangle to unclickable size.

### Sketch picking near bodies
- **Clicking the origin while a body edge passes through it picks the edge** (an origin-to-line dimension became a 90Â° angle). Picking the tree's Origin row in dimension mode does nothing. Fix: hide the parts (Parts list â†’ right-click â†’ Hide / Hide all parts) while dimensioning to the origin. Cylinder silhouettes aren't edges and can't be picked at all.
- Unsigned dimensions keep the side: to put a point between two others, dimension it from the point it should be near (e.g. tip-to-point 1.4) rather than dragging; a synthetic drag made the sketch over-constrained (all red) and needed Undo.
- Two points: click the first, then hover + click the second (both turn orange), then `h`/`v`.
- `n` can give a mirrored "Back" view; `shift+1` gives Front.

### Revolve, shell, fillet, chamfer
- Revolve (search `Revolve`): click the region, click the "Revolve axis" field, click a sketch line on the axis (a line drawn on the axis works; the origin vertical isn't pickable). Full revolve default.
- Shell: click the face to remove, set thickness (field moves down after the face is listed).
- Fillet / Chamfer: click the edge (hover first). Clicking a face chamfers all its edges. Radius / distance field moves down after the first selection; triple-click it and check `document.activeElement.value`.
- A selected face appears as "Face of <feature>" in the field; read it before trusting the click.

### Helix
- Helix tabs: **Cylinder/Cone** (pick a cylindrical face; starts at the face's bottom edge at Start angle, runs along the face), **Axis**, **Circle** (pick a sketched circle edge; starts in the circle's plane).
- Input type "Turns and pitch" gives exact pitch and turn count. The Height option follows the face.
- "Start point" with Cylinder/Axis kept failing (red field, no preview) with a sketch vertex on the face. Don't fight it: use the **Circle tab** with a circle sketched on an offset Plane at the exact start height.
- Same tab, same Start angle and same "Clockwise" setting gave helices in the same phase (a Cylinder-tab helix and a Circle-tab helix both started on +X).
- Helix curves are wire bodies: `/parts?includeWireBodies=true` lists them; their bounding boxes give the z range; `bodydetails` returns nothing for them.

### Sweep (threads)
- Sweep: region, Sweep path (click the curve in the viewport, or expand Parts â†’ Curves and click "Curve N"), **Profile control â†’ Lock profile direction, Direction to lock = Top plane**. With "None" the thread profile drifted.
- A sweep still built when the profile was 4 mm off the path start.
- The API bounding box of swept parts is loose (showed 33.95 mm for a 29.4 mm thread, even with `precise=true`). Check real extents with `bodydetails` vertices or a view.
- Booleans: Add/Remove, then click the Merge scope field and click the part in the Parts list (works even when the part is hidden).

### Mating threads: cut the nut with the screw's own helix
- Internal thread = Remove-sweep a **larger** profile along the same helix as the external thread (base same radius, tip +0.4 mm, flanks +0.4 mm). Phases match by construction.
- **The groove helix must start at least one turn below the external thread's start** (and run out of the skirt bottom), otherwise the start of the external thread hits solid material as soon as the nut turns. Make it with an offset Plane at that height + circle + Circle-tab helix, with a turn or two more than the thread.

### Configurations
- Right panel icons: Appearance, **Configuration panel**, Custom tables, â€¦, Variable table (bottom). Configuration panel â†’ "Configure Part Studio" â†’ a list input with rows (Default + Name row; type in the empty Name cell, Return).
- Configure a variable: open the variable feature, right-click its Value field â†’ Configure. A column appears. Edit a cell: click it, press Return (sometimes twice) until `activeElement` is an INPUT, `ctrl+a`, type, Return.
- Switch configuration with the dropdown at the top of the feature list.
- API: `/configuration` â†’ `configurationParameters[0].parameterId` (e.g. `List_â€¦`) and `options[].option`. Pass `?configuration=` + encodeURIComponent(`<parameterId>=<option>`) to `/features`, `/parts`, `/boundingboxes` to check every configuration builds.

### Screw motion in an assembly
- There's no screw mate: make a **Cylindrical** mate, then search `Screw relation`, pick the cylindrical mate, set Distance per revolution, tick Reverse direction if needed.
- The relation's zero is where the mate connectors coincide; it moves the part there. Put the connectors at the closed position: in the mate dialog click the connector's small edit icon â†’ tick **Move** â†’ type the Z offset (the axial distance between the two connectors, read from the API) so the connector lands on the other part's connector.
- Check the screw direction with an interference check after rotating the nut 90Â° (triad X-tip handle). Wrong direction = interference along the whole thread.
- A large typed rotation (270Â°) was applied as +90Â° under the screw relation; step 90â€“180Â° at a time and read the transform.
- Animate needs a 1-DOF mate; a Cylindrical mate with a screw relation didn't offer a DOF to animate.
- Changing Part Studio geometry used by an assembly is fine; re-run interference checks after.

## Gears, patterns, gear relations (learned on a crank gear train)

- **Spur gear** custom feature (search `Spur gear`; Onshape's own FeatureScript): number of teeth, module, depth, center bore (tick, then Bore diameter), **Profile offsets â†’ Backlash** and **Clocking angle**. Default location is the Front plane at the origin; pick a sketch vertex in "Sketch vertex or mate connector" to place it elsewhere. Defaults reset on each new instance.
- Two gears at exact center distance (mÂ·(z1+z2)/2) with zero backlash are reported as interfering; 0.2 mm backlash on each cleared it. With equal "tooth on +X" phase, the second gear needs a half-tooth rotation (Transform â†’ Rotate about its bore edge, or Clocking angle = 180Â°/z).
- **Circular pattern** (search `Circular pattern`): dropdown Part / Feature / Face pattern. Feature pattern: click the feature in the tree, click "Axis of pattern" then a circular edge on the axis (a peg edge works), Instance count, Equal spacing default 360Â°.
- **Through all + Remove with Symmetric and a merge scope** cuts only the chosen part even when other parts lie on the path.
- A shown sketch's regions overlay everything. A region click can pick a region of a different shown sketch ("Face of Sketch 2" instead of the hole). Hide sketches you don't need before picking.
- Dimension placement clicks must land on empty canvas; landing on a sketch region gives "A dimension cannot be created between these items".
- A circle drawn near another sketch's (or gear's) construction circle can snap its centre onto it (coincident). Later dimensions then over-constrain ("Sketch could not be solved", red dims). Draw away from construction geometry, then dimension it into place.
- Extrude "Starting offset" has its **own** direction flip, separate from the main "Opposite direction".
- Typing into a dimension/field: if `document.activeElement` isn't the INPUT (the tab froze or the triple-click missed), the text goes to the canvas as shortcuts and Return may accept the whole feature. Focus inputs from JS instead:
  ```js
  window.__inputs = () => [...document.querySelectorAll('input')].filter(e=>e.offsetParent && e.type!=='checkbox' && e.getBoundingClientRect().x < 600*__k).map((e,i)=>{const r=e.getBoundingClientRect(); return i+': '+e.value+' @'+((r.x+r.width/2)/__k|0)+','+((r.y+r.height/2)/__k|0)}).join(' | ');
  window.__focusInput = (idx) => { const els=[...document.querySelectorAll('input')].filter(e=>e.offsetParent && e.type!=='checkbox' && e.getBoundingClientRect().x < 600*__k); const e=els[idx]; e.focus(); e.select(); return e.value; };
  ```
  then type the value with the computer tool and press Return.
- Tree rows: `__treePos('Extrude 1')` (find elements whose trimmed text equals the label â†’ centre Ã· `__k`) beats guessing row coordinates, which shift when panels open.

### Revolute + gear relation
- **Implicit connector picks for a gear on a peg:** the peg's flat end face (full disc) gives its centre. The gear's **front face** (planar with holes and a knob footprint) gives an area centroid 0.22 mm off-axis. Pick the **bore's inner cylinder** instead (hide the base so the peg doesn't cover it; the cylinder highlights along its length with the connector on the axis).
- In an edit, the âœ• buttons remove connectors by row; after one removal the rows shift. Read the list before clicking the next âœ•.
- The revolute dialog has an **Offset** checkbox â†’ Z field: fixes the axial jump from snapping connectors (here 1.5 mm).
- **Gear relation** (search `Gear relation`): pick the two revolute mates in the tree, set Relation ratio (second turns ratio Ã— first; 2 for 24T driving 12T), Reverse direction if the pinion turns the same way. Check with a triad rotation of the driver: +15Â° gave pinion âˆ’30Â°.
- Triad on a part whose axis is Y: the **Z-arrow tip handle** rotates about Y (X-tip â†’ about Z, Y-tip â†’ about X, Z-tip â†’ about Y).
- With a gear relation the system is 1-DOF and Animate on the driver revolute plays the whole train. The â–¶ button sits at about (193, 228) in a 1288-wide frame; check the "Current value" changes.

## Sheet metal (learned on an L-bracket)

- **Sheet metal model** (search `Sheet metal model`): tabs Convert / **Extrude** / Thicken. Extrude: pick open sketch lines (an L polyline), Blind depth, Thickness, Bend radius; bends appear at the sketch corners automatically. Sketch lines that lie on default-plane edges are hard to click in Front view; switch to isometric (`shift+7`) and `p` to hide planes first.
- **Flange** (search `Flange`): click an edge (e.g. a base's outer top edge), set Distance (default 25 mm, going up), alignment Inner by default (adds thickness outside: the base grew from 40 to 42 mm overall).
- Holes in sheet metal: a normal sketch + Extrude Remove (Through all, Symmetric, merge scope the part) works inside the sheet metal model.
- **Sheet metal table and flat view**: right-panel icon (tooltip "Sheet metal table and flat view") opens a split view with the bend table (name, radius, angle, direction) and the live flat pattern with bend lines and holes. Click the same icon (now inside the canvas strip) to close it.

## Drawings (learned on a drawing of an assembly)

- **The drawing editor is a cross-origin iframe** (`production-drawing-â€¦.onshape.com`). Page JS can't reach its DOM: `__clickText`, `__inputs`, `document.activeElement` all stop at the IFRAME. Use real clicks, `zoom` screenshots and typing only. Typing goes into whatever field inside the iframe has focus (e.g. a selected combobox text).
- Create: tab bar `+` â†’ "Create Drawingâ€¦" â†’ template tabs All / ANSI / ISO / JIS (ISO_A3.dwt etc.) â†’ OK. Options "Suggested views" are only enabled when started from a part; otherwise "No views".
- **Insert view** (first drawing-toolbar icon) opens a picker (Part Studios / Assemblies tabs); pick the assembly, then the base view follows the cursor at 1:1. Click to place, then move the cursor right/down and click to place projected views. The chain can end early; the **Projected view** tool (`p`, next icon) then: click the parent view, move, click.
- Clicking inside the Insert-view dialog area can drop an extra view where you clicked (two stray views landed on the sheet). Delete a view: click empty space inside its frame (the whole view turns orange), press Delete. Clicking a line selects only that edge.
- **View properties**: double-click a view. Scale is an editable combobox: click it (text gets selected), type `1:2`, Return, then the green check. "Scale label" ticks itself for non-sheet scales.
- Views move with a plain `left_click_drag` from inside the view.
- **Dimension** tool (`d`, the slanted-dimension icon with a dropdown): hover each edge until it turns orange, click; second edge; click to place. Don't click the tool icon again between dimensions, because it toggles the tool off. Picking near a corner where another body (an arm block) meets the edge grabs the wrong entity; pick mid-edge away from other geometry.
- **BOM table** icon (tooltip "BOM table"): the table follows the cursor anchored at the chosen corner (default top-right); click to place at the sheet's inner top-right corner. The default template shows Item / Quantity / Part number / Description; part names don't appear unless parts have part numbers or the template is changed.
- Title-block cells are linked-property notes; double-clicking opens a note editor (cancel with its red âœ•).
