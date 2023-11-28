---
layout: post
title: "alacritty: a highly customized terminal for linux/macos (terminal)"
author: "melon"
date: 2023-06-24 13:41
categories: "2023"
tags:
  - terminal
---

### # configurations: .alacritty.yml (macos)
the config file is fine with alacritty version: 0.12.1.  
put the config file under '/Users/mac/.alacritty.yml' and reload alacritty to view the result.
```text
# Configuration for Alacritty, the GPU enhanced terminal emulator.

# import:                        # could accept multiple imports
#   - ${dir}/alacritty.yml       # dir should start with '~' or '/'

# Any items in the `env` entry below will be added as
# environment variables. Some entries may override variables
# set by alacritty itself.
#env:
  # TERM variable
  #
  # This value is used to set the `$TERM` environment variable for
  # each instance of Alacritty. If it is not present, alacritty will
  # check the local terminfo database and use `alacritty` if it is
  # available, otherwise `xterm-256color` is used.
  #TERM: alacritty

window:
  # Window dimensions (changes require restart)
  #
  # Number of lines/columns (not pixels) in the terminal. Both lines and columns
  # must be non-zero for this to take effect. The number of columns must be at
  # least `2`, while using a value of `0` for columns and lines will fall back
  # to the window manager's recommended size
  #dimensions:
  #  columns: 0
  #  lines: 0

  # Window position (changes require restart)
  #
  # Specified in number of pixels.
  # If the position is not set, the window manager will handle the placement.
  #position:
  #  x: 0
  #  y: 0

  # Window padding (changes require restart)
  #
  # Blank space added around the window in pixels. This padding is scaled
  # by DPI and the specified value is always added at both opposing sides.
  #padding:
  #  x: 0
  #  y: 0

  # Spread additional padding evenly around the terminal content.
  dynamic_padding: false 

  # Window decorations
  # Values for `decorations`:
  #     - full: Borders and title bar
  #     - none: Neither borders nor title bar
  # Values for `decorations` (macOS only):
  #     - transparent: Title bar, transparent background and title bar buttons
  #     - buttonless: Title bar, transparent background and no title bar buttons
  decorations: full 

  # Background opacity
  opacity: 0.95  # (0,1)

  # start up
  startup_mode: Windowed   # Windowed / Maximized / Fullscreen / SimpleFullscreen

  # Window title
  title: BOOMMING MELON

  # Allow terminal applications to change Alacritty's window title.
  dynamic_title: false

  # Window class (Linux/BSD only):
  class:
    # Application instance name
    instance: Alacritty
    # General application class
    general: Alacritty

  # Decorations theme variant
  #
  # Override the variant of the System theme/GTK theme/Wayland client side
  # decorations. Commonly supported values are `Dark`, `Light`, and `None` for
  # auto pick-up. Set this to `None` to use the default theme variant.
  decorations_theme_variant: Light

  # Resize increments
  #
  # Prefer resizing window by discrete steps equal to cell dimensions.
  #resize_increments: false

  # Make `Option` key behave as `Alt` (macOS only):
  #   - OnlyLeft
  #   - OnlyRight
  #   - Both
  #   - None (default)
  #option_as_alt: None

#scrolling:
  # Maximum number of lines in the scrollback buffer.
  # Specifying '0' will disable scrolling.
  #history: 10000

  # Scrolling distance multiplier.
  multiplier: 1

font:
  normal:
    family: Courier
    style: Regular
  bold:
    family: Courier
    style: Bold
  italic:
    family: Courier
    style: Italic
  bold_italic:
    family: Courier
    style: Bold Italic
  size: 12.0

  offset:
    x: 2  # letter spacing
    y: 4  # line spacing

  # Glyph offset determines the locations of the glyphs within their cells with
  # the default being at the bottom. Increasing `x` moves the glyph to the
  # right, increasing `y` moves the glyph upward.
  #glyph_offset:
  #  x: 0
  #  y: 0

  builtin_box_drawing: false  # use built-in font for box drawing chars

draw_bold_text_with_bright_colors: false   # color variants


  # schemes:
  #   onehalfdark:  &onehalfdark
  #     primary:
  #       background: '0x282c34'
  #       foreground: '0xdcdfe4'
  # 
  #     normal:
  #       black: '0x282c34'
  #       red: '0xe06c75'
  #       green: '0x98c379'
  #       yellow: '0xe5c07b'
  #       blue: '0x61afef'
  #       magenta: '0xc678dd'
  #       cyan: '0x56b6c2'
  #       white: '0xdcdfe4'
  # 
  #     bright:
  #       black: '0x282c34'
  #       red: '0xe06c75'
  #       green: '0x98c379'
  #       yellow: '0xe5c07b'
  #       blue: '0x61afef'
  #       magenta: '0xc678dd'
  #       cyan: '0x56b6c2'
  #       white: '0xdcdfe4'
  # 
  #   gruvbox_dark: &gruvbox
  #     primary:
  #       # hard contrast: background: '0x1d2021'
  #       # medium contrast: background: '0x282828'
  #       background: '0x32302f'
  #       foreground: '0xebdbb2'
  # 
  #     normal:
  #       black:   '0x282828'
  #       red:     '0xcc241d'
  #       green:   '0x98971a'
  #       yellow:  '0xd79921'
  #       blue:    '0x458588'
  #       magenta: '0xb16286'
  #       cyan:    '0x689d6a'
  #       white:   '0xa89984'
  # 
  #     bright:
  #       black:   '0x928374'
  #       red:     '0xfb4934'
  #       green:   '0xb8bb26'
  #       yellow:  '0xfabd2f'
  #       blue:    '0x83a598'
  #       magenta: '0xd3869b'
  #       cyan:    '0x8ec07c'
  #       white:   '0xebdbb2'
  # 
  # colors: *onehalfdark


colors:
  primary:
    background: '#c5c8c6'
    foreground: '#1d1f21'

    # Bright and dim foreground colors
    #
    # The dimmed foreground color is calculated automatically if it is not
    # present. If the bright foreground color is not set, or
    # `draw_bold_text_with_bright_colors` is `false`, the normal foreground
    # color will be used.
    #dim_foreground: '#828482'
    #bright_foreground: '#eaeaea'

  cursor:
    text: CellBackground    # or hex color
    cursor: CellForeground  # or hex color

  vi_mode_cursor:           # cursor color in vi
    text: CellBackground
    cursor: CellForeground

  # search colors
  search:
    matches:
      foreground: '#000000'
      background: '#ffffff'
    focused_match:
      foreground: '#ffffff'
      background: '#000000'

  # Keyboard hints
  #hints:
    # First character in the hint label
    #
    # Allowed values are CellForeground/CellBackground, which reference the
    # affected cell, or hexadecimal colors like #ff00ff.
    #start:
    #  foreground: '#1d1f21'
    #  background: '#e9ff5e'

    # All characters after the first one in the hint label
    #
    # Allowed values are CellForeground/CellBackground, which reference the
    # affected cell, or hexadecimal colors like #ff00ff.
    #end:
    #  foreground: '#e9ff5e'
    #  background: '#1d1f21'

  # Line indicator
  #
  # Color used for the indicator displaying the position in history during
  # search and vi mode.
  #
  # By default, these will use the opposing primary color.
  line_indicator:
    foreground: None
    background: None

  # Footer bar
  footer_bar:
    background: '#c5c8c6'
    foreground: '#1d1f21'

  # Mouse Selection colors
  selection:
    text: CellBackground
    background: CellForeground


  # tomorrow
  bright:
    black: '#000000'   # 8
    blue: '#4271ae'    # 12
    cyan: '#3e999f'    # 14
    green: '#718c00'   # 10
    magenta: '#8959a8' # 13
    red: '#c82829'     # 9
    white: '#ffffff'   # 15 f
    yellow: '#eab700'  # 11
  cursor:
    cursor: '#4d4d4c'
    text: '#ffffff'
  normal:
    black: '#000000'   # 0
    blue: '#4271ae'    # 4
    cyan: '#3e999f'    # 6
    green: '#718c00'   # 2
    magenta: '#8959a8' # 5
    red: '#c82829'     # 1
    white: '#000000'   # 7 ffffff
    yellow: '#eab700'  # 3 ctrlg
  primary:
    background: '#ffffff'
    foreground: '#4d4d4c'

  # Indexed Colors
  indexed_colors:
    - { index: 236, color: '#ffffff'}  # fzf window search border, markdown icon
    - { index: 239, color: '#1d1f21'}  # fzf/grep matchup (the focused)
    - { index: 109, color: '#ba1320'}  # grep popup ::<>
    - { index: 161, color: '#000000'}  # fzf popup moving arrow >
    - { index: 254, color: '#000000'}  # fzf popup focus line
    - { index: 110, color: '#000000'}  # fzf popup Rg>
    - { index: 144, color: '#000000'}  # grep popup 1224/2244 (0)
    - { index: 151, color: '#139cff'}  # fzf/grep matchup (the focused)
    - { index: 108, color: '#0c6eff'}  # fzf/grep matchup (not the focused)
    # unchanged colors
    - {index: 16, color: '#000000'}
    - {index: 17, color: '#00004c'}
    - {index: 18, color: '#000074'}
    - {index: 19, color: '#00009f'}
    - {index: 20, color: '#0000CD'}
    - {index: 21, color: '#0000FF'}
    - {index: 22, color: '#0A4F01'}
    - {index: 23, color: '#0A4D4D'}
    - {index: 24, color: '#094C73'}
    - {index: 25, color: '#0949A0'}
    - {index: 26, color: '#0746CE'}
    - {index: 27, color: '#0542FF'}
    - {index: 28, color: '#107802'}
    - {index: 29, color: '#10764D'}
    - {index: 30, color: '#0F7674'}
    - {index: 31, color: '#0F749F'}
    - {index: 32, color: '#0D72CD'}
    - {index: 33, color: '#0C6EFE'}
    - {index: 34, color: '#16A404'}
    - {index: 35, color: '#16A44D'}
    - {index: 36, color: '#15A274'}
    - {index: 37, color: '#14A19F'}
    - {index: 38, color: '#149FCE'}
    - {index: 39, color: '#139CFF'}
    - {index: 40, color: '#1CD406'}
    - {index: 41, color: '#1CD34D'}
    - {index: 42, color: '#1BD374'}
    - {index: 43, color: '#1BD19F'}
    - {index: 44, color: '#1BD1CE'}
    - {index: 45, color: '#1ACEFF'}
    - {index: 46, color: '#22FF08'}
    - {index: 47, color: '#23FF4D'}
    - {index: 48, color: '#22FF75'}
    - {index: 49, color: '#22FF9F'}
    - {index: 50, color: '#22FFCD'}
    - {index: 51, color: '#21FFFF'}
    - {index: 52, color: '#4C0000'}
    - {index: 53, color: '#4B004D'}
    - {index: 54, color: '#4B0074'}
    - {index: 55, color: '#4B009F'}
    - {index: 56, color: '#4B00CD'}
    - {index: 57, color: '#4A00FF'}
    - {index: 58, color: '#4D4E01'}
    - {index: 59, color: '#4C4C4C'}
    - {index: 60, color: '#4C4B74'}
    - {index: 61, color: '#4C489F'}
    - {index: 62, color: '#4C45CE'}
    - {index: 63, color: '#4C41FE'}
    - {index: 64, color: '#4E7702'}
    - {index: 65, color: '#4E764C'}
    - {index: 66, color: '#4D7574'}
    - {index: 67, color: '#4E739F'}
    - {index: 68, color: '#4D71CD'}
    - {index: 69, color: '#4D6DFF'}
    - {index: 70, color: '#50A303'}
    - {index: 71, color: '#50A34C'}
    - {index: 72, color: '#50A274'}
    - {index: 73, color: '#4FA19F'}
    - {index: 74, color: '#4F9FCD'}
    - {index: 75, color: '#4F9DFF'}
    - {index: 76, color: '#52D405'}
    - {index: 77, color: '#52D34D'}
    - {index: 78, color: '#52D374'}
    - {index: 79, color: '#52D1A0'}
    - {index: 80, color: '#52D0CD'}
    - {index: 81, color: '#51CDFF'}
    - {index: 82, color: '#55FF08'}
    - {index: 83, color: '#55FF4D'}
    - {index: 84, color: '#55FF74'}
    - {index: 85, color: '#55FF9F'}
    - {index: 86, color: '#54FFCD'}
    - {index: 87, color: '#55FFFF'}
    - {index: 88, color: '#720002'}
    - {index: 89, color: '#72004C'}
    - {index: 90, color: '#730074'}
    - {index: 91, color: '#72009F'}
    - {index: 92, color: '#7200CE'}
    - {index: 93, color: '#7200FF'}
    - {index: 94, color: '#734D03'}
    - {index: 95, color: '#734B4D'}
    - {index: 96, color: '#734A74'}
    - {index: 97, color: '#73479F'}
    - {index: 98, color: '#7344CD'}
    - {index: 99, color: '#733FFE'}
    - {index: 100, color: '#747604'}
    - {index: 101, color: '#74754D'}
    - {index: 102, color: '#747474'}
    - {index: 103, color: '#7472A0'}
    - {index: 104, color: '#7370CE'}
    - {index: 105, color: '#746CFF'}
    - {index: 106, color: '#76A204'}
    - {index: 107, color: '#75A24D'}
    - {index: 108, color: '#75A174'}
    - {index: 109, color: '#75A0A0'}
    - {index: 110, color: '#759FCD'}
    - {index: 111, color: '#759CFE'}
    - {index: 112, color: '#77D304'}
    - {index: 113, color: '#77D24D'}
    - {index: 114, color: '#78D274'}
    - {index: 115, color: '#77D09F'}
    - {index: 116, color: '#77D0CD'}
    - {index: 117, color: '#77CEFF'}
    - {index: 118, color: '#79FF08'}
    - {index: 119, color: '#79FF4D'}
    - {index: 120, color: '#79FF74'}
    - {index: 121, color: '#79FFA0'}
    - {index: 122, color: '#7AFFCE'}
    - {index: 123, color: '#79FFFF'}
    - {index: 124, color: '#9D0003'}
    - {index: 125, color: '#9D004C'}
    - {index: 126, color: '#9D0074'}
    - {index: 127, color: '#9D009F'}
    - {index: 128, color: '#9C00CE'}
    - {index: 129, color: '#9D00FE'}
    - {index: 130, color: '#9D4C04'}
    - {index: 131, color: '#9E4A4C'}
    - {index: 132, color: '#9E4874'}
    - {index: 133, color: '#9E469F'}
    - {index: 134, color: '#9E42CD'}
    - {index: 135, color: '#9D3DFF'}
    - {index: 136, color: '#9E7505'}
    - {index: 137, color: '#9E754D'}
    - {index: 138, color: '#9E7375'}
    - {index: 139, color: '#9E719F'}
    - {index: 140, color: '#9E6FCE'}
    - {index: 141, color: '#9E6CFF'}
    - {index: 142, color: '#A0A306'}
    - {index: 143, color: '#A0A24D'}
    - {index: 144, color: '#9FA074'}
    - {index: 145, color: '#9F9F9F'}
    - {index: 146, color: '#9F9ECE'}
    - {index: 147, color: '#9F9BFF'}
    - {index: 148, color: '#A1D209'}
    - {index: 149, color: '#A1D14D'}
    - {index: 150, color: '#A1D175'}
    - {index: 151, color: '#A1D09F'}
    - {index: 152, color: '#A0CFCE'}
    - {index: 153, color: '#A0CDFE'}
    - {index: 154, color: '#A3FF0A'}
    - {index: 155, color: '#A2FF4D'}
    - {index: 156, color: '#A2FF75'}
    - {index: 157, color: '#A2FFA0'}
    - {index: 158, color: '#A3FFCE'}
    - {index: 159, color: '#A3FFFE'}
    - {index: 160, color: '#CB0004'}
    - {index: 161, color: '#CB004D'}
    - {index: 162, color: '#CB0075'}
    - {index: 163, color: '#CB009F'}
    - {index: 164, color: '#CB00CD'}
    - {index: 165, color: '#CA00FE'}
    - {index: 166, color: '#CB4A06'}
    - {index: 167, color: '#CB484C'}
    - {index: 168, color: '#CB4774'}
    - {index: 169, color: '#CB449F'}
    - {index: 170, color: '#CB41CE'}
    - {index: 171, color: '#CB3CFF'}
    - {index: 172, color: '#CC7407'}
    - {index: 173, color: '#CB734D'}
    - {index: 174, color: '#CB7174'}
    - {index: 175, color: '#CC70A0'}
    - {index: 176, color: '#CC6DCD'}
    - {index: 177, color: '#CC6BFE'}
    - {index: 178, color: '#CDA207'}
    - {index: 179, color: '#CDA04D'}
    - {index: 180, color: '#CDA075'}
    - {index: 181, color: '#CD9EA0'}
    - {index: 182, color: '#CC9CCE'}
    - {index: 183, color: '#CC9AFF'}
    - {index: 184, color: '#CED108'}
    - {index: 185, color: '#CED14D'}
    - {index: 186, color: '#CED074'}
    - {index: 187, color: '#CECFA0'}
    - {index: 188, color: '#CECECE'}
    - {index: 189, color: '#CDCCFF'}
    - {index: 190, color: '#D0FF0A'}
    - {index: 191, color: '#D0FF4D'}
    - {index: 192, color: '#D0FF74'}
    - {index: 193, color: '#CFFF9F'}
    - {index: 194, color: '#CFFFCD'}
    - {index: 195, color: '#CFFFFF'}
    - {index: 196, color: '#FB0006'}
    - {index: 197, color: '#FB004D'}
    - {index: 198, color: '#FB0074'}
    - {index: 199, color: '#FB009F'}
    - {index: 200, color: '#FB00CE'}
    - {index: 201, color: '#FA00FF'}
    - {index: 202, color: '#FB4808'}
    - {index: 203, color: '#FB454D'}
    - {index: 204, color: '#FA4474'}
    - {index: 205, color: '#F9419E'}
    - {index: 206, color: '#F83DCB'}
    - {index: 207, color: '#F838FB'}
    - {index: 208, color: '#FA7108'}
    - {index: 209, color: '#F9704C'}
    - {index: 210, color: '#FA6F73'}
    - {index: 211, color: '#F96E9D'}
    - {index: 212, color: '#FC6CCE'}
    - {index: 213, color: '#FC68FF'}
    - {index: 214, color: '#FDA009'}
    - {index: 215, color: '#FD9F4D'}
    - {index: 216, color: '#FD9E74'}
    - {index: 217, color: '#FC9C9F'}
    - {index: 218, color: '#FC9BCE'}
    - {index: 219, color: '#FD98FF'}
    - {index: 220, color: '#FED009'}
    - {index: 221, color: '#FED04E'}
    - {index: 222, color: '#FECF74'}
    - {index: 223, color: '#FECF9F'}
    - {index: 224, color: '#FECCCE'}
    - {index: 225, color: '#FDCBFF'}
    - {index: 226, color: '#FFFF0B'}
    - {index: 227, color: '#FFFF4E'}
    - {index: 228, color: '#FFFF74'}
    - {index: 229, color: '#FFFFA0'}
    - {index: 230, color: '#FFFFCE'}
    - {index: 231, color: '#FFFFFF'}
    - {index: 232, color: '#090909'}
    - {index: 233, color: '#0F0F0F'}
    - {index: 234, color: '#151515'}
    - {index: 235, color: '#1D1D1D'}
    - {index: 236, color: '#242424'}
    - {index: 237, color: '#2C2C2C'}
    - {index: 238, color: '#343434'}
    - {index: 239, color: '#3D3D3D'}
    - {index: 240, color: '#464646'}
    - {index: 241, color: '#4F4F4F'}
    - {index: 242, color: '#595959'}
    - {index: 243, color: '#636363'}
    - {index: 244, color: '#6D6D6D'}
    - {index: 245, color: '#777777'}
    - {index: 246, color: '#828282'}
    - {index: 247, color: '#8C8C8C'}
    - {index: 248, color: '#979797'}
    - {index: 249, color: '#A3A3A3'}
    - {index: 250, color: '#AEAEAE'}
    - {index: 251, color: '#BABABA'}
    - {index: 252, color: '#C5C5C5'}
    - {index: 253, color: '#D1D1D1'}
    - {index: 254, color: '#DDDDDD'}
    - {index: 255, color: '#EAEAEA'}

  # Transparent cell backgrounds
  #
  # Whether or not `window.opacity` applies to all cell backgrounds or only to
  # the default background. When set to `true` all cells will be transparent
  # regardless of their background color.
  transparent_background_colors: false

# 248: nvim tree indentation line, code indent-blankline
# Bell
#
# The bell is rung every time the BEL control character is received.
#bell:
  # Visual Bell Animation
  #
  # Animation effect for flashing the screen when the visual bell is rung.
  #
  # Values for `animation`:
  #   - Ease
  #   - EaseOut
  #   - EaseOutSine
  #   - EaseOutQuad
  #   - EaseOutCubic
  #   - EaseOutQuart
  #   - EaseOutQuint
  #   - EaseOutExpo
  #   - EaseOutCirc
  #   - Linear
  #animation: EaseOutExpo

  # Duration of the visual bell flash in milliseconds. A `duration` of `0` will
  # disable the visual bell animation.
  #duration: 0

  # Visual bell animation color.
  #color: '#ffffff'

  # Bell Command
  #
  # This program is executed whenever the bell is rung.
  #
  # When set to `command: None`, no command will be executed.
  #
  # Example:
  #   command:
  #     program: notify-send
  #     args: ["Hello, World!"]
  #
  #command: None

#selection:
  # This string contains all characters that are used as separators for
  # "semantic words" in Alacritty.
  #semantic_escape_chars: ",│`|:\"' ()[]{}<>\t"

  # When set to `true`, selected text will be copied to the primary clipboard.
  #save_to_clipboard: false

cursor:
  style:
    shape: Block   # Underline / Beam
    blinking: Off   # Never / Off / On / Always

  # Vi mode cursor style
  #
  # If the vi mode cursor style is `None` or not specified, it will fall back to
  # the style of the active value of the normal cursor.
  #
  # See `cursor.style` for available options.
  #vi_mode_style: None

  # Cursor blinking interval in milliseconds.
  #blink_interval: 750

  # Time after which cursor stops blinking, in seconds.
  #
  # Specifying '0' will disable timeout for blinking.
  #blink_timeout: 5

  # If this is `true`, the cursor will be rendered as a hollow box when the
  # window is not focused.
  #unfocused_hollow: true

  # Thickness of the cursor relative to the cell width as floating point number
  # from `0.0` to `1.0`.
  #thickness: 0.15

# Live config reload (changes require restart)
#live_config_reload: true

# Shell
#
# You can set `shell.program` to the path of your favorite shell, e.g.
# `/bin/fish`. Entries in `shell.args` are passed unmodified as arguments to the
# shell.
#
# Default:
#   - (Linux/BSD/macOS) `$SHELL` or the user's login shell, if `$SHELL` is unset
#   - (Windows) powershell
#shell:
#  program: /bin/bash
#  args:
#    - --login

# Startup directory
#
# Directory the shell is started in. If this is unset, or `None`, the working
# directory of the parent process will be used.
#working_directory: None

# Offer IPC using `alacritty msg` (unix only)
#ipc_socket: true

#mouse:
  # Click settings
  #
  # The `double_click` and `triple_click` settings control the time
  # alacritty should wait for accepting multiple clicks as one double
  # or triple click.
  #double_click: { threshold: 300 }
  #triple_click: { threshold: 300 }

  # If this is `true`, the cursor is temporarily hidden when typing.
  #hide_when_typing: false

# Hints
#
# Terminal hints can be used to find text or hyperlink in the visible part of
# the terminal and pipe it to other applications.
#hints:
  # Keys used for the hint labels.
  #alphabet: "jfkdls;ahgurieowpq"

  # List with all available hints
  #
  # Each hint must have any of `regex` or `hyperlinks` field and either an
  # `action` or a `command` field. The fields `mouse`, `binding` and
  # `post_processing` are optional.
  #
  # The `hyperlinks` option will cause OSC 8 escape sequence hyperlinks to be
  # highlighted.
  #
  # The fields `command`, `binding.key`, `binding.mods`, `binding.mode` and
  # `mouse.mods` accept the same values as they do in the `key_bindings` section.
  #
  # The `mouse.enabled` field controls if the hint should be underlined while
  # the mouse with all `mouse.mods` keys held or the vi mode cursor is above it.
  #
  # If the `post_processing` field is set to `true`, heuristics will be used to
  # shorten the match if there are characters likely not to be part of the hint
  # (e.g. a trailing `.`). This is most useful for URIs and applies only to
  # `regex` matches.
  #
  # Values for `action`:
  #   - Copy
  #       Copy the hint's text to the clipboard.
  #   - Paste
  #       Paste the hint's text to the terminal or search.
  #   - Select
  #       Select the hint's text.
  #   - MoveViModeCursor
  #       Move the vi mode cursor to the beginning of the hint.
  #enabled:
  # - regex: "(ipfs:|ipns:|magnet:|mailto:|gemini:|gopher:|https:|http:|news:|file:|git:|ssh:|ftp:)\
  #           [^\u0000-\u001F\u007F-\u009F<>\"\\s{-}\\^⟨⟩`]+"
  #   hyperlinks: true
  #   command: xdg-open
  #   post_processing: true
  #   mouse:
  #     enabled: true
  #     mods: None
  #   binding:
  #     key: U
  #     mods: Control|Shift

# Mouse bindings
#
# Mouse bindings are specified as a list of objects, much like the key
# bindings further below.
#
# To trigger mouse bindings when an application running within Alacritty
# captures the mouse, the `Shift` modifier is automatically added as a
# requirement.
#
# Each mouse binding will specify a:
#
# - `mouse`:
#
#   - Middle
#   - Left
#   - Right
#   - Numeric identifier such as `5`
#
# - `action` (see key bindings for actions not exclusive to mouse mode)
#
# - Mouse exclusive actions:
#
#   - ExpandSelection
#       Expand the selection to the current mouse cursor location.
#
# And optionally:
#
# - `mods` (see key bindings)
#mouse_bindings:
#  - { mouse: Right,                 action: ExpandSelection }
#  - { mouse: Right,  mods: Control, action: ExpandSelection }
#  - { mouse: Middle, mode: ~Vi,     action: PasteSelection  }

# Key bindings
#
# Key bindings are specified as a list of objects. For example, this is the
# default paste binding:
#
# `- { key: V, mods: Control|Shift, action: Paste }`
#
# Each key binding will specify a:
#
# - `key`: Identifier of the key pressed
#
#    - A-Z
#    - F1-F24
#    - Key0-Key9
#
#    A full list with available key codes can be found here:
#    https://docs.rs/winit/*/winit/event/enum.VirtualKeyCode.html#variants
#
#    Instead of using the name of the keys, the `key` field also supports using
#    the scancode of the desired key. Scancodes have to be specified as a
#    decimal number. This command will allow you to display the hex scancodes
#    for certain keys:
#
#       `showkey --scancodes`.
#
# Then exactly one of:
#
# - `chars`: Send a byte sequence to the running application
#
#    The `chars` field writes the specified string to the terminal. This makes
#    it possible to pass escape sequences. To find escape codes for bindings
#    like `PageUp` (`"\x1b[5~"`), you can run the command `showkey -a` outside
#    of tmux. Note that applications use terminfo to map escape sequences back
#    to keys. It is therefore required to update the terminfo when changing an
#    escape sequence.
#
# - `action`: Execute a predefined action
#
#   - ToggleViMode
#   - SearchForward
#       Start searching toward the right of the search origin.
#   - SearchBackward
#       Start searching toward the left of the search origin.
#   - Copy
#   - Paste
#   - IncreaseFontSize
#   - DecreaseFontSize
#   - ResetFontSize
#   - ScrollPageUp
#   - ScrollPageDown
#   - ScrollHalfPageUp
#   - ScrollHalfPageDown
#   - ScrollLineUp
#   - ScrollLineDown
#   - ScrollToTop
#   - ScrollToBottom
#   - ClearHistory
#       Remove the terminal's scrollback history.
#   - Hide
#       Hide the Alacritty window.
#   - Minimize
#       Minimize the Alacritty window.
#   - Quit
#       Quit Alacritty.
#   - ToggleFullscreen
#   - ToggleMaximized
#   - SpawnNewInstance
#       Spawn a new instance of Alacritty.
#   - CreateNewWindow
#       Create a new Alacritty window from the current process.
#   - ClearLogNotice
#       Clear Alacritty's UI warning and error notice.
#   - ClearSelection
#       Remove the active selection.
#   - ReceiveChar
#   - None
#
# - Vi mode exclusive actions:
#
#   - Open
#       Perform the action of the first matching hint under the vi mode cursor
#       with `mouse.enabled` set to `true`.
#   - ToggleNormalSelection
#   - ToggleLineSelection
#   - ToggleBlockSelection
#   - ToggleSemanticSelection
#       Toggle semantic selection based on `selection.semantic_escape_chars`.
#   - CenterAroundViCursor
#       Center view around vi mode cursor
#
# - Vi mode exclusive cursor motion actions:
#
#   - Up
#       One line up.
#   - Down
#       One line down.
#   - Left
#       One character left.
#   - Right
#       One character right.
#   - First
#       First column, or beginning of the line when already at the first column.
#   - Last
#       Last column, or beginning of the line when already at the last column.
#   - FirstOccupied
#       First non-empty cell in this terminal row, or first non-empty cell of
#       the line when already at the first cell of the row.
#   - High
#       Top of the screen.
#   - Middle
#       Center of the screen.
#   - Low
#       Bottom of the screen.
#   - SemanticLeft
#       Start of the previous semantically separated word.
#   - SemanticRight
#       Start of the next semantically separated word.
#   - SemanticLeftEnd
#       End of the previous semantically separated word.
#   - SemanticRightEnd
#       End of the next semantically separated word.
#   - WordLeft
#       Start of the previous whitespace separated word.
#   - WordRight
#       Start of the next whitespace separated word.
#   - WordLeftEnd
#       End of the previous whitespace separated word.
#   - WordRightEnd
#       End of the next whitespace separated word.
#   - Bracket
#       Character matching the bracket at the cursor's location.
#   - SearchNext
#       Beginning of the next match.
#   - SearchPrevious
#       Beginning of the previous match.
#   - SearchStart
#       Start of the match to the left of the vi mode cursor.
#   - SearchEnd
#       End of the match to the right of the vi mode cursor.
#
# - Search mode exclusive actions:
#   - SearchFocusNext
#       Move the focus to the next search match.
#   - SearchFocusPrevious
#       Move the focus to the previous search match.
#   - SearchConfirm
#   - SearchCancel
#   - SearchClear
#       Reset the search regex.
#   - SearchDeleteWord
#       Delete the last word in the search regex.
#   - SearchHistoryPrevious
#       Go to the previous regex in the search history.
#   - SearchHistoryNext
#       Go to the next regex in the search history.
#
# - macOS exclusive actions:
#   - ToggleSimpleFullscreen
#       Enter fullscreen without occupying another space.
#
# - Linux/BSD exclusive actions:
#
#   - CopySelection
#       Copy from the selection buffer.
#   - PasteSelection
#       Paste from the selection buffer.
#
# - `command`: Fork and execute a specified command plus arguments
#
#    The `command` field must be a map containing a `program` string and an
#    `args` array of command line parameter strings. For example:
#       `{ program: "alacritty", args: ["-e", "vttest"] }`
#
# And optionally:
#
# - `mods`: Key modifiers to filter binding actions
#
#    - Command
#    - Control
#    - Option
#    - Super
#    - Shift
#    - Alt
#
#    Multiple `mods` can be combined using `|` like this:
#       `mods: Control|Shift`.
#    Whitespace and capitalization are relevant and must match the example.
#
# - `mode`: Indicate a binding for only specific terminal reported modes
#
#    This is mainly used to send applications the correct escape sequences
#    when in different modes.
#
#    - AppCursor
#    - AppKeypad
#    - Search
#    - Alt
#    - Vi
#
#    A `~` operator can be used before a mode to apply the binding whenever
#    the mode is *not* active, e.g. `~Alt`.
#
# Bindings are always filled by default, but will be replaced when a new
# binding with the same triggers is defined. To unset a default binding, it can
# be mapped to the `ReceiveChar` action. Alternatively, you can use `None` for
# a no-op if you do not wish to receive input characters for that binding.
#
# If the same trigger is assigned to multiple actions, all of them are executed
# in the order they were defined in.
#key_bindings:
  #- { key: Paste,                                       action: Paste          }
  #- { key: Copy,                                        action: Copy           }
  #- { key: L,         mods: Control,                    action: ClearLogNotice }
  #- { key: L,         mods: Control, mode: ~Vi|~Search, chars: "\x0c"          }
  #- { key: PageUp,    mods: Shift,   mode: ~Alt,        action: ScrollPageUp   }
  #- { key: PageDown,  mods: Shift,   mode: ~Alt,        action: ScrollPageDown }
  #- { key: Home,      mods: Shift,   mode: ~Alt,        action: ScrollToTop    }
  #- { key: End,       mods: Shift,   mode: ~Alt,        action: ScrollToBottom }

  # Vi Mode
  #- { key: Space,  mods: Shift|Control, mode: ~Search,    action: ToggleViMode            }
  #- { key: Space,  mods: Shift|Control, mode: Vi|~Search, action: ScrollToBottom          }
  #- { key: Escape,                      mode: Vi|~Search, action: ClearSelection          }
  #- { key: I,                           mode: Vi|~Search, action: ToggleViMode            }
  #- { key: I,                           mode: Vi|~Search, action: ScrollToBottom          }
  #- { key: C,      mods: Control,       mode: Vi|~Search, action: ToggleViMode            }
  #- { key: Y,      mods: Control,       mode: Vi|~Search, action: ScrollLineUp            }
  #- { key: E,      mods: Control,       mode: Vi|~Search, action: ScrollLineDown          }
  #- { key: G,                           mode: Vi|~Search, action: ScrollToTop             }
  #- { key: G,      mods: Shift,         mode: Vi|~Search, action: ScrollToBottom          }
  #- { key: B,      mods: Control,       mode: Vi|~Search, action: ScrollPageUp            }
  #- { key: F,      mods: Control,       mode: Vi|~Search, action: ScrollPageDown          }
  #- { key: U,      mods: Control,       mode: Vi|~Search, action: ScrollHalfPageUp        }
  #- { key: D,      mods: Control,       mode: Vi|~Search, action: ScrollHalfPageDown      }
  #- { key: Y,                           mode: Vi|~Search, action: Copy                    }
  #- { key: Y,                           mode: Vi|~Search, action: ClearSelection          }
  #- { key: Copy,                        mode: Vi|~Search, action: ClearSelection          }
  #- { key: V,                           mode: Vi|~Search, action: ToggleNormalSelection   }
  #- { key: V,      mods: Shift,         mode: Vi|~Search, action: ToggleLineSelection     }
  #- { key: V,      mods: Control,       mode: Vi|~Search, action: ToggleBlockSelection    }
  #- { key: V,      mods: Alt,           mode: Vi|~Search, action: ToggleSemanticSelection }
  #- { key: Return,                      mode: Vi|~Search, action: Open                    }
  #- { key: Z,                           mode: Vi|~Search, action: CenterAroundViCursor    }
  #- { key: K,                           mode: Vi|~Search, action: Up                      }
  #- { key: J,                           mode: Vi|~Search, action: Down                    }
  #- { key: H,                           mode: Vi|~Search, action: Left                    }
  #- { key: L,                           mode: Vi|~Search, action: Right                   }
  #- { key: Up,                          mode: Vi|~Search, action: Up                      }
  #- { key: Down,                        mode: Vi|~Search, action: Down                    }
  #- { key: Left,                        mode: Vi|~Search, action: Left                    }
  #- { key: Right,                       mode: Vi|~Search, action: Right                   }
  #- { key: Key0,                        mode: Vi|~Search, action: First                   }
  #- { key: Key4,   mods: Shift,         mode: Vi|~Search, action: Last                    }
  #- { key: Key6,   mods: Shift,         mode: Vi|~Search, action: FirstOccupied           }
  #- { key: H,      mods: Shift,         mode: Vi|~Search, action: High                    }
  #- { key: M,      mods: Shift,         mode: Vi|~Search, action: Middle                  }
  #- { key: L,      mods: Shift,         mode: Vi|~Search, action: Low                     }
  #- { key: B,                           mode: Vi|~Search, action: SemanticLeft            }
  #- { key: W,                           mode: Vi|~Search, action: SemanticRight           }
  #- { key: E,                           mode: Vi|~Search, action: SemanticRightEnd        }
  #- { key: B,      mods: Shift,         mode: Vi|~Search, action: WordLeft                }
  #- { key: W,      mods: Shift,         mode: Vi|~Search, action: WordRight               }
  #- { key: E,      mods: Shift,         mode: Vi|~Search, action: WordRightEnd            }
  #- { key: Key5,   mods: Shift,         mode: Vi|~Search, action: Bracket                 }
  #- { key: Slash,                       mode: Vi|~Search, action: SearchForward           }
  #- { key: Slash,  mods: Shift,         mode: Vi|~Search, action: SearchBackward          }
  #- { key: N,                           mode: Vi|~Search, action: SearchNext              }
  #- { key: N,      mods: Shift,         mode: Vi|~Search, action: SearchPrevious          }

  # Search Mode
  #- { key: Return,                mode: Search|Vi,  action: SearchConfirm         }
  #- { key: Escape,                mode: Search,     action: SearchCancel          }
  #- { key: C,      mods: Control, mode: Search,     action: SearchCancel          }
  #- { key: U,      mods: Control, mode: Search,     action: SearchClear           }
  #- { key: W,      mods: Control, mode: Search,     action: SearchDeleteWord      }
  #- { key: P,      mods: Control, mode: Search,     action: SearchHistoryPrevious }
  #- { key: N,      mods: Control, mode: Search,     action: SearchHistoryNext     }
  #- { key: Up,                    mode: Search,     action: SearchHistoryPrevious }
  #- { key: Down,                  mode: Search,     action: SearchHistoryNext     }
  #- { key: Return,                mode: Search|~Vi, action: SearchFocusNext       }
  #- { key: Return, mods: Shift,   mode: Search|~Vi, action: SearchFocusPrevious   }

  # (Windows, Linux, and BSD only)
  #- { key: V,              mods: Control|Shift, mode: ~Vi,        action: Paste            }
  #- { key: C,              mods: Control|Shift,                   action: Copy             }
  #- { key: F,              mods: Control|Shift, mode: ~Search,    action: SearchForward    }
  #- { key: B,              mods: Control|Shift, mode: ~Search,    action: SearchBackward   }
  #- { key: C,              mods: Control|Shift, mode: Vi|~Search, action: ClearSelection   }
  #- { key: Insert,         mods: Shift,                           action: PasteSelection   }
  #- { key: Key0,           mods: Control,                         action: ResetFontSize    }
  #- { key: Equals,         mods: Control,                         action: IncreaseFontSize }
  #- { key: Plus,           mods: Control,                         action: IncreaseFontSize }
  #- { key: NumpadAdd,      mods: Control,                         action: IncreaseFontSize }
  #- { key: Minus,          mods: Control,                         action: DecreaseFontSize }
  #- { key: NumpadSubtract, mods: Control,                         action: DecreaseFontSize }

  # (Windows only)
  #- { key: Return,   mods: Alt,           action: ToggleFullscreen }

  # (macOS only)
  #- { key: K,              mods: Command, mode: ~Vi|~Search, chars: "\x0c"                 }
  #- { key: K,              mods: Command, mode: ~Vi|~Search, action: ClearHistory          }
  #- { key: Key0,           mods: Command,                    action: ResetFontSize         }
  #- { key: Equals,         mods: Command,                    action: IncreaseFontSize      }
  #- { key: Plus,           mods: Command,                    action: IncreaseFontSize      }
  #- { key: NumpadAdd,      mods: Command,                    action: IncreaseFontSize      }
  #- { key: Minus,          mods: Command,                    action: DecreaseFontSize      }
  #- { key: NumpadSubtract, mods: Command,                    action: DecreaseFontSize      }
  #- { key: V,              mods: Command,                    action: Paste                 }
  #- { key: C,              mods: Command,                    action: Copy                  }
  #- { key: C,              mods: Command, mode: Vi|~Search,  action: ClearSelection        }
  #- { key: H,              mods: Command,                    action: Hide                  }
  #- { key: H,              mods: Command|Alt,                action: HideOtherApplications }
  #- { key: M,              mods: Command,                    action: Minimize              }
  #- { key: Q,              mods: Command,                    action: Quit                  }
  #- { key: W,              mods: Command,                    action: Quit                  }
  #- { key: N,              mods: Command,                    action: CreateNewWindow       }
  #- { key: F,              mods: Command|Control,            action: ToggleFullscreen      }
  #- { key: F,              mods: Command, mode: ~Search,     action: SearchForward         }
  #- { key: B,              mods: Command, mode: ~Search,     action: SearchBackward        }

debug:
  # Display the time it takes to redraw each frame.
  render_timer: false

  # Keep the log file after quitting Alacritty.
  #persistent_logging: false

  # Log level
  #
  # Values for `log_level`:
  #   - Off
  #   - Error
  #   - Warn
  #   - Info
  #   - Debug
  #   - Trace
  #log_level: Warn

  # Renderer override.
  #   - glsl3
  #   - gles2
  #   - gles2_pure
  #renderer: None

  # Print all received window events.
  #print_events: false

  # Highlight window damage information.
  #highlight_damage: false
```

<hr>

### # reference
https://github.com/alacritty/alacritty  
https://github.com/mbadolato/iTerm2-Color-Schemes  
https://github.com/mbadolato/iTerm2-Color-Schemes/blob/master/alacritty/Tomorrow.yml  
