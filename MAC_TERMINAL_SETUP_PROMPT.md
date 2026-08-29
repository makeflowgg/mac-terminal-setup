# Mac 터미널 환경 재현 프롬프트

Mac을 초기화한 뒤 이 문서 전체를 Codex에 붙여넣어 실행한다. 목표는 Ghostty + Catppuccin Mocha + Starship + Zsh 개발 환경을 재현하는 것이다. Claude Code는 설치하지 않는다.

이 문서는 한 번에 무조건 실행하는 셸 스크립트가 아니다. Codex는 각 번호를 실행·검증·보고·승인 요청의 순환으로 처리해야 한다. 설정 파일은 아래에 명시된 내용을 실제 파일에 작성한 뒤 문법 검사를 실행한다.

## 반드시 지킬 실행 규칙

1. 작업은 아래 번호 순서대로만 실행한다. 서로 다른 번호의 명령을 동시에 실행하지 않는다.
2. 한 번호의 명령이 끝나면 실제 확인 명령을 실행하고, 결과를 한국어로 보고한 뒤 사용자에게 다음 번호를 진행해도 되는지 묻는다.
3. 확인이 실패하면 원인을 설명하고 수정한 뒤 같은 번호를 다시 확인한다. 성공 전에는 다음 번호로 넘어가지 않는다.
4. 기존 설정 파일은 먼저 타임스탬프 백업하고, 설정을 중복해서 추가하지 않는다.
5. sudo, 관리자 승인, 네트워크 권한, 로그인 또는 인증이 필요하면 실행 전에 사용자에게 알린다.
6. TLS/SSL 검증을 끄거나 인증서 검사를 우회하지 않는다. 비밀번호·API 키·토큰을 수집하거나 파일에 저장하지 않는다.
7. Claude Code 및 Claude 관련 패키지는 설치하지 않는다.

## 0. 현재 상태 확인

다음 명령으로 운영체제와 셸을 확인하고, 각 결과를 기록한다.

    sw_vers
    uname -m
    printf 'shell=%s\n' "$SHELL"
    command -v brew || true
    command -v ghostty || true
    command -v starship || true
    command -v mise || true

## 1. Homebrew 설치 및 확인

Homebrew가 없을 때만 공식 설치기를 실행한다. Apple Silicon은 /opt/homebrew, Intel Mac은 /usr/local을 사용한다.

    if ! command -v brew >/dev/null 2>&1; then
      /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    fi
    if [ -x /opt/homebrew/bin/brew ]; then
      eval "$(/opt/homebrew/bin/brew shellenv)"
      grep -qxF 'eval "$(/opt/homebrew/bin/brew shellenv)"' "$HOME/.zprofile" 2>/dev/null || \
        printf '%s\n' 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> "$HOME/.zprofile"
    elif [ -x /usr/local/bin/brew ]; then
      eval "$(/usr/local/bin/brew shellenv)"
      grep -qxF 'eval "$(/usr/local/bin/brew shellenv)"' "$HOME/.zprofile" 2>/dev/null || \
        printf '%s\n' 'eval "$(/usr/local/bin/brew shellenv)"' >> "$HOME/.zprofile"
    fi
    brew --version

확인: brew 버전이 출력되어야 한다.

## 2. Ghostty와 글꼴 설치

    brew install --cask ghostty
    brew install --cask font-hack-nerd-font
    brew install --cask font-noto-sans-cjk-kr

확인: brew 목록과 글꼴 디렉터리를 확인한다.

    brew list --cask | grep -E 'ghostty|font-hack-nerd-font|font-noto-sans-cjk-kr'
    test -d "$HOME/Library/Fonts" && find "$HOME/Library/Fonts" -maxdepth 1 -iname '*Hack*' -o -iname '*Noto*' | head

## 3. CLI 도구 설치

    brew install lsd bat fzf fd ripgrep git-delta btop dust duf fastfetch neovim
    brew install zoxide lazygit navi starship mise gemini-cli

확인:

    for tool in lsd bat fzf fd rg delta btop dust duf fastfetch nvim zoxide lazygit navi starship mise gemini; do
      command -v "$tool" || { echo "missing: $tool"; exit 1; }
    done

## 4. Oh My Zsh 설치

    if [ ! -d "$HOME/.oh-my-zsh" ]; then
      RUNZSH=no CHSH=no sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended
    fi
    test -f "$HOME/.oh-my-zsh/oh-my-zsh.sh"

## 5. Zinit 및 SCM Breeze 설치

    if [ ! -d "$HOME/.local/share/zinit/zinit.git" ]; then
      mkdir -p "$HOME/.local/share/zinit"
      git clone https://github.com/zdharma-continuum/zinit "$HOME/.local/share/zinit/zinit.git"
    fi
    if [ ! -d "$HOME/.scm_breeze" ]; then
      git clone https://github.com/scmbreeze/scm_breeze.git "$HOME/.scm_breeze"
      "$HOME/.scm_breeze/install.sh"
    fi
    test -f "$HOME/.local/share/zinit/zinit.git/zinit.zsh"
    test -f "$HOME/.scm_breeze/scm_breeze.sh"

## 6. mise로 Node.js 24 설치

    eval "$(mise activate zsh)"
    mise use --global node@24
    mise current node
    node --version
    npm --version

Node 24.x가 확인되어야 한다.

## 7. Codex CLI 설치

이미 codex 명령이 있으면 설치를 건너뛴다. 없을 때만 OpenAI 공식 standalone 설치기를 사용한다. npm 인증서 오류가 나면 SSL 검증을 끄지 말고 중단하여 사용자에게 보고한다.

    if ! command -v codex >/dev/null 2>&1; then
      curl -fsSL https://chatgpt.com/codex/install.sh | sh
    fi
    command -v codex
    codex --version || true

## 8. Ghostty 설정 파일 생성

    mkdir -p "$HOME/.config/ghostty"
    if [ -f "$HOME/.config/ghostty/config" ]; then
      cp "$HOME/.config/ghostty/config" "$HOME/.config/ghostty/config.backup.$(date +%Y%m%d-%H%M%S)"
    fi

다음 내용을 "$HOME/.config/ghostty/config"에 저장한다.

    theme = Catppuccin Mocha
    font-family = Hack Nerd Font Mono
    font-family = Noto Sans CJK KR
    window-padding-x = 12
    window-padding-y = 10
    window-decoration = true
    macos-titlebar-style = tabs
    background-opacity = 0.9
    background-blur-radius = 20
    palette = 8=#7f849c
    cursor-style = block
    cursor-style-blink = false
    mouse-hide-while-typing = true
    copy-on-select = true
    clipboard-paste-protection = false
    scrollback-limit = 10000
    macos-option-as-alt = true
    shell-integration = zsh
    shell-integration-features = cursor,sudo,title
    keybind = cmd+d=new_split:right
    keybind = cmd+shift+d=new_split:down
    keybind = cmd+w=close_surface
    keybind = cmd+alt+left=goto_split:left
    keybind = cmd+alt+right=goto_split:right
    keybind = cmd+alt+up=goto_split:top
    keybind = cmd+alt+down=goto_split:bottom

확인: Ghostty를 새로 실행하거나 설정 재로드 후 Catppuccin Mocha, 두 글꼴, 투명 배경과 분할 단축키를 확인한다.

## 9. Starship 설정 파일 생성

    mkdir -p "$HOME/.config"
    if [ -f "$HOME/.config/starship.toml" ]; then
      cp "$HOME/.config/starship.toml" "$HOME/.config/starship.toml.backup.$(date +%Y%m%d-%H%M%S)"
    fi

다음 내용을 "$HOME/.config/starship.toml"에 저장한다.

    "$schema" = 'https://starship.rs/config-schema.json'
    format = """
    [](mauve)\
    $directory\
    [](fg:mauve bg:blue)\
    $git_branch\
    $git_status\
    [](fg:blue) \
    $cmd_duration\
    $character"""
    palette = 'catppuccin_mocha'

    [directory]
    style = "bg:mauve fg:crust"
    format = "[ $path ]($style)"
    truncation_length = 3
    truncation_symbol = "…/"

    [git_branch]
    symbol = " "
    style = "bg:blue"
    format = '[[ $symbol$branch ](fg:crust bg:blue)]($style)'

    [git_status]
    style = "bg:blue"
    format = '[[($all_status$ahead_behind )](fg:crust bg:blue)]($style)'

    [line_break]
    disabled = true

    [character]
    success_symbol = '[❯](bold fg:green)'
    error_symbol = '[❯](bold fg:red)'

    [cmd_duration]
    min_time = 2_000
    format = "[ 󱎫 $duration](fg:overlay1) "

    [palettes.catppuccin_mocha]
    rosewater = "#f5e0dc"
    flamingo = "#f2cdcd"
    pink = "#f5c2e7"
    mauve = "#cba6f7"
    red = "#f38ba8"
    maroon = "#eba0ac"
    peach = "#fab387"
    yellow = "#f9e2af"
    green = "#a6e3a1"
    teal = "#94e2d5"
    sky = "#89dceb"
    sapphire = "#74c7ec"
    blue = "#89b4fa"
    lavender = "#b4befe"
    text = "#cdd6f4"
    subtext1 = "#bac2de"
    subtext0 = "#a6adc8"
    overlay2 = "#9399b2"
    overlay1 = "#7f849c"
    overlay0 = "#6c7086"
    surface2 = "#585b70"
    surface1 = "#45475a"
    surface0 = "#313244"
    base = "#1e1e2e"
    mantle = "#181825"
    crust = "#11111b"

확인: starship 설정 검사 명령을 실행하고 오류가 없어야 한다.

    starship config || true

## 10. .zshrc 구성

기존 "$HOME/.zshrc"는 타임스탬프 백업 후 아래 기능을 중복 없이 반영한다.

    cp "$HOME/.zshrc" "$HOME/.zshrc.backup.$(date +%Y%m%d-%H%M%S)" 2>/dev/null || true

반영할 내용:

    # Fastfetch (interactive shells only)
    if [[ $- == *i* ]] && command -v fastfetch &>/dev/null; then
      fastfetch
    fi

    export ZSH="$HOME/.oh-my-zsh"
    ZSH_THEME=""
    plugins=(git)
    source "$ZSH/oh-my-zsh.sh"

    if command -v starship &>/dev/null; then
      eval "$(starship init zsh)"
    fi

    if [[ -f "$HOME/.local/share/zinit/zinit.git/zinit.zsh" ]]; then
      source "$HOME/.local/share/zinit/zinit.git/zinit.zsh"
      autoload -Uz _zinit
      (( ${+_comps} )) && _comps[zinit]=_zinit
      zinit light zdharma-continuum/fast-syntax-highlighting
      zinit light zsh-users/zsh-autosuggestions
      zinit light zsh-users/zsh-completions
    fi

    if [[ -z "$CLAUDECODE" ]]; then
      [ -s "$HOME/.scm_breeze/scm_breeze.sh" ] && source "$HOME/.scm_breeze/scm_breeze.sh"
    fi

    alias ls="lsd"
    alias ll="lsd -la"
    alias lt="lsd --tree"
    alias cat="bat"
    alias find="fd"
    alias grep="rg"
    alias top="btop"
    alias df="duf"
    alias du="dust"
    alias vim="nvim"
    alias vi="nvim"
    alias lg="lazygit"
    alias c="clear"
    alias ..="cd .."
    alias ...="cd ../.."

    if command -v fzf &>/dev/null; then source <(fzf --zsh); fi
    if command -v zoxide &>/dev/null; then eval "$(zoxide init --cmd cd zsh)"; fi
    if command -v navi &>/dev/null; then eval "$(navi widget zsh)"; fi
    if command -v mise &>/dev/null; then eval "$(mise activate zsh)"; fi

    tools() {
      local choice
      choice=$(printf '%s\n' btop lazygit duf dust fastfetch | fzf --prompt='tool> ') || return
      case "$choice" in
        btop) btop;; lazygit) lazygit;; duf) duf;; dust) dust;; fastfetch) fastfetch;;
      esac
    }

확인:

    zsh -n "$HOME/.zshrc"
    zsh -ic 'command -v starship; command -v fzf; command -v zoxide; command -v navi; node --version; alias ll' </dev/null

## 11. Git Delta 설정

    git config --global core.pager delta
    git config --global interactive.diffFilter "delta --color-only"
    git config --global delta.navigate true
    git config --global delta.side-by-side true
    git config --global --get core.pager
    git config --global --get interactive.diffFilter
    git config --global --get delta.navigate
    git config --global --get delta.side-by-side

## 12. 최종 검증

다음 검증을 모두 실행하고 실패한 항목이 있으면 수정 후 재검증한다.

    brew doctor || true
    zsh -n "$HOME/.zshrc"
    test -s "$HOME/.config/ghostty/config"
    test -s "$HOME/.config/starship.toml"
    command -v ghostty || true
    command -v codex || true
    command -v gemini || true
    mise current node
    git config --global --get core.pager

성공하면 새 Ghostty 창에서 다음을 확인한다.

    fastfetch
    pwd
    git status
    codex --version
    gemini --version

## 13. 사용자 인증 안내

설치가 모두 성공한 뒤에만 사용자가 직접 다음 명령을 실행한다. 인증 화면, 브라우저 로그인, API 키 입력은 Codex가 대신 처리하거나 기록하지 않는다.

    codex
    gemini

최종 보고에는 설치 성공 항목, 실패 또는 건너뛴 항목, 백업 파일 위치, 사용자가 직접 해야 할 인증 단계를 간단히 정리한다.
