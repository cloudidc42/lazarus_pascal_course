# Part 52 - MVC Architecture ใน Lazarus/Pascal

## บทนำ

MVC (Model-View-Controller) เป็น Architectural Pattern ที่แยกส่วนประกอบของแอปพลิเคชันออกเป็น 3 ส่วน ทำให้โค้ดมีความเป็นระเบียบ บำรุงรักษาได้ง่าย และทดสอบได้ง่ายขึ้น

### โครงสร้าง MVC

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│    Model    │◄────►│ Controller  │◄────►│    View     │
│             │      │             │      │             │
│ - ข้อมูล    │      │ - Logic     │      │ - แสดงผล   │
│ - Business  │      │ - Events    │      │ - Input     │
│   Rules     │      │ - Updates   │      │   UI        │
└─────────────┘      └─────────────┘      └─────────────┘
```

---

## ส่วนที่ 1: Model

Model คือส่วนที่จัดการข้อมูลและ Business Logic ไม่รู้จัก View และ Controller

```pascal
unit ContactModel;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections;

type
  { Contact Data }
  TContact = class
  private
    FID: Integer;
    FFirstName: string;
    FLastName: string;
    FEmail: string;
    FPhone: string;
    FAddress: string;
    FBirthDate: TDateTime;
    FCreatedAt: TDateTime;
  public
    constructor Create;
    constructor CreateWithData(AID: Integer; const AFirst, ALast, AEmail, APhone: string);
    function FullName: string;
    function Age: Integer;
    function IsValid: Boolean;
    function Validate(out AErrors: TStringList): Boolean;
    function Clone: TContact;
    procedure CopyFrom(ASource: TContact);
    
    property ID: Integer read FID write FID;
    property FirstName: string read FFirstName write FFirstName;
    property LastName: string read FLastName write FLastName;
    property Email: string read FEmail write FEmail;
    property Phone: string read FPhone write FPhone;
    property Address: string read FAddress write FAddress;
    property BirthDate: TDateTime read FBirthDate write FBirthDate;
    property CreatedAt: TDateTime read FCreatedAt;
  end;

  TContactList = specialize TObjectList<TContact>;

  { Repository Interface }
  IContactRepository = interface
    ['{REPO-1111-2222-3333-444444444444}']
    function GetAll: TContactList;
    function GetByID(AID: Integer): TContact;
    function FindByName(const AName: string): TContactList;
    function FindByEmail(const AEmail: string): TContact;
    function Save(AContact: TContact): Boolean;
    function Delete(AID: Integer): Boolean;
    function Count: Integer;
  end;

  { In-Memory Repository }
  TContactMemoryRepository = class(TInterfacedObject, IContactRepository)
  private
    FContacts: TContactList;
    FNextID: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    function GetAll: TContactList;
    function GetByID(AID: Integer): TContact;
    function FindByName(const AName: string): TContactList;
    function FindByEmail(const AEmail: string): TContact;
    function Save(AContact: TContact): Boolean;
    function Delete(AID: Integer): Boolean;
    function Count: Integer;
    procedure LoadSampleData;
  end;

  { Contact Service - Business Logic Layer }
  TContactService = class
  private
    FRepository: IContactRepository;
  public
    constructor Create(ARepository: IContactRepository);
    function GetAllContacts: TContactList;
    function GetContact(AID: Integer): TContact;
    function SearchContacts(const AQuery: string): TContactList;
    function AddContact(const AFirst, ALast, AEmail, APhone: string): TContact;
    function UpdateContact(AContact: TContact): Boolean;
    function DeleteContact(AID: Integer): Boolean;
    function GetContactCount: Integer;
    function ImportContacts(const AFileName: string): Integer;
    function ExportContacts(const AFileName: string): Boolean;
  end;

  { Events }
  TContactChangedEvent = procedure(AContact: TContact) of object;
  TContactDeletedEvent = procedure(AID: Integer) of object;

  { Model with Events }
  TContactModel = class
  private
    FService: TContactService;
    FOnContactAdded: TContactChangedEvent;
    FOnContactUpdated: TContactChangedEvent;
    FOnContactDeleted: TContactDeletedEvent;
    FOnDataChanged: TNotifyEvent;
    procedure NotifyDataChanged;
  public
    constructor Create;
    destructor Destroy; override;
    
    { CRUD Operations }
    function GetAll: TContactList;
    function GetByID(AID: Integer): TContact;
    function Search(const AQuery: string): TContactList;
    function Add(const AFirst, ALast, AEmail, APhone: string): TContact;
    function Update(AContact: TContact): Boolean;
    function Delete(AID: Integer): Boolean;
    function Count: Integer;
    
    { Events }
    property OnContactAdded: TContactChangedEvent 
      read FOnContactAdded write FOnContactAdded;
    property OnContactUpdated: TContactChangedEvent 
      read FOnContactUpdated write FOnContactUpdated;
    property OnContactDeleted: TContactDeletedEvent 
      read FOnContactDeleted write FOnContactDeleted;
    property OnDataChanged: TNotifyEvent 
      read FOnDataChanged write FOnDataChanged;
  end;

implementation

{ TContact }
constructor TContact.Create;
begin
  FID := 0;
  FCreatedAt := Now;
end;

constructor TContact.CreateWithData(AID: Integer; const AFirst, ALast, AEmail, APhone: string);
begin
  Create;
  FID := AID;
  FFirstName := AFirst;
  FLastName := ALast;
  FEmail := AEmail;
  FPhone := APhone;
end;

function TContact.FullName: string;
begin
  Result := FFirstName + ' ' + FLastName;
end;

function TContact.Age: Integer;
begin
  if FBirthDate = 0 then
    Result := 0
  else
    Result := YearsBetween(Now, FBirthDate);
end;

function TContact.IsValid: Boolean;
var
  Errors: TStringList;
begin
  Errors := TStringList.Create;
  try
    Result := Validate(Errors);
  finally
    Errors.Free;
  end;
end;

function TContact.Validate(out AErrors: TStringList): Boolean;
begin
  AErrors := TStringList.Create;
  
  if Trim(FFirstName) = '' then
    AErrors.Add('ชื่อต้องไม่ว่าง');
  
  if Trim(FLastName) = '' then
    AErrors.Add('นามสกุลต้องไม่ว่าง');
  
  if FEmail <> '' then
  begin
    if (Pos('@', FEmail) = 0) or (Pos('.', FEmail) = 0) then
      AErrors.Add('รูปแบบอีเมลไม่ถูกต้อง');
  end;
  
  if (FPhone <> '') and (Length(FPhone) < 9) then
    AErrors.Add('เบอร์โทรต้องมีอย่างน้อย 9 หลัก');
  
  Result := AErrors.Count = 0;
end;

function TContact.Clone: TContact;
begin
  Result := TContact.Create;
  Result.CopyFrom(Self);
end;

procedure TContact.CopyFrom(ASource: TContact);
begin
  FID := ASource.FID;
  FFirstName := ASource.FFirstName;
  FLastName := ASource.FLastName;
  FEmail := ASource.FEmail;
  FPhone := ASource.FPhone;
  FAddress := ASource.FAddress;
  FBirthDate := ASource.FBirthDate;
end;

{ TContactMemoryRepository }
constructor TContactMemoryRepository.Create;
begin
  FContacts := TContactList.Create(True);
  FNextID := 1;
end;

destructor TContactMemoryRepository.Destroy;
begin
  FContacts.Free;
  inherited;
end;

function TContactMemoryRepository.GetAll: TContactList;
var
  C: TContact;
  Result_: TContactList;
begin
  Result_ := TContactList.Create(False);
  for C in FContacts do
    Result_.Add(C);
  Result := Result_;
end;

function TContactMemoryRepository.GetByID(AID: Integer): TContact;
var
  C: TContact;
begin
  Result := nil;
  for C in FContacts do
    if C.ID = AID then
    begin
      Result := C;
      Exit;
    end;
end;

function TContactMemoryRepository.FindByName(const AName: string): TContactList;
var
  C: TContact;
  LowerName: string;
begin
  Result := TContactList.Create(False);
  LowerName := LowerCase(AName);
  for C in FContacts do
    if (Pos(LowerName, LowerCase(C.FirstName)) > 0) or
       (Pos(LowerName, LowerCase(C.LastName)) > 0) or
       (Pos(LowerName, LowerCase(C.FullName)) > 0) then
      Result.Add(C);
end;

function TContactMemoryRepository.FindByEmail(const AEmail: string): TContact;
var
  C: TContact;
begin
  Result := nil;
  for C in FContacts do
    if LowerCase(C.Email) = LowerCase(AEmail) then
    begin
      Result := C;
      Exit;
    end;
end;

function TContactMemoryRepository.Save(AContact: TContact): Boolean;
var
  Existing: TContact;
begin
  Result := True;
  if AContact.ID = 0 then
  begin
    { New Contact }
    AContact.ID := FNextID;
    Inc(FNextID);
    FContacts.Add(AContact.Clone);
  end
  else
  begin
    { Update Existing }
    Existing := GetByID(AContact.ID);
    if Existing <> nil then
      Existing.CopyFrom(AContact)
    else
      Result := False;
  end;
end;

function TContactMemoryRepository.Delete(AID: Integer): Boolean;
var
  I: Integer;
begin
  Result := False;
  for I := 0 to FContacts.Count - 1 do
    if FContacts[I].ID = AID then
    begin
      FContacts.Delete(I);
      Result := True;
      Exit;
    end;
end;

function TContactMemoryRepository.Count: Integer;
begin
  Result := FContacts.Count;
end;

procedure TContactMemoryRepository.LoadSampleData;
var
  C: TContact;
begin
  C := TContact.CreateWithData(0, 'สมชาย', 'ใจดี', 'somchai@email.com', '0812345678');
  Save(C); C.Free;

  C := TContact.CreateWithData(0, 'สมหญิง', 'สวยงาม', 'somying@email.com', '0898765432');
  Save(C); C.Free;

  C := TContact.CreateWithData(0, 'วิชัย', 'รักเรียน', 'wichai@email.com', '0856789012');
  Save(C); C.Free;

  C := TContact.CreateWithData(0, 'นงนุช', 'มีสุข', 'nongnuch@email.com', '0823456789');
  Save(C); C.Free;
end;

{ TContactService }
constructor TContactService.Create(ARepository: IContactRepository);
begin
  FRepository := ARepository;
end;

function TContactService.GetAllContacts: TContactList;
begin
  Result := FRepository.GetAll;
end;

function TContactService.GetContact(AID: Integer): TContact;
begin
  Result := FRepository.GetByID(AID);
end;

function TContactService.SearchContacts(const AQuery: string): TContactList;
begin
  if Trim(AQuery) = '' then
    Result := FRepository.GetAll
  else
    Result := FRepository.FindByName(AQuery);
end;

function TContactService.AddContact(const AFirst, ALast, AEmail, APhone: string): TContact;
var
  NewContact: TContact;
  Errors: TStringList;
begin
  NewContact := TContact.CreateWithData(0, AFirst, ALast, AEmail, APhone);
  if not NewContact.Validate(Errors) then
  begin
    Errors.Free;
    NewContact.Free;
    raise Exception.Create('ข้อมูลไม่ถูกต้อง');
  end;
  Errors.Free;
  
  if FRepository.Save(NewContact) then
    Result := NewContact
  else
  begin
    NewContact.Free;
    Result := nil;
  end;
end;

function TContactService.UpdateContact(AContact: TContact): Boolean;
var
  Errors: TStringList;
begin
  if not AContact.Validate(Errors) then
  begin
    Errors.Free;
    raise Exception.Create('ข้อมูลไม่ถูกต้อง');
  end;
  Errors.Free;
  Result := FRepository.Save(AContact);
end;

function TContactService.DeleteContact(AID: Integer): Boolean;
begin
  Result := FRepository.Delete(AID);
end;

function TContactService.GetContactCount: Integer;
begin
  Result := FRepository.Count;
end;

function TContactService.ImportContacts(const AFileName: string): Integer;
var
  Lines: TStringList;
  I: Integer;
  Parts: TStringArray;
  C: TContact;
begin
  Result := 0;
  Lines := TStringList.Create;
  try
    Lines.LoadFromFile(AFileName);
    for I := 1 to Lines.Count - 1 do  { ข้าม Header }
    begin
      Parts := Lines[I].Split([',']);
      if Length(Parts) >= 4 then
      begin
        C := TContact.CreateWithData(0, Parts[0], Parts[1], Parts[2], Parts[3]);
        if FRepository.Save(C) then
          Inc(Result);
        C.Free;
      end;
    end;
  finally
    Lines.Free;
  end;
end;

function TContactService.ExportContacts(const AFileName: string): Boolean;
var
  Lines: TStringList;
  Contacts: TContactList;
  C: TContact;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('ชื่อ,นามสกุล,อีเมล,เบอร์โทร,ที่อยู่');
    Contacts := FRepository.GetAll;
    try
      for C in Contacts do
        Lines.Add(Format('%s,%s,%s,%s,%s',
          [C.FirstName, C.LastName, C.Email, C.Phone, C.Address]));
    finally
      Contacts.Free;
    end;
    Lines.SaveToFile(AFileName);
    Result := True;
  except
    Result := False;
  end;
  Lines.Free;
end;

{ TContactModel }
constructor TContactModel.Create;
var
  Repo: TContactMemoryRepository;
begin
  Repo := TContactMemoryRepository.Create;
  Repo.LoadSampleData;
  FService := TContactService.Create(Repo);
end;

destructor TContactModel.Destroy;
begin
  FService.Free;
  inherited;
end;

procedure TContactModel.NotifyDataChanged;
begin
  if Assigned(FOnDataChanged) then
    FOnDataChanged(Self);
end;

function TContactModel.GetAll: TContactList;
begin
  Result := FService.GetAllContacts;
end;

function TContactModel.GetByID(AID: Integer): TContact;
begin
  Result := FService.GetContact(AID);
end;

function TContactModel.Search(const AQuery: string): TContactList;
begin
  Result := FService.SearchContacts(AQuery);
end;

function TContactModel.Add(const AFirst, ALast, AEmail, APhone: string): TContact;
begin
  Result := FService.AddContact(AFirst, ALast, AEmail, APhone);
  if Result <> nil then
  begin
    if Assigned(FOnContactAdded) then FOnContactAdded(Result);
    NotifyDataChanged;
  end;
end;

function TContactModel.Update(AContact: TContact): Boolean;
begin
  Result := FService.UpdateContact(AContact);
  if Result then
  begin
    if Assigned(FOnContactUpdated) then FOnContactUpdated(AContact);
    NotifyDataChanged;
  end;
end;

function TContactModel.Delete(AID: Integer): Boolean;
begin
  Result := FService.DeleteContact(AID);
  if Result then
  begin
    if Assigned(FOnContactDeleted) then FOnContactDeleted(AID);
    NotifyDataChanged;
  end;
end;

function TContactModel.Count: Integer;
begin
  Result := FService.GetContactCount;
end;

end.
```

---

## ส่วนที่ 2: View

View คือส่วนที่แสดงผลให้ User เห็น ใน Lazarus View คือ Form และ Control ต่างๆ

```pascal
unit ContactView;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Grids, StdCtrls, 
  ExtCtrls, Buttons, ComCtrls, Dialogs, ContactModel;

type
  { Interface สำหรับ View }
  IContactView = interface
    ['{VIEW-1111-2222-3333-444444444444}']
    procedure ShowContacts(AContacts: TContactList);
    procedure ShowContact(AContact: TContact);
    procedure ClearForm;
    procedure ShowError(const AMessage: string);
    procedure ShowSuccess(const AMessage: string);
    function GetSelectedContactID: Integer;
    function GetFirstName: string;
    function GetLastName: string;
    function GetEmail: string;
    function GetPhone: string;
    function GetSearchText: string;
    procedure SetStatusText(const AText: string);
  end;

  { Main Contact Form }
  TContactForm = class(TForm, IContactView)
  private
    { UI Controls }
    FGrid: TStringGrid;
    FEdtFirstName: TEdit;
    FEdtLastName: TEdit;
    FEdtEmail: TEdit;
    FEdtPhone: TEdit;
    FEdtSearch: TEdit;
    FBtnAdd: TButton;
    FBtnUpdate: TButton;
    FBtnDelete: TButton;
    FBtnClear: TButton;
    FBtnSearch: TButton;
    FBtnImport: TButton;
    FBtnExport: TButton;
    FStatusBar: TStatusBar;
    FPanelForm: TPanel;
    FPanelGrid: TPanel;
    FPanelSearch: TPanel;
    
    { Event Handlers - จะถูก Assign โดย Controller }
    FOnAddContact: TNotifyEvent;
    FOnUpdateContact: TNotifyEvent;
    FOnDeleteContact: TNotifyEvent;
    FOnSearchContact: TNotifyEvent;
    FOnContactSelected: TNotifyEvent;
    FOnImport: TNotifyEvent;
    FOnExport: TNotifyEvent;
    
    procedure SetupUI;
    procedure SetupGrid;
    procedure BtnAddClick(Sender: TObject);
    procedure BtnUpdateClick(Sender: TObject);
    procedure BtnDeleteClick(Sender: TObject);
    procedure BtnClearClick(Sender: TObject);
    procedure BtnSearchClick(Sender: TObject);
    procedure GridSelectCell(Sender: TObject; ACol, ARow: Integer; 
                             var CanSelect: Boolean);
    procedure EdtSearchKeyPress(Sender: TObject; var Key: Char);
  public
    constructor Create(AOwner: TComponent); override;
    
    { IContactView Implementation }
    procedure ShowContacts(AContacts: TContactList);
    procedure ShowContact(AContact: TContact);
    procedure ClearForm;
    procedure ShowError(const AMessage: string);
    procedure ShowSuccess(const AMessage: string);
    function GetSelectedContactID: Integer;
    function GetFirstName: string;
    function GetLastName: string;
    function GetEmail: string;
    function GetPhone: string;
    function GetSearchText: string;
    procedure SetStatusText(const AText: string);
    
    { Events ที่ Controller จะ Subscribe }
    property OnAddContact: TNotifyEvent read FOnAddContact write FOnAddContact;
    property OnUpdateContact: TNotifyEvent read FOnUpdateContact write FOnUpdateContact;
    property OnDeleteContact: TNotifyEvent read FOnDeleteContact write FOnDeleteContact;
    property OnSearchContact: TNotifyEvent read FOnSearchContact write FOnSearchContact;
    property OnContactSelected: TNotifyEvent read FOnContactSelected write FOnContactSelected;
    property OnImport: TNotifyEvent read FOnImport write FOnImport;
    property OnExport: TNotifyEvent read FOnExport write FOnExport;
  end;

implementation

constructor TContactForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  Caption := 'ระบบจัดการผู้ติดต่อ - MVC Demo';
  Width := 900;
  Height := 600;
  SetupUI;
end;

procedure TContactForm.SetupUI;
begin
  { Panel สำหรับ Search }
  FPanelSearch := TPanel.Create(Self);
  FPanelSearch.Parent := Self;
  FPanelSearch.Align := alTop;
  FPanelSearch.Height := 50;
  FPanelSearch.BevelOuter := bvNone;
  
  FEdtSearch := TEdit.Create(FPanelSearch);
  FEdtSearch.Parent := FPanelSearch;
  FEdtSearch.Left := 10;
  FEdtSearch.Top := 12;
  FEdtSearch.Width := 300;
  FEdtSearch.TextHint := 'ค้นหาชื่อหรือนามสกุล...';
  FEdtSearch.OnKeyPress := EdtSearchKeyPress;
  
  FBtnSearch := TButton.Create(FPanelSearch);
  FBtnSearch.Parent := FPanelSearch;
  FBtnSearch.Left := 320;
  FBtnSearch.Top := 10;
  FBtnSearch.Width := 80;
  FBtnSearch.Caption := 'ค้นหา';
  FBtnSearch.OnClick := BtnSearchClick;
  
  FBtnImport := TButton.Create(FPanelSearch);
  FBtnImport.Parent := FPanelSearch;
  FBtnImport.Left := 410;
  FBtnImport.Top := 10;
  FBtnImport.Width := 80;
  FBtnImport.Caption := 'นำเข้า';
  
  FBtnExport := TButton.Create(FPanelSearch);
  FBtnExport.Parent := FPanelSearch;
  FBtnExport.Left := 500;
  FBtnExport.Top := 10;
  FBtnExport.Width := 80;
  FBtnExport.Caption := 'ส่งออก';
  
  { Panel สำหรับ Form Input }
  FPanelForm := TPanel.Create(Self);
  FPanelForm.Parent := Self;
  FPanelForm.Align := alRight;
  FPanelForm.Width := 280;
  FPanelForm.BevelOuter := bvLowered;
  FPanelForm.Caption := '';
  
  with TLabel.Create(FPanelForm) do
  begin
    Parent := FPanelForm;
    Left := 10; Top := 15;
    Caption := 'ชื่อ:';
  end;
  FEdtFirstName := TEdit.Create(FPanelForm);
  FEdtFirstName.Parent := FPanelForm;
  FEdtFirstName.Left := 10; FEdtFirstName.Top := 32;
  FEdtFirstName.Width := 250;
  
  with TLabel.Create(FPanelForm) do
  begin
    Parent := FPanelForm;
    Left := 10; Top := 62;
    Caption := 'นามสกุล:';
  end;
  FEdtLastName := TEdit.Create(FPanelForm);
  FEdtLastName.Parent := FPanelForm;
  FEdtLastName.Left := 10; FEdtLastName.Top := 79;
  FEdtLastName.Width := 250;
  
  with TLabel.Create(FPanelForm) do
  begin
    Parent := FPanelForm;
    Left := 10; Top := 109;
    Caption := 'อีเมล:';
  end;
  FEdtEmail := TEdit.Create(FPanelForm);
  FEdtEmail.Parent := FPanelForm;
  FEdtEmail.Left := 10; FEdtEmail.Top := 126;
  FEdtEmail.Width := 250;
  
  with TLabel.Create(FPanelForm) do
  begin
    Parent := FPanelForm;
    Left := 10; Top := 156;
    Caption := 'เบอร์โทร:';
  end;
  FEdtPhone := TEdit.Create(FPanelForm);
  FEdtPhone.Parent := FPanelForm;
  FEdtPhone.Left := 10; FEdtPhone.Top := 173;
  FEdtPhone.Width := 250;
  
  { Buttons }
  FBtnAdd := TButton.Create(FPanelForm);
  FBtnAdd.Parent := FPanelForm;
  FBtnAdd.Left := 10; FBtnAdd.Top := 220;
  FBtnAdd.Width := 115; FBtnAdd.Height := 35;
  FBtnAdd.Caption := 'เพิ่มผู้ติดต่อ';
  FBtnAdd.OnClick := BtnAddClick;
  
  FBtnUpdate := TButton.Create(FPanelForm);
  FBtnUpdate.Parent := FPanelForm;
  FBtnUpdate.Left := 135; FBtnUpdate.Top := 220;
  FBtnUpdate.Width := 125; FBtnUpdate.Height := 35;
  FBtnUpdate.Caption := 'อัปเดต';
  FBtnUpdate.OnClick := BtnUpdateClick;
  FBtnUpdate.Enabled := False;
  
  FBtnDelete := TButton.Create(FPanelForm);
  FBtnDelete.Parent := FPanelForm;
  FBtnDelete.Left := 10; FBtnDelete.Top := 265;
  FBtnDelete.Width := 115; FBtnDelete.Height := 35;
  FBtnDelete.Caption := 'ลบ';
  FBtnDelete.OnClick := BtnDeleteClick;
  FBtnDelete.Enabled := False;
  
  FBtnClear := TButton.Create(FPanelForm);
  FBtnClear.Parent := FPanelForm;
  FBtnClear.Left := 135; FBtnClear.Top := 265;
  FBtnClear.Width := 125; FBtnClear.Height := 35;
  FBtnClear.Caption := 'ล้างฟอร์ม';
  FBtnClear.OnClick := BtnClearClick;
  
  { Grid }
  FPanelGrid := TPanel.Create(Self);
  FPanelGrid.Parent := Self;
  FPanelGrid.Align := alClient;
  FPanelGrid.BevelOuter := bvNone;
  
  SetupGrid;
  
  { Status Bar }
  FStatusBar := TStatusBar.Create(Self);
  FStatusBar.Parent := Self;
  FStatusBar.SimplePanel := True;
  FStatusBar.SimpleText := 'พร้อมใช้งาน';
end;

procedure TContactForm.SetupGrid;
begin
  FGrid := TStringGrid.Create(FPanelGrid);
  FGrid.Parent := FPanelGrid;
  FGrid.Align := alClient;
  FGrid.RowCount := 2;
  FGrid.ColCount := 5;
  FGrid.FixedRows := 1;
  FGrid.Options := FGrid.Options + [goColSizing, goRowSelect];
  FGrid.OnSelectCell := GridSelectCell;
  
  { Header }
  FGrid.Cells[0, 0] := 'ID';
  FGrid.Cells[1, 0] := 'ชื่อ';
  FGrid.Cells[2, 0] := 'นามสกุล';
  FGrid.Cells[3, 0] := 'อีเมล';
  FGrid.Cells[4, 0] := 'เบอร์โทร';
  
  { Column Widths }
  FGrid.ColWidths[0] := 40;
  FGrid.ColWidths[1] := 120;
  FGrid.ColWidths[2] := 120;
  FGrid.ColWidths[3] := 200;
  FGrid.ColWidths[4] := 120;
end;

procedure TContactForm.ShowContacts(AContacts: TContactList);
var
  I: Integer;
  C: TContact;
begin
  FGrid.RowCount := AContacts.Count + 1;
  if FGrid.RowCount < 2 then FGrid.RowCount := 2;
  
  for I := 0 to AContacts.Count - 1 do
  begin
    C := AContacts[I];
    FGrid.Cells[0, I + 1] := IntToStr(C.ID);
    FGrid.Cells[1, I + 1] := C.FirstName;
    FGrid.Cells[2, I + 1] := C.LastName;
    FGrid.Cells[3, I + 1] := C.Email;
    FGrid.Cells[4, I + 1] := C.Phone;
  end;
end;

procedure TContactForm.ShowContact(AContact: TContact);
begin
  if AContact = nil then
  begin
    ClearForm;
    Exit;
  end;
  FEdtFirstName.Text := AContact.FirstName;
  FEdtLastName.Text := AContact.LastName;
  FEdtEmail.Text := AContact.Email;
  FEdtPhone.Text := AContact.Phone;
  FBtnUpdate.Enabled := True;
  FBtnDelete.Enabled := True;
end;

procedure TContactForm.ClearForm;
begin
  FEdtFirstName.Clear;
  FEdtLastName.Clear;
  FEdtEmail.Clear;
  FEdtPhone.Clear;
  FBtnUpdate.Enabled := False;
  FBtnDelete.Enabled := False;
  if FGrid.Row > 0 then FGrid.Row := 0;
end;

procedure TContactForm.ShowError(const AMessage: string);
begin
  MessageDlg('ข้อผิดพลาด', AMessage, mtError, [mbOK], 0);
  SetStatusText('ข้อผิดพลาด: ' + AMessage);
end;

procedure TContactForm.ShowSuccess(const AMessage: string);
begin
  SetStatusText(AMessage);
end;

function TContactForm.GetSelectedContactID: Integer;
begin
  if FGrid.Row > 0 then
    Result := StrToIntDef(FGrid.Cells[0, FGrid.Row], 0)
  else
    Result := 0;
end;

function TContactForm.GetFirstName: string;
begin Result := FEdtFirstName.Text; end;

function TContactForm.GetLastName: string;
begin Result := FEdtLastName.Text; end;

function TContactForm.GetEmail: string;
begin Result := FEdtEmail.Text; end;

function TContactForm.GetPhone: string;
begin Result := FEdtPhone.Text; end;

function TContactForm.GetSearchText: string;
begin Result := FEdtSearch.Text; end;

procedure TContactForm.SetStatusText(const AText: string);
begin
  FStatusBar.SimpleText := AText;
end;

procedure TContactForm.BtnAddClick(Sender: TObject);
begin
  if Assigned(FOnAddContact) then FOnAddContact(Self);
end;

procedure TContactForm.BtnUpdateClick(Sender: TObject);
begin
  if Assigned(FOnUpdateContact) then FOnUpdateContact(Self);
end;

procedure TContactForm.BtnDeleteClick(Sender: TObject);
begin
  if Assigned(FOnDeleteContact) then FOnDeleteContact(Self);
end;

procedure TContactForm.BtnClearClick(Sender: TObject);
begin
  ClearForm;
end;

procedure TContactForm.BtnSearchClick(Sender: TObject);
begin
  if Assigned(FOnSearchContact) then FOnSearchContact(Self);
end;

procedure TContactForm.GridSelectCell(Sender: TObject; ACol, ARow: Integer;
  var CanSelect: Boolean);
begin
  CanSelect := True;
  if (ARow > 0) and Assigned(FOnContactSelected) then
    FOnContactSelected(Self);
end;

procedure TContactForm.EdtSearchKeyPress(Sender: TObject; var Key: Char);
begin
  if Key = #13 then BtnSearchClick(nil);
end;

end.
```

---

## ส่วนที่ 3: Controller

Controller เป็นตัวกลางระหว่าง Model และ View จัดการ Event และอัปเดต View

```pascal
unit ContactController;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Dialogs, ContactModel, ContactView;

type
  TContactController = class
  private
    FModel: TContactModel;
    FView: TContactForm;
    FCurrentContactID: Integer;
    
    procedure HandleAddContact(Sender: TObject);
    procedure HandleUpdateContact(Sender: TObject);
    procedure HandleDeleteContact(Sender: TObject);
    procedure HandleSearchContact(Sender: TObject);
    procedure HandleContactSelected(Sender: TObject);
    procedure HandleImport(Sender: TObject);
    procedure HandleExport(Sender: TObject);
    procedure HandleModelDataChanged(Sender: TObject);
    procedure RefreshView;
    procedure RefreshView(AContacts: TContactList);
  public
    constructor Create(AModel: TContactModel; AView: TContactForm);
    destructor Destroy; override;
    procedure Initialize;
  end;

implementation

constructor TContactController.Create(AModel: TContactModel; AView: TContactForm);
begin
  FModel := AModel;
  FView := AView;
  FCurrentContactID := 0;
end;

destructor TContactController.Destroy;
begin
  inherited;
end;

procedure TContactController.Initialize;
begin
  { ผูก View Events กับ Controller Handlers }
  FView.OnAddContact := HandleAddContact;
  FView.OnUpdateContact := HandleUpdateContact;
  FView.OnDeleteContact := HandleDeleteContact;
  FView.OnSearchContact := HandleSearchContact;
  FView.OnContactSelected := HandleContactSelected;
  FView.OnImport := HandleImport;
  FView.OnExport := HandleExport;
  
  { ผูก Model Events กับ Controller Handlers }
  FModel.OnDataChanged := HandleModelDataChanged;
  
  { โหลดข้อมูลครั้งแรก }
  RefreshView;
  FView.SetStatusText(Format('โหลดข้อมูลแล้ว: %d รายการ', [FModel.Count]));
end;

procedure TContactController.RefreshView;
var
  Contacts: TContactList;
begin
  Contacts := FModel.GetAll;
  try
    RefreshView(Contacts);
  finally
    Contacts.Free;
  end;
end;

procedure TContactController.RefreshView(AContacts: TContactList);
begin
  FView.ShowContacts(AContacts);
  FView.SetStatusText(Format('แสดง %d รายการ', [AContacts.Count]));
end;

procedure TContactController.HandleAddContact(Sender: TObject);
var
  NewContact: TContact;
begin
  try
    NewContact := FModel.Add(
      FView.GetFirstName,
      FView.GetLastName,
      FView.GetEmail,
      FView.GetPhone
    );
    if NewContact <> nil then
    begin
      FView.ShowSuccess(Format('เพิ่ม "%s %s" เรียบร้อย', 
        [NewContact.FirstName, NewContact.LastName]));
      FView.ClearForm;
    end;
  except
    on E: Exception do
      FView.ShowError(E.Message);
  end;
end;

procedure TContactController.HandleUpdateContact(Sender: TObject);
var
  Contact: TContact;
begin
  if FCurrentContactID = 0 then
  begin
    FView.ShowError('กรุณาเลือกผู้ติดต่อที่ต้องการแก้ไข');
    Exit;
  end;
  
  Contact := FModel.GetByID(FCurrentContactID);
  if Contact = nil then
  begin
    FView.ShowError('ไม่พบผู้ติดต่อที่ต้องการแก้ไข');
    Exit;
  end;
  
  Contact.FirstName := FView.GetFirstName;
  Contact.LastName := FView.GetLastName;
  Contact.Email := FView.GetEmail;
  Contact.Phone := FView.GetPhone;
  
  try
    if FModel.Update(Contact) then
    begin
      FView.ShowSuccess('อัปเดตข้อมูลเรียบร้อย');
      FView.ClearForm;
      FCurrentContactID := 0;
    end
    else
      FView.ShowError('ไม่สามารถอัปเดตข้อมูลได้');
  except
    on E: Exception do
      FView.ShowError(E.Message);
  end;
end;

procedure TContactController.HandleDeleteContact(Sender: TObject);
var
  Contact: TContact;
begin
  if FCurrentContactID = 0 then
  begin
    FView.ShowError('กรุณาเลือกผู้ติดต่อที่ต้องการลบ');
    Exit;
  end;
  
  Contact := FModel.GetByID(FCurrentContactID);
  if (Contact <> nil) and 
     (MessageDlg('ยืนยันการลบ', 
      Format('ต้องการลบ "%s" ?', [Contact.FullName]), 
      mtConfirmation, [mbYes, mbNo], 0) = mrYes) then
  begin
    if FModel.Delete(FCurrentContactID) then
    begin
      FView.ShowSuccess('ลบข้อมูลเรียบร้อย');
      FView.ClearForm;
      FCurrentContactID := 0;
    end
    else
      FView.ShowError('ไม่สามารถลบข้อมูลได้');
  end;
end;

procedure TContactController.HandleSearchContact(Sender: TObject);
var
  Contacts: TContactList;
begin
  Contacts := FModel.Search(FView.GetSearchText);
  try
    RefreshView(Contacts);
    if Contacts.Count = 0 then
      FView.SetStatusText('ไม่พบผู้ติดต่อที่ค้นหา')
    else
      FView.SetStatusText(Format('พบ %d รายการ', [Contacts.Count]));
  finally
    Contacts.Free;
  end;
end;

procedure TContactController.HandleContactSelected(Sender: TObject);
var
  Contact: TContact;
begin
  FCurrentContactID := FView.GetSelectedContactID;
  if FCurrentContactID > 0 then
  begin
    Contact := FModel.GetByID(FCurrentContactID);
    FView.ShowContact(Contact);
  end;
end;

procedure TContactController.HandleImport(Sender: TObject);
var
  Dialog: TOpenDialog;
  Count: Integer;
begin
  Dialog := TOpenDialog.Create(nil);
  try
    Dialog.Filter := 'CSV Files (*.csv)|*.csv|All Files (*.*)|*.*';
    Dialog.Title := 'เลือกไฟล์ CSV สำหรับนำเข้า';
    if Dialog.Execute then
    begin
      Count := FModel.GetAll.Count;
      { นำเข้าข้อมูล... }
      FView.ShowSuccess(Format('นำเข้าข้อมูลเรียบร้อย'));
    end;
  finally
    Dialog.Free;
  end;
end;

procedure TContactController.HandleExport(Sender: TObject);
var
  Dialog: TSaveDialog;
begin
  Dialog := TSaveDialog.Create(nil);
  try
    Dialog.Filter := 'CSV Files (*.csv)|*.csv';
    Dialog.Title := 'บันทึกไฟล์ CSV';
    Dialog.DefaultExt := 'csv';
    if Dialog.Execute then
      FView.ShowSuccess('ส่งออกข้อมูลเรียบร้อย');
  finally
    Dialog.Free;
  end;
end;

procedure TContactController.HandleModelDataChanged(Sender: TObject);
begin
  RefreshView;
end;

end.
```

---

## MVP Pattern (Model-View-Presenter)

MVP คล้ายกับ MVC แต่ View ไม่รู้จัก Model โดยตรง ผ่าน Presenter ทั้งหมด

```pascal
unit MVPExample;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Model }
  TLoginModel = class
  private
    FUsers: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    function ValidateCredentials(const AUsername, APassword: string): Boolean;
    function GetUserRole(const AUsername: string): string;
  end;

  { View Interface }
  ILoginView = interface
    ['{LOGIN-VIEW-1111-2222-3333}']
    function GetUsername: string;
    function GetPassword: string;
    procedure ShowError(const AMessage: string);
    procedure ShowSuccess(const AMessage: string);
    procedure SetLoginButtonEnabled(AEnabled: Boolean);
    procedure NavigateToMain(const ARole: string);
  end;

  { Presenter }
  TLoginPresenter = class
  private
    FModel: TLoginModel;
    FView: ILoginView;
    FLoginAttempts: Integer;
    procedure CheckLoginLock;
  public
    constructor Create(AModel: TLoginModel; AView: ILoginView);
    procedure HandleLogin;
    procedure HandleForgotPassword;
    function IsValidInput: Boolean;
  end;

implementation

{ TLoginModel }
constructor TLoginModel.Create;
begin
  FUsers := TStringList.Create;
  { เพิ่มข้อมูล User จำลอง (password:role) }
  FUsers.Values['admin'] := 'admin123:Administrator';
  FUsers.Values['user1'] := 'pass123:User';
  FUsers.Values['manager'] := 'mgr456:Manager';
end;

destructor TLoginModel.Destroy;
begin
  FUsers.Free;
  inherited;
end;

function TLoginModel.ValidateCredentials(const AUsername, APassword: string): Boolean;
var
  StoredData, StoredPass: string;
begin
  StoredData := FUsers.Values[AUsername];
  if StoredData = '' then
  begin
    Result := False;
    Exit;
  end;
  StoredPass := Copy(StoredData, 1, Pos(':', StoredData) - 1);
  Result := StoredPass = APassword;
end;

function TLoginModel.GetUserRole(const AUsername: string): string;
var
  StoredData: string;
begin
  StoredData := FUsers.Values[AUsername];
  if StoredData = '' then
    Result := ''
  else
    Result := Copy(StoredData, Pos(':', StoredData) + 1, MaxInt);
end;

{ TLoginPresenter }
constructor TLoginPresenter.Create(AModel: TLoginModel; AView: ILoginView);
begin
  FModel := AModel;
  FView := AView;
  FLoginAttempts := 0;
end;

procedure TLoginPresenter.CheckLoginLock;
begin
  if FLoginAttempts >= 3 then
  begin
    FView.SetLoginButtonEnabled(False);
    FView.ShowError('บัญชีถูกล็อกชั่วคราว เนื่องจากเข้าสู่ระบบผิดพลาดเกิน 3 ครั้ง');
  end;
end;

function TLoginPresenter.IsValidInput: Boolean;
begin
  Result := False;
  
  if Trim(FView.GetUsername) = '' then
  begin
    FView.ShowError('กรุณากรอกชื่อผู้ใช้');
    Exit;
  end;
  
  if Trim(FView.GetPassword) = '' then
  begin
    FView.ShowError('กรุณากรอกรหัสผ่าน');
    Exit;
  end;
  
  if Length(FView.GetPassword) < 6 then
  begin
    FView.ShowError('รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร');
    Exit;
  end;
  
  Result := True;
end;

procedure TLoginPresenter.HandleLogin;
var
  Role: string;
begin
  if FLoginAttempts >= 3 then
  begin
    CheckLoginLock;
    Exit;
  end;
  
  if not IsValidInput then Exit;
  
  if FModel.ValidateCredentials(FView.GetUsername, FView.GetPassword) then
  begin
    FLoginAttempts := 0;
    Role := FModel.GetUserRole(FView.GetUsername);
    FView.ShowSuccess('เข้าสู่ระบบสำเร็จ');
    FView.NavigateToMain(Role);
  end
  else
  begin
    Inc(FLoginAttempts);
    FView.ShowError(Format('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง (ลองผิดแล้ว %d ครั้ง)', 
      [FLoginAttempts]));
    CheckLoginLock;
  end;
end;

procedure TLoginPresenter.HandleForgotPassword;
begin
  { นำทางไปหน้า Reset Password }
  FView.ShowSuccess('ระบบส่งอีเมลรีเซ็ตรหัสผ่านแล้ว กรุณาตรวจสอบอีเมลของคุณ');
end;

end.
```

---

## แบบฝึกหัด

### ข้อ 1 - Contact Book MVC
พัฒนาต่อจากตัวอย่าง Contact Book ให้สมบูรณ์:
- เพิ่มฟิลด์ Address, Birthday
- Validate ข้อมูลก่อน Save
- เพิ่มฟังก์ชัน Group (จัดกลุ่ม เช่น ครอบครัว, เพื่อน, ที่ทำงาน)
- Export เป็น CSV

### ข้อ 2 - Todo App MVP
สร้าง Todo Application ด้วย MVP Pattern:
- เพิ่ม/แก้ไข/ลบ Todo
- Mark ว่าเสร็จแล้ว
- Filter: All, Active, Completed
- บันทึกข้อมูลลงไฟล์

### ข้อ 3 - InventorySystem MVC
สร้างระบบจัดการสินค้าคงคลัง:
- Model: สินค้า, คลังสินค้า
- View: รายการสินค้า, รายงาน
- Controller: เพิ่ม/ลดสต็อก, ค้นหา

### ข้อ 4 - Login MVVM
แปลง MVP Login เป็น MVVM โดยใช้ Data Binding

### ข้อ 5 - Calculator MVP
สร้าง Calculator ด้วย MVP:
- Model: Logic การคำนวณ
- Presenter: จัดการ Input Validation
- View: แสดงผล
