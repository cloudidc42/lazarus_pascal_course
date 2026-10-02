# Part 39 - Network Programming

## บทนำ

Network Programming คือการเขียนโปรแกรมที่สื่อสารผ่านเครือข่าย ไม่ว่าจะเป็น LAN หรืออินเทอร์เน็ต Lazarus มี Indy (Internet Direct) library ที่ทรงพลังสำหรับงาน Network Programming รวมถึง Synapse library ที่เบากว่า

---

## 39.1 แนวคิดพื้นฐาน TCP/IP

### TCP/IP และ Sockets

**TCP (Transmission Control Protocol)** คือ Protocol ที่น่าเชื่อถือ มีการยืนยันการส่งข้อมูล ใช้สำหรับ:
- Web browsing (HTTP/HTTPS)
- File transfer (FTP)
- Email (SMTP, POP3, IMAP)
- Chat applications

**UDP (User Datagram Protocol)** คือ Protocol ที่เร็วแต่ไม่มีการยืนยัน ใช้สำหรับ:
- Online gaming
- Video streaming
- DNS queries
- VoIP

```
[Client] ----TCP/IP---- [Server]
   |                       |
   | Connect()             |
   |----SYN-------------->  |
   |<----SYN-ACK----------  |
   | ---ACK-------------->  |
   |  (Connection เปิดแล้ว)  |
   | ---Data------------>   |
   |<----Data-----------    |
   | ---Close----------->   |
```

### Ports

Port คือหมายเลขที่ระบุว่าข้อมูลนั้นส่งไปยัง Service ใด:
- 80: HTTP
- 443: HTTPS
- 21: FTP
- 22: SSH
- 25: SMTP
- 110: POP3
- 143: IMAP
- 3306: MySQL
- 5432: PostgreSQL
- 1-1023: Well-known ports (ต้องการ admin)
- 1024-49151: Registered ports
- 49152-65535: Dynamic/Private ports

---

## 39.2 Indy Components

Indy (Internet Direct) เป็น Library สำหรับ Network Programming ใน Delphi/Lazarus

### การติดตั้ง Indy

ใน Lazarus Indy มักมาพร้อมกันแล้ว แต่ถ้าไม่มีให้:
1. ดาวน์โหลดจาก https://www.indyproject.org/
2. หรือใน Package Manager: `lazarus-ide-components` 
3. เพิ่ม `Indy10` หรือ `IndyCore, IndySystem, IndyProtocols` ใน uses

```pascal
// ตรวจสอบว่า Indy มีหรือไม่
uses
  IdTCPClient, IdTCPServer, IdGlobal;
```

---

## 39.3 TCP Client

```pascal
unit TCPClientDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  IdTCPClient, IdGlobal, IdComponent;

type
  TForm1 = class(TForm)
    EditHost: TEdit;
    EditPort: TEdit;
    btnConnect: TButton;
    btnDisconnect: TButton;
    btnSend: TButton;
    MemoSend: TMemo;
    MemoReceive: TMemo;
    lblStatus: TLabel;
    IdTCPClient1: TIdTCPClient;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnConnectClick(Sender: TObject);
    procedure btnDisconnectClick(Sender: TObject);
    procedure btnSendClick(Sender: TObject);
  private
    FReceiveThread: TThread;
    procedure Log(const Msg: string);
    procedure StartReceiving;
    procedure StopReceiving;
    procedure UpdateStatus(const Status: string);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

type
  TReceiveThread = class(TThread)
  private
    FClient: TIdTCPClient;
    FOwner: TForm1;
    FMessage: string;
    procedure ShowMessage;
  protected
    procedure Execute; override;
  public
    constructor Create(AClient: TIdTCPClient; AOwner: TForm1);
  end;

constructor TReceiveThread.Create(AClient: TIdTCPClient; AOwner: TForm1);
begin
  inherited Create(False);
  FClient := AClient;
  FOwner := AOwner;
  FreeOnTerminate := True;
end;

procedure TReceiveThread.ShowMessage;
begin
  FOwner.Log('Received: ' + FMessage);
end;

procedure TReceiveThread.Execute;
var
  Line: string;
begin
  while not Terminated do
  begin
    try
      if FClient.Connected then
      begin
        // รับข้อมูลแบบ Line
        Line := FClient.IOHandler.ReadLn('', 1000);  // 1 second timeout
        if Line <> '' then
        begin
          FMessage := Line;
          Synchronize(ShowMessage);
        end;
      end else
        Break;
    except
      on E: EIdReadTimeout do
        Continue;  // Timeout is OK, try again
      on E: Exception do
      begin
        if not Terminated then
        begin
          FMessage := 'Error: ' + E.Message;
          Synchronize(ShowMessage);
        end;
        Break;
      end;
    end;
  end;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FReceiveThread := nil;
  IdTCPClient1.ConnectTimeout := 5000;
  IdTCPClient1.ReadTimeout := 1000;
  EditHost.Text := '127.0.0.1';
  EditPort.Text := '9999';
  UpdateStatus('ยังไม่ได้เชื่อมต่อ');
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  StopReceiving;
  if IdTCPClient1.Connected then
    IdTCPClient1.Disconnect;
end;

procedure TForm1.Log(const Msg: string);
begin
  MemoReceive.Lines.Add(FormatDateTime('[hh:nn:ss] ', Now) + Msg);
  // Scroll to bottom
  SendMessage(MemoReceive.Handle, $00B6, 0, MemoReceive.Lines.Count);
end;

procedure TForm1.UpdateStatus(const Status: string);
begin
  lblStatus.Caption := 'สถานะ: ' + Status;
end;

procedure TForm1.StartReceiving;
begin
  FReceiveThread := TReceiveThread.Create(IdTCPClient1, Self);
end;

procedure TForm1.StopReceiving;
begin
  if Assigned(FReceiveThread) then
  begin
    (FReceiveThread as TReceiveThread).Terminate;
    FReceiveThread := nil;
  end;
end;

procedure TForm1.btnConnectClick(Sender: TObject);
begin
  try
    IdTCPClient1.Host := EditHost.Text;
    IdTCPClient1.Port := StrToInt(EditPort.Text);
    IdTCPClient1.Connect;
    
    if IdTCPClient1.Connected then
    begin
      UpdateStatus('เชื่อมต่อแล้ว: ' + EditHost.Text + ':' + EditPort.Text);
      Log('เชื่อมต่อสำเร็จ');
      
      btnConnect.Enabled := False;
      btnDisconnect.Enabled := True;
      btnSend.Enabled := True;
      
      StartReceiving;
    end;
  except
    on E: Exception do
    begin
      UpdateStatus('เชื่อมต่อไม่สำเร็จ');
      Log('Error: ' + E.Message);
    end;
  end;
end;

procedure TForm1.btnDisconnectClick(Sender: TObject);
begin
  StopReceiving;
  
  if IdTCPClient1.Connected then
  begin
    IdTCPClient1.Disconnect;
    Log('ตัดการเชื่อมต่อแล้ว');
    UpdateStatus('ยังไม่ได้เชื่อมต่อ');
  end;
  
  btnConnect.Enabled := True;
  btnDisconnect.Enabled := False;
  btnSend.Enabled := False;
end;

procedure TForm1.btnSendClick(Sender: TObject);
var
  Msg: string;
begin
  if not IdTCPClient1.Connected then
  begin
    ShowMessage('ยังไม่ได้เชื่อมต่อ');
    Exit;
  end;
  
  Msg := MemoSend.Lines.Text;
  if Msg = '' then Exit;
  
  try
    // ส่งข้อมูลแบบ Line
    IdTCPClient1.IOHandler.WriteLn(Msg);
    Log('Sent: ' + Trim(Msg));
    MemoSend.Clear;
  except
    on E: Exception do
    begin
      Log('Error sending: ' + E.Message);
      btnDisconnectClick(nil);
    end;
  end;
end;

end.
```

---

## 39.4 TCP Server

```pascal
unit TCPServerDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  IdTCPServer, IdContext, IdGlobal, IdComponent;

type
  TClientInfo = record
    Context: TIdContext;
    Username: string;
    ConnectedAt: TDateTime;
  end;
  
  TForm1 = class(TForm)
    EditPort: TEdit;
    btnStart: TButton;
    btnStop: TButton;
    MemoLog: TMemo;
    ListClients: TListBox;
    lblStatus: TLabel;
    lblClientCount: TLabel;
    MemoSendAll: TMemo;
    btnSendAll: TButton;
    
    IdTCPServer1: TIdTCPServer;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnStartClick(Sender: TObject);
    procedure btnStopClick(Sender: TObject);
    procedure btnSendAllClick(Sender: TObject);
    procedure IdTCPServer1Connect(AContext: TIdContext);
    procedure IdTCPServer1Disconnect(AContext: TIdContext);
    procedure IdTCPServer1Execute(AContext: TIdContext);
  private
    FClientLock: TCriticalSection;
    FClientCount: Integer;
    
    procedure Log(const Msg: string);
    procedure UpdateClientList;
    procedure BroadcastMessage(const Msg: string; Sender: TIdContext = nil);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

uses
  SyncObjs;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FClientLock := TCriticalSection.Create;
  FClientCount := 0;
  
  EditPort.Text := '9999';
  
  // ตั้งค่า Server
  IdTCPServer1.MaxConnections := 50;
  IdTCPServer1.OnConnect := IdTCPServer1Connect;
  IdTCPServer1.OnDisconnect := IdTCPServer1Disconnect;
  IdTCPServer1.OnExecute := IdTCPServer1Execute;
  
  lblStatus.Caption := 'สถานะ: ปิด';
  lblClientCount.Caption := 'ผู้ใช้: 0';
  
  btnStop.Enabled := False;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  if IdTCPServer1.Active then
    IdTCPServer1.StopListening;
  FClientLock.Free;
end;

procedure TForm1.Log(const Msg: string);
begin
  // ต้องเรียกใน Main thread
  TThread.Synchronize(nil, procedure
  begin
    MemoLog.Lines.Add(FormatDateTime('[hh:nn:ss] ', Now) + Msg);
  end);
end;

procedure TForm1.UpdateClientList;
begin
  TThread.Synchronize(nil, procedure
  begin
    ListClients.Clear;
    var Contexts := IdTCPServer1.Contexts.LockList;
    try
      var i: Integer;
      for i := 0 to Contexts.Count - 1 do
      begin
        var Ctx := TIdContext(Contexts[i]);
        var IP := Ctx.Connection.Socket.Binding.PeerIP;
        var Port := IntToStr(Ctx.Connection.Socket.Binding.PeerPort);
        ListClients.Items.Add(IP + ':' + Port);
      end;
    finally
      IdTCPServer1.Contexts.UnlockList;
    end;
    lblClientCount.Caption := Format('ผู้ใช้: %d', [ListClients.Items.Count]);
  end);
end;

procedure TForm1.BroadcastMessage(const Msg: string; Sender: TIdContext);
var
  Contexts: TList;
  i: Integer;
  Ctx: TIdContext;
begin
  Contexts := IdTCPServer1.Contexts.LockList;
  try
    for i := 0 to Contexts.Count - 1 do
    begin
      Ctx := TIdContext(Contexts[i]);
      if (Sender = nil) or (Ctx <> Sender) then
      begin
        try
          Ctx.Connection.IOHandler.WriteLn(Msg);
        except
          // Client อาจ disconnect แล้ว - ข้ามไป
        end;
      end;
    end;
  finally
    IdTCPServer1.Contexts.UnlockList;
  end;
end;

procedure TForm1.btnStartClick(Sender: TObject);
begin
  try
    IdTCPServer1.DefaultPort := StrToInt(EditPort.Text);
    IdTCPServer1.Active := True;
    
    lblStatus.Caption := Format('สถานะ: รับฟัง Port %s', [EditPort.Text]);
    Log('Server เริ่มทำงานที่ Port ' + EditPort.Text);
    
    btnStart.Enabled := False;
    btnStop.Enabled := True;
  except
    on E: Exception do
    begin
      lblStatus.Caption := 'สถานะ: เกิดข้อผิดพลาด';
      Log('Error: ' + E.Message);
    end;
  end;
end;

procedure TForm1.btnStopClick(Sender: TObject);
begin
  // ส่งข้อความแจ้ง clients
  BroadcastMessage('SERVER: กำลังปิดเซิร์ฟเวอร์...');
  
  IdTCPServer1.Active := False;
  
  lblStatus.Caption := 'สถานะ: ปิด';
  Log('Server ปิดทำงาน');
  
  btnStart.Enabled := True;
  btnStop.Enabled := False;
  
  ListClients.Clear;
  lblClientCount.Caption := 'ผู้ใช้: 0';
end;

procedure TForm1.btnSendAllClick(Sender: TObject);
var
  Msg: string;
begin
  Msg := MemoSendAll.Text;
  if Msg = '' then Exit;
  
  BroadcastMessage('SERVER: ' + Trim(Msg));
  Log('Broadcast: ' + Trim(Msg));
  MemoSendAll.Clear;
end;

// เรียกเมื่อ Client เชื่อมต่อ (จาก Thread ของ Server)
procedure TForm1.IdTCPServer1Connect(AContext: TIdContext);
var
  IP: string;
  Port: string;
begin
  IP := AContext.Connection.Socket.Binding.PeerIP;
  Port := IntToStr(AContext.Connection.Socket.Binding.PeerPort);
  
  Log('Client เชื่อมต่อ: ' + IP + ':' + Port);
  
  // ส่งข้อความต้อนรับ
  AContext.Connection.IOHandler.WriteLn('WELCOME: ยินดีต้อนรับสู่เซิร์ฟเวอร์');
  
  // แจ้ง Clients อื่น
  BroadcastMessage('INFO: ' + IP + ' เข้าร่วมแล้ว', AContext);
  
  UpdateClientList;
end;

// เรียกเมื่อ Client ตัดการเชื่อมต่อ
procedure TForm1.IdTCPServer1Disconnect(AContext: TIdContext);
var
  IP: string;
begin
  IP := AContext.Connection.Socket.Binding.PeerIP;
  Log('Client ตัดการเชื่อมต่อ: ' + IP);
  
  BroadcastMessage('INFO: ' + IP + ' ออกจากระบบ', AContext);
  
  UpdateClientList;
end;

// เรียกเมื่อรับข้อมูลจาก Client (จาก Thread แยก)
procedure TForm1.IdTCPServer1Execute(AContext: TIdContext);
var
  Line: string;
  IP: string;
begin
  try
    // รับข้อมูล 1 บรรทัด
    Line := AContext.Connection.IOHandler.ReadLn('', 5000);  // 5s timeout
    
    if Line <> '' then
    begin
      IP := AContext.Connection.Socket.Binding.PeerIP;
      Log('From ' + IP + ': ' + Line);
      
      // ส่งต่อให้ clients อื่น (Chat broadcast)
      BroadcastMessage(IP + ': ' + Line, AContext);
      
      // ประมวลผลคำสั่ง
      if UpperCase(Line) = 'QUIT' then
        AContext.Connection.Disconnect
      else if UpperCase(Line) = 'PING' then
        AContext.Connection.IOHandler.WriteLn('PONG')
      else if UpperCase(Copy(Line, 1, 4)) = 'WHO?' then
      begin
        var Contexts := IdTCPServer1.Contexts.LockList;
        try
          var Response := 'USERS: ' + IntToStr(Contexts.Count) + ' คนออนไลน์';
          AContext.Connection.IOHandler.WriteLn(Response);
        finally
          IdTCPServer1.Contexts.UnlockList;
        end;
      end;
    end;
  except
    on E: EIdReadTimeout do
      ;  // Timeout ปกติ - รอต่อไป
    on E: Exception do
    begin
      Log('Execute error for ' + AContext.Connection.Socket.Binding.PeerIP + 
           ': ' + E.Message);
      AContext.Connection.Disconnect;
    end;
  end;
end;

end.
```

---

## 39.5 Multi-Client Server

```pascal
unit MultiClientServer;

interface

uses
  Classes, SysUtils,
  IdTCPServer, IdContext, IdGlobal;

type
  // เก็บข้อมูล Client แต่ละคน
  TClientData = class
  public
    Username: string;
    Room: string;
    LastActivity: TDateTime;
    constructor Create(const AUsername: string);
  end;

  TChatServer = class
  private
    FServer: TIdTCPServer;
    procedure OnConnect(AContext: TIdContext);
    procedure OnDisconnect(AContext: TIdContext);
    procedure OnExecute(AContext: TIdContext);
    procedure ProcessCommand(AContext: TIdContext; const Command, Params: string);
    procedure BroadcastToRoom(const Room, Msg: string; Sender: TIdContext = nil);
    function GetClientData(AContext: TIdContext): TClientData;
    procedure SetClientData(AContext: TIdContext; Data: TClientData);
  public
    constructor Create;
    destructor Destroy; override;
    procedure Start(Port: Integer);
    procedure Stop;
    function GetOnlineCount: Integer;
    function GetRoomList: TStringList;
  end;

implementation

constructor TClientData.Create(const AUsername: string);
begin
  inherited Create;
  Username := AUsername;
  Room := 'General';
  LastActivity := Now;
end;

constructor TChatServer.Create;
begin
  inherited;
  FServer := TIdTCPServer.Create(nil);
  FServer.OnConnect := OnConnect;
  FServer.OnDisconnect := OnDisconnect;
  FServer.OnExecute := OnExecute;
  FServer.MaxConnections := 100;
end;

destructor TChatServer.Destroy;
begin
  Stop;
  FServer.Free;
  inherited;
end;

procedure TChatServer.Start(Port: Integer);
begin
  FServer.DefaultPort := Port;
  FServer.Active := True;
end;

procedure TChatServer.Stop;
begin
  BroadcastToRoom('', 'SERVER: เซิร์ฟเวอร์กำลังปิด...');
  FServer.Active := False;
end;

function TChatServer.GetClientData(AContext: TIdContext): TClientData;
begin
  Result := TClientData(AContext.Data);
end;

procedure TChatServer.SetClientData(AContext: TIdContext; Data: TClientData);
begin
  AContext.Data := Data;
end;

procedure TChatServer.OnConnect(AContext: TIdContext);
var
  Data: TClientData;
  IP: string;
begin
  IP := AContext.Connection.Socket.Binding.PeerIP;
  
  // กำหนด Username ชั่วคราว
  Data := TClientData.Create('Guest_' + IP.Replace('.', '_'));
  SetClientData(AContext, Data);
  
  // ถามชื่อ
  AContext.Connection.IOHandler.WriteLn('ENTER_NAME: กรุณาใส่ชื่อผู้ใช้:');
end;

procedure TChatServer.OnDisconnect(AContext: TIdContext);
var
  Data: TClientData;
begin
  Data := GetClientData(AContext);
  if Assigned(Data) then
  begin
    BroadcastToRoom(Data.Room, 
      Format('*** %s ออกจากห้อง %s', [Data.Username, Data.Room]), AContext);
    Data.Free;
    AContext.Data := nil;
  end;
end;

procedure TChatServer.OnExecute(AContext: TIdContext);
var
  Line: string;
  Data: TClientData;
begin
  try
    Line := Trim(AContext.Connection.IOHandler.ReadLn('', 30000));
    
    if Line = '' then Exit;
    
    Data := GetClientData(AContext);
    if not Assigned(Data) then Exit;
    
    Data.LastActivity := Now;
    
    // ถ้ายังไม่มีชื่อ ให้ตั้งชื่อก่อน
    if Copy(Data.Username, 1, 6) = 'Guest_' then
    begin
      if Length(Line) >= 3 then
      begin
        Data.Username := Line;
        AContext.Connection.IOHandler.WriteLn('WELCOME: ยินดีต้อนรับ ' + Data.Username);
        BroadcastToRoom(Data.Room, 
          '*** ' + Data.Username + ' เข้าร่วมห้อง ' + Data.Room, AContext);
      end else
        AContext.Connection.IOHandler.WriteLn('ERROR: ชื่อต้องมีอย่างน้อย 3 ตัวอักษร');
      Exit;
    end;
    
    // ประมวลผลข้อความ
    if Copy(Line, 1, 1) = '/' then
    begin
      var SpacePos := Pos(' ', Line);
      var Command, Params: string;
      if SpacePos > 0 then
      begin
        Command := UpperCase(Copy(Line, 2, SpacePos - 2));
        Params := Copy(Line, SpacePos + 1, MaxInt);
      end else
      begin
        Command := UpperCase(Copy(Line, 2, MaxInt));
        Params := '';
      end;
      ProcessCommand(AContext, Command, Params);
    end else
    begin
      // ข้อความปกติ - ส่ง broadcast
      BroadcastToRoom(Data.Room, 
        Format('[%s] %s: %s', [FormatDateTime('hh:nn', Now), Data.Username, Line]));
    end;
    
  except
    on E: EIdReadTimeout do;
    on E: Exception do
      AContext.Connection.Disconnect;
  end;
end;

procedure TChatServer.ProcessCommand(AContext: TIdContext; const Command, Params: string);
var
  Data: TClientData;
  IOHandler: TIdIOHandler;
begin
  Data := GetClientData(AContext);
  IOHandler := AContext.Connection.IOHandler;
  
  if Command = 'JOIN' then
  begin
    // เข้าห้องใหม่
    var OldRoom := Data.Room;
    Data.Room := Params;
    IOHandler.WriteLn('JOIN_OK: เข้าห้อง ' + Params);
    BroadcastToRoom(OldRoom, '*** ' + Data.Username + ' ออกจากห้อง', AContext);
    BroadcastToRoom(Params, '*** ' + Data.Username + ' เข้าร่วมห้อง', AContext);
  end
  else if Command = 'WHO' then
  begin
    // แสดงรายชื่อในห้อง
    var Users := TStringList.Create;
    try
      var Contexts := FServer.Contexts.LockList;
      try
        var i: Integer;
        for i := 0 to Contexts.Count - 1 do
        begin
          var Ctx := TIdContext(Contexts[i]);
          var CD := GetClientData(Ctx);
          if Assigned(CD) and (CD.Room = Data.Room) then
            Users.Add(CD.Username);
        end;
      finally
        FServer.Contexts.UnlockList;
      end;
      IOHandler.WriteLn('USERS: ' + Users.CommaText);
    finally
      Users.Free;
    end;
  end
  else if Command = 'PM' then
  begin
    // Private Message
    var SpacePos := Pos(' ', Params);
    if SpacePos > 0 then
    begin
      var Target := Copy(Params, 1, SpacePos - 1);
      var Msg := Copy(Params, SpacePos + 1, MaxInt);
      
      var Contexts := FServer.Contexts.LockList;
      var Found := False;
      try
        var i: Integer;
        for i := 0 to Contexts.Count - 1 do
        begin
          var Ctx := TIdContext(Contexts[i]);
          var CD := GetClientData(Ctx);
          if Assigned(CD) and (CD.Username = Target) then
          begin
            Ctx.Connection.IOHandler.WriteLn(
              Format('PM from %s: %s', [Data.Username, Msg]));
            Found := True;
            Break;
          end;
        end;
      finally
        FServer.Contexts.UnlockList;
      end;
      
      if Found then
        IOHandler.WriteLn('PM_SENT: ส่งข้อความถึง ' + Target + ' แล้ว')
      else
        IOHandler.WriteLn('PM_ERROR: ไม่พบผู้ใช้ ' + Target);
    end;
  end
  else if Command = 'ROOMS' then
  begin
    var Rooms := GetRoomList;
    try
      IOHandler.WriteLn('ROOMS: ' + Rooms.CommaText);
    finally
      Rooms.Free;
    end;
  end
  else if Command = 'HELP' then
  begin
    IOHandler.WriteLn('HELP:');
    IOHandler.WriteLn('  /join <room> - เข้าห้อง');
    IOHandler.WriteLn('  /who - ดูรายชื่อในห้อง');
    IOHandler.WriteLn('  /pm <user> <msg> - ส่งข้อความส่วนตัว');
    IOHandler.WriteLn('  /rooms - ดูรายชื่อห้อง');
    IOHandler.WriteLn('  /quit - ออก');
  end
  else if Command = 'QUIT' then
    AContext.Connection.Disconnect
  else
    IOHandler.WriteLn('ERROR: คำสั่งไม่ถูกต้อง พิมพ์ /help สำหรับความช่วยเหลือ');
end;

procedure TChatServer.BroadcastToRoom(const Room, Msg: string; Sender: TIdContext);
var
  Contexts: TList;
  i: Integer;
begin
  Contexts := FServer.Contexts.LockList;
  try
    for i := 0 to Contexts.Count - 1 do
    begin
      var Ctx := TIdContext(Contexts[i]);
      if Ctx = Sender then Continue;
      
      var Data := GetClientData(Ctx);
      if not Assigned(Data) then Continue;
      
      if (Room = '') or (Data.Room = Room) then
      begin
        try
          Ctx.Connection.IOHandler.WriteLn(Msg);
        except
          // Client ออกไปแล้ว
        end;
      end;
    end;
  finally
    FServer.Contexts.UnlockList;
  end;
end;

function TChatServer.GetOnlineCount: Integer;
begin
  Result := FServer.Contexts.LockList.Count;
  FServer.Contexts.UnlockList;
end;

function TChatServer.GetRoomList: TStringList;
var
  Contexts: TList;
  i: Integer;
begin
  Result := TStringList.Create;
  Result.Duplicates := dupIgnore;
  Result.Sorted := True;
  
  Contexts := FServer.Contexts.LockList;
  try
    for i := 0 to Contexts.Count - 1 do
    begin
      var CD := GetClientData(TIdContext(Contexts[i]));
      if Assigned(CD) then
        Result.Add(CD.Room);
    end;
  finally
    FServer.Contexts.UnlockList;
  end;
end;

end.
```

---

## 39.6 UDP Client/Server

```pascal
unit UDPDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls,
  IdUDPClient, IdUDPServer, IdGlobal;

type
  TUDPExample = class
  private
    FClient: TIdUDPClient;
    FServer: TIdUDPServer;
    procedure OnServerRead(AThread: TIdUDPListenerThread;
                            const AData: TIdBytes;
                            ABinding: TIdSocketHandle);
  public
    constructor Create;
    destructor Destroy; override;
    procedure StartServer(Port: Integer);
    procedure StopServer;
    procedure SendMessage(const Host: string; Port: Integer; const Msg: string);
    procedure Broadcast(Port: Integer; const Msg: string);
    procedure MulticastSend(const GroupIP: string; Port: Integer; const Msg: string);
  end;

implementation

constructor TUDPExample.Create;
begin
  inherited;
  FClient := TIdUDPClient.Create(nil);
  FServer := TIdUDPServer.Create(nil);
  FServer.OnUDPRead := OnServerRead;
end;

destructor TUDPExample.Destroy;
begin
  FServer.Free;
  FClient.Free;
  inherited;
end;

procedure TUDPExample.StartServer(Port: Integer);
begin
  FServer.DefaultPort := Port;
  FServer.Active := True;
end;

procedure TUDPExample.StopServer;
begin
  FServer.Active := False;
end;

procedure TUDPExample.OnServerRead(AThread: TIdUDPListenerThread;
  const AData: TIdBytes; ABinding: TIdSocketHandle);
var
  Msg: string;
begin
  Msg := BytesToString(AData, IndyTextEncoding_UTF8);
  TThread.Synchronize(nil, procedure
  begin
    WriteLn('UDP Received from ' + ABinding.PeerIP + ':' + 
             IntToStr(ABinding.PeerPort) + ' - ' + Msg);
  end);
  
  // ส่ง ACK กลับ
  FServer.SendBuffer(ABinding.PeerIP, ABinding.PeerPort, 
                      ToBytes('ACK: ' + Msg));
end;

procedure TUDPExample.SendMessage(const Host: string; Port: Integer; const Msg: string);
begin
  FClient.Host := Host;
  FClient.Port := Port;
  FClient.Send(Msg);
end;

procedure TUDPExample.Broadcast(Port: Integer; const Msg: string);
begin
  // Broadcast ไปทุก Host ใน Network
  FClient.Host := '255.255.255.255';
  FClient.Port := Port;
  FClient.BroadcastEnabled := True;
  FClient.Send(Msg);
  FClient.BroadcastEnabled := False;
end;

procedure TUDPExample.MulticastSend(const GroupIP: string; Port: Integer; const Msg: string);
begin
  // Multicast ไปยัง Group (224.0.0.0 - 239.255.255.255)
  FClient.Host := GroupIP;
  FClient.Port := Port;
  FClient.Send(Msg);
end;

end.
```

---

## 39.7 Port Scanner

```pascal
unit PortScanner;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ComCtrls,
  IdTCPClient, IdGlobal;

type
  TScanResult = record
    Port: Integer;
    IsOpen: Boolean;
    ServiceName: string;
    ResponseTime: Integer;
  end;
  
  TPortScannerThread = class(TThread)
  private
    FHost: string;
    FStartPort: Integer;
    FEndPort: Integer;
    FTimeout: Integer;
    FResults: array of TScanResult;
    FOnProgress: procedure(Port: Integer; IsOpen: Boolean) of object;
    FOnComplete: procedure(const Results: array of TScanResult) of object;
    procedure NotifyProgress(Port: Integer; IsOpen: Boolean);
  protected
    procedure Execute; override;
  public
    constructor Create(const AHost: string; StartPort, EndPort, Timeout: Integer);
    property OnProgress: procedure(Port: Integer; IsOpen: Boolean) of object 
             read FOnProgress write FOnProgress;
    property OnComplete: procedure(const Results: array of TScanResult) of object 
             read FOnComplete write FOnComplete;
  end;
  
  TForm1 = class(TForm)
    EditHost: TEdit;
    EditStartPort: TEdit;
    EditEndPort: TEdit;
    EditTimeout: TEdit;
    btnScan: TButton;
    btnStop: TButton;
    ProgressBar1: TProgressBar;
    ListViewResults: TListView;
    lblStatus: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnScanClick(Sender: TObject);
    procedure btnStopClick(Sender: TObject);
  private
    FScanThread: TPortScannerThread;
    FOpenPorts: Integer;
    procedure OnScanProgress(Port: Integer; IsOpen: Boolean);
    procedure OnScanComplete(const Results: array of TScanResult);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

// ชื่อ Service ทั่วไป
function GetServiceName(Port: Integer): string;
begin
  case Port of
    21: Result := 'FTP';
    22: Result := 'SSH';
    23: Result := 'Telnet';
    25: Result := 'SMTP';
    53: Result := 'DNS';
    80: Result := 'HTTP';
    110: Result := 'POP3';
    143: Result := 'IMAP';
    443: Result := 'HTTPS';
    445: Result := 'SMB';
    3306: Result := 'MySQL';
    3389: Result := 'RDP';
    5432: Result := 'PostgreSQL';
    6379: Result := 'Redis';
    8080: Result := 'HTTP-Alt';
    27017: Result := 'MongoDB';
    else Result := '';
  end;
end;

constructor TPortScannerThread.Create(const AHost: string; 
  StartPort, EndPort, Timeout: Integer);
begin
  inherited Create(True);
  FHost := AHost;
  FStartPort := StartPort;
  FEndPort := EndPort;
  FTimeout := Timeout;
  FreeOnTerminate := False;
  SetLength(FResults, 0);
end;

procedure TPortScannerThread.NotifyProgress(Port: Integer; IsOpen: Boolean);
begin
  if Assigned(FOnProgress) then
    TThread.Synchronize(Self, procedure
    begin
      FOnProgress(Port, IsOpen);
    end);
end;

procedure TPortScannerThread.Execute;
var
  Client: TIdTCPClient;
  Port: Integer;
  IsOpen: Boolean;
  StartTime: Cardinal;
begin
  Client := TIdTCPClient.Create(nil);
  try
    Client.ConnectTimeout := FTimeout;
    
    for Port := FStartPort to FEndPort do
    begin
      if Terminated then Break;
      
      IsOpen := False;
      StartTime := GetTickCount;
      
      try
        Client.Host := FHost;
        Client.Port := Port;
        Client.Connect;
        IsOpen := True;
        Client.Disconnect;
      except
        // Port ปิดหรือ timeout
      end;
      
      var ResponseTime := GetTickCount - StartTime;
      
      if IsOpen then
      begin
        var Result: TScanResult;
        Result.Port := Port;
        Result.IsOpen := True;
        Result.ServiceName := GetServiceName(Port);
        Result.ResponseTime := ResponseTime;
        
        SetLength(FResults, Length(FResults) + 1);
        FResults[High(FResults)] := Result;
      end;
      
      NotifyProgress(Port, IsOpen);
    end;
  finally
    Client.Free;
  end;
  
  // แจ้ง Complete
  if Assigned(FOnComplete) then
    TThread.Synchronize(Self, procedure
    begin
      FOnComplete(FResults);
    end);
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FScanThread := nil;
  FOpenPorts := 0;
  
  EditHost.Text := '127.0.0.1';
  EditStartPort.Text := '1';
  EditEndPort.Text := '1024';
  EditTimeout.Text := '500';
  
  // ตั้งค่า ListView
  with ListViewResults.Columns do
  begin
    with Add do begin Caption := 'Port'; Width := 80; end;
    with Add do begin Caption := 'Service'; Width := 100; end;
    with Add do begin Caption := 'Response (ms)'; Width := 120; end;
  end;
  ListViewResults.ViewStyle := vsReport;
  
  btnStop.Enabled := False;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  if Assigned(FScanThread) then
  begin
    FScanThread.Terminate;
    FScanThread.WaitFor;
    FScanThread.Free;
  end;
end;

procedure TForm1.btnScanClick(Sender: TObject);
var
  Host: string;
  StartPort, EndPort, Timeout: Integer;
begin
  Host := Trim(EditHost.Text);
  if Host = '' then begin ShowMessage('กรุณาใส่ Host'); Exit; end;
  
  try
    StartPort := StrToInt(EditStartPort.Text);
    EndPort := StrToInt(EditEndPort.Text);
    Timeout := StrToInt(EditTimeout.Text);
  except
    ShowMessage('กรุณาใส่ตัวเลขที่ถูกต้อง');
    Exit;
  end;
  
  if StartPort > EndPort then begin ShowMessage('Start > End'); Exit; end;
  
  ListViewResults.Clear;
  FOpenPorts := 0;
  ProgressBar1.Min := StartPort;
  ProgressBar1.Max := EndPort;
  ProgressBar1.Position := StartPort;
  
  lblStatus.Caption := Format('กำลังสแกน %s (%d-%d)...', [Host, StartPort, EndPort]);
  
  btnScan.Enabled := False;
  btnStop.Enabled := True;
  
  FScanThread := TPortScannerThread.Create(Host, StartPort, EndPort, Timeout);
  FScanThread.OnProgress := OnScanProgress;
  FScanThread.OnComplete := OnScanComplete;
  FScanThread.Start;
end;

procedure TForm1.btnStopClick(Sender: TObject);
begin
  if Assigned(FScanThread) then
  begin
    FScanThread.Terminate;
    lblStatus.Caption := 'หยุดสแกน';
  end;
end;

procedure TForm1.OnScanProgress(Port: Integer; IsOpen: Boolean);
begin
  ProgressBar1.Position := Port;
  
  if IsOpen then
  begin
    Inc(FOpenPorts);
    var Item := ListViewResults.Items.Add;
    Item.Caption := IntToStr(Port);
    Item.SubItems.Add(GetServiceName(Port));
    Item.SubItems.Add('---');
    Item.ImageIndex := 0;
  end;
  
  lblStatus.Caption := Format('กำลังสแกน port %d... พบ %d ports', [Port, FOpenPorts]);
end;

procedure TForm1.OnScanComplete(const Results: array of TScanResult);
begin
  // อัปเดต Response time ใน ListView
  ListViewResults.Clear;
  var i: Integer;
  for i := 0 to High(Results) do
  begin
    var Item := ListViewResults.Items.Add;
    Item.Caption := IntToStr(Results[i].Port);
    Item.SubItems.Add(Results[i].ServiceName);
    Item.SubItems.Add(IntToStr(Results[i].ResponseTime) + ' ms');
  end;
  
  lblStatus.Caption := Format('สแกนเสร็จ: พบ %d ports เปิด', [Length(Results)]);
  
  btnScan.Enabled := True;
  btnStop.Enabled := False;
  
  FScanThread.Free;
  FScanThread := nil;
end;

end.
```

---

## 39.8 Ping Implementation

```pascal
unit PingDemo;

interface

uses
  Classes, SysUtils, Windows, WinSock;

type
  TICMPHeader = packed record
    icmp_type: Byte;
    icmp_code: Byte;
    icmp_cksum: Word;
    icmp_id: Word;
    icmp_seq: Word;
    icmp_data: array[0..31] of Byte;
  end;

  TIPHeader = packed record
    ip_verlen: Byte;
    ip_tos: Byte;
    ip_len: Word;
    ip_id: Word;
    ip_off: Word;
    ip_ttl: Byte;
    ip_proto: Byte;
    ip_cksum: Word;
    ip_src: Cardinal;
    ip_dst: Cardinal;
  end;

  TPingResult = record
    Host: string;
    IP: string;
    Success: Boolean;
    RoundTripTime: Integer;
    TTL: Integer;
    ErrorMsg: string;
  end;

// ส่ง Ping ไปยัง Host
function PingHost(const Host: string; Timeout: Integer = 3000): TPingResult;

// Ping แบบ Multiple (ส่งหลายครั้ง)
function PingMultiple(const Host: string; Count: Integer; Timeout: Integer = 3000): TArray<TPingResult>;

implementation

uses
  IdIcmpClient;

// ใช้ Indy TIdIcmpClient (ง่ายกว่า Raw Socket)
function PingHost(const Host: string; Timeout: Integer): TPingResult;
var
  ICMP: TIdIcmpClient;
begin
  Result.Host := Host;
  Result.Success := False;
  Result.RoundTripTime := 0;
  Result.TTL := 0;
  
  ICMP := TIdIcmpClient.Create(nil);
  try
    ICMP.Host := Host;
    ICMP.ReceiveTimeout := Timeout;
    ICMP.PacketSize := 32;
    
    try
      ICMP.Ping;
      
      if ICMP.ReplyStatus.BytesReceived > 0 then
      begin
        Result.Success := True;
        Result.IP := ICMP.ReplyStatus.FromIpAddress;
        Result.RoundTripTime := ICMP.ReplyStatus.MsRoundTripTime;
        Result.TTL := ICMP.ReplyStatus.TimeToLive;
      end else
        Result.ErrorMsg := 'ไม่ได้รับการตอบกลับ';
        
    except
      on E: Exception do
        Result.ErrorMsg := E.Message;
    end;
  finally
    ICMP.Free;
  end;
end;

function PingMultiple(const Host: string; Count: Integer; Timeout: Integer): TArray<TPingResult>;
var
  i: Integer;
begin
  SetLength(Result, Count);
  for i := 0 to Count - 1 do
  begin
    Result[i] := PingHost(Host, Timeout);
    if i < Count - 1 then
      Sleep(1000);  // รอ 1 วินาทีระหว่าง Ping
  end;
end;

// ตัวอย่าง UI สำหรับ Ping
unit PingUI;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ComCtrls, ExtCtrls;

type
  TForm1 = class(TForm)
    EditHost: TEdit;
    EditCount: TEdit;
    btnPing: TButton;
    MemoResult: TMemo;
    ProgressBar1: TProgressBar;
    lblStatus: TLabel;
    
    procedure btnPingClick(Sender: TObject);
  private
    procedure RunPing;
  end;

implementation

procedure TForm1.btnPingClick(Sender: TObject);
begin
  btnPing.Enabled := False;
  MemoResult.Clear;
  
  TThread.CreateAnonymousThread(procedure
  begin
    RunPing;
    TThread.Synchronize(nil, procedure
    begin
      btnPing.Enabled := True;
      lblStatus.Caption := 'เสร็จแล้ว';
    end);
  end).Start;
end;

procedure TForm1.RunPing;
var
  Host: string;
  Count: Integer;
  Results: TArray<TPingResult>;
  i: Integer;
  Sent, Received: Integer;
  MinTime, MaxTime, TotalTime: Integer;
begin
  Host := Trim(EditHost.Text);
  Count := StrToIntDef(EditCount.Text, 4);
  if Count < 1 then Count := 1;
  if Count > 100 then Count := 100;
  
  TThread.Synchronize(nil, procedure
  begin
    MemoResult.Lines.Add('Pinging ' + Host + '...');
    ProgressBar1.Min := 0;
    ProgressBar1.Max := Count;
    ProgressBar1.Position := 0;
  end);
  
  SetLength(Results, Count);
  Sent := 0;
  Received := 0;
  MinTime := MaxInt;
  MaxTime := 0;
  TotalTime := 0;
  
  for i := 0 to Count - 1 do
  begin
    Results[i] := PingHost(Host);
    Inc(Sent);
    
    if Results[i].Success then
    begin
      Inc(Received);
      TotalTime := TotalTime + Results[i].RoundTripTime;
      if Results[i].RoundTripTime < MinTime then MinTime := Results[i].RoundTripTime;
      if Results[i].RoundTripTime > MaxTime then MaxTime := Results[i].RoundTripTime;
      
      var Msg := Format('Reply from %s: bytes=32 time=%dms TTL=%d',
        [Results[i].IP, Results[i].RoundTripTime, Results[i].TTL]);
      TThread.Synchronize(nil, procedure
      begin
        MemoResult.Lines.Add(Msg);
        ProgressBar1.Position := i + 1;
      end);
    end else
    begin
      var Msg := 'Request timeout for icmp_seq ' + IntToStr(i + 1);
      if Results[i].ErrorMsg <> '' then
        Msg := 'Error: ' + Results[i].ErrorMsg;
      TThread.Synchronize(nil, procedure
      begin
        MemoResult.Lines.Add(Msg);
        ProgressBar1.Position := i + 1;
      end);
    end;
    
    if i < Count - 1 then Sleep(1000);
  end;
  
  // Statistics
  var LossPercent := Round((Sent - Received) * 100 / Sent);
  var Stats := Format(
    #13#10 +
    'Ping statistics for %s:'#13#10 +
    '    Packets: Sent = %d, Received = %d, Lost = %d (%d%% loss)'#13#10 +
    'Approximate round trip times in milli-seconds:'#13#10 +
    '    Minimum = %dms, Maximum = %dms, Average = %dms',
    [Host, Sent, Received, Sent - Received, LossPercent,
     MinTime, MaxTime, TotalTime div Max(1, Received)]
  );
  
  TThread.Synchronize(nil, procedure
  begin
    MemoResult.Lines.Add(Stats);
  end);
end;

end.
```

---

## 39.9 Network Monitoring

```pascal
unit NetworkMonitor;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls, Graphics,
  Windows, IPHlpApi, IpTypes, WinSock;

type
  TNetworkStats = record
    BytesSent: Int64;
    BytesReceived: Int64;
    PacketsSent: Int64;
    PacketsReceived: Int64;
    SendSpeed: Double;    // bytes per second
    ReceiveSpeed: Double;
    InterfaceName: string;
  end;
  
  TForm1 = class(TForm)
    Timer1: TTimer;
    MemoStats: TMemo;
    PaintBox1: TPaintBox;
    lblSpeed: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure Timer1Timer(Sender: TObject);
    procedure PaintBox1Paint(Sender: TObject);
  private
    FLastStats: TNetworkStats;
    FSpeedHistory: array[0..99] of Double;
    FHistoryIndex: Integer;
    procedure GetNetworkStats(out Stats: TNetworkStats);
    procedure UpdateGraph;
  end;

implementation

{$R *.lfm}

procedure TForm1.GetNetworkStats(out Stats: TNetworkStats);
var
  Table: PMibIfTable;
  TableSize: DWORD;
  i: Integer;
begin
  FillChar(Stats, SizeOf(Stats), 0);
  
  TableSize := 0;
  GetIfTable(nil, TableSize, False);
  
  GetMem(Table, TableSize);
  try
    if GetIfTable(Table, TableSize, False) = NO_ERROR then
    begin
      for i := 0 to Table^.dwNumEntries - 1 do
      begin
        with Table^.table[i] do
        begin
          Stats.BytesSent := Stats.BytesSent + dwOutOctets;
          Stats.BytesReceived := Stats.BytesReceived + dwInOctets;
          Stats.PacketsSent := Stats.PacketsSent + dwOutUcastPkts;
          Stats.PacketsReceived := Stats.PacketsReceived + dwInUcastPkts;
        end;
      end;
    end;
  finally
    FreeMem(Table);
  end;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FHistoryIndex := 0;
  FillChar(FSpeedHistory, SizeOf(FSpeedHistory), 0);
  GetNetworkStats(FLastStats);
  Timer1.Interval := 1000;
  Timer1.Enabled := True;
end;

procedure TForm1.Timer1Timer(Sender: TObject);
var
  CurrentStats: TNetworkStats;
begin
  GetNetworkStats(CurrentStats);
  
  // คำนวณความเร็ว
  CurrentStats.SendSpeed := (CurrentStats.BytesSent - FLastStats.BytesSent);
  CurrentStats.ReceiveSpeed := (CurrentStats.BytesReceived - FLastStats.BytesReceived);
  
  // บันทึก History
  FSpeedHistory[FHistoryIndex mod 100] := CurrentStats.ReceiveSpeed;
  Inc(FHistoryIndex);
  
  // แสดงข้อมูล
  MemoStats.Clear;
  MemoStats.Lines.Add('=== Network Statistics ===');
  MemoStats.Lines.Add(Format('Download: %.2f KB/s', [CurrentStats.ReceiveSpeed / 1024]));
  MemoStats.Lines.Add(Format('Upload:   %.2f KB/s', [CurrentStats.SendSpeed / 1024]));
  MemoStats.Lines.Add('');
  MemoStats.Lines.Add(Format('Total Downloaded: %.2f MB', 
    [CurrentStats.BytesReceived / 1024 / 1024]));
  MemoStats.Lines.Add(Format('Total Uploaded:   %.2f MB', 
    [CurrentStats.BytesSent / 1024 / 1024]));
  
  lblSpeed.Caption := Format('↓ %.1f KB/s  ↑ %.1f KB/s',
    [CurrentStats.ReceiveSpeed / 1024, CurrentStats.SendSpeed / 1024]);
  
  FLastStats := CurrentStats;
  
  UpdateGraph;
end;

procedure TForm1.UpdateGraph;
begin
  PaintBox1.Invalidate;
end;

procedure TForm1.PaintBox1Paint(Sender: TObject);
var
  i, x, y: Integer;
  MaxSpeed: Double;
  H, W: Integer;
begin
  with PaintBox1.Canvas do
  begin
    Brush.Color := clBlack;
    FillRect(Rect(0, 0, PaintBox1.Width, PaintBox1.Height));
    
    H := PaintBox1.Height;
    W := PaintBox1.Width;
    
    // หา MaxSpeed
    MaxSpeed := 1024;  // minimum 1 KB/s
    for i := 0 to 99 do
      if FSpeedHistory[i] > MaxSpeed then
        MaxSpeed := FSpeedHistory[i];
    
    // วาด Grid
    Pen.Color := TColor(RGB(30, 30, 30));
    Pen.Width := 1;
    for i := 1 to 4 do
    begin
      y := H * i div 5;
      MoveTo(0, y);
      LineTo(W, y);
    end;
    
    // วาด Speed Graph
    Pen.Color := clGreen;
    Pen.Width := 2;
    
    var StartIdx := FHistoryIndex mod 100;
    for i := 0 to 98 do
    begin
      var Idx1 := (StartIdx + i) mod 100;
      var Idx2 := (StartIdx + i + 1) mod 100;
      
      x := W * i div 99;
      y := H - Round(H * FSpeedHistory[Idx1] / MaxSpeed);
      
      if i = 0 then MoveTo(x, y)
      else LineTo(x, y);
    end;
    
    // Label
    Font.Color := clWhite;
    Font.Size := 8;
    Brush.Style := bsClear;
    TextOut(5, 5, Format('Max: %.1f KB/s', [MaxSpeed / 1024]));
  end;
end;

end.
```

---

## 39.10 ตัวอย่างสมบูรณ์: Chat Application

```pascal
// ดูได้จาก TCPClientDemo และ TCPServerDemo ด้านบน
// ตัวอย่างนี้เป็น Chat Application เต็มรูปแบบ

unit ChatApp;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  IdTCPClient, IdTCPServer, IdContext, IdGlobal;

type
  TChatMode = (cmClient, cmServer);
  
  TForm1 = class(TForm)
    PanelConnect: TPanel;
    RadioClient: TRadioButton;
    RadioServer: TRadioButton;
    EditHost: TEdit;
    EditPort: TEdit;
    EditUsername: TEdit;
    btnConnect: TButton;
    btnDisconnect: TButton;
    
    PanelChat: TPanel;
    MemoChat: TMemo;
    EditMessage: TEdit;
    btnSend: TButton;
    
    ListBoxUsers: TListBox;
    lblUserCount: TLabel;
    
    IdTCPClient1: TIdTCPClient;
    IdTCPServer1: TIdTCPServer;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnConnectClick(Sender: TObject);
    procedure btnDisconnectClick(Sender: TObject);
    procedure btnSendClick(Sender: TObject);
    procedure EditMessageKeyPress(Sender: TObject; var Key: Char);
    procedure IdTCPServer1Connect(AContext: TIdContext);
    procedure IdTCPServer1Disconnect(AContext: TIdContext);
    procedure IdTCPServer1Execute(AContext: TIdContext);
  private
    FMode: TChatMode;
    FUsername: string;
    FReceiveThread: TThread;
    
    procedure ConnectAsClient;
    procedure StartServer;
    procedure Disconnect;
    procedure SendMessage(const Msg: string);
    procedure BroadcastToAll(const Msg: string; Sender: TIdContext = nil);
    procedure AddChatLine(const Msg: string);
    procedure UpdateUserList;
    procedure StartReceiveThread;
    procedure StopReceiveThread;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

type
  TClientReceiveThread = class(TThread)
  private
    FClient: TIdTCPClient;
    FOwner: TForm1;
  protected
    procedure Execute; override;
  public
    constructor Create(AClient: TIdTCPClient; AOwner: TForm1);
  end;

constructor TClientReceiveThread.Create(AClient: TIdTCPClient; AOwner: TForm1);
begin
  inherited Create(False);
  FClient := AClient;
  FOwner := AOwner;
  FreeOnTerminate := True;
end;

procedure TClientReceiveThread.Execute;
var
  Line: string;
begin
  while not Terminated do
  begin
    try
      if not FClient.Connected then Break;
      Line := FClient.IOHandler.ReadLn('', 2000);
      if Line <> '' then
      begin
        var Msg := Line;
        TThread.Synchronize(Self, procedure
        begin
          FOwner.AddChatLine(Msg);
        end);
      end;
    except
      on E: EIdReadTimeout do Continue;
      on E: Exception do
      begin
        if not Terminated then
        begin
          var Msg := 'ตัดการเชื่อมต่อ: ' + E.Message;
          TThread.Synchronize(Self, procedure
          begin
            FOwner.AddChatLine(Msg);
          end);
        end;
        Break;
      end;
    end;
  end;
end;

procedure TForm1.FormCreate(Sender: TObject);
begin
  FMode := cmClient;
  FReceiveThread := nil;
  
  EditHost.Text := '127.0.0.1';
  EditPort.Text := '9090';
  EditUsername.Text := 'User' + IntToStr(Random(1000));
  
  IdTCPClient1.ConnectTimeout := 5000;
  IdTCPServer1.MaxConnections := 50;
  IdTCPServer1.OnConnect := IdTCPServer1Connect;
  IdTCPServer1.OnDisconnect := IdTCPServer1Disconnect;
  IdTCPServer1.OnExecute := IdTCPServer1Execute;
  
  PanelChat.Enabled := False;
  btnDisconnect.Enabled := False;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  Disconnect;
end;

procedure TForm1.btnConnectClick(Sender: TObject);
begin
  FUsername := Trim(EditUsername.Text);
  if FUsername = '' then
  begin
    ShowMessage('กรุณาใส่ชื่อผู้ใช้');
    Exit;
  end;
  
  FMode := TChatMode(RadioClient.Checked.ToInteger xor 1);
  
  if RadioClient.Checked then
    ConnectAsClient
  else
    StartServer;
end;

procedure TForm1.ConnectAsClient;
begin
  try
    IdTCPClient1.Host := EditHost.Text;
    IdTCPClient1.Port := StrToInt(EditPort.Text);
    IdTCPClient1.Connect;
    
    // ส่งชื่อ
    IdTCPClient1.IOHandler.WriteLn('NICK:' + FUsername);
    
    AddChatLine('* เชื่อมต่อสำเร็จ');
    
    PanelChat.Enabled := True;
    btnConnect.Enabled := False;
    btnDisconnect.Enabled := True;
    
    StartReceiveThread;
  except
    on E: Exception do
      ShowMessage('เชื่อมต่อไม่สำเร็จ: ' + E.Message);
  end;
end;

procedure TForm1.StartServer;
begin
  try
    IdTCPServer1.DefaultPort := StrToInt(EditPort.Text);
    IdTCPServer1.Active := True;
    
    AddChatLine(Format('* Server เริ่มทำงานที่ Port %s', [EditPort.Text]));
    AddChatLine('* รอการเชื่อมต่อ...');
    
    PanelChat.Enabled := True;
    btnConnect.Enabled := False;
    btnDisconnect.Enabled := True;
  except
    on E: Exception do
      ShowMessage('เริ่ม Server ไม่สำเร็จ: ' + E.Message);
  end;
end;

procedure TForm1.btnDisconnectClick(Sender: TObject);
begin
  Disconnect;
end;

procedure TForm1.Disconnect;
begin
  StopReceiveThread;
  
  if IdTCPClient1.Connected then
  begin
    IdTCPClient1.IOHandler.WriteLn('QUIT');
    IdTCPClient1.Disconnect;
  end;
  
  if IdTCPServer1.Active then
  begin
    BroadcastToAll('SERVER: กำลังปิดเซิร์ฟเวอร์');
    IdTCPServer1.Active := False;
  end;
  
  AddChatLine('* ตัดการเชื่อมต่อ');
  
  PanelChat.Enabled := False;
  btnConnect.Enabled := True;
  btnDisconnect.Enabled := False;
  ListBoxUsers.Clear;
  lblUserCount.Caption := 'ผู้ใช้: 0';
end;

procedure TForm1.SendMessage(const Msg: string);
begin
  if FMode = cmClient then
  begin
    if IdTCPClient1.Connected then
    begin
      IdTCPClient1.IOHandler.WriteLn('MSG:' + Msg);
      AddChatLine('[' + FUsername + ']: ' + Msg);
    end;
  end else
  begin
    // Server ส่งข้อความ
    BroadcastToAll('[Server-' + FUsername + ']: ' + Msg);
    AddChatLine('[Server-' + FUsername + ']: ' + Msg);
  end;
end;

procedure TForm1.BroadcastToAll(const Msg: string; Sender: TIdContext);
begin
  var Contexts := IdTCPServer1.Contexts.LockList;
  try
    var i: Integer;
    for i := 0 to Contexts.Count - 1 do
    begin
      var Ctx := TIdContext(Contexts[i]);
      if Ctx = Sender then Continue;
      try
        Ctx.Connection.IOHandler.WriteLn(Msg);
      except
      end;
    end;
  finally
    IdTCPServer1.Contexts.UnlockList;
  end;
end;

procedure TForm1.AddChatLine(const Msg: string);
begin
  TThread.Synchronize(nil, procedure
  begin
    MemoChat.Lines.Add(FormatDateTime('[hh:nn:ss] ', Now) + Msg);
    SendMessage(MemoChat.Handle, $00B6, 0, MemoChat.Lines.Count);
  end);
end;

procedure TForm1.UpdateUserList;
begin
  TThread.Synchronize(nil, procedure
  begin
    ListBoxUsers.Clear;
    var Contexts := IdTCPServer1.Contexts.LockList;
    try
      var i: Integer;
      for i := 0 to Contexts.Count - 1 do
      begin
        var IP := TIdContext(Contexts[i]).Connection.Socket.Binding.PeerIP;
        var Username := string(TIdContext(Contexts[i]).Data);
        if Username <> '' then
          ListBoxUsers.Items.Add(Username)
        else
          ListBoxUsers.Items.Add(IP);
      end;
    finally
      IdTCPServer1.Contexts.UnlockList;
    end;
    lblUserCount.Caption := Format('ผู้ใช้: %d', [ListBoxUsers.Items.Count]);
  end);
end;

procedure TForm1.btnSendClick(Sender: TObject);
begin
  var Msg := Trim(EditMessage.Text);
  if Msg = '' then Exit;
  
  SendMessage(Msg);
  EditMessage.Clear;
  EditMessage.SetFocus;
end;

procedure TForm1.EditMessageKeyPress(Sender: TObject; var Key: Char);
begin
  if Key = #13 then
  begin
    Key := #0;
    btnSendClick(nil);
  end;
end;

procedure TForm1.StartReceiveThread;
begin
  FReceiveThread := TClientReceiveThread.Create(IdTCPClient1, Self);
end;

procedure TForm1.StopReceiveThread;
begin
  if Assigned(FReceiveThread) then
  begin
    (FReceiveThread as TClientReceiveThread).Terminate;
    FReceiveThread := nil;
  end;
end;

procedure TForm1.IdTCPServer1Connect(AContext: TIdContext);
var
  IP: string;
begin
  IP := AContext.Connection.Socket.Binding.PeerIP;
  AContext.Data := TObject(Pointer(IP));  // เก็บ IP ชั่วคราว
  
  AddChatLine('* ' + IP + ' เชื่อมต่อแล้ว');
  UpdateUserList;
end;

procedure TForm1.IdTCPServer1Disconnect(AContext: TIdContext);
var
  Username: string;
begin
  Username := string(AContext.Data);
  if Username = '' then
    Username := AContext.Connection.Socket.Binding.PeerIP;
  
  AddChatLine('* ' + Username + ' ออกจากระบบ');
  BroadcastToAll('* ' + Username + ' ออกจากระบบ', AContext);
  UpdateUserList;
end;

procedure TForm1.IdTCPServer1Execute(AContext: TIdContext);
var
  Line, Command, Data: string;
  ColonPos: Integer;
begin
  try
    Line := AContext.Connection.IOHandler.ReadLn('', 5000);
    if Line = '' then Exit;
    
    ColonPos := Pos(':', Line);
    if ColonPos > 0 then
    begin
      Command := Copy(Line, 1, ColonPos - 1);
      Data := Copy(Line, ColonPos + 1, MaxInt);
    end else
    begin
      Command := Line;
      Data := '';
    end;
    
    if Command = 'NICK' then
    begin
      AContext.Data := TObject(Pointer(Data));
      AddChatLine('* ' + Data + ' ตั้งชื่อแล้ว');
      BroadcastToAll('* ' + Data + ' เข้าร่วมแล้ว', AContext);
      AContext.Connection.IOHandler.WriteLn('WELCOME: ยินดีต้อนรับ ' + Data);
      UpdateUserList;
    end
    else if Command = 'MSG' then
    begin
      var Username := string(AContext.Data);
      if Username = '' then
        Username := AContext.Connection.Socket.Binding.PeerIP;
      
      var FullMsg := '[' + Username + ']: ' + Data;
      BroadcastToAll(FullMsg, AContext);
      AddChatLine(FullMsg);
    end
    else if Command = 'QUIT' then
      AContext.Connection.Disconnect;
      
  except
    on E: EIdReadTimeout do;
    on E: Exception do
      AContext.Connection.Disconnect;
  end;
end;

end.
```

---

## 39.11 File Transfer

```pascal
unit FileTransfer;

interface

uses
  Classes, SysUtils,
  IdTCPClient, IdTCPServer, IdContext, IdGlobal;

// Transfer Protocol:
// Client -> Server:
//   SEND <filename> <filesize>
//   [binary data]
//
// Server -> Client:
//   READY
//   OK or ERROR <reason>

type
  TTransferProgress = procedure(BytesTransferred, TotalBytes: Int64) of object;
  
  TFileTransferClient = class
  private
    FClient: TIdTCPClient;
    FOnProgress: TTransferProgress;
  public
    constructor Create;
    destructor Destroy; override;
    function Connect(const Host: string; Port: Integer): Boolean;
    procedure Disconnect;
    function SendFile(const FileName: string): Boolean;
    function ReceiveFile(const SaveDir: string): Boolean;
    property OnProgress: TTransferProgress read FOnProgress write FOnProgress;
  end;

  TFileTransferServer = class
  private
    FServer: TIdTCPServer;
    FSaveDir: string;
    FOnProgress: TTransferProgress;
    procedure OnServerExecute(AContext: TIdContext);
  public
    constructor Create;
    destructor Destroy; override;
    procedure Start(Port: Integer; const SaveDirectory: string);
    procedure Stop;
    property OnProgress: TTransferProgress read FOnProgress write FOnProgress;
  end;

implementation

const
  BUFFER_SIZE = 65536;  // 64 KB

constructor TFileTransferClient.Create;
begin
  inherited;
  FClient := TIdTCPClient.Create(nil);
end;

destructor TFileTransferClient.Destroy;
begin
  FClient.Free;
  inherited;
end;

function TFileTransferClient.Connect(const Host: string; Port: Integer): Boolean;
begin
  Result := False;
  try
    FClient.Host := Host;
    FClient.Port := Port;
    FClient.ConnectTimeout := 10000;
    FClient.Connect;
    Result := FClient.Connected;
  except
    // Connection failed
  end;
end;

procedure TFileTransferClient.Disconnect;
begin
  if FClient.Connected then
    FClient.Disconnect;
end;

function TFileTransferClient.SendFile(const FileName: string): Boolean;
var
  FS: TFileStream;
  FileSize: Int64;
  BytesSent: Int64;
  Buffer: TBytes;
  BytesToRead: Integer;
  Response: string;
begin
  Result := False;
  if not FileExists(FileName) then Exit;
  if not FClient.Connected then Exit;
  
  FS := TFileStream.Create(FileName, fmOpenRead or fmShareDenyNone);
  try
    FileSize := FS.Size;
    
    // ส่ง Header
    FClient.IOHandler.WriteLn(Format('SEND %s %d',
      [ExtractFileName(FileName), FileSize]));
    
    // รอ Response
    Response := FClient.IOHandler.ReadLn;
    if Response <> 'READY' then
    begin
      // Server ไม่พร้อม
      Exit;
    end;
    
    // ส่งข้อมูล
    SetLength(Buffer, BUFFER_SIZE);
    BytesSent := 0;
    
    while BytesSent < FileSize do
    begin
      BytesToRead := Min(BUFFER_SIZE, FileSize - BytesSent);
      FS.Read(Buffer[0], BytesToRead);
      FClient.IOHandler.Write(Buffer, BytesToRead);
      BytesSent := BytesSent + BytesToRead;
      
      if Assigned(FOnProgress) then
        FOnProgress(BytesSent, FileSize);
    end;
    
    // รอ Confirmation
    Response := FClient.IOHandler.ReadLn('', 30000);
    Result := Response = 'OK';
    
  finally
    FS.Free;
  end;
end;

function TFileTransferClient.ReceiveFile(const SaveDir: string): Boolean;
var
  Header: string;
  Parts: TStringArray;
  FileName: string;
  FileSize: Int64;
  FS: TFileStream;
  Buffer: TBytes;
  BytesReceived: Int64;
  BytesToRead: Integer;
begin
  Result := False;
  if not FClient.Connected then Exit;
  
  // รับ Header
  Header := FClient.IOHandler.ReadLn('', 30000);
  if not Header.StartsWith('SEND ') then Exit;
  
  Parts := Header.Split([' ']);
  if Length(Parts) < 3 then Exit;
  
  FileName := Parts[1];
  FileSize := StrToInt64(Parts[2]);
  
  // ส่ง Ready
  FClient.IOHandler.WriteLn('READY');
  
  // รับไฟล์
  var SavePath := IncludeTrailingPathDelimiter(SaveDir) + FileName;
  FS := TFileStream.Create(SavePath, fmCreate);
  try
    SetLength(Buffer, BUFFER_SIZE);
    BytesReceived := 0;
    
    while BytesReceived < FileSize do
    begin
      BytesToRead := Min(BUFFER_SIZE, FileSize - BytesReceived);
      FClient.IOHandler.ReadBytes(Buffer, BytesToRead, False);
      FS.Write(Buffer[0], BytesToRead);
      BytesReceived := BytesReceived + BytesToRead;
      
      if Assigned(FOnProgress) then
        FOnProgress(BytesReceived, FileSize);
    end;
    
    FClient.IOHandler.WriteLn('OK');
    Result := True;
    
  finally
    FS.Free;
  end;
end;

constructor TFileTransferServer.Create;
begin
  inherited;
  FServer := TIdTCPServer.Create(nil);
  FServer.OnExecute := OnServerExecute;
end;

destructor TFileTransferServer.Destroy;
begin
  FServer.Free;
  inherited;
end;

procedure TFileTransferServer.Start(Port: Integer; const SaveDirectory: string);
begin
  FSaveDir := SaveDirectory;
  FServer.DefaultPort := Port;
  FServer.Active := True;
end;

procedure TFileTransferServer.Stop;
begin
  FServer.Active := False;
end;

procedure TFileTransferServer.OnServerExecute(AContext: TIdContext);
var
  Header: string;
  Parts: TStringArray;
  FileName: string;
  FileSize: Int64;
  FS: TFileStream;
  Buffer: TBytes;
  BytesReceived: Int64;
  BytesToRead: Integer;
begin
  try
    Header := AContext.Connection.IOHandler.ReadLn('', 30000);
    
    if Header.StartsWith('SEND ') then
    begin
      Parts := Header.Split([' ']);
      if Length(Parts) < 3 then
      begin
        AContext.Connection.IOHandler.WriteLn('ERROR Invalid header');
        Exit;
      end;
      
      FileName := Parts[1];
      FileSize := StrToInt64(Parts[2]);
      
      // ตรวจสอบชื่อไฟล์
      if (FileName.IndexOf('/') >= 0) or (FileName.IndexOf('\') >= 0) then
      begin
        AContext.Connection.IOHandler.WriteLn('ERROR Invalid filename');
        Exit;
      end;
      
      AContext.Connection.IOHandler.WriteLn('READY');
      
      var SavePath := IncludeTrailingPathDelimiter(FSaveDir) + FileName;
      FS := TFileStream.Create(SavePath, fmCreate);
      try
        SetLength(Buffer, 65536);
        BytesReceived := 0;
        
        while BytesReceived < FileSize do
        begin
          BytesToRead := Min(65536, FileSize - BytesReceived);
          AContext.Connection.IOHandler.ReadBytes(Buffer, BytesToRead, False);
          FS.Write(Buffer[0], BytesToRead);
          BytesReceived := BytesReceived + BytesToRead;
          
          if Assigned(FOnProgress) then
            TThread.Synchronize(nil, procedure
            begin
              FOnProgress(BytesReceived, FileSize);
            end);
        end;
        
        AContext.Connection.IOHandler.WriteLn('OK');
      finally
        FS.Free;
      end;
    end;
  except
    on E: Exception do
    begin
      try
        AContext.Connection.IOHandler.WriteLn('ERROR ' + E.Message);
      except end;
    end;
  end;
end;

end.
```

---

## แบบฝึกหัดบทที่ 39

### ข้อ 1: Echo Server/Client
สร้าง Echo Server ที่ส่งข้อความกลับมาหา Client และ Client ที่ส่งข้อความทดสอบ

### ข้อ 2: Time Server
สร้าง Time Server ที่ส่งเวลาปัจจุบันเมื่อ Client เชื่อมต่อ

### ข้อ 3: Calculator Server
สร้าง Server ที่รับ Expression เช่น "2+3" และส่งผลลัพธ์กลับ

### ข้อ 4: Multi-room Chat
พัฒนา Chat Application ให้รองรับหลายห้องสนทนา (Rooms)

### ข้อ 5: UDP Discovery
สร้างระบบค้นหา Services ใน Local Network ผ่าน UDP Broadcast

```pascal
// แนวทาง: ส่ง Broadcast Discover และรอ Response
procedure DiscoverServices(Port: Integer; var Services: TStringList);
var
  Client: TIdUDPClient;
  Server: TIdUDPServer;
begin
  Client.BroadcastEnabled := True;
  Client.Send('255.255.255.255', Port, 'DISCOVER');
  // รอ Response ใน OnUDPRead
end;
```

### ข้อ 6: FTP Client อย่างง่าย
สร้าง FTP Client ที่ Connect, List, Download, Upload ไฟล์ได้

### ข้อ 7: SMTP Client
สร้างโปรแกรมส่ง Email ผ่าน SMTP protocol โดยตรง

### ข้อ 8: Network File Sharing
สร้างระบบ Share ไฟล์ใน LAN แบบ Peer-to-Peer

### ข้อ 9: Secure Chat ด้วย SSL/TLS
เพิ่ม SSL/TLS ให้กับ Chat Application โดยใช้ OpenSSL

```pascal
// แนวทาง: ใช้ TIdSSLIOHandlerSocketOpenSSL
uses
  IdSSLOpenSSL, IdSSLOpenSSLHeaders;

procedure TForm1.SetupSSL;
var
  SSLHandler: TIdSSLIOHandlerSocketOpenSSL;
begin
  SSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(nil);
  SSLHandler.SSLOptions.Method := sslvSSLv23;
  SSLHandler.SSLOptions.Mode := sslmClient;
  IdTCPClient1.IOHandler := SSLHandler;
end;
```

### ข้อ 10: Load Balancer
สร้าง Simple Load Balancer ที่กระจาย Request ไปยัง Server หลายตัว

---

## สรุปบทที่ 39

ในบทนี้เราได้เรียนรู้:
1. **แนวคิด TCP/IP** - Protocol, Ports, Client-Server model
2. **TCP Client** - TIdTCPClient, Connect, Send, Receive
3. **TCP Server** - TIdTCPServer, Multi-client, Thread-safe
4. **Multi-Client Server** - Chat server พร้อมห้องและ PM
5. **UDP** - TIdUDPClient, TIdUDPServer, Broadcast, Multicast
6. **Port Scanner** - Scan ports แบบ Multi-thread
7. **Ping** - ICMP Implementation ด้วย Indy
8. **Network Monitoring** - วิเคราะห์ Traffic แบบ Real-time
9. **Chat Application** - Full-featured Chat ทั้ง Client และ Server
10. **File Transfer** - ส่งไฟล์ผ่าน TCP
