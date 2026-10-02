# ตอนที่ 91: IoT Programming กับ Pascal/Lazarus

## บทนำ: Pascal กับ Internet of Things

Pascal สามารถใช้กับ IoT ได้ผ่าน Serial Port, MQTT Protocol และ Raspberry Pi โดยตรง

## 1. Serial Port Communication

```pascal
// uSerialPort.pas - Serial Port สำหรับ Arduino/Sensors
unit uSerialPort;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Serial;

type
  TBaudRate = (br9600, br19200, br38400, br57600, br115200);
  TDataBits = (db7, db8);
  TStopBits = (sb1, sb2);
  TParity = (paNone, paEven, paOdd);

  TSerialConfig = record
    Port: string;        // '/dev/ttyUSB0' หรือ 'COM3'
    BaudRate: TBaudRate;
    DataBits: TDataBits;
    StopBits: TStopBits;
    Parity: TParity;
    TimeoutMs: Integer;
  end;

  TSerialDataEvent = procedure(const AData: TBytes) of object;

  TSerialPort = class
  private
    FHandle: TSerialHandle;
    FConfig: TSerialConfig;
    FIsOpen: Boolean;
    FReadThread: TThread;
    FOnDataReceived: TSerialDataEvent;
    FReadBuffer: TBytes;
    FReadTimeout: Integer;

    function BaudRateToInt(ABaud: TBaudRate): Integer;
    procedure StartReadThread;

  public
    constructor Create(const AConfig: TSerialConfig);
    destructor Destroy; override;

    procedure Open;
    procedure Close;

    function Write(const AData: TBytes): Integer;
    function WriteStr(const AStr: string): Integer;
    function Read(AMaxBytes: Integer; ATimeoutMs: Integer = -1): TBytes;
    function ReadLine(ATimeoutMs: Integer = 2000): string;
    function ReadUntil(const ATerminator: Byte;
      ATimeoutMs: Integer = 2000): TBytes;

    property IsOpen: Boolean read FIsOpen;
    property OnDataReceived: TSerialDataEvent
      read FOnDataReceived write FOnDataReceived;
  end;

  // Arduino Protocol Helper
  TArduinoProtocol = class
  private
    FPort: TSerialPort;

  public
    constructor Create(const APort: TSerialPort);

    // DigitalIO
    procedure SetPinMode(APin: Byte; AMode: Byte); // 0=INPUT, 1=OUTPUT
    procedure DigitalWrite(APin: Byte; AValue: Byte); // 0=LOW, 1=HIGH
    function DigitalRead(APin: Byte): Byte;

    // AnalogIO
    function AnalogRead(APin: Byte): Integer; // 0-1023
    procedure AnalogWrite(APin: Byte; AValue: Byte); // 0-255

    // Servo
    procedure SetServoAngle(APin: Byte; AAngle: Integer); // 0-180

    // Sensor Helpers
    function ReadDHT22(APin: Byte; out ATemperature, AHumidity: Double): Boolean;
    function ReadUltrasonic(ATrigPin, AEchoPin: Byte): Double; // Distance in cm
  end;

implementation

function TSerialPort.BaudRateToInt(ABaud: TBaudRate): Integer;
const
  BaudRates: array[TBaudRate] of Integer =
    (9600, 19200, 38400, 57600, 115200);
begin
  Result := BaudRates[ABaud];
end;

procedure TSerialPort.Open;
var
  Baud: Integer;
begin
  if FIsOpen then Exit;

  FHandle := SerOpen(FConfig.Port);
  if FHandle = Invalid_Handle_Value then
    raise Exception.CreateFmt('Cannot open serial port: %s', [FConfig.Port]);

  Baud := BaudRateToInt(FConfig.BaudRate);
  SerSetParams(FHandle, Baud, 8, NoneParity, 1, []);

  FIsOpen := True;

  if Assigned(FOnDataReceived) then
    StartReadThread;
end;

procedure TSerialPort.Close;
begin
  if not FIsOpen then Exit;

  if Assigned(FReadThread) then
  begin
    FReadThread.Terminate;
    FReadThread.WaitFor;
    FreeAndNil(FReadThread);
  end;

  SerClose(FHandle);
  FIsOpen := False;
end;

function TSerialPort.WriteStr(const AStr: string): Integer;
var
  Data: TBytes;
begin
  Data := TEncoding.ASCII.GetBytes(AStr);
  Result := Write(Data);
end;

function TSerialPort.Read(AMaxBytes: Integer;
  ATimeoutMs: Integer): TBytes;
var
  Buffer: array[0..255] of Byte;
  BytesRead: Integer;
  StartTime: TDateTime;
  Timeout: Integer;
begin
  SetLength(Result, 0);

  if ATimeoutMs < 0 then
    Timeout := FReadTimeout
  else
    Timeout := ATimeoutMs;

  StartTime := Now;

  while Length(Result) < AMaxBytes do
  begin
    BytesRead := SerRead(FHandle, Buffer[0], Min(AMaxBytes - Length(Result), 256));

    if BytesRead > 0 then
    begin
      var OldLen := Length(Result);
      SetLength(Result, OldLen + BytesRead);
      Move(Buffer[0], Result[OldLen], BytesRead);
    end;

    if (Timeout > 0) and
       (MilliSecondsBetween(Now, StartTime) > Timeout) then
      Break;

    if BytesRead = 0 then
      Sleep(1);
  end;
end;

function TSerialPort.ReadLine(ATimeoutMs: Integer): string;
var
  Buffer: TBytes;
  Byte_: Byte;
  StartTime: TDateTime;
begin
  Result := '';
  StartTime := Now;

  repeat
    var ReadByte := Read(1, 10);
    if Length(ReadByte) > 0 then
    begin
      Byte_ := ReadByte[0];
      if Byte_ = 10 then // LF
        Break
      else if Byte_ <> 13 then // Skip CR
        Result := Result + Chr(Byte_);
    end;
  until MilliSecondsBetween(Now, StartTime) > ATimeoutMs;
end;

// Arduino Protocol
procedure TArduinoProtocol.DigitalWrite(APin: Byte; AValue: Byte);
begin
  FPort.WriteStr(Format('D%d=%d', [APin, AValue]) + #13#10);
  Sleep(10);
end;

function TArduinoProtocol.AnalogRead(APin: Byte): Integer;
var
  Response: string;
begin
  FPort.WriteStr(Format('AR%d', [APin]) + #13#10);
  Response := FPort.ReadLine(500);
  Result := StrToIntDef(Trim(Response), -1);
end;

function TArduinoProtocol.ReadDHT22(APin: Byte;
  out ATemperature, AHumidity: Double): Boolean;
var
  Response: string;
  Parts: TStringList;
begin
  Result := False;
  FPort.WriteStr(Format('DHT22:%d', [APin]) + #13#10);
  Response := FPort.ReadLine(1000);

  // Expected: "T=25.50,H=60.20"
  Parts := TStringList.Create;
  try
    Parts.Delimiter := ',';
    Parts.DelimitedText := Response;

    if Parts.Count >= 2 then
    begin
      ATemperature := StrToFloatDef(
        Copy(Parts[0], 3, MaxInt), 0);
      AHumidity := StrToFloatDef(
        Copy(Parts[1], 3, MaxInt), 0);
      Result := True;
    end;
  finally
    Parts.Free;
  end;
end;

end.
```

## 2. MQTT Protocol Client

```pascal
// uMQTTClient.pas - MQTT Client สำหรับ IoT
unit uMQTTClient;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Sockets, BlSocket, Generics.Collections;

type
  TMQTTQoS = (qos0, qos1, qos2); // At most once, At least once, Exactly once

  TMQTTMessage = class
  public
    Topic: string;
    Payload: TBytes;
    QoS: TMQTTQoS;
    Retained: Boolean;
    Duplicate: Boolean;
    MessageId: Word;

    function PayloadAsString: string;
    function PayloadAsJSON: string;
  end;

  TMQTTMessageEvent = procedure(const AMessage: TMQTTMessage) of object;
  TMQTTConnectEvent = procedure of object;
  TMQTTDisconnectEvent = procedure(const AReason: string) of object;

  TMQTTConfig = record
    Host: string;
    Port: Word;          // Default 1883, TLS: 8883
    ClientId: string;
    Username: string;
    Password: string;
    KeepAliveSeconds: Word;
    CleanSession: Boolean;
    AutoReconnect: Boolean;
    ReconnectIntervalMs: Integer;

    // Will Message (Last Will and Testament)
    WillTopic: string;
    WillPayload: string;
    WillQoS: TMQTTQoS;
    WillRetained: Boolean;
  end;

  TSubscription = record
    Topic: string;
    QoS: TMQTTQoS;
    Callback: TMQTTMessageEvent;
  end;

  TMQTTClient = class
  private
    FConfig: TMQTTConfig;
    FSocket: TSocket;
    FConnected: Boolean;
    FPacketId: Word;
    FSubscriptions: TList<TSubscription>;
    FReadThread: TThread;
    FReconnectThread: TThread;
    FLogger: ILogger;

    FOnConnect: TMQTTConnectEvent;
    FOnDisconnect: TMQTTDisconnectEvent;
    FOnMessage: TMQTTMessageEvent;

    function NextPacketId: Word;
    function BuildConnectPacket: TBytes;
    function BuildPublishPacket(const ATopic: string;
      const APayload: TBytes; AQoS: TMQTTQoS;
      ARetained: Boolean; APacketId: Word): TBytes;
    function BuildSubscribePacket(const ATopic: string;
      AQoS: TMQTTQoS; APacketId: Word): TBytes;
    function BuildUnsubscribePacket(const ATopic: string;
      APacketId: Word): TBytes;
    function BuildDisconnectPacket: TBytes;
    function BuildPingReqPacket: TBytes;

    procedure SendPacket(const APacket: TBytes);
    procedure ProcessPacket(const APacket: TBytes);
    procedure StartReadThread;
    function MatchTopic(const APattern, ATopic: string): Boolean;

  public
    constructor Create(const AConfig: TMQTTConfig);
    destructor Destroy; override;

    procedure Connect;
    procedure Disconnect;
    procedure Reconnect;

    procedure Publish(const ATopic: string; const APayload: string;
      AQoS: TMQTTQoS = qos0; ARetained: Boolean = False); overload;
    procedure Publish(const ATopic: string; const APayload: TBytes;
      AQoS: TMQTTQoS = qos0; ARetained: Boolean = False); overload;

    procedure Subscribe(const ATopic: string;
      AQoS: TMQTTQoS = qos0;
      const ACallback: TMQTTMessageEvent = nil);
    procedure Unsubscribe(const ATopic: string);

    property Connected: Boolean read FConnected;
    property OnConnect: TMQTTConnectEvent read FOnConnect write FOnConnect;
    property OnDisconnect: TMQTTDisconnectEvent
      read FOnDisconnect write FOnDisconnect;
    property OnMessage: TMQTTMessageEvent read FOnMessage write FOnMessage;
  end;

implementation

constructor TMQTTClient.Create(const AConfig: TMQTTConfig);
begin
  inherited Create;
  FConfig := AConfig;
  FPacketId := 0;
  FSubscriptions := TList<TSubscription>.Create;
  FConnected := False;
end;

function TMQTTClient.NextPacketId: Word;
begin
  Inc(FPacketId);
  if FPacketId = 0 then Inc(FPacketId); // 0 is reserved
  Result := FPacketId;
end;

function TMQTTClient.BuildConnectPacket: TBytes;
var
  VariableHeader: TBytes;
  Payload: TBytes;
  ConnectFlags: Byte;
  ProtocolName: string;
begin
  // MQTT 3.1.1 Connect packet
  ProtocolName := 'MQTT';

  ConnectFlags := 0;
  if FConfig.CleanSession then
    ConnectFlags := ConnectFlags or $02;
  if FConfig.Username <> '' then
    ConnectFlags := ConnectFlags or $80;
  if FConfig.Password <> '' then
    ConnectFlags := ConnectFlags or $40;
  if FConfig.WillTopic <> '' then
  begin
    ConnectFlags := ConnectFlags or $04;
    ConnectFlags := ConnectFlags or
      (Ord(FConfig.WillQoS) shl 3);
    if FConfig.WillRetained then
      ConnectFlags := ConnectFlags or $20;
  end;

  // Build variable header
  SetLength(VariableHeader, 10);
  VariableHeader[0] := 0;
  VariableHeader[1] := 4; // Protocol name length
  VariableHeader[2] := Ord('M');
  VariableHeader[3] := Ord('Q');
  VariableHeader[4] := Ord('T');
  VariableHeader[5] := Ord('T');
  VariableHeader[6] := 4; // Protocol level (MQTT 3.1.1)
  VariableHeader[7] := ConnectFlags;
  VariableHeader[8] := Hi(FConfig.KeepAliveSeconds);
  VariableHeader[9] := Lo(FConfig.KeepAliveSeconds);

  // Build payload
  var PayloadParts: TList<TBytes> := TList<TBytes>.Create;
  try
    // ClientId
    var ClientIdBytes := TEncoding.UTF8.GetBytes(FConfig.ClientId);
    var ClientIdLen: TBytes;
    SetLength(ClientIdLen, 2);
    ClientIdLen[0] := Hi(Length(ClientIdBytes));
    ClientIdLen[1] := Lo(Length(ClientIdBytes));
    PayloadParts.Add(ClientIdLen);
    PayloadParts.Add(ClientIdBytes);

    // Username
    if FConfig.Username <> '' then
    begin
      var UserBytes := TEncoding.UTF8.GetBytes(FConfig.Username);
      var UserLen: TBytes;
      SetLength(UserLen, 2);
      UserLen[0] := Hi(Length(UserBytes));
      UserLen[1] := Lo(Length(UserBytes));
      PayloadParts.Add(UserLen);
      PayloadParts.Add(UserBytes);
    end;

    // Password
    if FConfig.Password <> '' then
    begin
      var PassBytes := TEncoding.UTF8.GetBytes(FConfig.Password);
      var PassLen: TBytes;
      SetLength(PassLen, 2);
      PassLen[0] := Hi(Length(PassBytes));
      PassLen[1] := Lo(Length(PassBytes));
      PayloadParts.Add(PassLen);
      PayloadParts.Add(PassBytes);
    end;

    // Calculate total payload size
    var TotalSize := 0;
    for var Part in PayloadParts do
      Inc(TotalSize, Length(Part));
    SetLength(Payload, TotalSize);
    var Pos := 0;
    for var Part in PayloadParts do
    begin
      Move(Part[0], Payload[Pos], Length(Part));
      Inc(Pos, Length(Part));
    end;
  finally
    PayloadParts.Free;
  end;

  // Build fixed header
  var RemainingLen := Length(VariableHeader) + Length(Payload);
  SetLength(Result, 2 + RemainingLen);
  Result[0] := $10; // CONNECT packet type
  Result[1] := RemainingLen; // Remaining length (simplified, no VarInt)

  Move(VariableHeader[0], Result[2], Length(VariableHeader));
  Move(Payload[0], Result[2 + Length(VariableHeader)], Length(Payload));
end;

procedure TMQTTClient.Connect;
var
  Addr: TInetSockAddr;
  ConnectPacket: TBytes;
begin
  FSocket := fpSocket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  if FSocket < 0 then
    raise Exception.Create('Cannot create socket');

  FillChar(Addr, SizeOf(Addr), 0);
  Addr.sin_family := AF_INET;
  Addr.sin_port := htons(FConfig.Port);
  Addr.sin_addr.s_addr := longword(NetAddrToStr(FConfig.Host));

  if fpConnect(FSocket, @Addr, SizeOf(Addr)) <> 0 then
    raise Exception.CreateFmt('Cannot connect to %s:%d',
      [FConfig.Host, FConfig.Port]);

  ConnectPacket := BuildConnectPacket;
  SendPacket(ConnectPacket);

  // Wait for CONNACK
  var Response: TBytes;
  SetLength(Response, 4);
  fpRecv(FSocket, Response[0], 4, 0);

  if (Response[0] = $20) and (Response[3] = 0) then
  begin
    FConnected := True;
    StartReadThread;

    if Assigned(FOnConnect) then
      FOnConnect;
  end
  else
    raise Exception.Create('MQTT Connection refused');
end;

procedure TMQTTClient.Publish(const ATopic: string;
  const APayload: string; AQoS: TMQTTQoS; ARetained: Boolean);
var
  PayloadBytes: TBytes;
begin
  PayloadBytes := TEncoding.UTF8.GetBytes(APayload);
  Publish(ATopic, PayloadBytes, AQoS, ARetained);
end;

function TMQTTClient.MatchTopic(const APattern, ATopic: string): Boolean;
var
  PatternParts, TopicParts: TArray<string>;
  I: Integer;
begin
  // MQTT wildcard matching: + (single level), # (multi level)
  if APattern = '#' then
  begin
    Result := True;
    Exit;
  end;

  PatternParts := APattern.Split(['/']);
  TopicParts := ATopic.Split(['/']);

  Result := False;
  for I := 0 to High(PatternParts) do
  begin
    if PatternParts[I] = '#' then
    begin
      Result := True;
      Exit;
    end;

    if I >= Length(TopicParts) then Exit;

    if (PatternParts[I] <> '+') and
       (PatternParts[I] <> TopicParts[I]) then
      Exit;
  end;

  Result := Length(PatternParts) = Length(TopicParts);
end;

end.
```

## 3. Raspberry Pi GPIO Control

```pascal
// uRaspberryPiGPIO.pas - GPIO สำหรับ Raspberry Pi
unit uRaspberryPiGPIO;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, BaseUnix;

type
  TGPIODirection = (gdIn, gdOut);
  TGPIOEdge = (geNone, geRising, geFalling, geBoth);
  TGPIOPull = (gpNone, gpUp, gpDown);

  TGPIOPin = class
  private
    FPin: Integer;
    FDirection: TGPIODirection;
    FValuePath: string;

    procedure Export;
    procedure Unexport;
    procedure SetDirection(ADir: TGPIODirection);

  public
    constructor Create(APin: Integer; ADir: TGPIODirection);
    destructor Destroy; override;

    function GetValue: Integer;
    procedure SetValue(AValue: Integer);
    procedure Toggle;

    property Pin: Integer read FPin;
    property Direction: TGPIODirection read FDirection;
  end;

  // WiringPi-compatible API (via sysfs)
  TRaspberryPi = class
  private
    FPins: array[0..40] of TGPIOPin;
    FPinMap: array[0..40] of Integer; // WiringPi to BCM mapping

    procedure InitPinMap;

  public
    constructor Create;
    destructor Destroy; override;

    procedure PinMode(APin: Integer; AMode: TGPIODirection);
    procedure DigitalWrite(APin: Integer; AValue: Integer);
    function DigitalRead(APin: Integer): Integer;
    procedure PWMWrite(APin: Integer; AValue: Integer); // 0-1023
    function AnalogRead(APin: Integer): Integer;

    // I2C
    function I2CSetup(ADevId: Integer): Integer;
    function I2CReadByte(AFd: Integer): Integer;
    procedure I2CWriteByte(AFd: Integer; AData: Integer);

    // SPI
    function SPISetup(AChannel, ASpeed: Integer): Integer;
    function SPIDataRW(AChannel: Integer; var AData: TBytes): Integer;
  end;

implementation

procedure TGPIOPin.Export;
var
  F: TextFile;
begin
  if not FileExists(Format('/sys/class/gpio/gpio%d', [FPin])) then
  begin
    AssignFile(F, '/sys/class/gpio/export');
    Rewrite(F);
    Write(F, IntToStr(FPin));
    CloseFile(F);
    Sleep(100); // Wait for sysfs
  end;
end;

procedure TGPIOPin.SetDirection(ADir: TGPIODirection);
var
  F: TextFile;
begin
  AssignFile(F, Format('/sys/class/gpio/gpio%d/direction', [FPin]));
  Rewrite(F);
  if ADir = gdOut then
    Write(F, 'out')
  else
    Write(F, 'in');
  CloseFile(F);
  FDirection := ADir;
end;

function TGPIOPin.GetValue: Integer;
var
  F: TextFile;
  S: string;
begin
  AssignFile(F, FValuePath);
  Reset(F);
  ReadLn(F, S);
  CloseFile(F);
  Result := StrToIntDef(Trim(S), 0);
end;

procedure TGPIOPin.SetValue(AValue: Integer);
var
  F: TextFile;
begin
  if FDirection <> gdOut then
    raise Exception.Create('Pin is not configured as OUTPUT');

  AssignFile(F, FValuePath);
  Rewrite(F);
  Write(F, IntToStr(AValue));
  CloseFile(F);
end;

procedure TGPIOPin.Toggle;
begin
  if GetValue = 0 then
    SetValue(1)
  else
    SetValue(0);
end;

end.
```

## 4. IoT Sensor Data Collection System

```pascal
// IoTGateway/uSensorManager.pas - Sensor Data Collection
unit uSensorManager;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils,
  uMQTTClient, uSerialPort;

type
  TSensorType = (stTemperature, stHumidity, stPressure,
    stLight, stMotion, stGas, stCustom);

  TSensorReading = class
  public
    SensorId: string;
    SensorType: TSensorType;
    Value: Double;
    Unit_: string;
    Timestamp: TDateTime;
    Quality: Integer; // 0-100

    function ToJSON: string;
  end;

  TSensorConfig = record
    Id: string;
    Name: string;
    SensorType: TSensorType;
    ReadIntervalMs: Integer;
    MinValue, MaxValue: Double;
    CalibrationOffset: Double;
    AlertThresholdHigh: Double;
    AlertThresholdLow: Double;
  end;

  TSensorAlert = record
    SensorId: string;
    Value: Double;
    Threshold: Double;
    IsHigh: Boolean;
    Timestamp: TDateTime;
  end;

  TAlertEvent = procedure(const AAlert: TSensorAlert) of object;

  TSensorManager = class
  private
    FSensors: TList<TSensorConfig>;
    FReadings: TObjectList<TSensorReading>;
    FMqtt: TMQTTClient;
    FSerial: TSerialPort;
    FTimers: TList<TThread>;
    FLogger: ILogger;
    FOnAlert: TAlertEvent;
    FMqttTopicPrefix: string;
    FCriticalSection: TCriticalSection;

    procedure CollectSensorData(const AConfig: TSensorConfig);
    procedure PublishReading(const AReading: TSensorReading);
    procedure CheckAlerts(const AReading: TSensorReading;
      const AConfig: TSensorConfig);
    function ReadArduinoSensor(const AConfig: TSensorConfig): Double;

  public
    constructor Create(const AMqtt: TMQTTClient;
      const ASerial: TSerialPort;
      const ATopicPrefix: string = 'sensors');
    destructor Destroy; override;

    procedure AddSensor(const AConfig: TSensorConfig);
    procedure RemoveSensor(const AId: string);
    procedure Start;
    procedure Stop;

    function GetLatestReading(const ASensorId: string): TSensorReading;
    function GetReadingHistory(const ASensorId: string;
      AStartTime, AEndTime: TDateTime): TList<TSensorReading>;
    function GetAllLatestReadings: TList<TSensorReading>;

    property OnAlert: TAlertEvent read FOnAlert write FOnAlert;
  end;

implementation

function TSensorReading.ToJSON: string;
const
  SensorTypeNames: array[TSensorType] of string =
    ('temperature', 'humidity', 'pressure',
     'light', 'motion', 'gas', 'custom');
begin
  Result := Format(
    '{"sensor_id":"%s","type":"%s","value":%.4f,' +
    '"unit":"%s","timestamp":"%s","quality":%d}',
    [SensorId, SensorTypeNames[SensorType], Value,
     Unit_,
     FormatDateTime('yyyy-mm-dd"T"hh:nn:ss', Timestamp),
     Quality]
  );
end;

procedure TSensorManager.CollectSensorData(const AConfig: TSensorConfig);
var
  Reading: TSensorReading;
  RawValue: Double;
begin
  try
    RawValue := ReadArduinoSensor(AConfig);

    Reading := TSensorReading.Create;
    Reading.SensorId := AConfig.Id;
    Reading.SensorType := AConfig.SensorType;
    Reading.Value := RawValue + AConfig.CalibrationOffset;
    Reading.Timestamp := Now;

    // Validate range
    if (Reading.Value >= AConfig.MinValue) and
       (Reading.Value <= AConfig.MaxValue) then
      Reading.Quality := 100
    else
      Reading.Quality := 0;

    case AConfig.SensorType of
      stTemperature: Reading.Unit_ := '°C';
      stHumidity: Reading.Unit_ := '%';
      stPressure: Reading.Unit_ := 'hPa';
      stLight: Reading.Unit_ := 'lux';
    end;

    // Store reading
    FCriticalSection.Acquire;
    try
      FReadings.Add(Reading);

      // Keep only last 1000 readings per sensor
      while FReadings.Count > 10000 do
        FReadings.Delete(0);
    finally
      FCriticalSection.Release;
    end;

    PublishReading(Reading);
    CheckAlerts(Reading, AConfig);

    FLogger.Debug(Format('[%s] %s: %.2f %s',
      [AConfig.Id, AConfig.Name, Reading.Value, Reading.Unit_]));

  except
    on E: Exception do
      FLogger.Error(Format('Error reading sensor %s: %s',
        [AConfig.Id, E.Message]));
  end;
end;

procedure TSensorManager.PublishReading(const AReading: TSensorReading);
var
  Topic: string;
begin
  if not Assigned(FMqtt) or not FMqtt.Connected then Exit;

  Topic := Format('%s/%s/%s',
    [FMqttTopicPrefix, AReading.SensorId,
     SensorTypeNames[AReading.SensorType]]);

  FMqtt.Publish(Topic, AReading.ToJSON, qos0);
end;

procedure TSensorManager.CheckAlerts(const AReading: TSensorReading;
  const AConfig: TSensorConfig);
var
  Alert: TSensorAlert;
begin
  Alert.SensorId := AConfig.Id;
  Alert.Value := AReading.Value;
  Alert.Timestamp := AReading.Timestamp;

  if (AConfig.AlertThresholdHigh > 0) and
     (AReading.Value > AConfig.AlertThresholdHigh) then
  begin
    Alert.Threshold := AConfig.AlertThresholdHigh;
    Alert.IsHigh := True;

    FLogger.Warning(Format('ALERT: %s value %.2f exceeds high threshold %.2f',
      [AConfig.Name, AReading.Value, AConfig.AlertThresholdHigh]));

    // Publish alert to MQTT
    FMqtt.Publish(Format('%s/%s/alert', [FMqttTopicPrefix, AConfig.Id]),
      Format('HIGH: %.2f > %.2f', [AReading.Value, AConfig.AlertThresholdHigh]));

    if Assigned(FOnAlert) then
      FOnAlert(Alert);
  end
  else if (AConfig.AlertThresholdLow > 0) and
          (AReading.Value < AConfig.AlertThresholdLow) then
  begin
    Alert.Threshold := AConfig.AlertThresholdLow;
    Alert.IsHigh := False;

    FLogger.Warning(Format('ALERT: %s value %.2f below low threshold %.2f',
      [AConfig.Name, AReading.Value, AConfig.AlertThresholdLow]));

    if Assigned(FOnAlert) then
      FOnAlert(Alert);
  end;
end;

end.
```

## 5. Complete IoT Application

```pascal
// SmartHome/SmartHomeGateway.pas - Complete IoT Application
program SmartHomeGateway;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes, fphttpserver,
  uMQTTClient, uSerialPort, uSensorManager;

var
  MqttClient: TMQTTClient;
  SerialPort: TSerialPort;
  SensorManager: TSensorManager;
  HttpServer: TFPHTTPServer;

procedure OnSensorAlert(const AAlert: TSensorAlert);
begin
  WriteLn(Format('[ALERT] Sensor %s: %.2f (threshold: %.2f)',
    [AAlert.SensorId, AAlert.Value, AAlert.Threshold]));

  // Send notification via MQTT
  MqttClient.Publish('home/alerts',
    Format('{"sensor":"%s","value":%.2f,"high":%s}',
      [AAlert.SensorId, AAlert.Value,
       BoolToStr(AAlert.IsHigh, 'true', 'false')]),
    qos1);
end;

procedure SetupSensors;
var
  Config: TSensorConfig;
begin
  // Temperature sensor
  Config.Id := 'temp1';
  Config.Name := 'Living Room Temperature';
  Config.SensorType := stTemperature;
  Config.ReadIntervalMs := 5000;
  Config.MinValue := -10;
  Config.MaxValue := 60;
  Config.CalibrationOffset := -1.5;
  Config.AlertThresholdHigh := 35;
  Config.AlertThresholdLow := 10;
  SensorManager.AddSensor(Config);

  // Humidity sensor
  Config.Id := 'hum1';
  Config.Name := 'Living Room Humidity';
  Config.SensorType := stHumidity;
  Config.ReadIntervalMs := 5000;
  Config.MinValue := 0;
  Config.MaxValue := 100;
  Config.CalibrationOffset := 0;
  Config.AlertThresholdHigh := 85;
  Config.AlertThresholdLow := 20;
  SensorManager.AddSensor(Config);
end;

procedure OnMqttMessage(const AMessage: TMQTTMessage);
begin
  // Handle incoming commands
  if AMessage.Topic = 'home/commands/lights' then
  begin
    WriteLn('Light command: ', AMessage.PayloadAsString);
    // Control lights via GPIO or serial
  end;
end;

begin
  WriteLn('Smart Home Gateway Starting...');

  // Setup MQTT
  var MqttConfig: TMQTTConfig;
  MqttConfig.Host := 'localhost';
  MqttConfig.Port := 1883;
  MqttConfig.ClientId := 'SmartHomeGateway';
  MqttConfig.KeepAliveSeconds := 60;
  MqttConfig.CleanSession := True;
  MqttConfig.WillTopic := 'home/gateway/status';
  MqttConfig.WillPayload := 'offline';

  MqttClient := TMQTTClient.Create(MqttConfig);
  MqttClient.OnMessage := @OnMqttMessage;
  MqttClient.Connect;
  MqttClient.Publish('home/gateway/status', 'online', qos1, True);
  MqttClient.Subscribe('home/commands/#');

  // Setup Serial Port (Arduino)
  var SerialConfig: TSerialConfig;
  SerialConfig.Port := '/dev/ttyUSB0';
  SerialConfig.BaudRate := br9600;
  SerialPort := TSerialPort.Create(SerialConfig);
  SerialPort.Open;

  // Setup Sensor Manager
  SensorManager := TSensorManager.Create(MqttClient, SerialPort, 'home');
  SensorManager.OnAlert := @OnSensorAlert;
  SetupSensors;
  SensorManager.Start;

  WriteLn('System running. Press Enter to stop.');
  ReadLn;

  SensorManager.Stop;
  MqttClient.Publish('home/gateway/status', 'offline', qos1, True);
  MqttClient.Disconnect;

  FreeAndNil(SensorManager);
  FreeAndNil(SerialPort);
  FreeAndNil(MqttClient);

  WriteLn('Shutdown complete.');
end.
```

## 6. สรุป IoT Programming

**Technologies ที่ใช้:**
1. **Serial Port** - เชื่อมต่อ Arduino, Sensors
2. **MQTT** - Lightweight messaging สำหรับ IoT
3. **GPIO (Raspberry Pi)** - ควบคุม Hardware โดยตรง
4. **I2C/SPI** - สื่อสารกับ IC Components

**IoT Architecture:**
```
Sensors → Arduino/MCU → Serial/WiFi → Gateway (Pascal) → MQTT Broker → Cloud/Dashboard
                                      ↓
                                 Local Processing (Alerts, Storage)
```

**Use Cases:**
- Smart Home Automation
- Industrial Monitoring
- Environmental Sensing
- Agricultural IoT
- Asset Tracking
