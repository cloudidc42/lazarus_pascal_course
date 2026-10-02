# Part 48 - Application Packaging ใน Lazarus/Pascal

## บทนำ

การ package แอปพลิเคชันให้พร้อมแจกจ่ายต้องการการกำหนดค่า build ที่เหมาะสม การ optimize ขนาดไฟล์ และการสร้าง package format ที่เหมาะกับแต่ละ platform

---

## 48.1 Debug vs Release Configurations

### ความแตกต่างระหว่าง Debug และ Release

```pascal
{$mode objfpc}{$H+}

// การตรวจสอบ build mode ใน code
program BuildModeDemo;

uses
  SysUtils;

procedure ShowBuildInfo;
begin
  WriteLn('=== Build Information ===');
  
  {$IFDEF DEBUG}
  WriteLn('Mode: DEBUG');
  WriteLn('Assertions: ENABLED');
  WriteLn('Range checks: ENABLED');
  {$ELSE}
  WriteLn('Mode: RELEASE');
  WriteLn('Assertions: DISABLED');
  WriteLn('Optimizations: ENABLED');
  {$ENDIF}
  
  {$IFDEF FPC}
  WriteLn('Compiler: Free Pascal Compiler');
  WriteLn('FPC Version: ' + 
    IntToStr({$I %FPCVERSION_MAJOR%}) + '.' +
    IntToStr({$I %FPCVERSION_MINOR%}) + '.' +
    IntToStr({$I %FPCVERSION_PATCH%}));
  {$ENDIF}
  
  WriteLn('Target OS: {$I %FPCTARGET%}');
  WriteLn('CPU: {$I %FPCCPU%}');
  WriteLn('Compiled: {$I %DATE%} {$I %TIME%}');
end;

procedure DemoOptimizations;
var
  i: Integer;
  Sum: Int64;
  StartTime: TDateTime;
begin
  WriteLn('');
  WriteLn('=== Performance Test ===');
  
  StartTime := Now;
  Sum := 0;
  
  // Loop ที่จะถูก optimize ใน Release mode
  for i := 1 to 100000000 do
    Inc(Sum, i);
  
  WriteLn(Format('Sum: %d', [Sum]));
  WriteLn(Format('Time: %.3f seconds', [(Now - StartTime) * 86400]));
end;

begin
  ShowBuildInfo;
  DemoOptimizations;
end.
```

### Project Options ใน Lazarus

**Debug Build Settings:**
```
Project > Project Options > Compiler Options > Code Generation
  - Optimization level: None (-O0)
  - Generate debug info: Yes (-g)
  - Generate debug info for gdb: Yes (-gg)
  - Generate line info: Yes (-gl)

Compiler Options > Checks:
  - Range checking: Yes (-Cr)
  - Overflow checking: Yes (-Co)
  - IO checking: Yes (-Ci)
  - Stack size checking: Yes (-Cs)
  - Pointer dereferencing: Yes (-Cp)

Compiler Options > Verbosity:
  - Write FPC logo: Yes
  - Show warnings: Yes
  - Show notes: Yes
  - Show hints: Yes
```

**Release Build Settings:**
```
Project > Project Options > Compiler Options > Code Generation
  - Optimization level: Level 3 (-O3) หรือ -O2
  - Generate debug info: No
  - Smart link: Yes (-CX)
  - Strip symbols: Yes (Linker > Strip symbols: -Xs)

Compiler Options > Checks:
  - ปิด checks ทั้งหมด

Compiler Options > Verbosity:
  - Write FPC logo: No
  - Show warnings: No (ใน production)
```

---

## 48.2 Compiler Optimizations

### Optimization Flags ใน FPC

```pascal
{$mode objfpc}{$H+}

// การใช้ compiler directives สำหรับ optimization
{$OPTIMIZATION ON}        // เปิด optimization
{$OPTIMIZATION REGVAR}    // ใช้ registers สำหรับ local variables
{$OPTIMIZATION PEEPHOLE}  // Peephole optimization
{$OPTIMIZATION CSE}       // Common subexpression elimination

// หรือ inline functions
procedure {$IFDEF RELEASE}inline{$ENDIF} FastAdd(var A: Integer; B: Integer);
begin
  Inc(A, B);
end;

// ตัวอย่าง: String optimization
program StringOptimization;

uses
  SysUtils;

// ใช้ ShortString ถ้าขนาดแน่นอน (เร็วกว่า AnsiString สำหรับขนาดเล็ก)
function GetShortName: ShortString;
begin
  Result := 'Hello World';
end;

// ใช้ AnsiString สำหรับ dynamic strings
function GetDynamicText(Count: Integer): AnsiString;
var
  SB: TStringBuilder;  // ถ้ามี
  i: Integer;
begin
  // ไม่ดี: สร้าง string ใหม่ทุก iteration
  Result := '';
  for i := 1 to Count do
    Result := Result + IntToStr(i) + ' ';
end;

function GetDynamicTextFast(Count: Integer): AnsiString;
var
  Parts: array of AnsiString;
  i: Integer;
begin
  // ดีกว่า: สร้าง array แล้ว join
  SetLength(Parts, Count);
  for i := 0 to Count - 1 do
    Parts[i] := IntToStr(i + 1);
  Result := String.Join(' ', Parts);  // หรือใช้ loop ด้วย buffer
end;

begin
  WriteLn(GetShortName);
end.
```

### Compiler Switches ใน Command Line

```bash
# Debug build
fpc -g -gl -gw -O0 -Cr -Co -Ci myapp.pas

# Release build (maximum optimization)
fpc -O3 -CX -Xs -XX -fPIC myapp.pas

# Release build for specific CPU
fpc -O3 -CX -Xs -Cp686 myapp.pas       # x86 with 686 instructions
fpc -O3 -CX -Xs -CpSSE3 myapp.pas      # SSE3 support
fpc -O3 -CX -Xs -CpAVX2 myapp.pas      # AVX2 support (modern x64)

# Cross-compile
FPC_CROSSCOMPILE=1 fpc -O2 -T linux -P x86_64 myapp.pas
```

---

## 48.3 Stripping Symbols

### การลดขนาดไฟล์

```bash
#!/bin/bash
# strip_and_compress.sh

APP_NAME="myapp"
BUILD_DIR="./dist"

echo "=== Build and Strip ==="

# Build release
lazbuild --build-mode=Release "${APP_NAME}.lpi"

echo "Size before strip:"
ls -lh "${BUILD_DIR}/${APP_NAME}"

# Strip debug symbols (Linux/macOS)
if [ "$(uname)" = "Linux" ]; then
  strip "${BUILD_DIR}/${APP_NAME}"
elif [ "$(uname)" = "Darwin" ]; then
  strip -S "${BUILD_DIR}/${APP_NAME}"
fi

echo "Size after strip:"
ls -lh "${BUILD_DIR}/${APP_NAME}"

# UPX compression (optional - ลดขนาดเพิ่มเติม)
if command -v upx &> /dev/null; then
  echo "Compressing with UPX..."
  upx --best "${BUILD_DIR}/${APP_NAME}"
  echo "Size after UPX:"
  ls -lh "${BUILD_DIR}/${APP_NAME}"
fi
```

### Pascal Code สำหรับตรวจสอบขนาด Build

```pascal
{$mode objfpc}{$H+}

program BuildSizeInfo;

uses
  SysUtils;

type
  TBuildInfo = record
    AppName: String;
    Version: String;
    BuildDate: TDateTime;
    IsDebug: Boolean;
    IsOptimized: Boolean;
  end;

function GetBuildInfo: TBuildInfo;
begin
  Result.AppName := 'MyApplication';
  Result.Version := '2.0.0';
  Result.BuildDate := Now;
  
  {$IFDEF DEBUG}
  Result.IsDebug := True;
  Result.IsOptimized := False;
  {$ELSE}
  Result.IsDebug := False;
  Result.IsOptimized := True;
  {$ENDIF}
end;

procedure PrintBuildInfo(const Info: TBuildInfo);
begin
  WriteLn('=========================');
  WriteLn('Application: ', Info.AppName);
  WriteLn('Version: ', Info.Version);
  WriteLn('Build Date: ', DateTimeToStr(Info.BuildDate));
  WriteLn('Debug Build: ', BoolToStr(Info.IsDebug, 'Yes', 'No'));
  WriteLn('Optimized: ', BoolToStr(Info.IsOptimized, 'Yes', 'No'));
  WriteLn('=========================');
end;

begin
  PrintBuildInfo(GetBuildInfo);
end.
```

---

## 48.4 Cross-Compilation

### Cross-Compile ด้วย FPC/Lazarus

```bash
#!/bin/bash
# cross_compile.sh

APP_NAME="myapp"
SOURCE_DIR="./src"
OUTPUT_DIR="./release"

# ตรวจสอบว่า cross-compiler ถูกติดตั้งแล้ว
check_cross_compiler() {
  local TARGET=$1
  if ! fpc -T${TARGET} -v 2>&1 | grep -q "Free Pascal"; then
    echo "Warning: Cross-compiler for ${TARGET} not found"
    return 1
  fi
  return 0
}

# Build สำหรับ Linux x86_64
build_linux_x64() {
  echo "Building for Linux x86_64..."
  mkdir -p "${OUTPUT_DIR}/linux-x64"
  
  fpc \
    -Tlinux \
    -Px86_64 \
    -O2 -CX -Xs \
    -FU"${OUTPUT_DIR}/linux-x64/units" \
    -o"${OUTPUT_DIR}/linux-x64/${APP_NAME}" \
    "${SOURCE_DIR}/${APP_NAME}.pas"
  
  if [ $? -eq 0 ]; then
    echo "Linux x64 build successful"
    strip "${OUTPUT_DIR}/linux-x64/${APP_NAME}"
  else
    echo "Linux x64 build FAILED"
  fi
}

# Build สำหรับ Windows x86_64
build_windows_x64() {
  echo "Building for Windows x86_64..."
  mkdir -p "${OUTPUT_DIR}/windows-x64"
  
  # ต้องมี FPC cross-compiler สำหรับ Windows
  fpc \
    -Twin64 \
    -Px86_64 \
    -O2 -CX -Xs \
    -FU"${OUTPUT_DIR}/windows-x64/units" \
    -o"${OUTPUT_DIR}/windows-x64/${APP_NAME}.exe" \
    "${SOURCE_DIR}/${APP_NAME}.pas"
  
  if [ $? -eq 0 ]; then
    echo "Windows x64 build successful"
    x86_64-w64-mingw32-strip "${OUTPUT_DIR}/windows-x64/${APP_NAME}.exe" 2>/dev/null || true
  else
    echo "Windows x64 build FAILED"
  fi
}

# Build สำหรับ macOS (ต้องรันบน macOS หรือใช้ cross-tools)
build_macos_x64() {
  echo "Building for macOS x86_64..."
  mkdir -p "${OUTPUT_DIR}/macos-x64"
  
  fpc \
    -Tdarwin \
    -Px86_64 \
    -O2 -CX -Xs \
    -FU"${OUTPUT_DIR}/macos-x64/units" \
    -o"${OUTPUT_DIR}/macos-x64/${APP_NAME}" \
    "${SOURCE_DIR}/${APP_NAME}.pas"
}

# Build สำหรับ ARM (Raspberry Pi)
build_linux_arm() {
  echo "Building for Linux ARM..."
  mkdir -p "${OUTPUT_DIR}/linux-arm"
  
  fpc \
    -Tlinux \
    -Parm \
    -Cfvfpv3 \
    -O2 \
    -FU"${OUTPUT_DIR}/linux-arm/units" \
    -o"${OUTPUT_DIR}/linux-arm/${APP_NAME}" \
    "${SOURCE_DIR}/${APP_NAME}.pas"
}

# รัน builds
mkdir -p "${OUTPUT_DIR}"
build_linux_x64
build_windows_x64
# build_macos_x64  # uncomment on macOS
# build_linux_arm  # uncomment ถ้ามี ARM cross-compiler

echo ""
echo "=== Build Results ==="
find "${OUTPUT_DIR}" -type f | while read f; do
  echo "  $f ($(du -sh "$f" | cut -f1))"
done
```

### Lazarus Cross-Build ด้วย lazbuild

```bash
#!/bin/bash
# lazarus_cross_build.sh

PROJECT_FILE="myapp.lpi"

# Build mode configurations (กำหนดใน Lazarus IDE ก่อน)

# Windows 64-bit
echo "Building Windows 64-bit..."
lazbuild \
  --build-mode=Release_Win64 \
  --os=win64 \
  --cpu=x86_64 \
  "${PROJECT_FILE}"

# Linux 64-bit
echo "Building Linux 64-bit..."
lazbuild \
  --build-mode=Release_Linux64 \
  --os=linux \
  --cpu=x86_64 \
  "${PROJECT_FILE}"

# macOS 64-bit (Intel)
echo "Building macOS x64..."
lazbuild \
  --build-mode=Release_macOS64 \
  --os=darwin \
  --cpu=x86_64 \
  "${PROJECT_FILE}"

# macOS ARM (Apple Silicon)
echo "Building macOS ARM64..."
lazbuild \
  --build-mode=Release_macOSARM \
  --os=darwin \
  --cpu=aarch64 \
  "${PROJECT_FILE}"
```

---

## 48.5 Linux DEB/RPM Packaging

### สร้าง DEB Package

```bash
#!/bin/bash
# create_deb.sh

APP_NAME="myapp"
VERSION="2.0.1"
ARCH="amd64"
MAINTAINER="Developer Name <dev@example.com>"
DESCRIPTION="MyApp - A Lazarus Application"

PACKAGE_DIR="${APP_NAME}_${VERSION}_${ARCH}"

# สร้าง directory structure
mkdir -p "${PACKAGE_DIR}/DEBIAN"
mkdir -p "${PACKAGE_DIR}/usr/bin"
mkdir -p "${PACKAGE_DIR}/usr/share/${APP_NAME}"
mkdir -p "${PACKAGE_DIR}/usr/share/applications"
mkdir -p "${PACKAGE_DIR}/usr/share/pixmaps"
mkdir -p "${PACKAGE_DIR}/usr/share/doc/${APP_NAME}"
mkdir -p "${PACKAGE_DIR}/etc/${APP_NAME}"

# Copy files
cp "dist/linux-x64/${APP_NAME}" "${PACKAGE_DIR}/usr/bin/"
chmod 755 "${PACKAGE_DIR}/usr/bin/${APP_NAME}"

cp -r "dist/resources/" "${PACKAGE_DIR}/usr/share/${APP_NAME}/"
cp "resources/app_icon_48.png" "${PACKAGE_DIR}/usr/share/pixmaps/${APP_NAME}.png"
cp "docs/copyright" "${PACKAGE_DIR}/usr/share/doc/${APP_NAME}/"
gzip -9 "changelog.gz" 2>/dev/null
cp "changelog.gz" "${PACKAGE_DIR}/usr/share/doc/${APP_NAME}/" 2>/dev/null || true

# สร้าง .desktop file
cat > "${PACKAGE_DIR}/usr/share/applications/${APP_NAME}.desktop" << EOF
[Desktop Entry]
Version=1.0
Type=Application
Name=MyApplication
Comment=${DESCRIPTION}
Exec=/usr/bin/${APP_NAME}
Icon=${APP_NAME}
Terminal=false
Categories=Office;Utility;
Keywords=myapp;productivity;
StartupNotify=true
EOF

# สร้าง DEBIAN/control
cat > "${PACKAGE_DIR}/DEBIAN/control" << EOF
Package: ${APP_NAME}
Version: ${VERSION}
Section: utils
Priority: optional
Architecture: ${ARCH}
Installed-Size: $(du -sk "${PACKAGE_DIR}" | cut -f1)
Depends: libglib2.0-0 (>= 2.48), libgtk-3-0 (>= 3.18)
Maintainer: ${MAINTAINER}
Description: ${DESCRIPTION}
 This is a cross-platform application built with Lazarus/Free Pascal.
 It provides comprehensive features for managing data and reports.
EOF

# สร้าง DEBIAN/postinst (run after install)
cat > "${PACKAGE_DIR}/DEBIAN/postinst" << 'EOF'
#!/bin/sh
set -e

# Update icon cache
if command -v gtk-update-icon-cache > /dev/null 2>&1; then
  gtk-update-icon-cache /usr/share/pixmaps/ 2>/dev/null || true
fi

# Update desktop database
if command -v update-desktop-database > /dev/null 2>&1; then
  update-desktop-database /usr/share/applications/ 2>/dev/null || true
fi

echo "MyApplication installed successfully!"
EOF
chmod 755 "${PACKAGE_DIR}/DEBIAN/postinst"

# สร้าง DEBIAN/prerm (run before remove)
cat > "${PACKAGE_DIR}/DEBIAN/prerm" << 'EOF'
#!/bin/sh
set -e
echo "Removing MyApplication..."
EOF
chmod 755 "${PACKAGE_DIR}/DEBIAN/prerm"

# สร้าง DEBIAN/postrm (run after remove)
cat > "${PACKAGE_DIR}/DEBIAN/postrm" << 'EOF'
#!/bin/sh
set -e

if command -v update-desktop-database > /dev/null 2>&1; then
  update-desktop-database /usr/share/applications/ 2>/dev/null || true
fi
EOF
chmod 755 "${PACKAGE_DIR}/DEBIAN/postrm"

# Build DEB package
dpkg-deb --build "${PACKAGE_DIR}"

echo "Package created: ${PACKAGE_DIR}.deb"
```

### สร้าง RPM Package

```bash
#!/bin/bash
# create_rpm.sh

APP_NAME="myapp"
VERSION="2.0.1"
RELEASE="1"
ARCH="x86_64"

# สร้าง RPM spec file
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}

cat > ~/rpmbuild/SPECS/${APP_NAME}.spec << EOF
Name:           ${APP_NAME}
Version:        ${VERSION}
Release:        ${RELEASE}%{?dist}
Summary:        MyApp - A Lazarus Application

License:        GPL-3.0
URL:            https://www.example.com
Source0:        %{name}-%{version}.tar.gz

BuildRequires:  gtk3-devel
Requires:       gtk3 >= 3.18, glib2 >= 2.48

%description
This is a cross-platform application built with Lazarus/Free Pascal.
It provides comprehensive features for managing data and reports.

%prep
%setup -q

%install
mkdir -p %{buildroot}/usr/bin
mkdir -p %{buildroot}/usr/share/%{name}
mkdir -p %{buildroot}/usr/share/applications
mkdir -p %{buildroot}/usr/share/pixmaps

install -m 755 %{name} %{buildroot}/usr/bin/%{name}
install -m 644 %{name}.desktop %{buildroot}/usr/share/applications/
install -m 644 %{name}.png %{buildroot}/usr/share/pixmaps/
cp -r resources/* %{buildroot}/usr/share/%{name}/

%files
%license LICENSE
%doc README.md
/usr/bin/%{name}
/usr/share/%{name}/
/usr/share/applications/%{name}.desktop
/usr/share/pixmaps/%{name}.png

%post
gtk-update-icon-cache /usr/share/pixmaps/ 2>/dev/null || true
update-desktop-database /usr/share/applications/ 2>/dev/null || true

%postun
gtk-update-icon-cache /usr/share/pixmaps/ 2>/dev/null || true
update-desktop-database /usr/share/applications/ 2>/dev/null || true

%changelog
* $(date +'%a %b %d %Y') Developer Name <dev@example.com> - ${VERSION}-${RELEASE}
- Initial package release
EOF

echo "RPM spec created at ~/rpmbuild/SPECS/${APP_NAME}.spec"
echo "Run: rpmbuild -ba ~/rpmbuild/SPECS/${APP_NAME}.spec"
```

---

## 48.6 macOS App Bundles

### สร้าง macOS .app Bundle

```bash
#!/bin/bash
# create_macos_app.sh

APP_NAME="MyApplication"
APP_VERSION="2.0.1"
BUNDLE_ID="com.mycompany.myapp"
EXEC_NAME="myapp"
OUTPUT_DIR="./dist/macos"

BUNDLE_DIR="${OUTPUT_DIR}/${APP_NAME}.app"

# สร้าง directory structure
mkdir -p "${BUNDLE_DIR}/Contents/MacOS"
mkdir -p "${BUNDLE_DIR}/Contents/Resources"
mkdir -p "${BUNDLE_DIR}/Contents/Frameworks"

# Copy executable
cp "dist/macos-x64/${EXEC_NAME}" "${BUNDLE_DIR}/Contents/MacOS/${EXEC_NAME}"
chmod 755 "${BUNDLE_DIR}/Contents/MacOS/${EXEC_NAME}"

# Copy resources
cp -r "resources/" "${BUNDLE_DIR}/Contents/Resources/"
cp "resources/app.icns" "${BUNDLE_DIR}/Contents/Resources/${APP_NAME}.icns" 2>/dev/null || true

# สร้าง Info.plist
cat > "${BUNDLE_DIR}/Contents/Info.plist" << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CFBundleDevelopmentRegion</key>
  <string>en</string>
  
  <key>CFBundleDisplayName</key>
  <string>${APP_NAME}</string>
  
  <key>CFBundleExecutable</key>
  <string>${EXEC_NAME}</string>
  
  <key>CFBundleIconFile</key>
  <string>${APP_NAME}</string>
  
  <key>CFBundleIdentifier</key>
  <string>${BUNDLE_ID}</string>
  
  <key>CFBundleInfoDictionaryVersion</key>
  <string>6.0</string>
  
  <key>CFBundleName</key>
  <string>${APP_NAME}</string>
  
  <key>CFBundlePackageType</key>
  <string>APPL</string>
  
  <key>CFBundleShortVersionString</key>
  <string>${APP_VERSION}</string>
  
  <key>CFBundleVersion</key>
  <string>${APP_VERSION}.0</string>
  
  <key>LSMinimumSystemVersion</key>
  <string>10.14</string>
  
  <key>NSHighResolutionCapable</key>
  <true/>
  
  <key>NSHumanReadableCopyright</key>
  <string>Copyright © 2024 MyCompany Ltd. All rights reserved.</string>
  
  <!-- File associations -->
  <key>CFBundleDocumentTypes</key>
  <array>
    <dict>
      <key>CFBundleTypeName</key>
      <string>MyApp Document</string>
      <key>CFBundleTypeRole</key>
      <string>Editor</string>
      <key>CFBundleTypeExtensions</key>
      <array>
        <string>myapp</string>
      </array>
      <key>CFBundleTypeIconFile</key>
      <string>document</string>
    </dict>
  </array>
</dict>
</plist>
EOF

# สร้าง PkgInfo
echo "APPL????" > "${BUNDLE_DIR}/Contents/PkgInfo"

echo "App bundle created at: ${BUNDLE_DIR}"

# สร้าง DMG (ต้องรันบน macOS)
if [ "$(uname)" = "Darwin" ]; then
  echo "Creating DMG..."
  
  DMG_TEMP="${OUTPUT_DIR}/tmp_dmg"
  mkdir -p "${DMG_TEMP}"
  cp -r "${BUNDLE_DIR}" "${DMG_TEMP}/"
  ln -s /Applications "${DMG_TEMP}/Applications"
  
  hdiutil create \
    -volname "${APP_NAME}" \
    -srcfolder "${DMG_TEMP}" \
    -ov \
    -format UDZO \
    "${OUTPUT_DIR}/${APP_NAME}_${APP_VERSION}.dmg"
  
  rm -rf "${DMG_TEMP}"
  echo "DMG created: ${OUTPUT_DIR}/${APP_NAME}_${APP_VERSION}.dmg"
fi
```

---

## 48.7 Windows Portable Apps

### สร้าง Portable Windows Package

```pascal
{$mode objfpc}{$H+}

// portable_check.pas - ตรวจสอบว่ารันแบบ portable หรือไม่
unit PortableConfig;

interface

uses
  SysUtils, IniFiles;

type
  TPortableMode = (pmInstalled, pmPortable, pmAutoDetect);

  TAppConfig = class
  private
    FPortableMode: TPortableMode;
    FConfigPath: String;
    FDataPath: String;
    function DetectPortableMode: Boolean;
    function GetAppDir: String;
  public
    constructor Create(Mode: TPortableMode = pmAutoDetect);
    procedure Initialize;
    property ConfigPath: String read FConfigPath;
    property DataPath: String read FDataPath;
    property IsPortable: Boolean read (FPortableMode = pmPortable);
  end;

implementation

constructor TAppConfig.Create(Mode: TPortableMode = pmAutoDetect);
begin
  inherited Create;
  FPortableMode := Mode;
end;

function TAppConfig.GetAppDir: String;
begin
  Result := ExtractFilePath(ParamStr(0));
end;

function TAppConfig.DetectPortableMode: Boolean;
begin
  // Portable mode ถ้ามีไฟล์ portable.txt ใน app directory
  Result := FileExists(GetAppDir + 'portable.txt') or
            FileExists(GetAppDir + '.portable');
end;

procedure TAppConfig.Initialize;
var
  IsPortable: Boolean;
begin
  case FPortableMode of
    pmInstalled:
      IsPortable := False;
    pmPortable:
      IsPortable := True;
    pmAutoDetect:
      IsPortable := DetectPortableMode;
  end;
  
  if IsPortable then
  begin
    // Portable: เก็บทุกอย่างใน app directory
    FPortableMode := pmPortable;
    FConfigPath := GetAppDir + 'config' + PathDelim;
    FDataPath := GetAppDir + 'data' + PathDelim;
  end
  else
  begin
    // Installed: เก็บใน %APPDATA%
    FPortableMode := pmInstalled;
    FConfigPath := GetEnvironmentVariable('APPDATA') + PathDelim + 
                   'MyApp' + PathDelim + 'config' + PathDelim;
    FDataPath := GetEnvironmentVariable('APPDATA') + PathDelim +
                 'MyApp' + PathDelim + 'data' + PathDelim;
  end;
  
  // สร้าง directories
  ForceDirectories(FConfigPath);
  ForceDirectories(FDataPath);
end;

end.
```

### Batch Script สำหรับ Portable Package

```batch
@echo off
:: create_portable_package.bat

set APP_NAME=MyApplication
set VERSION=2.0.1
set OUTPUT_DIR=release\portable

echo Creating portable package for %APP_NAME% v%VERSION%...

:: สร้าง directory
mkdir "%OUTPUT_DIR%\%APP_NAME%_v%VERSION%_Portable" 2>nul

set DEST=%OUTPUT_DIR%\%APP_NAME%_v%VERSION%_Portable

:: Copy executable
copy "dist\windows-x64\MyApp.exe" "%DEST%\MyApp.exe"
copy "dist\windows-x64\*.dll" "%DEST%\"

:: Copy resources
xcopy "resources" "%DEST%\resources\" /E /I /Q
xcopy "locales" "%DEST%\locales\" /E /I /Q
xcopy "docs" "%DEST%\docs\" /E /I /Q

:: สร้าง portable marker
echo This is a portable installation > "%DEST%\portable.txt"

:: สร้าง empty directories
mkdir "%DEST%\config"
mkdir "%DEST%\data"
mkdir "%DEST%\logs"

:: สร้าง README
(
echo MyApplication v%VERSION% - Portable Edition
echo =============================================
echo.
echo This is the portable version of MyApplication.
echo All settings and data are stored in this folder.
echo.
echo To run: Double-click MyApp.exe
echo.
echo To move: Copy this entire folder to a new location.
echo All settings will be preserved.
) > "%DEST%\README.txt"

:: สร้าง ZIP ถ้ามี 7zip
where 7z >nul 2>&1
if %ERRORLEVEL% equ 0 (
  echo Creating ZIP archive...
  7z a "%OUTPUT_DIR%\%APP_NAME%_v%VERSION%_Portable.zip" "%DEST%\*"
  echo ZIP created: %OUTPUT_DIR%\%APP_NAME%_v%VERSION%_Portable.zip
) else (
  echo Note: 7-Zip not found. ZIP not created.
  echo Package folder: %DEST%
)

echo Done!
```

---

## 48.8 Version Info

### การฝัง Version Information

```pascal
{$mode objfpc}{$H+}

// version_info.pas
unit VersionInfo;

interface

const
  APP_VERSION_MAJOR = 2;
  APP_VERSION_MINOR = 0;
  APP_VERSION_PATCH = 1;
  APP_VERSION_BUILD = 1234;
  
  APP_VERSION_STR = '2.0.1';
  APP_BUILD_DATE = {$I build_date.inc};  // ไฟล์ที่ generate จาก build script
  APP_GIT_HASH = {$I git_hash.inc};      // ไฟล์ที่ generate จาก build script

function GetVersionString: String;
function GetFullVersionString: String;
function GetBuildInfo: String;

implementation

uses
  SysUtils;

function GetVersionString: String;
begin
  Result := Format('%d.%d.%d', [
    APP_VERSION_MAJOR, APP_VERSION_MINOR, APP_VERSION_PATCH
  ]);
end;

function GetFullVersionString: String;
begin
  Result := Format('%d.%d.%d.%d', [
    APP_VERSION_MAJOR, APP_VERSION_MINOR, APP_VERSION_PATCH, APP_VERSION_BUILD
  ]);
end;

function GetBuildInfo: String;
begin
  Result := Format('Version %s (Build %d, %s)', [
    APP_VERSION_STR, APP_VERSION_BUILD, APP_BUILD_DATE
  ]);
end;

end.
```

### Windows Version Resource (RC File)

```rc
// version.rc - Windows Version Resource

#define APP_VERSION_MAJOR 2
#define APP_VERSION_MINOR 0
#define APP_VERSION_PATCH 1
#define APP_VERSION_BUILD 1234

1 VERSIONINFO
  FILEVERSION     APP_VERSION_MAJOR,APP_VERSION_MINOR,APP_VERSION_PATCH,APP_VERSION_BUILD
  PRODUCTVERSION  APP_VERSION_MAJOR,APP_VERSION_MINOR,APP_VERSION_PATCH,APP_VERSION_BUILD
  FILEFLAGSMASK   0x3fL
  FILEFLAGS       0x0L
  FILEOS          0x40004L
  FILETYPE        0x1L
  FILESUBTYPE     0x0L
BEGIN
  BLOCK "StringFileInfo"
  BEGIN
    BLOCK "040904b0"
    BEGIN
      VALUE "CompanyName",      "MyCompany Ltd.\0"
      VALUE "FileDescription",  "MyApplication\0"
      VALUE "FileVersion",      "2.0.1.1234\0"
      VALUE "InternalName",     "MyApp\0"
      VALUE "LegalCopyright",   "Copyright © 2024 MyCompany Ltd.\0"
      VALUE "OriginalFilename", "MyApp.exe\0"
      VALUE "ProductName",      "MyApplication\0"
      VALUE "ProductVersion",   "2.0.1\0"
    END
  END
  BLOCK "VarFileInfo"
  BEGIN
    VALUE "Translation", 0x0409, 1200
  END
END
```

### Lazarus Version Info (ตั้งค่าใน IDE)

```
Project > Project Options > Version Info:
  Enable: checked
  
  Major version: 2
  Minor version: 0
  Revision: 1
  Build: 1234
  
  File description: MyApplication
  Product name: MyApplication
  Product version: 2.0.1
  File version: 2.0.1.1234
  Comments: Built with Lazarus/Free Pascal
  
  Company name: MyCompany Ltd.
  Legal copyright: Copyright © 2024 MyCompany Ltd.
  Internal name: MyApp
  Original filename: MyApp.exe
```

---

## 48.9 Icon Embedding

### การฝัง Icons

```pascal
// สร้าง RC file สำหรับ icon
// app_resources.rc

AppIcon ICON "resources/app.ico"
SplashBitmap BITMAP "resources/splash.bmp"
```

```pascal
{$mode objfpc}{$H+}

// icon_helper.pas
unit IconHelper;

interface

uses
  Classes, Graphics;

// โหลด icon จาก resources (Windows)
function LoadAppIcon: TIcon;

// โหลด icon จาก file
function LoadIconFromFile(const FileName: String; Size: Integer = 32): TIcon;

implementation

uses
  SysUtils;

{$IFDEF WINDOWS}
uses
  Windows, CommCtrl;
{$ENDIF}

function LoadAppIcon: TIcon;
begin
  Result := TIcon.Create;
  try
    {$IFDEF WINDOWS}
    // โหลดจาก embedded resource
    Result.Handle := LoadIcon(HInstance, 'APPICON');
    if Result.Handle = 0 then
      Result.Handle := LoadIcon(HInstance, MakeIntResource(1));
    {$ELSE}
    // โหลดจากไฟล์
    if FileExists('/usr/share/pixmaps/myapp.png') then
      Result.LoadFromFile('/usr/share/pixmaps/myapp.png')
    else if FileExists(ExtractFilePath(ParamStr(0)) + 'resources/app.png') then
      Result.LoadFromFile(ExtractFilePath(ParamStr(0)) + 'resources/app.png');
    {$ENDIF}
  except
    FreeAndNil(Result);
  end;
end;

function LoadIconFromFile(const FileName: String; Size: Integer = 32): TIcon;
begin
  Result := TIcon.Create;
  try
    if FileExists(FileName) then
      Result.LoadFromFile(FileName);
  except
    FreeAndNil(Result);
  end;
end;

end.
```

---

## 48.10 Manifest Files (Windows)

### Windows Application Manifest

```xml
<!-- myapp.manifest -->
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
  
  <assemblyIdentity
    version="2.0.1.0"
    processorArchitecture="amd64"
    name="MyCompany.MyApp"
    type="win32" />
  
  <description>MyApplication</description>
  
  <!-- Require Windows 10+ visuals -->
  <dependency>
    <dependentAssembly>
      <assemblyIdentity
        type="win32"
        name="Microsoft.Windows.Common-Controls"
        version="6.0.0.0"
        processorArchitecture="*"
        publicKeyToken="6595b64144ccf1df"
        language="*" />
    </dependentAssembly>
  </dependency>
  
  <!-- UAC: requestedExecutionLevel -->
  <trustInfo xmlns="urn:schemas-microsoft-com:asm.v2">
    <security>
      <requestedPrivileges>
        <requestedExecutionLevel
          level="asInvoker"
          uiAccess="false" />
      </requestedPrivileges>
    </security>
  </trustInfo>
  
  <!-- DPI Awareness -->
  <application xmlns="urn:schemas-microsoft-com:asm.v3">
    <windowsSettings>
      <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">
        true/PM
      </dpiAware>
      <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">
        PerMonitorV2
      </dpiAwareness>
    </windowsSettings>
  </application>
  
  <!-- Windows compatibility -->
  <compatibility xmlns="urn:schemas-microsoft-com:compatibility.v1">
    <application>
      <!-- Windows 10 -->
      <supportedOS Id="{8e0f7a12-bfb3-4fe8-b9a5-48fd50a15a9a}" />
      <!-- Windows 11 -->
      <supportedOS Id="{8e0f7a12-bfb3-4fe8-b9a5-48fd50a15a9a}" />
    </application>
  </compatibility>
  
</assembly>
```

---

## 48.11 Build Script Example

### Complete Build Script

```bash
#!/bin/bash
# build_all.sh - Comprehensive build script

set -e

# ============================================================
# Configuration
# ============================================================
APP_NAME="MyApplication"
APP_VERSION=$(cat VERSION 2>/dev/null || echo "2.0.1")
BUILD_DIR="./build"
DIST_DIR="./dist"
PROJECT_FILE="./src/myapp.lpi"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# ============================================================
# Functions
# ============================================================
log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }

check_dependencies() {
  log_info "Checking dependencies..."
  
  if ! command -v lazbuild &> /dev/null; then
    log_error "lazbuild not found. Please install Lazarus."
    exit 1
  fi
  
  if ! command -v fpc &> /dev/null; then
    log_error "FPC not found. Please install Free Pascal Compiler."
    exit 1
  fi
  
  log_info "Dependencies OK"
}

generate_version_files() {
  log_info "Generating version files..."
  
  # build_date.inc
  echo "'$(date +%Y-%m-%d)'" > src/build_date.inc
  
  # git_hash.inc
  if command -v git &> /dev/null && git rev-parse --git-dir > /dev/null 2>&1; then
    echo "'$(git rev-parse --short HEAD)'" > src/git_hash.inc
  else
    echo "'unknown'" > src/git_hash.inc
  fi
  
  log_info "Version files generated"
}

run_tests() {
  log_info "Running tests..."
  
  if [ -f "./tests/run_tests.sh" ]; then
    bash ./tests/run_tests.sh
  else
    log_warn "No test script found, skipping tests"
  fi
}

build_platform() {
  local PLATFORM=$1
  local OS=$2
  local CPU=$3
  local BUILD_MODE=$4
  local OUTPUT_EXT=${5:-""}
  
  log_info "Building for ${PLATFORM}..."
  
  local OUTPUT_DIR="${DIST_DIR}/${PLATFORM}"
  mkdir -p "${OUTPUT_DIR}"
  
  lazbuild \
    --build-mode="${BUILD_MODE}" \
    --os="${OS}" \
    --cpu="${CPU}" \
    "${PROJECT_FILE}" 2>&1
  
  if [ $? -eq 0 ]; then
    log_info "Build successful for ${PLATFORM}"
    
    # Strip binaries
    if [ -z "${OUTPUT_EXT}" ] && command -v strip &> /dev/null; then
      strip "${OUTPUT_DIR}/myapp${OUTPUT_EXT}" 2>/dev/null || true
    fi
    
    # Show size
    log_info "Binary size: $(du -sh "${OUTPUT_DIR}/myapp${OUTPUT_EXT}" 2>/dev/null | cut -f1)"
  else
    log_error "Build FAILED for ${PLATFORM}"
    return 1
  fi
}

create_packages() {
  log_info "Creating distribution packages..."
  
  # Linux DEB
  if command -v dpkg-deb &> /dev/null; then
    log_info "Creating DEB package..."
    bash ./packaging/create_deb.sh
  fi
  
  # Linux RPM
  if command -v rpmbuild &> /dev/null; then
    log_info "Creating RPM package..."
    bash ./packaging/create_rpm.sh
  fi
  
  # Windows Installer (ถ้ามี ISCC)
  if command -v iscc &> /dev/null; then
    log_info "Creating Windows installer..."
    iscc /Q "./packaging/installer.iss"
  elif [ "$(uname)" = "Linux" ] && command -v wine &> /dev/null; then
    log_info "Creating Windows installer via Wine..."
    wine "C:/Program Files (x86)/Inno Setup 6/ISCC.exe" /Q "./packaging/installer.iss"
  fi
  
  log_info "Package creation complete"
}

generate_checksums() {
  log_info "Generating checksums..."
  
  cd "${DIST_DIR}"
  
  # SHA256 checksums
  find . -type f -name "*.exe" -o -name "*.deb" -o -name "*.rpm" -o \
    -name "*.zip" -o -name "*.dmg" | \
    xargs sha256sum > checksums.sha256 2>/dev/null || true
  
  cd - > /dev/null
  log_info "Checksums saved to ${DIST_DIR}/checksums.sha256"
}

# ============================================================
# Main
# ============================================================
echo "========================================="
echo "  Building ${APP_NAME} v${APP_VERSION}"
echo "========================================="
echo ""

# Parse arguments
BUILD_TARGET=${1:-"all"}
SKIP_TESTS=${SKIP_TESTS:-"false"}

# Setup
check_dependencies
generate_version_files
mkdir -p "${BUILD_DIR}" "${DIST_DIR}"

# Tests
if [ "${SKIP_TESTS}" != "true" ]; then
  run_tests
fi

# Build
case "${BUILD_TARGET}" in
  "linux")
    build_platform "linux-x64" "linux" "x86_64" "Release_Linux64"
    ;;
  "windows")
    build_platform "windows-x64" "win64" "x86_64" "Release_Win64" ".exe"
    ;;
  "macos")
    build_platform "macos-x64" "darwin" "x86_64" "Release_macOS64"
    ;;
  "all")
    build_platform "linux-x64" "linux" "x86_64" "Release_Linux64"
    build_platform "windows-x64" "win64" "x86_64" "Release_Win64" ".exe"
    ;;
  *)
    log_error "Unknown target: ${BUILD_TARGET}"
    echo "Usage: $0 [linux|windows|macos|all]"
    exit 1
    ;;
esac

# Packages
create_packages

# Checksums
generate_checksums

echo ""
echo "========================================="
log_info "Build complete!"
echo "Output: ${DIST_DIR}/"
ls -la "${DIST_DIR}/"
echo "========================================="
```

---

## แบบฝึกหัด 5 ข้อ

### ข้อ 1: Build Configurations
ตั้งค่า Lazarus project ให้มี build configurations:
- Debug: มี debug symbols, all checks enabled
- Release: O3 optimization, stripped, no debug
- เพิ่ม version info ในทั้งสอง configurations

### ข้อ 2: Cross-Platform Build
สร้าง build script ที่:
- Compile สำหรับ Linux และ Windows
- Strip binaries อัตโนมัติ
- แสดง file size ก่อนและหลัง strip
- Generate checksums

### ข้อ 3: Linux DEB Package
สร้าง DEB package สำหรับ Hello World app:
- ติดตั้งใน /usr/bin/
- สร้าง .desktop file
- postinst script update icon cache
- prerm และ postrm scripts

### ข้อ 4: Portable Application
สร้าง app ที่:
- ตรวจจับ portable mode จาก portable.txt
- เก็บ config ใน app directory ถ้า portable
- เก็บ config ใน %APPDATA% ถ้า installed
- สร้าง portable package script

### ข้อ 5: Version Information System
สร้างระบบ version ที่:
- อ่าน version จาก VERSION file
- Embed git commit hash ณ เวลา build
- แสดง version ใน About dialog
- Windows: ฝัง version resource ใน executable
