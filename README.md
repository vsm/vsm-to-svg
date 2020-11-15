# vsm-to-svg


&nbsp;  
## Introduction

This is a **draft design** for a module called `vsm-to-svg`,
that would simultaneously:
1.  serve as basis for **modularizing and simplifying** key aspects of the
    **[vsm-box](https://github.com/vsm/vsm-box)** code.
    > So this would support a _'vsm-box version 2'_. It would handle all
    > coordinate calculations for both terms and connectors in
    > this focused, reusable module.
    > It would also support making the many automated tests of vsm-box
    > nearly independent of tricky and slow
    > [DOM](https://en.wikipedia.org/wiki/Document_Object_Model) operations;
    > etc., see further below.
2.  serve as an improved basis for
    **converting [VSM](https://vsm.github.io) data to**
    **[SVG](https://en.wikipedia.org/wiki/Scalable_Vector_Graphics) file**
    output.
    > This functionality is currently (partly) embedded in
    > [VSM demo](https://vsm.github.io/demo)'s `dom-to-pure-svg.js`
    > [script](https://github.com/vsm/demo/blob/master/docs/dom-to-pure-svg.js).
    > But that code still depends on a vsm-box that needs to be drawn in a
    > browser, because it needs to read calculated coordinates from that vsm-box.
    > And it also can't reflect any of the vsm-box's custom styling,
    > like a larger font size. (In the demo, the two supported styles
    > just duplicate the CSS-code, but hard-coded into JS).

## Implementation
Unfortunately VSM's main designer and developer
[Steven V](https://orcid.org/0000-0002-3136-7353)
currently has no time **nor funding** to implement this design.
The full design idea was placed here though, where it may inspire contributions
from the VSM community – whose growth must be prioritized now.


&nbsp;  
## The design: a draft description

### Overview

Main goal: upgrade [`vsm-box`](https://github.com/vsm/vsm-box)
to a version 2, with a design like this:

- All calculation of coordinates, drawing of lines, and placing of text (plain
  or styled via html) should be done by a new, separate package called
  **`vsm-to-svg`**.
- This refactoring creates *simplified &amp; shared code* for drawing
  vsm-sentences:  
  - **simpler** for display and user-customization of a vsm-box in a webpage,
    and
  - **shared** between the vsm-box, and the code generating .svg-file output –
    which is currently in the vsm-box [demo](https://vsm.github.io/demo).
- This upgrade will enable:
  - **Multi-line vsm-sentences** would become feasible to implement,
    via multi-panel vsm-boxes.
  - **Multi-line vsm-terms** would be become more easily doable too.
  - Even **vertical vsm-boxes** would become a possibility, i.e.:
    with vsm-connectors on the left side, next to a column of
    (possibly long) vsm-terms on the right, each on its own line.
  - The **tests** would become _a lot_ lighter. It would be easier to add new
    ones, as it would nearly eliminate our need for the breakable `jsdom-global`
    module (esp. so when upgrading this to the latest version).
  - The whole **drawing configuration** (sizes, margins, colors) for terms and
    connectors would be united into a single `styles` Object, instead of having
    to manage a `sizes` Object for connectors, plus a perhaps rather convoluted
    and fragile CSS definition for terms.

- This new `vsm-to-svg` module will:  
  ◦ transform an **input data-Object** `{ vsm, state, style, options }`  
  ◦ into     an **output data-Object** `{ svgs, sensors }`,  
  as specified below.


&nbsp;  
### Input Object &nbsp;(+functionality notes)

The input data-Object has as properties: `vsm`, `state`, `styles`, `options`:

- __1.__ `vsm`: {Object} (required):
  - This is a vsm-sentence Object, in a format like `{ terms: […], conns: […] }`
    as described in the [vsm-box](https://github.com/vsm/vsm-box) documentation.
  - The **connectors** will be **stacked in the order they are listed**
    in `conns`, from bottom to top.
    &rarr; So they should have been sorted in a good order
    by external functionality (which vsm-box v1 does).
- __2.__ `state`: {Object} (optional):
  - This represents the full state of the vsm-box user-interface, w.r.t.
    how terms and connectors should be drawn, contain text, or be highlighted.
    It excludes things that are handled by other components, like:
    an autocomplete panel, or a popup for a term.
  - _Properties (all optional) are:_
  - `endTerm`: {Object}, with optional properties:
    - `value`: {String}: This is the text that should be shown in the _endTerm_
      (which is the input component that always remains at the bottom-end in a
      vsm-box interface).
    - `width`: {Number}: How wide the endTerm should be drawn.
    - `border`: {Boolean}: Whether its border should be shown
      (e.g. `true` when focused),
  - `editTerms`: {Object}, with optional property+value pairs.
    - Each **property** would correspond to an Edit-type vsm-term's index
      (called its 'term-Number', below) in `vsm.terms`. So `state.editTerms`
      could for example look like: `{ 0: {…}, 3: …, 5: …, …… }`.
    - Each associated **value** is an {Object}, with optional properties:
      - `value`: {String}: The text that has already been typed
        in that Edit-term (like before pressing Enter).
      - `placeholder`: {String}: The placeholder text that should be shown
        if `value` is empty or absent.
      - `placeholderSmall`: {Boolean}: Tells if the placeholder should be shown
        smaller and higher. This would be the case when an Edit-term without
        `value`-text is focused.
    - Note that no Edit-term is required to have a corresponding
      property+value here.
      For example if a VSM-sentence contains Edit-terms, but none of these
      contains text or a placeholder,
      then `state.editTerms` would be `{}` or absent.
  - `dragTerm`: {Object}, with required properties:
    - `nr`: (term-Nr): Indicates that the vsm-term at this index is being
      dragged. So it will be drawn with a placeholder-marker instead.
    - `dx`: {Number}: horizontal displacement of the dragged term, relative to
      its placeholder's position.
    - `dy`: {Number}: vertical displacement.
    - _Note_: if `state.dragTerm` is given, then the output data-Object will
      contain an additional SVG `svgs.drag` that draws only the dragged vsm-term.
      The caller (=vsm-box on a webpage) can then 'absolutely position'
      this SVG relative to the main SVG(s) showing the connectors and
      non-dragged terms.
  - `popupTerm`: {Object}: with required property:
    - `nr`: (term-Nr): Indicates that the vsm-term at this index is having its
      popup shown in the vsm-box.
    - _Note_: popups are _not_ handled in `vsm-to-svg`.  
      But when given a `state.popupTerm` property, `vsm-to-svg` also returns
      coordinates for a 'sensor' data-object for the area around this vsm-term;
      see later.
  - `posHL`: (term-Nr): to draw a 'position-highlight'
    for the column above the vsm-term with this index in `vsm.terms`.
  - `connHL`: (conn-Nr): to draw a 'connector-highlight' for the
    vsm-connector with this index in `vsm.conns`.
  - `ucConn`: (conn-Nr): to draw the last leg of this connector
    as an 'under-construction' leg.
  - `rmConn`: (conn-Nr): makes that this connector still occupies stacking-space
    between any other connectors, but it won't be drawn anymore.  
    This represents a vsm-box's state in the short time between when a user
    clicked on a connector's remove-icon (which first only hides it and
    marks it for deletion), and when the connector is actually removed from
    `vsm.conns` and the remaining connectors' new optimal stacking-order
    is also applied.
  - `riState`: (0/1/2): the state of the _remove-icon_ corresponding to
    the `connHL` (if a connector-highlight is in fact shown):
    0=normal, 1=mouse-hovered, 2=mouse-pressed.
  - `textCursor`: (term-Nr): to draw a (hidden) text cursor in the Edit-term
    at this position (if an Edit-term is actually present there).
  - `mouseCursor`: (term-Nr): to draw a (hidden) mouse cursor above the vsm-term
    at this position; but only so if a connector-highlights's leg or
    a position-highlight is currently present above this term.
    Vertically, this hidden mouse cursor will be placed just under the
    connector/position-highlight's top.
  - `mouseClick`: {Boolean}: to draw a (hidden) mouse-click indicator around
    the mouse cursor; but only if a mouse cursor is currently added.
  - _Note:_ `textCursor`, `mouseCursor`, and `mouseClick` are drawn hidden,
    and can be useful when editing an SVG-file (after un-hiding them).
- __3.__ `styles`: {Object} (optional):
  - This Object may include all styling aspects for the vsm-box,
    such as: colors; dimensions for connectors and arrows;
    margins and paddings for terms (incl. for all term type variations), etc.
  - Only non-default values need to be given, whereby default values are
    based on a `options.stylesPreset`.
  - This would be the `sizes` Object from vsm-box v1,
    but extended with new properties like those in the vsm-box demo's
    [`styleVals`](https://github.com/vsm/demo/blob/4e72ec2/docs/dom-to-pure-svg.js#L129).
- __4.__ `options`: {Object} (optional), with all optional properties:
  - `stylesPreset`: {String} (e.g. 'normal' or 'sketch') (default: 'normal'):  
      tells which 'styles-preset Object' should be used to get default values,
      for values that are not in the given `styles`.  
      Cf.: the vsm-box demo's two sets of
      [`styleVals`](https://github.com/vsm/demo/blob/4e72ec2/docs/dom-to-pure-svg.js#L129)
      (normal vs. sketch-box style).
  - `showBox`: {Boolean} (default: true):  
      to show or hide the vsm-box's border and its shaded background
      of the connectors panel.  
      If `false`, then only the terms and connectors will be shown,
      and the SVG(s) will be trimmed so it has no empty borders.
      This is useful for SVG-file output.
  - `showEndTerm`: {null|Boolean} (default: null):  
      if `null`, then it will follow the `options.showBox` value.  
      Note: if `false`, and `options.showBox` is also false,
      then the output-SVG's `width` (or `viewBox`-width)
      will exclude the endTerm – (the vsm-box demo already does this too).
  - `useViewBox`: {Boolean} (default: false):  
      if `false`: define output-SVG's dimensions via `width` and `height`
      attributes on the `<svg>`-tag (this may be good for web display);
      or if `true`: then as a `viewBox` attribute (good for SVG-files).
  - `strWidth`: Function: String => Number: _(see below)_.
  - `domElem`: DOMElement: _(see below)_.
  - `font`: opentype.js Font: _(see below)_.
  - _About the above three:_  
    In order to calculate how wide a vsm-term should be so that a text-string
    fits well in it, it should be possible to measure the width of text-strings,
    styled in the vsm-term's font etc. (Note: this is the width that is – or
    would be – needed if the vsm-term has no maxWidth to consider).  
    For this, one of the following three should be given (or else a default
    function is used) :
    - `strWidth`: a function that calculates this.
    - `domElem`: a DOMElement (e.g. the `<div>` containing a vsm-box
      on a web page).  
      Under this `domElem`, a hidden element will be added that is CSS-styled
      like a normal vsm-term. This hidden element will then repeatedly be filled
      with text-strings, and its auto-fitted width can then be read from the DOM.
    - `font`: a Font Object loaded with
      [opentype.js](https://github.com/opentypejs/opentype.js):
      its `getAdvanceWidth()` will be used.
  - `outlineText`: {Boolean} (default: false):
    - if `false`:
      - for terms without `style` property (i.e. no `vsm.term[*].style`):  
        it outputs a `<text>` element.
      - for terms with a `style` property without html-tags:  
        it outputs a `<text>` element, including `<tspan>` elements.
      - for terms with a `style` property that includes html-tags:  
        it outputs a `<foreignObject>` element that contains (+positions)
        it literally, with those html-tags.
        - if `style` includes only some simple, recognized html-tags,
          like `<b>`, then the code _may_ output more widely supported
          `<text>` and `<tspan>` elements instead.
    - if `true`:
      - _note:_ this requires that `options.font` is given too;
        if not, then `outlineText` will be treated as `false`.
      - it will output text (not as `<text>` elements but) only
        as `<path>`s.  
        This is safer for SVG-file output because it ensures that the SVGs
        look as intended, also on systems where some font may be missing.
      - for terms with a `style` property that contains html-tags:  
        these tags will be analyzed, and bold/italic/superscript/subscript
        will be kept, while other styling will be dropped.
  - `addClasses`: {Boolean|Array(String)} (default: true):
    - if `true`: adds CSS-classes to elements, so that the caller can analyze
      and further manage the generated svg code.  
      For example, a vsm-box needs to be able to detect what `<g>…</g>`
      corresponds to a connector-highlight, position-highlight, or vsm-term
      (see later for why).  
      And users of SVG-files need to be able to identify some elements,
      in order to un-hide a hidden text cursor, mouse cursor, etc.
    - if `false`: adds none.
    - if `Array`: only adds class-tags that are also in this array.  
      For example for `['mouse', 'hide']`, it would add only `class="mouse"`,
      `class="hide"`, or `class="mouse hide"`, where relevant.


&nbsp;  
### Output Object &nbsp;(+functionality notes)

The output data-Object has as properties: `svgs`, `sensors`:

- __1.__ `svgs`: {Array/Object}:
  - is an Array of SVG-data as Strings, one per multiline vsm-box v2 'panel'.
    So for a vsm-box with terms on a single line, this array's length is 1.
  - can have `drag` as an extra property (set on the same `svgs` Array-object):  
    this is the SVG-data String for the separate SVG image for a dragged term.
    - Note: terms can be dragged to slightly outside a vsm-box, so they cannot
      be placed inside the main vsm-box SVG; as that would need a vsm-box to
      get scaled along with drag position, which is bad behavior in a web-page.
    - This property is added only if a `dragTerm` was given.
    - See that `options.state.dragTerm`'s description.
- __2.__ `sensors`: {Array}:
  - Is a list of _'sensor area'_ data-Objects.
    - Each sensor defines an area where a vsm-box should listen for mouse events,
      and what vsm-box subcomponent this area corresponds to.
    - These areas do not overlap.
    - The vsm-box v2 would then add a separate, transparent SVG `<path>`
      element for each sensor area, and listen for mouse-events on it.  
      For efficiency, these paths would not be inserted in the SVG panels
      from `svgs`. Instead the vsm-box would manage one transparent
      SVG _'sensor-container'_ on top of of each SVG panel, and it would insert
      for each sensor a path-element in the relevant container.
    - Path coordinates are relative to the relevant SVG panel's top left.
    - Note that a sensor's presence depends on if its associated component is
      drawn in `svgs`; e.g. on if an endTerm is drawn, or any terms or conns,
      or any space for connectors – which would be omitted if there are
      no connectors while `options.showBox` is also false.
  - Each sensor is an Object _with the following properties_:
  - `type`: {String}: is one of:
    - `'term'` (term; one sensor per term):  
      a rounded rectangle with the same coordinates as the vsm-term,
      including the full border line-width.  
      This includes the endTerm, if it is drawn.
    - `'conn'` (connector; one or more sensors per connector):  
      together, the group of all `'conn'`-type sensors for the same `elemNr`
      will cover all the _cells_ (i.e. row+column rectangles in the drawing
      space for connectors) that the connector with `elemNr` as connNr occupies.
      This group can consist of multiple rectangles (e.g. one for each of the
      connector's legs or its backbone), or of a single shape that is the union
      of these rectangles, or of multiple shapes.
      The group will definitely consist of multiple areas if the connector
      is spread over multiple panels for a multiline vsm-sentence.  
      Note that these sensor areas cover entire cell rectangles. They differ
      from connector-highlights that can leave some margins uncovered
      inside these cells.
    - `'connFree'` (space above a term, unoccupied by connectors; any number):  
      together, the group of all `'connFree'`-type sensors for the same `elemNr`
      will cover all cells in the column above the term with `elemNr` as termNr,
      that are not occupied by any connector. Unless no space is drawn above
      the terms drawn, this group will consist of a rectangle that covers
      all cells above the highest-stacked connector above that term,
      plus rectangles to cover any other unoccupied, single or adjacent cells
      between connectors above that term.
    - `'connEnd'` (space above the endTerm; maximum one):  
      a rectangle that covers the column of cells above the endTerm
      (where no connectors can be added yet).
      This sensor is only present if the endTerm is drawn.
    - `'ri'` (remove-icon; maximum one):  
      a rounded rectangle that corresponds to the remove-icon's hover-area.
      This sensor is only present if a connector-highlight is drawn.
    - `'termPop'` (a term-with-shown-popup's 'halo' area; maximum two):  
      together, the group of `'termPop'`-type sensors cover a particular area
      around a vsm-term for which a popup is currently shown in a vsm-box.
      (Note that these popups are not drawn by `vsm-to-svg`).
      The area consists of two rectangles (or a joined path) that cover
      the term's left-margin and right-margin.  
      This enables a vsm-box to listen to the correct area around a vsm-term
      (with popup shown), where the mouse may leave the term but should
      not yet cause the popup to disappear.  
      Note: this sensor group does not include a bottom-margin under the term,
      even though some of that area should also not cause the popup to disappear.
      Actually, that area is not the term's bottom-margin; it is the popup's
      top-margin, and it should therefore be calculated by the vsm-box instead
      (based the popup's width and its distance to the vsm-term).
  - `elemNr`: (Number): (termNr|connNr|absent if not applicable):  
    is present for sensors of type `'term'`, `'conn'`, `'connFree'`, and
    `'connEnd'`, and absent for others.
    It tells which term or connector this sensor pertains to.
  - `panelNr`: {Number|`'drag'`} (optional, absent means 0):  
    tells which panel from `svgs` this area is relative to
    (in case it is a multi-panel vsm-box).
  - `d`: {String} (path definition as
    [SVG path commands](https://developer.mozilla.org/en-US/docs/Web/SVG/Attribute/d)):  
    this determines the shape and position for the SVG `<path>` element that
    vsm-box v2 would generate for each sensor.
    It is meant to be the `<path>`'s `d` attribute, and is relative to
    the top-left of SVG-panel number `panelNr` from `svgs`.
    - (_Alternatively:_ vsm-box v2 could use `<area shape="poly" coords=…>`
      elements instead, in which case the senser could (instead of `d`) have a
      `coos` property: a comma-separated String created by from a list of
      x- and y-coordinates, e.g. 'x1, y1, x2, y2, …, xn, yn').


&nbsp;  
### Note

`vsm-to-svg` does *not handle fading*.
- It adds maximum one connector-highlight (connHL) or position-highlight (posHL)
  based on `options.state`.
  So it manages _no_ parallel ones that may still be fading in or out.
- A **fade-in** effect for a connHL would therefore work like this:  
  the _caller_ (vsm-box) should find `<g class="connHL">…</g>` in `svgs`,
  _extract_ this string (i.e. remove it from the SVG-panel),
  and use `setTimeout()`s to _reinsert_ it with particular fade-in states
  (opacities). For efficiency, it could be added into an extra SVG container
  that fully covers an SVG-panel, and that is covered by the sensors-overlay.
- A **fade-out** effect for this connHL would work like this:  
  because `vsm-to-svg` does not include a connHL anymore as soon as its
  connector is unhovered, the caller should make a copy of the connHL `<path>`
  at deletion time, before redrawing it; and it should reinsert this copy
  at regular times and with less opacity, until faded out.
- A darker/lighter fading effect of a vsm-term border, caused by a
  mouse-hover/unhover, would work similarly.


&nbsp;  
## Results of this refactoring and modularization

### Effect 1

The vsm-box v2 code would now only have to manage a few data-Objects
acting as the **single-source-of-truth** (`vsm`, `state`), send these to
`vsm-to-svg` which generates SVG-code for what to show (based on this data only),
show it, listen to the sensor-areas with coordinates as instructed,
update the `vsm` and `state` objects when needed (e.g. after user-interaction
on a sensor), and after any update to these:
delegate all coordinate-calculation work to `vsm-to-svg` again.


### Effect 2

- The **test-code** would now only need a few nicely isolatable tests, that
  check that each 'sensor-area to initial-response' works well. This initial
  response could be simply be: to add an event-Object to
  the vsm-box v2's `eventQueue`.
- The rest of the tests can then ignore coordinate calculations, UI clicking
  etc., and the presence/absence of certain DOM-elements.  
  The tests can simply start by adding an event-Object to the `eventQueue`, and
  then test if the vsm-box made the correct changes to the `vsm` and/or `state`
  Objects after a short while.
- Also, the way how these Objects are translated into SVG code
  can then be nicely separated and placed into the `vsm-to-svg` repository.
- (Implementation note):
  vsm-box would respond to mouse-interaction on any sensor, by adding
  an entry to `eventQueue`, whose length is reactively `watch()`ed.
  When a watcher detects that an event-Object is added (i.e. new queue-length
  is larger than old queue-length; so removing an event-Object is ignored),
  it would run a `forEach()` to `shift()` each item and call the
  appropriate event-handler for it, wrapped in a `setTimeout(0)`.
  - Like this, multiple events can be queued at once (i.e. in a single
    JS event-loop), whereby the main handler will call a specialized handler
    exactly once per queued event, without opening any possibility that
    asynchronous changes to the queue could add new events during the
    `forEach()` loop (because it empties the queue synchronously, and only
    after that it executes event-responses asynchronously).
  - And therefore, the tests can simply start by adding event-Objects to this
    queue. I.e. all user-interaction can then be mocked by adding event
    data-Objects.
    And even other events (e.g. from the `vsm-autocomplete` component) could be
    merged into this event-handling hub. All events would have a (partly) shared,
    well-definable data-Object structure. Like this, the HTML-template code in
    Vue components would not have to call any particular event-handler function
    directly. The template-code would only have to send event-Objects
    to this shared, event-handling hub function.  
    (This may relate to Vuex; but then for this single component, not for a
     full app. Or perhaps this can be implemented with Vue's 'Store' pattern
     instead, i.e. the 'Flux' pattern.
     The vsm-box code has reached a point where this is useful).


### Effect 3

Supporting alternative ways of laying out a vsm-box would now be much easier
(in particular multiline or vertical vsm-boxes).
One would only have to update how `vsm-to-svg` maps input data-Objects to
output SVG panels and sensor-Object coordinates etc.
The vsm-box would still be fine; nothing really changes for it.
The vsm-box is now mainly just a controller of 'events to data-change responses'.
(Note that a vsm-box will still manage and position any vsm-autocomplete
or term-popup component that needs to be shown).



<br>

## License

This project is licensed under the AGPL license – see [LICENSE.md](LICENSE.md).

<br>
