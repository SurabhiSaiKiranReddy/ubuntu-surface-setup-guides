# Developer Typography: Fira Code Retina in Ghostty and VS Code

This guide documents the developer-font configuration finalized on the 15-inch Intel Surface Laptop 4. It uses Fira Code Retina for code-oriented surfaces while retaining Ubuntu Sans for application controls and long-form text.

The result is intentionally mixed rather than applying a monospaced font everywhere:

| Surface | Final font or behavior |
|---|---|
| VS Code editor and line numbers | Fira Code, weight `450` (Retina) |
| VS Code integrated terminal | Fira Code, weight `450` (Retina) |
| VS Code Copilot Chat code blocks | Fira Code, weight `450` (Retina) |
| VS Code Copilot Chat prose | Default workbench/system UI font |
| VS Code tabs, Explorer, menus, and Settings | Default workbench/system UI font |
| Ghostty terminal | Fira Code, Retina style |
| GNOME and other application interfaces | Ubuntu Sans/system UI font |

This follows a common desktop typography pattern: use a proportional font for controls and paragraphs, and a monospaced font for code and terminal cells. Fira Code is wider and denser than Ubuntu Sans, so applying it to long explanations or the entire interface is usually less readable.

## Scope and limitations

- OS when documented: Ubuntu 26.04 LTS
- Desktop: GNOME Shell 50.1 on Wayland
- VS Code uses supported settings only; no custom CSS or font-injection extension is required.
- VS Code does not provide a supported `workbench.fontFamily` setting. Explorer, tabs, menus, the Command Palette, and Settings continue to use the system UI font.
- Webviews and rendered Markdown previews can define their own fonts.
- Emoji, icons, and characters absent from Fira Code use fallback fonts or VS Code's custom glyph renderer.
- Font family and display scaling are independent. See [Ubuntu Picture-Perfect Scaling](surface-ubuntu-picture-perfect-scaling.md) for the selected native resolution, GNOME scale, and VS Code application multiplier.

## 1. Install Fira Code for one user

Download the current Fira Code release archive from the project's official GitHub releases page. Replace `Fira_Code_vX.Y.zip` below with the downloaded filename.

Install the TrueType faces into the per-user font directory:

```bash
archive="$HOME/Downloads/Fira_Code_vX.Y.zip"
temp_dir="$(mktemp -d)"

unzip "$archive" -d "$temp_dir"
mkdir -p ~/.local/share/fonts/firacode
install -m 0644 "$temp_dir"/ttf/*.ttf \
  ~/.local/share/fonts/firacode/
fc-cache -f ~/.local/share/fonts
rm -rf "$temp_dir"
```

This keeps package-owned font directories unchanged. The tested installation contains these faces:

```text
FiraCode-Light.ttf
FiraCode-Regular.ttf
FiraCode-Retina.ttf
FiraCode-Medium.ttf
FiraCode-SemiBold.ttf
FiraCode-Bold.ttf
```

Confirm that the Retina face resolves to the installed file:

```bash
fc-match -f 'family=%{family}\nstyle=%{style}\nfile=%{file}\n' \
  'Fira Code:style=Retina'
```

Expected family and style:

```text
family=Fira Code
style=Retina,Regular
```

The file should end in `FiraCode-Retina.ttf`. Fontconfig and CSS use different numeric weight scales, so do not test the VS Code CSS weight by passing `weight=450` directly to `fc-match`. Match the `Retina` style instead.

## 2. Configure Ghostty

Create or update:

```text
~/.config/ghostty/config
```

The finalized appearance settings are:

```ini
theme = Vesper

font-family = Fira Code
font-style = Retina
font-size = 13

window-padding-x = 12
window-padding-y = 10
window-padding-balance = true

background-opacity = 0.96
background-blur = true

cursor-style = bar
cursor-style-blink = true

shell-integration = detect
shell-integration-features = cursor,sudo,title
```

Validate the effective configuration:

```bash
ghostty +show-config | grep -E \
  '^(theme|font-family|font-style|font-size|background-opacity|background-blur)'
```

Expected typography lines:

```text
font-family = Fira Code
font-style = Retina
font-size = 13
```

Restart Ghostty or reload its configuration after changing the file.

## 3. Configure the VS Code editor and terminal

Open **Preferences: Open User Settings (JSON)** and add:

```jsonc
{
    "editor.fontFamily": "'Fira Code', monospace",
    "editor.fontLigatures": true,
    "editor.fontWeight": "450",
    "editor.fontSize": 15,
    "terminal.integrated.fontFamily": "Fira Code",
    "terminal.integrated.fontWeight": "450"
}
```

Fira Code Retina is the face between Regular (`400`) and Medium (`500`). The `450` CSS weight selects that installed face in the VS Code editor and integrated terminal.

The `monospace` fallback in `editor.fontFamily` prevents missing text if Fira Code is unavailable. The terminal can also use a fallback list if prompt symbols require another font:

```jsonc
"terminal.integrated.fontFamily": "'Fira Code', monospace"
```

Changing the font does not install Powerline or Nerd Font symbols. VS Code draws some terminal symbols itself, but a prompt that uses a broader icon set may still need a Nerd Font fallback.

## 4. Configure Copilot Chat code blocks

VS Code has native settings for chat typography. The finalized setup explicitly applies Fira Code Retina to code blocks while leaving explanatory prose in the proportional system UI font:

```jsonc
{
    "chat.editor.fontFamily": "Fira Code",
    "chat.editor.fontWeight": "450"
}
```

This affects code blocks rendered inside Copilot Chat. The main `editor.fontLigatures` setting supplies the code-block ligature preference.

VS Code also supports changing ordinary chat messages. This is optional and was not selected for the final setup:

```jsonc
"chat.fontFamily": "'Fira Code Retina', 'Fira Code', monospace"
```

Using Fira Code for all chat prose is valid, but the monospaced letters consume more horizontal space and make long responses denser. Keep `chat.fontFamily` unset or set it to `default` to retain the workbench font:

```jsonc
"chat.fontFamily": "default"
```

There is no native setting that changes every VS Code surface at once. Avoid custom-CSS injection unless accepting that VS Code updates can break it.

## 5. Optional native VS Code surfaces

Other code-oriented surfaces can be configured independently when needed. For example, the Debug Console supports:

```jsonc
"debug.console.fontFamily": "Fira Code"
```

These settings do not change tabs, the Explorer, menus, notifications, Settings, or other workbench controls.

## 6. Apply and verify

Run **Developer: Reload Window** after editing VS Code settings if an existing editor or chat view does not refresh immediately.

Check the configured values:

```bash
grep -nE \
  'editor\.font|terminal\.integrated\.font|chat\.editor\.font|chat\.fontFamily' \
  ~/.config/Code/User/settings.json
```

Check the installed Retina face separately:

```bash
fc-match -f '%{family}|%{style}|%{file}\n' \
  'Fira Code:style=Retina'
```

A correct result ends with:

```text
Fira Code|Retina,Regular|.../FiraCode-Retina.ttf
```

Visual checks:

1. Open a source file containing operators such as `->`, `=>`, `!=`, and `>=` to confirm editor ligatures.
2. Open the integrated terminal and compare its normal text weight with the editor.
3. Open Copilot Chat and inspect a fenced code block separately from response prose.
4. Open Ghostty and confirm its effective configuration reports the Retina style.

Do not use the VS Code interface font as evidence that the editor setting failed. Workbench controls deliberately retain the system UI font.

## Rollback

Remove these entries from VS Code user settings to return each surface to its default:

```jsonc
"editor.fontFamily": "'Fira Code', monospace",
"editor.fontLigatures": true,
"editor.fontWeight": "450",
"editor.fontSize": 15,
"terminal.integrated.fontFamily": "Fira Code",
"terminal.integrated.fontWeight": "450",
"chat.editor.fontFamily": "Fira Code",
"chat.editor.fontWeight": "450"
```

Remove or replace the `font-family`, `font-style`, and `font-size` lines in `~/.config/ghostty/config` to restore Ghostty's defaults. Keep unrelated theme, padding, cursor, and shell-integration settings unless those should also be rolled back.

If the font files were installed only for this guide and are no longer needed:

```bash
rm -rf ~/.local/share/fonts/firacode
fc-cache -f ~/.local/share/fonts
```

Reload VS Code and restart Ghostty after rollback.

## Final recommendation

Keep the following split unless personal readability preferences change:

```text
Code and terminal text:      Fira Code Retina
Copilot Chat code blocks:    Fira Code Retina
Copilot Chat prose:          Ubuntu Sans/system UI font
VS Code workbench controls:  Ubuntu Sans/system UI font
Ghostty theme:               Vesper
Ghostty font size:           13
VS Code editor font size:    15
```

This gives programming surfaces consistent character widths and a clear high-DPI weight without forcing a monospaced font onto navigation controls and paragraph-heavy content.
