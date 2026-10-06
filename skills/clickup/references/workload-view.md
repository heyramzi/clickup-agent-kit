# The Workload view: the three click paths, and why a synthetic click does nothing

What the two view-bar menus do, and which field the bar actually sums, stay in the skill.

Three click paths, all verified at 1900x861:

- **Collapse a row**: `.row-details-assignee-settings` inside `cu-timeline-group-row`,
  around x=600. Collapsing takes the row from 559px to 47px, which is what makes seven
  people fit one frame. Rows are virtualised, so collapse the topmost expanded row, re-probe,
  repeat, rather than mapping them all once.
- **Set a person's capacity**: `.cu-timeline-group-row__capacity-toggle` reads either
  `24/15` or `Set capacity`. Clicking it opens a popover whose only input carries
  `placeholder="-"`. **Double-click the input, type, then click Save**: a click plus
  `Meta+a` left the old value in place and the save was a no-op.
- **Keep the layout**: the state is per session until `Save view` is clicked, so a reload
  re-expands every row. Click it when the arrangement is going to be photographed twice.

**Synthetic `.click()` does nothing here.** `[...document.querySelectorAll(sel)].forEach(b
=> b.click())` ran twelve times against the collapse control and moved nothing, while the
same element clicked through `click([x, y])` collapsed on the first try. Probe with `js()`
to get coordinates; act with the mouse.
