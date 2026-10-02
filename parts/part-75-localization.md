# ตอนที่ 75: Internationalization (i18n) ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการสร้างแอปพลิเคชันที่รองรับหลายภาษา (i18n/l10n) ด้วย Lazarus โดยใช้ .po files และ gettext

---

## 75.1 การใช้ Lazarus i18n

```pascal
unit main_form;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Buttons, StdCtrls,
  Menus, ComCtrls,
  LCLTranslator,  // สำคัญ: ต้องใช้ unit นี้
  Translations;   // สำหรับ .po file handling

type
  TMainForm = class(TForm)
    MenuBar: TMainMenu;
    mnuFile: TMenuItem;
    mnuFileNew: TMenuItem;
    mnuFileOpen: TMenuItem;
    mnuFileSave: TMenuItem;
    mnuFileSep: TMenuItem;
    mnuFileExit: TMenuItem;
    mnuLanguage: TMenuItem;
    mnuLangThai: TMenuItem;
    mnuLangEnglish: TMenuItem;
    mnuLangChinese: TMenuItem;
    
    lblTitle: TLabel;
    lblUsername: TLabel;
    lblPassword: TLabel;
    edtUsername: TEdit;
    edtPassword: TEdit;
    btnLogin: TButton;
    btnCancel: TButton;
    
    StatusBar: TStatusBar;
    
    procedure FormCreate(Sender: TObject);
    procedure mnuLangThaiClick(Sender: TObject);
    procedure mnuLangEnglishClick(Sender: TObject);
    procedure mnuLangChineseClick(Sender: TObject);
    procedure btnLoginClick(Sender: TObject);
    
  private
    FCurrentLang: string;
    
    procedure SwitchLanguage(const ALangCode: string);
    procedure UpdateUI;
    
  public
    property CurrentLang: string read FCurrentLang;
  end;

var
  MainForm: TMainForm;

implementation

{$R *.lfm}

procedure TMainForm.FormCreate(Sender: TObject);
begin
  FCurrentLang := 'en';
  
  // โหลดภาษาตามระบบ
  var SysLang := Copy(GetDefaultLanguage, 1, 2);
  if SysLang = 'th' then
    SwitchLanguage('th')
  else
    SwitchLanguage('en');
end;

procedure TMainForm.SwitchLanguage(const ALangCode: string);
var
  POFile: string;
begin
  FCurrentLang := ALangCode;
  
  // หา .po file
  POFile := ExtractFilePath(Application.ExeName) + 
    'locale' + PathDelim + ALangCode + PathDelim +
    ExtractFileName(ChangeFileExt(Application.ExeName, '.po'));
    
  // โหลด translation
  if FileExists(POFile) then
  begin
    LCLTranslator.TranslateUnitResourceStrings('MainForm', POFile);
    // Translate all forms
    SetDefaultLang(ALangCode);
  end;
  
  UpdateUI;
end;

procedure TMainForm.UpdateUI;
begin
  // Re-apply translations to all visible strings
  Caption := rsMainTitle;
  lblTitle.Caption := rsLoginTitle;
  lblUsername.Caption := rsUsername;
  lblPassword.Caption := rsPassword;
  btnLogin.Caption := rsLogin;
  btnCancel.Caption := rsCancel;
  
  mnuFile.Caption := rsMenuFile;
  mnuFileNew.Caption := rsMenuNew;
  mnuFileOpen.Caption := rsMenuOpen;
  mnuFileSave.Caption := rsMenuSave;
  mnuFileExit.Caption := rsMenuExit;
  mnuLanguage.Caption := rsMenuLanguage;
  
  StatusBar.Panels[0].Text := Format(rsStatusReady, [FCurrentLang]);
end;

procedure TMainForm.mnuLangThaiClick(Sender: TObject);
begin
  SwitchLanguage('th');
  mnuLangThai.Checked := True;
  mnuLangEnglish.Checked := False;
  mnuLangChinese.Checked := False;
end;

procedure TMainForm.mnuLangEnglishClick(Sender: TObject);
begin
  SwitchLanguage('en');
  mnuLangThai.Checked := False;
  mnuLangEnglish.Checked := True;
  mnuLangChinese.Checked := False;
end;

procedure TMainForm.mnuLangChineseClick(Sender: TObject);
begin
  SwitchLanguage('zh');
  mnuLangThai.Checked := False;
  mnuLangEnglish.Checked := False;
  mnuLangChinese.Checked := True;
end;

procedure TMainForm.btnLoginClick(Sender: TObject);
begin
  if edtUsername.Text = '' then
    MessageDlg(rsError, rsUsernameRequired, mtError, [mbOK], 0)
  else if edtPassword.Text = '' then
    MessageDlg(rsError, rsPasswordRequired, mtError, [mbOK], 0)
  else
    MessageDlg(rsSuccess, Format(rsWelcome, [edtUsername.Text]), 
      mtInformation, [mbOK], 0);
end;

end.
```

---

## 75.2 Resource Strings

```pascal
unit strings_res;

{$mode objfpc}{$H+}

interface

resourcestring
  // ชื่อแอป
  rsAppName = 'My Application';
  rsMainTitle = 'Main Window';
  rsLoginTitle = 'Please Login';
  
  // Labels
  rsUsername = 'Username:';
  rsPassword = 'Password:';
  rsEmail = 'Email:';
  rsPhone = 'Phone:';
  rsAddress = 'Address:';
  rsFullName = 'Full Name:';
  
  // Buttons
  rsLogin = 'Login';
  rsLogout = 'Logout';
  rsCancel = 'Cancel';
  rsSave = 'Save';
  rsDelete = 'Delete';
  rsNew = 'New';
  rsEdit = 'Edit';
  rsClose = 'Close';
  rsOK = 'OK';
  rsYes = 'Yes';
  rsNo = 'No';
  rsSearch = 'Search';
  rsRefresh = 'Refresh';
  rsPrint = 'Print';
  rsExport = 'Export';
  rsImport = 'Import';
  
  // Menus
  rsMenuFile = '&File';
  rsMenuNew = '&New';
  rsMenuOpen = '&Open...';
  rsMenuSave = '&Save';
  rsMenuSaveAs = 'Save &As...';
  rsMenuExit = 'E&xit';
  rsMenuEdit = '&Edit';
  rsMenuView = '&View';
  rsMenuHelp = '&Help';
  rsMenuLanguage = '&Language';
  rsMenuAbout = '&About';
  
  // Messages
  rsError = 'Error';
  rsWarning = 'Warning';
  rsSuccess = 'Success';
  rsInfo = 'Information';
  rsConfirm = 'Confirm';
  
  rsUsernameRequired = 'Please enter username';
  rsPasswordRequired = 'Please enter password';
  rsEmailInvalid = 'Invalid email address';
  
  rsWelcome = 'Welcome, %s!';
  rsLoginSuccess = 'Login successful';
  rsLoginFailed = 'Login failed. Please try again.';
  
  rsSaveSuccess = 'Data saved successfully';
  rsSaveFailed = 'Failed to save data';
  rsDeleteConfirm = 'Are you sure you want to delete this?';
  rsDeleteSuccess = 'Deleted successfully';
  
  rsStatusReady = 'Ready [%s]';
  rsStatusSaving = 'Saving...';
  rsStatusLoading = 'Loading...';
  rsStatusConnecting = 'Connecting...';
  
  rsRecordCount = '%d records';
  rsPageOf = 'Page %d of %d';
  rsNoRecords = 'No records found';
  
  // Dates/Times
  rsDateFormat = 'MM/DD/YYYY';
  rsTimeFormat = 'HH:MM:SS';
  rsDateTimeFormat = 'MM/DD/YYYY HH:MM:SS';
  
  // Number formats
  rsCurrencyFormat = '$%,.2f';
  rsPercentFormat = '%.1f%%';
  
  // Validation
  rsFieldRequired = '%s is required';
  rsFieldTooShort = '%s must be at least %d characters';
  rsFieldTooLong = '%s cannot exceed %d characters';
  rsValueOutOfRange = '%s must be between %d and %d';

implementation

end.
```

---

## 75.3 .po Translation Files

```text
# locale/th/myapp.po
# Thai translation of My Application
# Author: ผู้แปล ภาษาไทย
# 

msgid ""
msgstr ""
"Project-Id-Version: MyApp 1.0\n"
"PO-Revision-Date: 2024-01-01 00:00+0700\n"
"Last-Translator: ผู้แปล <translator@email.com>\n"
"Language-Team: Thai <th@li.org>\n"
"Language: th\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"

msgid "My Application"
msgstr "แอปพลิเคชันของฉัน"

msgid "Main Window"
msgstr "หน้าต่างหลัก"

msgid "Please Login"
msgstr "กรุณาเข้าสู่ระบบ"

msgid "Username:"
msgstr "ชื่อผู้ใช้:"

msgid "Password:"
msgstr "รหัสผ่าน:"

msgid "Login"
msgstr "เข้าสู่ระบบ"

msgid "Cancel"
msgstr "ยกเลิก"

msgid "Error"
msgstr "ข้อผิดพลาด"

msgid "Warning"
msgstr "คำเตือน"

msgid "Success"
msgstr "สำเร็จ"

msgid "Please enter username"
msgstr "กรุณากรอกชื่อผู้ใช้"

msgid "Please enter password"
msgstr "กรุณากรอกรหัสผ่าน"

msgid "Welcome, %s!"
msgstr "ยินดีต้อนรับ, %s!"

msgid "Ready [%s]"
msgstr "พร้อมใช้งาน [%s]"

msgid "&File"
msgstr "&ไฟล์"

msgid "&New"
msgstr "&ใหม่"

msgid "&Open..."
msgstr "&เปิด..."

msgid "&Save"
msgstr "&บันทึก"

msgid "E&xit"
msgstr "อ&อก"

msgid "&Language"
msgstr "&ภาษา"
```

```text
# locale/zh/myapp.po  
# Chinese (Simplified) translation
msgid ""
msgstr ""
"Language: zh\n"
"Content-Type: text/plain; charset=UTF-8\n"

msgid "My Application"
msgstr "我的应用程序"

msgid "Please Login"
msgstr "请登录"

msgid "Username:"
msgstr "用户名："

msgid "Password:"
msgstr "密码："

msgid "Login"
msgstr "登录"

msgid "Cancel"
msgstr "取消"
```

---

## 75.4 Translation Manager

```pascal
unit translation_manager;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Translations;

type
  TLanguageInfo = record
    Code: string;      // ISO 639-1: 'en', 'th', 'zh'
    Name: string;      // Display name
    NativeName: string; // ชื่อในภาษานั้น
    RTL: Boolean;      // Right-to-Left
    POFile: string;    // Path to .po file
    FlagImage: string; // Path to flag image
  end;

  TTranslationManager = class
  private
    FLanguages: array of TLanguageInfo;
    FLanguageCount: Integer;
    FCurrentLang: string;
    FLocalePath: string;
    FAppName: string;
    
    class var FInstance: TTranslationManager;
    
    function FindLanguage(const ACode: string): Integer;
    function GetPOFilePath(const ALangCode: string): string;
    
  public
    constructor Create(const AAppName, ALocalePath: string);
    destructor Destroy; override;
    
    class function Instance: TTranslationManager;
    
    procedure RegisterLanguage(const ACode, AName, ANativeName: string;
      ARTL: Boolean = False);
    procedure LoadBuiltinLanguages;
    
    function SwitchTo(const ALangCode: string): Boolean;
    function GetCurrentLang: string;
    function IsRTL: Boolean;
    
    function GetAvailableLanguages: TStringList;
    function GetLanguageInfo(const ACode: string): TLanguageInfo;
    
    // Pluralization
    function Plural(const ASingular, APlural: string; ACount: Integer): string;
    
    // Number/Date formatting
    function FormatNumber(AValue: Double; ADecimals: Integer = 2): string;
    function FormatDate(ADate: TDateTime): string;
    function FormatCurrency(AAmount: Double; const ACurrencyCode: string = ''): string;
    
    property CurrentLang: string read FCurrentLang;
    property LocalePath: string read FLocalePath;
  end;

// Convenience function
function T(const AKey: string): string;

implementation

class function TTranslationManager.Instance: TTranslationManager;
begin
  if FInstance = nil then
    FInstance := TTranslationManager.Create('myapp', 'locale');
  Result := FInstance;
end;

constructor TTranslationManager.Create(const AAppName, ALocalePath: string);
begin
  inherited Create;
  FAppName := AAppName;
  FLocalePath := ALocalePath;
  FLanguageCount := 0;
  SetLength(FLanguages, 20);
  FCurrentLang := 'en';
  
  LoadBuiltinLanguages;
end;

destructor TTranslationManager.Destroy;
begin
  inherited Destroy;
end;

procedure TTranslationManager.LoadBuiltinLanguages;
begin
  RegisterLanguage('en', 'English', 'English');
  RegisterLanguage('th', 'Thai', 'ภาษาไทย');
  RegisterLanguage('zh', 'Chinese (Simplified)', '简体中文');
  RegisterLanguage('zh_TW', 'Chinese (Traditional)', '繁體中文');
  RegisterLanguage('ja', 'Japanese', '日本語');
  RegisterLanguage('ko', 'Korean', '한국어');
  RegisterLanguage('ar', 'Arabic', 'العربية', True);   // RTL
  RegisterLanguage('he', 'Hebrew', 'עברית', True);     // RTL
  RegisterLanguage('fr', 'French', 'Français');
  RegisterLanguage('de', 'German', 'Deutsch');
  RegisterLanguage('es', 'Spanish', 'Español');
  RegisterLanguage('pt', 'Portuguese', 'Português');
end;

procedure TTranslationManager.RegisterLanguage(const ACode, AName, ANativeName: string;
  ARTL: Boolean);
begin
  if FLanguageCount >= Length(FLanguages) then
    SetLength(FLanguages, Length(FLanguages) + 10);
    
  FLanguages[FLanguageCount].Code := ACode;
  FLanguages[FLanguageCount].Name := AName;
  FLanguages[FLanguageCount].NativeName := ANativeName;
  FLanguages[FLanguageCount].RTL := ARTL;
  FLanguages[FLanguageCount].POFile := GetPOFilePath(ACode);
  Inc(FLanguageCount);
end;

function TTranslationManager.FindLanguage(const ACode: string): Integer;
var
  i: Integer;
begin
  Result := -1;
  for i := 0 to FLanguageCount - 1 do
    if FLanguages[i].Code = ACode then
    begin
      Result := i;
      Exit;
    end;
end;

function TTranslationManager.GetPOFilePath(const ALangCode: string): string;
begin
  Result := FLocalePath + PathDelim + ALangCode + PathDelim + FAppName + '.po';
end;

function TTranslationManager.SwitchTo(const ALangCode: string): Boolean;
var
  POFile: string;
  LangIdx: Integer;
begin
  Result := False;
  
  LangIdx := FindLanguage(ALangCode);
  if LangIdx < 0 then
  begin
    WriteLn('Language not found: ', ALangCode);
    Exit;
  end;
  
  POFile := FLanguages[LangIdx].POFile;
  
  if (ALangCode = 'en') or not FileExists(POFile) then
  begin
    // English is default, no file needed
    FCurrentLang := ALangCode;
    SetDefaultLang(ALangCode);
    Result := True;
    WriteLn('Language switched to: ', ALangCode);
    Exit;
  end;
  
  try
    SetDefaultLang(ALangCode, POFile);
    FCurrentLang := ALangCode;
    Result := True;
    WriteLn('Language switched to: ', ALangCode, ' (', POFile, ')');
  except
    on E: Exception do
      WriteLn('Error switching language: ', E.Message);
  end;
end;

function TTranslationManager.IsRTL: Boolean;
var
  Idx: Integer;
begin
  Idx := FindLanguage(FCurrentLang);
  if Idx >= 0 then
    Result := FLanguages[Idx].RTL
  else
    Result := False;
end;

function TTranslationManager.GetAvailableLanguages: TStringList;
var
  i: Integer;
begin
  Result := TStringList.Create;
  for i := 0 to FLanguageCount - 1 do
    Result.AddPair(FLanguages[i].Code, FLanguages[i].NativeName);
end;

function TTranslationManager.Plural(const ASingular, APlural: string; ACount: Integer): string;
begin
  // Thai doesn't have plural forms typically
  if (FCurrentLang = 'th') or (FCurrentLang = 'ja') or (FCurrentLang = 'zh') then
    Result := ASingular
  else if ACount = 1 then
    Result := ASingular
  else
    Result := APlural;
end;

function TTranslationManager.FormatNumber(AValue: Double; ADecimals: Integer): string;
begin
  case FCurrentLang of
    'de', 'fr':  // European: 1.234,56
      Result := FormatFloat(Format('#,##0.%s', [StringOfChar('0', ADecimals)]), 
        AValue);
    else  // Default: 1,234.56
      Result := FormatFloat(Format('#,##0.%s', [StringOfChar('0', ADecimals)]), 
        AValue);
  end;
end;

function TTranslationManager.FormatDate(ADate: TDateTime): string;
begin
  case FCurrentLang of
    'th':  Result := FormatDateTime('dd/mm/yyyy', ADate);
    'ja':  Result := FormatDateTime('yyyy/mm/dd', ADate);
    'zh':  Result := FormatDateTime('yyyy-mm-dd', ADate);
    'en':  Result := FormatDateTime('mm/dd/yyyy', ADate);
    else   Result := FormatDateTime('dd/mm/yyyy', ADate);
  end;
end;

function TTranslationManager.FormatCurrency(AAmount: Double; 
  const ACurrencyCode: string): string;
var
  Code: string;
begin
  Code := ACurrencyCode;
  if Code = '' then
    case FCurrentLang of
      'th': Code := 'THB';
      'en': Code := 'USD';
      'ja': Code := 'JPY';
      'zh': Code := 'CNY';
      else Code := 'USD';
    end;
    
  case Code of
    'THB': Result := Format('฿%,.2f', [AAmount]);
    'USD': Result := Format('$%,.2f', [AAmount]);
    'EUR': Result := Format('€%,.2f', [AAmount]);
    'JPY': Result := Format('¥%,.0f', [AAmount]);
    'CNY': Result := Format('¥%,.2f', [AAmount]);
    else   Result := Format('%s %,.2f', [Code, AAmount]);
  end;
end;

function TTranslationManager.GetLanguageInfo(const ACode: string): TLanguageInfo;
var
  Idx: Integer;
begin
  Idx := FindLanguage(ACode);
  if Idx >= 0 then
    Result := FLanguages[Idx]
  else
  begin
    FillChar(Result, SizeOf(Result), 0);
    Result.Code := ACode;
    Result.Name := ACode;
  end;
end;

function T(const AKey: string): string;
begin
  // Wrapper สำหรับ translation
  Result := AKey;  // Default: return key as-is
end;

end.
```

---

## 75.5 ตัวอย่างการใช้งาน

```pascal
program i18n_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  translation_manager, strings_res;

var
  TM: TTranslationManager;
  Languages: TStringList;
  i: Integer;
begin
  WriteLn('=== Internationalization Demo ===');
  
  TM := TTranslationManager.Instance;
  
  // แสดงภาษาที่รองรับ
  Languages := TM.GetAvailableLanguages;
  try
    WriteLn('Supported languages:');
    for i := 0 to Languages.Count - 1 do
      WriteLn(Format('  %s: %s', [Languages.Names[i], Languages.ValueFromIndex[i]]));
  finally
    Languages.Free;
  end;
  
  WriteLn('');
  
  // ทดสอบแต่ละภาษา
  var TestLangs: array of string = ('en', 'th', 'zh', 'ja', 'de');
  var TestAmount := 1234567.89;
  var TestDate := EncodeDate(2024, 1, 15);
  
  for var Lang in TestLangs do
  begin
    if TM.SwitchTo(Lang) then
    begin
      WriteLn('--- ', TM.GetLanguageInfo(Lang).NativeName, ' ---');
      WriteLn('RTL: ', BoolToStr(TM.IsRTL, True));
      WriteLn('Currency: ', TM.FormatCurrency(TestAmount));
      WriteLn('Number: ', TM.FormatNumber(TestAmount));
      WriteLn('Date: ', TM.FormatDate(TestDate));
      WriteLn('Plural(1 item): ', TM.Plural('item', 'items', 1));
      WriteLn('Plural(5 items): ', TM.Plural('item', 'items', 5));
      WriteLn('');
    end;
  end;
  
  // ตรวจสอบ resource strings
  TM.SwitchTo('th');
  WriteLn('=== Thai Resource Strings ===');
  WriteLn('Username: ', rsUsername);
  WriteLn('Password: ', rsPassword);
  WriteLn('Login: ', rsLogin);
  WriteLn('Error: ', rsError);
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Resource Strings** - การประกาศข้อความที่แปลได้
2. **.po Files** - รูปแบบไฟล์แปลภาษา gettext
3. **LCLTranslator** - โมดูล Lazarus สำหรับแปลภาษา
4. **Translation Manager** - จัดการหลายภาษา
5. **Number/Date Formatting** - จัดรูปแบบตามภาษา
6. **RTL Support** - รองรับภาษาที่เขียนขวาไปซ้าย

i18n ที่ดีทำให้แอปพลิเคชันใช้งานได้ทั่วโลกและสร้างประสบการณ์ที่ดีให้กับผู้ใช้ในทุกภาษา
