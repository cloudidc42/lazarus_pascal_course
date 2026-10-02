# Part 17 - Events และ Event Handling

## บทนำ

Event-driven Programming เป็นรูปแบบการเขียนโปรแกรมที่การทำงานของโปรแกรมถูกควบคุมโดย Events ที่เกิดขึ้น เช่น การคลิกเมาส์ การกดแป้นพิมพ์ หรือการเปลี่ยนแปลงข้อมูล ใน Lazarus ทุก Control มี Events ที่สามารถ Handle ได้

---

## 17.1 แนวคิด Event-Driven Programming

### 17.1.1 หลักการพื้นฐาน

```pascal
// ใน Event-driven Programming:
// 1. โปรแกรมรอรับ Events
// 2. เมื่อเกิด Event จะเรียก Handler ที่ลงทะเบียนไว้
// 3. Handler ทำงานและโปรแกรมกลับมารอ

// ตัวอย่างง่ายๆ
procedure TForm1.ButtonClick(Sender: TObject);
begin
  // นี่คือ Event Handler
  // ถูกเรียกเมื่อผู้ใช้คลิกปุ่ม
  ShowMessage('ปุ่มถูกคลิก!');
end;

// การ Register Event Handler
// ทำผ่าน Object Inspector หรือ Code:
Button1.OnClick := ButtonClick;
```

### 17.1.2 Event vs Procedure ธรรมดา

```pascal
// Procedure ธรรมดา - เรียกโดยตรงจาก Code
procedure DoSomething;
begin
  // ทำงาน...
end;

// เรียกใช้
DoSomething; // เรียกตรงๆ

// Event Handler - ถูกเรียกโดย Framework เมื่อเกิด Event
procedure TForm1.Button1Click(Sender: TObject);
begin
  // ถูกเรียกเมื่อผู้ใช้คลิกปุ่ม
  // Sender คือ Object ที่ trigger Event นี้
end;
```

### 17.1.3 Event Propagation

```pascal
// บาง Events มีการ Propagate (ส่งต่อ)
// เช่น KeyDown จาก Form ถูก Propagate ไปยัง Control ที่มี Focus

// KeyPreview - ให้ Form รับ Keyboard Events ก่อน Controls
procedure TForm1.FormCreate(Sender: TObject);
begin
  KeyPreview := True; // Form จะได้รับ Key Events ก่อน
end;

procedure TForm1.FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  // ตรวจสอบ Hotkey ระดับ Form
  if (Key = VK_F5) then
    RefreshData;
end;
```

---

## 17.2 ประเภท Events ใน Lazarus

### 17.2.1 รายการ Events หลัก

```pascal
// Mouse Events
// OnClick        - คลิกซ้าย
// OnDblClick     - ดับเบิ้ลคลิก
// OnMouseDown    - กดปุ่มเมาส์
// OnMouseUp      - ปล่อยปุ่มเมาส์
// OnMouseMove    - เลื่อนเมาส์
// OnMouseEnter   - เมาส์เข้าพื้นที่ Control
// OnMouseLeave   - เมาส์ออกจากพื้นที่ Control
// OnMouseWheel   - เลื่อนล้อเมาส์

// Keyboard Events
// OnKeyDown      - กดปุ่มคีย์บอร์ด (ทุกปุ่ม)
// OnKeyUp        - ปล่อยปุ่มคีย์บอร์ด
// OnKeyPress     - กดปุ่มที่มีรหัส ASCII

// Focus Events
// OnEnter        - ได้รับ Focus
// OnExit         - เสีย Focus
// OnFocusChanged (Form level)

// Form/Window Events  
// OnCreate       - สร้าง Form
// OnDestroy      - ลบ Form
// OnShow         - แสดง Form
// OnHide         - ซ่อน Form
// OnClose        - กำลังปิด Form
// OnCloseQuery   - ถามก่อนปิด Form
// OnResize       - Form ถูก Resize
// OnMove         - Form ถูกย้าย
// OnPaint        - วาด Form ใหม่
// OnActivate     - Form กลายเป็น Active
// OnDeactivate   - Form ไม่ใช่ Active

// Data Events
// OnChange       - ข้อมูลเปลี่ยน
// OnSelect       - มีการเลือก
// OnSelectionChange - การเลือกเปลี่ยน
```

---

## 17.3 Mouse Events

### 17.3.1 OnClick และ OnDblClick

```pascal
// OnClick - Event พื้นฐานที่สุด
procedure TForm1.Button1Click(Sender: TObject);
begin
  // Sender คือ Object ที่ถูกคลิก
  if Sender is TButton then
    ShowMessage('ปุ่ม: ' + TButton(Sender).Caption + ' ถูกคลิก');
end;

// กำหนด Handler เดียวกันให้หลายปุ่ม
procedure TForm1.FormCreate(Sender: TObject);
begin
  Button1.OnClick := UniversalButtonClick;
  Button2.OnClick := UniversalButtonClick;
  Button3.OnClick := UniversalButtonClick;
end;

procedure TForm1.UniversalButtonClick(Sender: TObject);
begin
  if Sender = Button1 then
    DoAction1
  else if Sender = Button2 then
    DoAction2
  else if Sender = Button3 then
    DoAction3;
end;

// หรือใช้ Tag
procedure TForm1.SetupButtons;
begin
  Button1.Tag := 1;
  Button2.Tag := 2;
  Button3.Tag := 3;
  Button1.OnClick := TagButtonClick;
  Button2.OnClick := TagButtonClick;
  Button3.OnClick := TagButtonClick;
end;

procedure TForm1.TagButtonClick(Sender: TObject);
begin
  case TButton(Sender).Tag of
    1: DoAction1;
    2: DoAction2;
    3: DoAction3;
  end;
end;

// OnDblClick
procedure TForm1.ListBox1DblClick(Sender: TObject);
begin
  if ListBox1.ItemIndex >= 0 then
    OpenItem(ListBox1.Items[ListBox1.ItemIndex]);
end;
```

### 17.3.2 OnMouseDown, OnMouseUp

```pascal
// OnMouseDown - รับข้อมูลปุ่มเมาส์และตำแหน่ง
procedure TForm1.PanelMouseDown(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
begin
  case Button of
    mbLeft:
    begin
      // คลิกซ้าย
      FIsDrawing := True;
      FLastPoint := Point(X, Y);
    end;
    mbRight:
    begin
      // คลิกขวา - แสดง Context Menu
      PopupMenu1.Popup(
        Panel1.ClientToScreen(Point(X, Y)).X,
        Panel1.ClientToScreen(Point(X, Y)).Y
      );
    end;
    mbMiddle:
    begin
      // คลิกกลาง
      ShowMessage('คลิกกลาง ที่ (' + IntToStr(X) + ', ' + IntToStr(Y) + ')');
    end;
  end;
  
  // Shift State
  if ssCtrl in Shift then
    ShowMessage('กด Ctrl ด้วย');
  if ssAlt in Shift then
    ShowMessage('กด Alt ด้วย');
  if ssShift in Shift then
    ShowMessage('กด Shift ด้วย');
end;

// OnMouseUp
procedure TForm1.PanelMouseUp(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
begin
  if Button = mbLeft then
  begin
    FIsDrawing := False;
    // บันทึกการวาด ฯลฯ
  end;
end;
```

### 17.3.3 OnMouseMove

```pascal
// OnMouseMove - ติดตามการเคลื่อนที่ของเมาส์
procedure TForm1.PanelMouseMove(Sender: TObject; Shift: TShiftState;
  X, Y: Integer);
begin
  // แสดงตำแหน่งเมาส์
  StatusBar1.Panels[0].Text := 'X: ' + IntToStr(X) + ' Y: ' + IntToStr(Y);
  
  // วาดขณะลากเมาส์
  if FIsDrawing and (ssLeft in Shift) then
  begin
    Canvas.Pen.Color := SelectedColor;
    Canvas.Pen.Width := PenWidth;
    Canvas.MoveTo(FLastPoint.X, FLastPoint.Y);
    Canvas.LineTo(X, Y);
    FLastPoint := Point(X, Y);
  end;
end;

// ตัวอย่าง Drag and Drop ง่ายๆ
type
  TDragForm = class(TForm)
  private
    FDragging: Boolean;
    FDragStartX, FDragStartY: Integer;
    FDragControl: TControl;
    
    procedure ControlMouseDown(Sender: TObject; Button: TMouseButton;
      Shift: TShiftState; X, Y: Integer);
    procedure ControlMouseMove(Sender: TObject; Shift: TShiftState;
      X, Y: Integer);
    procedure ControlMouseUp(Sender: TObject; Button: TMouseButton;
      Shift: TShiftState; X, Y: Integer);
  end;

procedure TDragForm.ControlMouseDown(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
begin
  if Button = mbLeft then
  begin
    FDragging := True;
    FDragControl := TControl(Sender);
    FDragStartX := X;
    FDragStartY := Y;
    FDragControl.BringToFront;
  end;
end;

procedure TDragForm.ControlMouseMove(Sender: TObject; Shift: TShiftState;
  X, Y: Integer);
begin
  if FDragging and (ssLeft in Shift) and Assigned(FDragControl) then
  begin
    FDragControl.Left := FDragControl.Left + (X - FDragStartX);
    FDragControl.Top := FDragControl.Top + (Y - FDragStartY);
  end;
end;

procedure TDragForm.ControlMouseUp(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
begin
  FDragging := False;
  FDragControl := nil;
end;
```

### 17.3.4 OnMouseWheel

```pascal
// OnMouseWheel - เลื่อนล้อเมาส์
procedure TForm1.FormMouseWheel(Sender: TObject; Shift: TShiftState;
  WheelDelta: Integer; MousePos: TPoint; var Handled: Boolean);
begin
  // WheelDelta > 0 = เลื่อนขึ้น
  // WheelDelta < 0 = เลื่อนลง
  
  if ssCtrl in Shift then
  begin
    // Ctrl + Wheel = Zoom
    if WheelDelta > 0 then
      ZoomIn
    else
      ZoomOut;
    Handled := True;
  end
  else
  begin
    // Wheel ปกติ = Scroll
    ScrollBox1.VertScrollBar.Position :=
      ScrollBox1.VertScrollBar.Position - WheelDelta div 3;
    Handled := True;
  end;
end;
```

---

## 17.4 Keyboard Events

### 17.4.1 OnKeyDown

```pascal
// OnKeyDown - จับทุกปุ่ม รวม Function Keys, Arrow Keys
procedure TForm1.FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  case Key of
    VK_F1: ShowHelp;
    VK_F5: RefreshData;
    VK_DELETE: DeleteSelected;
    VK_ESCAPE: Close;
    VK_RETURN:
    begin
      if ssCtrl in Shift then
        SubmitForm
      else
        MoveToNextControl;
    end;
  end;
  
  // Ctrl + Z = Undo
  if (ssCtrl in Shift) and (Key = Ord('Z')) then
    Undo;
  
  // Ctrl + Y = Redo
  if (ssCtrl in Shift) and (Key = Ord('Y')) then
    Redo;
  
  // Ctrl + S = Save
  if (ssCtrl in Shift) and (Key = Ord('S')) then
  begin
    SaveDocument;
    Key := 0; // ป้องกันไม่ให้ส่งต่อไปยัง Control
  end;
end;

// Virtual Key Constants ที่สำคัญ
const
  // ปุ่มพิเศษ
  VK_BACK      = $08; // Backspace
  VK_TAB       = $09; // Tab
  VK_RETURN    = $0D; // Enter
  VK_ESCAPE    = $1B; // Escape
  VK_SPACE     = $20; // Space
  VK_DELETE    = $2E; // Delete
  VK_INSERT    = $2D; // Insert
  
  // Arrow Keys
  VK_LEFT      = $25;
  VK_UP        = $26;
  VK_RIGHT     = $27;
  VK_DOWN      = $28;
  
  // Function Keys
  VK_F1        = $70;
  VK_F2        = $71;
  // ...
  VK_F12       = $7B;
  
  // Modifier Keys
  VK_SHIFT     = $10;
  VK_CONTROL   = $11;
  VK_MENU      = $12; // Alt
```

### 17.4.2 OnKeyUp

```pascal
// OnKeyUp - เหมาะสำหรับ Single-press actions
procedure TForm1.EditSearchKeyUp(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  // ค้นหาทุกครั้งที่พิมพ์ (หลังปล่อยปุ่ม)
  if Key <> VK_RETURN then
    SearchRealTime(EditSearch.Text);
end;
```

### 17.4.3 OnKeyPress

```pascal
// OnKeyPress - สำหรับปุ่มที่มี ASCII Code
// ดีสำหรับกรองตัวอักษรที่รับได้

// รับเฉพาะตัวเลข
procedure TForm1.EditNumberKeyPress(Sender: TObject; var Key: Char);
begin
  if not (Key in ['0'..'9', #8, #13]) then
    Key := #0;
end;

// รับเฉพาะตัวเลขและจุดทศนิยม
procedure TForm1.EditDecimalKeyPress(Sender: TObject; var Key: Char);
var
  edit: TEdit;
begin
  edit := TEdit(Sender);
  if not (Key in ['0'..'9', '.', #8, #13]) then
    Key := #0
  else if (Key = '.') and (Pos('.', edit.Text) > 0) then
    Key := #0; // ไม่ให้มีจุดสองครั้ง
end;

// รับเฉพาะตัวอักษรภาษาอังกฤษ
procedure TForm1.EditAlphaKeyPress(Sender: TObject; var Key: Char);
begin
  if not (Key in ['A'..'Z', 'a'..'z', #8, #13, ' ']) then
    Key := #0;
end;

// แปลงเป็นตัวพิมพ์ใหญ่อัตโนมัติ
procedure TForm1.EditUpperCaseKeyPress(Sender: TObject; var Key: Char);
begin
  Key := UpCase(Key);
end;

// จับ Enter เพื่อ Move Focus
procedure TForm1.EditEnterKeyPress(Sender: TObject; var Key: Char);
begin
  if Key = #13 then
  begin
    Key := #0;
    SelectNext(TWinControl(Sender), True, True);
  end;
end;
```

---

## 17.5 Focus Events

### 17.5.1 OnEnter และ OnExit

```pascal
// OnEnter - เมื่อ Control ได้รับ Focus
procedure TForm1.EditFieldEnter(Sender: TObject);
var
  edit: TEdit;
begin
  if Sender is TEdit then
  begin
    edit := TEdit(Sender);
    edit.Color := $00FFFFE0; // สีเหลืองอ่อน
    edit.SelectAll;          // เลือกข้อความทั้งหมดเมื่อได้ Focus
  end;
end;

// OnExit - เมื่อ Control เสีย Focus
procedure TForm1.EditFieldExit(Sender: TObject);
var
  edit: TEdit;
begin
  if Sender is TEdit then
  begin
    edit := TEdit(Sender);
    edit.Color := clWhite;   // คืนสีเดิม
    ValidateField(edit);     // Validate เมื่อออกจาก Field
  end;
end;

// การ Validate เมื่อ Exit
procedure TForm1.ValidateField(edit: TEdit);
begin
  if edit = EditEmail then
  begin
    if not IsValidEmail(edit.Text) then
    begin
      edit.Color := $00CCFFFF; // แดงอ่อน
      ShowMessage('อีเมลไม่ถูกต้อง');
      edit.SetFocus;
    end;
  end
  else if edit = EditAge then
  begin
    if StrToIntDef(edit.Text, -1) < 0 then
    begin
      edit.Color := $00CCFFFF;
      ShowMessage('อายุต้องเป็นตัวเลขบวก');
      edit.SetFocus;
    end;
  end;
end;
```

### 17.5.2 OnChange

```pascal
// OnChange - เมื่อค่าใน Control เปลี่ยน

// TEdit OnChange - เปลี่ยนแปลงทุกครั้งที่พิมพ์
procedure TForm1.EditSearch TextChange(Sender: TObject);
begin
  // Real-time Search
  PerformSearch(EditSearch.Text);
  
  // อัปเดตปุ่มตามข้อมูล
  ButtonSearch.Enabled := Length(EditSearch.Text) > 0;
end;

// TComboBox OnChange
procedure TForm1.ComboBoxCountryChange(Sender: TObject);
begin
  // โหลดจังหวัดตามประเทศที่เลือก
  LoadCities(ComboBoxCountry.Text);
end;

// TCheckBox OnClick (ใช้แทน OnChange สำหรับ CheckBox)
procedure TForm1.CheckBoxRememberClick(Sender: TObject);
begin
  if CheckBoxRemember.Checked then
    SaveCredentials
  else
    ClearSavedCredentials;
end;

// TTrackBar OnChange
procedure TForm1.TrackBarVolumeChange(Sender: TObject);
begin
  LabelVolume.Caption := IntToStr(TrackBarVolume.Position) + '%';
  SetAudioVolume(TrackBarVolume.Position);
end;
```

---

## 17.6 Form Events

### 17.6.1 OnCreate และ OnDestroy

```pascal
// OnCreate - เมื่อ Form ถูกสร้าง (ใช้สำหรับ Initialize)
procedure TMainForm.FormCreate(Sender: TObject);
begin
  // Initialize ตัวแปร
  FDocumentList := TStringList.Create;
  FSettings := TAppSettings.Create;
  
  // โหลดการตั้งค่า
  LoadSettings;
  
  // Setup Controls
  SetupControls;
  
  // Register Services
  FTimer := TTimer.Create(Self);
  FTimer.Interval := 1000;
  FTimer.OnTimer := TimerTick;
  FTimer.Enabled := True;
  
  // แสดงข้อความ Welcome
  StatusBar1.Panels[0].Text := 'ยินดีต้อนรับ!';
end;

// OnDestroy - เมื่อ Form ถูกลบ (ทำ Cleanup)
procedure TMainForm.FormDestroy(Sender: TObject);
begin
  // บันทึกการตั้งค่า
  SaveSettings;
  
  // Free Objects ที่สร้างไว้
  FDocumentList.Free;
  FSettings.Free;
  
  // Cleanup Resources
  FTimer.Free;
end;
```

### 17.6.2 OnShow, OnHide

```pascal
// OnShow - เมื่อ Form แสดงขึ้นมา
procedure TDataForm.FormShow(Sender: TObject);
begin
  // โหลดข้อมูลล่าสุด
  RefreshData;
  
  // Focus ที่ Control แรก
  EditSearch.SetFocus;
  
  // อัปเดต Title
  Caption := 'ข้อมูล - ' + FormatDateTime('dd/mm/yyyy', Now);
end;

// OnHide - เมื่อ Form ถูกซ่อน
procedure TDataForm.FormHide(Sender: TObject);
begin
  // หยุด Timer ที่ไม่จำเป็น
  RefreshTimer.Enabled := False;
end;
```

### 17.6.3 OnClose และ OnCloseQuery

```pascal
// OnCloseQuery - ถามก่อนปิด Form
procedure TEditorForm.FormCloseQuery(Sender: TObject; var CanClose: Boolean);
begin
  CanClose := True;
  
  if FDocumentModified then
  begin
    case MessageDlg('เอกสารมีการเปลี่ยนแปลง' + #13#10 +
                    'ต้องการบันทึกหรือไม่?',
                    mtConfirmation, [mbYes, mbNo, mbCancel], 0) of
      mrYes:
      begin
        if not SaveDocument then
          CanClose := False; // บันทึกไม่สำเร็จ ไม่ปิด
      end;
      mrCancel:
        CanClose := False;   // ยกเลิก ไม่ปิด
    end;
  end;
end;

// OnClose - เมื่อ Form กำลังจะปิด
procedure TMainForm.FormClose(Sender: TObject; var CloseAction: TCloseAction);
begin
  // CloseAction สามารถเปลี่ยนได้:
  // caFree    - Free Form (ลบออกจาก Memory)
  // caHide    - ซ่อน Form (ยังอยู่ใน Memory)
  // caMinimize - Minimize Form
  // caNone    - ไม่ทำอะไร (ยกเลิกการปิด)
  
  CloseAction := caFree; // ค่าเริ่มต้น
  
  // บันทึก Position
  SaveWindowPosition;
end;
```

### 17.6.4 OnResize และ OnMove

```pascal
// OnResize - เมื่อขนาด Form เปลี่ยน
procedure TMainForm.FormResize(Sender: TObject);
begin
  // ปรับ Layout อัตโนมัติ
  PanelLeft.Height := ClientHeight - PanelToolbar.Height - StatusBar1.Height;
  MemoContent.Height := PanelLeft.Height;
  
  // อัปเดต Status Bar
  StatusBar1.Panels[1].Text := 
    IntToStr(ClientWidth) + 'x' + IntToStr(ClientHeight);
end;

// OnMove - เมื่อ Form ถูกย้าย
procedure TMainForm.FormMove(Sender: TObject);
begin
  // บันทึกตำแหน่ง
  FWindowLeft := Left;
  FWindowTop := Top;
end;
```

---

## 17.7 การสร้าง Custom Events

### 17.7.1 นิยาม Custom Event Type

```pascal
// กำหนด Event Type ใหม่
type
  // Simple Event (ไม่มี Parameter เพิ่ม)
  TSimpleEvent = procedure(Sender: TObject) of object;
  
  // Event ที่มี Parameter
  TDataChangedEvent = procedure(Sender: TObject; const NewData: string) of object;
  
  // Event ที่มี หลาย Parameter
  TPositionChangedEvent = procedure(Sender: TObject; X, Y: Integer) of object;
  
  // Event ที่คืนค่า (ผ่าน var parameter)
  TValidateEvent = procedure(Sender: TObject; const Value: string;
    var IsValid: Boolean; var ErrorMsg: string) of object;

// Class ที่มี Custom Event
type
  TCustomControl = class(TControl)
  private
    FOnDataChanged: TDataChangedEvent;
    FOnValidate: TValidateEvent;
    FData: string;
    
    procedure SetData(const Value: string);
    
  protected
    procedure DoDataChanged(const NewData: string); virtual;
    function DoValidate(const Value: string): Boolean; virtual;
    
  public
    property Data: string read FData write SetData;
    
  published
    property OnDataChanged: TDataChangedEvent read FOnDataChanged
      write FOnDataChanged;
    property OnValidate: TValidateEvent read FOnValidate write FOnValidate;
  end;

implementation

procedure TCustomControl.SetData(const Value: string);
begin
  if FData <> Value then
  begin
    if DoValidate(Value) then
    begin
      FData := Value;
      DoDataChanged(Value);
    end;
  end;
end;

procedure TCustomControl.DoDataChanged(const NewData: string);
begin
  // Fire Event ถ้ามี Handler
  if Assigned(FOnDataChanged) then
    FOnDataChanged(Self, NewData);
end;

function TCustomControl.DoValidate(const Value: string): Boolean;
var
  ErrorMsg: string;
begin
  Result := True;
  ErrorMsg := '';
  
  if Assigned(FOnValidate) then
    FOnValidate(Self, Value, Result, ErrorMsg);
    
  if not Result then
    ShowMessage('ข้อผิดพลาด: ' + ErrorMsg);
end;
```

### 17.7.2 การใช้ Custom Event

```pascal
// การใช้งาน Custom Event
procedure TMainForm.FormCreate(Sender: TObject);
begin
  MyCustomControl.OnDataChanged := HandleDataChanged;
  MyCustomControl.OnValidate := HandleValidate;
end;

procedure TMainForm.HandleDataChanged(Sender: TObject; const NewData: string);
begin
  LabelStatus.Caption := 'ข้อมูลเปลี่ยนเป็น: ' + NewData;
  // อัปเดต Database
  SaveToDatabase(NewData);
end;

procedure TMainForm.HandleValidate(Sender: TObject; const Value: string;
  var IsValid: Boolean; var ErrorMsg: string);
begin
  IsValid := Length(Value) >= 3;
  if not IsValid then
    ErrorMsg := 'ข้อมูลต้องมีอย่างน้อย 3 ตัวอักษร';
end;
```

### 17.7.3 Event ที่ใช้ TNotifyEvent

```pascal
// TNotifyEvent เป็น Event Type ที่ใช้บ่อยที่สุด
// นิยาม: procedure(Sender: TObject) of object;

type
  TDataManager = class
  private
    FOnDataLoaded: TNotifyEvent;
    FOnError: TNotifyEvent;
    FLastError: string;
    
  public
    procedure LoadData(const FileName: string);
    property OnDataLoaded: TNotifyEvent read FOnDataLoaded write FOnDataLoaded;
    property OnError: TNotifyEvent read FOnError write FOnError;
    property LastError: string read FLastError;
  end;

procedure TDataManager.LoadData(const FileName: string);
begin
  try
    // โหลดข้อมูล
    // ...
    
    // แจ้ง Success
    if Assigned(FOnDataLoaded) then
      FOnDataLoaded(Self);
  except
    on E: Exception do
    begin
      FLastError := E.Message;
      if Assigned(FOnError) then
        FOnError(Self);
    end;
  end;
end;

// การใช้งาน
procedure TMainForm.FormCreate(Sender: TObject);
begin
  FDataManager := TDataManager.Create;
  FDataManager.OnDataLoaded := DataLoadedHandler;
  FDataManager.OnError := DataErrorHandler;
end;

procedure TMainForm.DataLoadedHandler(Sender: TObject);
begin
  ShowMessage('โหลดข้อมูลสำเร็จ');
  DisplayData;
end;

procedure TMainForm.DataErrorHandler(Sender: TObject);
begin
  ShowMessage('เกิดข้อผิดพลาด: ' + FDataManager.LastError);
end;
```

---

## 17.8 Event Delegation

### 17.8.1 Handler เดียวสำหรับหลาย Controls

```pascal
// Delegate Event จาก Controls หลายตัวไปยัง Handler เดียว
procedure TCalculatorForm.FormCreate(Sender: TObject);
var
  i: Integer;
begin
  // กำหนด OnClick ให้ปุ่มทุกตัว
  for i := 0 to ControlCount - 1 do
    if Controls[i] is TButton then
      TButton(Controls[i]).OnClick := CalculatorButtonClick;
end;

procedure TCalculatorForm.CalculatorButtonClick(Sender: TObject);
var
  caption: string;
begin
  caption := TButton(Sender).Caption;
  
  if caption[1] in ['0'..'9'] then
    HandleDigit(caption)
  else if caption = '+' then
    HandleOperator(opAdd)
  else if caption = '-' then
    HandleOperator(opSubtract)
  else if caption = '=' then
    HandleEquals;
end;
```

### 17.8.2 ใช้ Interface สำหรับ Event Delegation

```pascal
// Interface สำหรับ Event Handling
type
  IDataConsumer = interface
    ['{12345678-1234-1234-1234-123456789012}']
    procedure OnDataReceived(const Data: string);
    procedure OnError(const ErrMsg: string);
  end;

// Producer ที่ส่ง Events ผ่าน Interface
type
  TDataProducer = class
  private
    FConsumers: TInterfaceList;
  public
    constructor Create;
    destructor Destroy; override;
    procedure AddConsumer(Consumer: IDataConsumer);
    procedure RemoveConsumer(Consumer: IDataConsumer);
    procedure FetchData;
  end;

procedure TDataProducer.FetchData;
var
  i: Integer;
  data: string;
begin
  try
    data := GetDataFromSource;
    
    for i := 0 to FConsumers.Count - 1 do
      IDataConsumer(FConsumers[i]).OnDataReceived(data);
  except
    on E: Exception do
      for i := 0 to FConsumers.Count - 1 do
        IDataConsumer(FConsumers[i]).OnError(E.Message);
  end;
end;

// Form ที่ Implement Interface
type
  TDataForm = class(TForm, IDataConsumer)
  public
    procedure OnDataReceived(const Data: string);
    procedure OnError(const ErrMsg: string);
  end;

procedure TDataForm.OnDataReceived(const Data: string);
begin
  MemoData.Lines.Add(Data);
end;

procedure TDataForm.OnError(const ErrMsg: string);
begin
  ShowMessage('ข้อผิดพลาด: ' + ErrMsg);
end;
```

---

## 17.9 Event Parameters อย่างละเอียด

### 17.9.1 Sender: TObject

```pascal
// Sender คือ Object ที่ trigger Event
// ใช้เพื่อหาว่า Control ไหนส่ง Event มา

procedure TForm1.GenericClickHandler(Sender: TObject);
begin
  // ตรวจสอบประเภทของ Sender
  if Sender is TButton then
  begin
    ShowMessage('Button: ' + TButton(Sender).Caption + ' clicked');
  end
  else if Sender is TMenuItem then
  begin
    ShowMessage('MenuItem: ' + TMenuItem(Sender).Caption + ' clicked');
  end
  else if Sender is TLabel then
  begin
    ShowMessage('Label: ' + TLabel(Sender).Caption + ' clicked');
  end;
  
  // ใช้ Name ของ Component
  if TComponent(Sender).Name = 'ButtonSave' then
    SaveData
  else if TComponent(Sender).Name = 'ButtonLoad' then
    LoadData;
  
  // ใช้ Tag
  case TControl(Sender).Tag of
    1: Action1;
    2: Action2;
    3: Action3;
  end;
end;
```

### 17.9.2 Key Parameters

```pascal
// Key: Word - Virtual Key Code (สำหรับ OnKeyDown, OnKeyUp)
// Key: Char - ASCII Character (สำหรับ OnKeyPress)

procedure TForm1.EditKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
begin
  // ดัก Keyboard Shortcuts
  if (ssCtrl in Shift) then
  begin
    case Key of
      Ord('A'): // Ctrl+A = Select All
      begin
        if Sender is TEdit then
          TEdit(Sender).SelectAll;
        Key := 0;
      end;
      Ord('C'): // Ctrl+C = Copy
      begin
        // ไม่ต้อง Handle - Browser/OS จัดการเอง
      end;
      Ord('Z'): // Ctrl+Z = Undo
      begin
        UndoLastAction;
        Key := 0; // ป้องกัน Default Action
      end;
    end;
  end;
  
  // Arrow Keys สำหรับ Custom Navigation
  case Key of
    VK_UP:
    begin
      SelectPreviousItem;
      Key := 0;
    end;
    VK_DOWN:
    begin
      SelectNextItem;
      Key := 0;
    end;
  end;
end;
```

### 17.9.3 Mouse Parameters

```pascal
// Button: TMouseButton - ปุ่มเมาส์ที่กด
// Shift: TShiftState  - Modifier Keys
// X, Y: Integer       - ตำแหน่งเมาส์ใน Client Coordinates

procedure TForm1.PanelMouseDown(Sender: TObject; Button: TMouseButton;
  Shift: TShiftState; X, Y: Integer);
var
  screenPos: TPoint;
begin
  // แปลง Client Coordinates เป็น Screen Coordinates
  screenPos := Panel1.ClientToScreen(Point(X, Y));
  
  // ตรวจสอบปุ่มเมาส์
  case Button of
    mbLeft:   HandleLeftClick(X, Y);
    mbRight:  ShowContextMenu(screenPos.X, screenPos.Y);
    mbMiddle: HandleMiddleClick(X, Y);
  end;
  
  // ตรวจสอบ Modifier Keys
  if ssDouble in Shift then
    HandleDoubleClick(X, Y);  // Double Click
    
  if ssCtrl in Shift then
    HandleCtrlClick(X, Y);    // Ctrl + Click
    
  if ssShift in Shift then
    HandleShiftClick(X, Y);   // Shift + Click (เลือกหลายรายการ)
end;
```

---

## 17.10 โปรแกรมตัวอย่าง: Drawing App

```pascal
unit DrawingApp;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  ExtCtrls, StdCtrls, ComCtrls, Menus, Spin;

type
  TDrawTool = (dtPen, dtLine, dtRectangle, dtEllipse, dtEraser);
  
  TDrawingApp = class(TForm)
    PanelToolbar: TPanel;
    PanelCanvas: TPanel;
    StatusBar: TStatusBar;
    PopupMenuCanvas: TPopupMenu;
    
    // Tool Buttons
    SpeedButtonPen: TSpeedButton;
    SpeedButtonLine: TSpeedButton;
    SpeedButtonRect: TSpeedButton;
    SpeedButtonEllipse: TSpeedButton;
    SpeedButtonEraser: TSpeedButton;
    
    // Controls
    SpinEditPenWidth: TSpinEdit;
    LabelPenWidth: TLabel;
    PanelColorPreview: TPanel;
    ButtonChooseColor: TButton;
    CheckBoxFill: TCheckBox;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure PanelCanvasMouseDown(Sender: TObject; Button: TMouseButton;
      Shift: TShiftState; X, Y: Integer);
    procedure PanelCanvasMouseMove(Sender: TObject; Shift: TShiftState;
      X, Y: Integer);
    procedure PanelCanvasMouseUp(Sender: TObject; Button: TMouseButton;
      Shift: TShiftState; X, Y: Integer);
    procedure PanelCanvasPaint(Sender: TObject);
    procedure ToolButtonClick(Sender: TObject);
    procedure ButtonChooseColorClick(Sender: TObject);
    procedure SpinEditPenWidthChange(Sender: TObject);
    procedure MenuItemClearClick(Sender: TObject);
    procedure MenuItemSaveClick(Sender: TObject);
    procedure FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    
  private
    FBitmap: TBitmap;         // Offscreen Buffer
    FOverlayBitmap: TBitmap;  // สำหรับ Preview ขณะวาด
    
    FCurrentTool: TDrawTool;
    FDrawColor: TColor;
    FPenWidth: Integer;
    FFillShape: Boolean;
    
    FIsDrawing: Boolean;
    FStartPoint: TPoint;
    FCurrentPoint: TPoint;
    FLastPoint: TPoint;
    
    FDrawHistory: TList;   // สำหรับ Undo
    FHistoryIndex: Integer;
    
    procedure SetupCanvas;
    procedure SetCurrentTool(Tool: TDrawTool);
    procedure DrawPenStroke(X, Y: Integer);
    procedure DrawPreview;
    procedure CommitDrawing;
    procedure SaveToHistory;
    procedure Undo;
    procedure ClearCanvas;
    function GetCanvasPoint(X, Y: Integer): TPoint;
    procedure UpdateStatus(X, Y: Integer);
  end;

var
  DrawingApp: TDrawingApp;

implementation

uses
  Dialogs, LCLType;

{$R *.lfm}

procedure TDrawingApp.FormCreate(Sender: TObject);
begin
  Caption := 'โปรแกรมวาดภาพ';
  Width := 900;
  Height := 650;
  Position := poScreenCenter;
  KeyPreview := True;
  
  // Initialize Bitmaps
  FBitmap := TBitmap.Create;
  FBitmap.Width := PanelCanvas.ClientWidth;
  FBitmap.Height := PanelCanvas.ClientHeight;
  FBitmap.Canvas.Brush.Color := clWhite;
  FBitmap.Canvas.FillRect(Rect(0, 0, FBitmap.Width, FBitmap.Height));
  
  FOverlayBitmap := TBitmap.Create;
  FOverlayBitmap.Width := FBitmap.Width;
  FOverlayBitmap.Height := FBitmap.Height;
  
  // Initialize State
  FCurrentTool := dtPen;
  FDrawColor := clBlack;
  FPenWidth := 2;
  FFillShape := False;
  FIsDrawing := False;
  
  // Initialize History
  FDrawHistory := TList.Create;
  FHistoryIndex := -1;
  
  SetupCanvas;
  SetCurrentTool(dtPen);
  SaveToHistory;
  
  // Setup Color Preview
  PanelColorPreview.Color := FDrawColor;
  PanelColorPreview.Hint := 'สีปัจจุบัน - คลิกเพื่อเลือก';
  PanelColorPreview.ShowHint := True;
  
  // Status
  StatusBar.Panels[0].Text := 'เลือกเครื่องมือและเริ่มวาด';
end;

procedure TDrawingApp.FormDestroy(Sender: TObject);
var
  i: Integer;
begin
  FBitmap.Free;
  FOverlayBitmap.Free;
  
  for i := 0 to FDrawHistory.Count - 1 do
    TBitmap(FDrawHistory[i]).Free;
  FDrawHistory.Free;
end;

procedure TDrawingApp.SetupCanvas;
begin
  PanelCanvas.OnMouseDown := PanelCanvasMouseDown;
  PanelCanvas.OnMouseMove := PanelCanvasMouseMove;
  PanelCanvas.OnMouseUp := PanelCanvasMouseUp;
  PanelCanvas.OnPaint := PanelCanvasPaint;
  PanelCanvas.Cursor := crCross;
end;

procedure TDrawingApp.SetCurrentTool(Tool: TDrawTool);
begin
  FCurrentTool := Tool;
  
  // อัปเดต Tool Buttons
  SpeedButtonPen.Down := Tool = dtPen;
  SpeedButtonLine.Down := Tool = dtLine;
  SpeedButtonRect.Down := Tool = dtRectangle;
  SpeedButtonEllipse.Down := Tool = dtEllipse;
  SpeedButtonEraser.Down := Tool = dtEraser;
  
  // เปลี่ยน Cursor
  case Tool of
    dtEraser: PanelCanvas.Cursor := crCross;
    else PanelCanvas.Cursor := crCross;
  end;
  
  // อัปเดต Status
  case Tool of
    dtPen: StatusBar.Panels[0].Text := 'เครื่องมือ: ดินสอ - ลากเพื่อวาดเส้น';
    dtLine: StatusBar.Panels[0].Text := 'เครื่องมือ: เส้นตรง - คลิกและลากเพื่อวาด';
    dtRectangle: StatusBar.Panels[0].Text := 'เครื่องมือ: สี่เหลี่ยม - คลิกและลากเพื่อวาด';
    dtEllipse: StatusBar.Panels[0].Text := 'เครื่องมือ: วงรี - คลิกและลากเพื่อวาด';
    dtEraser: StatusBar.Panels[0].Text := 'เครื่องมือ: ยางลบ - ลากเพื่อลบ';
  end;
end;

function TDrawingApp.GetCanvasPoint(X, Y: Integer): TPoint;
begin
  Result := Point(X, Y);
end;

procedure TDrawingApp.UpdateStatus(X, Y: Integer);
begin
  StatusBar.Panels[1].Text := 'X: ' + IntToStr(X) + ' Y: ' + IntToStr(Y);
end;

procedure TDrawingApp.PanelCanvasMouseDown(Sender: TObject;
  Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  if Button = mbLeft then
  begin
    FIsDrawing := True;
    FStartPoint := GetCanvasPoint(X, Y);
    FCurrentPoint := FStartPoint;
    FLastPoint := FStartPoint;
    
    if FCurrentTool = dtPen then
      DrawPenStroke(X, Y);
  end;
end;

procedure TDrawingApp.PanelCanvasMouseMove(Sender: TObject;
  Shift: TShiftState; X, Y: Integer);
begin
  FCurrentPoint := GetCanvasPoint(X, Y);
  UpdateStatus(X, Y);
  
  if FIsDrawing and (ssLeft in Shift) then
  begin
    case FCurrentTool of
      dtPen, dtEraser:
        DrawPenStroke(X, Y);
      dtLine, dtRectangle, dtEllipse:
        DrawPreview;
    end;
    FLastPoint := FCurrentPoint;
  end;
end;

procedure TDrawingApp.PanelCanvasMouseUp(Sender: TObject;
  Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  if (Button = mbLeft) and FIsDrawing then
  begin
    FIsDrawing := False;
    FCurrentPoint := GetCanvasPoint(X, Y);
    
    case FCurrentTool of
      dtLine, dtRectangle, dtEllipse:
        CommitDrawing;
    end;
    
    SaveToHistory;
    PanelCanvas.Invalidate;
  end;
end;

procedure TDrawingApp.DrawPenStroke(X, Y: Integer);
begin
  with FBitmap.Canvas do
  begin
    if FCurrentTool = dtEraser then
    begin
      Pen.Color := clWhite;
      Pen.Width := FPenWidth * 5; // ยางลบใหญ่กว่า
    end
    else
    begin
      Pen.Color := FDrawColor;
      Pen.Width := FPenWidth;
    end;
    
    Pen.Style := psSolid;
    MoveTo(FLastPoint.X, FLastPoint.Y);
    LineTo(X, Y);
  end;
  
  FLastPoint := Point(X, Y);
  PanelCanvas.Invalidate;
end;

procedure TDrawingApp.DrawPreview;
begin
  // คัดลอก Bitmap หลักไปยัง Overlay
  FOverlayBitmap.Canvas.Draw(0, 0, FBitmap);
  
  // วาด Preview บน Overlay
  with FOverlayBitmap.Canvas do
  begin
    Pen.Color := FDrawColor;
    Pen.Width := FPenWidth;
    Pen.Style := psDash; // เส้นประสำหรับ Preview
    
    if FFillShape then
    begin
      Brush.Color := FDrawColor;
      Brush.Style := bsSolid;
    end
    else
      Brush.Style := bsClear;
    
    case FCurrentTool of
      dtLine:
        begin
          MoveTo(FStartPoint.X, FStartPoint.Y);
          LineTo(FCurrentPoint.X, FCurrentPoint.Y);
        end;
      dtRectangle:
        Rectangle(
          Min(FStartPoint.X, FCurrentPoint.X),
          Min(FStartPoint.Y, FCurrentPoint.Y),
          Max(FStartPoint.X, FCurrentPoint.X),
          Max(FStartPoint.Y, FCurrentPoint.Y)
        );
      dtEllipse:
        Ellipse(
          Min(FStartPoint.X, FCurrentPoint.X),
          Min(FStartPoint.Y, FCurrentPoint.Y),
          Max(FStartPoint.X, FCurrentPoint.X),
          Max(FStartPoint.Y, FCurrentPoint.Y)
        );
    end;
  end;
  
  PanelCanvas.Invalidate;
end;

procedure TDrawingApp.CommitDrawing;
begin
  with FBitmap.Canvas do
  begin
    Pen.Color := FDrawColor;
    Pen.Width := FPenWidth;
    Pen.Style := psSolid;
    
    if FFillShape then
    begin
      Brush.Color := FDrawColor;
      Brush.Style := bsSolid;
    end
    else
      Brush.Style := bsClear;
    
    case FCurrentTool of
      dtLine:
        begin
          MoveTo(FStartPoint.X, FStartPoint.Y);
          LineTo(FCurrentPoint.X, FCurrentPoint.Y);
        end;
      dtRectangle:
        Rectangle(
          Min(FStartPoint.X, FCurrentPoint.X),
          Min(FStartPoint.Y, FCurrentPoint.Y),
          Max(FStartPoint.X, FCurrentPoint.X),
          Max(FStartPoint.Y, FCurrentPoint.Y)
        );
      dtEllipse:
        Ellipse(
          Min(FStartPoint.X, FCurrentPoint.X),
          Min(FStartPoint.Y, FCurrentPoint.Y),
          Max(FStartPoint.X, FCurrentPoint.X),
          Max(FStartPoint.Y, FCurrentPoint.Y)
        );
    end;
  end;
end;

procedure TDrawingApp.SaveToHistory;
var
  snapshot: TBitmap;
  i: Integer;
begin
  // ลบ History หลัง Current Index
  for i := FDrawHistory.Count - 1 downto FHistoryIndex + 1 do
  begin
    TBitmap(FDrawHistory[i]).Free;
    FDrawHistory.Delete(i);
  end;
  
  // เก็บสูงสุด 20 ขั้นตอน
  if FDrawHistory.Count >= 20 then
  begin
    TBitmap(FDrawHistory[0]).Free;
    FDrawHistory.Delete(0);
    Dec(FHistoryIndex);
  end;
  
  // บันทึก Snapshot
  snapshot := TBitmap.Create;
  snapshot.Assign(FBitmap);
  FDrawHistory.Add(snapshot);
  FHistoryIndex := FDrawHistory.Count - 1;
end;

procedure TDrawingApp.Undo;
begin
  if FHistoryIndex > 0 then
  begin
    Dec(FHistoryIndex);
    FBitmap.Assign(TBitmap(FDrawHistory[FHistoryIndex]));
    PanelCanvas.Invalidate;
    StatusBar.Panels[0].Text := 'ย้อนกลับสำเร็จ';
  end
  else
    StatusBar.Panels[0].Text := 'ไม่มีการกระทำที่จะย้อนกลับ';
end;

procedure TDrawingApp.ClearCanvas;
begin
  FBitmap.Canvas.Brush.Color := clWhite;
  FBitmap.Canvas.FillRect(Rect(0, 0, FBitmap.Width, FBitmap.Height));
  SaveToHistory;
  PanelCanvas.Invalidate;
end;

procedure TDrawingApp.PanelCanvasPaint(Sender: TObject);
begin
  if FIsDrawing and (FCurrentTool in [dtLine, dtRectangle, dtEllipse]) then
    PanelCanvas.Canvas.Draw(0, 0, FOverlayBitmap)
  else
    PanelCanvas.Canvas.Draw(0, 0, FBitmap);
end;

procedure TDrawingApp.ToolButtonClick(Sender: TObject);
begin
  if Sender = SpeedButtonPen then SetCurrentTool(dtPen)
  else if Sender = SpeedButtonLine then SetCurrentTool(dtLine)
  else if Sender = SpeedButtonRect then SetCurrentTool(dtRectangle)
  else if Sender = SpeedButtonEllipse then SetCurrentTool(dtEllipse)
  else if Sender = SpeedButtonEraser then SetCurrentTool(dtEraser);
end;

procedure TDrawingApp.ButtonChooseColorClick(Sender: TObject);
var
  ColorDlg: TColorDialog;
begin
  ColorDlg := TColorDialog.Create(Self);
  try
    ColorDlg.Color := FDrawColor;
    if ColorDlg.Execute then
    begin
      FDrawColor := ColorDlg.Color;
      PanelColorPreview.Color := FDrawColor;
    end;
  finally
    ColorDlg.Free;
  end;
end;

procedure TDrawingApp.SpinEditPenWidthChange(Sender: TObject);
begin
  FPenWidth := SpinEditPenWidth.Value;
end;

procedure TDrawingApp.MenuItemClearClick(Sender: TObject);
begin
  if MessageDlg('ต้องการล้างภาพทั้งหมดหรือไม่?',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
    ClearCanvas;
end;

procedure TDrawingApp.MenuItemSaveClick(Sender: TObject);
var
  SaveDlg: TSaveDialog;
begin
  SaveDlg := TSaveDialog.Create(Self);
  try
    SaveDlg.Title := 'บันทึกภาพ';
    SaveDlg.Filter := 'PNG Image|*.png|BMP Image|*.bmp|JPEG Image|*.jpg';
    SaveDlg.DefaultExt := 'png';
    
    if SaveDlg.Execute then
    begin
      FBitmap.SaveToFile(SaveDlg.FileName);
      StatusBar.Panels[0].Text := 'บันทึกแล้ว: ' + SaveDlg.FileName;
    end;
  finally
    SaveDlg.Free;
  end;
end;

procedure TDrawingApp.FormKeyDown(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  // Keyboard Shortcuts
  if ssCtrl in Shift then
  begin
    case Key of
      Ord('Z'): Undo;
      Ord('S'): MenuItemSaveClick(nil);
    end;
  end;
  
  // Tool shortcuts
  case Key of
    Ord('P'): SetCurrentTool(dtPen);
    Ord('L'): SetCurrentTool(dtLine);
    Ord('R'): SetCurrentTool(dtRectangle);
    Ord('E'): SetCurrentTool(dtEllipse);
    Ord('X'): SetCurrentTool(dtEraser);
    VK_DELETE: 
      if MessageDlg('ล้างภาพ?', mtConfirmation, [mbYes, mbNo], 0) = mrYes then
        ClearCanvas;
  end;
end;

end.
```

---

## 17.11 โปรแกรมตัวอย่าง: Keyboard Input Tracker

```pascal
unit KeyboardTracker;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, LCLType;

type
  TKeyEvent = record
    EventType: string;  // 'Down', 'Up', 'Press'
    KeyCode: Word;
    KeyChar: Char;
    Shift: TShiftState;
    Timestamp: TDateTime;
  end;

  TKeyboardTrackerForm = class(TForm)
    PanelInput: TPanel;
    EditTest: TEdit;
    MemoKeyEvents: TMemo;
    LabelInstruction: TLabel;
    PanelInfo: TPanel;
    LabelCurrentKey: TLabel;
    LabelLastKey: TLabel;
    LabelKeyCount: TLabel;
    ButtonClear: TButton;
    ButtonCopyLog: TButton;
    CheckBoxCapture: TCheckBox;
    PanelHotkeys: TPanel;
    GroupBoxHotkeys: TGroupBox;
    
    procedure FormCreate(Sender: TObject);
    procedure EditTestKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure EditTestKeyUp(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure EditTestKeyPress(Sender: TObject; var Key: Char);
    procedure FormKeyDown(Sender: TObject; var Key: Word; Shift: TShiftState);
    procedure ButtonClearClick(Sender: TObject);
    procedure ButtonCopyLogClick(Sender: TObject);
    
  private
    FKeyEvents: array of TKeyEvent;
    FKeyCount: Integer;
    FTotalKeyPresses: Integer;
    
    procedure LogKeyEvent(const EventType: string; Key: Word; 
      KeyChar: Char; Shift: TShiftState);
    function ShiftStateToString(Shift: TShiftState): string;
    function VKeyToString(Key: Word): string;
    procedure UpdateKeyInfo(const KeyName: string);
    procedure SetupHotkeyDemo;
  end;

var
  KeyboardTrackerForm: TKeyboardTrackerForm;

implementation

{$R *.lfm}

procedure TKeyboardTrackerForm.FormCreate(Sender: TObject);
begin
  Caption := 'Keyboard Input Tracker';
  Width := 700;
  Height := 550;
  Position := poScreenCenter;
  KeyPreview := True;
  
  // Setup
  SetLength(FKeyEvents, 1000);
  FKeyCount := 0;
  FTotalKeyPresses := 0;
  
  // Labels
  LabelInstruction.Caption := 'พิมพ์ในช่อง Input เพื่อดู Events ที่เกิดขึ้น';
  LabelCurrentKey.Caption := 'ปุ่มปัจจุบัน: -';
  LabelLastKey.Caption := 'ปุ่มล่าสุด: -';
  LabelKeyCount.Caption := 'จำนวนที่กด: 0';
  
  // Memo Setup
  MemoKeyEvents.Font.Name := 'Courier New';
  MemoKeyEvents.Font.Size := 9;
  MemoKeyEvents.ScrollBars := ssVertical;
  
  // CheckBox
  CheckBoxCapture.Caption := 'เปิดการบันทึก Events';
  CheckBoxCapture.Checked := True;
  
  SetupHotkeyDemo;
end;

procedure TKeyboardTrackerForm.SetupHotkeyDemo;
begin
  GroupBoxHotkeys.Caption := 'Hotkeys ที่รองรับ';
  
  // เพิ่ม Labels แสดง Hotkeys
  with TLabel.Create(GroupBoxHotkeys) do
  begin
    Parent := GroupBoxHotkeys;
    Caption := 'F1 = แสดงวิธีใช้' + #13#10 +
               'F5 = ล้าง Log' + #13#10 +
               'Ctrl+C = คัดลอก Log' + #13#10 +
               'Ctrl+A = เลือกทั้งหมด' + #13#10 +
               'Escape = ล้าง Input';
    Left := 10;
    Top := 20;
    WordWrap := True;
  end;
end;

function TKeyboardTrackerForm.ShiftStateToString(Shift: TShiftState): string;
var
  parts: TStringList;
begin
  parts := TStringList.Create;
  try
    if ssShift in Shift then parts.Add('Shift');
    if ssCtrl in Shift then parts.Add('Ctrl');
    if ssAlt in Shift then parts.Add('Alt');
    if ssLeft in Shift then parts.Add('Left');
    if ssRight in Shift then parts.Add('Right');
    if ssMiddle in Shift then parts.Add('Middle');
    if ssDouble in Shift then parts.Add('Double');
    
    Result := parts.CommaText;
    if Result = '' then Result := '-';
  finally
    parts.Free;
  end;
end;

function TKeyboardTrackerForm.VKeyToString(Key: Word): string;
begin
  case Key of
    VK_BACK:    Result := 'Backspace';
    VK_TAB:     Result := 'Tab';
    VK_RETURN:  Result := 'Enter';
    VK_ESCAPE:  Result := 'Escape';
    VK_SPACE:   Result := 'Space';
    VK_PRIOR:   Result := 'PageUp';
    VK_NEXT:    Result := 'PageDown';
    VK_END:     Result := 'End';
    VK_HOME:    Result := 'Home';
    VK_LEFT:    Result := 'Arrow Left';
    VK_UP:      Result := 'Arrow Up';
    VK_RIGHT:   Result := 'Arrow Right';
    VK_DOWN:    Result := 'Arrow Down';
    VK_INSERT:  Result := 'Insert';
    VK_DELETE:  Result := 'Delete';
    VK_F1..VK_F12: Result := 'F' + IntToStr(Key - VK_F1 + 1);
    VK_NUMLOCK: Result := 'NumLock';
    VK_CAPITAL: Result := 'CapsLock';
    VK_SHIFT:   Result := 'Shift';
    VK_CONTROL: Result := 'Ctrl';
    VK_MENU:    Result := 'Alt';
    else
      if (Key >= Ord('A')) and (Key <= Ord('Z')) then
        Result := Chr(Key)
      else if (Key >= Ord('0')) and (Key <= Ord('9')) then
        Result := Chr(Key)
      else
        Result := 'VK_' + IntToStr(Key);
  end;
end;

procedure TKeyboardTrackerForm.LogKeyEvent(const EventType: string;
  Key: Word; KeyChar: Char; Shift: TShiftState);
var
  logLine: string;
  keyName: string;
begin
  if not CheckBoxCapture.Checked then Exit;
  
  keyName := VKeyToString(Key);
  
  logLine := FormatDateTime('hh:nn:ss.zzz', Now) + ' | ' +
             PadRight(EventType, 8) + ' | ' +
             PadRight('Key=' + keyName, 20) + ' | ' +
             PadRight('Code=' + IntToStr(Key), 12) + ' | ' +
             PadRight('Char=' + IfThen(KeyChar >= ' ', 
               '"' + KeyChar + '"', '#' + IntToStr(Ord(KeyChar))), 12) + ' | ' +
             'Modifiers: ' + ShiftStateToString(Shift);
  
  MemoKeyEvents.Lines.Add(logLine);
  
  // เลื่อนไปบรรทัดล่าสุด
  MemoKeyEvents.SelStart := Length(MemoKeyEvents.Text);
  
  Inc(FTotalKeyPresses);
  UpdateKeyInfo(keyName);
end;

procedure TKeyboardTrackerForm.UpdateKeyInfo(const KeyName: string);
begin
  LabelLastKey.Caption := 'ปุ่มล่าสุด: ' + LabelCurrentKey.Caption;
  LabelCurrentKey.Caption := 'ปุ่มปัจจุบัน: ' + KeyName;
  LabelKeyCount.Caption := 'จำนวนที่กด: ' + IntToStr(FTotalKeyPresses);
end;

procedure TKeyboardTrackerForm.EditTestKeyDown(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  LogKeyEvent('DOWN', Key, #0, Shift);
end;

procedure TKeyboardTrackerForm.EditTestKeyUp(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  LogKeyEvent('UP', Key, #0, Shift);
end;

procedure TKeyboardTrackerForm.EditTestKeyPress(Sender: TObject; var Key: Char);
begin
  LogKeyEvent('PRESS', Ord(Key), Key, []);
end;

procedure TKeyboardTrackerForm.FormKeyDown(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  // Global Hotkeys
  case Key of
    VK_F1:
    begin
      ShowMessage('วิธีใช้:' + #13#10 +
        '- พิมพ์ในช่อง Input เพื่อดู Events' + #13#10 +
        '- F5 = ล้าง Log' + #13#10 +
        '- Ctrl+C = คัดลอก Log ทั้งหมด' + #13#10 +
        '- Escape ใน Input = ล้างข้อความ');
      Key := 0;
    end;
    VK_F5:
    begin
      ButtonClearClick(nil);
      Key := 0;
    end;
    VK_ESCAPE:
    begin
      if ActiveControl = EditTest then
      begin
        EditTest.Clear;
        Key := 0;
      end;
    end;
  end;
  
  if (ssCtrl in Shift) then
  begin
    case Key of
      Ord('A'):
      begin
        if ActiveControl = MemoKeyEvents then
        begin
          MemoKeyEvents.SelectAll;
          Key := 0;
        end;
      end;
    end;
  end;
end;

procedure TKeyboardTrackerForm.ButtonClearClick(Sender: TObject);
begin
  MemoKeyEvents.Clear;
  FTotalKeyPresses := 0;
  LabelCurrentKey.Caption := 'ปุ่มปัจจุบัน: -';
  LabelLastKey.Caption := 'ปุ่มล่าสุด: -';
  LabelKeyCount.Caption := 'จำนวนที่กด: 0';
end;

procedure TKeyboardTrackerForm.ButtonCopyLogClick(Sender: TObject);
begin
  MemoKeyEvents.SelectAll;
  MemoKeyEvents.CopyToClipboard;
  ShowMessage('คัดลอก Log ' + IntToStr(MemoKeyEvents.Lines.Count) + 
              ' บรรทัดแล้ว');
end;

end.
```

---

## 17.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Click Counter
สร้างโปรแกรมนับจำนวนครั้งที่คลิกปุ่ม และแสดงในทุกๆ 10 ครั้ง

```pascal
procedure TClickCounterForm.ButtonClick(Sender: TObject);
begin
  Inc(FClickCount);
  LabelCount.Caption := 'คลิก: ' + IntToStr(FClickCount);
  
  if FClickCount mod 10 = 0 then
    ShowMessage('คุณคลิกครบ ' + IntToStr(FClickCount) + ' ครั้งแล้ว!');
end;
```

### แบบฝึกหัดที่ 2: Mouse Tracker
สร้างโปรแกรมติดตามเส้นทางการเคลื่อนที่ของเมาส์บน Panel

```pascal
procedure TMouseTrackerForm.PanelMouseMove(Sender: TObject;
  Shift: TShiftState; X, Y: Integer);
begin
  // วาดจุดที่เมาส์ผ่าน
  Panel1.Canvas.Pen.Color := clRed;
  Panel1.Canvas.Pen.Width := 3;
  Panel1.Canvas.Pixels[X, Y] := clRed;
  
  // แสดงตำแหน่ง
  StatusBar1.Panels[0].Text := Format('X=%d Y=%d', [X, Y]);
end;
```

### แบบฝึกหัดที่ 3: Hotkey System
สร้างระบบ Hotkeys สำหรับ Application ที่มีหลาย Functions

```pascal
procedure THotkeyForm.FormKeyDown(Sender: TObject; var Key: Word;
  Shift: TShiftState);
begin
  if ssCtrl in Shift then
    case Key of
      Ord('N'): NewDocument;
      Ord('O'): OpenDocument;
      Ord('S'): SaveDocument;
      Ord('P'): PrintDocument;
      Ord('Z'): UndoAction;
      Ord('Y'): RedoAction;
    end;
    
  if (ssCtrl in Shift) and (ssShift in Shift) then
    case Key of
      Ord('S'): SaveAsDocument;
      Ord('N'): NewWindow;
    end;
end;
```

### แบบฝึกหัดที่ 4: Drag & Drop ระหว่าง ListBox
สร้างโปรแกรมที่ Drag ไอเทมจาก ListBox หนึ่งไปยังอีก ListBox หนึ่ง

```pascal
procedure TDragListForm.ListBox1MouseDown(Sender: TObject;
  Button: TMouseButton; Shift: TShiftState; X, Y: Integer);
begin
  if Button = mbLeft then
    if ListBox1.ItemIndex >= 0 then
      ListBox1.BeginDrag(False, 5);
end;

procedure TDragListForm.ListBox2DragOver(Sender, Source: TObject;
  X, Y: Integer; State: TDragState; var Accept: Boolean);
begin
  Accept := Source = ListBox1;
end;

procedure TDragListForm.ListBox2DragDrop(Sender, Source: TObject; X, Y: Integer);
begin
  if Source = ListBox1 then
  begin
    if ListBox1.ItemIndex >= 0 then
    begin
      ListBox2.Items.Add(ListBox1.Items[ListBox1.ItemIndex]);
      ListBox1.Items.Delete(ListBox1.ItemIndex);
    end;
  end;
end;
```

### แบบฝึกหัดที่ 5: Custom Event ในคลาสของตัวเอง
สร้าง Class ที่มี Custom Event `OnValueChanged` เมื่อค่าเปลี่ยน

```pascal
type
  TValueChangedEvent = procedure(Sender: TObject; OldValue, NewValue: Integer) 
    of object;

  TCounter = class
  private
    FValue: Integer;
    FOnValueChanged: TValueChangedEvent;
    procedure SetValue(const AValue: Integer);
  public
    procedure Increment;
    procedure Decrement;
    procedure Reset;
    property Value: Integer read FValue write SetValue;
    property OnValueChanged: TValueChangedEvent read FOnValueChanged 
      write FOnValueChanged;
  end;

procedure TCounter.SetValue(const AValue: Integer);
var
  oldValue: Integer;
begin
  if FValue <> AValue then
  begin
    oldValue := FValue;
    FValue := AValue;
    if Assigned(FOnValueChanged) then
      FOnValueChanged(Self, oldValue, AValue);
  end;
end;
```

### แบบฝึกหัดที่ 6-15 (หัวข้อเพิ่มเติม)

แบบฝึกหัดที่ 6: สร้าง Piano ด้วย Keyboard Events (A-K = โน้ต Do-Si)
แบบฝึกหัดที่ 7: Timer ที่ควบคุมด้วย Keyboard (Space = Start/Stop, R = Reset)
แบบฝึกหัดที่ 8: Snake Game อย่างง่ายด้วย Arrow Keys
แบบฝึกหัดที่ 9: Form ที่ย้ายได้โดย Drag Title Bar
แบบฝึกหัดที่ 10: ระบบ Undo/Redo ด้วย Event History
แบบฝึกหัดที่ 11: Tooltip แบบ Custom ด้วย MouseEnter/Leave
แบบฝึกหัดที่ 12: Context Menu แบบ Dynamic ตาม Context
แบบฝึกหัดที่ 13: Zoom Image ด้วย Ctrl+MouseWheel
แบบฝึกหัดที่ 14: Grid Navigation ด้วย Arrow Keys
แบบฝึกหัดที่ 15: Form Hotkey Manager ที่ Customize ได้

---

## สรุปบทที่ 17

ในบทนี้เราได้เรียนรู้:

1. **Event-Driven Programming** - แนวคิดและหลักการ
2. **Mouse Events** - Click, DblClick, MouseDown/Up/Move, Wheel
3. **Keyboard Events** - KeyDown, KeyUp, KeyPress และ Virtual Keys
4. **Focus Events** - Enter, Exit, Change
5. **Form Events** - Create, Destroy, Show, Hide, Close, Resize
6. **Custom Events** - การสร้าง Event Type ใหม่
7. **Event Delegation** - Handler เดียวสำหรับหลาย Controls
8. **Event Parameters** - Sender, Key, Mouse Coordinates, Shift State

บทถัดไปจะเรียนรู้เกี่ยวกับ **Menus และ Dialogs**
