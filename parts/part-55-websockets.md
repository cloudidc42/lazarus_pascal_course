# Part 55 - WebSockets ใน Lazarus/Pascal

## บทนำ

WebSocket เป็น Protocol ที่ช่วยให้ Client และ Server สื่อสารแบบ Two-way (Bidirectional) ผ่าน Connection เดียว ต่างจาก HTTP ที่เป็น Request-Response

### HTTP vs WebSocket

```
HTTP (Request-Response):
Client → Request → Server
Client ← Response ← Server
(ต้องสร้าง Connection ใหม่ทุกครั้ง)

WebSocket (Full-Duplex):
Client ←─────────────────→ Server
(Connection เดียว ส่งได้ทั้งสองทาง)
```

### WebSocket Handshake

```
Client → HTTP Upgrade Request:
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → 101 Switching Protocols:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

---

## การติดตั้ง WebSocket ใน Lazarus

ใช้ **Indy** (ที่มากับ Lazarus) หรือ **lNet** หรือ **fpWeb WebSocket**

### ติดตั้ง Package
```
Component → Package → Open Package File
เปิด: /usr/lib/fpc/3.x.x/units/x86_64-linux/IndyProtocols.pas
หรือใช้ fpWebSocket จาก fpc-source
```

---

## WebSocket Server

```pascal
unit WebSocketServer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, BlckSock, Sockets,
  Generics.Collections, base64, sha1;

type
  TWSOpCode = (
    wsoContinuation = $0,
    wsoText         = $1,
    wsoBinary       = $2,
    wsoClose        = $8,
    wsoPing         = $9,
    wsoPong         = $A
  );

  TWebSocketFrame = record
    OpCode: TWSOpCode;
    IsFin: Boolean;
    IsMasked: Boolean;
    Payload: TBytes;
    MaskKey: array[0..3] of Byte;
  end;

  TWSClient = class;
  
  TWSMessageEvent = procedure(AClient: TWSClient; const AMessage: string) of object;
  TWSBinaryEvent = procedure(AClient: TWSClient; const AData: TBytes) of object;
  TWSClientEvent = procedure(AClient: TWSClient) of object;
  TWSErrorEvent = procedure(AClient: TWSClient; const AError: string) of object;

  { WebSocket Client Connection }
  TWSClient = class
  private
    FSocket: TTCPBlockSocket;
    FID: string;
    FRoom: string;
    FUserID: Integer;
    FUsername: string;
    FConnectedAt: TDateTime;
    FIsConnected: Boolean;
    FLock: TCriticalSection;
    
    function ReadFrame(out AFrame: TWebSocketFrame): Boolean;
    function SendFrame(AOpCode: TWSOpCode; const AData: TBytes): Boolean;
    procedure ProcessHandshake;
    procedure UnmaskPayload(var APayload: TBytes; const AMask: array of Byte);
    function EncodeAcceptKey(const AKey: string): string;
  public
    constructor Create(ASocket: TTCPBlockSocket);
    destructor Destroy; override;
    
    function SendText(const AMessage: string): Boolean;
    function SendBinary(const AData: TBytes): Boolean;
    function SendPing: Boolean;
    procedure Close(ACode: Word = 1000; const AReason: string = '');
    
    property ID: string read FID;
    property Room: string read FRoom write FRoom;
    property UserID: Integer read FUserID write FUserID;
    property Username: string read FUsername write FUsername;
    property ConnectedAt: TDateTime read FConnectedAt;
    property IsConnected: Boolean read FIsConnected;
  end;

  { WebSocket Room }
  TWSRoom = class
  private
    FName: string;
    FClients: specialize TObjectList<TWSClient>;
    FLock: TCriticalSection;
    FMaxClients: Integer;
  public
    constructor Create(const AName: string; AMaxClients: Integer = 0);
    destructor Destroy; override;
    procedure AddClient(AClient: TWSClient);
    procedure RemoveClient(AClient: TWSClient);
    procedure Broadcast(const AMessage: string; AExclude: TWSClient = nil);
    function GetClientCount: Integer;
    function GetClients: specialize TObjectList<TWSClient>;
    property Name: string read FName;
    property MaxClients: Integer read FMaxClients write FMaxClients;
  end;

  { WebSocket Server }
  TWebSocketServer = class
  private
    FServerSocket: TTCPBlockSocket;
    FClients: specialize TObjectList<TWSClient>;
    FRooms: specialize TObjectDictionary<string, TWSRoom>;
    FLock: TCriticalSection;
    FPort: Integer;
    FRunning: Boolean;
    FListenThread: TThread;
    
    FOnClientConnect: TWSClientEvent;
    FOnClientDisconnect: TWSClientEvent;
    FOnMessage: TWSMessageEvent;
    FOnBinaryMessage: TWSBinaryEvent;
    FOnError: TWSErrorEvent;
    
    procedure AcceptClients;
    procedure HandleClient(AClient: TWSClient);
    function GenerateClientID: string;
  public
    constructor Create(APort: Integer);
    destructor Destroy; override;
    
    procedure Start;
    procedure Stop;
    
    { Room Management }
    function CreateRoom(const ARoomName: string): TWSRoom;
    function GetRoom(const ARoomName: string): TWSRoom;
    procedure DeleteRoom(const ARoomName: string);
    procedure JoinRoom(AClient: TWSClient; const ARoomName: string);
    procedure LeaveRoom(AClient: TWSClient);
    
    { Messaging }
    procedure Broadcast(const AMessage: string; AExclude: TWSClient = nil);
    procedure BroadcastToRoom(const ARoomName, AMessage: string; AExclude: TWSClient = nil);
    function SendToClient(const AClientID, AMessage: string): Boolean;
    function SendToUser(AUserID: Integer; const AMessage: string): Boolean;
    
    { Client Management }
    function GetClient(const AClientID: string): TWSClient;
    function GetClientsByUser(AUserID: Integer): specialize TObjectList<TWSClient>;
    function GetClientCount: Integer;
    function GetRoomCount: Integer;
    
    property Port: Integer read FPort;
    property Running: Boolean read FRunning;
    property OnClientConnect: TWSClientEvent read FOnClientConnect write FOnClientConnect;
    property OnClientDisconnect: TWSClientEvent read FOnClientDisconnect write FOnClientDisconnect;
    property OnMessage: TWSMessageEvent read FOnMessage write FOnMessage;
    property OnBinaryMessage: TWSBinaryEvent read FOnBinaryMessage write FOnBinaryMessage;
    property OnError: TWSErrorEvent read FOnError write FOnError;
  end;

implementation

{ TWSClient }
constructor TWSClient.Create(ASocket: TTCPBlockSocket);
begin
  FSocket := ASocket;
  FConnectedAt := Now;
  FIsConnected := True;
  FLock := TCriticalSection.Create;
  FID := '';
  FRoom := '';
  FUserID := 0;
  FUsername := 'Anonymous';
end;

destructor TWSClient.Destroy;
begin
  FLock.Free;
  { ASocket ถูก Free โดย Server }
  inherited;
end;

function TWSClient.EncodeAcceptKey(const AKey: string): string;
var
  Combined: string;
  Hash: TSHA1Digest;
begin
  Combined := AKey + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
  Hash := SHA1String(Combined);
  Result := EncodeStringBase64(string(Hash));
end;

procedure TWSClient.ProcessHandshake;
var
  Line, Key: string;
  Headers: TStringList;
  Response: string;
begin
  Headers := TStringList.Create;
  try
    { อ่าน HTTP Request Headers }
    repeat
      Line := FSocket.RecvString(5000);
      if Line <> '' then
        Headers.Add(Line);
    until Line = '';
    
    { หา WebSocket Key }
    Key := '';
    for Line in Headers do
      if Copy(Line, 1, 22) = 'Sec-WebSocket-Key: ' then
      begin
        Key := Trim(Copy(Line, 20, MaxInt));
        Break;
      end;
    
    if Key = '' then
      raise Exception.Create('ไม่พบ WebSocket Key ใน Headers');
    
    { ส่ง Response }
    Response :=
      'HTTP/1.1 101 Switching Protocols' + #13#10 +
      'Upgrade: websocket' + #13#10 +
      'Connection: Upgrade' + #13#10 +
      'Sec-WebSocket-Accept: ' + EncodeAcceptKey(Key) + #13#10 +
      #13#10;
    
    FSocket.SendString(Response);
  finally
    Headers.Free;
  end;
end;

procedure TWSClient.UnmaskPayload(var APayload: TBytes; const AMask: array of Byte);
var
  I: Integer;
begin
  for I := 0 to High(APayload) do
    APayload[I] := APayload[I] xor AMask[I mod 4];
end;

function TWSClient.ReadFrame(out AFrame: TWebSocketFrame): Boolean;
var
  B1, B2: Byte;
  PayloadLen: Int64;
  I: Integer;
begin
  Result := False;
  
  if FSocket.RecvBufferEx(@B1, 1, 5000) < 1 then Exit;
  if FSocket.RecvBufferEx(@B2, 1, 5000) < 1 then Exit;
  
  AFrame.IsFin := (B1 and $80) <> 0;
  AFrame.OpCode := TWSOpCode(B1 and $0F);
  AFrame.IsMasked := (B2 and $80) <> 0;
  
  PayloadLen := B2 and $7F;
  if PayloadLen = 126 then
  begin
    var Len16: Word;
    FSocket.RecvBufferEx(@Len16, 2, 5000);
    PayloadLen := (Len16 shr 8) or (Len16 shl 8);
  end
  else if PayloadLen = 127 then
  begin
    var Len64: Int64;
    FSocket.RecvBufferEx(@Len64, 8, 5000);
    PayloadLen := Len64;
  end;
  
  if AFrame.IsMasked then
    FSocket.RecvBufferEx(@AFrame.MaskKey, 4, 5000);
  
  if PayloadLen > 0 then
  begin
    SetLength(AFrame.Payload, PayloadLen);
    FSocket.RecvBufferEx(@AFrame.Payload[0], PayloadLen, 10000);
    
    if AFrame.IsMasked then
      UnmaskPayload(AFrame.Payload, AFrame.MaskKey);
  end
  else
    SetLength(AFrame.Payload, 0);
  
  Result := True;
end;

function TWSClient.SendFrame(AOpCode: TWSOpCode; const AData: TBytes): Boolean;
var
  Header: TBytes;
  HeaderLen: Integer;
  DataLen: Int64;
begin
  Result := False;
  if not FIsConnected then Exit;
  
  FLock.Enter;
  try
    DataLen := Length(AData);
    
    if DataLen <= 125 then
    begin
      SetLength(Header, 2);
      Header[0] := $80 or Byte(AOpCode);  { FIN + OpCode }
      Header[1] := Byte(DataLen);          { ไม่ Mask (Server ไม่ต้อง Mask) }
      HeaderLen := 2;
    end
    else if DataLen <= 65535 then
    begin
      SetLength(Header, 4);
      Header[0] := $80 or Byte(AOpCode);
      Header[1] := 126;
      Header[2] := (DataLen shr 8) and $FF;
      Header[3] := DataLen and $FF;
      HeaderLen := 4;
    end
    else
    begin
      SetLength(Header, 10);
      Header[0] := $80 or Byte(AOpCode);
      Header[1] := 127;
      { Write 8-byte length big-endian }
      Header[2] := 0; Header[3] := 0; Header[4] := 0; Header[5] := 0;
      Header[6] := (DataLen shr 24) and $FF;
      Header[7] := (DataLen shr 16) and $FF;
      Header[8] := (DataLen shr 8) and $FF;
      Header[9] := DataLen and $FF;
      HeaderLen := 10;
    end;
    
    FSocket.SendBuffer(@Header[0], HeaderLen);
    if DataLen > 0 then
      FSocket.SendBuffer(@AData[0], DataLen);
    
    Result := FSocket.LastError = 0;
  finally
    FLock.Leave;
  end;
end;

function TWSClient.SendText(const AMessage: string): Boolean;
var
  Data: TBytes;
begin
  Data := TEncoding.UTF8.GetBytes(AMessage);
  Result := SendFrame(wsoText, Data);
end;

function TWSClient.SendBinary(const AData: TBytes): Boolean;
begin
  Result := SendFrame(wsoBinary, AData);
end;

function TWSClient.SendPing: Boolean;
begin
  Result := SendFrame(wsoPing, nil);
end;

procedure TWSClient.Close(ACode: Word; const AReason: string);
var
  Data: TBytes;
  ReasonBytes: TBytes;
begin
  if not FIsConnected then Exit;
  FIsConnected := False;
  
  SetLength(Data, 2 + Length(AReason));
  Data[0] := (ACode shr 8) and $FF;
  Data[1] := ACode and $FF;
  
  if AReason <> '' then
  begin
    ReasonBytes := TEncoding.UTF8.GetBytes(AReason);
    Move(ReasonBytes[0], Data[2], Length(ReasonBytes));
  end;
  
  SendFrame(wsoClose, Data);
end;

{ TWSRoom }
constructor TWSRoom.Create(const AName: string; AMaxClients: Integer);
begin
  FName := AName;
  FMaxClients := AMaxClients;
  FClients := specialize TObjectList<TWSClient>.Create(False);
  FLock := TCriticalSection.Create;
end;

destructor TWSRoom.Destroy;
begin
  FClients.Free;
  FLock.Free;
  inherited;
end;

procedure TWSRoom.AddClient(AClient: TWSClient);
begin
  FLock.Enter;
  try
    if (FMaxClients > 0) and (FClients.Count >= FMaxClients) then
      raise Exception.CreateFmt('ห้อง %s เต็มแล้ว (สูงสุด %d คน)', [FName, FMaxClients]);
    
    if FClients.IndexOf(AClient) < 0 then
      FClients.Add(AClient);
  finally
    FLock.Leave;
  end;
end;

procedure TWSRoom.RemoveClient(AClient: TWSClient);
begin
  FLock.Enter;
  try
    FClients.Remove(AClient);
  finally
    FLock.Leave;
  end;
end;

procedure TWSRoom.Broadcast(const AMessage: string; AExclude: TWSClient);
var
  Client: TWSClient;
begin
  FLock.Enter;
  try
    for Client in FClients do
      if (Client <> AExclude) and Client.IsConnected then
        Client.SendText(AMessage);
  finally
    FLock.Leave;
  end;
end;

function TWSRoom.GetClientCount: Integer;
begin
  FLock.Enter;
  try
    Result := FClients.Count;
  finally
    FLock.Leave;
  end;
end;

function TWSRoom.GetClients: specialize TObjectList<TWSClient>;
begin
  Result := FClients;
end;

{ TWebSocketServer }
constructor TWebSocketServer.Create(APort: Integer);
begin
  FPort := APort;
  FRunning := False;
  FClients := specialize TObjectList<TWSClient>.Create(True);
  FRooms := specialize TObjectDictionary<string, TWSRoom>.Create([doOwnsValues]);
  FLock := TCriticalSection.Create;
  FServerSocket := TTCPBlockSocket.Create;
end;

destructor TWebSocketServer.Destroy;
begin
  Stop;
  FClients.Free;
  FRooms.Free;
  FLock.Free;
  FServerSocket.Free;
  inherited;
end;

function TWebSocketServer.GenerateClientID: string;
begin
  Result := Format('ws_%s_%d', 
    [FormatDateTime('hhnnsszzz', Now), Random(9999)]);
end;

procedure TWebSocketServer.Start;
begin
  FServerSocket.CreateSocket;
  FServerSocket.SetLinger(True, 1);
  FServerSocket.Bind('0.0.0.0', IntToStr(FPort));
  FServerSocket.Listen;
  FRunning := True;
  
  WriteLn(Format('WebSocket Server เริ่มทำงานที่ ws://localhost:%d', [FPort]));
  
  { เริ่ม Accept Loop ใน Background Thread }
  FListenThread := TThread.CreateAnonymousThread(AcceptClients);
  FListenThread.FreeOnTerminate := False;
  FListenThread.Start;
end;

procedure TWebSocketServer.Stop;
begin
  FRunning := False;
  FServerSocket.CloseSocket;
  if FListenThread <> nil then
  begin
    FListenThread.WaitFor;
    FreeAndNil(FListenThread);
  end;
  WriteLn('WebSocket Server หยุดทำงาน');
end;

procedure TWebSocketServer.AcceptClients;
var
  ClientSocket: TTCPBlockSocket;
  Client: TWSClient;
  ClientThread: TThread;
begin
  while FRunning do
  begin
    if FServerSocket.CanRead(100) then
    begin
      ClientSocket := TTCPBlockSocket.Create;
      ClientSocket.Socket := FServerSocket.Accept;
      
      if FServerSocket.LastError = 0 then
      begin
        Client := TWSClient.Create(ClientSocket);
        Client.FID := GenerateClientID;
        
        FLock.Enter;
        try
          FClients.Add(Client);
        finally
          FLock.Leave;
        end;
        
        { Handle Client ใน Thread แยก }
        ClientThread := TThread.CreateAnonymousThread(
          procedure
          begin
            HandleClient(Client);
          end
        );
        ClientThread.FreeOnTerminate := True;
        ClientThread.Start;
      end
      else
        ClientSocket.Free;
    end;
  end;
end;

procedure TWebSocketServer.HandleClient(AClient: TWSClient);
var
  Frame: TWebSocketFrame;
  Message: string;
begin
  try
    { WebSocket Handshake }
    AClient.ProcessHandshake;
    
    if Assigned(FOnClientConnect) then
      FOnClientConnect(AClient);
    
    { Main Message Loop }
    while AClient.IsConnected do
    begin
      if not AClient.ReadFrame(Frame) then Break;
      
      case Frame.OpCode of
        wsoText:
        begin
          Message := TEncoding.UTF8.GetString(Frame.Payload);
          if Assigned(FOnMessage) then
            FOnMessage(AClient, Message);
        end;
        
        wsoBinary:
        begin
          if Assigned(FOnBinaryMessage) then
            FOnBinaryMessage(AClient, Frame.Payload);
        end;
        
        wsoClose:
        begin
          AClient.Close(1000, 'Normal Closure');
          Break;
        end;
        
        wsoPing:
        begin
          { ตอบ Pong }
          AClient.SendFrame(wsoPong, Frame.Payload);
        end;
        
        wsoPong: { ไม่ต้องทำอะไร } ;
      end;
    end;
  except
    on E: Exception do
      if Assigned(FOnError) then FOnError(AClient, E.Message);
  end;
  
  { Cleanup }
  AClient.FIsConnected := False;
  
  if AClient.Room <> '' then
    LeaveRoom(AClient);
  
  if Assigned(FOnClientDisconnect) then
    FOnClientDisconnect(AClient);
  
  FLock.Enter;
  try
    FClients.Remove(AClient);
  finally
    FLock.Leave;
  end;
end;

function TWebSocketServer.CreateRoom(const ARoomName: string): TWSRoom;
begin
  FLock.Enter;
  try
    if FRooms.ContainsKey(ARoomName) then
      Result := FRooms[ARoomName]
    else
    begin
      Result := TWSRoom.Create(ARoomName);
      FRooms.Add(ARoomName, Result);
    end;
  finally
    FLock.Leave;
  end;
end;

function TWebSocketServer.GetRoom(const ARoomName: string): TWSRoom;
begin
  FLock.Enter;
  try
    if FRooms.ContainsKey(ARoomName) then
      Result := FRooms[ARoomName]
    else
      Result := nil;
  finally
    FLock.Leave;
  end;
end;

procedure TWebSocketServer.DeleteRoom(const ARoomName: string);
begin
  FLock.Enter;
  try
    FRooms.Remove(ARoomName);
  finally
    FLock.Leave;
  end;
end;

procedure TWebSocketServer.JoinRoom(AClient: TWSClient; const ARoomName: string);
var
  Room: TWSRoom;
begin
  { ออกจาก Room เดิมก่อน }
  if AClient.Room <> '' then
    LeaveRoom(AClient);
  
  Room := GetRoom(ARoomName);
  if Room = nil then
    Room := CreateRoom(ARoomName);
  
  Room.AddClient(AClient);
  AClient.FRoom := ARoomName;
end;

procedure TWebSocketServer.LeaveRoom(AClient: TWSClient);
var
  Room: TWSRoom;
begin
  if AClient.Room = '' then Exit;
  
  Room := GetRoom(AClient.Room);
  if Room <> nil then
  begin
    Room.RemoveClient(AClient);
    { ลบ Room ถ้าว่าง }
    if Room.GetClientCount = 0 then
      DeleteRoom(AClient.Room);
  end;
  AClient.FRoom := '';
end;

procedure TWebSocketServer.Broadcast(const AMessage: string; AExclude: TWSClient);
var
  Client: TWSClient;
begin
  FLock.Enter;
  try
    for Client in FClients do
      if (Client <> AExclude) and Client.IsConnected then
        Client.SendText(AMessage);
  finally
    FLock.Leave;
  end;
end;

procedure TWebSocketServer.BroadcastToRoom(const ARoomName, AMessage: string; AExclude: TWSClient);
var
  Room: TWSRoom;
begin
  Room := GetRoom(ARoomName);
  if Room <> nil then
    Room.Broadcast(AMessage, AExclude);
end;

function TWebSocketServer.SendToClient(const AClientID, AMessage: string): Boolean;
var
  Client: TWSClient;
begin
  Client := GetClient(AClientID);
  if (Client <> nil) and Client.IsConnected then
    Result := Client.SendText(AMessage)
  else
    Result := False;
end;

function TWebSocketServer.SendToUser(AUserID: Integer; const AMessage: string): Boolean;
var
  Clients: specialize TObjectList<TWSClient>;
  Client: TWSClient;
begin
  Result := False;
  Clients := GetClientsByUser(AUserID);
  try
    for Client in Clients do
      if Client.IsConnected then
      begin
        Client.SendText(AMessage);
        Result := True;
      end;
  finally
    Clients.Free;
  end;
end;

function TWebSocketServer.GetClient(const AClientID: string): TWSClient;
var
  Client: TWSClient;
begin
  Result := nil;
  FLock.Enter;
  try
    for Client in FClients do
      if Client.ID = AClientID then
      begin
        Result := Client;
        Exit;
      end;
  finally
    FLock.Leave;
  end;
end;

function TWebSocketServer.GetClientsByUser(AUserID: Integer): specialize TObjectList<TWSClient>;
var
  Client: TWSClient;
begin
  Result := specialize TObjectList<TWSClient>.Create(False);
  FLock.Enter;
  try
    for Client in FClients do
      if Client.UserID = AUserID then
        Result.Add(Client);
  finally
    FLock.Leave;
  end;
end;

function TWebSocketServer.GetClientCount: Integer;
begin
  FLock.Enter;
  try
    Result := FClients.Count;
  finally
    FLock.Leave;
  end;
end;

function TWebSocketServer.GetRoomCount: Integer;
begin
  FLock.Enter;
  try
    Result := FRooms.Count;
  finally
    FLock.Leave;
  end;
end;

end.
```

---

## Real-time Chat Server

```pascal
unit ChatServer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, jsonparser, DateUtils,
  WebSocketServer;

type
  TChatMessageType = (
    cmtJoin,       { ผู้ใช้เข้าห้อง }
    cmtLeave,      { ผู้ใช้ออกจากห้อง }
    cmtMessage,    { ข้อความทั่วไป }
    cmtPrivate,    { ข้อความส่วนตัว }
    cmtSystem,     { ข้อความจากระบบ }
    cmtUserList,   { รายชื่อผู้ใช้ในห้อง }
    cmtTyping,     { กำลังพิมพ์ }
    cmtError       { ข้อผิดพลาด }
  );

  TChatMessage = record
    MsgType: TChatMessageType;
    FromID: string;
    FromName: string;
    ToID: string;      { สำหรับ Private Message }
    Room: string;
    Content: string;
    Timestamp: TDateTime;
  end;

  TChatServer = class
  private
    FWSServer: TWebSocketServer;
    FDefaultRoom: string;
    
    function ParseMessage(const ARaw: string; out AMsg: TChatMessage): Boolean;
    function CreateMessage(AMsgType: TChatMessageType; 
                           const AFrom, ARoom, AContent: string): string;
    function CreateSystemMessage(const AContent: string): string;
    function CreateUserListMessage(const ARoomName: string): string;
    
    procedure HandleConnect(AClient: TWSClient);
    procedure HandleDisconnect(AClient: TWSClient);
    procedure HandleMessage(AClient: TWSClient; const AMessage: string);
    
    procedure ProcessJoin(AClient: TWSClient; const AMsg: TChatMessage);
    procedure ProcessMessage(AClient: TWSClient; const AMsg: TChatMessage);
    procedure ProcessPrivate(AClient: TWSClient; const AMsg: TChatMessage);
    procedure ProcessTyping(AClient: TWSClient; const AMsg: TChatMessage);
    
    function GetMsgTypeName(AMsgType: TChatMessageType): string;
    function ParseMsgType(const AName: string): TChatMessageType;
    function GetClientByUsername(const AUsername: string): TWSClient;
  public
    constructor Create(APort: Integer);
    destructor Destroy; override;
    
    procedure Start;
    procedure Stop;
    
    property DefaultRoom: string read FDefaultRoom write FDefaultRoom;
  end;

implementation

constructor TChatServer.Create(APort: Integer);
begin
  FWSServer := TWebSocketServer.Create(APort);
  FWSServer.OnClientConnect := HandleConnect;
  FWSServer.OnClientDisconnect := HandleDisconnect;
  FWSServer.OnMessage := HandleMessage;
  FDefaultRoom := 'general';
end;

destructor TChatServer.Destroy;
begin
  FWSServer.Free;
  inherited;
end;

procedure TChatServer.Start;
begin
  FWSServer.CreateRoom(FDefaultRoom);
  FWSServer.Start;
  WriteLn('Chat Server เริ่มทำงาน...');
end;

procedure TChatServer.Stop;
begin
  FWSServer.Stop;
end;

function TChatServer.GetMsgTypeName(AMsgType: TChatMessageType): string;
begin
  case AMsgType of
    cmtJoin:     Result := 'join';
    cmtLeave:    Result := 'leave';
    cmtMessage:  Result := 'message';
    cmtPrivate:  Result := 'private';
    cmtSystem:   Result := 'system';
    cmtUserList: Result := 'user_list';
    cmtTyping:   Result := 'typing';
    cmtError:    Result := 'error';
  else
    Result := 'unknown';
  end;
end;

function TChatServer.ParseMsgType(const AName: string): TChatMessageType;
begin
  case LowerCase(AName) of
    'join':      Result := cmtJoin;
    'leave':     Result := cmtLeave;
    'message':   Result := cmtMessage;
    'private':   Result := cmtPrivate;
    'typing':    Result := cmtTyping;
  else
    Result := cmtMessage;
  end;
end;

function TChatServer.ParseMessage(const ARaw: string; out AMsg: TChatMessage): Boolean;
var
  Parser: TJSONParser;
  Json: TJSONObject;
begin
  Result := False;
  FillChar(AMsg, SizeOf(AMsg), 0);
  
  Parser := TJSONParser.Create(ARaw, []);
  try
    try
      Json := TJSONObject(Parser.Parse);
      try
        AMsg.MsgType := ParseMsgType(Json.Get('type', 'message'));
        AMsg.Content := Json.Get('content', '');
        AMsg.Room := Json.Get('room', FDefaultRoom);
        AMsg.ToID := Json.Get('to', '');
        AMsg.Timestamp := Now;
        Result := True;
      finally
        Json.Free;
      end;
    except
      { Invalid JSON }
    end;
  finally
    Parser.Free;
  end;
end;

function TChatServer.CreateMessage(AMsgType: TChatMessageType;
  const AFrom, ARoom, AContent: string): string;
var
  Json: TJSONObject;
begin
  Json := TJSONObject.Create;
  try
    Json.Add('type', GetMsgTypeName(AMsgType));
    Json.Add('from', AFrom);
    Json.Add('room', ARoom);
    Json.Add('content', AContent);
    Json.Add('timestamp', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss', Now));
    Result := Json.AsJSON;
  finally
    Json.Free;
  end;
end;

function TChatServer.CreateSystemMessage(const AContent: string): string;
begin
  Result := CreateMessage(cmtSystem, 'SYSTEM', '', AContent);
end;

function TChatServer.CreateUserListMessage(const ARoomName: string): string;
var
  Json: TJSONObject;
  UsersArray: TJSONArray;
  Room: TWSRoom;
  Client: TWSClient;
  UserObj: TJSONObject;
begin
  Room := FWSServer.GetRoom(ARoomName);
  
  Json := TJSONObject.Create;
  try
    Json.Add('type', 'user_list');
    Json.Add('room', ARoomName);
    
    UsersArray := TJSONArray.Create;
    if Room <> nil then
    begin
      for Client in Room.GetClients do
      begin
        UserObj := TJSONObject.Create;
        UserObj.Add('id', Client.ID);
        UserObj.Add('username', Client.Username);
        UserObj.Add('joined_at', FormatDateTime('hh:nn:ss', Client.ConnectedAt));
        UsersArray.Add(UserObj);
      end;
    end;
    Json.Add('users', UsersArray);
    Json.Add('count', UsersArray.Count);
    
    Result := Json.AsJSON;
  finally
    Json.Free;
  end;
end;

procedure TChatServer.HandleConnect(AClient: TWSClient);
begin
  WriteLn(Format('[%s] Client เชื่อมต่อ: %s', 
    [FormatDateTime('hh:nn:ss', Now), AClient.ID]));
  
  { ส่งข้อความต้อนรับ }
  AClient.SendText(CreateSystemMessage(
    'ยินดีต้อนรับสู่ Chat Server! กรุณาส่งข้อความ join เพื่อเข้าห้องสนทนา'));
end;

procedure TChatServer.HandleDisconnect(AClient: TWSClient);
var
  Room: string;
  LeaveMsg: string;
begin
  Room := AClient.Room;
  
  WriteLn(Format('[%s] Client ตัดการเชื่อมต่อ: %s (%s)', 
    [FormatDateTime('hh:nn:ss', Now), AClient.Username, AClient.ID]));
  
  if Room <> '' then
  begin
    LeaveMsg := CreateMessage(cmtLeave, AClient.Username, Room,
      AClient.Username + ' ได้ออกจากห้องสนทนา');
    FWSServer.BroadcastToRoom(Room, LeaveMsg, AClient);
    
    { อัปเดตรายชื่อผู้ใช้ }
    FWSServer.BroadcastToRoom(Room, CreateUserListMessage(Room));
  end;
end;

procedure TChatServer.HandleMessage(AClient: TWSClient; const AMessage: string);
var
  Msg: TChatMessage;
begin
  if not ParseMessage(AMessage, Msg) then
  begin
    AClient.SendText(CreateSystemMessage('รูปแบบข้อมูลไม่ถูกต้อง (ต้องเป็น JSON)'));
    Exit;
  end;
  
  case Msg.MsgType of
    cmtJoin:    ProcessJoin(AClient, Msg);
    cmtMessage: ProcessMessage(AClient, Msg);
    cmtPrivate: ProcessPrivate(AClient, Msg);
    cmtTyping:  ProcessTyping(AClient, Msg);
  end;
end;

procedure TChatServer.ProcessJoin(AClient: TWSClient; const AMsg: TChatMessage);
var
  Room, OldRoom: string;
  JoinMsg: string;
begin
  Room := AMsg.Room;
  if Room = '' then Room := FDefaultRoom;
  
  { ตั้งชื่อ }
  if AMsg.Content <> '' then
    AClient.Username := AMsg.Content
  else
    AClient.Username := 'User_' + Copy(AClient.ID, 1, 6);
  
  OldRoom := AClient.Room;
  
  { เข้าห้องใหม่ }
  FWSServer.JoinRoom(AClient, Room);
  
  { แจ้ง Room เดิม (ถ้ามี) }
  if OldRoom <> '' then
  begin
    FWSServer.BroadcastToRoom(OldRoom,
      CreateMessage(cmtLeave, AClient.Username, OldRoom,
        AClient.Username + ' ย้ายไปห้อง ' + Room));
    FWSServer.BroadcastToRoom(OldRoom, CreateUserListMessage(OldRoom));
  end;
  
  { ส่ง Welcome ให้ Client }
  AClient.SendText(CreateMessage(cmtJoin, 'SYSTEM', Room,
    Format('คุณเข้าร่วมห้อง "%s" สำเร็จ', [Room])));
  
  { แจ้ง Room ใหม่ }
  JoinMsg := CreateMessage(cmtJoin, AClient.Username, Room,
    AClient.Username + ' เข้าร่วมห้องสนทนา');
  FWSServer.BroadcastToRoom(Room, JoinMsg, AClient);
  
  { ส่งรายชื่อผู้ใช้ }
  AClient.SendText(CreateUserListMessage(Room));
  
  WriteLn(Format('[%s] %s เข้าห้อง "%s"', 
    [FormatDateTime('hh:nn:ss', Now), AClient.Username, Room]));
end;

procedure TChatServer.ProcessMessage(AClient: TWSClient; const AMsg: TChatMessage);
var
  Room: string;
  ChatMsg: string;
begin
  if AClient.Room = '' then
  begin
    AClient.SendText(CreateSystemMessage('กรุณาเข้าร่วมห้องก่อน'));
    Exit;
  end;
  
  if Trim(AMsg.Content) = '' then Exit;
  
  Room := AClient.Room;
  ChatMsg := CreateMessage(cmtMessage, AClient.Username, Room, AMsg.Content);
  
  { ส่งให้ทุกคนใน Room รวมถึง Sender }
  FWSServer.BroadcastToRoom(Room, ChatMsg);
  
  WriteLn(Format('[%s][%s] %s: %s',
    [FormatDateTime('hh:nn:ss', Now), Room, AClient.Username, AMsg.Content]));
end;

procedure TChatServer.ProcessPrivate(AClient: TWSClient; const AMsg: TChatMessage);
var
  TargetClient: TWSClient;
  PrivateMsg: string;
begin
  if AMsg.ToID = '' then
  begin
    AClient.SendText(CreateSystemMessage('ต้องระบุผู้รับ (to)'));
    Exit;
  end;
  
  TargetClient := GetClientByUsername(AMsg.ToID);
  if TargetClient = nil then
  begin
    AClient.SendText(CreateSystemMessage('ไม่พบผู้ใช้: ' + AMsg.ToID));
    Exit;
  end;
  
  { ส่งให้ผู้รับ }
  PrivateMsg := CreateMessage(cmtPrivate, AClient.Username, '', AMsg.Content);
  TargetClient.SendText(PrivateMsg);
  
  { ยืนยันกลับไปยัง Sender }
  AClient.SendText(CreateMessage(cmtPrivate, 'ถึง: ' + AMsg.ToID, '', AMsg.Content));
end;

procedure TChatServer.ProcessTyping(AClient: TWSClient; const AMsg: TChatMessage);
var
  TypingMsg: TJSONObject;
begin
  if AClient.Room = '' then Exit;
  
  TypingMsg := TJSONObject.Create;
  try
    TypingMsg.Add('type', 'typing');
    TypingMsg.Add('from', AClient.Username);
    TypingMsg.Add('room', AClient.Room);
    TypingMsg.Add('is_typing', AMsg.Content = 'true');
    
    FWSServer.BroadcastToRoom(AClient.Room, TypingMsg.AsJSON, AClient);
  finally
    TypingMsg.Free;
  end;
end;

function TChatServer.GetClientByUsername(const AUsername: string): TWSClient;
var
  Room: TWSRoom;
  Client: TWSClient;
  RoomName: string;
begin
  Result := nil;
  for RoomName in FWSServer.FRooms.Keys do
  begin
    Room := FWSServer.FRooms[RoomName];
    for Client in Room.GetClients do
      if LowerCase(Client.Username) = LowerCase(AUsername) then
      begin
        Result := Client;
        Exit;
      end;
  end;
end;

end.
```

---

## Chat Client (JavaScript) สำหรับทดสอบ

```javascript
// chat-client.js
const ws = new WebSocket('ws://localhost:8888');

ws.onopen = function() {
    console.log('เชื่อมต่อสำเร็จ');
    
    // เข้าร่วมห้อง
    ws.send(JSON.stringify({
        type: 'join',
        room: 'general',
        content: 'MyUsername'  // ชื่อผู้ใช้
    }));
};

ws.onmessage = function(event) {
    const msg = JSON.parse(event.data);
    console.log(`[${msg.type}] ${msg.from}: ${msg.content}`);
    
    if (msg.type === 'user_list') {
        console.log(`ผู้ใช้ในห้อง (${msg.count} คน):`);
        msg.users.forEach(u => console.log(` - ${u.username}`));
    }
};

ws.onclose = function() {
    console.log('ตัดการเชื่อมต่อ');
};

// ส่งข้อความ
function sendMessage(content) {
    ws.send(JSON.stringify({
        type: 'message',
        content: content
    }));
}

// ส่ง Private Message
function sendPrivate(toUser, content) {
    ws.send(JSON.stringify({
        type: 'private',
        to: toUser,
        content: content
    }));
}
```

---

## Main Program

```pascal
program ChatApp;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, ChatServer;

var
  Chat: TChatServer;
  S: string;
begin
  Chat := TChatServer.Create(8888);
  try
    Chat.DefaultRoom := 'general';
    Chat.Start;
    
    WriteLn('Chat Server กำลังทำงาน...');
    WriteLn('กด Enter เพื่อหยุด');
    ReadLn(S);
    
    Chat.Stop;
    WriteLn('หยุดทำงาน');
  finally
    Chat.Free;
  end;
end.
```

---

## แบบฝึกหัด

### ข้อ 1 - Multi-Room Chat
ขยาย Chat Server ให้รองรับ:
- สร้างห้องใหม่
- แสดงรายการห้องทั้งหมด
- ออกจากห้อง
- Password-protected rooms

### ข้อ 2 - File Transfer
เพิ่มการส่งไฟล์ผ่าน WebSocket:
- Binary frame สำหรับข้อมูลไฟล์
- Progress indicator
- File size limit

### ข้อ 3 - Real-time Dashboard
สร้าง Dashboard ที่แสดงข้อมูล Real-time:
- System metrics (CPU, Memory)
- Active connections
- Messages per second
- Room statistics

### ข้อ 4 - Collaborative Editor
สร้าง Text Editor ที่แก้ไขร่วมกันได้:
- Operational Transformation (OT) พื้นฐาน
- Cursor tracking
- User identification

### ข้อ 5 - Game Server
สร้าง Simple Game Server:
- Player join/leave
- Game state synchronization
- Score broadcasting
