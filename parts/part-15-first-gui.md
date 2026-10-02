# Part 15 - GUI แรกด้วย Lazarus

## บทนำ

Lazarus เป็น IDE (Integrated Development Environment) ที่ทรงพลังสำหรับพัฒนา GUI applications ด้วย Free Pascal มี Form Designer, Component Palette, Object Inspector ที่ใช้งานง่าย ทำให้สร้าง desktop application ได้อย่างรวดเร็ว

---

## 15.1 Lazarus IDE Overview

### ส่วนประกอบหลักของ Lazarus IDE

```
+--------------------------------------------------+
|  Lazarus IDE                                      |
+--------------------------------------------------+
|  Menu Bar: File Edit View Project Run Tools ...   |
+--------------------------------------------------+
|  [Component Palette - แถวด้านบน]                  |
|  Standard | Additional | Data | ...               |
+------+-------------------+------------------------+
|      |                   |  Object Inspector      |
|  S   |  Form Designer    |  Properties | Events   |
|  o   |                   |  Name: Form1           |
|  u   |  +------------+   |  Caption: Form1        |
|  r   |  |            |   |  Width: 640            |
|  c   |  |  Your Form |   |  Height: 480           |
|  e   |  |            |   |  Color: clBtnFace      |
|  E   |  +------------+   |  ...                   |
|  d   |                   +------------------------+
|  i   |                                            |
|  t   |                                            |
|  o   |                                            |
|  r   |                                            |
+------+-------------------------------------------+
|  Messages / Output                                |
+--------------------------------------------------+
```

### ส่วนประกอบสำคัญ:

| ส่วน | หน้าที่ |
|-----|--------|
| **Menu Bar** | เมนูทั้งหมด (File, Edit, View, Project, Run...) |
| **Component Palette** | รายการ components ที่วางบน Form ได้ |
| **Form Designer** | ออกแบบ UI แบบ visual drag-and-drop |
| **Object Inspector** | ตั้งค่า properties และ events |
| **Source Editor** | เขียน Pascal code |
| **Messages** | แสดง compile errors, warnings |

---

## 15.2 การสร้างโปรเจกต์ใหม่

### ขั้นตอนสร้าง GUI Application

1. เปิด Lazarus
2. ไปที่ **File → New → Application** หรือกด **Ctrl+Shift+N**
3. IDE จะสร้างไฟล์หลัก:
   - `project1.lpr` - Main program file
   - `unit1.pas` - Unit ของ Form หลัก
   - `unit1.lfm` - Form layout file (XML)

### ไฟล์ project1.lpr (สร้างอัตโนมัติ)

```pascal
program Project1;

{$mode objfpc}{$H+}

uses
  {$IFDEF UNIX}
  cthreads,
  {$ENDIF}
  {$IFDEF HASAMIGA}
  athreads,
  {$ENDIF}
  Interfaces,  // สำคัญ: เปิด LCL
  Forms,
  Unit1
  { เพิ่ม units ใหม่ตรงนี้ };

{$R *.res}

begin
  RequireDerivedFormResource := True;
  Application.Title := 'My First App';
  Application.Scaled := True;
  Application.Initialize;
  Application.CreateForm(TForm1, Form1);
  Application.Run;
end.
```

### ไฟล์ unit1.pas (สร้างอัตโนมัติ)

```pascal
unit Unit1;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs;

type
  TForm1 = class(TForm)
    // Components จะถูกประกาศที่นี่อัตโนมัติ
  private
    // Private declarations
  public
    // Public declarations
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

end.
```

---

## 15.3 Form Designer

### การวาง Component ลงบน Form

1. เลือก component จาก **Component Palette** (คลิก 1 ครั้ง)
2. คลิกและลากบน Form เพื่อวาง หรือ double-click เพื่อวางกลาง Form
3. ปรับขนาดด้วยการลากขอบ

### Shortcut ที่ใช้บ่อย

| Shortcut | การกระทำ |
|----------|----------|
| F12 | สลับระหว่าง Form Designer และ Code Editor |
| Ctrl+A | เลือกทุก component |
| Delete | ลบ component ที่เลือก |
| F2 | Rename component |
| Ctrl+Z | Undo |
| Ctrl+Y | Redo |
| Tab | เลือก component ถัดไป |
| Arrow Keys | ย้าย component 1 pixel |
| Shift+Arrow | ย้าย 5 pixels |

---

## 15.4 Component Palette

### Standard Tab (ที่ใช้บ่อย)

| Component | ไอคอน | ใช้สำหรับ |
|-----------|-------|---------|
| **TButton** | [OK] | ปุ่มกด |
| **TLabel** | [A] | แสดงข้อความ |
| **TEdit** | [___] | รับ input 1 บรรทัด |
| **TMemo** | [≡] | รับ input หลายบรรทัด |
| **TCheckBox** | [✓] | checkbox |
| **TRadioButton** | [○] | radio button |
| **TListBox** | [≡] | list items |
| **TComboBox** | [▼] | dropdown list |
| **TGroupBox** | [□] | จัดกลุ่ม controls |
| **TPanel** | [▬] | container panel |
| **TMainMenu** | [≡] | menu bar |
| **TPopupMenu** | [↑≡] | right-click menu |
| **TScrollBar** | [◄──►] | scroll bar |

### Additional Tab (ที่ใช้บ่อย)

| Component | ใช้สำหรับ |
|-----------|---------|
| **TImage** | แสดงรูปภาพ |
| **TBitBtn** | ปุ่มพร้อมรูป |
| **TSpinEdit** | ตัวเลขพร้อมลูกศร |
| **TTrackBar** | slider |
| **TProgressBar** | progress bar |
| **TColorButton** | เลือกสี |
| **TCalendar** | ปฏิทิน |
| **TShapeBox** | รูปทรงเรขาคณิต |

### Dialogs Tab

| Component | ใช้สำหรับ |
|-----------|---------|
| **TOpenDialog** | เปิดไฟล์ |
| **TSaveDialog** | บันทึกไฟล์ |
| **TFontDialog** | เลือกฟอนต์ |
| **TColorDialog** | เลือกสี |
| **TPrintDialog** | พิมพ์ |
| **TMessageDlg** | แสดง message (built-in) |

---

## 15.5 Object Inspector

### Properties Tab

Properties หลักที่ควรรู้:

| Property | ชนิด | ความหมาย |
|----------|------|---------|
| `Name` | String | ชื่อ component ใน code |
| `Caption` | String | ข้อความที่แสดง (Form, Button, Label) |
| `Text` | String | ข้อความ (Edit, Memo) |
| `Width` | Integer | ความกว้าง (pixels) |
| `Height` | Integer | ความสูง (pixels) |
| `Left` | Integer | ตำแหน่งจากซ้าย |
| `Top` | Integer | ตำแหน่งจากบน |
| `Visible` | Boolean | มองเห็น/ซ่อน |
| `Enabled` | Boolean | เปิด/ปิดการใช้งาน |
| `Color` | TColor | สีพื้นหลัง |
| `Font` | TFont | ตั้งค่าฟอนต์ |
| `Hint` | String | tooltip text |
| `ShowHint` | Boolean | แสดง tooltip |
| `TabOrder` | Integer | ลำดับ Tab |
| `Anchors` | Set | ยึดกับขอบ Form |
| `Align` | TAlign | จัดตำแหน่งอัตโนมัติ |

### Events Tab

Events หลักที่ควรรู้:

| Event | เมื่อไหร่ |
|-------|---------|
| `OnClick` | คลิก |
| `OnDblClick` | ดับเบิลคลิก |
| `OnChange` | ค่าเปลี่ยน (Edit, Memo) |
| `OnKeyPress` | กดคีย์ |
| `OnKeyDown` | กดคีย์ (เต็ม) |
| `OnMouseEnter` | เมาส์เข้า |
| `OnMouseLeave` | เมาส์ออก |
| `OnMouseMove` | เมาส์ขยับ |
| `OnCreate` | สร้าง Form |
| `OnDestroy` | ปิด Form |
| `OnClose` | ก่อนปิด |
| `OnShow` | แสดง Form |
| `OnResize` | ปรับขนาด |
| `OnEnter` | focus เข้า control |
| `OnExit` | focus ออก control |

---

## 15.6 TForm Properties หลัก

### ตัวอย่างที่ 1: ตั้งค่า Form ใน Code

```pascal
unit MainForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs;

type
  TForm1 = class(TForm)
    procedure FormCreate(Sender: TObject);
    procedure FormClose(Sender: TObject; var CloseAction: TCloseAction);
  private
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  // ตั้งค่า Form ใน OnCreate
  Caption    := 'แอพพลิเคชั่นแรก';
  Width      := 800;
  Height     := 600;
  Position   := poScreenCenter;  // กลางหน้าจอ
  Color      := clWhite;
  
  // ปิด maximize button
  // BorderIcons := [biSystemMenu, biMinimize];
  
  // ขนาดคงที่
  // BorderStyle := bsSingle;
  
  WriteLn('Form Created!');
end;

procedure TForm1.FormClose(Sender: TObject; var CloseAction: TCloseAction);
begin
  WriteLn('Form Closing...');
  CloseAction := caFree;  // free memory เมื่อปิด
end;

end.
```

---

## 15.7 Button Click Event

### ตัวอย่างที่ 2: Button Click พื้นฐาน

```pascal
unit ButtonDemo;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs, StdCtrls;

type
  TForm1 = class(TForm)
    Button1: TButton;
    Button2: TButton;
    Button3: TButton;
    Label1 : TLabel;
    procedure FormCreate(Sender: TObject);
    procedure Button1Click(Sender: TObject);
    procedure Button2Click(Sender: TObject);
    procedure Button3Click(Sender: TObject);
  private
    FClickCount: Integer;
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  FClickCount := 0;
  
  // ตั้งค่า Form
  Caption := 'Button Demo';
  Width   := 400;
  Height  := 300;
  Position := poScreenCenter;
  
  // ตั้งค่า Buttons
  Button1.Caption := 'คลิกฉัน!';
  Button1.Left    := 50;
  Button1.Top     := 50;
  Button1.Width   := 100;
  Button1.Height  := 35;
  
  Button2.Caption := 'รีเซ็ต';
  Button2.Left    := 160;
  Button2.Top     := 50;
  Button2.Width   := 80;
  Button2.Height  := 35;
  
  Button3.Caption := 'ปิด';
  Button3.Left    := 250;
  Button3.Top     := 50;
  Button3.Width   := 80;
  Button3.Height  := 35;
  Button3.Color   := clRed;
  
  // ตั้งค่า Label
  Label1.Caption  := 'กดปุ่มเพื่อเริ่ม';
  Label1.Left     := 50;
  Label1.Top      := 120;
  Label1.Font.Size := 14;
end;

procedure TForm1.Button1Click(Sender: TObject);
begin
  Inc(FClickCount);
  Label1.Caption := Format('กดไปแล้ว %d ครั้ง!', [FClickCount]);
  
  // เปลี่ยนสีตามจำนวนครั้ง
  case FClickCount mod 4 of
    0: Label1.Font.Color := clBlue;
    1: Label1.Font.Color := clRed;
    2: Label1.Font.Color := clGreen;
    3: Label1.Font.Color := clPurple;
  end;
end;

procedure TForm1.Button2Click(Sender: TObject);
begin
  FClickCount := 0;
  Label1.Caption := 'รีเซ็ตแล้ว';
  Label1.Font.Color := clBlack;
end;

procedure TForm1.Button3Click(Sender: TObject);
begin
  Close;
end;

end.
```

---

## 15.8 Label และ Edit

### ตัวอย่างที่ 3: Label และ Edit พื้นฐาน

```pascal
unit LabelEditDemo;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs, StdCtrls;

type
  TForm1 = class(TForm)
    Label1  : TLabel;
    Label2  : TLabel;
    Label3  : TLabel;
    Edit1   : TEdit;
    Edit2   : TEdit;
    Edit3   : TEdit;
    Button1 : TButton;
    LabelResult: TLabel;
    procedure FormCreate(Sender: TObject);
    procedure Button1Click(Sender: TObject);
    procedure Edit1Change(Sender: TObject);
    procedure Edit2KeyPress(Sender: TObject; var Key: Char);
  private
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  Caption := 'Label และ Edit Demo';
  Width   := 400;
  Height  := 400;
  Position := poScreenCenter;
  
  // Labels
  Label1.Caption := 'ชื่อ:';
  Label2.Caption := 'นามสกุล:';
  Label3.Caption := 'อายุ:';
  
  // Edits
  Edit1.PlaceholderText := 'กรอกชื่อ...';
  Edit2.PlaceholderText := 'กรอกนามสกุล...';
  Edit3.PlaceholderText := 'กรอกอายุ...';
  Edit3.MaxLength := 3;  // รับได้ไม่เกิน 3 ตัว
  
  // Button
  Button1.Caption := 'ยืนยัน';
  
  // Result Label
  LabelResult.Caption := '';
  LabelResult.Font.Size := 14;
  LabelResult.Font.Bold := True;
end;

procedure TForm1.Button1Click(Sender: TObject);
var
  Name, Surname : String;
  Age           : Integer;
begin
  Name    := Trim(Edit1.Text);
  Surname := Trim(Edit2.Text);
  
  // ตรวจสอบ input
  if Name = '' then
  begin
    ShowMessage('กรุณากรอกชื่อ');
    Edit1.SetFocus;
    Exit;
  end;
  
  if Surname = '' then
  begin
    ShowMessage('กรุณากรอกนามสกุล');
    Edit2.SetFocus;
    Exit;
  end;
  
  if not TryStrToInt(Edit3.Text, Age) then
  begin
    ShowMessage('อายุต้องเป็นตัวเลข');
    Edit3.SetFocus;
    Edit3.SelectAll;
    Exit;
  end;
  
  if (Age < 1) or (Age > 150) then
  begin
    ShowMessage('อายุต้องอยู่ระหว่าง 1-150 ปี');
    Edit3.SetFocus;
    Exit;
  end;
  
  LabelResult.Caption := Format('สวัสดี %s %s! อายุ %d ปี', [Name, Surname, Age]);
  LabelResult.Font.Color := clBlue;
end;

// รับเฉพาะตัวเลขใน Edit3 (อายุ)
procedure TForm1.Edit2KeyPress(Sender: TObject; var Key: Char);
begin
  if Sender = Edit3 then
    if not (Key in ['0'..'9', #8]) then  // #8 = Backspace
      Key := #0;  // ยกเลิกการรับ key
end;

procedure TForm1.Edit1Change(Sender: TObject);
begin
  // แสดง preview แบบ real-time
  LabelResult.Caption := 'สวัสดี ' + Edit1.Text + '...';
end;

end.
```

---

## 15.9 การรับ Input จากผู้ใช้

### ตัวอย่างที่ 4: Input Validation หลายรูปแบบ

```pascal
unit InputDemo;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, Spin;

type
  TForm1 = class(TForm)
    // Controls
    EditName     : TEdit;
    EditEmail    : TEdit;
    EditPhone    : TEdit;
    SpinAge      : TSpinEdit;
    CheckMale    : TCheckBox;
    CheckFemale  : TCheckBox;
    ComboCity    : TComboBox;
    MemoAddress  : TMemo;
    BtnSubmit    : TButton;
    BtnClear     : TButton;
    PanelResult  : TPanel;
    LabelResult  : TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure BtnSubmitClick(Sender: TObject);
    procedure BtnClearClick(Sender: TObject);
    procedure CheckMaleChange(Sender: TObject);
    procedure CheckFemaleChange(Sender: TObject);
  private
    function ValidateForm: Boolean;
    procedure ShowResult;
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  Caption  := 'แบบฟอร์มลงทะเบียน';
  Width    := 500;
  Height   := 600;
  Position := poScreenCenter;
  
  // Setup ComboBox
  ComboCity.Clear;
  ComboCity.Items.Add('กรุงเทพมหานคร');
  ComboCity.Items.Add('เชียงใหม่');
  ComboCity.Items.Add('ขอนแก่น');
  ComboCity.Items.Add('นครราชสีมา');
  ComboCity.Items.Add('ภูเก็ต');
  ComboCity.Items.Add('สงขลา');
  ComboCity.ItemIndex := 0;
  
  // Setup SpinEdit
  SpinAge.MinValue := 1;
  SpinAge.MaxValue := 120;
  SpinAge.Value    := 25;
  
  // Setup Memo
  MemoAddress.ScrollBars := ssVertical;
  MemoAddress.Lines.Clear;
  
  // Setup Result Panel
  PanelResult.Visible := False;
  PanelResult.Color   := clLime;
end;

function TForm1.ValidateForm: Boolean;
var
  Email : String;
  Phone : String;
begin
  Result := False;
  
  if Trim(EditName.Text) = '' then
  begin
    ShowMessage('กรุณากรอกชื่อ-นามสกุล');
    EditName.SetFocus;
    Exit;
  end;
  
  Email := Trim(EditEmail.Text);
  if (Pos('@', Email) < 2) or (Pos('.', Email) < 3) then
  begin
    ShowMessage('กรุณากรอกอีเมลที่ถูกต้อง');
    EditEmail.SetFocus;
    Exit;
  end;
  
  Phone := Trim(EditPhone.Text);
  var CleanPhone := '';
  for var i := 1 to Length(Phone) do
    if Phone[i] in ['0'..'9'] then CleanPhone := CleanPhone + Phone[i];
  
  if Length(CleanPhone) <> 10 then
  begin
    ShowMessage('เบอร์โทรต้องมี 10 หลัก');
    EditPhone.SetFocus;
    Exit;
  end;
  
  if not (CheckMale.Checked or CheckFemale.Checked) then
  begin
    ShowMessage('กรุณาเลือกเพศ');
    Exit;
  end;
  
  Result := True;
end;

procedure TForm1.ShowResult;
var
  Gender  : String;
  City    : String;
  Address : String;
begin
  if CheckMale.Checked then Gender := 'ชาย'
  else if CheckFemale.Checked then Gender := 'หญิง'
  else Gender := 'ไม่ระบุ';
  
  if ComboCity.ItemIndex >= 0 then
    City := ComboCity.Items[ComboCity.ItemIndex]
  else
    City := 'ไม่ระบุ';
  
  Address := MemoAddress.Lines.Text;
  
  LabelResult.Caption :=
    'ชื่อ: ' + EditName.Text + #13#10 +
    'อีเมล: ' + EditEmail.Text + #13#10 +
    'โทร: ' + EditPhone.Text + #13#10 +
    'อายุ: ' + IntToStr(SpinAge.Value) + ' ปี' + #13#10 +
    'เพศ: ' + Gender + #13#10 +
    'จังหวัด: ' + City;
    
  PanelResult.Visible := True;
end;

procedure TForm1.BtnSubmitClick(Sender: TObject);
begin
  if ValidateForm then ShowResult;
end;

procedure TForm1.BtnClearClick(Sender: TObject);
begin
  EditName.Clear;
  EditEmail.Clear;
  EditPhone.Clear;
  SpinAge.Value       := 25;
  CheckMale.Checked   := False;
  CheckFemale.Checked := False;
  ComboCity.ItemIndex := 0;
  MemoAddress.Clear;
  PanelResult.Visible := False;
  EditName.SetFocus;
end;

// Radio button behavior สำหรับ checkbox เพศ
procedure TForm1.CheckMaleChange(Sender: TObject);
begin
  if CheckMale.Checked then CheckFemale.Checked := False;
end;

procedure TForm1.CheckFemaleChange(Sender: TObject);
begin
  if CheckFemale.Checked then CheckMale.Checked := False;
end;

end.
```

---

## 15.10 MessageBox Dialogs

### ตัวอย่างที่ 5: MessageBox ประเภทต่างๆ

```pascal
unit MessageBoxDemo;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs, StdCtrls;

type
  TForm1 = class(TForm)
    BtnInfo     : TButton;
    BtnWarning  : TButton;
    BtnError    : TButton;
    BtnConfirm  : TButton;
    BtnYesNo    : TButton;
    BtnInput    : TButton;
    LabelResult : TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure BtnInfoClick(Sender: TObject);
    procedure BtnWarningClick(Sender: TObject);
    procedure BtnErrorClick(Sender: TObject);
    procedure BtnConfirmClick(Sender: TObject);
    procedure BtnYesNoClick(Sender: TObject);
    procedure BtnInputClick(Sender: TObject);
  private
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  Caption := 'Dialog Demo';
  Width   := 450;
  Height  := 400;
  Position := poScreenCenter;
  
  BtnInfo.Caption     := 'ข้อมูล (Info)';
  BtnWarning.Caption  := 'คำเตือน (Warning)';
  BtnError.Caption    := 'ข้อผิดพลาด (Error)';
  BtnConfirm.Caption  := 'ยืนยัน (OK/Cancel)';
  BtnYesNo.Caption    := 'ถาม (Yes/No)';
  BtnInput.Caption    := 'รับ Input';
  LabelResult.Caption := 'ผลลัพธ์จะแสดงที่นี่';
end;

procedure TForm1.BtnInfoClick(Sender: TObject);
begin
  // ShowMessage - แบบง่ายที่สุด
  ShowMessage('นี่คือข้อความข้อมูลทั่วไป');
  
  // หรือใช้ MessageDlg
  // MessageDlg('ข้อมูล', 'นี่คือข้อมูล', mtInformation, [mbOK], 0);
  
  LabelResult.Caption := 'กด Info dialog OK';
end;

procedure TForm1.BtnWarningClick(Sender: TObject);
begin
  MessageDlg('คำเตือน', 'ระวัง! การดำเนินการนี้อาจมีผลต่อข้อมูล',
    mtWarning, [mbOK], 0);
  LabelResult.Caption := 'กด Warning dialog OK';
end;

procedure TForm1.BtnErrorClick(Sender: TObject);
begin
  MessageDlg('เกิดข้อผิดพลาด', 'ไม่สามารถบันทึกข้อมูลได้ กรุณาตรวจสอบการเชื่อมต่อ',
    mtError, [mbOK], 0);
  LabelResult.Caption := 'กด Error dialog OK';
end;

procedure TForm1.BtnConfirmClick(Sender: TObject);
var
  Response : Integer;
begin
  Response := MessageDlg('ยืนยัน', 'คุณต้องการบันทึกข้อมูลหรือไม่?',
    mtConfirmation, [mbOK, mbCancel], 0);
    
  if Response = mrOK then
    LabelResult.Caption := 'คลิก OK - บันทึกข้อมูล'
  else
    LabelResult.Caption := 'คลิก Cancel - ยกเลิก';
end;

procedure TForm1.BtnYesNoClick(Sender: TObject);
var
  Response : Integer;
begin
  Response := MessageDlg('ถาม', 'คุณต้องการออกจากโปรแกรมหรือไม่?',
    mtConfirmation, [mbYes, mbNo], 0);
    
  case Response of
    mrYes: 
      begin
        LabelResult.Caption := 'คลิก Yes - กำลังออก...';
        // Application.Terminate;
      end;
    mrNo:
      LabelResult.Caption := 'คลิก No - ยังอยู่ในโปรแกรม';
  end;
end;

procedure TForm1.BtnInputClick(Sender: TObject);
var
  Input : String;
begin
  // InputQuery - รับ input จากผู้ใช้
  Input := '';
  if InputQuery('รับข้อมูล', 'กรุณากรอกชื่อของคุณ:', Input) then
  begin
    if Trim(Input) <> '' then
      LabelResult.Caption := 'สวัสดี ' + Input + '!'
    else
      LabelResult.Caption := 'ไม่ได้กรอกชื่อ';
  end
  else
    LabelResult.Caption := 'กด Cancel';
end;

end.
```

---

## 15.11 โปรแกรมตัวอย่าง: Hello World GUI

**ไฟล์ unit1.pas:**

```pascal
unit Unit1;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls;

type
  TForm1 = class(TForm)
    LabelTitle   : TLabel;
    LabelGreet   : TLabel;
    EditName     : TEdit;
    BtnGreet     : TButton;
    BtnClear     : TButton;
    PanelHeader  : TPanel;
    
    procedure FormCreate(Sender: TObject);
    procedure BtnGreetClick(Sender: TObject);
    procedure BtnClearClick(Sender: TObject);
    procedure EditNameKeyPress(Sender: TObject; var Key: Char);
  private
    procedure SetupUI;
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.SetupUI;
begin
  // Form
  Caption  := 'Hello World GUI';
  Width    := 450;
  Height   := 350;
  Position := poScreenCenter;
  Color    := $00F5F5F5;
  
  // Header Panel
  PanelHeader.Align         := alTop;
  PanelHeader.Height        := 80;
  PanelHeader.Color         := $002196F3;  // Blue
  PanelHeader.BevelOuter    := bvNone;
  
  // Title Label (ในหัว panel)
  LabelTitle.Parent         := PanelHeader;
  LabelTitle.Caption        := '🌍 Hello World Application';
  LabelTitle.Font.Size      := 16;
  LabelTitle.Font.Bold      := True;
  LabelTitle.Font.Color     := clWhite;
  LabelTitle.AutoSize       := True;
  LabelTitle.Left           := 20;
  LabelTitle.Top            := 25;
  
  // Greeting Label
  LabelGreet.Caption        := 'กรอกชื่อของคุณ แล้วกดปุ่มทักทาย';
  LabelGreet.Left           := 30;
  LabelGreet.Top            := 110;
  LabelGreet.Font.Size      := 13;
  LabelGreet.Font.Color     := $00666666;
  LabelGreet.AutoSize       := True;
  
  // Edit
  EditName.Left             := 30;
  EditName.Top              := 145;
  EditName.Width            := 380;
  EditName.Height           := 36;
  EditName.Font.Size        := 14;
  EditName.PlaceholderText  := 'ชื่อของคุณ...';
  
  // Greet Button
  BtnGreet.Caption          := 'ทักทาย! 👋';
  BtnGreet.Left             := 30;
  BtnGreet.Top              := 200;
  BtnGreet.Width            := 180;
  BtnGreet.Height           := 40;
  BtnGreet.Font.Size        := 13;
  BtnGreet.Color            := $002196F3;
  BtnGreet.Font.Color       := clWhite;
  
  // Clear Button
  BtnClear.Caption          := 'ล้าง';
  BtnClear.Left             := 220;
  BtnClear.Top              := 200;
  BtnClear.Width            := 100;
  BtnClear.Height           := 40;
  BtnClear.Font.Size        := 13;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  SetupUI;
end;

procedure TForm1.BtnGreetClick(Sender: TObject);
var
  Name : String;
begin
  Name := Trim(EditName.Text);
  
  if Name = '' then
  begin
    LabelGreet.Caption    := 'กรุณากรอกชื่อก่อนนะครับ!';
    LabelGreet.Font.Color := clRed;
    EditName.SetFocus;
    Exit;
  end;
  
  // แสดงคำทักทาย
  var Hour := HourOf(Now);
  var Greeting := '';
  
  if Hour < 12 then Greeting := 'สวัสดีตอนเช้า'
  else if Hour < 18 then Greeting := 'สวัสดีตอนบ่าย'
  else Greeting := 'สวัสดีตอนเย็น';
  
  LabelGreet.Caption    := Greeting + ', คุณ' + Name + '! ยินดีต้อนรับสู่ Lazarus Pascal!';
  LabelGreet.Font.Color := $002E7D32;  // Dark Green
  LabelGreet.Font.Size  := 13;
end;

procedure TForm1.BtnClearClick(Sender: TObject);
begin
  EditName.Clear;
  LabelGreet.Caption    := 'กรอกชื่อของคุณ แล้วกดปุ่มทักทาย';
  LabelGreet.Font.Color := $00666666;
  EditName.SetFocus;
end;

procedure TForm1.EditNameKeyPress(Sender: TObject; var Key: Char);
begin
  if Key = #13 then  // Enter key
    BtnGreet.Click;
end;

end.
```

---

## 15.12 โปรแกรมตัวอย่าง: Simple Calculator GUI

```pascal
unit Calculator;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls;

type
  TForm1 = class(TForm)
    // Display
    PanelDisplay  : TPanel;
    LabelExpr     : TLabel;
    LabelResult   : TLabel;
    
    // Number buttons
    Btn0, Btn1, Btn2, Btn3, Btn4 : TButton;
    Btn5, Btn6, Btn7, Btn8, Btn9 : TButton;
    
    // Operator buttons
    BtnAdd, BtnSub, BtnMul, BtnDiv : TButton;
    BtnEq, BtnC, BtnCE, BtnDot     : TButton;
    BtnPlusMinus, BtnPercent        : TButton;
    
    procedure FormCreate(Sender: TObject);
    procedure NumberClick(Sender: TObject);
    procedure OperatorClick(Sender: TObject);
    procedure BtnEqClick(Sender: TObject);
    procedure BtnCClick(Sender: TObject);
    procedure BtnCEClick(Sender: TObject);
    procedure BtnDotClick(Sender: TObject);
    procedure BtnPlusMinusClick(Sender: TObject);
    procedure BtnPercentClick(Sender: TObject);
  private
    FCurrentInput : String;
    FOperand1     : Double;
    FOperator     : Char;
    FNewInput     : Boolean;
    
    procedure UpdateDisplay;
    procedure SetupButtons;
    procedure StyleButton(Btn: TButton; BtnType: String);
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.StyleButton(Btn: TButton; BtnType: String);
begin
  Btn.Font.Size := 16;
  Btn.Height    := 65;
  Btn.Width     := 80;
  
  case BtnType of
    'number':
      begin
        Btn.Color       := $00424242;
        Btn.Font.Color  := clWhite;
      end;
    'operator':
      begin
        Btn.Color       := $00F57C00;
        Btn.Font.Color  := clWhite;
      end;
    'function':
      begin
        Btn.Color       := $00616161;
        Btn.Font.Color  := clWhite;
      end;
    'equals':
      begin
        Btn.Color       := $00F57C00;
        Btn.Font.Color  := clWhite;
        Btn.Font.Bold   := True;
      end;
  end;
end;

procedure TForm1.SetupButtons;
const
  Gap = 5;

procedure PlaceBtn(Btn: TButton; Col, Row: Integer; Caption, BtnType: String);
begin
  Btn.Caption := Caption;
  Btn.Left    := 10 + (Col - 1) * 85;
  Btn.Top     := 150 + (Row - 1) * 70;
  StyleButton(Btn, BtnType);
end;

begin
  // Row 1
  PlaceBtn(BtnCE,       1, 1, 'CE',  'function');
  PlaceBtn(BtnC,        2, 1, 'C',   'function');
  PlaceBtn(BtnPercent,  3, 1, '%',   'function');
  PlaceBtn(BtnDiv,      4, 1, '÷',   'operator');
  
  // Row 2
  PlaceBtn(Btn7, 1, 2, '7', 'number');
  PlaceBtn(Btn8, 2, 2, '8', 'number');
  PlaceBtn(Btn9, 3, 2, '9', 'number');
  PlaceBtn(BtnMul, 4, 2, '×', 'operator');
  
  // Row 3
  PlaceBtn(Btn4, 1, 3, '4', 'number');
  PlaceBtn(Btn5, 2, 3, '5', 'number');
  PlaceBtn(Btn6, 3, 3, '6', 'number');
  PlaceBtn(BtnSub, 4, 3, '−', 'operator');
  
  // Row 4
  PlaceBtn(Btn1, 1, 4, '1', 'number');
  PlaceBtn(Btn2, 2, 4, '2', 'number');
  PlaceBtn(Btn3, 3, 4, '3', 'number');
  PlaceBtn(BtnAdd, 4, 4, '+', 'operator');
  
  // Row 5
  PlaceBtn(BtnPlusMinus, 1, 5, '±', 'function');
  PlaceBtn(Btn0, 2, 5, '0', 'number');
  PlaceBtn(BtnDot, 3, 5, '.', 'number');
  PlaceBtn(BtnEq, 4, 5, '=', 'equals');
  
  // Wire events
  Btn0.OnClick := @NumberClick; Btn1.OnClick := @NumberClick;
  Btn2.OnClick := @NumberClick; Btn3.OnClick := @NumberClick;
  Btn4.OnClick := @NumberClick; Btn5.OnClick := @NumberClick;
  Btn6.OnClick := @NumberClick; Btn7.OnClick := @NumberClick;
  Btn8.OnClick := @NumberClick; Btn9.OnClick := @NumberClick;
  
  BtnAdd.OnClick := @OperatorClick;
  BtnSub.OnClick := @OperatorClick;
  BtnMul.OnClick := @OperatorClick;
  BtnDiv.OnClick := @OperatorClick;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  Caption  := 'เครื่องคิดเลข';
  Width    := 360;
  Height   := 520;
  Position := poScreenCenter;
  Color    := $00212121;  // Dark background
  
  // Display Panel
  PanelDisplay.Align      := alTop;
  PanelDisplay.Height     := 140;
  PanelDisplay.Color      := $00212121;
  PanelDisplay.BevelOuter := bvNone;
  
  // Expression label
  LabelExpr.Parent     := PanelDisplay;
  LabelExpr.Caption    := '';
  LabelExpr.Font.Size  := 14;
  LabelExpr.Font.Color := $00BDBDBD;
  LabelExpr.Left       := 10;
  LabelExpr.Top        := 80;
  LabelExpr.AutoSize   := True;
  
  // Result label
  LabelResult.Parent     := PanelDisplay;
  LabelResult.Caption    := '0';
  LabelResult.Font.Size  := 36;
  LabelResult.Font.Color := clWhite;
  LabelResult.Font.Bold  := True;
  LabelResult.AutoSize   := True;
  LabelResult.Left       := 10;
  LabelResult.Top        := 95;
  
  // Setup buttons
  SetupButtons;
  
  // Initialize
  FCurrentInput := '0';
  FOperand1     := 0;
  FOperator     := #0;
  FNewInput     := True;
end;

procedure TForm1.UpdateDisplay;
begin
  LabelResult.Caption := FCurrentInput;
end;

procedure TForm1.NumberClick(Sender: TObject);
var
  Digit : String;
begin
  Digit := TButton(Sender).Caption;
  
  if FNewInput then
  begin
    FCurrentInput := Digit;
    FNewInput     := False;
  end
  else
  begin
    if (FCurrentInput = '0') and (Digit <> '.') then
      FCurrentInput := Digit
    else
      FCurrentInput := FCurrentInput + Digit;
  end;
  
  UpdateDisplay;
end;

procedure TForm1.OperatorClick(Sender: TObject);
var
  OpChar : Char;
begin
  case TButton(Sender).Caption of
    '+': OpChar := '+';
    '−': OpChar := '-';
    '×': OpChar := '*';
    '÷': OpChar := '/';
  else
    OpChar := '+';
  end;
  
  FOperand1 := StrToFloatDef(FCurrentInput, 0);
  FOperator := OpChar;
  LabelExpr.Caption := FCurrentInput + ' ' + TButton(Sender).Caption;
  FNewInput := True;
end;

procedure TForm1.BtnEqClick(Sender: TObject);
var
  Operand2, Result : Double;
begin
  if FOperator = #0 then Exit;
  
  Operand2 := StrToFloatDef(FCurrentInput, 0);
  
  case FOperator of
    '+': Result := FOperand1 + Operand2;
    '-': Result := FOperand1 - Operand2;
    '*': Result := FOperand1 * Operand2;
    '/':
      begin
        if Operand2 = 0 then
        begin
          ShowMessage('ไม่สามารถหารด้วยศูนย์ได้!');
          Exit;
        end;
        Result := FOperand1 / Operand2;
      end;
  end;
  
  LabelExpr.Caption := LabelExpr.Caption + ' ' + FCurrentInput + ' =';
  
  // แสดงผลลัพธ์ (ตัดศูนย์ท้าย)
  if Frac(Result) = 0 then
    FCurrentInput := IntToStr(Round(Result))
  else
    FCurrentInput := FloatToStrF(Result, ffGeneral, 12, 4);
  
  FOperator := #0;
  FNewInput := True;
  UpdateDisplay;
end;

procedure TForm1.BtnCClick(Sender: TObject);
begin
  FCurrentInput := '0';
  FOperand1     := 0;
  FOperator     := #0;
  FNewInput     := True;
  LabelExpr.Caption := '';
  UpdateDisplay;
end;

procedure TForm1.BtnCEClick(Sender: TObject);
begin
  FCurrentInput := '0';
  FNewInput     := True;
  UpdateDisplay;
end;

procedure TForm1.BtnDotClick(Sender: TObject);
begin
  if FNewInput then
  begin
    FCurrentInput := '0.';
    FNewInput     := False;
  end
  else if Pos('.', FCurrentInput) = 0 then
    FCurrentInput := FCurrentInput + '.';
  UpdateDisplay;
end;

procedure TForm1.BtnPlusMinusClick(Sender: TObject);
var
  Val : Double;
begin
  Val := StrToFloatDef(FCurrentInput, 0);
  Val := -Val;
  if Frac(Val) = 0 then
    FCurrentInput := IntToStr(Round(Val))
  else
    FCurrentInput := FloatToStr(Val);
  UpdateDisplay;
end;

procedure TForm1.BtnPercentClick(Sender: TObject);
var
  Val : Double;
begin
  Val := StrToFloatDef(FCurrentInput, 0);
  Val := Val / 100;
  FCurrentInput := FloatToStrF(Val, ffGeneral, 12, 4);
  UpdateDisplay;
end;

end.
```

---

## 15.13 โปรแกรมตัวอย่าง: Name Greeting App

```pascal
unit GreetApp;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls;

type
  TForm1 = class(TForm)
    // Components
    PanelTop      : TPanel;
    LabelTitle    : TLabel;
    
    GroupBoxInfo  : TGroupBox;
    LabelName     : TLabel;
    LabelSurname  : TLabel;
    LabelTitle2   : TLabel;
    EditFirstName : TEdit;
    EditLastName  : TEdit;
    ComboTitle    : TComboBox;
    
    GroupBoxOptions: TGroupBox;
    CheckFormal   : TCheckBox;
    CheckEmoji    : TCheckBox;
    RadioThai     : TRadioButton;
    RadioEng      : TRadioButton;
    
    PanelResult   : TPanel;
    LabelGreeting : TLabel;
    
    BtnGreet      : TButton;
    BtnSave       : TButton;
    BtnClear      : TButton;
    
    MemoHistory   : TMemo;
    LabelHistory  : TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure BtnGreetClick(Sender: TObject);
    procedure BtnSaveClick(Sender: TObject);
    procedure BtnClearClick(Sender: TObject);
  private
    function GenerateGreeting: String;
    procedure AddToHistory(Msg: String);
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  Caption  := 'โปรแกรมทักทาย';
  Width    := 600;
  Height   := 650;
  Position := poScreenCenter;
  
  // Titles
  ComboTitle.Items.Add('นาย');
  ComboTitle.Items.Add('นาง');
  ComboTitle.Items.Add('นางสาว');
  ComboTitle.Items.Add('เด็กชาย');
  ComboTitle.Items.Add('เด็กหญิง');
  ComboTitle.Items.Add('ดร.');
  ComboTitle.Items.Add('Prof.');
  ComboTitle.ItemIndex := 0;
  
  RadioThai.Checked := True;
  CheckFormal.Checked := False;
  CheckEmoji.Checked  := True;
  
  LabelGreeting.Caption := '';
  LabelGreeting.Font.Size := 16;
  LabelGreeting.WordWrap := True;
  
  MemoHistory.ReadOnly := True;
  MemoHistory.ScrollBars := ssVertical;
end;

function TForm1.GenerateGreeting: String;
var
  Title, First, Last : String;
  FullName           : String;
  Hour               : Integer;
  TimeGreet          : String;
  Emoji              : String;
begin
  Title := ComboTitle.Text;
  First := Trim(EditFirstName.Text);
  Last  := Trim(EditLastName.Text);
  
  if RadioThai.Checked then
  begin
    FullName := Title + First;
    if Last <> '' then FullName := FullName + ' ' + Last;
    
    Hour := HourOf(Now);
    if Hour < 12 then TimeGreet := 'ตอนเช้า'
    else if Hour < 18 then TimeGreet := 'ตอนบ่าย'
    else TimeGreet := 'ตอนเย็น';
    
    if CheckEmoji.Checked then Emoji := ' 🙏' else Emoji := '';
    
    if CheckFormal.Checked then
      Result := Format('เรียน %s%s  ขอแสดงความเคารพ%s', [FullName, #13#10, Emoji])
    else
      Result := Format('สวัสดี%s คุณ%s!%s', [TimeGreet, FullName, Emoji]);
  end
  else
  begin
    FullName := First;
    if Last <> '' then FullName := FullName + ' ' + Last;
    
    Hour := HourOf(Now);
    if Hour < 12 then TimeGreet := 'Good morning'
    else if Hour < 18 then TimeGreet := 'Good afternoon'
    else TimeGreet := 'Good evening';
    
    if CheckEmoji.Checked then Emoji := ' 👋' else Emoji := '';
    
    if CheckFormal.Checked then
      Result := Format('Dear %s %s,%s  I hope this message finds you well.',
        [Title, FullName, #13#10])
    else
      Result := Format('%s, %s!%s', [TimeGreet, FullName, Emoji]);
  end;
end;

procedure TForm1.AddToHistory(Msg: String);
begin
  MemoHistory.Lines.Insert(0,
    '[' + FormatDateTime('hh:nn:ss', Now) + '] ' + Msg);
end;

procedure TForm1.BtnGreetClick(Sender: TObject);
begin
  if Trim(EditFirstName.Text) = '' then
  begin
    ShowMessage('กรุณากรอกชื่อ');
    EditFirstName.SetFocus;
    Exit;
  end;
  
  var Greeting := GenerateGreeting;
  LabelGreeting.Caption := Greeting;
  
  // เปลี่ยนสี
  PanelResult.Color := $00E8F5E9;  // Light green
  LabelGreeting.Font.Color := $001B5E20;
  
  AddToHistory(Greeting);
end;

procedure TForm1.BtnSaveClick(Sender: TObject);
var
  DlgSave : TSaveDialog;
  F       : TextFile;
begin
  DlgSave := TSaveDialog.Create(Self);
  try
    DlgSave.Title       := 'บันทึกคำทักทาย';
    DlgSave.DefaultExt  := 'txt';
    DlgSave.Filter      := 'Text Files (*.txt)|*.txt|All Files (*.*)|*.*';
    DlgSave.FileName    := 'greeting.txt';
    
    if DlgSave.Execute then
    begin
      AssignFile(F, DlgSave.FileName);
      Rewrite(F);
      WriteLn(F, LabelGreeting.Caption);
      WriteLn(F, '');
      WriteLn(F, '--- ประวัติการทักทาย ---');
      for var i := 0 to MemoHistory.Lines.Count - 1 do
        WriteLn(F, MemoHistory.Lines[i]);
      CloseFile(F);
      ShowMessage('บันทึกสำเร็จ: ' + DlgSave.FileName);
    end;
  finally
    DlgSave.Free;
  end;
end;

procedure TForm1.BtnClearClick(Sender: TObject);
begin
  EditFirstName.Clear;
  EditLastName.Clear;
  ComboTitle.ItemIndex    := 0;
  LabelGreeting.Caption   := '';
  PanelResult.Color       := clBtnFace;
  MemoHistory.Clear;
  EditFirstName.SetFocus;
end;

end.
```

---

## 15.14 Tips สำหรับการพัฒนา GUI

### การตั้งค่า Anchors

```pascal
// ทำให้ component ยืดตามหน้าต่าง
Edit1.Anchors := [akLeft, akTop, akRight];  // ยืดด้านขวา

// ยึดด้านล่างขวา
BtnOK.Anchors := [akRight, akBottom];
```

### การใช้ Align

```pascal
// เต็มหน้าต่าง
Memo1.Align := alClient;

// ด้านบน
PanelTop.Align := alTop;
PanelTop.Height := 60;

// ด้านล่าง
PanelBottom.Align := alBottom;
PanelBottom.Height := 40;

// เหลือพื้นที่ตรงกลาง
Memo1.Align := alClient;
```

### ตัวอย่างที่ 6: Resizable Form

```pascal
unit ResizableForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics,
  Dialogs, StdCtrls, ExtCtrls;

type
  TForm1 = class(TForm)
    PanelTop    : TPanel;
    PanelBottom : TPanel;
    MemoMain    : TMemo;
    BtnAction   : TButton;
    EditSearch  : TEdit;
    LabelStatus : TLabel;
    
    procedure FormCreate(Sender: TObject);
  private
  public
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  Caption  := 'Resizable Form';
  Width    := 600;
  Height   := 400;
  Position := poScreenCenter;
  
  // Top Panel
  PanelTop.Align  := alTop;
  PanelTop.Height := 50;
  PanelTop.Color  := clGray;
  PanelTop.BevelOuter := bvNone;
  
  // Search Edit (ในหัว)
  EditSearch.Parent  := PanelTop;
  EditSearch.Left    := 10;
  EditSearch.Top     := 10;
  EditSearch.Height  := 28;
  EditSearch.Anchors := [akLeft, akTop, akRight];  // ยืดด้านขวา
  EditSearch.Width   := PanelTop.Width - 110;
  EditSearch.PlaceholderText := 'ค้นหา...';
  
  // Button (ในหัว)
  BtnAction.Parent  := PanelTop;
  BtnAction.Caption := 'ค้นหา';
  BtnAction.Width   := 80;
  BtnAction.Height  := 28;
  BtnAction.Top     := 10;
  BtnAction.Anchors := [akTop, akRight];  // ยึดด้านขวา
  BtnAction.Left    := PanelTop.Width - 90;
  
  // Bottom Panel
  PanelBottom.Align  := alBottom;
  PanelBottom.Height := 30;
  PanelBottom.BevelOuter := bvNone;
  
  // Status Label (ใน bottom)
  LabelStatus.Parent  := PanelBottom;
  LabelStatus.Caption := 'พร้อมใช้งาน';
  LabelStatus.Left    := 5;
  LabelStatus.Top     := 7;
  LabelStatus.Anchors := [akLeft, akBottom];
  
  // Main Memo
  MemoMain.Align := alClient;  // เต็มพื้นที่ที่เหลือ
  MemoMain.ScrollBars := ssBoth;
  MemoMain.Lines.Add('ลองปรับขนาดหน้าต่าง...');
  MemoMain.Lines.Add('Memo จะขยายตาม!');
end;

end.
```

---

## 15.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง Form ที่มี:
- TLabel แสดงชื่อโปรแกรม
- TEdit รับชื่อผู้ใช้
- TButton กด OK
- แสดง "สวัสดี [ชื่อ]!" ใน TLabel อีกตัว

### แบบฝึกหัดที่ 2
สร้างเครื่องคิดเลขอย่างง่ายที่รับ 2 ตัวเลขจาก TEdit
และมีปุ่ม +, -, *, / แสดงผลใน TLabel

### แบบฝึกหัดที่ 3
สร้าง Form ลงทะเบียนที่มี:
- ชื่อ, อีเมล, รหัสผ่าน
- ตรวจสอบ input ก่อนส่ง
- แสดง MessageBox ยืนยัน

### แบบฝึกหัดที่ 4
สร้าง To-Do List app:
- TEdit รับ task
- TButton เพิ่มและลบ
- TListBox แสดง tasks
- TCheckBox สำหรับ complete

### แบบฝึกหัดที่ 5
สร้าง Unit Converter:
- TComboBox เลือกหน่วย (km, m, cm, mm)
- TEdit รับค่า
- แสดงผลการแปลงหน่วยทั้งหมด

### แบบฝึกหัดที่ 6
สร้างแบบทดสอบ Quiz:
- แสดงคำถาม 5 ข้อทีละข้อ
- TRadioButton เลือกคำตอบ
- แสดงคะแนนตอนท้าย

### แบบฝึกหัดที่ 7
สร้าง Color Picker:
- TTrackBar 3 ตัว (R, G, B)
- TPanel แสดงสีผสม
- TLabel แสดง HEX code

### แบบฝึกหัดที่ 8
สร้าง Note Pad อย่างง่าย:
- TMemo เขียนข้อความ
- File → Open, Save ด้วย TOpenDialog, TSaveDialog
- Edit → Cut, Copy, Paste

### แบบฝึกหัดที่ 9
สร้าง Countdown Timer:
- TSpinEdit รับจำนวนวินาที
- TButton Start, Pause, Reset
- TLabel แสดงเวลาที่เหลือ
- TTimer component

### แบบฝึกหัดที่ 10
สร้าง Student Grade Calculator GUI:
- รับคะแนน 5 วิชา
- คำนวณค่าเฉลี่ย
- แสดงเกรด A, B, C, D, F
- แสดงกราฟ bar chart อย่างง่าย (วาดด้วย Canvas)

---

## สรุป Lazarus Components ที่ควรรู้

| Component | ใน Palette | ใช้ทำอะไร |
|-----------|-----------|---------|
| TButton | Standard | ปุ่มกด |
| TLabel | Standard | แสดงข้อความ |
| TEdit | Standard | รับ input 1 บรรทัด |
| TMemo | Standard | รับ input หลายบรรทัด |
| TCheckBox | Standard | เลือก/ไม่เลือก |
| TRadioButton | Standard | เลือก 1 จากหลาย |
| TListBox | Standard | รายการให้เลือก |
| TComboBox | Standard | dropdown |
| TPanel | Standard | container / layout |
| TGroupBox | Standard | จัดกลุ่ม controls |
| TImage | Additional | รูปภาพ |
| TSpinEdit | Additional | ตัวเลขพร้อมลูกศร |
| TTrackBar | Additional | slider |
| TProgressBar | Additional | progress |
| TTimer | System | timer event |
| TOpenDialog | Dialogs | เปิดไฟล์ |
| TSaveDialog | Dialogs | บันทึกไฟล์ |
| TMainMenu | Standard | เมนูบาร์ |

**Best Practices GUI:**
1. ตั้งชื่อ component ให้สื่อความหมาย (BtnSave, EditName)
2. ใช้ `Anchors` หรือ `Align` เพื่อให้ responsive
3. ตรวจสอบ input ก่อนประมวลผลเสมอ
4. ใช้ `try...except` รับมือกับ errors
5. บันทึก Form layout ด้วยไฟล์ .lfm
6. ใช้ `SetFocus` เพื่อ UX ที่ดี
7. แสดง feedback ให้ผู้ใช้เสมอ (Loading..., Success, Error)
