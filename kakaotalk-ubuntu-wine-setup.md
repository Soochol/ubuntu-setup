# Ubuntu에서 KakaoTalk 설치 가이드 (Wine)

> **환경**: Ubuntu 25.10 (Questing Quokka), GNOME, Wayland  
> **방법**: WineHQ 시스템 Wine + 32bit Windows 설치파일

---

## 1. 사전 준비

### 1-1. 32bit 아키텍처 활성화 및 WineHQ 저장소 추가

```bash
sudo dpkg --add-architecture i386
sudo mkdir -pm755 /etc/apt/keyrings

# GPG 키 등록 (dearmor 방식)
wget -qO- https://dl.winehq.org/wine-builds/winehq.key | gpg --dearmor | sudo tee /etc/apt/keyrings/winehq-archive.key > /dev/null

# 저장소 추가 (Ubuntu 25.10 Questing 기준)
sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/ubuntu/dists/questing/winehq-questing.sources

sudo apt update
```

### 1-2. Wine 설치

```bash
sudo apt install --install-recommends winehq-stable -y
```

### 1-3. 한글 폰트 설치

```bash
sudo apt install fonts-nanum -y
```

---

## 2. Wine Prefix 생성 및 카카오톡 설치

### 2-1. 32bit Prefix 생성

```bash
WINEPREFIX=~/.wine-kakao wineboot
```

> ⚠️ 시스템 Wine은 wow64 모드로 동작하므로 `WINEARCH=win32` 설정 불필요

### 2-2. 카카오톡 설치

**32bit 설치파일 사용** (64bit보다 Wine 호환성이 좋음)

```bash
WINEPREFIX=~/.wine-kakao LANG=ko_KR.UTF-8 wine ~/Downloads/KakaoTalk_Setup.exe
```

> 설치 언어는 **영어(English)** 로 선택 (한국어 선택 시 폰트 문제 발생 가능)

---

## 3. 한글 폰트 설정

### 3-1. 폰트 파일 복사

```bash
cp /usr/share/fonts/truetype/nanum/*.ttf ~/.wine-kakao/drive_c/windows/Fonts/
```

### 3-2. 레지스트리 등록

```bash
cat > /tmp/korean_font.reg << 'EOF'
REGEDIT4

[HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\Fonts]
"NanumGothic (TrueType)"="NanumGothic.ttf"
"NanumGothicBold (TrueType)"="NanumGothicBold.ttf"

[HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes]
"MS Shell Dlg"="NanumGothic"
"MS Shell Dlg 2"="NanumGothic"
"Tahoma"="NanumGothic"
"Arial"="NanumGothic"
"Gulim"="NanumGothic"
"Dotum"="NanumGothic"
"Batang"="NanumGothic"
"Malgun Gothic"="NanumGothic"
EOF

WINEPREFIX=~/.wine-kakao wine regedit /tmp/korean_font.reg
```

---

## 4. ibus 한글 입력 설정

### 4-1. ibus 입력 스타일 설정 (핵심)

채팅 입력창에서 한글이 깨지는 문제를 해결하는 레지스트리 설정:

```bash
cat > /tmp/ibus_fix.reg << 'EOF'
REGEDIT4

[HKEY_CURRENT_USER\Software\Wine\X11 Driver]
"inputStyle"="overthespot"
EOF

WINEPREFIX=~/.wine-kakao wine regedit /tmp/ibus_fix.reg
```

> - `root`: 한글 입력은 되지만 preedit(입력 미리보기) 폰트 깨짐
> - `overthespot`: preedit가 팝업으로 표시되어 폰트 깨짐 없음 ✅

---

## 5. 실행 스크립트 생성

```bash
cat > ~/.local/bin/kakaotalk.sh << 'EOF'
#!/bin/bash
if pgrep -f "KakaoTalk.exe" > /dev/null; then
    exit 0
fi
export WINEPREFIX=/home/dev/.wine-kakao
export WAYLAND_DISPLAY=wayland-0
export XDG_RUNTIME_DIR=/run/user/1000
export WINEDLLOVERRIDES="dwrite=d;libglesv2=d;msxml6=n,b"
export LANG=ko_KR.UTF-8
export XMODIFIERS="@im=ibus"
export GTK_IM_MODULE=ibus
export QT_IM_MODULE=ibus
exec wine "/home/dev/.wine-kakao/drive_c/Program Files (x86)/Kakao/KakaoTalk/KakaoTalk.exe"
EOF

chmod +x ~/.local/bin/kakaotalk.sh
```

---

## 6. GNOME 독 아이콘 등록

### 6-1. 아이콘 이미지 준비

카카오톡 아이콘 PNG 파일을 아래 경로에 저장:

```
~/.local/share/icons/hicolor/256x256/apps/kakaotalk.png
```

### 6-2. .desktop 파일 생성

```bash
cat > ~/.local/share/applications/kakaotalk.desktop << 'EOF'
[Desktop Entry]
Name=KakaoTalk
Comment=KakaoTalk Messenger
Exec=/home/dev/.local/bin/kakaotalk.sh
Icon=/home/dev/.local/share/icons/hicolor/256x256/apps/kakaotalk.png
Terminal=false
Type=Application
Categories=Network;InstantMessaging;
StartupNotify=false
StartupWMClass=kakaotalk.exe
EOF

chmod +x ~/.local/share/applications/kakaotalk.desktop
update-desktop-database ~/.local/share/applications/
```

### 6-3. 독에 고정

```bash
# 현재 독 목록에 추가
gsettings set org.gnome.shell favorite-apps "$(gsettings get org.gnome.shell favorite-apps | sed "s/]$/, 'kakaotalk.desktop']/")"
```

또는 카카오톡 실행 후 독 아이콘 우클릭 → **"Add to Favorites"**

---

## 7. 실행 구조

```
독 아이콘 (kakaotalk.desktop)
    └── kakaotalk.sh (환경변수 설정)
            └── wine KakaoTalk.exe
```

---

## 알려진 한계

| 항목 | 상태 | 비고 |
|------|------|------|
| 한글 표시 | ✅ 정상 | 나눔고딕 폰트 설정 |
| 한글 입력 | ✅ 정상 | inputStyle=overthespot |
| 한/영 전환 (채팅창) | ⚠️ 제한적 | ibus의 Wine 내 키 전달 한계 |
| 자동 업데이트 | ❌ 불가 | 수동 재설치 필요 |
| 트레이 아이콘 색상 | ⚠️ 회색 | Wine 트레이 렌더링 한계 |

---

## 문제 해결

### 카카오톡이 실행되지 않을 때
```bash
pkill -f KakaoTalk
~/.local/bin/kakaotalk.sh
```

### 폰트가 다시 깨질 때
레지스트리 재적용:
```bash
WINEPREFIX=~/.wine-kakao wine regedit /tmp/ibus_fix.reg
WINEPREFIX=~/.wine-kakao wine regedit /tmp/korean_font.reg
```

### Wine 버전 확인
```bash
wine --version
```
