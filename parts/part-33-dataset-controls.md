# ตอนที่ 33: Dataset Controls (DBGrid, DBEdit และอื่นๆ)

## บทนำ

Dataset Controls เป็น visual components ที่เชื่อมต่อกับฐานข้อมูลโดยตรงผ่าน TDataSource ทำให้สามารถแสดงและแก้ไขข้อมูลจาก dataset ได้โดยอัตโนมัติ ใน Lazarus มี DB-aware controls หลายตัวที่ช่วยให้การพัฒนา database application รวดเร็วขึ้นมาก

## TDataSource - ตัวกลางระหว่างข้อมูลและ Controls

```pascal
// TDataSource เชื่อม Dataset กับ DB Controls
// Dataset (TQuery, TTable) ---> TDataSource ---> DB Controls (TDBGrid, TDBEdit)

uses
  DB, ZDataSet;

var
  DataSource: TDataSource;
  Query: TZQuery;

// การเชื่อมต่อ
DataSource := TDataSource.Create(Self);
Query := TZQuery.Create(Self);
DataSource.DataSet := Query;  // เชื่อม Query กับ DataSource

// DBGrid จะแสดงข้อมูลจาก Query ผ่าน DataSource
DBGrid1.DataSource := DataSource;
DBEdit1.DataSource := DataSource;
DBEdit1.DataField := 'customer_name';
```

---

## TDBGrid

### การตั้งค่าพื้นฐาน

```pascal
unit DBGridBasics;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, DBGrids, DB, ZDataSet, ZConnection;

type
  TFormDBGrid = class(TForm)
    DBGrid1: TDBGrid;
    DataSource1: TDataSource;
    Query1: TZQuery;
    ZConnection1: TZConnection;
  private
    procedure SetupGrid;
    procedure ConfigureColumns;
  public
    procedure LoadData;
  end;

implementation

procedure TFormDBGrid.SetupGrid;
begin
  // คุณสมบัติพื้นฐาน
  DBGrid1.Options := [dgTitles, dgColumnResize, dgColLines, 
                      dgRowLines, dgTabs, dgAlwaysShowSelection,
                      dgConfirmDelete, dgCancelOnExit];
  DBGrid1.ReadOnly := False;
  DBGrid1.AlternateColor := $F0F0FF;  // สีแถวเว้น
  DBGrid1.Color := clWhite;
  DBGrid1.TitleFont.Style := [fsBold];
  
  // แสดง row numbers
  DBGrid1.Options := DBGrid1.Options + [dgRowIndicator];
  
  // ซ่อน row indicator column
  // DBGrid1.Options := DBGrid1.Options - [dgRowIndicator];
end;

procedure TFormDBGrid.ConfigureColumns;
var
  Col: TColumn;
begin
  // ล้าง columns เดิม
  DBGrid1.Columns.Clear;
  
  // เพิ่ม column แบบ manual
  // Column: Customer ID
  Col := DBGrid1.Columns.Add;
  Col.FieldName := 'customer_id';
  Col.Title.Caption := 'รหัส';
  Col.Width := 60;
  Col.Alignment := taCenter;
  Col.ReadOnly := True;
  
  // Column: Customer Name
  Col := DBGrid1.Columns.Add;
  Col.FieldName := 'customer_name';
  Col.Title.Caption := 'ชื่อลูกค้า';
  Col.Width := 200;
  
  // Column: Email
  Col := DBGrid1.Columns.Add;
  Col.FieldName := 'email';
  Col.Title.Caption := 'อีเมล';
  Col.Width := 180;
  
  // Column: Phone
  Col := DBGrid1.Columns.Add;
  Col.FieldName := 'phone';
  Col.Title.Caption := 'โทรศัพท์';
  Col.Width := 120;
  
  // Column: Registration Date
  Col := DBGrid1.Columns.Add;
  Col.FieldName := 'created_at';
  Col.Title.Caption := 'วันที่ลงทะเบียน';
  Col.Width := 130;
  Col.DisplayFormat := 'dd/mm/yyyy';
  Col.ReadOnly := True;
  
  // Column: Status (with ComboBox)
  Col := DBGrid1.Columns.Add;
  Col.FieldName := 'status';
  Col.Title.Caption := 'สถานะ';
  Col.Width := 100;
  Col.ButtonStyle := cbsAuto;
  Col.PickList.Add('active');
  Col.PickList.Add('inactive');
  Col.PickList.Add('suspended');
end;

procedure TFormDBGrid.LoadData;
begin
  if Query1.Active then Query1.Close;
  
  Query1.SQL.Text := 
    'SELECT customer_id, customer_name, email, phone, ' +
    '       status, created_at ' +
    'FROM customers ' +
    'ORDER BY customer_name';
  
  Query1.Open;
end;

end.
```

### TDBGrid: Custom Drawing

```pascal
unit DBGridCustomDraw;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, DBGrids, DB;

type
  TCustomDBGrid = class(TDBGrid)
  private
    FHighlightColor: TColor;
    FAlternateColor: TColor;
  protected
    procedure DrawCell(aCol, aRow: Integer; aRect: TRect; 
                       aState: TGridDrawState); override;
    procedure DrawColumnTitle(const aRect: TRect; 
                             DataCol: Integer; Column: TColumn; 
                             aState: TGridDrawState); override;
  public
    constructor Create(AOwner: TComponent); override;
    property HighlightColor: TColor read FHighlightColor write FHighlightColor;
    property AlternateColor: TColor read FAlternateColor write FAlternateColor;
  end;

  // Form ที่ใช้ Custom DBGrid
  TFormCustomGrid = class(TForm)
    procedure GridDrawColumnCell(Sender: TObject; const Rect: TRect;
                                DataCol: Integer; Column: TColumn;
                                State: TGridDrawState);
  end;

implementation

constructor TCustomDBGrid.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FHighlightColor := $FFE4B5;  // สีส้มอ่อน
  FAlternateColor := $F8F8FF;  // สีม่วงอ่อนมาก
end;

procedure TCustomDBGrid.DrawCell(aCol, aRow: Integer; aRect: TRect; 
                                 aState: TGridDrawState);
begin
  // ปรับสีพื้นหลังตามแถว
  if (gdSelected in aState) then
    Canvas.Brush.Color := FHighlightColor
  else if (aRow mod 2 = 0) then
    Canvas.Brush.Color := FAlternateColor
  else
    Canvas.Brush.Color := clWhite;
    
  Canvas.FillRect(aRect);
  inherited DrawCell(aCol, aRow, aRect, aState);
end;

procedure TCustomDBGrid.DrawColumnTitle(const aRect: TRect; DataCol: Integer; 
                                        Column: TColumn; aState: TGridDrawState);
begin
  Canvas.Brush.Color := $336699;  // สีน้ำเงิน header
  Canvas.Font.Color := clWhite;
  Canvas.Font.Style := [fsBold];
  Canvas.FillRect(aRect);
  Canvas.TextRect(aRect, aRect.Left + 4, aRect.Top + 3, Column.Title.Caption);
end;

// Event handler สำหรับ DrawColumnCell
procedure TFormCustomGrid.GridDrawColumnCell(Sender: TObject; const Rect: TRect;
                                            DataCol: Integer; Column: TColumn;
                                            State: TGridDrawState);
var
  Grid: TDBGrid;
  FieldValue: string;
  TextColor: TColor;
  BgColor: TColor;
begin
  Grid := TDBGrid(Sender);
  
  if Column.FieldName = 'status' then
  begin
    FieldValue := Column.Field.AsString;
    
    case FieldValue of
      'active':    begin TextColor := clGreen;  BgColor := $E6FFE6; end;
      'inactive':  begin TextColor := clGray;   BgColor := $F5F5F5; end;
      'suspended': begin TextColor := clRed;    BgColor := $FFE6E6; end;
      else         begin TextColor := clBlack;  BgColor := clWhite;  end;
    end;
    
    if gdSelected in State then
      BgColor := $CCE5FF;
    
    Grid.Canvas.Brush.Color := BgColor;
    Grid.Canvas.Font.Color := TextColor;
    Grid.Canvas.Font.Style := [fsBold];
    Grid.Canvas.FillRect(Rect);
    
    // Center text
    var TextWidth := Grid.Canvas.TextWidth(FieldValue);
    var TextX := Rect.Left + (Rect.Width - TextWidth) div 2;
    var TextY := Rect.Top + (Rect.Height - Grid.Canvas.TextHeight(FieldValue)) div 2;
    
    Grid.Canvas.TextOut(TextX, TextY, FieldValue);
  end
  else if Column.FieldName = 'total_amount' then
  begin
    // Format currency และเปลี่ยนสีตามมูลค่า
    var Amount := Column.Field.AsFloat;
    
    if Amount > 10000 then
      Grid.Canvas.Font.Color := $008000  // สีเขียวเข้ม
    else if Amount > 5000 then
      Grid.Canvas.Font.Color := $0080FF  // สีน้ำเงิน
    else
      Grid.Canvas.Font.Color := clBlack;
    
    Grid.Canvas.Brush.Color := clWhite;
    if gdSelected in State then
      Grid.Canvas.Brush.Color := $CCE5FF;
    
    Grid.Canvas.FillRect(Rect);
    
    // Right-align number
    var FormattedAmt := FormatFloat('#,##0.00', Amount);
    var TextWidth := Grid.Canvas.TextWidth(FormattedAmt);
    Grid.Canvas.TextOut(Rect.Right - TextWidth - 4, 
                       Rect.Top + 2, FormattedAmt);
  end;
end;

end.
```

### การจัดเรียงข้อมูลเมื่อคลิก Column Header

```pascal
unit DBGridSorting;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DBGrids, DB, ZDataSet, Controls;

type
  TSortDirection = (sdNone, sdAscending, sdDescending);
  
  TSortableDBGrid = class(TDBGrid)
  private
    FSortColumn: string;
    FSortDirection: TSortDirection;
    FBaseSQL: string;
    FDataQuery: TZQuery;
    
    procedure TitleClick(Column: TColumn);
    procedure ApplySort;
    procedure DrawSortIndicator(Column: TColumn; aRect: TRect);
  public
    constructor Create(AOwner: TComponent); override;
    
    property BaseSQL: string read FBaseSQL write FBaseSQL;
    property DataQuery: TZQuery read FDataQuery write FDataQuery;
  end;

implementation

constructor TSortableDBGrid.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FSortColumn := '';
  FSortDirection := sdNone;
  OnTitleClick := @TitleClick;
end;

procedure TSortableDBGrid.TitleClick(Column: TColumn);
begin
  if Column.FieldName = FSortColumn then
  begin
    // Toggle direction
    if FSortDirection = sdAscending then
      FSortDirection := sdDescending
    else
      FSortDirection := sdAscending;
  end
  else
  begin
    FSortColumn := Column.FieldName;
    FSortDirection := sdAscending;
  end;
  
  ApplySort;
  Invalidate;  // Redraw
end;

procedure TSortableDBGrid.ApplySort;
var
  SQL: string;
  DirectionStr: string;
begin
  if (FDataQuery = nil) or (FBaseSQL = '') then Exit;
  
  if FSortDirection = sdAscending then
    DirectionStr := 'ASC'
  else
    DirectionStr := 'DESC';
  
  if FSortColumn <> '' then
    SQL := FBaseSQL + ' ORDER BY ' + FSortColumn + ' ' + DirectionStr
  else
    SQL := FBaseSQL;
  
  if FDataQuery.Active then
    FDataQuery.Close;
  
  FDataQuery.SQL.Text := SQL;
  FDataQuery.Open;
end;

procedure TSortableDBGrid.DrawSortIndicator(Column: TColumn; aRect: TRect);
var
  IndicatorText: string;
  X, Y: Integer;
begin
  if Column.FieldName <> FSortColumn then Exit;
  
  if FSortDirection = sdAscending then
    IndicatorText := ' ▲'
  else
    IndicatorText := ' ▼';
  
  Canvas.Font.Color := clWhite;
  X := aRect.Right - Canvas.TextWidth(IndicatorText) - 2;
  Y := aRect.Top + (aRect.Height - Canvas.TextHeight(IndicatorText)) div 2;
  Canvas.TextOut(X, Y, IndicatorText);
end;

end.
```

### Live Filtering

```pascal
unit LiveFiltering;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, DBGrids, DB,
  ZDataSet, ZConnection;

type
  TFormLiveFilter = class(TForm)
    edtFilter: TEdit;
    cmbFilterField: TComboBox;
    dbgData: TDBGrid;
    dsData: TDataSource;
    qData: TZQuery;
    ZConn: TZConnection;
    
    procedure edtFilterChange(Sender: TObject);
    procedure cmbFilterFieldChange(Sender: TObject);
    procedure FormCreate(Sender: TObject);
    
  private
    FBaseSQL: string;
    FFilterTimer: TTimer;
    
    procedure TimerElapsed(Sender: TObject);
    procedure ApplyFilter;
  end;

implementation

procedure TFormLiveFilter.FormCreate(Sender: TObject);
begin
  FBaseSQL := 
    'SELECT customer_id, customer_name, email, phone, status, total_spent ' +
    'FROM customers';
  
  // Timer เพื่อ delay การค้นหา (ไม่ค้นหาทุก keystroke)
  FFilterTimer := TTimer.Create(Self);
  FFilterTimer.Interval := 300;  // 300ms delay
  FFilterTimer.Enabled := False;
  FFilterTimer.OnTimer := @TimerElapsed;
  
  // กำหนด filter fields
  cmbFilterField.Items.Add('ชื่อลูกค้า');
  cmbFilterField.Items.Add('อีเมล');
  cmbFilterField.Items.Add('โทรศัพท์');
  cmbFilterField.ItemIndex := 0;
  
  // โหลดข้อมูลครั้งแรก
  qData.SQL.Text := FBaseSQL + ' ORDER BY customer_name LIMIT 100';
  qData.Open;
end;

procedure TFormLiveFilter.edtFilterChange(Sender: TObject);
begin
  // รีเซ็ต timer ทุกครั้งที่พิมพ์
  FFilterTimer.Enabled := False;
  FFilterTimer.Enabled := True;
end;

procedure TFormLiveFilter.cmbFilterFieldChange(Sender: TObject);
begin
  ApplyFilter;
end;

procedure TFormLiveFilter.TimerElapsed(Sender: TObject);
begin
  FFilterTimer.Enabled := False;
  ApplyFilter;
end;

procedure TFormLiveFilter.ApplyFilter;
var
  FilterText: string;
  FieldName: string;
  SQL: string;
begin
  FilterText := Trim(edtFilter.Text);
  
  // กำหนด field ตาม combobox
  case cmbFilterField.ItemIndex of
    0: FieldName := 'customer_name';
    1: FieldName := 'email';
    2: FieldName := 'phone';
    else FieldName := 'customer_name';
  end;
  
  if FilterText = '' then
    SQL := FBaseSQL + ' ORDER BY customer_name LIMIT 100'
  else
    SQL := FBaseSQL + 
      ' WHERE LOWER(' + FieldName + ') LIKE LOWER(:filter_term)' +
      ' ORDER BY customer_name LIMIT 100';
  
  if qData.Active then qData.Close;
  qData.SQL.Text := SQL;
  
  if FilterText <> '' then
    qData.ParamByName('filter_term').AsString := '%' + FilterText + '%';
  
  qData.Open;
end;

end.
```

---

## TDBEdit, TDBMemo

```pascal
unit DBEditControls;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, DBCtrls, DB, StdCtrls, ExtCtrls;

type
  TFormDBControls = class(TForm)
    // Basic DB controls
    lblName: TLabel;
    dbedtName: TDBEdit;           // แสดงและแก้ไข text field
    
    lblDescription: TLabel;
    dbmemoDesc: TDBMemo;          // แสดงและแก้ไข text ยาว
    
    lblAmount: TLabel;
    dbedtAmount: TDBEdit;         // แสดงตัวเลข
    
    pnlButtons: TPanel;
    btnEdit: TButton;
    btnSave: TButton;
    btnCancel: TButton;
    
  private
    procedure SetupControls;
    procedure btnEditClick(Sender: TObject);
    procedure btnSaveClick(Sender: TObject);
    procedure btnCancelClick(Sender: TObject);
    
    procedure OnDataChange(Sender: TObject; Field: TField);
    procedure OnStateChange(Sender: TObject);
  end;

implementation

procedure TFormDBControls.SetupControls;
begin
  // TDBEdit setup
  dbedtName.DataSource := DataSource1;
  dbedtName.DataField := 'customer_name';
  dbedtName.MaxLength := 100;
  
  // Format สำหรับตัวเลข
  dbedtAmount.DataSource := DataSource1;
  dbedtAmount.DataField := 'total_spent';
  
  // TDBMemo setup
  dbmemoDesc.DataSource := DataSource1;
  dbmemoDesc.DataField := 'notes';
  dbmemoDesc.WantReturns := True;
  dbmemoDesc.ScrollBars := ssVertical;
  
  // ฟัง events
  DataSource1.OnDataChange := @OnDataChange;
  DataSource1.OnStateChange := @OnStateChange;
end;

procedure TFormDBControls.OnDataChange(Sender: TObject; Field: TField);
begin
  // อัปเดต UI เมื่อข้อมูลเปลี่ยน
  if Assigned(Field) then
    lblName.Caption := 'ฟิลด์ที่เปลี่ยน: ' + Field.FieldName;
end;

procedure TFormDBControls.OnStateChange(Sender: TObject);
begin
  // ปรับ button state ตาม dataset state
  case DataSource1.State of
    dsEdit, dsInsert:
    begin
      btnEdit.Enabled := False;
      btnSave.Enabled := True;
      btnCancel.Enabled := True;
    end;
    dsBrowse:
    begin
      btnEdit.Enabled := True;
      btnSave.Enabled := False;
      btnCancel.Enabled := False;
    end;
  end;
end;

procedure TFormDBControls.btnEditClick(Sender: TObject);
begin
  DataSource1.DataSet.Edit;
end;

procedure TFormDBControls.btnSaveClick(Sender: TObject);
begin
  try
    DataSource1.DataSet.Post;
    ShowMessage('บันทึกสำเร็จ');
  except
    on E: Exception do
      ShowMessage('ไม่สามารถบันทึก: ' + E.Message);
  end;
end;

procedure TFormDBControls.btnCancelClick(Sender: TObject);
begin
  DataSource1.DataSet.Cancel;
end;

end.
```

---

## TDBCheckBox, TDBComboBox, TDBListBox

```pascal
unit DBSelectControls;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, DBCtrls, DB, StdCtrls;

type
  TFormDBSelects = class(TForm)
    // TDBCheckBox
    dbchkActive: TDBCheckBox;      // checkbox ที่เชื่อมกับ boolean field
    
    // TDBComboBox
    dbcmbStatus: TDBComboBox;      // combobox ที่เชื่อมกับ text field
    dbcmbCategory: TDBComboBox;    // แสดงรายการหมวดหมู่
    
    // TDBListBox
    dblstTags: TDBListBox;         // listbox ที่เชื่อมกับ field
    
  private
    procedure SetupCheckBox;
    procedure SetupComboBox;
    procedure SetupListBox;
  end;

implementation

procedure TFormDBSelects.SetupCheckBox;
begin
  dbchkActive.DataSource := DataSource1;
  dbchkActive.DataField := 'is_active';
  dbchkActive.Caption := 'เปิดใช้งาน';
  
  // กำหนดค่า True/False ที่เก็บในฐานข้อมูล
  dbchkActive.ValueChecked := '1';   // ค่าที่เก็บเมื่อ checked
  dbchkActive.ValueUnchecked := '0'; // ค่าที่เก็บเมื่อ unchecked
  
  // สำหรับฐานข้อมูลที่ใช้ boolean จริงๆ
  // dbchkActive.ValueChecked := 'true';
  // dbchkActive.ValueUnchecked := 'false';
end;

procedure TFormDBSelects.SetupComboBox;
begin
  // TDBComboBox สำหรับ status field
  dbcmbStatus.DataSource := DataSource1;
  dbcmbStatus.DataField := 'status';
  dbcmbStatus.Style := csDropDownList;
  
  dbcmbStatus.Items.Add('active');
  dbcmbStatus.Items.Add('inactive');
  dbcmbStatus.Items.Add('suspended');
  dbcmbStatus.Items.Add('pending');
  
  // TDBComboBox สำหรับ category (โหลดจาก database)
  dbcmbCategory.DataSource := DataSource1;
  dbcmbCategory.DataField := 'category_id';
  
  // โหลด categories จาก database
  var Q := TZQuery.Create(nil);
  try
    Q.Connection := ZConn;
    Q.SQL.Text := 'SELECT category_id, name FROM categories ORDER BY name';
    Q.Open;
    
    while not Q.EOF do
    begin
      dbcmbCategory.Items.AddObject(
        Q.FieldByName('name').AsString,
        TObject(Q.FieldByName('category_id').AsInteger)
      );
      Q.Next;
    end;
    
    Q.Close;
  finally
    Q.Free;
  end;
end;

procedure TFormDBSelects.SetupListBox;
begin
  dblstTags.DataSource := DataSource1;
  dblstTags.DataField := 'tags';  // สำหรับ text field ที่เก็บรายการ
  
  dblstTags.Items.Add('electronics');
  dblstTags.Items.Add('clothing');
  dblstTags.Items.Add('food');
  dblstTags.Items.Add('books');
  dblstTags.MultiSelect := True;
end;

end.
```

---

## TDBImage

```pascal
unit DBImageControl;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, DBCtrls, DB, Graphics, ExtCtrls, Dialogs;

type
  TFormDBImage = class(TForm)
    dbimg: TDBImage;          // แสดงรูปภาพจาก BLOB field
    btnLoadImage: TButton;
    btnClearImage: TButton;
    
    procedure btnLoadImageClick(Sender: TObject);
    procedure btnClearImageClick(Sender: TObject);
  private
    procedure SetupImage;
    procedure LoadImageFromFile(const FileName: string);
    procedure SaveImageToFile(const FileName: string);
  end;

implementation

procedure TFormDBImage.SetupImage;
begin
  dbimg.DataSource := DataSource1;
  dbimg.DataField := 'product_image';  // BLOB field
  dbimg.Proportional := True;           // รักษาสัดส่วน
  dbimg.Stretch := True;                // ยืดให้เต็ม
  dbimg.Center := True;                 // จัดกลาง
end;

procedure TFormDBImage.btnLoadImageClick(Sender: TObject);
var
  OpenDlg: TOpenDialog;
begin
  OpenDlg := TOpenDialog.Create(Self);
  try
    OpenDlg.Filter := 'Image Files|*.jpg;*.jpeg;*.png;*.bmp;*.gif|All Files|*.*';
    
    if OpenDlg.Execute then
      LoadImageFromFile(OpenDlg.FileName);
  finally
    OpenDlg.Free;
  end;
end;

procedure TFormDBImage.LoadImageFromFile(const FileName: string);
var
  Stream: TMemoryStream;
  Bitmap: TBitmap;
  JPEGImage: TJPEGImage;
begin
  DataSource1.DataSet.Edit;
  
  // Detect file type and convert to BMP for storage
  Stream := TMemoryStream.Create;
  try
    if LowerCase(ExtractFileExt(FileName)) = '.jpg' or 
       LowerCase(ExtractFileExt(FileName)) = '.jpeg' then
    begin
      JPEGImage := TJPEGImage.Create;
      try
        JPEGImage.LoadFromFile(FileName);
        Bitmap := TBitmap.Create;
        try
          Bitmap.Assign(JPEGImage);
          Bitmap.SaveToStream(Stream);
        finally
          Bitmap.Free;
        end;
      finally
        JPEGImage.Free;
      end;
    end
    else
    begin
      // BMP หรือ PNG ตรงๆ
      Stream.LoadFromFile(FileName);
    end;
    
    Stream.Position := 0;
    
    // บันทึก stream ลงใน BLOB field
    TBlobField(DataSource1.DataSet.FieldByName('product_image')).LoadFromStream(Stream);
    
  finally
    Stream.Free;
  end;
end;

procedure TFormDBImage.btnClearImageClick(Sender: TObject);
begin
  DataSource1.DataSet.Edit;
  DataSource1.DataSet.FieldByName('product_image').Clear;
end;

procedure TFormDBImage.SaveImageToFile(const FileName: string);
var
  Stream: TMemoryStream;
begin
  if DataSource1.DataSet.FieldByName('product_image').IsNull then
  begin
    ShowMessage('ไม่มีรูปภาพ');
    Exit;
  end;
  
  Stream := TMemoryStream.Create;
  try
    TBlobField(DataSource1.DataSet.FieldByName('product_image')).SaveToStream(Stream);
    Stream.Position := 0;
    Stream.SaveToFile(FileName);
    ShowMessage('บันทึกรูปภาพแล้ว: ' + FileName);
  finally
    Stream.Free;
  end;
end;

end.
```

---

## TDBNavigator

```pascal
unit DBNavigatorControl;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, DBCtrls, DB;

type
  TFormNavigator = class(TForm)
    dbNav: TDBNavigator;
    lblRecordInfo: TLabel;
    
  private
    procedure SetupNavigator;
    procedure NavBeforeAction(Sender: TObject; Button: TNavigateBtn);
    procedure NavAfterAction(Sender: TObject; Button: TNavigateBtn);
    procedure UpdateRecordInfo;
    procedure CustomButtonClick(Sender: TObject);
  end;

implementation

procedure TFormNavigator.SetupNavigator;
begin
  dbNav.DataSource := DataSource1;
  
  // แสดงเฉพาะปุ่มที่ต้องการ
  dbNav.VisibleButtons := [nbFirst, nbPrior, nbNext, nbLast, 
                            nbInsert, nbDelete, nbEdit, nbPost, nbCancel];
  // ซ่อน Refresh button
  // nbRefresh ไม่อยู่ใน VisibleButtons
  
  // Hints สำหรับแต่ละปุ่ม
  dbNav.Hints.Clear;
  dbNav.Hints.Add('ไปยังรายการแรก');
  dbNav.Hints.Add('ย้อนกลับ');
  dbNav.Hints.Add('ถัดไป');
  dbNav.Hints.Add('ไปยังรายการสุดท้าย');
  dbNav.Hints.Add('เพิ่มรายการใหม่');
  dbNav.Hints.Add('ลบรายการ');
  dbNav.Hints.Add('แก้ไข');
  dbNav.Hints.Add('บันทึก');
  dbNav.Hints.Add('ยกเลิก');
  dbNav.ShowHint := True;
  
  dbNav.OnBeforeAction := @NavBeforeAction;
  dbNav.OnAfterAction := @NavAfterAction;
end;

procedure TFormNavigator.NavBeforeAction(Sender: TObject; Button: TNavigateBtn);
begin
  if Button = nbDelete then
  begin
    if MessageDlg('ยืนยันการลบ', 'คุณต้องการลบรายการนี้หรือไม่?',
                  mtConfirmation, [mbYes, mbNo], 0) = mrNo then
    begin
      // ยกเลิกการกระทำ
      // ไม่มีวิธีตรง - ต้อง override
    end;
  end;
end;

procedure TFormNavigator.NavAfterAction(Sender: TObject; Button: TNavigateBtn);
begin
  UpdateRecordInfo;
  
  case Button of
    nbInsert: ShowMessage('กำลังเพิ่มรายการใหม่');
    nbPost:   ShowMessage('บันทึกแล้ว');
    nbDelete: ShowMessage('ลบแล้ว');
  end;
end;

procedure TFormNavigator.UpdateRecordInfo;
var
  DS: TDataSet;
  RecNo, RecCount: Integer;
begin
  DS := DataSource1.DataSet;
  
  if not Assigned(DS) or not DS.Active then
  begin
    lblRecordInfo.Caption := 'ไม่มีข้อมูล';
    Exit;
  end;
  
  RecNo := DS.RecNo;
  RecCount := DS.RecordCount;
  
  lblRecordInfo.Caption := Format('รายการ %d จาก %d', [RecNo, RecCount]);
end;

end.
```

---

## Data Binding และ Virtual Datasets

```pascal
unit VirtualDataset;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, DBGrids, DB;

// TClientDataSet สำหรับ virtual data (ไม่ต้องเชื่อมฐานข้อมูล)
type
  TFormClientDataSet = class(TForm)
    CDS: TClientDataSet;    // ต้องติดตั้ง package ก่อน
    DS: TDataSource;
    Grid: TDBGrid;
    
  private
    procedure CreateFields;
    procedure PopulateData;
    procedure AddRecord(const Name, Email: string; Age: Integer);
  end;

// หรือใช้ TMemDataset จาก MemDS package
// ซึ่งมาพร้อมกับ Lazarus

implementation

procedure TFormClientDataSet.CreateFields;
begin
  CDS.Close;
  CDS.FieldDefs.Clear;
  
  // กำหนด fields
  CDS.FieldDefs.Add('id', ftInteger, 0, False);
  CDS.FieldDefs.Add('name', ftString, 100, True);
  CDS.FieldDefs.Add('email', ftString, 200, False);
  CDS.FieldDefs.Add('age', ftInteger, 0, False);
  CDS.FieldDefs.Add('salary', ftFloat, 0, False);
  CDS.FieldDefs.Add('hired_date', ftDate, 0, False);
  CDS.FieldDefs.Add('is_active', ftBoolean, 0, False);
  
  // สร้าง dataset
  CDS.CreateDataSet;
  
  // เชื่อม datasource
  DS.DataSet := CDS;
  Grid.DataSource := DS;
end;

procedure TFormClientDataSet.PopulateData;
begin
  AddRecord('สมชาย ใจดี', 'somchai@example.com', 35);
  AddRecord('สมหญิง รักดี', 'somying@example.com', 28);
  AddRecord('วิชัย เก่งมาก', 'wichai@example.com', 42);
end;

procedure TFormClientDataSet.AddRecord(const Name, Email: string; Age: Integer);
begin
  CDS.Append;
  CDS.FieldByName('id').AsInteger := CDS.RecordCount + 1;
  CDS.FieldByName('name').AsString := Name;
  CDS.FieldByName('email').AsString := Email;
  CDS.FieldByName('age').AsInteger := Age;
  CDS.FieldByName('salary').AsFloat := Random * 100000;
  CDS.FieldByName('hired_date').AsDateTime := Now - Random(365 * 5);
  CDS.FieldByName('is_active').AsBoolean := True;
  CDS.Post;
end;

end.
```

---

## Export DBGrid เป็น CSV, Excel, PDF

```pascal
unit DBGridExport;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, DBGrids, DB, Dialogs, 
  ComCtrls, fpSpreadsheet, fpsTypes, fpsAllFormats;

type
  TDBGridExporter = class
  private
    FGrid: TDBGrid;
    FDataset: TDataSet;
  public
    constructor Create(AGrid: TDBGrid);
    
    function ExportToCSV(const FileName: string): Boolean;
    function ExportToExcel(const FileName: string): Boolean;
    function ExportToHTML(const FileName: string): Boolean;
    function GetExportFileName(const DefaultExt: string): string;
  end;

  // Form ที่ใช้งาน
  TFormExport = class(TForm)
    btnExportCSV: TButton;
    btnExportExcel: TButton;
    dbgData: TDBGrid;
    
    procedure btnExportCSVClick(Sender: TObject);
    procedure btnExportExcelClick(Sender: TObject);
  end;

implementation

constructor TDBGridExporter.Create(AGrid: TDBGrid);
begin
  FGrid := AGrid;
  FDataset := AGrid.DataSource.DataSet;
end;

function TDBGridExporter.ExportToCSV(const FileName: string): Boolean;
var
  SL: TStringList;
  Row: TStringList;
  i: Integer;
  ColCount: Integer;
  HeaderRow: TStringList;
  OrigBookmark: TBookmark;
begin
  Result := False;
  
  if not Assigned(FDataset) or not FDataset.Active then
    Exit;
  
  SL := TStringList.Create;
  HeaderRow := TStringList.Create;
  Row := TStringList.Create;
  
  try
    // Header row
    for i := 0 to FGrid.Columns.Count - 1 do
    begin
      if FGrid.Columns[i].Visible then
        HeaderRow.Add('"' + FGrid.Columns[i].Title.Caption + '"');
    end;
    SL.Add(HeaderRow.CommaText);
    
    // Save current position
    OrigBookmark := FDataset.GetBookmark;
    FDataset.DisableControls;
    
    try
      FDataset.First;
      while not FDataset.EOF do
      begin
        Row.Clear;
        for i := 0 to FGrid.Columns.Count - 1 do
        begin
          if not FGrid.Columns[i].Visible then Continue;
          
          var Col := FGrid.Columns[i];
          var FieldVal := '';
          
          if Assigned(Col.Field) then
          begin
            case Col.Field.DataType of
              ftString, ftWideString:
                FieldVal := '"' + StringReplace(Col.Field.AsString, '"', '""', [rfReplaceAll]) + '"';
              ftDate:
                FieldVal := FormatDateTime('dd/mm/yyyy', Col.Field.AsDateTime);
              ftDateTime:
                FieldVal := FormatDateTime('dd/mm/yyyy hh:nn:ss', Col.Field.AsDateTime);
              ftFloat, ftCurrency:
                FieldVal := FormatFloat('0.00', Col.Field.AsFloat);
              else
                FieldVal := Col.Field.AsString;
            end;
          end;
          
          Row.Add(FieldVal);
        end;
        SL.Add(Row.CommaText);
        FDataset.Next;
      end;
      
    finally
      FDataset.GotoBookmark(OrigBookmark);
      FDataset.FreeBookmark(OrigBookmark);
      FDataset.EnableControls;
    end;
    
    // บันทึกพร้อม BOM สำหรับ UTF-8
    var Stream := TFileStream.Create(FileName, fmCreate);
    try
      // Write UTF-8 BOM
      var BOM: array[0..2] of Byte = ($EF, $BB, $BF);
      Stream.Write(BOM, 3);
      
      var Encoding := TEncoding.UTF8;
      var Content := Encoding.GetBytes(SL.Text);
      Stream.Write(Content[0], Length(Content));
    finally
      Stream.Free;
    end;
    
    Result := True;
    ShowMessage(Format('ส่งออก CSV สำเร็จ: %s'#13#10'จำนวน %d รายการ',
      [FileName, SL.Count - 1]));
    
  finally
    SL.Free;
    HeaderRow.Free;
    Row.Free;
  end;
end;

function TDBGridExporter.ExportToExcel(const FileName: string): Boolean;
var
  Workbook: TsWorkbook;
  Sheet: TsWorksheet;
  RowIdx, ColIdx: Integer;
  OrigBookmark: TBookmark;
begin
  Result := False;
  
  if not Assigned(FDataset) or not FDataset.Active then
    Exit;
  
  Workbook := TsWorkbook.Create;
  try
    Sheet := Workbook.AddWorksheet('ข้อมูล');
    
    // Header row - format
    var HeaderFmt: TsCellFormat;
    InitFormatRecord(HeaderFmt);
    HeaderFmt.Background.FgColor := $004472C4;  // สีน้ำเงิน
    HeaderFmt.Background.BgColor := $004472C4;
    HeaderFmt.Background.Style := fsSolidFill;
    HeaderFmt.Font.Color := clWhite;
    HeaderFmt.Font.Style := [fssBold];
    var HeaderFmtIdx := Workbook.AddCellFormat(HeaderFmt);
    
    // เพิ่ม header
    ColIdx := 0;
    for var i := 0 to FGrid.Columns.Count - 1 do
    begin
      if FGrid.Columns[i].Visible then
      begin
        Sheet.WriteText(0, ColIdx, FGrid.Columns[i].Title.Caption);
        Sheet.WriteFormat(0, ColIdx, HeaderFmtIdx);
        Inc(ColIdx);
      end;
    end;
    
    // Data rows
    OrigBookmark := FDataset.GetBookmark;
    FDataset.DisableControls;
    
    try
      FDataset.First;
      RowIdx := 1;
      
      while not FDataset.EOF do
      begin
        // Alternate row color
        var RowFmt: TsCellFormat;
        InitFormatRecord(RowFmt);
        if RowIdx mod 2 = 0 then
        begin
          RowFmt.Background.FgColor := $00F2F2F2;
          RowFmt.Background.Style := fsSolidFill;
        end;
        var RowFmtIdx := Workbook.AddCellFormat(RowFmt);
        
        ColIdx := 0;
        for var i := 0 to FGrid.Columns.Count - 1 do
        begin
          if not FGrid.Columns[i].Visible then Continue;
          
          var Col := FGrid.Columns[i];
          if Assigned(Col.Field) then
          begin
            case Col.Field.DataType of
              ftInteger, ftSmallint:
                Sheet.WriteNumber(RowIdx, ColIdx, Col.Field.AsInteger);
              ftFloat, ftCurrency:
                begin
                  Sheet.WriteNumber(RowIdx, ColIdx, Col.Field.AsFloat);
                  // Format เป็น currency
                  var NumFmt: TsCellFormat;
                  InitFormatRecord(NumFmt);
                  NumFmt.NumberFormatStr := '#,##0.00';
                  Sheet.WriteFormat(RowIdx, ColIdx, Workbook.AddCellFormat(NumFmt));
                end;
              ftDate:
                Sheet.WriteDateTime(RowIdx, ColIdx, Col.Field.AsDateTime);
              ftBoolean:
                Sheet.WriteBoolean(RowIdx, ColIdx, Col.Field.AsBoolean);
              else
                Sheet.WriteText(RowIdx, ColIdx, Col.Field.AsString);
            end;
            
            Sheet.WriteFormat(RowIdx, ColIdx, RowFmtIdx);
          end;
          
          Inc(ColIdx);
        end;
        
        Inc(RowIdx);
        FDataset.Next;
      end;
      
    finally
      FDataset.GotoBookmark(OrigBookmark);
      FDataset.FreeBookmark(OrigBookmark);
      FDataset.EnableControls;
    end;
    
    // ปรับความกว้าง columns อัตโนมัติ
    for var i := 0 to ColIdx - 1 do
      Sheet.WriteColWidth(i, -1);  // Auto
    
    // บันทึก
    Workbook.WriteToFile(FileName, sfExcel);
    
    Result := True;
    ShowMessage(Format('ส่งออก Excel สำเร็จ: %s'#13#10'จำนวน %d รายการ',
      [FileName, RowIdx - 1]));
    
  finally
    Workbook.Free;
  end;
end;

function TDBGridExporter.GetExportFileName(const DefaultExt: string): string;
var
  SaveDlg: TSaveDialog;
begin
  Result := '';
  
  SaveDlg := TSaveDialog.Create(nil);
  try
    if DefaultExt = '.csv' then
      SaveDlg.Filter := 'CSV Files|*.csv|All Files|*.*'
    else if DefaultExt = '.xlsx' then
      SaveDlg.Filter := 'Excel Files|*.xlsx|All Files|*.*';
    
    SaveDlg.DefaultExt := DefaultExt;
    SaveDlg.FileName := 'export_' + FormatDateTime('yyyymmdd_hhnnss', Now);
    
    if SaveDlg.Execute then
      Result := SaveDlg.FileName;
  finally
    SaveDlg.Free;
  end;
end;

// Form event handlers
procedure TFormExport.btnExportCSVClick(Sender: TObject);
var
  Exporter: TDBGridExporter;
  FileName: string;
begin
  Exporter := TDBGridExporter.Create(dbgData);
  try
    FileName := Exporter.GetExportFileName('.csv');
    if FileName <> '' then
      Exporter.ExportToCSV(FileName);
  finally
    Exporter.Free;
  end;
end;

procedure TFormExport.btnExportExcelClick(Sender: TObject);
var
  Exporter: TDBGridExporter;
  FileName: string;
begin
  Exporter := TDBGridExporter.Create(dbgData);
  try
    FileName := Exporter.GetExportFileName('.xlsx');
    if FileName <> '' then
      Exporter.ExportToExcel(FileName);
  finally
    Exporter.Free;
  end;
end;

end.
```

---

## Custom DBGrid พร้อม Checkboxes

```pascal
unit DBGridWithCheckbox;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, DBGrids, DB, Graphics, StdCtrls;

type
  TCheckableDBGrid = class(TDBGrid)
  private
    FCheckedIDs: TList;
    FCheckColumnIndex: Integer;
    
    procedure MouseDown(Button: TMouseButton; Shift: TShiftState; 
                       X, Y: Integer); override;
    procedure DrawCheckbox(aRect: TRect; Checked: Boolean);
    function GetCurrentRowID: Integer;
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure CheckAll;
    procedure UncheckAll;
    function GetCheckedIDs: TList;
    function IsChecked(ID: Integer): Boolean;
    
    property CheckColumnIndex: Integer read FCheckColumnIndex 
                                       write FCheckColumnIndex;
  end;

  TFormCheckGrid = class(TForm)
    cdbgProducts: TCheckableDBGrid;
    btnCheckAll: TButton;
    btnUncheckAll: TButton;
    btnDeleteSelected: TButton;
    lblSelectedCount: TLabel;
    
    procedure btnCheckAllClick(Sender: TObject);
    procedure btnUncheckAllClick(Sender: TObject);
    procedure btnDeleteSelectedClick(Sender: TObject);
  end;

implementation

constructor TCheckableDBGrid.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FCheckedIDs := TList.Create;
  FCheckColumnIndex := 0;
end;

destructor TCheckableDBGrid.Destroy;
begin
  FCheckedIDs.Free;
  inherited Destroy;
end;

procedure TCheckableDBGrid.MouseDown(Button: TMouseButton; Shift: TShiftState; 
                                     X, Y: Integer);
var
  Col, Row: Integer;
  CellRect: TRect;
begin
  // ตรวจสอบว่าคลิกที่ checkbox column หรือไม่
  MouseToCell(X, Y, Col, Row);
  
  if (Col = FCheckColumnIndex) and (Row > 0) then
  begin
    var ID := GetCurrentRowID;
    if IsChecked(ID) then
      FCheckedIDs.Remove(Pointer(ID))
    else
      FCheckedIDs.Add(Pointer(ID));
    
    Invalidate;
    
    // อัปเดต count label
    var Form := TFormCheckGrid(Owner);
    Form.lblSelectedCount.Caption := 
      Format('เลือกแล้ว %d รายการ', [FCheckedIDs.Count]);
  end
  else
    inherited MouseDown(Button, Shift, X, Y);
end;

procedure TCheckableDBGrid.DrawCheckbox(aRect: TRect; Checked: Boolean);
var
  CX, CY: Integer;
  CheckSize: Integer;
begin
  CheckSize := 14;
  CX := aRect.Left + (aRect.Width - CheckSize) div 2;
  CY := aRect.Top + (aRect.Height - CheckSize) div 2;
  
  var CheckRect := Rect(CX, CY, CX + CheckSize, CY + CheckSize);
  
  Canvas.Brush.Color := clWhite;
  Canvas.Pen.Color := clGray;
  Canvas.Rectangle(CheckRect);
  
  if Checked then
  begin
    Canvas.Pen.Color := clBlue;
    Canvas.Pen.Width := 2;
    
    // Draw checkmark
    Canvas.MoveTo(CheckRect.Left + 2, CheckRect.Top + CheckSize div 2);
    Canvas.LineTo(CheckRect.Left + CheckSize div 3, CheckRect.Bottom - 3);
    Canvas.LineTo(CheckRect.Right - 2, CheckRect.Top + 2);
    
    Canvas.Pen.Width := 1;
  end;
end;

function TCheckableDBGrid.GetCurrentRowID: Integer;
begin
  Result := DataSource.DataSet.FieldByName('id').AsInteger;
end;

procedure TCheckableDBGrid.CheckAll;
var
  Bookmark: TBookmark;
begin
  FCheckedIDs.Clear;
  
  if not Assigned(DataSource) or not DataSource.DataSet.Active then Exit;
  
  Bookmark := DataSource.DataSet.GetBookmark;
  DataSource.DataSet.DisableControls;
  
  try
    DataSource.DataSet.First;
    while not DataSource.DataSet.EOF do
    begin
      FCheckedIDs.Add(Pointer(GetCurrentRowID));
      DataSource.DataSet.Next;
    end;
  finally
    DataSource.DataSet.GotoBookmark(Bookmark);
    DataSource.DataSet.FreeBookmark(Bookmark);
    DataSource.DataSet.EnableControls;
  end;
  
  Invalidate;
end;

procedure TCheckableDBGrid.UncheckAll;
begin
  FCheckedIDs.Clear;
  Invalidate;
end;

function TCheckableDBGrid.GetCheckedIDs: TList;
begin
  Result := FCheckedIDs;
end;

function TCheckableDBGrid.IsChecked(ID: Integer): Boolean;
begin
  Result := FCheckedIDs.IndexOf(Pointer(ID)) >= 0;
end;

// Form handlers
procedure TFormCheckGrid.btnCheckAllClick(Sender: TObject);
begin
  cdbgProducts.CheckAll;
  lblSelectedCount.Caption := Format('เลือกแล้ว %d รายการ',
    [cdbgProducts.GetCheckedIDs.Count]);
end;

procedure TFormCheckGrid.btnUncheckAllClick(Sender: TObject);
begin
  cdbgProducts.UncheckAll;
  lblSelectedCount.Caption := 'ไม่มีรายการที่เลือก';
end;

procedure TFormCheckGrid.btnDeleteSelectedClick(Sender: TObject);
var
  IDs: TList;
  i: Integer;
begin
  IDs := cdbgProducts.GetCheckedIDs;
  
  if IDs.Count = 0 then
  begin
    ShowMessage('กรุณาเลือกรายการที่ต้องการลบ');
    Exit;
  end;
  
  if MessageDlg('ยืนยันการลบ', 
                Format('คุณต้องการลบ %d รายการที่เลือกหรือไม่?', [IDs.Count]),
                mtConfirmation, [mbYes, mbNo], 0) <> mrYes then
    Exit;
  
  // สร้าง ID list สำหรับ DELETE
  var IDList := '';
  for i := 0 to IDs.Count - 1 do
  begin
    if IDList <> '' then IDList := IDList + ',';
    IDList := IDList + IntToStr(Integer(IDs[i]));
  end;
  
  var Q := TZQuery.Create(nil);
  try
    Q.Connection := ZConn;
    Q.SQL.Text := 'DELETE FROM products WHERE product_id IN (' + IDList + ')';
    Q.ExecSQL;
    
    ShowMessage(Format('ลบ %d รายการสำเร็จ', [Q.RowsAffected]));
    cdbgProducts.UncheckAll;
    
    // Reload data
    DataSource1.DataSet.Close;
    DataSource1.DataSet.Open;
    
  finally
    Q.Free;
  end;
end;

end.
```

---

## ตัวอย่าง Product Catalog Browser สมบูรณ์

```pascal
unit ProductCatalog;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ComCtrls, ExtCtrls, DBGrids, DB,
  ZConnection, ZDataSet;

type
  TFormProductCatalog = class(TForm)
    // Layout
    pnlTop: TPanel;
    pnlLeft: TPanel;
    pnlCenter: TPanel;
    pnlBottom: TPanel;
    pnlSearch: TPanel;
    
    // Search and Filter
    edtSearch: TEdit;
    btnSearch: TButton;
    cmbCategory: TComboBox;
    lblPriceRange: TLabel;
    edtMinPrice: TEdit;
    edtMaxPrice: TEdit;
    btnFilter: TButton;
    btnClearFilter: TButton;
    
    // Category Tree
    tvCategories: TTreeView;
    
    // Product Grid
    dbgProducts: TDBGrid;
    
    // Product Detail
    pnlDetail: TPanel;
    imgProduct: TImage;
    lblProductName: TLabel;
    lblSKU: TLabel;
    lblPrice: TLabel;
    lblStock: TLabel;
    memoDescription: TMemo;
    
    // Navigation and Status
    pnlNav: TPanel;
    lblTotal: TLabel;
    lblPage: TLabel;
    btnFirst: TButton;
    btnPrev: TButton;
    btnNext: TButton;
    btnLast: TButton;
    
    // DB Components
    dsProducts: TDataSource;
    qProducts: TZQuery;
    ZConn: TZConnection;
    
    procedure FormCreate(Sender: TObject);
    procedure edtSearchKeyPress(Sender: TObject; var Key: Char);
    procedure btnSearchClick(Sender: TObject);
    procedure btnFilterClick(Sender: TObject);
    procedure btnClearFilterClick(Sender: TObject);
    procedure tvCategoriesChange(Sender: TObject; Node: TTreeNode);
    procedure dbgProductsSelectionChange(Sender: TObject);
    procedure dbgProductsTitleClick(Column: TColumn);
    procedure btnFirstClick(Sender: TObject);
    procedure btnPrevClick(Sender: TObject);
    procedure btnNextClick(Sender: TObject);
    procedure btnLastClick(Sender: TObject);
    
  private
    FCurrentPage: Integer;
    FPageSize: Integer;
    FTotalRecords: Integer;
    FSortColumn: string;
    FSortAscending: Boolean;
    FSelectedCategoryID: Integer;
    
    procedure InitializeForm;
    procedure LoadCategories;
    procedure LoadProducts;
    procedure UpdateProductDetails;
    procedure UpdatePagination;
    function BuildWhereClause: string;
    function BuildOrderClause: string;
  end;

var
  frmProductCatalog: TFormProductCatalog;

implementation

{$R *.lfm}

procedure TFormProductCatalog.FormCreate(Sender: TObject);
begin
  FCurrentPage := 1;
  FPageSize := 20;
  FSortColumn := 'name';
  FSortAscending := True;
  FSelectedCategoryID := 0;
  
  InitializeForm;
end;

procedure TFormProductCatalog.InitializeForm;
begin
  // เชื่อม DB
  ZConn.Protocol := 'postgresql';
  ZConn.HostName := 'localhost';
  ZConn.Database := 'ecommerce_db';
  ZConn.User := 'app_user';
  ZConn.Password := 'password';
  ZConn.Connect;
  
  qProducts.Connection := ZConn;
  dsProducts.DataSet := qProducts;
  dbgProducts.DataSource := dsProducts;
  
  // Setup Grid columns
  dbgProducts.Columns.Clear;
  
  var Col: TColumn;
  
  Col := dbgProducts.Columns.Add;
  Col.FieldName := 'product_id';
  Col.Title.Caption := 'ID';
  Col.Width := 50;
  Col.ReadOnly := True;
  
  Col := dbgProducts.Columns.Add;
  Col.FieldName := 'sku';
  Col.Title.Caption := 'SKU';
  Col.Width := 100;
  Col.ReadOnly := True;
  
  Col := dbgProducts.Columns.Add;
  Col.FieldName := 'name';
  Col.Title.Caption := 'ชื่อสินค้า';
  Col.Width := 250;
  Col.ReadOnly := True;
  
  Col := dbgProducts.Columns.Add;
  Col.FieldName := 'price';
  Col.Title.Caption := 'ราคา';
  Col.Width := 100;
  Col.DisplayFormat := '#,##0.00';
  Col.ReadOnly := True;
  
  Col := dbgProducts.Columns.Add;
  Col.FieldName := 'stock_qty';
  Col.Title.Caption := 'คงเหลือ';
  Col.Width := 80;
  Col.ReadOnly := True;
  
  Col := dbgProducts.Columns.Add;
  Col.FieldName := 'category_name';
  Col.Title.Caption := 'หมวดหมู่';
  Col.Width := 150;
  Col.ReadOnly := True;
  
  // Load categories สำหรับ combobox และ treeview
  LoadCategories;
  
  // Load initial data
  LoadProducts;
end;

procedure TFormProductCatalog.LoadCategories;
var
  Q: TZQuery;
  Node, ParentNode: TTreeNode;
  CategoryNodes: TStringList;
begin
  // ComboBox
  cmbCategory.Items.Clear;
  cmbCategory.Items.Add('ทุกหมวดหมู่');
  cmbCategory.ItemIndex := 0;
  
  Q := TZQuery.Create(nil);
  CategoryNodes := TStringList.Create;
  try
    Q.Connection := ZConn;
    Q.SQL.Text := 
      'WITH RECURSIVE cat_tree AS (' +
      '  SELECT category_id, parent_id, name, 0 AS depth FROM categories WHERE parent_id IS NULL' +
      '  UNION ALL' +
      '  SELECT c.category_id, c.parent_id, c.name, ct.depth + 1' +
      '  FROM categories c INNER JOIN cat_tree ct ON c.parent_id = ct.category_id' +
      ')' +
      'SELECT category_id, parent_id, name, depth FROM cat_tree ORDER BY name';
    Q.Open;
    
    tvCategories.Items.Clear;
    var AllCatsNode := tvCategories.Items.AddFirst(nil, 'ทุกหมวดหมู่');
    AllCatsNode.Data := Pointer(0);
    
    while not Q.EOF do
    begin
      var ID := Q.FieldByName('category_id').AsInteger;
      var ParentID := Q.FieldByName('parent_id').AsInteger;
      var Name := Q.FieldByName('name').AsString;
      var Depth := Q.FieldByName('depth').AsInteger;
      
      cmbCategory.Items.AddObject(StringOfChar('  ', Depth) + Name, TObject(ID));
      
      if ParentID = 0 then
        Node := tvCategories.Items.AddChild(AllCatsNode, Name)
      else
      begin
        var ParentStr := IntToStr(ParentID);
        var ParentIdx := CategoryNodes.IndexOf(ParentStr);
        if ParentIdx >= 0 then
          ParentNode := TTreeNode(CategoryNodes.Objects[ParentIdx])
        else
          ParentNode := AllCatsNode;
        Node := tvCategories.Items.AddChild(ParentNode, Name);
      end;
      
      Node.Data := Pointer(ID);
      CategoryNodes.AddObject(IntToStr(ID), Node);
      
      Q.Next;
    end;
    
    Q.Close;
    tvCategories.FullExpand;
    
  finally
    Q.Free;
    CategoryNodes.Free;
  end;
end;

procedure TFormProductCatalog.LoadProducts;
var
  CountQ: TZQuery;
begin
  // นับจำนวนรายการทั้งหมด
  CountQ := TZQuery.Create(nil);
  try
    CountQ.Connection := ZConn;
    CountQ.SQL.Text := 
      'SELECT COUNT(*) FROM products p ' +
      'LEFT JOIN categories c ON p.category_id = c.category_id ' +
      BuildWhereClause;
    
    CountQ.Open;
    FTotalRecords := CountQ.Fields[0].AsInteger;
    CountQ.Close;
  finally
    CountQ.Free;
  end;
  
  // Load data
  if qProducts.Active then qProducts.Close;
  
  qProducts.SQL.Text := 
    'SELECT p.product_id, p.sku, p.name, p.description, ' +
    '       p.price, p.sale_price, p.stock_qty, ' +
    '       p.images, p.rating_avg, ' +
    '       c.name AS category_name ' +
    'FROM products p ' +
    'LEFT JOIN categories c ON p.category_id = c.category_id ' +
    BuildWhereClause +
    BuildOrderClause +
    Format(' LIMIT %d OFFSET %d', [FPageSize, (FCurrentPage - 1) * FPageSize]);
  
  qProducts.Open;
  
  UpdatePagination;
end;

function TFormProductCatalog.BuildWhereClause: string;
var
  Conditions: TStringList;
begin
  Conditions := TStringList.Create;
  try
    Conditions.Add('p.is_active = TRUE');
    
    // Search term
    if Trim(edtSearch.Text) <> '' then
      Conditions.Add(Format('(p.name ILIKE ''%%%s%%'' OR p.sku ILIKE ''%%%s%%'')',
        [edtSearch.Text, edtSearch.Text]));
    
    // Category filter
    if FSelectedCategoryID > 0 then
      Conditions.Add(Format('p.category_id = %d', [FSelectedCategoryID]));
    
    // Price range
    var MinPrice := StrToFloatDef(edtMinPrice.Text, 0);
    var MaxPrice := StrToFloatDef(edtMaxPrice.Text, 0);
    
    if MinPrice > 0 then
      Conditions.Add(Format('p.price >= %g', [MinPrice]));
    if MaxPrice > 0 then
      Conditions.Add(Format('p.price <= %g', [MaxPrice]));
    
    if Conditions.Count > 0 then
      Result := ' WHERE ' + Conditions.CommaText.Replace(',', ' AND ')
    else
      Result := '';
  finally
    Conditions.Free;
  end;
end;

function TFormProductCatalog.BuildOrderClause: string;
begin
  Result := ' ORDER BY ' + FSortColumn;
  if FSortAscending then
    Result := Result + ' ASC'
  else
    Result := Result + ' DESC';
end;

procedure TFormProductCatalog.UpdatePagination;
var
  TotalPages: Integer;
begin
  TotalPages := (FTotalRecords + FPageSize - 1) div FPageSize;
  
  lblTotal.Caption := Format('ทั้งหมด %s รายการ', 
    [FormatFloat('#,##0', FTotalRecords)]);
  lblPage.Caption := Format('หน้า %d จาก %d', [FCurrentPage, TotalPages]);
  
  btnFirst.Enabled := FCurrentPage > 1;
  btnPrev.Enabled := FCurrentPage > 1;
  btnNext.Enabled := FCurrentPage < TotalPages;
  btnLast.Enabled := FCurrentPage < TotalPages;
end;

procedure TFormProductCatalog.UpdateProductDetails;
begin
  if qProducts.EOF or qProducts.BOF then Exit;
  
  lblProductName.Caption := qProducts.FieldByName('name').AsString;
  lblSKU.Caption := 'SKU: ' + qProducts.FieldByName('sku').AsString;
  
  var Price := qProducts.FieldByName('price').AsFloat;
  var SalePrice := qProducts.FieldByName('sale_price').AsFloat;
  
  if not qProducts.FieldByName('sale_price').IsNull and (SalePrice > 0) then
    lblPrice.Caption := Format('ราคา: %.2f บาท (ลดจาก %.2f)', [SalePrice, Price])
  else
    lblPrice.Caption := Format('ราคา: %.2f บาท', [Price]);
  
  var Stock := qProducts.FieldByName('stock_qty').AsInteger;
  if Stock = 0 then
    lblStock.Caption := 'สินค้าหมด'
  else if Stock < 10 then
    lblStock.Caption := Format('คงเหลือ: %d (ใกล้หมด)', [Stock])
  else
    lblStock.Caption := Format('คงเหลือ: %d', [Stock]);
  
  memoDescription.Text := qProducts.FieldByName('description').AsString;
end;

procedure TFormProductCatalog.dbgProductsSelectionChange(Sender: TObject);
begin
  UpdateProductDetails;
end;

procedure TFormProductCatalog.dbgProductsTitleClick(Column: TColumn);
begin
  if FSortColumn = Column.FieldName then
    FSortAscending := not FSortAscending
  else
  begin
    FSortColumn := Column.FieldName;
    FSortAscending := True;
  end;
  
  FCurrentPage := 1;
  LoadProducts;
end;

procedure TFormProductCatalog.tvCategoriesChange(Sender: TObject; Node: TTreeNode);
begin
  if Assigned(Node) then
    FSelectedCategoryID := Integer(Node.Data)
  else
    FSelectedCategoryID := 0;
  
  FCurrentPage := 1;
  LoadProducts;
end;

procedure TFormProductCatalog.btnFirstClick(Sender: TObject);
begin
  FCurrentPage := 1;
  LoadProducts;
end;

procedure TFormProductCatalog.btnPrevClick(Sender: TObject);
begin
  if FCurrentPage > 1 then
  begin
    Dec(FCurrentPage);
    LoadProducts;
  end;
end;

procedure TFormProductCatalog.btnNextClick(Sender: TObject);
begin
  var TotalPages := (FTotalRecords + FPageSize - 1) div FPageSize;
  if FCurrentPage < TotalPages then
  begin
    Inc(FCurrentPage);
    LoadProducts;
  end;
end;

procedure TFormProductCatalog.btnLastClick(Sender: TObject);
begin
  FCurrentPage := (FTotalRecords + FPageSize - 1) div FPageSize;
  LoadProducts;
end;

procedure TFormProductCatalog.btnSearchClick(Sender: TObject);
begin
  FCurrentPage := 1;
  LoadProducts;
end;

procedure TFormProductCatalog.edtSearchKeyPress(Sender: TObject; var Key: Char);
begin
  if Key = #13 then
  begin
    btnSearchClick(nil);
    Key := #0;
  end;
end;

procedure TFormProductCatalog.btnFilterClick(Sender: TObject);
begin
  FCurrentPage := 1;
  LoadProducts;
end;

procedure TFormProductCatalog.btnClearFilterClick(Sender: TObject);
begin
  edtSearch.Text := '';
  edtMinPrice.Text := '';
  edtMaxPrice.Text := '';
  cmbCategory.ItemIndex := 0;
  FSelectedCategoryID := 0;
  FCurrentPage := 1;
  LoadProducts;
end;

end.
```

---

## แบบฝึกหัด 10 ข้อ

**ข้อ 1:** สร้าง TDBGrid ที่แสดงสีพื้นหลังแถวแตกต่างกันตามเงื่อนไข
```pascal
// สต็อกน้อยกว่า 10: สีแดงอ่อน
// สต็อก 10-50: สีเหลืองอ่อน  
// สต็อกมากกว่า 50: สีเขียวอ่อน
```

**ข้อ 2:** เพิ่มปุ่ม toolbar ลงใน TDBNavigator แบบ custom
```pascal
// ปุ่ม Print
// ปุ่ม Export
// ปุ่ม Refresh
```

**ข้อ 3:** สร้างระบบ search ขั้นสูงพร้อม filter หลายเงื่อนไข
```pascal
// ค้นหาด้วยชื่อ
// กรองด้วยช่วงวันที่
// กรองด้วย status
// กรองด้วย price range
// บันทึก filter criteria ที่ใช้บ่อย
```

**ข้อ 4:** สร้าง DBGrid ที่ edit ในตารางได้โดยตรง
```pascal
// Double-click เพื่อแก้ไข
// Inline editing สำหรับ text fields
// Combobox สำหรับ enum fields
// Date picker สำหรับ date fields
// ยืนยันก่อน save
```

**ข้อ 5:** เขียน export เป็น PDF พร้อม page header/footer
```pascal
// ใช้ FPReport หรือ LazReport
// Header: ชื่อรายงาน, วันที่พิมพ์
// Footer: เลขหน้า, จำนวนรายการทั้งหมด
// Auto page break
```

**ข้อ 6:** สร้าง pivot table อย่างง่ายจาก DBGrid
```pascal
// จัดกลุ่มข้อมูลตาม 1 field
// แสดงยอดรวมแต่ละกลุ่ม
// แสดง sub-total และ grand total
```

**ข้อ 7:** เพิ่มระบบ row highlighting เมื่อ search
```pascal
// Highlight แถวที่ตรงกับคำค้นหา
// Highlight ข้อความที่ตรงกันภายในแถว
// สีต่างกันสำหรับ exact match และ partial match
```

**ข้อ 8:** สร้าง Master-Detail DBGrid
```pascal
// DBGrid บนแสดง orders
// DBGrid ล่างแสดง order items ที่ตรงกับ order ที่เลือก
// อัปเดต detail grid อัตโนมัติเมื่อเลือก master row
```

**ข้อ 9:** เขียนระบบ import จาก CSV เข้า DBGrid
```pascal
// เปิดไฟล์ CSV
// Match columns กับ database fields
// Preview ก่อน import จริง
// Validate ข้อมูล
// Import พร้อม progress bar
```

**ข้อ 10:** โปรเจกต์: สร้าง Data Browser อเนกประสงค์
```pascal
// เลือก table จาก combobox
// แสดงข้อมูลใน DBGrid
// Edit, Add, Delete ได้
// ค้นหาและกรอง
// Export ได้หลายรูปแบบ
// แสดง SQL ที่ใช้
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- TDBGrid และการ customize อย่างละเอียด
- Custom drawing สำหรับ DBGrid
- การจัดเรียงข้อมูลเมื่อคลิก column header
- Live filtering
- TDBEdit, TDBMemo, TDBCheckBox, TDBComboBox
- TDBImage สำหรับรูปภาพ BLOB
- TDBNavigator
- Data binding
- Virtual datasets ด้วย TClientDataSet
- Export DBGrid เป็น CSV, Excel
- DBGrid พร้อม checkboxes
- Product catalog browser สมบูรณ์

Dataset Controls ช่วยให้การพัฒนา database application ใน Lazarus รวดเร็วและมีประสิทธิภาพสูง
