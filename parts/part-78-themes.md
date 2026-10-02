# ตอนที่ 78: Themes และ Styling ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการสร้างระบบ theming สำหรับ Lazarus/Pascal รวมถึง dark mode, skin switching และ dynamic styling

---

## 78.1 Theme System

```pascal
unit theme_system;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, Controls, Forms, StdCtrls,
  Buttons, ComCtrls, ExtCtrls;

type
  TThemeType = (ttLight, ttDark, ttHighContrast, ttSolarized, ttNord, ttCustom);

  TThemeColors = record
    // Backgrounds
    BgPrimary: TColor;
    BgSecondary: TColor;
    BgTertiary: TColor;
    BgCard: TColor;
    
    // Text
    TextPrimary: TColor;
    TextSecondary: TColor;
    TextDisabled: TColor;
    TextInverse: TColor;
    
    // Accent
    AccentPrimary: TColor;
    AccentSecondary: TColor;
    AccentDanger: TColor;
    AccentWarning: TColor;
    AccentSuccess: TColor;
    
    // Borders
    BorderLight: TColor;
    BorderMedium: TColor;
    BorderDark: TColor;
    
    // Inputs
    InputBg: TColor;
    InputBorder: TColor;
    InputFocusBorder: TColor;
    InputText: TColor;
    
    // Buttons
    BtnPrimaryBg: TColor;
    BtnPrimaryText: TColor;
    BtnSecondaryBg: TColor;
    BtnSecondaryText: TColor;
    
    // Selection
    SelectionBg: TColor;
    SelectionText: TColor;
    
    // Scrollbar
    ScrollbarThumb: TColor;
    ScrollbarTrack: TColor;
  end;

  TThemeFonts = record
    FontFamily: string;
    FontSizeSmall: Integer;
    FontSizeNormal: Integer;
    FontSizeLarge: Integer;
    FontSizeXLarge: Integer;
    FontSizeTitle: Integer;
  end;

  TThemeSpacing = record
    Tiny: Integer;    // 4px
    Small: Integer;   // 8px
    Medium: Integer;  // 16px
    Large: Integer;   // 24px
    XLarge: Integer;  // 32px
  end;

  TTheme = class
  private
    FName: string;
    FType: TThemeType;
    FColors: TThemeColors;
    FFonts: TThemeFonts;
    FSpacing: TThemeSpacing;
    
  public
    constructor Create(const AName: string; AType: TThemeType);
    
    procedure LoadDefaults;
    procedure Assign(ASource: TTheme);
    
    property Name: string read FName write FName;
    property ThemeType: TThemeType read FType write FType;
    property Colors: TThemeColors read FColors write FColors;
    property Fonts: TThemeFonts read FFonts write FFonts;
    property Spacing: TThemeSpacing read FSpacing write FSpacing;
  end;

  TThemeChangeEvent = procedure(ATheme: TTheme) of object;

  TThemeManager = class
  private
    FThemes: TList;
    FCurrentTheme: TTheme;
    FOnThemeChange: TThemeChangeEvent;
    
    class var FInstance: TThemeManager;
    
    procedure BuildDefaultThemes;
    procedure ApplyThemeToForm(AForm: TForm; ATheme: TTheme);
    procedure ApplyThemeToControl(AControl: TControl; ATheme: TTheme);
    
  public
    constructor Create;
    destructor Destroy; override;
    
    class function Instance: TThemeManager;
    
    procedure RegisterTheme(ATheme: TTheme);
    function GetTheme(const AName: string): TTheme;
    function GetBuiltinTheme(AType: TThemeType): TTheme;
    
    procedure ApplyTheme(const AName: string); overload;
    procedure ApplyTheme(ATheme: TTheme); overload;
    procedure ApplyToAllForms;
    
    function ListThemes: TStringList;
    
    property CurrentTheme: TTheme read FCurrentTheme;
    property OnThemeChange: TThemeChangeEvent read FOnThemeChange write FOnThemeChange;
  end;

// Built-in theme factories
function LightTheme: TTheme;
function DarkTheme: TTheme;
function HighContrastTheme: TTheme;
function SolarizedTheme: TTheme;
function NordTheme: TTheme;

implementation

{ TTheme }

constructor TTheme.Create(const AName: string; AType: TThemeType);
begin
  inherited Create;
  FName := AName;
  FType := AType;
  LoadDefaults;
end;

procedure TTheme.LoadDefaults;
begin
  // Default fonts
  FFonts.FontFamily := 'Segoe UI';
  FFonts.FontSizeSmall := 9;
  FFonts.FontSizeNormal := 10;
  FFonts.FontSizeLarge := 12;
  FFonts.FontSizeXLarge := 14;
  FFonts.FontSizeTitle := 18;
  
  // Default spacing
  FSpacing.Tiny := 4;
  FSpacing.Small := 8;
  FSpacing.Medium := 16;
  FSpacing.Large := 24;
  FSpacing.XLarge := 32;
end;

procedure TTheme.Assign(ASource: TTheme);
begin
  FName := ASource.FName;
  FType := ASource.FType;
  FColors := ASource.FColors;
  FFonts := ASource.FFonts;
  FSpacing := ASource.FSpacing;
end;

function LightTheme: TTheme;
begin
  Result := TTheme.Create('Light', ttLight);
  with Result.Colors do
  begin
    BgPrimary := $00FFFFFF;
    BgSecondary := $00F5F5F5;
    BgTertiary := $00EBEBEB;
    BgCard := $00FFFFFF;
    
    TextPrimary := $00212121;
    TextSecondary := $00757575;
    TextDisabled := $00BDBDBD;
    TextInverse := $00FFFFFF;
    
    AccentPrimary := $00E87722;    // Orange
    AccentSecondary := $00F5A623;
    AccentDanger := $00D32F2F;
    AccentWarning := $00F57C00;
    AccentSuccess := $00388E3C;
    
    BorderLight := $00E0E0E0;
    BorderMedium := $00BDBDBD;
    BorderDark := $009E9E9E;
    
    InputBg := $00FFFFFF;
    InputBorder := $00BDBDBD;
    InputFocusBorder := $00E87722;
    InputText := $00212121;
    
    BtnPrimaryBg := $00E87722;
    BtnPrimaryText := $00FFFFFF;
    BtnSecondaryBg := $00F5F5F5;
    BtnSecondaryText := $00212121;
    
    SelectionBg := $00E87722;
    SelectionText := $00FFFFFF;
    
    ScrollbarThumb := $00BDBDBD;
    ScrollbarTrack := $00F5F5F5;
  end;
end;

function DarkTheme: TTheme;
begin
  Result := TTheme.Create('Dark', ttDark);
  with Result.Colors do
  begin
    BgPrimary := $00121212;
    BgSecondary := $001E1E1E;
    BgTertiary := $002D2D2D;
    BgCard := $001E1E1E;
    
    TextPrimary := $00FFFFFF;
    TextSecondary := $00B0B0B0;
    TextDisabled := $00616161;
    TextInverse := $00121212;
    
    AccentPrimary := $00F4A740;
    AccentSecondary := $00FFB74D;
    AccentDanger := $00EF5350;
    AccentWarning := $00FFCA28;
    AccentSuccess := $0066BB6A;
    
    BorderLight := $00333333;
    BorderMedium := $00484848;
    BorderDark := $00606060;
    
    InputBg := $002D2D2D;
    InputBorder := $00484848;
    InputFocusBorder := $00F4A740;
    InputText := $00FFFFFF;
    
    BtnPrimaryBg := $00F4A740;
    BtnPrimaryText := $00121212;
    BtnSecondaryBg := $002D2D2D;
    BtnSecondaryText := $00FFFFFF;
    
    SelectionBg := $00F4A740;
    SelectionText := $00121212;
    
    ScrollbarThumb := $00484848;
    ScrollbarTrack := $001E1E1E;
  end;
end;

function SolarizedTheme: TTheme;
begin
  Result := TTheme.Create('Solarized', ttSolarized);
  with Result.Colors do
  begin
    BgPrimary := $00FDF6E3;
    BgSecondary := $00EEE8D5;
    BgTertiary := $00E8E0CC;
    BgCard := $00FDF6E3;
    
    TextPrimary := $00657B83;
    TextSecondary := $00839496;
    TextDisabled := $0093A1A1;
    TextInverse := $00FDF6E3;
    
    AccentPrimary := $00CB4B16;
    AccentSecondary := $00D33682;
    AccentDanger := $00DC322F;
    AccentWarning := $00B58900;
    AccentSuccess := $00859900;
    
    BorderLight := $00EEE8D5;
    BorderMedium := $00D0CAB8;
    BorderDark := $00B3ADBB;
    
    InputBg := $00FDF6E3;
    InputBorder := $00D0CAB8;
    InputFocusBorder := $00CB4B16;
    InputText := $00657B83;
    
    BtnPrimaryBg := $00CB4B16;
    BtnPrimaryText := $00FDF6E3;
    BtnSecondaryBg := $00EEE8D5;
    BtnSecondaryText := $00657B83;
    
    SelectionBg := $00268BD2;
    SelectionText := $00FDF6E3;
    
    ScrollbarThumb := $00B3ADBB;
    ScrollbarTrack := $00EEE8D5;
  end;
end;

function NordTheme: TTheme;
begin
  Result := TTheme.Create('Nord', ttNord);
  with Result.Colors do
  begin
    BgPrimary := $002E3440;
    BgSecondary := $003B4252;
    BgTertiary := $00434C5E;
    BgCard := $003B4252;
    
    TextPrimary := $00ECEFF4;
    TextSecondary := $00D8DEE9;
    TextDisabled := $004C566A;
    TextInverse := $002E3440;
    
    AccentPrimary := $0088C0D0;
    AccentSecondary := $0081A1C1;
    AccentDanger := $00BF616A;
    AccentWarning := $00EBCB8B;
    AccentSuccess := $00A3BE8C;
    
    BorderLight := $00434C5E;
    BorderMedium := $004C566A;
    BorderDark := $00616C80;
    
    InputBg := $003B4252;
    InputBorder := $004C566A;
    InputFocusBorder := $0088C0D0;
    InputText := $00ECEFF4;
    
    BtnPrimaryBg := $0088C0D0;
    BtnPrimaryText := $002E3440;
    BtnSecondaryBg := $00434C5E;
    BtnSecondaryText := $00ECEFF4;
    
    SelectionBg := $0088C0D0;
    SelectionText := $002E3440;
    
    ScrollbarThumb := $004C566A;
    ScrollbarTrack := $003B4252;
  end;
end;

function HighContrastTheme: TTheme;
begin
  Result := TTheme.Create('High Contrast', ttHighContrast);
  with Result.Colors do
  begin
    BgPrimary := $00000000;
    BgSecondary := $00000000;
    BgTertiary := $00333333;
    BgCard := $00000000;
    
    TextPrimary := $00FFFFFF;
    TextSecondary := $00FFFF00;
    TextDisabled := $00808080;
    TextInverse := $00000000;
    
    AccentPrimary := $00FFFF00;
    AccentSecondary := $0000FFFF;
    AccentDanger := $00FF0000;
    AccentWarning := $00FF8000;
    AccentSuccess := $0000FF00;
    
    BorderLight := $00FFFFFF;
    BorderMedium := $00FFFFFF;
    BorderDark := $00FFFFFF;
    
    InputBg := $00000000;
    InputBorder := $00FFFFFF;
    InputFocusBorder := $00FFFF00;
    InputText := $00FFFFFF;
    
    BtnPrimaryBg := $00FFFFFF;
    BtnPrimaryText := $00000000;
    BtnSecondaryBg := $00000000;
    BtnSecondaryText := $00FFFFFF;
    
    SelectionBg := $00FFFFFF;
    SelectionText := $00000000;
    
    ScrollbarThumb := $00FFFFFF;
    ScrollbarTrack := $00000000;
  end;
end;

{ TThemeManager }

class function TThemeManager.Instance: TThemeManager;
begin
  if FInstance = nil then
    FInstance := TThemeManager.Create;
  Result := FInstance;
end;

constructor TThemeManager.Create;
begin
  inherited Create;
  FThemes := TList.Create;
  BuildDefaultThemes;
  
  // Default to light theme
  FCurrentTheme := GetBuiltinTheme(ttLight);
end;

destructor TThemeManager.Destroy;
var
  i: Integer;
begin
  for i := 0 to FThemes.Count - 1 do
    TTheme(FThemes[i]).Free;
  FThemes.Free;
  inherited Destroy;
end;

procedure TThemeManager.BuildDefaultThemes;
begin
  RegisterTheme(LightTheme);
  RegisterTheme(DarkTheme);
  RegisterTheme(SolarizedTheme);
  RegisterTheme(NordTheme);
  RegisterTheme(HighContrastTheme);
end;

procedure TThemeManager.RegisterTheme(ATheme: TTheme);
begin
  FThemes.Add(ATheme);
end;

function TThemeManager.GetTheme(const AName: string): TTheme;
var
  i: Integer;
begin
  Result := nil;
  for i := 0 to FThemes.Count - 1 do
    if TTheme(FThemes[i]).Name = AName then
    begin
      Result := TTheme(FThemes[i]);
      Exit;
    end;
end;

function TThemeManager.GetBuiltinTheme(AType: TThemeType): TTheme;
var
  i: Integer;
begin
  Result := nil;
  for i := 0 to FThemes.Count - 1 do
    if TTheme(FThemes[i]).ThemeType = AType then
    begin
      Result := TTheme(FThemes[i]);
      Exit;
    end;
end;

procedure TThemeManager.ApplyThemeToControl(AControl: TControl; ATheme: TTheme);
begin
  // Apply colors based on control type
  if AControl is TButton then
  begin
    AControl.Color := ATheme.Colors.BtnSecondaryBg;
    AControl.Font.Color := ATheme.Colors.BtnSecondaryText;
  end
  else if AControl is TEdit then
  begin
    AControl.Color := ATheme.Colors.InputBg;
    AControl.Font.Color := ATheme.Colors.InputText;
  end
  else if AControl is TLabel then
  begin
    TLabel(AControl).Font.Color := ATheme.Colors.TextPrimary;
    // Transparent labels don't need background
  end
  else if AControl is TPanel then
  begin
    AControl.Color := ATheme.Colors.BgSecondary;
  end
  else if AControl is TListBox then
  begin
    AControl.Color := ATheme.Colors.InputBg;
    AControl.Font.Color := ATheme.Colors.InputText;
  end
  else if AControl is TComboBox then
  begin
    AControl.Color := ATheme.Colors.InputBg;
    AControl.Font.Color := ATheme.Colors.InputText;
  end
  else
  begin
    AControl.Color := ATheme.Colors.BgPrimary;
    if Assigned(AControl.Font) then
      AControl.Font.Color := ATheme.Colors.TextPrimary;
  end;
  
  // Apply font settings
  if Assigned(AControl.Font) then
  begin
    AControl.Font.Name := ATheme.Fonts.FontFamily;
    AControl.Font.Size := ATheme.Fonts.FontSizeNormal;
  end;
  
  // Recurse into children
  if AControl is TWinControl then
    for var i := 0 to TWinControl(AControl).ControlCount - 1 do
      ApplyThemeToControl(TWinControl(AControl).Controls[i], ATheme);
end;

procedure TThemeManager.ApplyThemeToForm(AForm: TForm; ATheme: TTheme);
begin
  AForm.Color := ATheme.Colors.BgPrimary;
  AForm.Font.Name := ATheme.Fonts.FontFamily;
  AForm.Font.Size := ATheme.Fonts.FontSizeNormal;
  AForm.Font.Color := ATheme.Colors.TextPrimary;
  
  for var i := 0 to AForm.ControlCount - 1 do
    ApplyThemeToControl(AForm.Controls[i], ATheme);
    
  AForm.Refresh;
end;

procedure TThemeManager.ApplyTheme(const AName: string);
var
  Theme: TTheme;
begin
  Theme := GetTheme(AName);
  if Assigned(Theme) then
    ApplyTheme(Theme)
  else
    WriteLn('Theme not found: ', AName);
end;

procedure TThemeManager.ApplyTheme(ATheme: TTheme);
begin
  FCurrentTheme := ATheme;
  ApplyToAllForms;
  
  if Assigned(FOnThemeChange) then
    FOnThemeChange(ATheme);
    
  WriteLn('Theme applied: ', ATheme.Name);
end;

procedure TThemeManager.ApplyToAllForms;
var
  i: Integer;
begin
  for i := 0 to Screen.FormCount - 1 do
    ApplyThemeToForm(Screen.Forms[i], FCurrentTheme);
end;

function TThemeManager.ListThemes: TStringList;
var
  i: Integer;
begin
  Result := TStringList.Create;
  for i := 0 to FThemes.Count - 1 do
    Result.Add(TTheme(FThemes[i]).Name);
end;

end.
```

---

## 78.2 Theme Switcher Component

```pascal
unit theme_switcher;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Controls, StdCtrls, ExtCtrls, Graphics,
  theme_system;

type
  TThemeSwitcherPanel = class(TPanel)
  private
    FThemeButtons: TList;
    
    procedure CreateThemeButton(const AName: string; AColor: TColor);
    procedure ThemeButtonClick(Sender: TObject);
    procedure SetupButtons;
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure Refresh; override;
  end;

// Dark mode toggle switch
type
  TDarkModeToggle = class(TCustomControl)
  private
    FIsDark: Boolean;
    FAnimPos: Double;  // 0.0 to 1.0
    FAnimTimer: TTimer;
    FOnChange: TNotifyEvent;
    
    procedure SetIsDark(AValue: Boolean);
    procedure AnimTimer(Sender: TObject);
    
  protected
    procedure Paint; override;
    procedure MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer); override;
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure Toggle;
    
  published
    property IsDark: Boolean read FIsDark write SetIsDark;
    property OnChange: TNotifyEvent read FOnChange write FOnChange;
    property Width default 56;
    property Height default 28;
  end;

implementation

{ TDarkModeToggle }

constructor TDarkModeToggle.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FIsDark := False;
  FAnimPos := 0.0;
  Width := 56;
  Height := 28;
  Cursor := crHandPoint;
  
  FAnimTimer := TTimer.Create(Self);
  FAnimTimer.Interval := 16;  // ~60fps
  FAnimTimer.Enabled := False;
  FAnimTimer.OnTimer := @AnimTimer;
end;

destructor TDarkModeToggle.Destroy;
begin
  FAnimTimer.Free;
  inherited Destroy;
end;

procedure TDarkModeToggle.SetIsDark(AValue: Boolean);
begin
  if FIsDark <> AValue then
  begin
    FIsDark := AValue;
    FAnimTimer.Enabled := True;  // Start animation
    
    if Assigned(FOnChange) then FOnChange(Self);
    
    // Apply theme
    if FIsDark then
      TThemeManager.Instance.ApplyTheme('Dark')
    else
      TThemeManager.Instance.ApplyTheme('Light');
  end;
end;

procedure TDarkModeToggle.AnimTimer(Sender: TObject);
const
  AnimSpeed = 0.1;
begin
  if FIsDark then
  begin
    FAnimPos := Min(1.0, FAnimPos + AnimSpeed);
    if FAnimPos >= 1.0 then
      FAnimTimer.Enabled := False;
  end
  else
  begin
    FAnimPos := Max(0.0, FAnimPos - AnimSpeed);
    if FAnimPos <= 0.0 then
      FAnimTimer.Enabled := False;
  end;
  
  Invalidate;
end;

procedure TDarkModeToggle.Paint;
var
  R: TRect;
  ThumbX: Integer;
  TrackColor, ThumbColor: TColor;
begin
  R := ClientRect;
  
  // Track
  if FIsDark then
    TrackColor := $00F4A740
  else
    TrackColor := $00CCCCCC;
    
  // Smooth transition
  if FAnimPos > 0 then
  begin
    var R1 := $00CCCCCC;
    var G1 := $00CCCCCC;
    var B1 := $00CCCCCC;
    var R2 := Red($00F4A740);
    var G2 := Green($00F4A740);
    var B2 := Blue($00F4A740);
    
    TrackColor := RGBToColor(
      Round(R1 + (R2 - R1) * FAnimPos),
      Round(G1 + (G2 - G1) * FAnimPos),
      Round(B1 + (B2 - B1) * FAnimPos)
    );
  end;
  
  // Draw track
  Canvas.Brush.Color := TrackColor;
  Canvas.Pen.Color := TrackColor;
  Canvas.RoundRect(R.Left, R.Top, R.Right, R.Bottom, 
    R.Height, R.Height);
  
  // Draw thumb
  ThumbColor := clWhite;
  ThumbX := Round(R.Left + 2 + FAnimPos * (R.Width - R.Height));
  
  Canvas.Brush.Color := ThumbColor;
  Canvas.Pen.Color := $00DDDDDD;
  Canvas.Ellipse(ThumbX, R.Top + 2, ThumbX + R.Height - 4, R.Bottom - 2);
  
  // Draw icons
  Canvas.Font.Size := 10;
  
  if FAnimPos < 0.5 then
  begin
    // Show sun icon (light mode)
    Canvas.Font.Color := clGray;
    Canvas.TextOut(R.Right - 18, R.Top + 6, '☀');
  end
  else
  begin
    // Show moon icon (dark mode)
    Canvas.Font.Color := $00F4A740;
    Canvas.TextOut(R.Left + 4, R.Top + 6, '🌙');
  end;
end;

procedure TDarkModeToggle.MouseDown(Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  inherited MouseDown(Button, Shift, X, Y);
  if Button = mbLeft then
    Toggle;
end;

procedure TDarkModeToggle.Toggle;
begin
  SetIsDark(not FIsDark);
end;

end.
```

---

## 78.3 ตัวอย่างการใช้งาน Theme

```pascal
program theme_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  theme_system, theme_switcher;

type
  TThemeDemoForm = class(TForm)
  private
    pnlTop: TPanel;
    pnlContent: TPanel;
    lblTitle: TLabel;
    lblBody: TLabel;
    edtSample: TEdit;
    btnPrimary: TButton;
    btnSecondary: TButton;
    cmbTheme: TComboBox;
    DarkToggle: TDarkModeToggle;
    
    procedure SetupUI;
    procedure cmbThemeChange(Sender: TObject);
    procedure DarkToggleChange(Sender: TObject);
    
  public
    constructor Create(AOwner: TComponent); override;
  end;

constructor TThemeDemoForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  Caption := 'Theme Demo';
  Width := 600;
  Height := 400;
  Position := poScreenCenter;
  
  SetupUI;
  
  // Apply default theme
  TThemeManager.Instance.ApplyTheme(TThemeManager.Instance.CurrentTheme);
end;

procedure TThemeDemoForm.SetupUI;
var
  Themes: TStringList;
  i: Integer;
begin
  // Top bar
  pnlTop := TPanel.Create(Self);
  pnlTop.Parent := Self;
  pnlTop.Align := alTop;
  pnlTop.Height := 50;
  pnlTop.BevelOuter := bvNone;
  
  lblTitle := TLabel.Create(Self);
  lblTitle.Parent := pnlTop;
  lblTitle.Caption := 'Theme Demo Application';
  lblTitle.Font.Size := 14;
  lblTitle.Font.Style := [fsBold];
  lblTitle.Left := 16;
  lblTitle.Top := 14;
  
  // Theme combo
  cmbTheme := TComboBox.Create(Self);
  cmbTheme.Parent := pnlTop;
  cmbTheme.Left := 400;
  cmbTheme.Top := 12;
  cmbTheme.Width := 120;
  cmbTheme.Style := csDropDownList;
  cmbTheme.OnChange := @cmbThemeChange;
  
  Themes := TThemeManager.Instance.ListThemes;
  try
    for i := 0 to Themes.Count - 1 do
      cmbTheme.Items.Add(Themes[i]);
    cmbTheme.ItemIndex := 0;
  finally
    Themes.Free;
  end;
  
  // Dark mode toggle
  DarkToggle := TDarkModeToggle.Create(Self);
  DarkToggle.Parent := pnlTop;
  DarkToggle.Left := 540;
  DarkToggle.Top := 11;
  DarkToggle.OnChange := @DarkToggleChange;
  
  // Content area
  pnlContent := TPanel.Create(Self);
  pnlContent.Parent := Self;
  pnlContent.Align := alClient;
  pnlContent.BevelOuter := bvNone;
  
  lblBody := TLabel.Create(Self);
  lblBody.Parent := pnlContent;
  lblBody.Caption := 'This is sample body text';
  lblBody.Left := 24;
  lblBody.Top := 24;
  
  edtSample := TEdit.Create(Self);
  edtSample.Parent := pnlContent;
  edtSample.Text := 'Sample input field';
  edtSample.Left := 24;
  edtSample.Top := 60;
  edtSample.Width := 300;
  
  btnPrimary := TButton.Create(Self);
  btnPrimary.Parent := pnlContent;
  btnPrimary.Caption := 'Primary Button';
  btnPrimary.Left := 24;
  btnPrimary.Top := 100;
  btnPrimary.Width := 130;
  
  btnSecondary := TButton.Create(Self);
  btnSecondary.Parent := pnlContent;
  btnSecondary.Caption := 'Secondary';
  btnSecondary.Left := 164;
  btnSecondary.Top := 100;
  btnSecondary.Width := 100;
end;

procedure TThemeDemoForm.cmbThemeChange(Sender: TObject);
begin
  if cmbTheme.ItemIndex >= 0 then
    TThemeManager.Instance.ApplyTheme(cmbTheme.Items[cmbTheme.ItemIndex]);
end;

procedure TThemeDemoForm.DarkToggleChange(Sender: TObject);
begin
  if DarkToggle.IsDark then
    cmbTheme.ItemIndex := cmbTheme.Items.IndexOf('Dark')
  else
    cmbTheme.ItemIndex := cmbTheme.Items.IndexOf('Light');
end;

var
  App: TApplication;
  Form: TThemeDemoForm;
begin
  Application.Initialize;
  
  Form := TThemeDemoForm.Create(nil);
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
1. **Theme System** - สร้างระบบ theme ที่ครบถ้วน
2. **Built-in Themes** - Light, Dark, Solarized, Nord, High Contrast
3. **TThemeManager** - Singleton สำหรับจัดการ themes
4. **Dark Mode Toggle** - Animation toggle switch
5. **Dynamic Theming** - เปลี่ยน theme ขณะ runtime
6. **Color Tokens** - ตัวแปร colors สำหรับ consistency

การมีระบบ theme ที่ดีทำให้แอปพลิเคชันดูสวยงาม รองรับ dark mode ที่เป็นที่นิยม และตอบสนองต่อ accessibility requirements
