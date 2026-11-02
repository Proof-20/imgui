# Reaver Panels

Status: current implementation. Windows Release compilation verified;
complete interactive acceptance and Linux validation remain pending.

## Ownership

ImGui owns panel presentation and category layout. The caller owns visibility,
control values and the lifetime of objects referenced by callbacks.

The API does not register panels or configure engine subsystems. Panels use the
main viewport by default, with docking disabled. When the caller supplies
`context.detached = true`, BeginPanel requests an owned viewport through ImGui's
installed platform/renderer backends. The caller executes host-transfer requests
returned by EndPanel and owns native creation failure recovery.

## Title-bar window toggle

The window toggle appears beside maximize and close when `context.enabled` is true
and `ImGuiStyle::PanelWindowIcon` contains a non-empty UTF-8 icon. Both
`PanelWindowIcon` and `PanelWindowSlashIcon` are borrowed strings that must remain
valid while used. The glyphs must exist in the current title-bar font; the fork
does not include Font Awesome headers or load another font.

EditorStyle supplies Font Awesome DESKTOP and SLASH from the existing merged font.
Set `style.PanelWindowIconScale` in EditorStyle to resize both glyphs together;
the default is `0.75f`. Values must be greater than zero and at most one. This
relative scale stays unchanged by ScaleAllSizes and preserves the button hit box.
The detached state highlights the button and overlays the slash glyph. The button
reports `ImGuiPanelAction_CreateWindow` or `ImGuiPanelAction_ReturnToMainWindow`
through EndPanel; it does not change host state itself. The caller's confirmed
`context.detached` state controls the presentation on the following frame.

Creation requires `context.can_create_window`, ViewportsEnable and platform plus
renderer viewport capability. Otherwise the button is disabled with an explanatory
tooltip. Return remains available for an already detached panel. EndPanel must be
called even for a collapsed panel; it also submits the deferred tooltip and restores
the panel's style scope. Right-click no longer opens a panel-transfer menu.

Verification: Windows Release ReaverEditor compiled and linked. Seven headless
checks passed using the real ImGui library and Font Awesome file with mock platform
callbacks: glyph presence, create/return dispatch, collapsed-panel interaction,
unsupported capability, denied permission, hidden toggle and absence of a right-click
popup. These checks do not validate native rendering, swapchains, DPI or Linux.

## Basic usage

Call the drawing function once per frame. Keep its state alive between frames.

```cpp
#include <imgui.h>

struct SettingsPanel
{
    bool open = true;
    bool enabled = true;
    float intensity = 1.0f;

    void draw()
    {
        if (!open)
            return;

        const ImGuiPanelConfig config{
            .size = {
                .width = 360.0f,
                .height = 480.0f,
                .min_width = 240.0f,
                .min_height = 160.0f,
            },
            .layout = {
                .padding_x = 6.0f,
                .padding_y = 4.0f,
                .row_spacing = 2.0f,
                .column_spacing = 5.0f,
                .border_radius = 5.0f,
            },
        };

        if (ImGui::BeginPanel("Settings###settings.main", &open, config))
        {
            if (ImGui::BeginCategory("Lighting"))
            {
                ImGui::CreateOption({
                    .title = "Enabled",
                    .output = [&] { return ImGui::Checkbox("##Value", &enabled); },
                });

                ImGui::CreateOption({
                    .title = "Intensity",
                    .output = [&] { return ImGui::SliderFloat("##Value", &intensity, 0.0f, 10.0f); },
                });
            }
            ImGui::EndCategory();
        }
        ImGui::EndPanel();
    }
};
```

Always call EndPanel after BeginPanel, including when BeginPanel returns false.
Always call EndCategory after BeginCategory, including when BeginCategory returns
false. Submit content only when the corresponding Begin call returns true.

Do not return early across an open panel, category or callback-owned ImGui scope.

## Panel configuration

Dimensions and spacing use ImGui coordinate units, matching native ImGui APIs.
They are not CSS values or guaranteed physical pixels on every DPI configuration.

| Setting | Default | Meaning |
| --- | --- | --- |
| size.width / height | 0 | Initial size; zero leaves that axis to ImGui automatic sizing |
| size.min_width / min_height | 0 | Additional minimum; the theme minimum still applies |
| size.max_width / max_height | FLT_MAX | No additional maximum |
| layout.padding_x / padding_y | -1 | Inherit WindowPadding |
| layout.row_spacing | -1 | Inherit ItemSpacing.y |
| layout.column_spacing | -1 | Inherit ItemSpacing.x |
| layout.border_radius | -1 | Inherit WindowRounding; zero selects square corners |
| layout.initial_x / initial_y | FLT_MAX | Use ImGui placement |
| behavior.resizable | true | Allow manual resize |
| behavior.movable | true | Allow manual movement |
| behavior.collapsible | true | Show the collapse control |
| behavior.closable | true | Show close when an open-state pointer is supplied |
| behavior.maximizable | true | Show maximize/restore |

Initial size and position use ImGuiCond_FirstUseEver. They do not continuously
force geometry after the user moves or resizes the panel.

Set both initial coordinates together. They are relative to the main viewport
work area. Set maximum sizes at least as large as the effective minimum sizes.

Maximize fills the main host work area and temporarily overrides floating size
constraints. Restore clamps the saved geometry to the available host area.

Pass panel configuration on each draw call. ImGui does not retain a reference
to the configuration object.

## Categories

BeginCategory uses a native CollapsingHeader followed by a two-column table.

ImGuiCategoryConfig defaults:
- title_ratio = 0.45f: relative width of the title column.
- default_open = true: initial expanded state.

title_ratio must be strictly between zero and one. default_open does not force
the category open on every frame.

## Options and callbacks

CreateOption draws the title in the first column and immediately invokes output
in the second column. It returns the callback's boolean result.

Use native ImGui controls directly. No CheckboxOption or SliderOption wrapper
is required.

The callback is borrowed, not owned. Inline lambdas passed directly to
CreateOption remain valid for that call. Do not store descriptors constructed
from temporary lambdas for later execution.

For read-only output, draw the value and return false.

Callbacks must balance any ImGui scopes they open.

## Output widget gallery

This category demonstrates the native value-control families, buttons, images
and read-only outputs. Overloads with different getters or formatting parameters
are not repeated. Containers such as trees, menus, tabs and popups have their own
Begin/End contracts and are not individual value outputs.

Keep one OutputWidgetGallery instance in the owning panel state. Call its
draw_content() inside a successful BeginPanel, once per frame. The example uses
C++20 designated initializers.

Variants of the same numeric family intentionally share arrays so their
representations can be compared. Scalar examples use double storage paired with
ImGuiDataType_Double. SliderAngle stores radians while displaying degrees.
The input text overload requires both a writable buffer and its capacity.

```cpp
#include <imgui.h>

struct OutputWidgetGallery
{
    char text[128] = "Example";
    char hint[128] = "";
    char multiline[512] = "First line\nSecond line";
    float floats[4] = { 10.0f, 20.0f, 30.0f, 40.0f };
    int integers[4] = { 10, 20, 30, 40 };
    double doubles[4] = { 10.0, 20.0, 30.0, 40.0 };
    float float_min = 10.0f;
    float float_max = 80.0f;
    int int_min = 10;
    int int_max = 80;
    float angle = 0.0f;
    bool enabled = true;
    int flags = 0;
    unsigned int unsigned_flags = 0;
    int radio = 0;
    bool selected = false;
    int combo_index = 0;
    int list_index = 0;
    float color[4] = { 0.82f, 0.25f, 0.25f, 1.0f };
    float progress = 0.65f;
    float samples[8] = { 0.1f, 0.5f, 0.3f, 0.8f, 0.6f, 0.9f, 0.4f, 0.7f };
    ImTextureRef texture{};

    void draw_content()
    {
        const double lower = 0.0;
        const double upper = 100.0;
        const double step = 0.1;
        const char* items[] = { "First", "Second", "Third" };
        const ImVec2 vertical_size(
            ImGui::GetFrameHeight() * 2.0f,
            ImGui::GetFrameHeight() * 4.0f);

        if (ImGui::BeginCategory("Native ImGui outputs###output_gallery"))
        {
            ImGui::CreateOption({ .title = "InputText", .output = [&] { return ImGui::InputText("##Value", text, sizeof(text)); } });
            ImGui::CreateOption({ .title = "InputTextWithHint", .output = [&] { return ImGui::InputTextWithHint("##Value", "Enter a name", hint, sizeof(hint)); } });
            ImGui::CreateOption({ .title = "InputTextMultiline", .output = [&] { return ImGui::InputTextMultiline("##Value", multiline, sizeof(multiline), ImVec2(-FLT_MIN, ImGui::GetTextLineHeight() * 4.0f)); } });
            ImGui::CreateOption({ .title = "DragFloat", .output = [&] { return ImGui::DragFloat("##Value", floats, 0.01f, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "DragFloat2", .output = [&] { return ImGui::DragFloat2("##Value", floats, 0.01f, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "DragFloat3", .output = [&] { return ImGui::DragFloat3("##Value", floats, 0.01f, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "DragFloat4", .output = [&] { return ImGui::DragFloat4("##Value", floats, 0.01f, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "DragInt", .output = [&] { return ImGui::DragInt("##Value", integers, 1.0f, 0, 100); } });
            ImGui::CreateOption({ .title = "DragInt2", .output = [&] { return ImGui::DragInt2("##Value", integers, 1.0f, 0, 100); } });
            ImGui::CreateOption({ .title = "DragInt3", .output = [&] { return ImGui::DragInt3("##Value", integers, 1.0f, 0, 100); } });
            ImGui::CreateOption({ .title = "DragInt4", .output = [&] { return ImGui::DragInt4("##Value", integers, 1.0f, 0, 100); } });
            ImGui::CreateOption({ .title = "SliderFloat", .output = [&] { return ImGui::SliderFloat("##Value", floats, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "SliderFloat2", .output = [&] { return ImGui::SliderFloat2("##Value", floats, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "SliderFloat3", .output = [&] { return ImGui::SliderFloat3("##Value", floats, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "SliderFloat4", .output = [&] { return ImGui::SliderFloat4("##Value", floats, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "SliderInt", .output = [&] { return ImGui::SliderInt("##Value", integers, 0, 100); } });
            ImGui::CreateOption({ .title = "SliderInt2", .output = [&] { return ImGui::SliderInt2("##Value", integers, 0, 100); } });
            ImGui::CreateOption({ .title = "SliderInt3", .output = [&] { return ImGui::SliderInt3("##Value", integers, 0, 100); } });
            ImGui::CreateOption({ .title = "SliderInt4", .output = [&] { return ImGui::SliderInt4("##Value", integers, 0, 100); } });
            ImGui::CreateOption({ .title = "InputFloat", .output = [&] { return ImGui::InputFloat("##Value", floats); } });
            ImGui::CreateOption({ .title = "InputFloat2", .output = [&] { return ImGui::InputFloat2("##Value", floats); } });
            ImGui::CreateOption({ .title = "InputFloat3", .output = [&] { return ImGui::InputFloat3("##Value", floats); } });
            ImGui::CreateOption({ .title = "InputFloat4", .output = [&] { return ImGui::InputFloat4("##Value", floats); } });
            ImGui::CreateOption({ .title = "InputInt", .output = [&] { return ImGui::InputInt("##Value", integers); } });
            ImGui::CreateOption({ .title = "InputInt2", .output = [&] { return ImGui::InputInt2("##Value", integers); } });
            ImGui::CreateOption({ .title = "InputInt3", .output = [&] { return ImGui::InputInt3("##Value", integers); } });
            ImGui::CreateOption({ .title = "InputInt4", .output = [&] { return ImGui::InputInt4("##Value", integers); } });
            ImGui::CreateOption({ .title = "DragFloatRange2", .output = [&] { return ImGui::DragFloatRange2("##Value", &float_min, &float_max, 0.1f, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "DragIntRange2", .output = [&] { return ImGui::DragIntRange2("##Value", &int_min, &int_max, 1.0f, 0, 100); } });
            ImGui::CreateOption({ .title = "DragScalar", .output = [&] { return ImGui::DragScalar("##Value", ImGuiDataType_Double, doubles, 0.1f, &lower, &upper); } });
            ImGui::CreateOption({ .title = "DragScalarN", .output = [&] { return ImGui::DragScalarN("##Value", ImGuiDataType_Double, doubles, 4, 0.1f, &lower, &upper); } });
            ImGui::CreateOption({ .title = "SliderScalar", .output = [&] { return ImGui::SliderScalar("##Value", ImGuiDataType_Double, doubles, &lower, &upper); } });
            ImGui::CreateOption({ .title = "SliderScalarN", .output = [&] { return ImGui::SliderScalarN("##Value", ImGuiDataType_Double, doubles, 4, &lower, &upper); } });
            ImGui::CreateOption({ .title = "InputDouble", .output = [&] { return ImGui::InputDouble("##Value", doubles); } });
            ImGui::CreateOption({ .title = "InputScalar", .output = [&] { return ImGui::InputScalar("##Value", ImGuiDataType_Double, doubles, &step); } });
            ImGui::CreateOption({ .title = "InputScalarN", .output = [&] { return ImGui::InputScalarN("##Value", ImGuiDataType_Double, doubles, 4, &step); } });
            ImGui::CreateOption({ .title = "SliderAngle", .output = [&] { return ImGui::SliderAngle("##Value", &angle); } });
            ImGui::CreateOption({ .title = "VSliderFloat", .output = [&] { return ImGui::VSliderFloat("##Value", vertical_size, floats, 0.0f, 100.0f); } });
            ImGui::CreateOption({ .title = "VSliderInt", .output = [&] { return ImGui::VSliderInt("##Value", vertical_size, integers, 0, 100); } });
            ImGui::CreateOption({ .title = "VSliderScalar", .output = [&] { return ImGui::VSliderScalar("##Value", vertical_size, ImGuiDataType_Double, doubles, &lower, &upper); } });
            ImGui::CreateOption({ .title = "Checkbox", .output = [&] { return ImGui::Checkbox("##Value", &enabled); } });
            ImGui::CreateOption({ .title = "CheckboxFlags (int)", .output = [&] { return ImGui::CheckboxFlags("##Value", &flags, 1); } });
            ImGui::CreateOption({ .title = "CheckboxFlags (unsigned)", .output = [&] { return ImGui::CheckboxFlags("##Value", &unsigned_flags, 1u); } });
            ImGui::CreateOption({ .title = "RadioButton", .output = [&] {
                bool changed = ImGui::RadioButton("A", &radio, 0);
                ImGui::SameLine();
                changed |= ImGui::RadioButton("B", &radio, 1);
                return changed;
            } });
            ImGui::CreateOption({ .title = "Selectable", .output = [&] { return ImGui::Selectable("Select item", &selected); } });
            ImGui::CreateOption({ .title = "Combo", .output = [&] { return ImGui::Combo("##Value", &combo_index, items, IM_ARRAYSIZE(items)); } });
            ImGui::CreateOption({ .title = "ListBox", .output = [&] { return ImGui::ListBox("##Value", &list_index, items, IM_ARRAYSIZE(items), 3); } });
            ImGui::CreateOption({ .title = "ColorEdit3", .output = [&] { return ImGui::ColorEdit3("##Value", color); } });
            ImGui::CreateOption({ .title = "ColorEdit4", .output = [&] { return ImGui::ColorEdit4("##Value", color); } });
            ImGui::CreateOption({ .title = "ColorPicker3", .output = [&] { return ImGui::ColorPicker3("##Value", color); } });
            ImGui::CreateOption({ .title = "ColorPicker4", .output = [&] { return ImGui::ColorPicker4("##Value", color); } });
            ImGui::CreateOption({ .title = "ColorButton", .output = [&] { return ImGui::ColorButton("##Value", ImVec4(color[0], color[1], color[2], color[3])); } });
            ImGui::CreateOption({ .title = "Button", .output = [&] { return ImGui::Button("Apply"); } });
            ImGui::CreateOption({ .title = "SmallButton", .output = [&] { return ImGui::SmallButton("Reset"); } });
            ImGui::CreateOption({ .title = "ArrowButton", .output = [&] { return ImGui::ArrowButton("##Value", ImGuiDir_Right); } });
            ImGui::CreateOption({ .title = "InvisibleButton", .output = [&] { return ImGui::InvisibleButton("##Value", ImVec2(ImGui::GetContentRegionAvail().x, ImGui::GetFrameHeight())); } });
            ImGui::CreateOption({ .title = "TextLink", .output = [&] { return ImGui::TextLink("Details"); } });
            ImGui::CreateOption({ .title = "TextUnformatted", .output = [&] {
                ImGui::TextUnformatted("Read-only value");
                return false;
            } });
            ImGui::CreateOption({ .title = "ProgressBar", .output = [&] {
                ImGui::ProgressBar(progress);
                return false;
            } });
            ImGui::CreateOption({ .title = "PlotLines", .output = [&] {
                ImGui::PlotLines("##Value", samples, IM_ARRAYSIZE(samples), 0, nullptr,
                    0.0f, 1.0f, ImVec2(0.0f, ImGui::GetFrameHeight() * 3.0f));
                return false;
            } });
            ImGui::CreateOption({ .title = "PlotHistogram", .output = [&] {
                ImGui::PlotHistogram("##Value", samples, IM_ARRAYSIZE(samples), 0, nullptr,
                    0.0f, 1.0f, ImVec2(0.0f, ImGui::GetFrameHeight() * 3.0f));
                return false;
            } });

            // Supply a valid renderer-managed texture to enable these examples.
            if (texture.GetTexID() != ImTextureID_Invalid)
            {
                ImGui::CreateOption({ .title = "Image", .output = [&] {
                    ImGui::Image(texture, ImVec2(64.0f, 64.0f));
                    return false;
                } });
                ImGui::CreateOption({ .title = "ImageWithBg", .output = [&] {
                    ImGui::ImageWithBg(texture, ImVec2(64.0f, 64.0f));
                    return false;
                } });
                ImGui::CreateOption({ .title = "ImageButton", .output = [&] {
                    return ImGui::ImageButton("##Value", texture, ImVec2(32.0f, 32.0f));
                } });
            }
        }
        ImGui::EndCategory();
    }
};
```

Checkbox, numeric, text, selection and color-edit callbacks return their native
changed result. Buttons return their clicked result; this gallery attaches no
engine action to them. InvisibleButton intentionally provides a hit area without
drawing a background. Read-only widgets return false because they have no
changed result.

Image examples appear only after the caller supplies a valid texture. The
renderer retains resource ownership and must keep it valid for rendering.
Do not pass fabricated texture IDs.

These examples have been checked against the repository headers and passed
MSVC C++20 syntax checking with /Zs. They were not linked or interactively tested;
runtime acceptance remains pending.

## Identity

Use "Display title###stable_id" for panels and categories whose visible titles
may change. Distinct panel instances require distinct stable IDs.

Each option creates its own ID scope, allowing controls to reuse "##Value".

By default, an option's title supplies its ID. Supply option.id when titles
repeat or change. An empty title requires an explicit non-empty ID.

## Styling

Panel padding, spacing and corner radius are scoped and restored by EndPanel.

Control height follows the current font size plus twice FramePadding.y.
Adjust FramePadding in EditorStyle for a more compact global theme.
Panel row_spacing controls spacing rather than the height of the control itself.

Categories inherit the active header colors. Their cell padding is derived
from the current ItemSpacing.

## Current limits

The engine Host now supplies registration and Options > Panels reopening for
its diagnostic panels. Those are Editor responsibilities, not features owned by
the ImGui fork. See the
[Host integration](../../../ReaverEngine/docs/architecture/RendererFocusedHost.md#options-bar-and-panel-registration-2026-10-23)
for callback lifetime and visibility ownership. Explicit native-window detachment
remains a later milestone.

Verify resize, collapse, close, maximize/restore, scrolling and control popups
interactively before treating a new integration as accepted.


## Hierarchy rows

`CreateRow` is a native tree row with an optional icon and an immediate children
callback. It does not require BeginCategory and does not open a panel. Expansion
uses the current window's ImGui state storage. Supply a stable ID when labels can
change or repeat. The callback and strings are borrowed only during the call.

```cpp
ImGui::CreateRow({
    .title = "Scene Collection",
    .icon = ICON_FA_LAYER_GROUP, // Caller-owned icon definition/font.
    .output = [&] {
        ImGui::CreateRow({ .title = "Mesh", .icon = ICON_FA_CUBE, .id = "mesh_42", .leaf = true });
    },
    .id = "scene_collection"
});
```

Nested CreateRow calls share visible-row parity in their current window. The
counter resets on the window's first Begin each frame; appended Begin calls
continue it. Colors come from TableRowBg/TableRowBgAlt, Header/HeaderHovered and
the current text style. Icons use the current font; there is no engine dependency.
The normal ImGui clipping rectangle also clips row backgrounds.

```cpp
void draw_node(const Node& node)
{
    const ImGuiRowResult result = ImGui::CreateRow({
        .title = node.name.c_str(),
        .icon = node.icon,
        .output = [&] { for (const Node& child : node.children) draw_node(child); },
        .id = node.stable_id.c_str(),
        .selected = node.selected,
        .leaf = node.children.empty()
    });
    if (result.clicked) select_node(node); // Caller-defined scene operation.
}
```

The second example assumes the caller's Node and select_node API. `clicked` is a
label/row click, excluding arrow expansion. `toggled` reports native expansion
interaction; `open` reports the current tree state and `hovered` the native item.
Leaf nodes never call output. Non-leaf callbacks run only while open. Callers may
use SetNextItemOpen before CreateRow to control expansion externally. Invalid
empty IDs are rejected; use an explicit ID with an empty display title.


## Maximized panel display layers

BeginPanel applies the shared rule to every maximized panel. Ordinary focus still
routes keyboard/input focus, but does not raise a maximized panel over floating
windows. Explicit maximize, reopen and ActivatePanel promote the panel within its
maximized layer. Other maximized panels keep their state and relative order; none
is automatically restored. Restoring a panel returns its saved geometry and normal
floating behavior.

At EndFrame, ImGui keeps maximized panel roots below other visible root windows in
the same ImGui viewport. Existing display order is retained within each layer;
normal native sorting keeps child windows with their parents. This uses the
existing window list and maximize flags, with no second active-panel registry or
per-frame ordering allocations. Native OS window stacking is unaffected.

Call ImGui::ActivatePanel(full_window_name) inside a frame for a previously
submitted open panel. It returns false for unknown/inactive/non-panel windows or
when a modal blocks activation. It expands a collapsed panel, assigns ImGui focus
and raises it within the applicable layer. It does not own panel visibility or
request native OS focus. Editor Host exposes this as Tools > Panels > Activate;
the existing panel visibility checkboxes remain independent.

Verification and remaining native/platform acceptance are recorded in the engine's
EditorPanelsChecklist.md. Headless checks exercise multiple maximized panels,
stable ordering across submission order changes, ordinary focus, explicit
activation, restore/reopen, children, modal blocking and distinct native viewports.
