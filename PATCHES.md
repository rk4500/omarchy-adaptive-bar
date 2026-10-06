# Local patches to the cloned bar

Cloned from `omarchy.bar` (Omarchy 4.0.2-1) with `omarchy plugin clone omarchy.bar`.

## Required-property fix — needed for the clone to load at all

A freshly cloned `omarchy.bar` does not work. The host mounts a custom bar with

    Loader { source: <plugin url>; onLoaded: shell.configureBar(item, manifest) }

(`/usr/share/omarchy/shell/shell.qml`, around lines 249-252), which assigns
`omarchyPath`, `barWidgetRegistry`, and `barConfig` *after* the component is
built. But `Bar.qml` declares all three as `required property`, and QML demands
those be set at construction time. The component therefore never instantiates,
the bar silently disappears, and the log shows:

    WARN scene: .../Bar.qml[15:3]: Required property omarchyPath was not initialized
    WARN scene: .../Bar.qml[17:3]: Required property barWidgetRegistry was not initialized
    WARN scene: .../Bar.qml[22:3]: Required property barConfig was not initialized

The stock bar avoids this because the host loads it through `sourceComponent`
with inline property bindings (same file, ~lines 229-231), which does satisfy
required properties at construction.

Fix applied here — drop `required` and give each a default, so the host's
`onLoaded` assignment lands on an already-built object. Match on the property
names, not line numbers, since upstream line numbers drift between releases:

    required property string omarchyPath      ->  property string omarchyPath: ""
    required property var barWidgetRegistry   ->  property var barWidgetRegistry: null
    required property var barConfig           ->  property var barConfig: null

Re-apply after any re-clone or after copying a newer upstream Bar.qml:

    sed -i \
      -e 's|^  required property string omarchyPath$|  property string omarchyPath: ""|' \
      -e 's|^  required property var barWidgetRegistry$|  property var barWidgetRegistry: null|' \
      -e 's|^  required property var barConfig$|  property var barConfig: null|' \
      Bar.qml

## Notes

- `omarchy bar reset` switches back to the stock `omarchy.bar` at any time.
- The bar layout still comes from the `bar:` subtree of ~/.config/omarchy/shell.json;
  this plugin only owns the rendering.

## Gotcha: some edits need a full shell restart

Saving a file under ~/.config/omarchy/plugins/ hot-reloads plugin code, but
`readonly property` bindings such as `barSize` (~line 330) do not re-evaluate
on a hot reload. Use `omarchy restart shell` when an edit appears to do nothing.
Verified by bumping barSize +20: the bar only grew 26px -> 46px after a restart.
