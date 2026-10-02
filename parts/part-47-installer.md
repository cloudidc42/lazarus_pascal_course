# Part 47 - Creating Installers ใน Lazarus/Pascal

## บทนำ

การสร้าง installer ทำให้ผู้ใช้สามารถติดตั้งโปรแกรมได้ง่าย จัดการ files, registry entries, shortcuts และ uninstaller อย่างเป็นระบบ เครื่องมือที่นิยมสำหรับ Windows ได้แก่ Inno Setup และ NSIS

---

## 47.1 Inno Setup Basics

### ทำความรู้จัก Inno Setup

Inno Setup เป็น free installer creator สำหรับ Windows มี features:
- สร้าง installer .exe ขนาดเล็ก
- รองรับ compression หลายแบบ
- Scripting ด้วย Pascal-based language
- Custom UI และ wizard pages
- Unicode support

**การติดตั้ง Inno Setup:**
1. ดาวน์โหลดจาก https://jrsoftware.org/isinfo.php
2. ติดตั้ง Inno Setup Compiler
3. รัน ISCC.exe จาก command line

---

## 47.2 Creating Installer Script

### Inno Setup Script พื้นฐาน

```pascal
; app_installer.iss - Inno Setup Script

[Setup]
; ข้อมูลแอปพลิเคชัน
AppId={{F1234567-ABCD-1234-EFGH-1234567890AB}
AppName=MyApplication
AppVersion=2.0.1
AppPublisher=MyCompany Ltd.
AppPublisherURL=https://www.mycompany.com
AppSupportURL=https://support.mycompany.com
AppUpdatesURL=https://www.mycompany.com/updates

; ไดเรกทอรีการติดตั้ง
DefaultDirName={autopf}\MyApplication
DefaultGroupName=MyApplication
AllowNoIcons=yes

; Output
OutputDir=output
OutputBaseFilename=MyApplication_Setup_v2.0.1

; Compression
Compression=lzma2/ultra64
SolidCompression=yes

; UI
WizardStyle=modern
DisableProgramGroupPage=yes
LicenseFile=license.txt
InfoBeforeFile=readme.txt
InfoAfterFile=info_after.txt

; Architecture
ArchitecturesAllowed=x64compatible
ArchitecturesInstallIn64BitMode=x64compatible

; ต้องการสิทธิ์ admin
PrivilegesRequired=admin

; ไอคอน installer
SetupIconFile=app.ico

; Uninstall icon
UninstallDisplayIcon={app}\MyApp.exe

[Languages]
; รองรับหลายภาษา
Name: "english"; MessagesFile: "compiler:Default.isl"
Name: "thai"; MessagesFile: "compiler:Languages\Thai.isl"

[Tasks]
; Tasks ที่ user เลือกได้
Name: "desktopicon"; Description: "สร้าง Desktop shortcut"; GroupDescription: "Additional icons:"; Flags: unchecked
Name: "quicklaunchicon"; Description: "สร้าง Quick Launch shortcut"; GroupDescription: "Additional icons:"; Flags: unchecked; OnlyBelowVersion: 6.1
Name: "autostart"; Description: "เริ่มโปรแกรมอัตโนมัติเมื่อ Windows เริ่มต้น"; GroupDescription: "Startup:"

[Files]
; ไฟล์หลัก
Source: "dist\MyApp.exe"; DestDir: "{app}"; Flags: ignoreversion
Source: "dist\MyApp.dll"; DestDir: "{app}"; Flags: ignoreversion

; Runtime libraries
Source: "redist\vcredist_x64.exe"; DestDir: "{tmp}"; Flags: deleteafterinstall

; Documentation
Source: "docs\*"; DestDir: "{app}\docs"; Flags: ignoreversion recursesubdirs createallsubdirs

; Configuration files
Source: "config\default_config.json"; DestDir: "{app}\config"; Flags: ignoreversion onlyifdoesntexist

; Resources
Source: "resources\*"; DestDir: "{app}\resources"; Flags: ignoreversion recursesubdirs

[Dirs]
; สร้าง directories
Name: "{app}\logs"
Name: "{app}\data"
Name: "{app}\backups"

[Registry]
; เขียน registry entries
Root: HKLM; Subkey: "Software\MyCompany\MyApp"; ValueType: string; ValueName: "Version"; ValueData: "2.0.1"; Flags: createvalueifdoesntexist
Root: HKLM; Subkey: "Software\MyCompany\MyApp"; ValueType: string; ValueName: "InstallPath"; ValueData: "{app}"; Flags: createvalueifdoesntexist
Root: HKLM; Subkey: "Software\MyCompany\MyApp"; ValueType: dword; ValueName: "InstallDate"; ValueData: "{code:GetInstallDateInt}"; Flags: createvalueifdoesntexist

; Auto-start (conditional)
Root: HKCU; Subkey: "Software\Microsoft\Windows\CurrentVersion\Run"; ValueType: string; ValueName: "MyApplication"; ValueData: """{app}\MyApp.exe"" --minimized"; Tasks: autostart

; File association
Root: HKCR; Subkey: ".myapp"; ValueType: string; ValueName: ""; ValueData: "MyApp.Document"
Root: HKCR; Subkey: "MyApp.Document"; ValueType: string; ValueName: ""; ValueData: "MyApp Document"
Root: HKCR; Subkey: "MyApp.Document\DefaultIcon"; ValueType: string; ValueName: ""; ValueData: "{app}\MyApp.exe,0"
Root: HKCR; Subkey: "MyApp.Document\shell\open\command"; ValueType: string; ValueName: ""; ValueData: """{app}\MyApp.exe"" ""%1"""

[Icons]
; สร้าง shortcuts
Name: "{group}\MyApplication"; Filename: "{app}\MyApp.exe"; Comment: "Launch MyApplication"
Name: "{group}\Uninstall MyApplication"; Filename: "{uninstallexe}"
Name: "{autodesktop}\MyApplication"; Filename: "{app}\MyApp.exe"; Tasks: desktopicon
Name: "{userappdata}\Microsoft\Internet Explorer\Quick Launch\MyApplication"; Filename: "{app}\MyApp.exe"; Tasks: quicklaunchicon

[Run]
; รัน programs หลังติดตั้ง
; ติดตั้ง Visual C++ Redistributable ถ้าจำเป็น
Filename: "{tmp}\vcredist_x64.exe"; Parameters: "/quiet /norestart"; StatusMsg: "Installing required components..."; Check: NeedVCRedist

; เปิดโปรแกรมหลังติดตั้ง
Filename: "{app}\MyApp.exe"; Description: "Launch MyApplication"; Flags: nowait postinstall skipifsilent

; เปิด readme
Filename: "{app}\docs\readme.html"; Description: "View documentation"; Flags: nowait postinstall skipifsilent shellexec unchecked

[UninstallRun]
; รัน programs ก่อน uninstall
Filename: "{app}\MyApp.exe"; Parameters: "--uninstall-cleanup"; Flags: runhidden

[UninstallDelete]
; ลบ directories ที่สร้างโดยโปรแกรม
Type: filesandordirs; Name: "{app}\logs"
Type: filesandordirs; Name: "{app}\backups"

[Code]
{ Pascal code sections }

function InitializeSetup: Boolean;
begin
  { ตรวจสอบก่อน setup เริ่ม }
  Result := True;
  
  { ตรวจสอบ Windows version }
  if not IsWindows64BitInstall then
  begin
    MsgBox('This application requires 64-bit Windows.', mbError, MB_OK);
    Result := False;
    Exit;
  end;
  
  { ตรวจสอบ .NET Framework (ตัวอย่าง) }
  { if not IsDotNetInstalled then ... }
end;

function GetInstallDateInt(Param: String): String;
var
  Y, M, D: Word;
begin
  DecodeDate(Now, Y, M, D);
  Result := IntToStr(Y * 10000 + M * 100 + D);
end;

function NeedVCRedist: Boolean;
var
  Version: String;
begin
  Result := True;
  { ตรวจสอบว่าติดตั้ง VC++ Redist แล้วหรือยัง }
  if RegQueryStringValue(HKLM, 'SOFTWARE\Microsoft\VisualStudio\14.0\VC\Runtimes\x64', 
    'Version', Version) then
  begin
    Result := False; { Already installed }
  end;
end;

procedure InitializeWizard;
begin
  { Customize wizard pages }
  WizardForm.WelcomeLabel2.Caption := 
    'This will install MyApplication version 2.0.1 on your computer.' + #13#10 + #13#10 +
    'Please close all other applications before proceeding.';
end;

function NextButtonClick(CurPageID: Integer): Boolean;
begin
  Result := True;
  
  { Validate on specific pages }
  case CurPageID of
    wpSelectDir:
    begin
      { ตรวจสอบว่ามีพื้นที่เพียงพอ }
      if DiskSpaceMBLeft(WizardDirValue[1]) < 100 then
      begin
        MsgBox('Not enough disk space. Need at least 100 MB.', mbError, MB_OK);
        Result := False;
      end;
    end;
  end;
end;

procedure CurStepChanged(CurStep: TSetupStep);
begin
  if CurStep = ssPostInstall then
  begin
    { ทำงานหลังจากติดตั้งเสร็จ }
    { อาจสร้าง initial configuration, ฯลฯ }
  end;
end;

procedure CurUninstallStepChanged(CurUninstallStep: TUninstallStep);
begin
  if CurUninstallStep = usUninstall then
  begin
    { ถามผู้ใช้ก่อน uninstall }
    if MsgBox('Do you want to keep your settings and data?', mbConfirmation, MB_YESNO) = IDYES then
    begin
      { ไม่ลบ user data }
    end
    else
    begin
      { ลบ user data }
      DelTree(ExpandConstant('{app}\data'), True, True, True);
    end;
  end;
end;
```

---

## 47.3 File Installation

### ตัวอย่าง Script สำหรับ File Installation

```pascal
; file_installation.iss

[Files]
; ==========================================================
; Core Application Files
; ==========================================================
Source: "dist\release\MyApp.exe"; DestDir: "{app}"; Flags: ignoreversion
Source: "dist\release\*.dll"; DestDir: "{app}"; Flags: ignoreversion

; ==========================================================
; Database Files
; ==========================================================
; ติดตั้ง database template (overwrite ถ้าใหม่กว่า)
Source: "db\schema.sql"; DestDir: "{app}\db"; Flags: ignoreversion

; ติดตั้งเฉพาะถ้ายังไม่มี (ป้องกัน overwrite user data)
Source: "db\initial_data.db"; DestDir: "{userappdata}\MyApp"; Flags: onlyifdoesntexist uninsneveruninstall

; ==========================================================
; Configuration Files
; ==========================================================
; Default config (overwrite only if new)
Source: "config\app.json"; DestDir: "{app}"; Flags: ignoreversion

; User config (never overwrite)
Source: "config\user_defaults.json"; DestDir: "{userappdata}\MyApp"; DestName: "user.json"; Flags: onlyifdoesntexist uninsneveruninstall

; ==========================================================
; Resource Files
; ==========================================================
Source: "resources\themes\*"; DestDir: "{app}\themes"; Flags: ignoreversion recursesubdirs createallsubdirs
Source: "resources\icons\*"; DestDir: "{app}\icons"; Flags: ignoreversion recursesubdirs createallsubdirs
Source: "resources\sounds\*"; DestDir: "{app}\sounds"; Flags: ignoreversion recursesubdirs createallsubdirs

; ==========================================================
; Language Files
; ==========================================================
Source: "locales\*.json"; DestDir: "{app}\locales"; Flags: ignoreversion

; ==========================================================
; Documentation
; ==========================================================
Source: "docs\*.html"; DestDir: "{app}\help"; Flags: ignoreversion
Source: "docs\images\*"; DestDir: "{app}\help\images"; Flags: ignoreversion recursesubdirs

; ==========================================================
; Runtime Redistributables
; ==========================================================
; Visual C++ Redistributable
Source: "redist\VC_redist.x64.exe"; DestDir: "{tmp}"; Flags: deleteafterinstall; Check: NeedVCRedist

; ==========================================================
; Certificates (ถ้าจำเป็น)
; ==========================================================
; Source: "certs\myapp.cer"; DestDir: "{app}\certs"

[Code]
function NeedVCRedist: Boolean;
begin
  Result := not RegKeyExists(HKLM, 
    'SOFTWARE\Microsoft\VisualStudio\14.0\VC\Runtimes\x64');
end;
```

---

## 47.4 Registry Operations

### Registry Section ใน Inno Setup

```pascal
; registry_operations.iss

[Registry]
; ============================================================
; Application Registration
; ============================================================
; Version and install info
Root: HKLM; Subkey: "SOFTWARE\MyCompany\MyApp"; ValueType: string; ValueName: "Version"; ValueData: "2.0.1"
Root: HKLM; Subkey: "SOFTWARE\MyCompany\MyApp"; ValueType: string; ValueName: "InstallPath"; ValueData: "{app}"
Root: HKLM; Subkey: "SOFTWARE\MyCompany\MyApp"; ValueType: dword; ValueName: "BuildNumber"; ValueData: "1234"

; ============================================================
; Uninstall Information (for Add/Remove Programs)
; ============================================================
; Inno Setup จัดการส่วนนี้อัตโนมัติ แต่สามารถเพิ่มได้
Root: HKLM; Subkey: "SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{#SetupSetting('AppId')}_is1"; ValueType: string; ValueName: "DisplayVersion"; ValueData: "2.0.1"
Root: HKLM; Subkey: "SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{#SetupSetting('AppId')}_is1"; ValueType: string; ValueName: "Publisher"; ValueData: "MyCompany Ltd."
Root: HKLM; Subkey: "SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{#SetupSetting('AppId')}_is1"; ValueType: string; ValueName: "HelpLink"; ValueData: "https://support.mycompany.com"

; ============================================================
; File Associations
; ============================================================
Root: HKCR; Subkey: ".myapp"; ValueType: string; ValueName: ""; ValueData: "MyApp.Document"; Flags: uninsdeletevalue
Root: HKCR; Subkey: ".myapp"; ValueType: string; ValueName: "Content Type"; ValueData: "application/x-myapp"
Root: HKCR; Subkey: "MyApp.Document"; ValueType: string; ValueName: ""; ValueData: "MyApp Document"; Flags: uninsdeletekey
Root: HKCR; Subkey: "MyApp.Document\DefaultIcon"; ValueType: string; ValueName: ""; ValueData: "{app}\MyApp.exe,0"
Root: HKCR; Subkey: "MyApp.Document\shell\open\command"; ValueType: string; ValueName: ""; ValueData: """{app}\MyApp.exe"" ""%1"""
Root: HKCR; Subkey: "MyApp.Document\shell\print\command"; ValueType: string; ValueName: ""; ValueData: """{app}\MyApp.exe"" /print ""%1"""

; ============================================================
; Protocol Handlers (เช่น myapp://)
; ============================================================
Root: HKCR; Subkey: "myapp"; ValueType: string; ValueName: ""; ValueData: "URL:MyApp Protocol"; Flags: uninsdeletekey
Root: HKCR; Subkey: "myapp"; ValueType: string; ValueName: "URL Protocol"; ValueData: ""
Root: HKCR; Subkey: "myapp\shell\open\command"; ValueType: string; ValueName: ""; ValueData: """{app}\MyApp.exe"" ""%1"""

; ============================================================
; Context Menu (right-click on files)
; ============================================================
Root: HKCR; Subkey: "*\shell\Open with MyApp"; ValueType: string; ValueName: ""; ValueData: "Open with MyApp"
Root: HKCR; Subkey: "*\shell\Open with MyApp"; ValueType: string; ValueName: "Icon"; ValueData: "{app}\MyApp.exe,0"
Root: HKCR; Subkey: "*\shell\Open with MyApp\command"; ValueType: string; ValueName: ""; ValueData: """{app}\MyApp.exe"" ""%1"""

; ============================================================
; Auto-Start (Tasks conditional)
; ============================================================
Root: HKCU; Subkey: "Software\Microsoft\Windows\CurrentVersion\Run"; ValueType: string; ValueName: "MyApplication"; ValueData: """{app}\MyApp.exe"" --silent"; Tasks: autostart
Root: HKCU; Subkey: "Software\Microsoft\Windows\CurrentVersion\Run"; ValueType: none; ValueName: "MyApplication"; Flags: deletevalue; Tasks: not autostart
```

---

## 47.5 Shortcuts

### Shortcuts Section

```pascal
; shortcuts.iss

[Icons]
; ============================================================
; Start Menu
; ============================================================
; Main application shortcut
Name: "{group}\{cm:AppName}"; Filename: "{app}\MyApp.exe"; Comment: "Launch MyApplication"
Name: "{group}\{cm:AppName} (Safe Mode)"; Filename: "{app}\MyApp.exe"; Parameters: "--safe-mode"; IconFilename: "{app}\MyApp.exe"; IconIndex: 1
Name: "{group}\User Guide"; Filename: "{app}\help\index.html"; Flags: shellexec
Name: "{group}\{cm:UninstallProgram,{cm:AppName}}"; Filename: "{uninstallexe}"

; ============================================================
; Desktop
; ============================================================
Name: "{autodesktop}\{cm:AppName}"; Filename: "{app}\MyApp.exe"; Tasks: desktopicon

; ============================================================
; Quick Launch
; ============================================================
Name: "{userappdata}\Microsoft\Internet Explorer\Quick Launch\{cm:AppName}"; Filename: "{app}\MyApp.exe"; Tasks: quicklaunchicon; OnlyBelowVersion: 0,6.1

; ============================================================
; TaskBar Pin (Windows 7+)
; ============================================================
; ทำผ่าน code section:
; นี่คือตัวอย่างใน Code section

[Code]
procedure PinToTaskbar(const ExePath: String);
var
  Shell: Variant;
  Folder: Variant;
  Item: Variant;
  Verb: Variant;
  VerbName: String;
  i: Integer;
begin
  Shell := CreateOleObject('Shell.Application');
  Folder := Shell.NameSpace(ExtractFilePath(ExePath));
  Item := Folder.ParseName(ExtractFileName(ExePath));
  
  for i := 0 to Item.Verbs.Count - 1 do
  begin
    Verb := Item.Verbs.Item(i);
    VerbName := Verb.Name;
    if (Pos('taskbar', LowerCase(VerbName)) > 0) and 
       (Pos('pin', LowerCase(VerbName)) > 0) then
    begin
      Verb.DoIt;
      Break;
    end;
  end;
end;

procedure CurStepChanged(CurStep: TSetupStep);
begin
  if CurStep = ssPostInstall then
  begin
    // Pin to taskbar ถ้าผู้ใช้เลือก
    // if IsTaskSelected('taskbarpin') then
    //   PinToTaskbar(ExpandConstant('{app}\MyApp.exe'));
  end;
end;
```

---

## 47.6 Uninstaller

### การสร้าง Uninstaller

```pascal
; uninstaller.iss

[Setup]
; Uninstaller settings
CreateUninstallRegKey=yes
UninstallDisplayName=MyApplication 2.0.1
UninstallDisplayIcon={app}\MyApp.exe

[UninstallRun]
; รัน cleanup ก่อน uninstall
Filename: "{app}\MyApp.exe"; Parameters: "--cleanup-before-uninstall"; Flags: runhidden; RunOnceId: "CleanupBeforeUninstall"

[UninstallDelete]
; ลบ directories และ files ที่สร้างระหว่างการใช้งาน
Type: files; Name: "{app}\*.log"
Type: files; Name: "{app}\*.tmp"
Type: files; Name: "{app}\*.cache"
Type: filesandordirs; Name: "{app}\logs"
Type: filesandordirs; Name: "{app}\cache"
Type: dirifempty; Name: "{app}"

[Code]
procedure CurUninstallStepChanged(CurUninstallStep: TUninstallStep);
var
  KeepSettings: Boolean;
begin
  if CurUninstallStep = usUninstall then
  begin
    // ถามผู้ใช้
    KeepSettings := MsgBox(
      'Do you want to keep your personal settings and data?' + #13#10 + #13#10 +
      'Click Yes to keep your settings.' + #13#10 +
      'Click No to remove everything.',
      mbConfirmation, MB_YESNO or MB_DEFBUTTON1
    ) = IDYES;
    
    if not KeepSettings then
    begin
      // ลบ user data
      DelTree(ExpandConstant('{userappdata}\MyApp'), True, True, True);
      
      // ลบ registry user settings
      RegDeleteKeyIncludingSubkeys(HKCU, 'Software\MyCompany\MyApp');
    end
    else
    begin
      // บอกผู้ใช้ว่า settings ยังอยู่
      MsgBox(
        'Your settings have been kept at:' + #13#10 +
        ExpandConstant('{userappdata}\MyApp'),
        mbInformation, MB_OK
      );
    end;
  end;
  
  if CurUninstallStep = usPostUninstall then
  begin
    // Cleanup เพิ่มเติมหลัง uninstall
    
    // ลบ registry keys ที่เหลือ
    RegDeleteKeyIncludingSubkeys(HKLM, 'SOFTWARE\MyCompany\MyApp');
    
    // ถ้าเป็น company's last product ลบ company key ด้วย
    if not RegKeyExists(HKLM, 'SOFTWARE\MyCompany') then
      RegDeleteKeyIncludingSubkeys(HKLM, 'SOFTWARE\MyCompany');
  end;
end;
```

---

## 47.7 Custom Installer Pages

### Custom Pages ใน Inno Setup

```pascal
; custom_pages.iss

[Code]

{ ================================================================
  Custom Pages
  ================================================================ }

var
  LicensePage: TWizardPage;
  DatabasePage: TWizardPage;
  DBServerEdit: TEdit;
  DBPortEdit: TEdit;
  DBNameEdit: TEdit;
  DBUserEdit: TEdit;
  DBPasswordEdit: TEdit;
  TestConnectionBtn: TButton;

procedure TestConnectionBtnClick(Sender: TObject);
begin
  MsgBox('Testing connection to ' + DBServerEdit.Text + ':' + DBPortEdit.Text + '...',
    mbInformation, MB_OK);
  // ใน production จะทดสอบ connection จริงๆ
end;

procedure InitializeWizard;
begin
  { Page 1: Custom Database Configuration }
  DatabasePage := CreateCustomPage(wpSelectComponents,
    'Database Configuration',
    'Please enter your database connection details.');
  
  // Server
  with TLabel.Create(DatabasePage) do
  begin
    Caption := 'Database Server:';
    Left := 0; Top := 0;
    Width := 150; Height := 20;
    Parent := DatabasePage.Surface;
  end;
  
  DBServerEdit := TEdit.Create(DatabasePage);
  DBServerEdit.Left := 0;
  DBServerEdit.Top := 22;
  DBServerEdit.Width := 250;
  DBServerEdit.Text := 'localhost';
  DBServerEdit.Parent := DatabasePage.Surface;
  
  // Port
  with TLabel.Create(DatabasePage) do
  begin
    Caption := 'Port:';
    Left := 270; Top := 0;
    Width := 50; Height := 20;
    Parent := DatabasePage.Surface;
  end;
  
  DBPortEdit := TEdit.Create(DatabasePage);
  DBPortEdit.Left := 270;
  DBPortEdit.Top := 22;
  DBPortEdit.Width := 80;
  DBPortEdit.Text := '5432';
  DBPortEdit.Parent := DatabasePage.Surface;
  
  // Database Name
  with TLabel.Create(DatabasePage) do
  begin
    Caption := 'Database Name:';
    Left := 0; Top := 55;
    Width := 150; Height := 20;
    Parent := DatabasePage.Surface;
  end;
  
  DBNameEdit := TEdit.Create(DatabasePage);
  DBNameEdit.Left := 0;
  DBNameEdit.Top := 77;
  DBNameEdit.Width := 350;
  DBNameEdit.Text := 'myapp_db';
  DBNameEdit.Parent := DatabasePage.Surface;
  
  // Username
  with TLabel.Create(DatabasePage) do
  begin
    Caption := 'Username:';
    Left := 0; Top := 110;
    Width := 150; Height := 20;
    Parent := DatabasePage.Surface;
  end;
  
  DBUserEdit := TEdit.Create(DatabasePage);
  DBUserEdit.Left := 0;
  DBUserEdit.Top := 132;
  DBUserEdit.Width := 170;
  DBUserEdit.Text := 'admin';
  DBUserEdit.Parent := DatabasePage.Surface;
  
  // Password
  with TLabel.Create(DatabasePage) do
  begin
    Caption := 'Password:';
    Left := 190; Top := 110;
    Width := 150; Height := 20;
    Parent := DatabasePage.Surface;
  end;
  
  DBPasswordEdit := TEdit.Create(DatabasePage);
  DBPasswordEdit.Left := 190;
  DBPasswordEdit.Top := 132;
  DBPasswordEdit.Width := 170;
  DBPasswordEdit.PasswordChar := '*';
  DBPasswordEdit.Parent := DatabasePage.Surface;
  
  // Test Connection button
  TestConnectionBtn := TButton.Create(DatabasePage);
  TestConnectionBtn.Left := 0;
  TestConnectionBtn.Top := 165;
  TestConnectionBtn.Width := 150;
  TestConnectionBtn.Height := 25;
  TestConnectionBtn.Caption := 'Test Connection';
  TestConnectionBtn.OnClick := @TestConnectionBtnClick;
  TestConnectionBtn.Parent := DatabasePage.Surface;
end;

function NextButtonClick(CurPageID: Integer): Boolean;
begin
  Result := True;
  
  // Validate database page
  if CurPageID = DatabasePage.ID then
  begin
    if Trim(DBServerEdit.Text) = '' then
    begin
      MsgBox('Please enter a database server.', mbError, MB_OK);
      DBServerEdit.SetFocus;
      Result := False;
      Exit;
    end;
    
    if Trim(DBNameEdit.Text) = '' then
    begin
      MsgBox('Please enter a database name.', mbError, MB_OK);
      DBNameEdit.SetFocus;
      Result := False;
      Exit;
    end;
    
    // บันทึกค่า (จะใช้ในการสร้าง config file)
    SetIniString('Database', 'Server', DBServerEdit.Text, ExpandConstant('{tmp}\setup_config.ini'));
    SetIniString('Database', 'Port', DBPortEdit.Text, ExpandConstant('{tmp}\setup_config.ini'));
    SetIniString('Database', 'Name', DBNameEdit.Text, ExpandConstant('{tmp}\setup_config.ini'));
    SetIniString('Database', 'User', DBUserEdit.Text, ExpandConstant('{tmp}\setup_config.ini'));
  end;
end;

procedure CurStepChanged(CurStep: TSetupStep);
var
  ConfigFile: String;
begin
  if CurStep = ssPostInstall then
  begin
    // อ่านค่าจาก temp config และเขียนลง app config
    ConfigFile := ExpandConstant('{app}\config.ini');
    
    SetIniString('Database', 'Server', 
      GetIniString('Database', 'Server', 'localhost', ExpandConstant('{tmp}\setup_config.ini')),
      ConfigFile);
    // ... etc
  end;
end;
```

---

## 47.8 Silent Installation

### Silent Installation

```pascal
; silent_install.iss

[Setup]
; อนุญาต silent install
AllowCancelDuringInstall=no

; ============================================================
; Command line options:
; /SILENT - แสดง progress window แต่ไม่มี wizard
; /VERYSILENT - ไม่แสดง window เลย
; /SUPPRESSMSGBOXES - suppress error dialog boxes
; /NORESTART - ไม่ restart หลังติดตั้ง
; /DIR="path" - กำหนด install directory
; /GROUP="name" - กำหนด start menu group
; /NOICONS - ไม่สร้าง icons
; /TYPE=full|minimal - กำหนด install type
; /COMPONENTS="comp1,comp2" - เลือก components
; /TASKS="task1,task2" - เลือก tasks
; ============================================================

[Code]

procedure InitializeWizard;
begin
  if WizardSilent then
  begin
    // ไม่ต้องทำอะไรใน silent mode
    Exit;
  end;
end;

function PrepareToInstall(var NeedsRestart: Boolean): String;
begin
  // ตรวจสอบก่อนติดตั้ง
  Result := '';
  
  // ถ้า silent mode และมีปัญหา return error message
  if SomeCheckFails then
    Result := 'Required component not found: XYZ';
end;

{ Silent logging }
procedure Log(const Msg: String);
var
  LogFile: String;
begin
  LogFile := ExpandConstant('{log}');
  // Inno Setup มี logging built-in ผ่าน /LOG parameter
end;
```

### Batch Script สำหรับ Silent Install

```batch
@echo off
:: silent_install.bat

:: ติดตั้งแบบ silent พร้อม log
MyApplication_Setup_v2.0.1.exe ^
  /VERYSILENT ^
  /SUPPRESSMSGBOXES ^
  /NORESTART ^
  /DIR="C:\Program Files\MyApplication" ^
  /LOG="C:\Temp\install_log.txt"

:: ตรวจสอบ exit code
if %ERRORLEVEL% EQU 0 (
  echo Installation successful
) else (
  echo Installation failed with code %ERRORLEVEL%
)

:: รอให้ installer จบก่อน continue
```

---

## 47.9 MSI Creation

### การสร้าง MSI Package

สำหรับ Enterprise deployment มักต้องการ MSI format ซึ่งสามารถใช้ WiX Toolset

```xml
<!-- product.wxs - WiX Source File -->
<?xml version="1.0" encoding="UTF-8"?>
<Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">

  <Product Id="{F1234567-ABCD-1234-EFGH-1234567890AB}"
           Name="MyApplication"
           Language="1033"
           Version="2.0.1.0"
           Manufacturer="MyCompany Ltd."
           UpgradeCode="{AAAABBBB-CCCC-DDDD-EEEE-FFFFAAAABBBB}">

    <Package InstallerVersion="500"
             Compressed="yes"
             InstallScope="perMachine"
             Description="MyApplication Setup"
             Comments="MyApplication v2.0.1" />

    <MajorUpgrade DowngradeErrorMessage="A newer version of [ProductName] is already installed." />

    <MediaTemplate EmbedCab="yes" />

    <!-- Features -->
    <Feature Id="ProductFeature" Title="MyApplication" Level="1">
      <ComponentGroupRef Id="ProductComponents" />
    </Feature>

  </Product>

  <Fragment>
    <Directory Id="TARGETDIR" Name="SourceDir">
      <Directory Id="ProgramFiles64Folder">
        <Directory Id="INSTALLFOLDER" Name="MyApplication" />
      </Directory>
      <Directory Id="ProgramMenuFolder">
        <Directory Id="ApplicationProgramsFolder" Name="MyApplication"/>
      </Directory>
    </Directory>
  </Fragment>

  <Fragment>
    <ComponentGroup Id="ProductComponents" Directory="INSTALLFOLDER">

      <!-- Main executable -->
      <Component Id="MyApp.exe" Guid="{AAAABBBB-1111-2222-3333-444455556666}">
        <File Id="MyApp.exe"
              Source="dist\MyApp.exe"
              KeyPath="yes" />
      </Component>

      <!-- Start Menu Shortcut -->
      <Component Id="ApplicationShortcut" Guid="{AAAABBBB-7777-8888-9999-000011112222}">
        <Shortcut Id="ApplicationStartMenuShortcut"
                  Name="MyApplication"
                  Description="Launch MyApplication"
                  Target="[INSTALLFOLDER]MyApp.exe"
                  WorkingDirectory="INSTALLFOLDER" />
        <RemoveFolder Id="CleanUpShortCut"
                      Directory="ApplicationProgramsFolder"
                      On="uninstall" />
        <RegistryValue Root="HKCU"
                       Key="Software\MyCompany\MyApp"
                       Name="installed"
                       Type="integer"
                       Value="1"
                       KeyPath="yes" />
      </Component>

    </ComponentGroup>
  </Fragment>

</Wix>
```

### Build MSI ด้วย WiX

```bash
# Build MSI
candle.exe product.wxs -out product.wixobj -arch x64
light.exe product.wixobj -out MyApplication.msi -ext WixUIExtension

# Silent install
msiexec /i MyApplication.msi /quiet /norestart

# Silent uninstall
msiexec /x {F1234567-ABCD-1234-EFGH-1234567890AB} /quiet /norestart
```

---

## 47.10 NSIS Alternative

### NSIS (Nullsoft Scriptable Install System)

```nsis
; nsis_installer.nsi

!include "MUI2.nsh"
!include "x64.nsh"
!include "LogicLib.nsh"

; ============================================================
; Application Information
; ============================================================
Name "MyApplication"
OutFile "MyApplication_Setup_NSIS.exe"
InstallDir "$PROGRAMFILES64\MyApplication"
InstallDirRegKey HKLM "Software\MyCompany\MyApp" "InstallPath"
RequestExecutionLevel admin

; ============================================================
; Interface Settings
; ============================================================
!define MUI_ABORTWARNING
!define MUI_ICON "app.ico"
!define MUI_UNICON "uninstall.ico"

; Pages
!insertmacro MUI_PAGE_WELCOME
!insertmacro MUI_PAGE_LICENSE "license.txt"
!insertmacro MUI_PAGE_COMPONENTS
!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_PAGE_FINISH

; Uninstall pages
!insertmacro MUI_UNPAGE_WELCOME
!insertmacro MUI_UNPAGE_CONFIRM
!insertmacro MUI_UNPAGE_INSTFILES
!insertmacro MUI_UNPAGE_FINISH

; Languages
!insertmacro MUI_LANGUAGE "English"
!insertmacro MUI_LANGUAGE "Thai"

; ============================================================
; Install Sections
; ============================================================
Section "Core Application" SecCore
  SectionIn RO  ; Required - cannot be deselected
  
  SetOutPath "$INSTDIR"
  File "dist\MyApp.exe"
  File "dist\*.dll"
  
  SetOutPath "$INSTDIR\config"
  File "config\*.json"
  
  ; Create directories
  CreateDirectory "$INSTDIR\logs"
  CreateDirectory "$INSTDIR\data"
  
  ; Registry entries
  WriteRegStr HKLM "Software\MyCompany\MyApp" "Version" "2.0.1"
  WriteRegStr HKLM "Software\MyCompany\MyApp" "InstallPath" "$INSTDIR"
  
  ; Uninstall info
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyApp" \
    "DisplayName" "MyApplication"
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyApp" \
    "UninstallString" '"$INSTDIR\uninstall.exe"'
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyApp" \
    "DisplayIcon" "$INSTDIR\MyApp.exe"
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyApp" \
    "Publisher" "MyCompany Ltd."
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyApp" \
    "DisplayVersion" "2.0.1"
  
  ; Write uninstaller
  WriteUninstaller "$INSTDIR\uninstall.exe"
  
  ; Start menu shortcuts
  CreateDirectory "$SMPROGRAMS\MyApplication"
  CreateShortcut "$SMPROGRAMS\MyApplication\MyApplication.lnk" "$INSTDIR\MyApp.exe"
  CreateShortcut "$SMPROGRAMS\MyApplication\Uninstall.lnk" "$INSTDIR\uninstall.exe"
SectionEnd

Section "Documentation" SecDocs
  SetOutPath "$INSTDIR\help"
  File /r "docs\*.*"
SectionEnd

Section "Desktop Shortcut" SecDesktop
  CreateShortcut "$DESKTOP\MyApplication.lnk" "$INSTDIR\MyApp.exe"
SectionEnd

; ============================================================
; Uninstall Section
; ============================================================
Section "Uninstall"
  ; ถามว่าจะเก็บ data ไว้หรือไม่
  MessageBox MB_YESNO "Do you want to keep your personal data?" IDYES KeepData
  
  ; ลบ user data
  RMDir /r "$APPDATA\MyApp"
  
  KeepData:
  
  ; ลบ files
  Delete "$INSTDIR\MyApp.exe"
  Delete "$INSTDIR\*.dll"
  Delete "$INSTDIR\uninstall.exe"
  RMDir /r "$INSTDIR\config"
  RMDir /r "$INSTDIR\help"
  RMDir /r "$INSTDIR\logs"
  RMDir "$INSTDIR"
  
  ; ลบ shortcuts
  Delete "$SMPROGRAMS\MyApplication\*.*"
  RMDir "$SMPROGRAMS\MyApplication"
  Delete "$DESKTOP\MyApplication.lnk"
  
  ; ลบ registry
  DeleteRegKey HKLM "Software\MyCompany\MyApp"
  DeleteRegKey HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyApp"
SectionEnd
```

---

## 47.11 Complete Example: App Installer

```pascal
; complete_installer.iss - Complete Installer Script

#define MyAppName "MyShop ERP"
#define MyAppVersion "3.0.0"
#define MyAppPublisher "MyCompany Solutions"
#define MyAppURL "https://www.mycompany.com"
#define MyAppExeName "MyShop.exe"
#define MyAppID "{{AABBCCDD-1234-5678-ABCD-EF0123456789}"

[Setup]
AppId={#MyAppID}
AppName={#MyAppName}
AppVersion={#MyAppVersion}
AppPublisher={#MyAppPublisher}
AppPublisherURL={#MyAppURL}
AppSupportURL={#MyAppURL}/support
DefaultDirName={autopf}\{#MyAppName}
DefaultGroupName={#MyAppName}
AllowNoIcons=yes
LicenseFile=LICENSE.txt
OutputDir=output
OutputBaseFilename={#MyAppName}_Setup_{#MyAppVersion}
SetupIconFile=resources\app.ico
Compression=lzma2/ultra64
SolidCompression=yes
WizardStyle=modern
PrivilegesRequired=admin
ArchitecturesAllowed=x64compatible
ArchitecturesInstallIn64BitMode=x64compatible
MinVersion=10.0
UninstallDisplayName={#MyAppName}
UninstallDisplayIcon={app}\{#MyAppExeName}
VersionInfoVersion={#MyAppVersion}.0
VersionInfoCompany={#MyAppPublisher}
VersionInfoDescription={#MyAppName} Setup
VersionInfoCopyright=Copyright © 2024 {#MyAppPublisher}

[Languages]
Name: "english"; MessagesFile: "compiler:Default.isl"

[Tasks]
Name: "desktopicon"; Description: "Create a &desktop icon"; GroupDescription: "{cm:AdditionalIcons}"; Flags: unchecked
Name: "autostart"; Description: "Start automatically with Windows"; GroupDescription: "Startup options:"
Name: "createdb"; Description: "Create and initialize database"; GroupDescription: "Database:"

[Components]
Name: "main"; Description: "Core Application"; Types: full compact custom; Flags: fixed
Name: "docs"; Description: "Documentation"; Types: full
Name: "samples"; Description: "Sample Data"; Types: full; Flags: unchecked
Name: "devtools"; Description: "Developer Tools"; Types: custom; Flags: unchecked

[Files]
; Core
Source: "dist\{#MyAppExeName}"; DestDir: "{app}"; Flags: ignoreversion; Components: main
Source: "dist\*.dll"; DestDir: "{app}"; Flags: ignoreversion; Components: main
Source: "dist\config\*"; DestDir: "{app}\config"; Flags: ignoreversion recursesubdirs; Components: main
Source: "dist\resources\*"; DestDir: "{app}\resources"; Flags: ignoreversion recursesubdirs; Components: main
Source: "dist\locales\*"; DestDir: "{app}\locales"; Flags: ignoreversion recursesubdirs; Components: main

; Documentation
Source: "docs\*"; DestDir: "{app}\docs"; Flags: ignoreversion recursesubdirs; Components: docs

; Sample data
Source: "samples\*"; DestDir: "{app}\samples"; Flags: ignoreversion recursesubdirs; Components: samples

; Dev tools
Source: "devtools\*"; DestDir: "{app}\devtools"; Flags: ignoreversion recursesubdirs; Components: devtools

; Redistributables
Source: "redist\vcredist_x64.exe"; DestDir: "{tmp}"; Flags: deleteafterinstall; Check: NeedVCRedist

[Dirs]
Name: "{app}\data"
Name: "{app}\logs"
Name: "{app}\backups"
Name: "{app}\exports"
Name: "{app}\imports"
Name: "{userappdata}\{#MyAppName}"
Name: "{userappdata}\{#MyAppName}\reports"

[Icons]
Name: "{group}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"
Name: "{group}\Documentation"; Filename: "{app}\docs\index.html"; Flags: shellexec; Components: docs
Name: "{group}\{cm:UninstallProgram,{#MyAppName}}"; Filename: "{uninstallexe}"
Name: "{autodesktop}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"; Tasks: desktopicon

[Registry]
Root: HKLM; Subkey: "SOFTWARE\{#MyAppPublisher}\{#MyAppName}"; ValueType: string; ValueName: "Version"; ValueData: "{#MyAppVersion}"; Flags: createvalueifdoesntexist
Root: HKLM; Subkey: "SOFTWARE\{#MyAppPublisher}\{#MyAppName}"; ValueType: string; ValueName: "InstallPath"; ValueData: "{app}"
Root: HKCU; Subkey: "Software\Microsoft\Windows\CurrentVersion\Run"; ValueType: string; ValueName: "{#MyAppName}"; ValueData: """{app}\{#MyAppExeName}"" --minimized"; Tasks: autostart

[Run]
Filename: "{tmp}\vcredist_x64.exe"; Parameters: "/quiet /norestart"; StatusMsg: "Installing required components..."; Check: NeedVCRedist
Filename: "{app}\{#MyAppExeName}"; Parameters: "--initialize-db"; StatusMsg: "Initializing database..."; Tasks: createdb; Flags: runhidden waituntilterminated
Filename: "{app}\{#MyAppExeName}"; Description: "{cm:LaunchProgram,{#StringChange(MyAppName, '&', '&&')}}"; Flags: nowait postinstall skipifsilent

[UninstallRun]
Filename: "{app}\{#MyAppExeName}"; Parameters: "--cleanup"; Flags: runhidden; RunOnceId: "Cleanup"

[UninstallDelete]
Type: filesandordirs; Name: "{app}\logs"
Type: filesandordirs; Name: "{app}\cache"

[Code]

function NeedVCRedist: Boolean;
begin
  Result := not RegKeyExists(HKLM, 'SOFTWARE\Microsoft\VisualStudio\14.0\VC\Runtimes\x64');
end;

var
  DBPage: TWizardPage;
  DBHostEdit, DBPortEdit, DBNameEdit, DBUserEdit, DBPasswordEdit: TEdit;

procedure InitializeWizard;
begin
  // Create custom DB configuration page
  DBPage := CreateCustomPage(wpSelectComponents,
    'Database Configuration',
    'Enter database connection settings for {#MyAppName}.');

  with TLabel.Create(DBPage) do begin
    Parent := DBPage.Surface;
    Left := 0; Top := 0; Width := 200; Height := 20;
    Caption := 'Database Server:';
  end;

  DBHostEdit := TEdit.Create(DBPage);
  DBHostEdit.Parent := DBPage.Surface;
  DBHostEdit.Left := 0; DBHostEdit.Top := 22;
  DBHostEdit.Width := 250; DBHostEdit.Text := 'localhost';

  with TLabel.Create(DBPage) do begin
    Parent := DBPage.Surface;
    Left := 260; Top := 0; Width := 80; Height := 20;
    Caption := 'Port:';
  end;

  DBPortEdit := TEdit.Create(DBPage);
  DBPortEdit.Parent := DBPage.Surface;
  DBPortEdit.Left := 260; DBPortEdit.Top := 22;
  DBPortEdit.Width := 80; DBPortEdit.Text := '3306';

  // More fields...
  with TLabel.Create(DBPage) do begin
    Parent := DBPage.Surface;
    Left := 0; Top := 55; Width := 200; Height := 20;
    Caption := 'Database Name:';
  end;

  DBNameEdit := TEdit.Create(DBPage);
  DBNameEdit.Parent := DBPage.Surface;
  DBNameEdit.Left := 0; DBNameEdit.Top := 77;
  DBNameEdit.Width := 340; DBNameEdit.Text := 'myshop_db';
end;

function NextButtonClick(CurPageID: Integer): Boolean;
begin
  Result := True;

  if CurPageID = DBPage.ID then
  begin
    if Trim(DBHostEdit.Text) = '' then
    begin
      MsgBox('Please enter the database server address.', mbError, MB_OK);
      DBHostEdit.SetFocus;
      Result := False;
    end;
  end;
end;

procedure CurStepChanged(CurStep: TSetupStep);
var
  ConfigFile: String;
begin
  if CurStep = ssPostInstall then
  begin
    // Write database config
    ConfigFile := ExpandConstant('{app}\config\database.ini');
    SetIniString('Database', 'Host', DBHostEdit.Text, ConfigFile);
    SetIniString('Database', 'Port', DBPortEdit.Text, ConfigFile);
    SetIniString('Database', 'Name', DBNameEdit.Text, ConfigFile);

    // Create initial directories
    ForceDirectories(ExpandConstant('{userappdata}\{#MyAppName}\reports'));
    ForceDirectories(ExpandConstant('{userappdata}\{#MyAppName}\backups'));
  end;
end;

procedure CurUninstallStepChanged(CurUninstallStep: TUninstallStep);
begin
  if CurUninstallStep = usUninstall then
  begin
    if MsgBox(
      'Do you want to remove all {#MyAppName} data and settings?' + #13#10 +
      'WARNING: This will permanently delete all your data!',
      mbConfirmation, MB_YESNO or MB_DEFBUTTON2
    ) = IDYES then
    begin
      DelTree(ExpandConstant('{userappdata}\{#MyAppName}'), True, True, True);
      RegDeleteKeyIncludingSubkeys(HKLM, 'SOFTWARE\{#MyAppPublisher}\{#MyAppName}');
    end;
  end;
end;
```

---

## แบบฝึกหัด 5 ข้อ

### ข้อ 1: Simple Installer
สร้าง installer สำหรับ Hello World app ที่มี:
- Welcome page
- License agreement
- Install directory selection
- Desktop shortcut option
- Launch after install

### ข้อ 2: Multi-Language Installer
สร้าง installer รองรับ 2 ภาษา:
- ภาษาไทย
- ภาษาอังกฤษ
- Custom messages ในแต่ละภาษา

### ข้อ 3: Component-Based Installer
Installer ที่มี optional components:
- Core (required)
- Documentation (optional)
- Plugins (optional, multiple)
- แต่ละ component มี description และ size

### ข้อ 4: Database Setup Installer
Installer ที่:
- ขอ database connection info
- Test connection ก่อนติดตั้ง
- สร้าง database schema
- Validate settings

### ข้อ 5: Enterprise MSI
สร้าง MSI ด้วย WiX Toolset ที่:
- รองรับ Group Policy installation
- Logging
- Transform files สำหรับ customization
- Per-machine installation
