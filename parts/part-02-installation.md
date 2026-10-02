# Part 02 - การติดตั้งและตั้งค่า Lazarus IDE

## สารบัญ

1. [ความต้องการของระบบ](#ความต้องการของระบบ)
2. [ดาวน์โหลด Lazarus](#ดาวน์โหลด-lazarus)
3. [ติดตั้งบน Windows](#ติดตั้งบน-windows)
4. [ติดตั้งบน Linux](#ติดตั้งบน-linux)
5. [ติดตั้งบน macOS](#ติดตั้งบน-macos)
6. [การตั้งค่า IDE](#การตั้งค่า-ide)
7. [ปลั๊กอินที่แนะนำ](#ปลั๊กอินที่แนะนำ)
8. [การสร้างโปรเจคแรก](#การสร้างโปรเจคแรก)
9. [โครงสร้างของโปรเจค Lazarus](#โครงสร้างของโปรเจค-lazarus)
10. [การใช้งาน Debugger](#การใช้งาน-debugger)
11. [Keyboard Shortcuts ที่สำคัญ](#keyboard-shortcuts-ที่สำคัญ)
12. [แบบฝึกหัด 10 ข้อ](#แบบฝึกหัด-10-ข้อ)

---

## ความต้องการของระบบ

### Windows
```
ระบบปฏิบัติการ: Windows 7, 8, 10, 11 (32-bit หรือ 64-bit)
RAM:            ขั้นต่ำ 512 MB (แนะนำ 2 GB+)
พื้นที่ฮาร์ดดิสก์: ขั้นต่ำ 500 MB (แนะนำ 2 GB+)
CPU:            x86 หรือ x86-64
```

### Linux
```
ระบบปฏิบัติการ: ส่วนใหญ่ของ Linux distributions
RAM:            ขั้นต่ำ 256 MB (แนะนำ 1 GB+)
พื้นที่ฮาร์ดดิสก์: ขั้นต่ำ 300 MB
Package manager: apt, yum, pacman, ฯลฯ
```

### macOS
```
ระบบปฏิบัติการ: macOS 10.12 Sierra หรือใหม่กว่า
RAM:            ขั้นต่ำ 512 MB (แนะนำ 2 GB+)
พื้นที่ฮาร์ดดิสก์: ขั้นต่ำ 500 MB
CPU:            Intel x86-64 หรือ Apple Silicon (ARM64)
```

---

## ดาวน์โหลด Lazarus

### เว็บไซต์ทางการ

เว็บไซต์หลัก: **https://www.lazarus-ide.org/**

เว็บไซต์ sourceforge: **https://sourceforge.net/projects/lazarus/**

### เวอร์ชันที่แนะนำ

| เวอร์ชัน | FPC Version | สถานะ | แนะนำสำหรับ |
|---------|-------------|--------|-------------|
| Lazarus 3.x | FPC 3.2.x | Stable | ผู้เริ่มต้น (แนะนำ) |
| Lazarus 2.2.x | FPC 3.2.x | Stable | ใช้งานทั่วไป |
| Lazarus Nightly | FPC trunk | Unstable | นักพัฒนา/ทดสอบ |

### ไฟล์ที่ต้องดาวน์โหลด

```
สำหรับ Windows 64-bit:
  lazarus-x.x.x-fpc-x.x.x-win64.exe

สำหรับ Windows 32-bit:
  lazarus-x.x.x-fpc-x.x.x-win32.exe

สำหรับ Linux (Ubuntu/Debian .deb):
  fpc_x.x.x-x_amd64.deb
  lazarus_x.x.x-x_amd64.deb

สำหรับ macOS (dmg):
  lazarus-x.x.x-fpc-x.x.x-macosx-x86_64.dmg
```

---

## ติดตั้งบน Windows

### ขั้นตอนการติดตั้ง

**ขั้นตอนที่ 1: เตรียมไฟล์ติดตั้ง**

ดาวน์โหลดไฟล์ `.exe` จาก sourceforge.net:

```
เลือกไฟล์ตามระบบ:
  Windows 64-bit → lazarus-3.x-fpc-3.2.x-win64.exe (แนะนำ)
  Windows 32-bit → lazarus-3.x-fpc-3.2.x-win32.exe
```

**ขั้นตอนที่ 2: รันโปรแกรมติดตั้ง**

```
[คลิกขวา] lazarus-3.x.exe
[เลือก] Run as administrator
```

หน้าจอ Welcome Screen:

```
+--------------------------------------------------+
|  Lazarus IDE Setup                               |
|  ================================================|
|                                                  |
|  Welcome to the Lazarus IDE Setup Wizard         |
|                                                  |
|  This will install Lazarus 3.x on your          |
|  computer.                                       |
|                                                  |
|  Click Next to continue                          |
|                                                  |
|              [Next >]  [Cancel]                  |
+--------------------------------------------------+
```

**ขั้นตอนที่ 3: License Agreement**

```
+--------------------------------------------------+
|  License Agreement                               |
|  ================================================|
|  GNU GENERAL PUBLIC LICENSE                      |
|  Version 2, June 1991                            |
|                                                  |
|  [ข้อความ license...]                             |
|                                                  |
|  (•) I accept the agreement                      |
|  ( ) I do not accept the agreement               |
|                                                  |
|          [< Back]  [Next >]  [Cancel]            |
+--------------------------------------------------+
```

เลือก **"I accept the agreement"** แล้วกด **Next**

**ขั้นตอนที่ 4: เลือกโฟลเดอร์ติดตั้ง**

```
+--------------------------------------------------+
|  Select Destination Location                     |
|  ================================================|
|                                                  |
|  Where should Lazarus be installed?              |
|                                                  |
|  Destination folder:                             |
|  [C:\lazarus                            ] [Browse]|
|                                                  |
|  แนะนำ: C:\lazarus (อย่าใช้โฟลเดอร์ที่มีช่องว่าง!)  |
|                                                  |
|          [< Back]  [Next >]  [Cancel]            |
+--------------------------------------------------+
```

> **คำเตือน:** อย่าติดตั้งใน `C:\Program Files\` เพราะ path มีช่องว่างอาจทำให้มีปัญหา แนะนำ `C:\lazarus`

**ขั้นตอนที่ 5: เลือก Components**

```
+--------------------------------------------------+
|  Select Components                               |
|  ================================================|
|                                                  |
|  [✓] Lazarus IDE
|  [✓] Free Pascal Compiler (FPC)
|  [✓] FPC Sources
|  [✓] Lazarus Sources
|  [✓] Help files
|  [ ] Associate .pas files (แนะนำให้ติ๊ก)
|  [✓] Create Desktop Shortcut
|  [✓] Create Start Menu Shortcut
|                                                  |
|  พื้นที่ที่ต้องการ: ~500 MB                        |
|                                                  |
|          [< Back]  [Install]  [Cancel]           |
+--------------------------------------------------+
```

**ขั้นตอนที่ 6: ติดตั้ง**

```
+--------------------------------------------------+
|  Installing                                      |
|  ================================================|
|                                                  |
|  Extracting files...                             |
|  [██████████████████████      ] 75%             |
|                                                  |
|  Installing FPC...                               |
|  Installing Lazarus IDE...                       |
|  Creating shortcuts...                           |
|                                                  |
+--------------------------------------------------+
```

รอจนเสร็จ (ใช้เวลาประมาณ 3-10 นาที)

**ขั้นตอนที่ 7: เสร็จสิ้น**

```
+--------------------------------------------------+
|  Completing Setup                                |
|  ================================================|
|                                                  |
|  Lazarus IDE has been installed successfully!    |
|                                                  |
|  [✓] Launch Lazarus IDE                          |
|                                                  |
|                          [Finish]               |
+--------------------------------------------------+
```

กด **Finish** เพื่อเปิด Lazarus

**ขั้นตอนที่ 8: การรันครั้งแรก**

เมื่อ Lazarus เปิดครั้งแรก จะถามว่าต้องการ rebuild IDE หรือไม่:

```
+--------------------------------------------------+
|  Lazarus IDE                                     |
|                                                  |
|  The IDE was built without some packages.        |
|  Rebuild the IDE?                                |
|                                                  |
|      [Yes]  [No]  [No (don't ask again)]        |
+--------------------------------------------------+
```

กด **Yes** เพื่อ rebuild IDE (ใช้เวลาสักครู่)

---

## ติดตั้งบน Linux

### Ubuntu / Debian (apt)

**วิธีที่ 1: ใช้ Package Manager (แนะนำ)**

```bash
# อัพเดท package list
sudo apt update

# ติดตั้ง Free Pascal
sudo apt install fpc

# ติดตั้ง Lazarus
sudo apt install lazarus

# ตรวจสอบการติดตั้ง
fpc -v
lazarus --version
```

**วิธีที่ 2: ติดตั้งจาก .deb ไฟล์**

ดาวน์โหลดไฟล์ .deb จาก sourceforge แล้ว:

```bash
# ติดตั้ง dependencies ก่อน
sudo apt install libgtk2.0-dev

# ติดตั้ง FPC ก่อน
sudo dpkg -i fpc_3.2.x-x_amd64.deb

# ติดตั้ง Lazarus
sudo dpkg -i lazarus_3.x.x-x_amd64.deb

# แก้ปัญหา dependencies (ถ้ามี)
sudo apt-get install -f
```

### Fedora / Red Hat / CentOS (dnf/yum)

```bash
# Fedora
sudo dnf install fpc lazarus

# CentOS/RHEL (ต้องเปิด EPEL ก่อน)
sudo yum install epel-release
sudo yum install fpc lazarus
```

### Arch Linux (pacman)

```bash
# ติดตั้งจาก AUR
yay -S lazarus-ide
# หรือ
pamac install lazarus
```

### openSUSE (zypper)

```bash
sudo zypper install fpc lazarus
```

### การรัน Lazarus บน Linux

```bash
# รันจาก command line
lazarus

# หรือหา Lazarus ใน Application Menu
# หมวด: Development หรือ Programming
```

### การตั้งค่า PATH สำหรับ FPC

เพิ่มใน `~/.bashrc` หรือ `~/.zshrc`:

```bash
# เพิ่ม FPC และ Lazarus ใน PATH
export PATH=$PATH:/usr/lib/fpc/3.2.x/
export PATH=$PATH:/usr/share/lazarus/

# โหลด config ใหม่
source ~/.bashrc
```

---

## ติดตั้งบน macOS

### ขั้นตอนการติดตั้ง (Intel)

**ขั้นตอนที่ 1:** ดาวน์โหลด `.dmg` file จาก lazarus-ide.org

**ขั้นตอนที่ 2:** เปิดไฟล์ .dmg และลาก Lazarus ไปที่ Applications

```
+------------------------------------------+
|  Lazarus 3.x                             |
|  ========================================|
|                                          |
|  [Lazarus]  →→→→→  [Applications]       |
|                                          |
|  ลากไอคอน Lazarus ไปยัง Applications    |
+------------------------------------------+
```

**ขั้นตอนที่ 3:** การรันครั้งแรก

macOS อาจบล็อคเนื่องจาก Gatekeeper:

```bash
# อนุญาตให้รัน
sudo xattr -rd com.apple.quarantine /Applications/Lazarus/

# หรือผ่าน System Preferences:
# System Preferences → Security & Privacy → Allow Anyway
```

### Apple Silicon (M1/M2/M3)

สำหรับ Mac ที่ใช้ Apple Silicon:

```bash
# ใช้ Homebrew ติดตั้ง
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง FPC
brew install fpc

# ดาวน์โหลด Lazarus DMG แบบ ARM64
# lazarus-x.x.x-fpc-x.x.x-macosx-aarch64.dmg
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ FPC
fpc -v
# ควรแสดง: Free Pascal Compiler version x.x.x...

# ตรวจสอบ Lazarus (command line)
/Applications/Lazarus/lazarus --version
```

---

## การตั้งค่า IDE

### การตั้งค่าครั้งแรก

เมื่อเปิด Lazarus ครั้งแรก ไปที่ **Tools → Options** (Ctrl+Shift+O)

### 1. การตั้งค่า Editor

```
Tools → Options → Editor → General

Font:
  Font: Consolas (Windows) / DejaVu Sans Mono (Linux) / Menlo (macOS)
  Size: 12-14 (แนะนำ 13)

Display:
  [✓] Show line numbers
  [✓] Show gutter
  [✓] Highlight current line
  Tab width: 2 หรือ 4
  [✓] Replace tabs with spaces (แนะนำ)
```

### 2. การตั้งค่า Colors / Theme

```
Tools → Options → Editor → Colors

Light theme:
  Scheme: Default
  ปรับ Background: White
  ปรับ Text: Black

Dark theme (แนะนำ):
  Scheme: Monokai (ถ้ามี) หรือ Dark
  Background: #1E1E1E หรือ #282828
```

**การเปิด Theme สำหรับ IDE ทั้งหมด (Lazarus 2.0+):**
```
Tools → Options → Environment → Desktop
  Skin: HighContrast, Office2007, ฯลฯ
```

### 3. การตั้งค่า FPC Paths

```
Tools → Options → Compiler (FPC) → Directories

FPC Source directory:
  Windows: C:\lazarus\fpc\x.x.x\source\
  Linux:   /usr/share/fpcsrc/x.x.x/
  macOS:   /usr/local/lib/fpc/x.x.x/source/

Make executable:
  Windows: C:\lazarus\fpc\bin\i386-win32\make.exe
  Linux:   /usr/bin/make
```

### 4. การตั้งค่า Code Completion

```
Tools → Options → Editor → Code Tools → Code Completion

[✓] Use Code Completion
[✓] Show hints for IDE functions
[✓] Automatic invocation delay: 500 ms
[✓] Complete unambiguous identifiers
```

### 5. การตั้งค่า Formatter (Code Style)

```
Tools → Options → Editor → CodeFormatting

Indentation:
  Width: 2 (แนะนำ) หรือ 4
  Use Spaces: Yes

Begin-End:
  BeginEnd in SameIndent: checked
```

### 6. ตั้งค่า Build Modes

สร้าง Build Modes สำหรับ Debug และ Release:

```
Project → Project Options → Compiler Options → Build Modes

Debug Mode:
  [✓] Generate debug info (-g)
  [✓] Generate line info (-gl)
  Optimization: None (-O0)

Release Mode:
  [ ] Generate debug info
  Optimization: Level 2 (-O2) หรือ Level 3 (-O3)
  [✓] Strip debug info from executable
```

---

## ปลั๊กอินที่แนะนำ

### การติดตั้ง Packages/Plugins

ใช้เมนู: **Package → Online Package Manager** หรือ **Package → Install/Uninstall Packages**

### ปลั๊กอินที่แนะนำ

#### 1. CoolBar
สำหรับ Toolbar ที่สามารถลากวางได้

```
ชื่อ: CoolBar
ฟังก์ชัน: TCoolBar component สำหรับ toolbar ที่ movable
การติดตั้ง: Package → Install Package → เลือก CoolBarPkg
```

#### 2. JVCL (JEDI VCL for Lazarus)
คอมโพเนนต์จำนวนมาก:

```
ชื่อ: JVCL
URL: https://github.com/project-jedi/jvcl
ฟังก์ชัน:
  - TJvDateEdit (date picker)
  - TJvSpinEdit (spin control)
  - TJvCheckListBox
  - และอีกกว่า 600 components
```

#### 3. Indy (Internet Direct)
สำหรับ Network Programming:

```
ชื่อ: Indy10
ฟังก์ชัน:
  - TIdHTTP (HTTP Client)
  - TIdFTP (FTP)
  - TIdSMTP (Email)
  - TIdTCPClient/Server
  - TIdUDP
```

#### 4. LazReport
สำหรับสร้าง Report:

```
ชื่อ: LazReport
ฟังก์ชัน:
  - Print reports
  - Export to PDF, HTML
  - Visual report designer
```

#### 5. BGRABitmap
สำหรับงานกราฟิก 2D:

```
ชื่อ: BGRABitmap
ฟังก์ชัน:
  - 2D drawing
  - Image processing
  - Animation
  - Gradients, transparency
```

#### 6. SQLDBComponents
สำหรับ Database:

```
ชื่อ: SQLDBComponents (มากับ Lazarus)
ฟังก์ชัน:
  - TSQLite3Connection
  - TMySQLConnection
  - TPostgreSQLConnection
  - TIBConnection (Firebird)
```

### การติดตั้ง Package ด้วยตนเอง

```pascal
// 1. ดาวน์โหลด package source
// 2. แตกไฟล์ไปยังโฟลเดอร์เช่น C:\lazarus\components\mypackage\

// 3. เปิด Package
Package → Open Package File
เลือกไฟล์ .lpk

// 4. Compile Package
กด "Compile" ใน Package window

// 5. Install Package
กด "Install" หรือ
Package → Install/Uninstall Packages
เลือก package แล้วกด "Install Selection"

// 6. Rebuild IDE
กด "Rebuild IDE" เมื่อถูกถามถาม
```

---

## การสร้างโปรเจคแรก

### โปรเจค Console Application

**ขั้นตอนที่ 1: สร้างโปรเจคใหม่**

```
File → New → Project → Program

หรือ

Project → New Project → Console Application
```

**ขั้นตอนที่ 2: บันทึกโปรเจค**

```
File → Save All (Ctrl+Shift+S)

เลือกโฟลเดอร์: C:\projects\MyFirst\
ชื่อ Unit: main.pas
ชื่อ Project: myfirst.lpi
```

**ขั้นตอนที่ 3: เขียนโค้ด**

```pascal
program MyFirstProgram;
uses
  SysUtils;
begin
  WriteLn('สวัสดี! โปรแกรมแรกของฉัน');
  WriteLn('วันที่: ', DateToStr(Date));
  WriteLn('กด Enter เพื่อออก...');
  ReadLn;
end.
```

**ขั้นตอนที่ 4: Compile และ Run**

```
กด F9 หรือ Run → Run
หรือ
กด Ctrl+F9 เพื่อ Compile เฉยๆ (ไม่รัน)
```

### โปรเจค GUI Application

**ขั้นตอนที่ 1: สร้างโปรเจค GUI**

```
File → New → Project → Application
```

**ขั้นตอนที่ 2: Form Designer จะเปิดขึ้นมา**

```
+----------------------------------+
|  Form1 (TForm1)                  |
|  ================================|
|                                  |
|  [พื้นที่ออกแบบ GUI]              |
|                                  |
|                                  |
+----------------------------------+
```

**ขั้นตอนที่ 3: เพิ่ม Component**

จาก Component Palette (ด้านบน) ลากวาง:
1. **TButton** - ปุ่มกด
2. **TLabel** - ป้ายข้อความ
3. **TEdit** - กล่องรับข้อมูล

```
การวาง Component:
1. คลิกที่ TButton ใน Component Palette
2. คลิกบน Form ตรงที่ต้องการวาง
3. ปรับขนาดโดยลากขอบ
```

**ขั้นตอนที่ 4: ตั้งค่า Properties**

Object Inspector (ด้านซ้าย):

```
TButton Properties:
  Name: btnHello
  Caption: คลิกที่นี่!
  Width: 120
  Height: 35

TLabel Properties:
  Name: lblMessage
  Caption: (ว่าง)
  Font.Size: 14
  
TEdit Properties:
  Name: edtName
  Text: (ว่าง)
```

**ขั้นตอนที่ 5: เพิ่ม Event Handler**

ดับเบิลคลิกที่ปุ่ม btnHello จะสร้าง event handler:

```pascal
procedure TForm1.btnHelloClick(Sender: TObject);
begin
  if edtName.Text <> '' then
    lblMessage.Caption := 'สวัสดี, ' + edtName.Text + '!'
  else
    lblMessage.Caption := 'กรุณาใส่ชื่อก่อน!';
end;
```

**ขั้นตอนที่ 6: บันทึกและรัน**

กด **F9** เพื่อรัน

---

## โครงสร้างของโปรเจค Lazarus

### โครงสร้างไฟล์

```
MyProject/
├── MyProject.lpi         ← Project file (XML format)
├── MyProject.lps         ← Project session (บันทึก layout ของ IDE)
├── main.pas              ← Unit หลัก
├── Unit1.pas             ← Unit form
├── Unit1.lfm             ← Form layout file
├── MyProject.res         ← Resources file
├── lib/                  ← Output directory สำหรับ object files
│   └── i386-win32/       ← หรือ x86_64-linux/ ตามแพลตฟอร์ม
│       ├── main.o
│       └── Unit1.o
└── MyProject.exe         ← Compiled executable (Windows)
    (หรือ MyProject บน Linux/macOS)
```

### ไฟล์ .lpi (Project Information)

ไฟล์ XML ที่เก็บการตั้งค่าโปรเจค:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CONFIG>
  <ProjectOptions>
    <Version Value="12"/>
    <PathDelim Value="\"/>
    <General>
      <Flags>
        <MainUnitHasUsesSectionForAllUnits Value="True"/>
      </Flags>
      <SessionStorage Value="InProjectDir"/>
      <Title Value="MyProject"/>
      <UseAppBundle Value="True"/>
    </General>
    <BuildModes Count="2">
      <Item1 Name="Debug" Default="True"/>
      <Item2 Name="Release"/>
    </BuildModes>
    <Units Count="2">
      <Unit0>
        <Filename Value="main.pas"/>
        <IsPartOfProject Value="True"/>
      </Unit0>
      <Unit1>
        <Filename Value="Unit1.pas"/>
        <IsPartOfProject Value="True"/>
      </Unit1>
    </Units>
  </ProjectOptions>
  <CompilerOptions>
    <Target>
      <Filename Value="MyProject"/>
    </Target>
    <SearchPaths>
      <UnitOutputDirectory Value="lib\$(TargetCPU)-$(TargetOS)"/>
    </SearchPaths>
  </CompilerOptions>
</CONFIG>
```

### ไฟล์ .lfm (Form Layout)

ไฟล์ text ที่เก็บ layout ของ Form:

```
object Form1: TForm1
  Left = 372
  Height = 400
  Top = 164
  Width = 600
  Caption = 'My Application'
  ClientHeight = 400
  ClientWidth = 600
  object btnHello: TButton
    Left = 16
    Height = 35
    Top = 16
    Width = 120
    Caption = 'คลิกที่นี่!'
    OnClick = btnHelloClick
    TabOrder = 0
  end
  object lblMessage: TLabel
    Left = 16
    Height = 18
    Top = 70
    Width = 50
    Caption = 'Message'
    Font.Color = clWindowText
    Font.Height = -18
    ParentFont = False
  end
end
```

### โครงสร้าง Unit

```pascal
unit Unit1;

{$mode objfpc}{$H+}

interface
// ส่วน interface - ประกาศสิ่งที่ unit อื่นมองเห็นได้

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs, StdCtrls;

type
  TForm1 = class(TForm)
    // Component declarations (สร้างโดย Form Designer)
    btnHello: TButton;
    lblMessage: TLabel;
    edtName: TEdit;
    // Event handler declarations
    procedure btnHelloClick(Sender: TObject);
  private
    // private declarations
    FMyVar: Integer;
  public
    // public declarations
  end;

var
  Form1: TForm1;  // Global form variable

implementation
// ส่วน implementation - โค้ดจริงของโปรแกรม

{$R *.lfm}  // Load form resource

procedure TForm1.btnHelloClick(Sender: TObject);
begin
  if edtName.Text <> '' then
    lblMessage.Caption := 'สวัสดี, ' + edtName.Text + '!'
  else
    lblMessage.Caption := 'กรุณาใส่ชื่อก่อน!';
end;

end.
```

---

## การใช้งาน Debugger

### การตั้ง Breakpoint

```
วิธีที่ 1: คลิกที่ Gutter (พื้นที่ซ้ายของ line number)
วิธีที่ 2: กด F5 ที่บรรทัดที่ต้องการ
วิธีที่ 3: Run → Toggle Breakpoint
```

Breakpoint จะแสดงเป็นวงกลมสีแดง:

```
  25 |• var
  26 |  x: Integer;
  27 |• begin                    ← Breakpoint ที่บรรทัด 27
  28 |    x := 10;
  29 |    WriteLn(x);
  30 | end.
```

### การ Debug

```
เริ่ม Debug:  F9 (หรือ Run → Run)
Step Over:   F8 (ข้ามไปบรรทัดถัดไป โดยไม่เข้า procedure)
Step Into:   F7 (เข้าไปใน procedure/function)
Step Out:    Shift+F8 (ออกจาก procedure ปัจจุบัน)
Continue:    F9 (ทำงานต่อจนถึง breakpoint ถัดไป)
Stop:        Ctrl+F2 (หยุด debug)
```

### การดู Variable Values

**วิธีที่ 1: Hover Mouse**
เอา mouse ไปวางบนชื่อตัวแปร จะแสดง tooltip ค่าปัจจุบัน

**วิธีที่ 2: Watch Windows**

```
View → Debug Windows → Watches (Ctrl+W)
```

เพิ่ม expression ที่ต้องการดู:

```
+------------------------------------------+
|  Watch List                              |
|  ========================================|
|  Expression          | Value             |
|  ----------------------------------------|
|  x                   | 10                |
|  y                   | 20                |
|  x + y               | 30                |
|  name                | 'Pascal'          |
+------------------------------------------+
```

**วิธีที่ 3: Evaluate/Modify**

```
Run → Evaluate/Modify (Ctrl+F7)
```

```
+------------------------------------------+
|  Evaluate and Modify                     |
|  ========================================|
|  Expression: [x + y              ]       |
|  Result:     30                          |
|  New Value:  [                   ]       |
|                                          |
|              [Evaluate]  [Modify]        |
+------------------------------------------+
```

### Locals Window

ดูตัวแปร local ทั้งหมดในขณะนั้น:

```
View → Debug Windows → Local Variables
```

### Call Stack

ดู function call history:

```
View → Debug Windows → Call Stack
```

```
+------------------------------------------+
|  Call Stack                              |
|  ========================================|
|  #0: TForm1.btnCalculate (line 45)       |
|  #1: TControl.Click (line 892)           |
|  #2: TButtonControl.Click (line 234)     |
|  #3: Application.HandleMessage (line 12) |
+------------------------------------------+
```

### การใช้ Conditional Breakpoint

```
คลิกขวาที่ breakpoint (วงกลมแดง) → Breakpoint Properties

Condition: [x > 5              ]  ← หยุดเฉพาะเมื่อ x > 5
Pass count: [5                  ]  ← หยุดหลังจากผ่านมา 5 ครั้ง
```

---

## Keyboard Shortcuts ที่สำคัญ

### การจัดการไฟล์

| Shortcut | การทำงาน |
|----------|---------|
| Ctrl+N | New file |
| Ctrl+O | Open file |
| Ctrl+S | Save file |
| Ctrl+Shift+S | Save all files |
| Ctrl+W | Close file |
| Alt+F4 | Exit IDE |

### การแก้ไขโค้ด

| Shortcut | การทำงาน |
|----------|---------|
| Ctrl+C | Copy |
| Ctrl+X | Cut |
| Ctrl+V | Paste |
| Ctrl+Z | Undo |
| Ctrl+Y | Redo |
| Ctrl+A | Select all |
| Ctrl+D | Duplicate line |
| Ctrl+Delete | Delete line |
| Tab | Indent |
| Shift+Tab | Unindent |
| Ctrl+/ | Toggle comment |
| Ctrl+Shift+/ | Block comment |

### การนำทางในโค้ด

| Shortcut | การทำงาน |
|----------|---------|
| Ctrl+G | Go to line number |
| Ctrl+F | Find |
| Ctrl+H | Find and Replace |
| Ctrl+L | Find next |
| F3 | Find next |
| Shift+F3 | Find previous |
| Ctrl+Home | ไปบรรทัดแรก |
| Ctrl+End | ไปบรรทัดสุดท้าย |
| Ctrl+Right | ข้ามคำ |
| Ctrl+Left | ย้อนคำ |
| Alt+Left | Go back |
| Alt+Right | Go forward |

### Code Intelligence

| Shortcut | การทำงาน |
|----------|---------|
| Ctrl+Space | Code completion |
| Ctrl+Shift+Space | Parameter hints |
| Ctrl+Click | Go to definition |
| Alt+Up | Jump to header |
| Ctrl+Shift+Up | Jump to implementation |
| F12 | Toggle unit/form |
| Ctrl+I | Identifier completion |
| Shift+Ctrl+I | Smart identifier completion |

### การ Compile และ Run

| Shortcut | การทำงาน |
|----------|---------|
| F9 | Run (Compile + Run) |
| Ctrl+F9 | Compile only |
| Shift+F9 | Build all |
| Ctrl+Shift+F9 | Clean build |
| Ctrl+F2 | Stop program |

### Debugging

| Shortcut | การทำงาน |
|----------|---------|
| F5 | Toggle breakpoint |
| Ctrl+F5 | Delete all breakpoints |
| F7 | Step into |
| F8 | Step over |
| Shift+F8 | Step out |
| Ctrl+F7 | Evaluate/Modify |
| Ctrl+W | Add watch |

### การจัดการ Window

| Shortcut | การทำงาน |
|----------|---------|
| F12 | Toggle Source/Form |
| Shift+F12 | View form list |
| Alt+1 | Editor |
| Alt+2 | Messages |
| Ctrl+Tab | Switch tabs |

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1: ติดตั้งและตั้งค่า
**คำสั่ง:** ติดตั้ง Lazarus และตั้งค่า font เป็น Consolas ขนาด 13 เปิด line numbers และตั้งค่า tab width เป็น 2 แล้ว Screenshot แสดง IDE

### ข้อ 2: โปรเจค Console แรก
**คำสั่ง:** สร้างโปรเจค Console Application ชื่อ "HelloLazarus" เขียนโปรแกรมแสดงข้อความ "สวัสดี Lazarus! ฉันเพิ่งติดตั้งเสร็จ" แล้ว Compile และ Run

### ข้อ 3: โปรเจค GUI แรก
**คำสั่ง:** สร้างโปรเจค GUI Application ที่มี:
- TLabel แสดงข้อความ "ยินดีต้อนรับ"
- TEdit สำหรับรับชื่อ
- TButton เมื่อคลิกให้แสดงชื่อในลาเบล

```pascal
// แนวทางโค้ด
procedure TForm1.btnSubmitClick(Sender: TObject);
begin
  lblWelcome.Caption := 'สวัสดี, ' + edtName.Text + '!';
end;
```

### ข้อ 4: ทดสอบ Debugger
**คำสั่ง:** เขียนโปรแกรม loop 1-10 ตั้ง breakpoint ที่ loop body แล้ว debug โดยกด F8 ทีละขั้น สังเกตค่าตัวแปรใน Watch Window

```pascal
program DebugTest;
var
  i, sum: Integer;
begin
  sum := 0;
  for i := 1 to 10 do
  begin
    sum := sum + i;   // ← ตั้ง breakpoint ที่บรรทัดนี้
    WriteLn('i=', i, ' sum=', sum);
  end;
end.
```

### ข้อ 5: Code Completion
**คำสั่ง:** พิมพ์โค้ดต่อไปนี้และทดสอบ Code Completion (Ctrl+Space):
```pascal
uses SysUtils;
var s: String;
begin
  s := IntToStr(42);
  // พิมพ์ "Str" แล้วกด Ctrl+Space ดู suggestions
end.
```

### ข้อ 6: Build Modes
**คำสั่ง:** สร้าง Build Modes สองแบบ:
- Debug: เปิด -g -gl, optimization ปิด
- Release: ปิด debug info, เปิด -O2

สังเกตขนาดไฟล์ที่ต่างกันระหว่าง Debug และ Release build

### ข้อ 7: Project Structure
**คำสั่ง:** สร้างโปรเจค GUI ที่มี 2 Forms:
- Form1: Main Window
- Form2: About Dialog

ศึกษาโครงสร้างไฟล์ที่สร้างขึ้น

```pascal
// ใน Unit1.pas เปิด Form2
procedure TForm1.btnAboutClick(Sender: TObject);
begin
  Form2.ShowModal;
end;
```

### ข้อ 8: Find and Replace
**คำสั่ง:** ใช้ Ctrl+H ทำ Find and Replace:
เปลี่ยน `WriteLn` ทั้งหมดเป็น `Write` ในโปรแกรม แล้วรันดูผลที่ต่างกัน

### ข้อ 9: Keyboard Navigation
**คำสั่ง:** ฝึก shortcuts โดยห้ามใช้ mouse:
1. เปิดไฟล์ใหม่ (Ctrl+N)
2. พิมพ์โค้ด Hello World
3. บันทึก (Ctrl+S)
4. Compile (Ctrl+F9)
5. Run (F9)
6. ปิดไฟล์ (Ctrl+W)

### ข้อ 10: Custom Theme
**คำสั่ง:** ตั้งค่า color scheme แบบ Dark:
```
Tools → Options → Editor → Colors
  Background: #1E1E1E
  Text: #D4D4D4
  Keywords: #569CD6 (blue)
  Strings: #CE9178 (orange)
  Comments: #6A9955 (green)
```
แล้ว Screenshot เปรียบเทียบก่อนและหลัง

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ความต้องการระบบ** - Windows/Linux/macOS พร้อมข้อกำหนดขั้นต่ำ
2. **การดาวน์โหลด** - จาก lazarus-ide.org หรือ sourceforge
3. **การติดตั้ง** - ขั้นตอนทีละขั้นบนแต่ละ OS
4. **การตั้งค่า IDE** - Font, Colors, FPC Paths, Build Modes
5. **ปลั๊กอินแนะนำ** - JVCL, Indy, LazReport, BGRABitmap
6. **สร้างโปรเจคแรก** - Console และ GUI Application
7. **โครงสร้างโปรเจค** - .lpi, .lfm, .pas files
8. **การใช้ Debugger** - Breakpoints, Watch, Evaluate
9. **Keyboard Shortcuts** - สำคัญที่ควรจำ

## บทต่อไป

**Part 03** จะพาเขียนโปรแกรมแรกอย่างละเอียด ทั้งแบบ Console และ GUI พร้อมการรับ input และแสดง output

---

*หมายเหตุ: ขั้นตอนการติดตั้งอาจแตกต่างกันเล็กน้อยตามเวอร์ชันของ Lazarus และ OS แนะนำให้ดูเอกสารอย่างเป็นทางการที่ wiki.lazarus.freepascal.org*
