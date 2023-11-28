---
layout: post
title: "vim - Spacevim ColorPallete"
subtitle: 'ColorPallete For SpaceVim'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-27 16:16
lang: ch 
catalog: true 
categories: vim 
tags:
  - Time 2021
---
### The instructions
**[Run:]`:help colorscheme` to open theme-adjust plugin.** <br>
**[Run:]`:hi[ghlight]` to list the highlight groups with attributes set.** <br>
**[The Code Block]: Just copy the default highlight groups(which cannot edit using vim) for easies to customize your own color scheme.** <br>
**[1] The code block consist of the default color configs for gruvbox theme.** <br>
**[2] ctermfg & ctermbg are xterm-256color settings.** <br>
**[3] guifg & guibg are true colors used for the terminal which support them.** <br>
**[4] The phrase 'links to' is something like 'inherit'** <br>

### The Cpp Color Pallete
<center><img src="/img/in-post/vim_img/colorpallete_1.pdf" width="100%"></center>

### The Code Block
```c++
SpecialKey     xxx ctermfg=81 guifg=Cyan
                   links to GruvboxBg2
EndOfBuffer    xxx ctermfg=236 ctermbg=236
TermCursor     xxx cterm=reverse gui=reverse
TermCursorNC   xxx cleared
NonText        xxx ctermfg=12 gui=bold guifg=Blue
                   links to GruvboxBg2
Directory      xxx ctermfg=159 guifg=Cyan
                   links to GruvboxGreenBold
ErrorMsg       xxx cterm=bold ctermfg=236 ctermbg=167 gui=bold guifg=#32302f guibg=#fb4934
IncSearch      xxx cterm=reverse ctermfg=208 ctermbg=236 gui=reverse guifg=#fe8019 guibg=#32302f
Search         xxx cterm=reverse ctermfg=214 ctermbg=236 gui=reverse guifg=#fabd2f guibg=#32302f
MoreMsg        xxx ctermfg=121 gui=bold guifg=SeaGreen
                   links to GruvboxYellowBold
ModeMsg        xxx cterm=bold gui=bold
                   links to GruvboxYellowBold
LineNr         xxx ctermfg=243 guifg=#7c6f64
CursorLineNr   xxx ctermfg=214 ctermbg=237 guifg=#fabd2f guibg=#3c3836
Question       xxx ctermfg=121 gui=bold guifg=Green
                  links to GruvboxOrangeBold
StatusLine     xxx cterm=reverse ctermfg=239 ctermbg=223 gui=reverse guifg=#504945 guibg=#ebdbb2
StatusLineNC   xxx cterm=reverse ctermfg=237 ctermbg=246 gui=reverse guifg=#3c3836 guibg=#a89984
VertSplit      xxx ctermfg=241 ctermbg=236 guifg=#181A1F guibg=#282828
Title          xxx ctermfg=225 gui=bold guifg=Magenta
                   links to GruvboxGreenBold
Visual         xxx ctermbg=241 guibg=#665c54
VisualNC       xxx cleared
WarningMsg     xxx ctermfg=224 guifg=Red
                   links to GruvboxRedBold
WildMenu       xxx cterm=bold ctermfg=109 ctermbg=239 gui=bold guifg=#83a598 guibg=#504945
Folded         xxx ctermfg=245 ctermbg=237 guifg=#928374 guibg=#3c3836
FoldColumn     xxx ctermfg=245 ctermbg=237 guifg=#928374 guibg=#3c3836
DiffAdd        xxx cterm=reverse ctermfg=142 ctermbg=236 gui=reverse guifg=#b8bb26 guibg=#32302f
DiffChange     xxx cterm=reverse ctermfg=108 ctermbg=236 gui=reverse guifg=#8ec07c guibg=#32302f
DiffDelete     xxx cterm=reverse ctermfg=167 ctermbg=236 gui=reverse guifg=#fb4934 guibg=#32302f
DiffText       xxx cterm=reverse ctermfg=214 ctermbg=236 gui=reverse guifg=#fabd2f guibg=#32302f
SignColumn     xxx ctermbg=237 guibg=#3c3836
Conceal        xxx ctermfg=239 guifg=#504945
SpellBad       xxx cterm=undercurl gui=undercurl guisp=#83a598
SpellCap       xxx cterm=undercurl gui=undercurl guisp=#fb4934
SpellRare      xxx cterm=undercurl gui=undercurl guisp=#d3869b
SpellLocal     xxx cterm=undercurl gui=undercurl guisp=#8ec07c
Pmenu          xxx ctermfg=223 ctermbg=239 guifg=#ebdbb2 guibg=#504945
PmenuSel       xxx cterm=bold ctermfg=239 ctermbg=109 gui=bold guifg=#504945 guibg=#83a598
PmenuSbar      xxx ctermbg=239 guibg=#504945
PmenuThumb     xxx ctermbg=243 guibg=#7c6f64
TabLine        xxx cterm=underline ctermfg=15 ctermbg=242 gui=underline guibg=DarkGrey
                   links to TabLineFill
TabLineSel     xxx ctermfg=142 ctermbg=237 guifg=#b8bb26 guibg=#3c3836
TabLineFill    xxx ctermfg=243 ctermbg=237 guifg=#7c6f64 guibg=#3c3836
CursorColumn   xxx ctermbg=242 guibg=Grey40
                   links to CursorLine
CursorLine     xxx ctermbg=237 guibg=#3c3836
ColorColumn    xxx ctermbg=237 guibg=#3c3836
QuickFixLine   xxx links to Search
Whitespace     xxx links to NonText
NormalNC       xxx cleared
MsgSeparator   xxx links to StatusLine
NormalFloat    xxx links to Pmenu
MsgArea        xxx cleared
RedrawDebugNormal xxx cterm=reverse gui=reverse
RedrawDebugClear xxx ctermbg=11 guibg=Yellow
RedrawDebugComposed xxx ctermbg=10 guibg=Green
RedrawDebugRecompose xxx ctermbg=9 guibg=Red
Cursor         xxx cterm=reverse gui=reverse
lCursor        xxx guifg=bg guibg=fg
                   links to Cursor
Substitute     xxx links to Search
MatchParen     xxx cterm=bold ctermbg=241 gui=bold guibg=#665c54
Normal         xxx ctermfg=223 ctermbg=236 guifg=#ebdbb2 guibg=#32302f
NvimInternalError xxx ctermfg=9 ctermbg=9 guifg=Red guibg=Red
NvimAssignment xxx links to Operator
Operator       xxx links to Normal
NvimPlainAssignment xxx links to NvimAssignment
NvimAugmentedAssignment xxx links to NvimAssignment
NvimAssignmentWithAddition xxx links to NvimAugmentedAssignment
NvimAssignmentWithSubtraction xxx links to NvimAugmentedAssignment
NvimAssignmentWithConcatenation xxx links to NvimAugmentedAssignment
NvimOperator   xxx links to Operator
NvimUnaryOperator xxx links to NvimOperator
NvimUnaryPlus  xxx links to NvimUnaryOperator
NvimUnaryMinus xxx links to NvimUnaryOperator
NvimNot        xxx links to NvimUnaryOperator
NvimBinaryOperator xxx links to NvimOperator
NvimComparison xxx links to NvimBinaryOperator
NvimComparisonModifier xxx links to NvimComparison
NvimBinaryPlus xxx links to NvimBinaryOperator
NvimBinaryMinus xxx links to NvimBinaryOperator
NvimConcat     xxx links to NvimBinaryOperator
NvimConcatOrSubscript xxx links to NvimConcat
NvimOr         xxx links to NvimBinaryOperator
NvimAnd        xxx links to NvimBinaryOperator
NvimMultiplication xxx links to NvimBinaryOperator
NvimDivision   xxx links to NvimBinaryOperator
NvimMod        xxx links to NvimBinaryOperator
NvimTernary    xxx links to NvimOperator
NvimTernaryColon xxx links to NvimTernary
NvimParenthesis xxx links to Delimiter
Delimiter      xxx links to Special
NvimLambda     xxx links to NvimParenthesis
NvimNestingParenthesis xxx links to NvimParenthesis
NvimCallingParenthesis xxx links to NvimParenthesis
NvimSubscript  xxx links to NvimParenthesis
NvimSubscriptBracket xxx links to NvimSubscript
NvimSubscriptColon xxx links to NvimSubscript
NvimCurly      xxx links to NvimSubscript
NvimContainer  xxx links to NvimParenthesis
NvimDict       xxx links to NvimContainer
NvimList       xxx links to NvimContainer
NvimIdentifier xxx links to Identifier
Identifier     xxx cterm=bold ctermfg=14 guifg=#40ffff
                   links to GruvboxBlue
NvimIdentifierScope xxx links to NvimIdentifier
NvimIdentifierScopeDelimiter xxx links to NvimIdentifier
NvimIdentifierName xxx links to NvimIdentifier
NvimIdentifierKey xxx links to NvimIdentifier
NvimColon      xxx links to Delimiter
NvimComma      xxx links to Delimiter
NvimArrow      xxx links to Delimiter
NvimRegister   xxx links to SpecialChar
SpecialChar    xxx links to Special
NvimNumber     xxx links to Number
Number         xxx links to GruvboxPurple
NvimFloat      xxx links to NvimNumber
NvimNumberPrefix xxx links to Type
Type           xxx ctermfg=121 gui=bold guifg=#60ff60
                   links to GruvboxYellow
NvimOptionSigil xxx links to Type
NvimOptionName xxx links to NvimIdentifier
NvimOptionScope xxx links to NvimIdentifierScope
NvimOptionScopeDelimiter xxx links to NvimIdentifierScopeDelimiter
NvimEnvironmentSigil xxx links to NvimOptionSigil
NvimEnvironmentName xxx links to NvimIdentifier
NvimString     xxx links to String
String         xxx ctermfg=142 guifg=#b8bb26
NvimStringBody xxx links to NvimString
NvimStringQuote xxx links to NvimString
NvimStringSpecial xxx links to SpecialChar
NvimSingleQuote xxx links to NvimStringQuote
NvimSingleQuotedBody xxx links to NvimStringBody
NvimSingleQuotedQuote xxx links to NvimStringSpecial
NvimDoubleQuote xxx links to NvimStringQuote
NvimDoubleQuotedBody xxx links to NvimStringBody
NvimDoubleQuotedEscape xxx links to NvimStringSpecial
NvimFigureBrace xxx links to NvimInternalError
NvimSingleQuotedUnknownEscape xxx links to NvimInternalError
NvimSpacing    xxx links to Normal
NvimInvalidSingleQuotedUnknownEscape xxx links to NvimInternalError
NvimInvalid    xxx links to Error
Error          xxx cterm=bold,reverse ctermfg=167 ctermbg=236 gui=bold,reverse guifg=#fb4934 guibg=bg
NvimInvalidAssignment xxx links to NvimInvalid
NvimInvalidPlainAssignment xxx links to NvimInvalidAssignment
NvimInvalidAugmentedAssignment xxx links to NvimInvalidAssignment
NvimInvalidAssignmentWithAddition xxx links to NvimInvalidAugmentedAssignment
NvimInvalidAssignmentWithSubtraction xxx links to NvimInvalidAugmentedAssignment
NvimInvalidAssignmentWithConcatenation xxx links to NvimInvalidAugmentedAssignment
NvimInvalidOperator xxx links to NvimInvalid
NvimInvalidUnaryOperator xxx links to NvimInvalidOperator
NvimInvalidUnaryPlus xxx links to NvimInvalidUnaryOperator
NvimInvalidUnaryMinus xxx links to NvimInvalidUnaryOperator
NvimInvalidNot xxx links to NvimInvalidUnaryOperator
NvimInvalidBinaryOperator xxx links to NvimInvalidOperator
NvimInvalidComparison xxx links to NvimInvalidBinaryOperator
NvimInvalidComparisonModifier xxx links to NvimInvalidComparison
NvimInvalidBinaryPlus xxx links to NvimInvalidBinaryOperator
NvimInvalidBinaryMinus xxx links to NvimInvalidBinaryOperator
NvimInvalidConcat xxx links to NvimInvalidBinaryOperator
NvimInvalidConcatOrSubscript xxx links to NvimInvalidConcat
NvimInvalidOr  xxx links to NvimInvalidBinaryOperator
NvimInvalidAnd xxx links to NvimInvalidBinaryOperator
NvimInvalidMultiplication xxx links to NvimInvalidBinaryOperator
NvimInvalidDivision xxx links to NvimInvalidBinaryOperator
NvimInvalidMod xxx links to NvimInvalidBinaryOperator
NvimInvalidTernary xxx links to NvimInvalidOperator
NvimInvalidTernaryColon xxx links to NvimInvalidTernary
NvimInvalidDelimiter xxx links to NvimInvalid
NvimInvalidParenthesis xxx links to NvimInvalidDelimiter
NvimInvalidLambda xxx links to NvimInvalidParenthesis
NvimInvalidNestingParenthesis xxx links to NvimInvalidParenthesis
NvimInvalidCallingParenthesis xxx links to NvimInvalidParenthesis
NvimInvalidSubscript xxx links to NvimInvalidParenthesis
NvimInvalidSubscriptBracket xxx links to NvimInvalidSubscript
NvimInvalidSubscriptColon xxx links to NvimInvalidSubscript
NvimInvalidCurly xxx links to NvimInvalidSubscript
NvimInvalidContainer xxx links to NvimInvalidParenthesis
NvimInvalidDict xxx links to NvimInvalidContainer
NvimInvalidList xxx links to NvimInvalidContainer
NvimInvalidValue xxx links to NvimInvalid
NvimInvalidIdentifier xxx links to NvimInvalidValue
NvimInvalidIdentifierScope xxx links to NvimInvalidIdentifier
NvimInvalidIdentifierScopeDelimiter xxx links to NvimInvalidIdentifier
NvimInvalidIdentifierName xxx links to NvimInvalidIdentifier
NvimInvalidIdentifierKey xxx links to NvimInvalidIdentifier
NvimInvalidColon xxx links to NvimInvalidDelimiter
NvimInvalidComma xxx links to NvimInvalidDelimiter
NvimInvalidArrow xxx links to NvimInvalidDelimiter
NvimInvalidRegister xxx links to NvimInvalidValue
NvimInvalidNumber xxx links to NvimInvalidValue
NvimInvalidFloat xxx links to NvimInvalidNumber
NvimInvalidNumberPrefix xxx links to NvimInvalidNumber
NvimInvalidOptionSigil xxx links to NvimInvalidIdentifier
NvimInvalidOptionName xxx links to NvimInvalidIdentifier
NvimInvalidOptionScope xxx links to NvimInvalidIdentifierScope
NvimInvalidOptionScopeDelimiter xxx links to NvimInvalidIdentifierScopeDelimiter
NvimInvalidEnvironmentSigil xxx links to NvimInvalidOptionSigil
NvimInvalidEnvironmentName xxx links to NvimInvalidIdentifier
NvimInvalidString xxx links to NvimInvalidValue
NvimInvalidStringBody xxx links to NvimStringBody
NvimInvalidStringQuote xxx links to NvimInvalidString
NvimInvalidStringSpecial xxx links to NvimStringSpecial
NvimInvalidSingleQuote xxx links to NvimInvalidStringQuote
NvimInvalidSingleQuotedBody xxx links to NvimInvalidStringBody
NvimInvalidSingleQuotedQuote xxx links to NvimInvalidStringSpecial
NvimInvalidDoubleQuote xxx links to NvimInvalidStringQuote
NvimInvalidDoubleQuotedBody xxx links to NvimInvalidStringBody
NvimInvalidDoubleQuotedEscape xxx links to NvimInvalidStringSpecial
NvimInvalidDoubleQuotedUnknownEscape xxx links to NvimInvalidValue
NvimInvalidFigureBrace xxx links to NvimInvalidDelimiter
NvimInvalidSpacing xxx links to ErrorMsg
NvimDoubleQuotedUnknownEscape xxx links to NvimInvalidValue
SpaceVim_signatures xxx cleared
GruvboxFg0     xxx ctermfg=229 guifg=#fbf1c7
GruvboxFg1     xxx ctermfg=223 guifg=#ebdbb2
GruvboxFg2     xxx ctermfg=250 guifg=#d5c4a1
GruvboxFg3     xxx ctermfg=248 guifg=#bdae93
GruvboxFg4     xxx ctermfg=246 guifg=#a89984
GruvboxGray    xxx ctermfg=245 guifg=#928374
GruvboxBg0     xxx ctermfg=236 guifg=#32302f
GruvboxBg1     xxx ctermfg=237 guifg=#3c3836
GruvboxBg2     xxx ctermfg=239 guifg=#504945
GruvboxBg3     xxx ctermfg=241 guifg=#665c54
GruvboxBg4     xxx ctermfg=243 guifg=#7c6f64
GruvboxRed     xxx ctermfg=167 guifg=#fb4934
GruvboxRedBold xxx cterm=bold ctermfg=167 gui=bold guifg=#fb4934
GruvboxGreen   xxx ctermfg=142 guifg=#b8bb26
GruvboxGreenBold xxx cterm=bold ctermfg=142 gui=bold guifg=#b8bb26
GruvboxYellow  xxx ctermfg=214 guifg=#fabd2f
GruvboxYellowBold xxx cterm=bold ctermfg=214 gui=bold guifg=#fabd2f
GruvboxBlue    xxx ctermfg=109 guifg=#83a598
GruvboxBlueBold xxx cterm=bold ctermfg=109 gui=bold guifg=#83a598
GruvboxPurple  xxx ctermfg=175 guifg=#d3869b
GruvboxPurpleBold xxx cterm=bold ctermfg=175 gui=bold guifg=#d3869b
GruvboxAqua    xxx ctermfg=108 guifg=#8ec07c
GruvboxAquaBold xxx cterm=bold ctermfg=108 gui=bold guifg=#8ec07c
GruvboxOrange  xxx ctermfg=208 guifg=#fe8019
GruvboxOrangeBold xxx cterm=bold ctermfg=208 gui=bold guifg=#fe8019
GruvboxRedSign xxx ctermfg=167 ctermbg=237 guifg=#fb4934 guibg=#3c3836
GruvboxGreenSign xxx ctermfg=142 ctermbg=237 guifg=#b8bb26 guibg=#3c3836
GruvboxYellowSign xxx ctermfg=214 ctermbg=237 guifg=#fabd2f guibg=#3c3836
GruvboxBlueSign xxx ctermfg=109 ctermbg=237 guifg=#83a598 guibg=#3c3836
GruvboxPurpleSign xxx ctermfg=175 ctermbg=237 guifg=#d3869b guibg=#3c3836
GruvboxAquaSign xxx ctermfg=108 ctermbg=237 guifg=#8ec07c guibg=#3c3836
GruvboxOrangeSign xxx ctermfg=208 ctermbg=237 guifg=#fe8019 guibg=#3c3836
VisualNOS      xxx links to Visual
Underlined     xxx cterm=underline ctermfg=109 gui=underline guifg=#83a598
vCursor        xxx links to Cursor
iCursor        xxx links to Cursor
Special        xxx ctermfg=224 guifg=Orange
                   links to GruvboxOrange
Comment        xxx ctermfg=245 guifg=#928374
Todo           xxx cterm=bold ctermfg=223 ctermbg=236 gui=bold guifg=fg guibg=bg
Statement      xxx ctermfg=11 gui=bold guifg=#ffff60
                   links to GruvboxRed
Conditional    xxx links to GruvboxRed
Repeat         xxx links to GruvboxRed
Label          xxx links to GruvboxRed
Exception      xxx links to GruvboxRed
Keyword        xxx links to GruvboxRed
Function       xxx links to GruvboxGreenBold
PreProc        xxx ctermfg=81 guifg=#ff80ff
                   links to GruvboxAqua
Include        xxx links to GruvboxAqua
Define         xxx links to GruvboxAqua
Macro          xxx links to GruvboxAqua
PreCondit      xxx links to GruvboxAqua
Constant       xxx ctermfg=13 guifg=#ffa0a0
                   links to GruvboxPurple
Character      xxx links to GruvboxPurple
Boolean        xxx links to GruvboxPurple
Float          xxx links to GruvboxPurple
StorageClass   xxx links to GruvboxOrange
Structure      xxx links to GruvboxAqua
Typedef        xxx links to GruvboxYellow
EasyMotionTarget xxx links to Search
EasyMotionShade xxx links to Comment
Sneak          xxx links to Search
SneakLabel     xxx links to Search
IndentGuidesOdd xxx ctermfg=236 ctermbg=239 guifg=bg guibg=#504945
IndentGuidesEven xxx ctermfg=236 ctermbg=237 guifg=bg guibg=#3c3836
GitGutterAdd   xxx links to GruvboxGreenSign
GitGutterChange xxx links to GruvboxAquaSign
GitGutterDelete xxx links to GruvboxRedSign
GitGutterChangeDelete xxx links to GruvboxAquaSign
gitcommitSelectedFile xxx links to GruvboxGreen
gitcommitDiscardedFile xxx links to GruvboxRed
SignifySignAdd xxx links to GruvboxGreenSign
SignifySignChange xxx links to GruvboxAquaSign
SignifySignDelete xxx links to GruvboxRedSign
SyntasticError xxx cterm=undercurl gui=undercurl guisp=#fb4934
SyntasticWarning xxx cterm=undercurl gui=undercurl guisp=#fabd2f
SyntasticErrorSign xxx links to GruvboxRedSign
SyntasticWarningSign xxx links to GruvboxYellowSign
SignatureMarkText xxx links to GruvboxBlueSign
SignatureMarkerText xxx links to GruvboxPurpleSign
ShowMarksHLl   xxx links to GruvboxBlueSign
ShowMarksHLu   xxx links to GruvboxBlueSign
ShowMarksHLo   xxx links to GruvboxBlueSign
ShowMarksHLm   xxx links to GruvboxBlueSign
CtrlPMatch     xxx links to GruvboxYellow
CtrlPNoEntries xxx links to GruvboxRed
CtrlPPrtBase   xxx links to GruvboxBg2
CtrlPPrtCursor xxx links to GruvboxBlue
CtrlPLinePre   xxx links to GruvboxBg2
CtrlPMode1     xxx cterm=bold ctermfg=109 ctermbg=239 gui=bold guifg=#83a598 guibg=#504945
CtrlPMode2     xxx cterm=bold ctermfg=236 ctermbg=109 gui=bold guifg=#32302f guibg=#83a598
CtrlPStats     xxx cterm=bold ctermfg=246 ctermbg=239 gui=bold guifg=#a89984 guibg=#504945
StartifyBracket xxx links to GruvboxFg3
StartifyFile   xxx links to GruvboxFg1
StartifyNumber xxx links to GruvboxBlue
StartifyPath   xxx links to GruvboxGray
StartifySlash  xxx links to GruvboxGray
StartifySection xxx links to GruvboxYellow
StartifySpecial xxx links to GruvboxBg2
StartifyHeader xxx links to GruvboxOrange
StartifyFooter xxx links to GruvboxBg2
BufTabLineCurrent xxx ctermfg=236 ctermbg=246 guifg=#32302f guibg=#a89984
BufTabLineActive xxx ctermfg=246 ctermbg=239 guifg=#a89984 guibg=#504945
BufTabLineHidden xxx ctermfg=243 ctermbg=237 guifg=#7c6f64 guibg=#3c3836
BufTabLineFill xxx ctermfg=236 ctermbg=236 guifg=#32302f guibg=#32302f
ALEError       xxx cterm=undercurl gui=undercurl guisp=#fb4934
ALEWarning     xxx cterm=undercurl gui=undercurl guisp=#fabd2f
ALEInfo        xxx cterm=undercurl gui=undercurl guisp=#83a598
ALEErrorSign   xxx links to GruvboxRedSign
ALEWarningSign xxx links to GruvboxYellowSign
ALEInfoSign    xxx links to GruvboxBlueSign
DirvishPathTail xxx links to GruvboxAqua
DirvishArg     xxx links to GruvboxYellow
netrwDir       xxx links to GruvboxAqua
netrwClassify  xxx links to GruvboxAqua
netrwLink      xxx links to GruvboxGray
netrwSymLink   xxx links to GruvboxFg1
netrwExe       xxx links to GruvboxYellow
netrwComment   xxx links to GruvboxGray
netrwList      xxx links to GruvboxBlue
netrwHelpCmd   xxx links to GruvboxAqua
netrwCmdSep    xxx links to GruvboxFg3
netrwVersion   xxx links to GruvboxGreen
NERDTreeDir    xxx links to GruvboxAqua
NERDTreeDirSlash xxx links to GruvboxAqua
NERDTreeOpenable xxx links to GruvboxOrange
NERDTreeClosable xxx links to GruvboxOrange
NERDTreeFile   xxx links to GruvboxFg1
NERDTreeExecFile xxx links to GruvboxYellow
NERDTreeUp     xxx links to GruvboxGray
NERDTreeCWD    xxx links to GruvboxGreen
NERDTreeHelp   xxx links to GruvboxFg1
NERDTreeToggleOn xxx links to GruvboxGreen
NERDTreeToggleOff xxx links to GruvboxRed
multiple_cursors_cursor xxx cterm=reverse gui=reverse
multiple_cursors_visual xxx ctermbg=239 guibg=#504945
CocErrorSign   xxx links to GruvboxRedSign
CocWarningSign xxx links to GruvboxOrangeSign
CocInfoSign    xxx links to GruvboxYellowSign
CocHintSign    xxx links to GruvboxBlueSign
CocErrorFloat  xxx links to GruvboxRed
CocWarningFloat xxx links to GruvboxOrange
CocInfoFloat   xxx links to GruvboxYellow
CocHintFloat   xxx links to GruvboxBlue
CocDiagnosticsError xxx links to GruvboxRed
CocDiagnosticsWarning xxx links to GruvboxOrange
CocDiagnosticsInfo xxx links to GruvboxYellow
CocDiagnosticsHint xxx links to GruvboxBlue
CocSelectedText xxx links to GruvboxRed
CocCodeLens    xxx links to GruvboxGray
CocErrorHighlight xxx cterm=undercurl gui=undercurl guisp=#fb4934
CocWarningHighlight xxx cterm=undercurl gui=undercurl guisp=#fe8019
CocInfoHighlight xxx cterm=undercurl gui=undercurl guisp=#fabd2f
CocHintHighlight xxx cterm=undercurl gui=undercurl guisp=#83a598
diffAdded      xxx links to GruvboxGreen
diffRemoved    xxx links to GruvboxRed
diffChanged    xxx links to GruvboxAqua
diffFile       xxx links to GruvboxOrange
diffNewFile    xxx links to GruvboxYellow
diffLine       xxx links to GruvboxBlue
htmlTag        xxx links to GruvboxBlue
htmlEndTag     xxx links to GruvboxBlue
htmlTagName    xxx links to GruvboxAquaBold
htmlArg        xxx links to GruvboxAqua
htmlScriptTag  xxx links to GruvboxPurple
htmlTagN       xxx links to GruvboxFg1
htmlSpecialTagName xxx links to GruvboxAquaBold
htmlLink       xxx cterm=underline ctermfg=246 gui=underline guifg=#a89984
htmlSpecialChar xxx links to GruvboxOrange
htmlBold       xxx cterm=bold ctermfg=223 ctermbg=236 gui=bold guifg=fg guibg=bg
htmlBoldUnderline xxx cterm=bold,underline ctermfg=223 ctermbg=236 gui=bold,underline guifg=fg guibg=bg
htmlBoldItalic xxx cterm=bold ctermfg=223 ctermbg=236 gui=bold guifg=fg guibg=bg
htmlBoldUnderlineItalic xxx cterm=bold,underline ctermfg=223 ctermbg=236 gui=bold,underline guifg=fg guibg=bg
htmlUnderline  xxx cterm=underline ctermfg=223 ctermbg=236 gui=underline guifg=fg guibg=bg
htmlUnderlineItalic xxx cterm=underline ctermfg=223 ctermbg=236 gui=underline guifg=fg guibg=bg
htmlItalic     xxx ctermfg=223 ctermbg=236 guifg=fg guibg=bg
xmlTag         xxx links to GruvboxBlue
xmlEndTag      xxx links to GruvboxBlue
xmlTagName     xxx links to GruvboxBlue
xmlEqual       xxx links to GruvboxBlue
docbkKeyword   xxx links to GruvboxAquaBold
xmlDocTypeDecl xxx links to GruvboxGray
xmlDocTypeKeyword xxx links to GruvboxPurple
xmlCdataStart  xxx links to GruvboxGray
xmlCdataCdata  xxx links to GruvboxPurple
dtdFunction    xxx links to GruvboxGray
dtdTagName     xxx links to GruvboxPurple
xmlAttrib      xxx links to GruvboxAqua
xmlProcessingDelim xxx links to GruvboxGray
dtdParamEntityPunct xxx links to GruvboxGray
dtdParamEntityDPunct xxx links to GruvboxGray
xmlAttribPunct xxx links to GruvboxGray
xmlEntity      xxx links to GruvboxOrange
xmlEntityPunct xxx links to GruvboxOrange
vimCommentTitle xxx cterm=bold ctermfg=246 gui=bold guifg=#a89984
vimNotation    xxx links to GruvboxOrange
vimBracket     xxx links to GruvboxOrange
vimMapModKey   xxx links to GruvboxOrange
vimFuncSID     xxx links to GruvboxFg3
vimSetSep      xxx links to GruvboxFg3
vimSep         xxx links to GruvboxFg3
vimContinue    xxx links to GruvboxFg3
clojureKeyword xxx links to GruvboxBlue
clojureCond    xxx links to GruvboxOrange
clojureSpecial xxx links to GruvboxOrange
clojureDefine  xxx links to GruvboxOrange
clojureFunc    xxx links to GruvboxYellow
clojureRepeat  xxx links to GruvboxYellow
clojureCharacter xxx links to GruvboxAqua
clojureStringEscape xxx links to GruvboxAqua
clojureException xxx links to GruvboxRed
clojureRegexp  xxx links to GruvboxAqua
clojureRegexpEscape xxx links to GruvboxAqua
clojureRegexpCharClass xxx cterm=bold ctermfg=248 gui=bold guifg=#bdae93
clojureRegexpMod xxx links to clojureRegexpCharClass
clojureRegexpQuantifier xxx links to clojureRegexpCharClass
clojureParen   xxx links to GruvboxFg3
clojureAnonArg xxx links to GruvboxYellow
clojureVariable xxx links to GruvboxBlue
clojureMacro   xxx links to GruvboxOrange
clojureMeta    xxx links to GruvboxYellow
clojureDeref   xxx links to GruvboxYellow
clojureQuote   xxx links to GruvboxYellow
clojureUnquote xxx links to GruvboxYellow
cOperator      xxx links to GruvboxPurple
cStructure     xxx links to GruvboxOrange
pythonBuiltin  xxx links to GruvboxOrange
pythonBuiltinObj xxx links to GruvboxOrange
pythonBuiltinFunc xxx links to GruvboxOrange
pythonFunction xxx links to GruvboxAqua
pythonDecorator xxx links to GruvboxRed
pythonInclude  xxx links to GruvboxBlue
pythonImport   xxx links to GruvboxBlue
pythonRun      xxx links to GruvboxBlue
pythonCoding   xxx links to GruvboxBlue
pythonOperator xxx links to GruvboxRed
pythonException xxx links to GruvboxRed
pythonExceptions xxx links to GruvboxPurple
pythonBoolean  xxx links to GruvboxPurple
pythonDot      xxx links to GruvboxFg3
pythonConditional xxx links to GruvboxRed
pythonRepeat   xxx links to GruvboxRed
pythonDottedName xxx links to GruvboxGreenBold
cssBraces      xxx links to GruvboxBlue
cssFunctionName xxx links to GruvboxYellow
cssIdentifier  xxx links to GruvboxOrange
cssClassName   xxx links to GruvboxGreen
cssColor       xxx links to GruvboxBlue
cssSelectorOp  xxx links to GruvboxBlue
cssSelectorOp2 xxx links to GruvboxBlue
cssImportant   xxx links to GruvboxGreen
cssVendor      xxx links to GruvboxFg1
cssTextProp    xxx links to GruvboxAqua
cssAnimationProp xxx links to GruvboxAqua
cssUIProp      xxx links to GruvboxYellow
cssTransformProp xxx links to GruvboxAqua
cssTransitionProp xxx links to GruvboxAqua
cssPrintProp   xxx links to GruvboxAqua
cssPositioningProp xxx links to GruvboxYellow
cssBoxProp     xxx links to GruvboxAqua
cssFontDescriptorProp xxx links to GruvboxAqua
cssFlexibleBoxProp xxx links to GruvboxAqua
cssBorderOutlineProp xxx links to GruvboxAqua
cssBackgroundProp xxx links to GruvboxAqua
cssMarginProp  xxx links to GruvboxAqua
cssListProp    xxx links to GruvboxAqua
cssTableProp   xxx links to GruvboxAqua
cssFontProp    xxx links to GruvboxAqua
cssPaddingProp xxx links to GruvboxAqua
cssDimensionProp xxx links to GruvboxAqua
cssRenderProp  xxx links to GruvboxAqua
cssColorProp   xxx links to GruvboxAqua
cssGeneratedContentProp xxx links to GruvboxAqua
javaScriptBraces xxx links to GruvboxFg1
javaScriptFunction xxx links to GruvboxAqua
javaScriptIdentifier xxx links to GruvboxOrange
javaScriptMember xxx links to GruvboxBlue
javaScriptNumber xxx links to GruvboxPurple
javaScriptNull xxx links to GruvboxPurple
javaScriptParens xxx links to GruvboxFg3
javascriptImport xxx links to GruvboxAqua
javascriptExport xxx links to GruvboxAqua
javascriptClassKeyword xxx links to GruvboxAqua
javascriptClassExtends xxx links to GruvboxAqua
javascriptDefault xxx links to GruvboxAqua
javascriptClassName xxx links to GruvboxYellow
javascriptClassSuperName xxx links to GruvboxYellow
javascriptGlobal xxx links to GruvboxYellow
javascriptEndColons xxx links to GruvboxFg1
javascriptFuncArg xxx links to GruvboxFg1
javascriptGlobalMethod xxx links to GruvboxFg1
javascriptNodeGlobal xxx links to GruvboxFg1
javascriptBOMWindowProp xxx links to GruvboxFg1
javascriptArrayMethod xxx links to GruvboxFg1
javascriptArrayStaticMethod xxx links to GruvboxFg1
javascriptCacheMethod xxx links to GruvboxFg1
javascriptDateMethod xxx links to GruvboxFg1
javascriptMathStaticMethod xxx links to GruvboxFg1
javascriptURLUtilsProp xxx links to GruvboxFg1
javascriptBOMNavigatorProp xxx links to GruvboxFg1
javascriptDOMDocMethod xxx links to GruvboxFg1
javascriptDOMDocProp xxx links to GruvboxFg1
javascriptBOMLocationMethod xxx links to GruvboxFg1
javascriptBOMWindowMethod xxx links to GruvboxFg1
javascriptStringMethod xxx links to GruvboxFg1
javascriptVariable xxx links to GruvboxOrange
javascriptClassSuper xxx links to GruvboxOrange
javascriptFuncKeyword xxx links to GruvboxAqua
javascriptAsyncFunc xxx links to GruvboxAqua
javascriptClassStatic xxx links to GruvboxOrange
javascriptOperator xxx links to GruvboxRed
javascriptForOperator xxx links to GruvboxRed
javascriptYield xxx links to GruvboxRed
javascriptExceptions xxx links to GruvboxRed
javascriptMessage xxx links to GruvboxRed
javascriptTemplateSB xxx links to GruvboxAqua
javascriptTemplateSubstitution xxx links to GruvboxFg1
javascriptLabel xxx links to GruvboxFg1
javascriptObjectLabel xxx links to GruvboxFg1
javascriptPropertyName xxx links to GruvboxFg1
javascriptLogicSymbols xxx links to GruvboxFg1
javascriptArrowFunc xxx links to GruvboxYellow
javascriptDocParamName xxx links to GruvboxFg4
javascriptDocTags xxx links to GruvboxFg4
javascriptDocNotation xxx links to GruvboxFg4
javascriptDocParamType xxx links to GruvboxFg4
javascriptDocNamedParamType xxx links to GruvboxFg4
javascriptBrackets xxx links to GruvboxFg1
javascriptDOMElemAttrs xxx links to GruvboxFg1
javascriptDOMEventMethod xxx links to GruvboxFg1
javascriptDOMNodeMethod xxx links to GruvboxFg1
javascriptDOMStorageMethod xxx links to GruvboxFg1
javascriptHeadersMethod xxx links to GruvboxFg1
javascriptAsyncFuncKeyword xxx links to GruvboxRed
javascriptAwaitFuncKeyword xxx links to GruvboxRed
jsClassKeyword xxx links to GruvboxAqua
jsExtendsKeyword xxx links to GruvboxAqua
jsExportDefault xxx links to GruvboxAqua
jsTemplateBraces xxx links to GruvboxAqua
jsGlobalNodeObjects xxx links to GruvboxFg1
jsGlobalObjects xxx links to GruvboxFg1
jsFunction     xxx links to GruvboxAqua
jsFuncParens   xxx links to GruvboxFg3
jsParens       xxx links to GruvboxFg3
jsNull         xxx links to GruvboxPurple
jsUndefined    xxx links to GruvboxPurple
jsClassDefinition xxx links to GruvboxYellow
typeScriptReserved xxx links to GruvboxAqua
typeScriptLabel xxx links to GruvboxAqua
typeScriptFuncKeyword xxx links to GruvboxAqua
typeScriptIdentifier xxx links to GruvboxOrange
typeScriptBraces xxx links to GruvboxFg1
typeScriptEndColons xxx links to GruvboxFg1
typeScriptDOMObjects xxx links to GruvboxFg1
typeScriptAjaxMethods xxx links to GruvboxFg1
typeScriptLogicSymbols xxx links to GruvboxFg1
typeScriptDocSeeTag xxx links to Comment
typeScriptDocParam xxx links to Comment
typeScriptDocTags xxx links to vimCommentTitle
typeScriptGlobalObjects xxx links to GruvboxFg1
typeScriptParens xxx links to GruvboxFg3
typeScriptOpSymbols xxx links to GruvboxFg3
typeScriptHtmlElemProperties xxx links to GruvboxFg1
typeScriptNull xxx links to GruvboxPurple
typeScriptInterpolationDelimiter xxx links to GruvboxAqua
purescriptModuleKeyword xxx links to GruvboxAqua
purescriptModuleName xxx links to GruvboxFg1
purescriptWhere xxx links to GruvboxAqua
purescriptDelimiter xxx links to GruvboxFg4
purescriptType xxx links to GruvboxFg1
purescriptImportKeyword xxx links to GruvboxAqua
purescriptHidingKeyword xxx links to GruvboxAqua
purescriptAsKeyword xxx links to GruvboxAqua
purescriptStructure xxx links to GruvboxAqua
purescriptOperator xxx links to GruvboxBlue
purescriptTypeVar xxx links to GruvboxFg1
purescriptConstructor xxx links to GruvboxFg1
purescriptFunction xxx links to GruvboxFg1
purescriptConditional xxx links to GruvboxOrange
purescriptBacktick xxx links to GruvboxOrange
coffeeExtendedOp xxx links to GruvboxFg3
coffeeSpecialOp xxx links to GruvboxFg3
coffeeCurly    xxx links to GruvboxOrange
coffeeParen    xxx links to GruvboxFg3
coffeeBracket  xxx links to GruvboxOrange
rubyStringDelimiter xxx links to GruvboxGreen
rubyInterpolationDelimiter xxx links to GruvboxAqua
objcTypeModifier xxx links to GruvboxRed
objcDirective  xxx links to GruvboxBlue
goDirective    xxx links to GruvboxAqua
goConstants    xxx links to GruvboxPurple
goDeclaration  xxx links to GruvboxRed
goDeclType     xxx links to GruvboxBlue
goBuiltins     xxx links to GruvboxOrange
luaIn          xxx links to GruvboxRed
luaFunction    xxx links to GruvboxAqua
luaTable       xxx links to GruvboxOrange
moonSpecialOp  xxx links to GruvboxFg3
moonExtendedOp xxx links to GruvboxFg3
moonFunction   xxx links to GruvboxFg3
moonObject     xxx links to GruvboxYellow
javaAnnotation xxx links to GruvboxBlue
javaDocTags    xxx links to GruvboxAqua
javaCommentTitle xxx links to vimCommentTitle
javaParen      xxx links to GruvboxFg3
javaParen1     xxx links to GruvboxFg3
javaParen2     xxx links to GruvboxFg3
javaParen3     xxx links to GruvboxFg3
javaParen4     xxx links to GruvboxFg3
javaParen5     xxx links to GruvboxFg3
javaOperator   xxx links to GruvboxOrange
javaVarArg     xxx links to GruvboxGreen
elixirDocString xxx links to Comment
elixirStringDelimiter xxx links to GruvboxGreen
elixirInterpolationDelimiter xxx links to GruvboxAqua
elixirModuleDeclaration xxx links to GruvboxYellow
scalaNameDefinition xxx links to GruvboxFg1
scalaCaseFollowing xxx links to GruvboxFg1
scalaCapitalWord xxx links to GruvboxFg1
scalaTypeExtension xxx links to GruvboxFg1
scalaKeyword   xxx links to GruvboxRed
scalaKeywordModifier xxx links to GruvboxRed
scalaSpecial   xxx links to GruvboxAqua
scalaOperator  xxx links to GruvboxFg1
scalaTypeDeclaration xxx links to GruvboxYellow
scalaTypeTypePostDeclaration xxx links to GruvboxYellow
scalaInstanceDeclaration xxx links to GruvboxFg1
scalaInterpolation xxx links to GruvboxAqua
markdownItalic xxx ctermfg=248 guifg=#bdae93
markdownH1     xxx links to GruvboxGreenBold
markdownH2     xxx links to GruvboxGreenBold
markdownH3     xxx links to GruvboxYellowBold
markdownH4     xxx links to GruvboxYellowBold
markdownH5     xxx links to GruvboxYellow
markdownH6     xxx links to GruvboxYellow
markdownCode   xxx links to GruvboxAqua
markdownCodeBlock xxx links to GruvboxAqua
markdownCodeDelimiter xxx links to GruvboxAqua
markdownBlockquote xxx links to GruvboxGray
markdownListMarker xxx links to GruvboxGray
markdownOrderedListMarker xxx links to GruvboxGray
markdownRule   xxx links to GruvboxGray
markdownHeadingRule xxx links to GruvboxGray
markdownUrlDelimiter xxx links to GruvboxFg3
markdownLinkDelimiter xxx links to GruvboxFg3
markdownLinkTextDelimiter xxx links to GruvboxFg3
markdownHeadingDelimiter xxx links to GruvboxOrange
markdownUrl    xxx links to GruvboxPurple
markdownUrlTitleDelimiter xxx links to GruvboxGreen
markdownLinkText xxx cterm=underline ctermfg=245 gui=underline guifg=#928374
markdownIdDeclaration xxx links to markdownLinkText
haskellType    xxx links to GruvboxFg1
haskellIdentifier xxx links to GruvboxFg1
haskellSeparator xxx links to GruvboxFg1
haskellDelimiter xxx links to GruvboxFg4
haskellOperators xxx links to GruvboxBlue
haskellBacktick xxx links to GruvboxOrange
haskellStatement xxx links to GruvboxOrange
haskellConditional xxx links to GruvboxOrange
haskellLet     xxx links to GruvboxAqua
haskellDefault xxx links to GruvboxAqua
haskellWhere   xxx links to GruvboxAqua
haskellBottom  xxx links to GruvboxAqua
haskellBlockKeywords xxx links to GruvboxAqua
haskellImportKeywords xxx links to GruvboxAqua
haskellDeclKeyword xxx links to GruvboxAqua
haskellDeriving xxx links to GruvboxAqua
haskellAssocType xxx links to GruvboxAqua
haskellNumber  xxx links to GruvboxPurple
haskellPragma  xxx links to GruvboxPurple
haskellString  xxx links to GruvboxGreen
haskellChar    xxx links to GruvboxGreen
jsonKeyword    xxx links to GruvboxGreen
jsonQuote      xxx links to GruvboxGreen
jsonBraces     xxx links to GruvboxFg1
jsonString     xxx links to GruvboxFg1
SpaceVim_statusline_a xxx cterm=bold ctermfg=235 ctermbg=246 gui=bold guifg=#282828 guibg=#a89984
SpaceVim_statusline_a_bold xxx cterm=bold ctermfg=235 ctermbg=246 gui=bold guifg=#282828 guibg=#a89984
SpaceVim_statusline_ia xxx cterm=bold ctermfg=235 ctermbg=246 gui=bold guifg=#282828 guibg=#a89984
SpaceVim_statusline_b xxx ctermfg=246 ctermbg=239 guifg=#a89984 guibg=#504945
SpaceVim_statusline_c xxx ctermfg=246 ctermbg=237 guifg=#a89984 guibg=#3c3836
SpaceVim_statusline_z xxx ctermfg=237 ctermbg=241 guifg=#a89984 guibg=#665c54
SpaceVim_statusline_error xxx ctermfg=0 ctermbg=3 gui=bold guifg=#fb4934 guibg=#504945
SpaceVim_statusline_warn xxx ctermfg=0 ctermbg=3 gui=bold guifg=#fabd2f guibg=#504945
SpaceVim_statusline_a_SpaceVim_statusline_b xxx ctermfg=246 ctermbg=239 guifg=#a89984 guibg=#504945
SpaceVim_statusline_b_SpaceVim_statusline_a xxx ctermfg=239 ctermbg=246 guifg=#504945 guibg=#a89984
SpaceVim_statusline_a_bold_SpaceVim_statusline_b xxx ctermfg=246 ctermbg=239 guifg=#a89984 guibg=#504945
SpaceVim_statusline_b_SpaceVim_statusline_a_bold xxx ctermfg=239 ctermbg=246 guifg=#504945 guibg=#a89984
SpaceVim_statusline_ia_SpaceVim_statusline_b xxx ctermfg=246 ctermbg=239 guifg=#a89984 guibg=#504945
SpaceVim_statusline_b_SpaceVim_statusline_ia xxx ctermfg=239 ctermbg=246 guifg=#504945 guibg=#a89984
SpaceVim_statusline_b_SpaceVim_statusline_c xxx ctermfg=239 ctermbg=237 guifg=#504945 guibg=#3c3836
SpaceVim_statusline_c_SpaceVim_statusline_b xxx ctermfg=237 ctermbg=239 guifg=#3c3836 guibg=#504945
SpaceVim_statusline_b_SpaceVim_statusline_z xxx ctermfg=239 ctermbg=241 guifg=#504945 guibg=#665c54
SpaceVim_statusline_z_SpaceVim_statusline_b xxx ctermfg=241 ctermbg=239 guifg=#665c54 guibg=#504945
SpaceVim_statusline_c_SpaceVim_statusline_z xxx ctermfg=237 ctermbg=241 guifg=#3c3836 guibg=#665c54
SpaceVim_statusline_z_SpaceVim_statusline_c xxx ctermfg=241 ctermbg=237 guifg=#665c54 guibg=#3c3836
SpaceVim_tabline_a xxx ctermfg=235 ctermbg=246 guifg=#282828 guibg=#a89984
SpaceVim_tabline_b xxx ctermfg=246 ctermbg=239 guifg=#a89984 guibg=#504945
SpaceVim_tabline_m xxx ctermfg=235 ctermbg=109 guifg=#282828 guibg=#83a598
SpaceVim_tabline_m_i xxx ctermfg=109 ctermbg=239 guifg=#83a598 guibg=#504945
SpaceVim_tabline_a_SpaceVim_tabline_b xxx ctermfg=246 ctermbg=239 guifg=#a89984 guibg=#504945
SpaceVim_tabline_b_SpaceVim_tabline_a xxx ctermfg=239 ctermbg=246 guifg=#504945 guibg=#a89984
SpaceVim_tabline_m_SpaceVim_tabline_b xxx ctermfg=109 ctermbg=239 guifg=#83a598 guibg=#504945
SpaceVim_tabline_b_SpaceVim_tabline_m xxx ctermfg=239 ctermbg=109 guifg=#504945 guibg=#83a598
SpaceVim_tabline_m_SpaceVim_tabline_a xxx ctermfg=109 ctermbg=246 guifg=#83a598 guibg=#a89984
SpaceVim_tabline_a_SpaceVim_tabline_m xxx ctermfg=246 ctermbg=109 guifg=#a89984 guibg=#83a598
Ignore         xxx ctermfg=0 guifg=bg
Tag            xxx links to Special
SpecialComment xxx links to Special
Debug          xxx links to Special
SpaceVimLeaderGuiderGroupName xxx cterm=bold ctermfg=175 gui=bold guifg=#d3869b
EasyMotionTargetDefault xxx cterm=bold ctermfg=196 gui=bold guifg=#ff0000
EasyMotionTarget2FirstDefault xxx cterm=bold ctermfg=11 gui=bold guifg=#ffb400
EasyMotionTarget2First xxx links to EasyMotionTarget2FirstDefault
EasyMotionTarget2SecondDefault xxx cterm=bold ctermfg=3 gui=bold guifg=#b98300
EasyMotionTarget2Second xxx links to EasyMotionTarget2SecondDefault
EasyMotionShadeDefault xxx ctermfg=242 guifg=#777777
EasyMotionIncSearchDefault xxx cterm=bold ctermfg=40 gui=bold guifg=#7fbf00
EasyMotionIncSearch xxx links to EasyMotionIncSearchDefault
EasyMotionIncCursorDefault xxx cterm=bold ctermfg=232 ctermbg=14 gui=bold guifg=#121813 guibg=#ACDBDA
EasyMotionIncCursor xxx links to EasyMotionIncCursorDefault
EasyMotionMoveHLDefault xxx cterm=bold ctermfg=15 ctermbg=10 gui=bold guifg=#121813 guibg=#7fbf00
EasyMotionMoveHL xxx links to EasyMotionMoveHLDefault
EasyOperatorFirstLineDefault xxx ctermfg=242 ctermbg=9 guifg=#FFFFFF guibg=red
EasyOperatorFirstLine xxx links to EasyOperatorFirstLineDefault
DefxIconsMarkIcon xxx links to Statement
DefxIconsCopyIcon xxx links to WarningMsg
DefxIconsMoveIcon xxx links to ErrorMsg
DefxIconsDirectory xxx links to Directory
DefxIconsParentDirectory xxx links to Directory
DefxIconsSymlinkDirectory xxx links to Directory
DefxIconsOpenedTreeIcon xxx links to Directory
DefxIconsNestedTreeIcon xxx links to Directory
DefxIconsClosedTreeIcon xxx links to Directory
MatchParenCur  xxx links to MatchParen
MatchWord      xxx links to MatchParen
MatchBackground xxx links to ColorColumn
NeomakeStatReset xxx links to StatusLine
NeomakeStatusGood xxx cterm=reverse ctermfg=10 ctermbg=239 gui=reverse guifg=#504945
NeomakeStatResetNC xxx links to StatusLineNC
NeomakeStatusGoodNC xxx cterm=reverse ctermfg=10 ctermbg=237 gui=reverse guifg=#3c3836
NeomakeStatColorDefault xxx ctermfg=15 ctermbg=12
NeomakeStatColorQuickfixDefault xxx links to NeomakeStatColorDefault
NeomakeStatColorTypeE xxx ctermfg=15 ctermbg=9
NeomakeStatColorQuickfixTypeE xxx links to NeomakeStatColorTypeE
NeomakeStatColorTypeW xxx ctermfg=15 ctermbg=11
NeomakeStatColorQuickfixTypeW xxx links to NeomakeStatColorTypeW
LanguageClientCodeLens xxx links to Title
LanguageClientWarningSign xxx links to Todo
LanguageClientWarning xxx links to SpellCap
LanguageClientInfoSign xxx links to LanguageClientWarningSign
LanguageClientInfo xxx links to LanguageClientWarning
LanguageClientErrorSign xxx links to Error
LanguageClientError xxx links to SpellBad
StartifySelect xxx links to Title
StartifyVar    xxx links to StartifyPath
Defx_mark_readonly xxx links to Comment
Defx_mark_selected xxx links to Statement
Defx_icon_directory_icon xxx links to Special
Defx_icon_opened_icon xxx links to Special
Defx_icon_root_icon xxx links to Identifier
Defx_filename_directory xxx links to PreProc
Defx_filename_root_marker xxx links to Constant
Defx_filename_root xxx links to Identifier
Defx_type_text xxx links to Constant
Defx_type_image xxx links to Type
Defx_type_archive xxx links to Special
Defx_type_executable xxx links to Statement
NeomakeVirtualtextInfoDefault xxx ctermfg=208 guifg=#fe8019
NeomakeVirtualtextInfo xxx links to NeomakeVirtualtextInfoDefault
NeomakeVirtualtextMessageDefault xxx ctermfg=214 guifg=#fabd2f
NeomakeVirtualtextMessage xxx links to NeomakeVirtualtextMessageDefault
NeomakeVirtualtextWarningDefault xxx ctermfg=223 guifg=fg
NeomakeVirtualtextWarning xxx links to NeomakeVirtualtextWarningDefault
NeomakeVirtualtextErrorDefault xxx ctermfg=167 guifg=#fb4934
NeomakeVirtualtextError xxx links to NeomakeVirtualtextErrorDefault
cStatement     xxx links to Statement
cLabel         xxx links to Label
cConditional   xxx links to Conditional
cRepeat        xxx links to Repeat
cTodo          xxx links to Todo
cBadContinuation xxx links to Error
cSpecial       xxx links to SpecialChar
cFormat        xxx links to cSpecial
cString        xxx links to String
cCppString     xxx links to cString
cSpaceError    xxx links to cError
cCppSkip       xxx cleared
cCharacter     xxx links to Character
cSpecialError  xxx links to cError
cSpecialCharacter xxx links to cSpecial
cBadBlock      xxx cleared
cCurlyError    xxx links to cError
cErrInParen    xxx links to cError
cCppParen      xxx cleared
cErrInBracket  xxx links to cError
cCppBracket    xxx cleared
cBlock         xxx cleared
cParenError    xxx links to cError
cIncluded      xxx links to cString
cCommentSkip   xxx links to cComment
cCommentString xxx links to cString
cComment2String xxx links to cString
cCommentStartError xxx links to cError
cUserLabel     xxx links to Label
cBitField      xxx cleared
cOctalZero     xxx links to PreProc
cNumber        xxx links to Number
cFloat         xxx links to Float
cOctal         xxx links to Number
cOctalError    xxx links to cError
cNumbersCom    xxx cleared
cParen         xxx cleared
cBracket       xxx cleared
cNumbers       xxx cleared
cWrongComTail  xxx links to cError
cCommentL      xxx links to cComment
cCommentStart  xxx links to cComment
cComment       xxx links to Comment
cCommentError  xxx links to cError
cType          xxx links to Type
cStorageClass  xxx links to StorageClass
cConstant      xxx links to Constant
cPreCondit     xxx links to PreCondit
cPreConditMatch xxx links to cPreCondit
cCppInIf       xxx cleared
cCppInElse     xxx cleared
cCppInElse2    xxx links to cCppOutIf2
cCppOutIf      xxx cleared
cCppOutIf2     xxx links to cCppOut
cCppOutElse    xxx cleared
cCppInSkip     xxx cleared
cCppOutSkip    xxx links to cCppOutIf2
cCppOutWrapper xxx links to cPreCondit
cCppInWrapper  xxx links to cCppOutWrapper
cPreProc       xxx links to PreProc
cInclude       xxx links to Include
cDefine        xxx links to Macro
cMulti         xxx cleared
cUserCont      xxx cleared
cError         xxx links to Error
cCppOut        xxx links to Comment
cCustomParen   xxx cleared
cCustomFunc    xxx links to Function
cCustomDot     xxx cleared
cCustomPtr     xxx cleared
cAnsiFunction  xxx links to cFunction
cAnsiName      xxx links to cIdentifier
cFunction      xxx links to Function
cIdentifier    xxx links to Identifier
cBoolean       xxx links to Boolean
cppStatement   xxx links to Statement
cppAccess      xxx links to cppStatement
cppModifier    xxx links to Type
cppType        xxx links to Type
cppExceptions  xxx links to Exception
cppOperator    xxx links to Operator
cppCast        xxx links to cppStatement
cppStorageClass xxx links to StorageClass
cppStructure   xxx links to Structure
cppBoolean     xxx links to Boolean
cppConstant    xxx links to Constant
cppRawStringDelimiter xxx links to Delimiter
cppRawString   xxx links to String
cppNumber      xxx links to Number
cppMinMax      xxx cleared
cCustomScope   xxx cleared
cCustomClassKey xxx cleared
cCustomAccessKey xxx cleared
cCustomClass   xxx cleared
cCustomAngleBrackets xxx cleared
cCustomBrack   xxx cleared
cCustomAngleBracketStart xxx cleared
cCustomAngleBracketEnd xxx cleared
cCustomTemplateFunc xxx cleared
cCustomTemplate xxx cleared
cCustomOperator xxx cleared
cppSTLfunction xxx links to Function
cppSTLfunctional xxx links to Typedef
cppSTLconstant xxx links to Constant
cppSTLnamespace xxx links to Constant
cppSTLtype     xxx links to Typedef
cppSTLexception xxx links to Exception
cppSTLiterator xxx links to Typedef
cppSTLiterator_tag xxx links to Typedef
cppSTLenum     xxx links to Typedef
cppSTLios      xxx links to Function
cppSTLcast     xxx links to Statement
cppRawDelimiter xxx links to Delimiter
cppSTLbool     xxx links to Boolean
cppSTLfuntion  xxx cleared
cppSTLconcept  xxx links to Typedef
NeomakeWarningSign xxx cleared
NeomakeInfoSign xxx cleared
NeomakeErrorSign xxx cleared
NeomakeMessageSign xxx cleared
NeomakeError   xxx cleared
SpaceVimGuideCursor xxx links to Cursor
LeaderGuideKeys xxx links to Type
LeaderGuideBrackets xxx links to Delimiter
LeaderGuideGroupName xxx links to SpaceVimLeaderGuiderGroupName
LeaderGuideDesc xxx links to Identifier
LeaderGuiderPrompt xxx cterm=bold ctermfg=235 ctermbg=246 gui=bold guifg=#282828 guibg=#a89984
LeaderGuiderSep1 xxx cterm=bold ctermfg=246 ctermbg=239 gui=bold guifg=#a89984 guibg=#504945
LeaderGuiderName xxx cterm=bold ctermfg=246 ctermbg=239 gui=bold guifg=#a89984 guibg=#504945
LeaderGuiderSep2 xxx cterm=bold ctermfg=239 ctermbg=237 gui=bold guifg=#504945 guibg=#3c3836
LeaderGuiderFill xxx ctermfg=246 ctermbg=237 guifg=#a89984 guibg=#3c3836
TagbarKind     xxx links to Identifier
TagbarScope    xxx links to Title
TagbarFoldIcon xxx links to Statement
TagbarVisibilityPublic xxx links to TagbarAccessPublic
TagbarVisibilityProtected xxx links to TagbarAccessProtected
TagbarVisibilityPrivate xxx links to TagbarAccessPrivate
TagbarHelpKey  xxx links to Identifier
TagbarHelpTitle xxx links to PreProc
TagbarHelp     xxx links to Comment
TagbarNestedKind xxx links to TagbarKind
TagbarType     xxx links to Type
TagbarSignature xxx links to SpecialKey
TagbarPseudoID xxx links to NonText
TagbarHighlight xxx links to Search
TagbarAccessPublic xxx ctermfg=10 guifg=Green
TagbarAccessProtected xxx ctermfg=12 guifg=Blue
TagbarAccessPrivate xxx ctermfg=9 guifg=Red
helpHeadline   xxx links to Statement
helpSectionDelim xxx links to PreProc
helpIgnore     xxx links to Ignore
helpExample    xxx links to Comment
helpBar        xxx links to Ignore
helpHyperTextJump xxx links to Identifier
helpStar       xxx links to Ignore
helpHyperTextEntry xxx links to String
helpBacktick   xxx links to Ignore
helpNormal     xxx cleared
helpVim        xxx links to Identifier
helpOption     xxx links to Type
helpCommand    xxx links to Comment
helpHeader     xxx links to PreProc
helpGraphic    xxx cleared
helpNote       xxx links to Todo
helpWarning    xxx links to Todo
helpDeprecated xxx links to Todo
helpSpecial    xxx links to Special
helpComment    xxx links to Comment
helpConstant   xxx links to Constant
helpString     xxx links to String
helpCharacter  xxx links to Character
helpNumber     xxx links to Number
helpBoolean    xxx links to Boolean
helpFloat      xxx links to Float
helpIdentifier xxx links to Identifier
helpFunction   xxx links to Function
helpStatement  xxx links to Statement
helpConditional xxx links to Conditional
helpRepeat     xxx links to Repeat
helpLabel      xxx links to Label
helpOperator   xxx links to Operator
helpKeyword    xxx links to Keyword
helpException  xxx links to Exception
helpPreProc    xxx links to PreProc
helpInclude    xxx links to Include
helpDefine     xxx links to Define
helpMacro      xxx links to Macro
helpPreCondit  xxx links to PreCondit
helpType       xxx links to Type
helpStorageClass xxx links to StorageClass
helpStructure  xxx links to Structure
helpTypedef    xxx links to Typedef
helpSpecialChar xxx links to SpecialChar
helpTag        xxx links to Tag
helpDelimiter  xxx links to Delimiter
helpSpecialComment xxx links to SpecialComment
helpDebug      xxx links to Debug
helpUnderlined xxx links to Underlined
helpError      xxx links to Error
helpTodo       xxx links to Todo
helpURL        xxx links to String
NONE           xxx cleared
VitalOverCommandLineCursor xxx cterm=reverse gui=reverse
VitalOverCommandLineCursorOn xxx links to VitalOverCommandLineCursor
VitalOverCommandLineOnCursor xxx cterm=underline gui=underline
HitAHintShade  xxx ctermfg=242 guifg=#777777
HitAHintTarget xxx ctermfg=81 guifg=#66D9EF
HitAHintCursor xxx links to Cursor
``` 

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
