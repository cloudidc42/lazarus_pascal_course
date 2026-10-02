# ตอนที่ 76: Accessibility ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการสร้างแอปพลิเคชันที่เข้าถึงได้สำหรับผู้พิการ รองรับ screen readers, keyboard navigation และ high contrast mode

---

## 76.1 Keyboard Navigation

```pascal
unit accessible_form;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, Buttons,
  ActnList, Menus, ComCtrls, LCLType;

type
  TAccessibleForm = class(TForm)
  private
    FTabOrder: TList;
    FCurrentFocusIdx: Integer;
    
    procedure SetupTabOrder;
    procedure SetupKeyboardShortcuts;
    procedure SetupAccessibilityProperties;
    
    procedure HandleKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure HandleTabKey(AForward: Boolean);
    procedure MoveFocusTo(AControl: TWinControl);
    procedure ShowFocusIndicator(AControl: TControl);
    
    procedure AnnounceToScreenReader(const AMessage: string);
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure RegisterFocusable(AControl: TWinControl; ATabIdx: Integer);
    function GetFocusedControl: TControl;
  end;

implementation

constructor TAccessibleForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FTabOrder := TList.Create;
  FCurrentFocusIdx := -1;
  
  KeyPreview := True;
  OnKeyDown := @HandleKeyDown;
  
  SetupAccessibilityProperties;
end;

destructor TAccessibleForm.Destroy;
begin
  FTabOrder.Free;
  inherited Destroy;
end;

procedure TAccessibleForm.SetupAccessibilityProperties;
begin
  // ตั้งค่า accessible name สำหรับ form
  // Hint จะถูกอ่านโดย screen reader
  Hint := 'Main Application Window';
  ShowHint := True;
end;

procedure TAccessibleForm.RegisterFocusable(AControl: TWinControl; ATabIdx: Integer);
begin
  AControl.TabOrder := ATabIdx;
  AControl.TabStop := True;
  FTabOrder.Add(AControl);
end;

procedure TAccessibleForm.HandleKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  case Key of
    VK_TAB:
    begin
      if ssShift in Shift then
        HandleTabKey(False)   // Shift+Tab: backward
      else
        HandleTabKey(True);    // Tab: forward
      Key := 0;  // Prevent default handling
    end;
    
    VK_F1:
      ShowHelpContext(HelpContext);
      
    VK_ESCAPE:
      Close;
  end;
end;

procedure TAccessibleForm.HandleTabKey(AForward: Boolean);
begin
  if FTabOrder.Count = 0 then Exit;
  
  if AForward then
  begin
    Inc(FCurrentFocusIdx);
    if FCurrentFocusIdx >= FTabOrder.Count then
      FCurrentFocusIdx := 0;
  end
  else
  begin
    Dec(FCurrentFocusIdx);
    if FCurrentFocusIdx < 0 then
      FCurrentFocusIdx := FTabOrder.Count - 1;
  end;
  
  MoveFocusTo(TWinControl(FTabOrder[FCurrentFocusIdx]));
end;

procedure TAccessibleForm.MoveFocusTo(AControl: TWinControl);
begin
  AControl.SetFocus;
  ShowFocusIndicator(AControl);
  AnnounceToScreenReader(AControl.Caption + ' ' + AControl.ClassName);
end;

procedure TAccessibleForm.ShowFocusIndicator(AControl: TControl);
begin
  // เพิ่ม visual focus indicator
  // ในแอปจริงอาจวาด rectangle รอบๆ control
  WriteLn('Focus: ', AControl.Name, ' (', AControl.Caption, ')');
end;

procedure TAccessibleForm.AnnounceToScreenReader(const AMessage: string);
begin
  // Platform-specific screen reader announcement
  {$IFDEF WINDOWS}
  // ใช้ UIA (UI Automation) หรือ MSAA
  // NotifyWinEvent(EVENT_SYSTEM_ALERT, Handle, OBJID_CLIENT, CHILDID_SELF);
  WriteLn('[SR] ', AMessage);
  {$ENDIF}
  
  {$IFDEF UNIX}
  // ใช้ AT-SPI หรือ Orca
  WriteLn('[SR] ', AMessage);
  {$ENDIF}
end;

end.
```

---

## 76.2 Accessible Components

```pascal
unit accessible_controls;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Controls, StdCtrls, ExtCtrls, Graphics,
  LCLType, LMessages;

// Button ที่ Accessible
type
  TAccessibleButton = class(TButton)
  private
    FAccessibleName: string;
    FAccessibleDescription: string;
    FRole: string;
    FHotKey: TShortCut;
    
    procedure HandleMouseEnter(Sender: TObject);
    procedure HandleMouseLeave(Sender: TObject);
    procedure HandleFocusIn(Sender: TObject);
    procedure HandleFocusOut(Sender: TObject);
    
  protected
    procedure Paint; override;
    procedure KeyDown(var Key: Word; Shift: TShiftState); override;
    
  public
    constructor Create(AOwner: TComponent); override;
    
    property AccessibleName: string read FAccessibleName write FAccessibleName;
    property AccessibleDescription: string read FAccessibleDescription 
      write FAccessibleDescription;
    property Role: string read FRole write FRole;
    property HotKey: TShortCut read FHotKey write FHotKey;
  end;

// Label ที่ Accessible (รองรับ for control)
type
  TAccessibleLabel = class(TLabel)
  private
    FForControl: TControl;
    
  public
    property ForControl: TControl read FForControl write FForControl;
  end;

// Panel สำหรับ grouping accessible controls
type
  TAccessiblePanel = class(TPanel)
  private
    FGroupName: string;
    FIsLandmark: Boolean;
    
  public
    constructor Create(AOwner: TComponent); override;
    
    property GroupName: string read FGroupName write FGroupName;
    property IsLandmark: Boolean read FIsLandmark write FIsLandmark;
  end;

// ProgressBar ที่ประกาศ progress ให้ screen reader
type
  TAccessibleProgressBar = class(TProgressBar)
  private
    FLastAnnounced: Integer;
    FAnnounceInterval: Integer;  // % interval to announce
    
  protected
    procedure Changed; override;
    
  public
    constructor Create(AOwner: TComponent); override;
    
    property AnnounceInterval: Integer read FAnnounceInterval 
      write FAnnounceInterval;
  end;

// Edit ที่มี accessible validation
type
  TAccessibleEdit = class(TEdit)
  private
    FValidationMessage: string;
    FIsValid: Boolean;
    FRequiredField: Boolean;
    
    procedure Validate;
    procedure ShowError(const AMessage: string);
    procedure ClearError;
    
  protected
    procedure Change; override;
    procedure Exit; override;
    
  public
    constructor Create(AOwner: TComponent); override;
    
    property IsValid: Boolean read FIsValid;
    property RequiredField: Boolean read FRequiredField write FRequiredField;
    property ValidationMessage: string read FValidationMessage;
  end;

implementation

{ TAccessibleButton }

constructor TAccessibleButton.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FRole := 'button';
  
  OnMouseEnter := @HandleMouseEnter;
  OnMouseLeave := @HandleMouseLeave;
  OnEnter := @HandleFocusIn;
  OnExit := @HandleFocusOut;
  
  // ทำให้เห็นชัดเมื่อ focus
  TabStop := True;
end;

procedure TAccessibleButton.Paint;
var
  R: TRect;
begin
  inherited Paint;
  
  // วาด focus indicator ที่ชัดเจน
  if Focused then
  begin
    R := ClientRect;
    Canvas.Pen.Color := clBlack;
    Canvas.Pen.Width := 2;
    Canvas.Pen.Style := psDot;
    Canvas.Brush.Style := bsClear;
    InflateRect(R, -3, -3);
    Canvas.Rectangle(R);
  end;
end;

procedure TAccessibleButton.HandleMouseEnter(Sender: TObject);
begin
  // แสดง tooltip ถ้ามี
  if Hint <> '' then
    Application.HintPause := 0;
end;

procedure TAccessibleButton.HandleMouseLeave(Sender: TObject);
begin
  Application.HintPause := Application.HintPause;
end;

procedure TAccessibleButton.HandleFocusIn(Sender: TObject);
begin
  // ประกาศให้ screen reader
  var Announcement := FAccessibleName;
  if Announcement = '' then Announcement := Caption;
  Announcement := Announcement + ', button';
  if not Enabled then Announcement := Announcement + ', dimmed';
  
  WriteLn('[Accessibility] Focus: ', Announcement);
end;

procedure TAccessibleButton.HandleFocusOut(Sender: TObject);
begin
  // ไม่ต้องทำอะไรพิเศษ
end;

procedure TAccessibleButton.KeyDown(var Key: Word; Shift: TShiftState);
begin
  // Space และ Enter ควร activate button
  if Key in [VK_SPACE, VK_RETURN] then
  begin
    Click;
    Key := 0;
  end
  else
    inherited KeyDown(Key, Shift);
end;

{ TAccessibleProgressBar }

constructor TAccessibleProgressBar.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FLastAnnounced := -1;
  FAnnounceInterval := 10;  // ประกาศทุก 10%
end;

procedure TAccessibleProgressBar.Changed;
var
  Percent: Integer;
begin
  inherited Changed;
  
  if Max > Min then
    Percent := Round((Position - Min) * 100 / (Max - Min))
  else
    Percent := 0;
    
  // ประกาศเมื่อถึง interval
  var ShouldAnnounce := 
    (FLastAnnounced < 0) or
    (Percent >= 100) or
    ((Percent div FAnnounceInterval) > (FLastAnnounced div FAnnounceInterval));
    
  if ShouldAnnounce then
  begin
    FLastAnnounced := Percent;
    WriteLn(Format('[Accessibility] Progress: %d%%', [Percent]));
  end;
end;

{ TAccessibleEdit }

constructor TAccessibleEdit.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FIsValid := True;
  FRequiredField := False;
end;

procedure TAccessibleEdit.Change;
begin
  inherited Change;
  ClearError;  // Clear error while typing
end;

procedure TAccessibleEdit.Exit;
begin
  inherited Exit;
  Validate;
end;

procedure TAccessibleEdit.Validate;
begin
  FIsValid := True;
  FValidationMessage := '';
  
  if FRequiredField and (Trim(Text) = '') then
  begin
    FIsValid := False;
    FValidationMessage := 'This field is required';
    ShowError(FValidationMessage);
  end;
end;

procedure TAccessibleEdit.ShowError(const AMessage: string);
begin
  Color := $00CCCCFF;  // Light red background
  Hint := AMessage;
  ShowHint := True;
  
  WriteLn('[Accessibility] Validation Error: ', AMessage);
end;

procedure TAccessibleEdit.ClearError;
begin
  if not FIsValid then
  begin
    Color := clWindow;  // Reset to default
    FIsValid := True;
    FValidationMessage := '';
  end;
end;

{ TAccessiblePanel }

constructor TAccessiblePanel.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FIsLandmark := False;
end;

end.
```

---

## 76.3 High Contrast Mode

```pascal
unit high_contrast;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics,
  Themes, LCLIntf;

type
  TContrastTheme = (ctSystem, ctHighContrast, ctHighContrastDark, ctLargeText);

  TColorScheme = record
    Background: TColor;
    Foreground: TColor;
    Accent: TColor;
    ButtonBG: TColor;
    ButtonText: TColor;
    DisabledBG: TColor;
    DisabledText: TColor;
    FocusColor: TColor;
    ErrorColor: TColor;
    WarningColor: TColor;
    SuccessColor: TColor;
    BorderColor: TColor;
    LinkColor: TColor;
    SelectionBG: TColor;
    SelectionText: TColor;
  end;

  TAccessibilityManager = class
  private
    FCurrentTheme: TContrastTheme;
    FColorScheme: TColorScheme;
    FFontSize: Integer;
    FHighContrastEnabled: Boolean;
    
    class var FInstance: TAccessibilityManager;
    
    procedure LoadColorScheme;
    procedure ApplyToForm(AForm: TForm);
    procedure ApplyToControl(AControl: TControl);
    function DetectSystemHighContrast: Boolean;
    
  public
    constructor Create;
    
    class function Instance: TAccessibilityManager;
    
    procedure SetTheme(ATheme: TContrastTheme);
    procedure SetFontSize(ASize: Integer);
    procedure ApplyToApplication;
    
    function IsHighContrast: Boolean;
    function GetColor(ARole: string): TColor;
    
    property CurrentTheme: TContrastTheme read FCurrentTheme;
    property ColorScheme: TColorScheme read FColorScheme;
    property FontSize: Integer read FFontSize;
  end;

// Global color schemes
function DefaultColorScheme: TColorScheme;
function HighContrastLightScheme: TColorScheme;
function HighContrastDarkScheme: TColorScheme;

implementation

function DefaultColorScheme: TColorScheme;
begin
  Result.Background := clWindow;
  Result.Foreground := clWindowText;
  Result.Accent := $00E87722;     // Orange
  Result.ButtonBG := clBtnFace;
  Result.ButtonText := clBtnText;
  Result.DisabledBG := clInactiveBorder;
  Result.DisabledText := clGrayText;
  Result.FocusColor := clHighlight;
  Result.ErrorColor := clRed;
  Result.WarningColor := $0000AAFF;
  Result.SuccessColor := clGreen;
  Result.BorderColor := clActiveBorder;
  Result.LinkColor := clBlue;
  Result.SelectionBG := clHighlight;
  Result.SelectionText := clHighlightText;
end;

function HighContrastLightScheme: TColorScheme;
begin
  Result.Background := clWhite;
  Result.Foreground := clBlack;
  Result.Accent := clBlack;
  Result.ButtonBG := clWhite;
  Result.ButtonText := clBlack;
  Result.DisabledBG := $00AAAAAA;
  Result.DisabledText := $00666666;
  Result.FocusColor := clBlue;
  Result.ErrorColor := $00000088;  // Dark blue (accessible)
  Result.WarningColor := $00000066;
  Result.SuccessColor := $00006600;
  Result.BorderColor := clBlack;
  Result.LinkColor := clBlue;
  Result.SelectionBG := clBlack;
  Result.SelectionText := clWhite;
end;

function HighContrastDarkScheme: TColorScheme;
begin
  Result.Background := clBlack;
  Result.Foreground := clWhite;
  Result.Accent := clYellow;
  Result.ButtonBG := clBlack;
  Result.ButtonText := clWhite;
  Result.DisabledBG := $00333333;
  Result.DisabledText := $00666666;
  Result.FocusColor := clYellow;
  Result.ErrorColor := clRed;
  Result.WarningColor := clYellow;
  Result.SuccessColor := $0000FF00;  // Bright green
  Result.BorderColor := clWhite;
  Result.LinkColor := clAqua;
  Result.SelectionBG := clWhite;
  Result.SelectionText := clBlack;
end;

{ TAccessibilityManager }

class function TAccessibilityManager.Instance: TAccessibilityManager;
begin
  if FInstance = nil then
    FInstance := TAccessibilityManager.Create;
  Result := FInstance;
end;

constructor TAccessibilityManager.Create;
begin
  inherited Create;
  FCurrentTheme := ctSystem;
  FFontSize := 10;
  FColorScheme := DefaultColorScheme;
  
  FHighContrastEnabled := DetectSystemHighContrast;
  if FHighContrastEnabled then
    SetTheme(ctHighContrast);
end;

function TAccessibilityManager.DetectSystemHighContrast: Boolean;
begin
  {$IFDEF WINDOWS}
  var HCI: THighContrastA;
  FillChar(HCI, SizeOf(HCI), 0);
  HCI.cbSize := SizeOf(HCI);
  if SystemParametersInfoA(SPI_GETHIGHCONTRAST, SizeOf(HCI), @HCI, 0) then
    Result := (HCI.dwFlags and HCF_HIGHCONTRASTON) <> 0
  else
    Result := False;
  {$ELSE}
  Result := False;
  {$ENDIF}
end;

procedure TAccessibilityManager.LoadColorScheme;
begin
  case FCurrentTheme of
    ctSystem:
      FColorScheme := DefaultColorScheme;
    ctHighContrast:
      FColorScheme := HighContrastLightScheme;
    ctHighContrastDark:
      FColorScheme := HighContrastDarkScheme;
    ctLargeText:
    begin
      FColorScheme := DefaultColorScheme;
      FFontSize := 14;
    end;
  end;
end;

procedure TAccessibilityManager.SetTheme(ATheme: TContrastTheme);
begin
  FCurrentTheme := ATheme;
  LoadColorScheme;
  ApplyToApplication;
end;

procedure TAccessibilityManager.SetFontSize(ASize: Integer);
begin
  FFontSize := ASize;
  ApplyToApplication;
end;

procedure TAccessibilityManager.ApplyToControl(AControl: TControl);
begin
  if AControl is TButton then
  begin
    AControl.Color := FColorScheme.ButtonBG;
    if AControl is TControl then
      TButton(AControl).Font.Color := FColorScheme.ButtonText;
  end
  else if AControl is TEdit then
  begin
    AControl.Color := FColorScheme.Background;
    TEdit(AControl).Font.Color := FColorScheme.Foreground;
  end
  else
  begin
    AControl.Color := FColorScheme.Background;
    if AControl.Font <> nil then
    begin
      AControl.Font.Color := FColorScheme.Foreground;
      AControl.Font.Size := FFontSize;
    end;
  end;
  
  // Apply to children
  if AControl is TWinControl then
  begin
    var WC := TWinControl(AControl);
    for var i := 0 to WC.ControlCount - 1 do
      ApplyToControl(WC.Controls[i]);
  end;
end;

procedure TAccessibilityManager.ApplyToForm(AForm: TForm);
begin
  AForm.Color := FColorScheme.Background;
  AForm.Font.Color := FColorScheme.Foreground;
  AForm.Font.Size := FFontSize;
  
  for var i := 0 to AForm.ControlCount - 1 do
    ApplyToControl(AForm.Controls[i]);
    
  AForm.Refresh;
end;

procedure TAccessibilityManager.ApplyToApplication;
var
  i: Integer;
begin
  // Apply to all forms
  for i := 0 to Screen.FormCount - 1 do
    ApplyToForm(Screen.Forms[i]);
    
  WriteLn('Accessibility theme applied: ', Ord(FCurrentTheme));
end;

function TAccessibilityManager.IsHighContrast: Boolean;
begin
  Result := FCurrentTheme in [ctHighContrast, ctHighContrastDark];
end;

function TAccessibilityManager.GetColor(ARole: string): TColor;
begin
  ARole := LowerCase(ARole);
  
  if ARole = 'background' then Result := FColorScheme.Background
  else if ARole = 'foreground' then Result := FColorScheme.Foreground
  else if ARole = 'accent' then Result := FColorScheme.Accent
  else if ARole = 'focus' then Result := FColorScheme.FocusColor
  else if ARole = 'error' then Result := FColorScheme.ErrorColor
  else if ARole = 'warning' then Result := FColorScheme.WarningColor
  else if ARole = 'success' then Result := FColorScheme.SuccessColor
  else if ARole = 'link' then Result := FColorScheme.LinkColor
  else Result := FColorScheme.Foreground;
end;

end.
```

---

## 76.4 Screen Reader Support

```pascal
unit screen_reader;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TLiveRegionRole = (lrrStatus, lrrAlert, lrrLog, lrrMarquee, lrrTimer);
  TAriaPoliteness = (apPolite, apAssertive, apOff);

  TScreenReaderInterface = class
  private
    FEnabled: Boolean;
    FProvider: string;
    
    procedure InitProvider;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Speak(const AText: string; APoliteness: TAriaPoliteness = apPolite);
    procedure SpeakInterrupt(const AText: string);
    procedure Silence;
    
    procedure AnnounceNavigation(const ALocation: string);
    procedure AnnounceAction(const AAction, ATarget: string);
    procedure AnnounceError(const AMessage: string);
    procedure AnnounceStatus(const AStatus: string);
    
    function IsAvailable: Boolean;
    function GetProviderName: string;
    
    property Enabled: Boolean read FEnabled write FEnabled;
    property Provider: string read FProvider;
  end;

  // Focus tracking สำหรับ screen reader
  TFocusTracker = class
  private
    FFocusHistory: TStringList;
    FMaxHistory: Integer;
    
  public
    constructor Create(AMaxHistory: Integer = 20);
    destructor Destroy; override;
    
    procedure TrackFocus(const AControlName, ARole, AState: string);
    procedure GetFocusPath(APath: TStringList);
    function GetCurrentFocusDescription: string;
  end;

implementation

constructor TScreenReaderInterface.Create;
begin
  inherited Create;
  FEnabled := True;
  InitProvider;
end;

destructor TScreenReaderInterface.Destroy;
begin
  inherited Destroy;
end;

procedure TScreenReaderInterface.InitProvider;
begin
  {$IFDEF WINDOWS}
  // ตรวจสอบ NVDA, JAWS, Narrator
  // ในแอปจริงใช้ Windows UIA หรือ MSAA
  FProvider := 'Windows MSAA';
  {$ELSE}
  {$IFDEF UNIX}
  // ตรวจสอบ Orca, BRLTTY
  FProvider := 'AT-SPI (Linux)';
  {$ELSE}
  FProvider := 'None';
  {$ENDIF}
  {$ENDIF}
end;

procedure TScreenReaderInterface.Speak(const AText: string; APoliteness: TAriaPoliteness);
begin
  if not FEnabled or not IsAvailable then Exit;
  
  case APoliteness of
    apPolite:    WriteLn('[SR-Polite] ', AText);
    apAssertive: WriteLn('[SR-Assert] ', AText);
    apOff:       ;  // Don't speak
  end;
  
  // Platform implementation
  {$IFDEF WINDOWS}
  // NotifyWinEvent or UIA TextNotify
  {$ENDIF}
end;

procedure TScreenReaderInterface.SpeakInterrupt(const AText: string);
begin
  Silence;
  Speak(AText, apAssertive);
end;

procedure TScreenReaderInterface.Silence;
begin
  WriteLn('[SR] Silence');
end;

procedure TScreenReaderInterface.AnnounceNavigation(const ALocation: string);
begin
  Speak(ALocation + ', page', apPolite);
end;

procedure TScreenReaderInterface.AnnounceAction(const AAction, ATarget: string);
begin
  Speak(Format('%s %s', [AAction, ATarget]), apPolite);
end;

procedure TScreenReaderInterface.AnnounceError(const AMessage: string);
begin
  SpeakInterrupt('Error: ' + AMessage);
end;

procedure TScreenReaderInterface.AnnounceStatus(const AStatus: string);
begin
  Speak(AStatus, apPolite);
end;

function TScreenReaderInterface.IsAvailable: Boolean;
begin
  {$IFDEF WINDOWS}
  // ตรวจสอบว่ามี screen reader ทำงานอยู่
  Result := True;  // Simplified
  {$ELSE}
  Result := True;
  {$ENDIF}
end;

function TScreenReaderInterface.GetProviderName: string;
begin
  Result := FProvider;
end;

{ TFocusTracker }

constructor TFocusTracker.Create(AMaxHistory: Integer);
begin
  inherited Create;
  FFocusHistory := TStringList.Create;
  FMaxHistory := AMaxHistory;
end;

destructor TFocusTracker.Destroy;
begin
  FFocusHistory.Free;
  inherited Destroy;
end;

procedure TFocusTracker.TrackFocus(const AControlName, ARole, AState: string);
var
  Entry: string;
begin
  Entry := Format('%s [%s] (%s) @ %s',
    [AControlName, ARole, AState, 
     FormatDateTime('hh:nn:ss', Now)]);
  
  FFocusHistory.Insert(0, Entry);
  
  while FFocusHistory.Count > FMaxHistory do
    FFocusHistory.Delete(FFocusHistory.Count - 1);
end;

procedure TFocusTracker.GetFocusPath(APath: TStringList);
begin
  APath.Assign(FFocusHistory);
end;

function TFocusTracker.GetCurrentFocusDescription: string;
begin
  if FFocusHistory.Count > 0 then
    Result := FFocusHistory[0]
  else
    Result := 'No focus';
end;

end.
```

---

## 76.5 ตัวอย่างฟอร์มที่ Accessible

```pascal
program accessible_app;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, Graphics,
  accessible_controls, high_contrast, screen_reader;

type
  TLoginForm = class(TForm)
  private
    lblTitle: TLabel;
    lblUsername: TLabel;
    edtUsername: TAccessibleEdit;
    lblPassword: TLabel;
    edtPassword: TAccessibleEdit;
    btnLogin: TAccessibleButton;
    btnCancel: TAccessibleButton;
    
    SR: TScreenReaderInterface;
    
    procedure SetupControls;
    procedure SetupAccessibility;
    procedure btnLoginClick(Sender: TObject);
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
  end;

constructor TLoginForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  SR := TScreenReaderInterface.Create;
  
  Caption := 'Login';
  Width := 360;
  Height := 280;
  Position := poScreenCenter;
  KeyPreview := True;
  
  SetupControls;
  SetupAccessibility;
  
  // Apply accessibility theme
  TAccessibilityManager.Instance.ApplyToForm(Self);
end;

destructor TLoginForm.Destroy;
begin
  SR.Free;
  inherited Destroy;
end;

procedure TLoginForm.SetupControls;
begin
  lblTitle := TLabel.Create(Self);
  with lblTitle do
  begin
    Parent := Self;
    Caption := 'System Login';
    Font.Size := 16;
    Font.Style := [fsBold];
    Left := 20; Top := 20;
    Width := 320;
    Alignment := taCenter;
  end;
  
  lblUsername := TLabel.Create(Self);
  with lblUsername do
  begin
    Parent := Self;
    Caption := 'Username: *';
    Left := 20; Top := 80;
  end;
  
  edtUsername := TAccessibleEdit.Create(Self);
  with edtUsername do
  begin
    Parent := Self;
    Left := 20; Top := 100;
    Width := 320; Height := 30;
    RequiredField := True;
    Hint := 'Enter your username. This field is required.';
    ShowHint := True;
    TabOrder := 0;
  end;
  
  lblPassword := TLabel.Create(Self);
  with lblPassword do
  begin
    Parent := Self;
    Caption := 'Password: *';
    Left := 20; Top := 145;
  end;
  
  edtPassword := TAccessibleEdit.Create(Self);
  with edtPassword do
  begin
    Parent := Self;
    Left := 20; Top := 165;
    Width := 320; Height := 30;
    PasswordChar := '*';
    RequiredField := True;
    Hint := 'Enter your password. This field is required.';
    ShowHint := True;
    TabOrder := 1;
  end;
  
  btnLogin := TAccessibleButton.Create(Self);
  with btnLogin do
  begin
    Parent := Self;
    Caption := '&Login';
    Left := 20; Top := 215;
    Width := 150; Height := 40;
    AccessibleName := 'Login Button';
    AccessibleDescription := 'Click to login with your credentials';
    Font.Size := 11;
    TabOrder := 2;
    OnClick := @btnLoginClick;
  end;
  
  btnCancel := TAccessibleButton.Create(Self);
  with btnCancel do
  begin
    Parent := Self;
    Caption := '&Cancel';
    Left := 190; Top := 215;
    Width := 150; Height := 40;
    AccessibleName := 'Cancel Button';
    Font.Size := 11;
    TabOrder := 3;
    OnClick := @ModalResult := mrCancel;
  end;
end;

procedure TLoginForm.SetupAccessibility;
begin
  // Set focus descriptions
  edtUsername.AccessibilityDescription := 
    'Username field. Required. Press Tab to move to next field.';
  edtPassword.AccessibilityDescription := 
    'Password field. Required. Characters are hidden.';
  
  // Announce form loaded
  SR.AnnounceNavigation('Login dialog opened');
  
  // Focus first field
  edtUsername.SetFocus;
end;

procedure TLoginForm.btnLoginClick(Sender: TObject);
begin
  // Validate
  if not edtUsername.IsValid then
  begin
    SR.AnnounceError('Username is required');
    edtUsername.SetFocus;
    Exit;
  end;
  
  if not edtPassword.IsValid then
  begin
    SR.AnnounceError('Password is required');
    edtPassword.SetFocus;
    Exit;
  end;
  
  SR.AnnounceStatus('Logging in...');
  
  // Process login...
  WriteLn('Login attempt: ', edtUsername.Text);
end;

var
  App: TApplication;
  Form: TLoginForm;
begin
  Application.Initialize;
  
  // Enable high contrast if needed
  if TAccessibilityManager.Instance.IsHighContrast then
    WriteLn('High contrast mode active');
  
  Form := TLoginForm.Create(nil);
  try
    Form.ShowModal;
  finally
    Form.Free;
  end;
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Keyboard Navigation** - Tab order, shortcut keys
2. **Accessible Components** - Custom controls ที่ accessible
3. **High Contrast Mode** - Theme ที่เห็นชัดสำหรับผู้มีปัญหาการมองเห็น
4. **Screen Reader** - ประกาศ UI changes
5. **Focus Management** - Track และจัดการ focus
6. **ARIA Roles** - กำหนด role ของ components

Accessibility ทำให้แอปพลิเคชันใช้ได้สำหรับทุกคน รวมถึงผู้ที่มีความพิการ ซึ่งเป็นสิ่งสำคัญทั้งด้านจริยธรรมและกฎหมายในหลายประเทศ
