# ตอนที่ 85: Event Sourcing กับ Pascal/Lazarus

## บทนำ: Event Sourcing คืออะไร?

Event Sourcing คือ Pattern ที่บันทึก State ของระบบในรูปแบบ Sequence of Events แทนที่จะบันทึก Current State โดยตรง

```
Traditional: State บันทึกโดยตรง
┌────────────────┐
│   Account      │
│   Balance: 500 │  ← เห็นแค่ปัจจุบัน
└────────────────┘

Event Sourcing: บันทึกทุก Event
┌──────────────────────────────────────────┐
│  Event 1: AccountOpened { balance: 0 }   │
│  Event 2: MoneyDeposited { amount: 1000 }│
│  Event 3: MoneyWithdrawn { amount: 300 } │
│  Event 4: MoneyWithdrawn { amount: 200 } │
│  ─────────────────────────────────────── │
│  Current Balance = 0+1000-300-200 = 500  │
└──────────────────────────────────────────┘
```

## 1. Event Store

```pascal
// EventSourcing/uEventStore.pas
unit uEventStore;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, fpjson, sqldb;

type
  TStoredEvent = class
  private
    FId: Int64;
    FStreamId: string;
    FEventType: string;
    FPayload: string;
    FMetadata: string;
    FVersion: Integer;
    FOccurredAt: TDateTime;
    FCreatedAt: TDateTime;

  public
    property Id: Int64 read FId write FId;
    property StreamId: string read FStreamId write FStreamId;
    property EventType: string read FEventType write FEventType;
    property Payload: string read FPayload write FPayload;
    property Metadata: string read FMetadata write FMetadata;
    property Version: Integer read FVersion write FVersion;
    property OccurredAt: TDateTime read FOccurredAt write FOccurredAt;
    property CreatedAt: TDateTime read FCreatedAt write FCreatedAt;
  end;

  TEventStream = class
  private
    FStreamId: string;
    FVersion: Integer;
    FEvents: TObjectList<TStoredEvent>;

  public
    constructor Create(const AStreamId: string; AVersion: Integer = 0);
    destructor Destroy; override;

    procedure AddEvent(const AEvent: TStoredEvent);

    property StreamId: string read FStreamId;
    property Version: Integer read FVersion;
    property Events: TObjectList<TStoredEvent> read FEvents;
  end;

  IEventStore = interface
    // Append Events to Stream
    procedure AppendToStream(const AStreamId: string;
      const AEvents: TList<TStoredEvent>;
      AExpectedVersion: Integer = -1); // -1 = any version

    // Load Events from Stream
    function LoadStream(const AStreamId: string;
      AFromVersion: Integer = 0;
      AToVersion: Integer = MaxInt): TEventStream;

    // Subscribe to Events
    procedure Subscribe(const AEventType: string;
      const AHandler: TProc<TStoredEvent>);
    procedure SubscribeAll(const AHandler: TProc<TStoredEvent>);

    // Read All Streams (for Projection)
    function GetAllStreams(const AFromPosition: Int64 = 0): TList<TStoredEvent>;
    function GetEventsByType(const AEventType: string): TList<TStoredEvent>;
  end;

  // PostgreSQL-based Event Store
  TPostgresEventStore = class(TInterfacedObject, IEventStore)
  private
    FConnection: TSQLConnection;
    FSubscribers: TDictionary<string, TList<TProc<TStoredEvent>>>;
    FAllSubscribers: TList<TProc<TStoredEvent>>;
    FCriticalSection: TCriticalSection;

    procedure EnsureTable;
    procedure NotifySubscribers(const AEvent: TStoredEvent);

  public
    constructor Create(const AConnectionString: string);
    destructor Destroy; override;

    procedure AppendToStream(const AStreamId: string;
      const AEvents: TList<TStoredEvent>;
      AExpectedVersion: Integer);
    function LoadStream(const AStreamId: string;
      AFromVersion, AToVersion: Integer): TEventStream;
    procedure Subscribe(const AEventType: string;
      const AHandler: TProc<TStoredEvent>);
    procedure SubscribeAll(const AHandler: TProc<TStoredEvent>);
    function GetAllStreams(const AFromPosition: Int64): TList<TStoredEvent>;
    function GetEventsByType(const AEventType: string): TList<TStoredEvent>;
  end;

implementation

constructor TEventStream.Create(const AStreamId: string; AVersion: Integer);
begin
  inherited Create;
  FStreamId := AStreamId;
  FVersion := AVersion;
  FEvents := TObjectList<TStoredEvent>.Create(True);
end;

destructor TEventStream.Destroy;
begin
  FEvents.Free;
  inherited Destroy;
end;

constructor TPostgresEventStore.Create(const AConnectionString: string);
begin
  inherited Create;
  FConnection := TPostgresConnection.Create(nil);
  // Parse connection string and connect
  FSubscribers := TDictionary<string, TList<TProc<TStoredEvent>>>.Create;
  FAllSubscribers := TList<TProc<TStoredEvent>>.Create;
  FCriticalSection := TCriticalSection.Create;
  EnsureTable;
end;

destructor TPostgresEventStore.Destroy;
var
  SubList: TList<TProc<TStoredEvent>>;
begin
  FCriticalSection.Free;
  FAllSubscribers.Free;
  for SubList in FSubscribers.Values do SubList.Free;
  FSubscribers.Free;
  FConnection.Free;
  inherited Destroy;
end;

procedure TPostgresEventStore.EnsureTable;
var
  Query: TSQLQuery;
begin
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FConnection;
    Query.SQL.Text :=
      'CREATE TABLE IF NOT EXISTS events (' +
      '  id BIGSERIAL PRIMARY KEY,' +
      '  stream_id VARCHAR(255) NOT NULL,' +
      '  event_type VARCHAR(255) NOT NULL,' +
      '  payload JSONB NOT NULL,' +
      '  metadata JSONB,' +
      '  version INTEGER NOT NULL,' +
      '  occurred_at TIMESTAMPTZ NOT NULL,' +
      '  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),' +
      '  UNIQUE(stream_id, version)' +
      ')';
    Query.ExecSQL;

    // Create Indexes
    Query.SQL.Text := 'CREATE INDEX IF NOT EXISTS idx_events_stream_id ON events(stream_id)';
    Query.ExecSQL;
    Query.SQL.Text := 'CREATE INDEX IF NOT EXISTS idx_events_event_type ON events(event_type)';
    Query.ExecSQL;
    Query.SQL.Text := 'CREATE INDEX IF NOT EXISTS idx_events_occurred_at ON events(occurred_at)';
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

procedure TPostgresEventStore.AppendToStream(const AStreamId: string;
  const AEvents: TList<TStoredEvent>; AExpectedVersion: Integer);
var
  Query: TSQLQuery;
  CurrentVersion: Integer;
  Event: TStoredEvent;
begin
  if AEvents.Count = 0 then Exit;

  FCriticalSection.Acquire;
  try
    // Get current version
    Query := TSQLQuery.Create(nil);
    try
      Query.DataBase := FConnection;

      if AExpectedVersion >= 0 then
      begin
        // Optimistic Concurrency Check
        Query.SQL.Text :=
          'SELECT COALESCE(MAX(version), -1) FROM events WHERE stream_id = :stream_id';
        Query.ParamByName('stream_id').AsString := AStreamId;
        Query.Open;
        CurrentVersion := Query.Fields[0].AsInteger;

        if CurrentVersion <> AExpectedVersion then
          raise EOptimisticConcurrencyException.CreateFmt(
            'Concurrency conflict on stream %s. Expected: %d, Actual: %d',
            [AStreamId, AExpectedVersion, CurrentVersion]);

        Query.Close;
      end
      else
      begin
        Query.SQL.Text :=
          'SELECT COALESCE(MAX(version), -1) FROM events WHERE stream_id = :stream_id';
        Query.ParamByName('stream_id').AsString := AStreamId;
        Query.Open;
        CurrentVersion := Query.Fields[0].AsInteger;
        Query.Close;
      end;

      // Insert Events
      for Event in AEvents do
      begin
        Inc(CurrentVersion);
        Event.Version := CurrentVersion;
        Event.CreatedAt := Now;

        Query.SQL.Text :=
          'INSERT INTO events (stream_id, event_type, payload, metadata, ' +
          '  version, occurred_at, created_at) ' +
          'VALUES (:stream_id, :event_type, :payload::jsonb, :metadata::jsonb, ' +
          '  :version, :occurred_at, :created_at) ' +
          'RETURNING id';

        Query.ParamByName('stream_id').AsString := AStreamId;
        Query.ParamByName('event_type').AsString := Event.EventType;
        Query.ParamByName('payload').AsString := Event.Payload;
        Query.ParamByName('metadata').AsString :=
          IfThen(Event.Metadata <> '', Event.Metadata, '{}');
        Query.ParamByName('version').AsInteger := Event.Version;
        Query.ParamByName('occurred_at').AsDateTime := Event.OccurredAt;
        Query.ParamByName('created_at').AsDateTime := Event.CreatedAt;

        Query.Open;
        Event.Id := Query.Fields[0].AsInt64;
        Query.Close;

        NotifySubscribers(Event);
      end;
    finally
      Query.Free;
    end;
  finally
    FCriticalSection.Release;
  end;
end;

function TPostgresEventStore.LoadStream(const AStreamId: string;
  AFromVersion, AToVersion: Integer): TEventStream;
var
  Query: TSQLQuery;
  Event: TStoredEvent;
  MaxVer: Integer;
begin
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FConnection;
    Query.SQL.Text :=
      'SELECT id, stream_id, event_type, payload, metadata, ' +
      '       version, occurred_at, created_at ' +
      'FROM events ' +
      'WHERE stream_id = :stream_id ' +
      '  AND version >= :from_version ' +
      '  AND version <= :to_version ' +
      'ORDER BY version ASC';

    Query.ParamByName('stream_id').AsString := AStreamId;
    Query.ParamByName('from_version').AsInteger := AFromVersion;
    Query.ParamByName('to_version').AsInteger :=
      IfThen(AToVersion = MaxInt, 2147483647, AToVersion);
    Query.Open;

    MaxVer := 0;
    Result := TEventStream.Create(AStreamId);

    while not Query.EOF do
    begin
      Event := TStoredEvent.Create;
      Event.Id := Query.FieldByName('id').AsInt64;
      Event.StreamId := Query.FieldByName('stream_id').AsString;
      Event.EventType := Query.FieldByName('event_type').AsString;
      Event.Payload := Query.FieldByName('payload').AsString;
      Event.Metadata := Query.FieldByName('metadata').AsString;
      Event.Version := Query.FieldByName('version').AsInteger;
      Event.OccurredAt := Query.FieldByName('occurred_at').AsDateTime;
      Event.CreatedAt := Query.FieldByName('created_at').AsDateTime;

      Result.AddEvent(Event);
      if Event.Version > MaxVer then MaxVer := Event.Version;
      Query.Next;
    end;
  finally
    Query.Free;
  end;
end;

end.
```

## 2. Event-Sourced Aggregate

```pascal
// EventSourcing/uEventSourcedAggregate.pas
unit uEventSourcedAggregate;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, uEventStore, fpjson;

type
  IDomainEvent = interface
    function GetEventType: string;
    function GetAggregateId: string;
    function GetVersion: Integer;
    function GetOccurredAt: TDateTime;
    function ToJson: TJSONObject;
  end;

  TEventSourcedAggregate = class abstract
  private
    FAggregateId: string;
    FVersion: Integer;
    FUncommittedEvents: TList<IDomainEvent>;

    procedure Apply(const AEvent: IDomainEvent; AIsNew: Boolean);

  protected
    // Subclasses implement this to handle each event type
    procedure When(const AEvent: IDomainEvent); virtual; abstract;

    procedure RaiseEvent(const AEvent: IDomainEvent);

  public
    constructor Create(const AAggregateId: string);
    destructor Destroy; override;

    // Load from event history
    procedure LoadFromHistory(const AEvents: TList<IDomainEvent>);

    function GetUncommittedEvents: TList<IDomainEvent>;
    procedure ClearUncommittedEvents;

    function ToStoredEvents: TList<TStoredEvent>;

    property AggregateId: string read FAggregateId;
    property Version: Integer read FVersion;
  end;

implementation

constructor TEventSourcedAggregate.Create(const AAggregateId: string);
begin
  inherited Create;
  FAggregateId := AAggregateId;
  FVersion := -1;
  FUncommittedEvents := TList<IDomainEvent>.Create;
end;

destructor TEventSourcedAggregate.Destroy;
begin
  FUncommittedEvents.Free;
  inherited Destroy;
end;

procedure TEventSourcedAggregate.RaiseEvent(const AEvent: IDomainEvent);
begin
  Apply(AEvent, True);
end;

procedure TEventSourcedAggregate.Apply(const AEvent: IDomainEvent;
  AIsNew: Boolean);
begin
  When(AEvent);
  Inc(FVersion);
  if AIsNew then
    FUncommittedEvents.Add(AEvent);
end;

procedure TEventSourcedAggregate.LoadFromHistory(
  const AEvents: TList<IDomainEvent>);
var
  Event: IDomainEvent;
begin
  for Event in AEvents do
    Apply(Event, False);
end;

function TEventSourcedAggregate.GetUncommittedEvents: TList<IDomainEvent>;
begin
  Result := FUncommittedEvents;
end;

procedure TEventSourcedAggregate.ClearUncommittedEvents;
begin
  FUncommittedEvents.Clear;
end;

function TEventSourcedAggregate.ToStoredEvents: TList<TStoredEvent>;
var
  Event: IDomainEvent;
  StoredEvent: TStoredEvent;
  EventJson: TJSONObject;
begin
  Result := TList<TStoredEvent>.Create;
  for Event in FUncommittedEvents do
  begin
    StoredEvent := TStoredEvent.Create;
    StoredEvent.StreamId := Format('%s-%s',
      [ClassName.Replace('T', '').Replace('Aggregate', ''), FAggregateId]);
    StoredEvent.EventType := Event.GetEventType;
    EventJson := Event.ToJson;
    try
      StoredEvent.Payload := EventJson.AsJSON;
    finally
      EventJson.Free;
    end;
    StoredEvent.OccurredAt := Event.GetOccurredAt;
    Result.Add(StoredEvent);
  end;
end;

end.
```

## 3. Bank Account Example (Event Sourced)

```pascal
// Examples/BankAccount/uBankAccountAggregate.pas
unit uBankAccountAggregate;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson,
  uEventSourcedAggregate;

type
  // Events
  TAccountOpenedEvent = class(TInterfacedObject, IDomainEvent)
  public
    AccountId: string;
    OwnerId: Integer;
    OwnerName: string;
    InitialBalance: Currency;
    Currency: string;
    OccurredAt: TDateTime;

    constructor Create(const AAccountId: string; AOwnerId: Integer;
      const AOwnerName: string; AInitialBalance: Currency;
      const ACurrency: string);

    function GetEventType: string;
    function GetAggregateId: string;
    function GetVersion: Integer;
    function GetOccurredAt: TDateTime;
    function ToJson: TJSONObject;
  end;

  TMoneyDepositedEvent = class(TInterfacedObject, IDomainEvent)
  public
    AccountId: string;
    Amount: Currency;
    Description: string;
    ReferenceId: string;
    OccurredAt: TDateTime;

    constructor Create(const AAccountId: string; AAmount: Currency;
      const ADescription, AReferenceId: string);

    function GetEventType: string;
    function GetAggregateId: string;
    function GetVersion: Integer;
    function GetOccurredAt: TDateTime;
    function ToJson: TJSONObject;
  end;

  TMoneyWithdrawnEvent = class(TInterfacedObject, IDomainEvent)
  public
    AccountId: string;
    Amount: Currency;
    Description: string;
    ReferenceId: string;
    OccurredAt: TDateTime;

    constructor Create(const AAccountId: string; AAmount: Currency;
      const ADescription, AReferenceId: string);

    function GetEventType: string;
    function GetAggregateId: string;
    function GetVersion: Integer;
    function GetOccurredAt: TDateTime;
    function ToJson: TJSONObject;
  end;

  TAccountClosedEvent = class(TInterfacedObject, IDomainEvent)
  public
    AccountId: string;
    Reason: string;
    OccurredAt: TDateTime;

    constructor Create(const AAccountId, AReason: string);

    function GetEventType: string;
    function GetAggregateId: string;
    function GetVersion: Integer;
    function GetOccurredAt: TDateTime;
    function ToJson: TJSONObject;
  end;

  TTransferInitiatedEvent = class(TInterfacedObject, IDomainEvent)
  public
    AccountId: string;
    TargetAccountId: string;
    Amount: Currency;
    TransferId: string;
    OccurredAt: TDateTime;

    constructor Create(const AAccountId, ATargetAccountId: string;
      AAmount: Currency; const ATransferId: string);

    function GetEventType: string;
    function GetAggregateId: string;
    function GetVersion: Integer;
    function GetOccurredAt: TDateTime;
    function ToJson: TJSONObject;
  end;

  // Bank Account Aggregate
  TBankAccountAggregate = class(TEventSourcedAggregate)
  private
    FAccountId: string;
    FOwnerId: Integer;
    FOwnerName: string;
    FBalance: Currency;
    FCurrency: string;
    FIsActive: Boolean;
    FDailyWithdrawalLimit: Currency;
    FDailyWithdrawn: Currency;
    FLastTransactionDate: TDate;

    procedure ResetDailyLimitIfNewDay;

  protected
    procedure When(const AEvent: IDomainEvent); override;

    // Event Handlers
    procedure OnAccountOpened(const AEvent: TAccountOpenedEvent);
    procedure OnMoneyDeposited(const AEvent: TMoneyDepositedEvent);
    procedure OnMoneyWithdrawn(const AEvent: TMoneyWithdrawnEvent);
    procedure OnAccountClosed(const AEvent: TAccountClosedEvent);
    procedure OnTransferInitiated(const AEvent: TTransferInitiatedEvent);

  public
    // Factory Method
    class function Open(const AAccountId: string; AOwnerId: Integer;
      const AOwnerName: string; AInitialBalance: Currency;
      const ACurrency: string): TBankAccountAggregate;

    // Business Methods
    procedure Deposit(AAmount: Currency; const ADescription: string;
      const AReferenceId: string = '');
    procedure Withdraw(AAmount: Currency; const ADescription: string;
      const AReferenceId: string = '');
    procedure Transfer(const ATargetAccountId: string;
      AAmount: Currency; const ATransferId: string);
    procedure Close(const AReason: string);

    property AccountId: string read FAccountId;
    property OwnerId: Integer read FOwnerId;
    property Balance: Currency read FBalance;
    property Currency: string read FCurrency;
    property IsActive: Boolean read FIsActive;
  end;

implementation

// TAccountOpenedEvent
constructor TAccountOpenedEvent.Create(const AAccountId: string;
  AOwnerId: Integer; const AOwnerName: string;
  AInitialBalance: Currency; const ACurrency: string);
begin
  inherited Create;
  AccountId := AAccountId;
  OwnerId := AOwnerId;
  OwnerName := AOwnerName;
  InitialBalance := AInitialBalance;
  Currency := ACurrency;
  OccurredAt := Now;
end;

function TAccountOpenedEvent.GetEventType: string;
begin
  Result := 'account.opened';
end;

function TAccountOpenedEvent.ToJson: TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('accountId', AccountId);
  Result.Add('ownerId', OwnerId);
  Result.Add('ownerName', OwnerName);
  Result.Add('initialBalance', InitialBalance);
  Result.Add('currency', Currency);
  Result.Add('occurredAt', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', OccurredAt));
end;

// TMoneyDepositedEvent
constructor TMoneyDepositedEvent.Create(const AAccountId: string;
  AAmount: Currency; const ADescription, AReferenceId: string);
begin
  inherited Create;
  AccountId := AAccountId;
  Amount := AAmount;
  Description := ADescription;
  ReferenceId := AReferenceId;
  OccurredAt := Now;
end;

function TMoneyDepositedEvent.GetEventType: string;
begin
  Result := 'account.money_deposited';
end;

function TMoneyDepositedEvent.ToJson: TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('accountId', AccountId);
  Result.Add('amount', Amount);
  Result.Add('description', Description);
  Result.Add('referenceId', ReferenceId);
  Result.Add('occurredAt', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', OccurredAt));
end;

// TBankAccountAggregate
class function TBankAccountAggregate.Open(const AAccountId: string;
  AOwnerId: Integer; const AOwnerName: string;
  AInitialBalance: Currency; const ACurrency: string): TBankAccountAggregate;
begin
  if AInitialBalance < 0 then
    raise EDomainException.Create('ยอดเงินเริ่มต้นต้องไม่ติดลบ');

  Result := TBankAccountAggregate.Create(AAccountId);
  Result.RaiseEvent(
    TAccountOpenedEvent.Create(AAccountId, AOwnerId, AOwnerName,
      AInitialBalance, ACurrency)
  );
end;

procedure TBankAccountAggregate.When(const AEvent: IDomainEvent);
begin
  if AEvent is TAccountOpenedEvent then
    OnAccountOpened(TAccountOpenedEvent(AEvent))
  else if AEvent is TMoneyDepositedEvent then
    OnMoneyDeposited(TMoneyDepositedEvent(AEvent))
  else if AEvent is TMoneyWithdrawnEvent then
    OnMoneyWithdrawn(TMoneyWithdrawnEvent(AEvent))
  else if AEvent is TAccountClosedEvent then
    OnAccountClosed(TAccountClosedEvent(AEvent))
  else if AEvent is TTransferInitiatedEvent then
    OnTransferInitiated(TTransferInitiatedEvent(AEvent));
end;

procedure TBankAccountAggregate.OnAccountOpened(
  const AEvent: TAccountOpenedEvent);
begin
  FAccountId := AEvent.AccountId;
  FOwnerId := AEvent.OwnerId;
  FOwnerName := AEvent.OwnerName;
  FBalance := AEvent.InitialBalance;
  FCurrency := AEvent.Currency;
  FIsActive := True;
  FDailyWithdrawalLimit := 50000; // Default 50,000
  FDailyWithdrawn := 0;
  FLastTransactionDate := Date;
end;

procedure TBankAccountAggregate.OnMoneyDeposited(
  const AEvent: TMoneyDepositedEvent);
begin
  FBalance := FBalance + AEvent.Amount;
end;

procedure TBankAccountAggregate.OnMoneyWithdrawn(
  const AEvent: TMoneyWithdrawnEvent);
begin
  FBalance := FBalance - AEvent.Amount;
  ResetDailyLimitIfNewDay;
  FDailyWithdrawn := FDailyWithdrawn + AEvent.Amount;
  FLastTransactionDate := Date;
end;

procedure TBankAccountAggregate.OnAccountClosed(
  const AEvent: TAccountClosedEvent);
begin
  FIsActive := False;
end;

procedure TBankAccountAggregate.Deposit(AAmount: Currency;
  const ADescription: string; const AReferenceId: string);
begin
  if not FIsActive then
    raise EDomainException.Create('บัญชีนี้ปิดแล้ว ไม่สามารถทำธุรกรรมได้');
  if AAmount <= 0 then
    raise EDomainException.Create('จำนวนเงินฝากต้องมากกว่า 0');
  if AAmount > 10000000 then
    raise EDomainException.Create('จำนวนเงินฝากต้องไม่เกิน 10,000,000 บาท/ครั้ง');

  RaiseEvent(TMoneyDepositedEvent.Create(FAccountId, AAmount,
    ADescription, AReferenceId));
end;

procedure TBankAccountAggregate.Withdraw(AAmount: Currency;
  const ADescription: string; const AReferenceId: string);
begin
  if not FIsActive then
    raise EDomainException.Create('บัญชีนี้ปิดแล้ว');
  if AAmount <= 0 then
    raise EDomainException.Create('จำนวนเงินถอนต้องมากกว่า 0');
  if AAmount > FBalance then
    raise EDomainException.Create('ยอดเงินในบัญชีไม่เพียงพอ');

  ResetDailyLimitIfNewDay;
  if (FDailyWithdrawn + AAmount) > FDailyWithdrawalLimit then
    raise EDomainException.CreateFmt(
      'เกินวงเงินถอนรายวัน (%.2f บาท)', [FDailyWithdrawalLimit]);

  RaiseEvent(TMoneyWithdrawnEvent.Create(FAccountId, AAmount,
    ADescription, AReferenceId));
end;

procedure TBankAccountAggregate.Transfer(const ATargetAccountId: string;
  AAmount: Currency; const ATransferId: string);
begin
  if not FIsActive then
    raise EDomainException.Create('บัญชีนี้ปิดแล้ว');
  if AAmount <= 0 then
    raise EDomainException.Create('จำนวนเงินโอนต้องมากกว่า 0');
  if AAmount > FBalance then
    raise EDomainException.Create('ยอดเงินในบัญชีไม่เพียงพอสำหรับการโอน');
  if ATargetAccountId = FAccountId then
    raise EDomainException.Create('ไม่สามารถโอนเงินไปยังบัญชีตัวเองได้');

  RaiseEvent(TTransferInitiatedEvent.Create(FAccountId, ATargetAccountId,
    AAmount, ATransferId));
  // Deduct from source account
  RaiseEvent(TMoneyWithdrawnEvent.Create(FAccountId, AAmount,
    Format('โอนเงินไปบัญชี %s', [ATargetAccountId]), ATransferId));
end;

procedure TBankAccountAggregate.Close(const AReason: string);
begin
  if not FIsActive then
    raise EDomainException.Create('บัญชีปิดอยู่แล้ว');
  if FBalance > 0 then
    raise EDomainException.Create(
      'ต้องถอนเงินทั้งหมดก่อนปิดบัญชี');

  RaiseEvent(TAccountClosedEvent.Create(FAccountId, AReason));
end;

procedure TBankAccountAggregate.ResetDailyLimitIfNewDay;
begin
  if Date > FLastTransactionDate then
    FDailyWithdrawn := 0;
end;

end.
```

## 4. Snapshots

```pascal
// EventSourcing/uSnapshot.pas - Snapshot Support
unit uSnapshot;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, sqldb;

type
  TSnapshot = class
  public
    AggregateId: string;
    AggregateType: string;
    Version: Integer;
    State: string; // JSON serialized state
    CreatedAt: TDateTime;
  end;

  ISnapshotStore = interface
    procedure SaveSnapshot(const ASnapshot: TSnapshot);
    function GetSnapshot(const AAggregateId, AAggregateType: string): TSnapshot;
    function SnapshotExists(const AAggregateId, AAggregateType: string): Boolean;
  end;

  TSnapshotStore = class(TInterfacedObject, ISnapshotStore)
  private
    FConnection: TSQLConnection;
  public
    constructor Create(const AConnection: TSQLConnection);

    procedure SaveSnapshot(const ASnapshot: TSnapshot);
    function GetSnapshot(const AAggregateId, AAggregateType: string): TSnapshot;
    function SnapshotExists(const AAggregateId, AAggregateType: string): Boolean;
  end;

  // Snapshot-aware Repository
  TSnapshotRepository<T: TEventSourcedAggregate> = class
  private
    FEventStore: IEventStore;
    FSnapshotStore: ISnapshotStore;
    FSnapshotFrequency: Integer; // Take snapshot every N events
    FSerializer: TEventSerializer;

    function CreateFromSnapshot(const ASnapshot: TSnapshot): T;
    procedure TakeSnapshotIfNeeded(const AAggregate: T);

  public
    constructor Create(
      const AEventStore: IEventStore;
      const ASnapshotStore: ISnapshotStore;
      ASnapshotFrequency: Integer = 50);

    function GetById(const AId: string): T;
    procedure Save(const AAggregate: T);
  end;

implementation

// Snapshot Store
procedure TSnapshotStore.SaveSnapshot(const ASnapshot: TSnapshot);
var
  Query: TSQLQuery;
begin
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FConnection;
    Query.SQL.Text :=
      'INSERT INTO snapshots (aggregate_id, aggregate_type, version, state, created_at) ' +
      'VALUES (:agg_id, :agg_type, :version, :state::jsonb, :created_at) ' +
      'ON CONFLICT (aggregate_id, aggregate_type) DO UPDATE SET ' +
      '  version = :version, state = :state::jsonb, created_at = :created_at ' +
      'WHERE snapshots.version < :version';

    Query.ParamByName('agg_id').AsString := ASnapshot.AggregateId;
    Query.ParamByName('agg_type').AsString := ASnapshot.AggregateType;
    Query.ParamByName('version').AsInteger := ASnapshot.Version;
    Query.ParamByName('state').AsString := ASnapshot.State;
    Query.ParamByName('created_at').AsDateTime := ASnapshot.CreatedAt;
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

// Snapshot Repository
function TSnapshotRepository<T>.GetById(const AId: string): T;
var
  Snapshot: TSnapshot;
  Stream: TEventStream;
  Events: TList<IDomainEvent>;
  StoredEvent: TStoredEvent;
  FromVersion: Integer;
begin
  // Try to load from snapshot first
  Snapshot := FSnapshotStore.GetSnapshot(AId, T.ClassName);

  if Assigned(Snapshot) then
  begin
    // Restore from snapshot
    Result := CreateFromSnapshot(Snapshot);
    FromVersion := Snapshot.Version + 1;
  end
  else
  begin
    // Create fresh aggregate
    Result := T.Create(AId);
    FromVersion := 0;
  end;

  // Load remaining events after snapshot
  Stream := FEventStore.LoadStream(
    Format('%s-%s', [T.ClassName, AId]), FromVersion);
  try
    if Stream.Events.Count > 0 then
    begin
      Events := TList<IDomainEvent>.Create;
      try
        for StoredEvent in Stream.Events do
          Events.Add(FSerializer.Deserialize(StoredEvent));
        Result.LoadFromHistory(Events);
      finally
        Events.Free;
      end;
    end;
  finally
    Stream.Free;
  end;
end;

procedure TSnapshotRepository<T>.Save(const AAggregate: T);
var
  Events: TList<TStoredEvent>;
begin
  Events := AAggregate.ToStoredEvents;
  try
    FEventStore.AppendToStream(
      Format('%s-%s', [T.ClassName, AAggregate.AggregateId]),
      Events,
      AAggregate.Version - Events.Count // Expected version before new events
    );

    // Take snapshot if needed
    TakeSnapshotIfNeeded(AAggregate);

    AAggregate.ClearUncommittedEvents;
  finally
    Events.Free;
  end;
end;

procedure TSnapshotRepository<T>.TakeSnapshotIfNeeded(const AAggregate: T);
var
  Snapshot: TSnapshot;
begin
  // Take snapshot every N events
  if (AAggregate.Version mod FSnapshotFrequency) = 0 then
  begin
    Snapshot := TSnapshot.Create;
    try
      Snapshot.AggregateId := AAggregate.AggregateId;
      Snapshot.AggregateType := AAggregate.ClassName;
      Snapshot.Version := AAggregate.Version;
      Snapshot.State := FSerializer.SerializeState(AAggregate);
      Snapshot.CreatedAt := Now;
      FSnapshotStore.SaveSnapshot(Snapshot);
    finally
      Snapshot.Free;
    end;
  end;
end;

end.
```

## 5. Projections with Event Sourcing

```pascal
// EventSourcing/Projections/uAccountBalanceProjection.pas
unit uAccountBalanceProjection;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, sqldb,
  uEventStore, uBankAccountEvents;

type
  TAccountBalance = class
  public
    AccountId: string;
    OwnerName: string;
    Balance: Currency;
    Currency: string;
    IsActive: Boolean;
    TotalDeposited: Currency;
    TotalWithdrawn: Currency;
    TransactionCount: Integer;
    LastTransactionAt: TDateTime;
  end;

  TAccountTransactionHistory = class
  public
    Items: TList<TTransactionItem>;
    TotalCount: Integer;
    constructor Create;
    destructor Destroy; override;
  end;

  TTransactionItem = class
  public
    EventId: Int64;
    TransactionType: string; // deposit, withdrawal, transfer
    Amount: Currency;
    BalanceAfter: Currency;
    Description: string;
    ReferenceId: string;
    OccurredAt: TDateTime;
  end;

  // Projection Handler
  TAccountBalanceProjection = class
  private
    FWriteDb: TSQLConnection;
    FReadDb: TSQLConnection;

    procedure EnsureTables;
    procedure UpdateBalance(const AAccountId: string;
      ADelta: Currency; const ATransactionType, ADescription: string;
      const AReferenceId: string; AOccurredAt: TDateTime);

  public
    constructor Create(const AWriteDb, AReadDb: TSQLConnection);

    // Event Handlers
    procedure OnAccountOpened(const AEvent: TStoredEvent);
    procedure OnMoneyDeposited(const AEvent: TStoredEvent);
    procedure OnMoneyWithdrawn(const AEvent: TStoredEvent);
    procedure OnAccountClosed(const AEvent: TStoredEvent);
    procedure OnTransferInitiated(const AEvent: TStoredEvent);

    // Query Methods
    function GetBalance(const AAccountId: string): TAccountBalance;
    function GetTransactionHistory(const AAccountId: string;
      APage, APageSize: Integer): TAccountTransactionHistory;
    function GetAllAccounts: TList<TAccountBalance>;
  end;

  // Projection Runner - Replays Events
  TProjectionRunner = class
  private
    FEventStore: IEventStore;
    FProjections: TList<TObject>;
    FLastPosition: Int64;
    FRunning: Boolean;
    FThread: TThread;

    procedure RunCatchUp;
    procedure DispatchEvent(const AStoredEvent: TStoredEvent);

  public
    constructor Create(const AEventStore: IEventStore);
    destructor Destroy; override;

    procedure AddProjection(const AProjection: TObject);
    procedure Start;
    procedure Stop;

    // Reset and replay all events
    procedure Reset;
    procedure Replay;
  end;

implementation

constructor TAccountBalance.Create;
begin
  inherited Create;
  Items := TList<TTransactionItem>.Create;
end;

destructor TAccountTransactionHistory.Destroy;
var
  Item: TTransactionItem;
begin
  for Item in Items do Item.Free;
  Items.Free;
  inherited Destroy;
end;

constructor TAccountBalanceProjection.Create(
  const AWriteDb, AReadDb: TSQLConnection);
begin
  inherited Create;
  FWriteDb := AWriteDb;
  FReadDb := AReadDb;
  EnsureTables;
end;

procedure TAccountBalanceProjection.EnsureTables;
var
  Query: TSQLQuery;
begin
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FWriteDb;

    Query.SQL.Text :=
      'CREATE TABLE IF NOT EXISTS account_balances (' +
      '  account_id VARCHAR(100) PRIMARY KEY,' +
      '  owner_name VARCHAR(200),' +
      '  balance DECIMAL(18,2) DEFAULT 0,' +
      '  currency VARCHAR(10),' +
      '  is_active BOOLEAN DEFAULT TRUE,' +
      '  total_deposited DECIMAL(18,2) DEFAULT 0,' +
      '  total_withdrawn DECIMAL(18,2) DEFAULT 0,' +
      '  transaction_count INTEGER DEFAULT 0,' +
      '  last_transaction_at TIMESTAMPTZ,' +
      '  updated_at TIMESTAMPTZ DEFAULT NOW()' +
      ')';
    Query.ExecSQL;

    Query.SQL.Text :=
      'CREATE TABLE IF NOT EXISTS account_transactions (' +
      '  id BIGSERIAL PRIMARY KEY,' +
      '  event_id BIGINT UNIQUE,' +
      '  account_id VARCHAR(100),' +
      '  transaction_type VARCHAR(50),' +
      '  amount DECIMAL(18,2),' +
      '  balance_after DECIMAL(18,2),' +
      '  description TEXT,' +
      '  reference_id VARCHAR(100),' +
      '  occurred_at TIMESTAMPTZ' +
      ')';
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

procedure TAccountBalanceProjection.OnAccountOpened(
  const AEvent: TStoredEvent);
var
  Query: TSQLQuery;
  Payload: TJSONObject;
begin
  Payload := TJSONObject(GetJSON(AEvent.Payload));
  try
    Query := TSQLQuery.Create(nil);
    try
      Query.DataBase := FWriteDb;
      Query.SQL.Text :=
        'INSERT INTO account_balances ' +
        '  (account_id, owner_name, balance, currency, is_active, updated_at) ' +
        'VALUES (:id, :name, :balance, :currency, TRUE, NOW())';
      Query.ParamByName('id').AsString := Payload.Get('accountId', '');
      Query.ParamByName('name').AsString := Payload.Get('ownerName', '');
      Query.ParamByName('balance').AsCurrency := Payload.Get('initialBalance', 0.0);
      Query.ParamByName('currency').AsString := Payload.Get('currency', 'THB');
      Query.ExecSQL;
    finally
      Query.Free;
    end;
  finally
    Payload.Free;
  end;
end;

procedure TAccountBalanceProjection.OnMoneyDeposited(
  const AEvent: TStoredEvent);
var
  Payload: TJSONObject;
begin
  Payload := TJSONObject(GetJSON(AEvent.Payload));
  try
    UpdateBalance(
      Payload.Get('accountId', ''),
      Payload.Get('amount', 0.0),
      'deposit',
      Payload.Get('description', ''),
      Payload.Get('referenceId', ''),
      AEvent.OccurredAt
    );
  finally
    Payload.Free;
  end;
end;

procedure TAccountBalanceProjection.UpdateBalance(
  const AAccountId: string; ADelta: Currency;
  const ATransactionType, ADescription, AReferenceId: string;
  AOccurredAt: TDateTime);
var
  Query: TSQLQuery;
  NewBalance: Currency;
begin
  // Get current balance
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FWriteDb;
    Query.SQL.Text :=
      'SELECT balance FROM account_balances WHERE account_id = :id';
    Query.ParamByName('id').AsString := AAccountId;
    Query.Open;
    if Query.IsEmpty then Exit;
    NewBalance := Query.Fields[0].AsCurrency + ADelta;
    Query.Close;

    // Update balance
    if ADelta > 0 then
      Query.SQL.Text :=
        'UPDATE account_balances SET ' +
        '  balance = balance + :delta, ' +
        '  total_deposited = total_deposited + :delta, ' +
        '  transaction_count = transaction_count + 1, ' +
        '  last_transaction_at = :occurred_at, ' +
        '  updated_at = NOW() ' +
        'WHERE account_id = :id'
    else
      Query.SQL.Text :=
        'UPDATE account_balances SET ' +
        '  balance = balance + :delta, ' +
        '  total_withdrawn = total_withdrawn + :abs_delta, ' +
        '  transaction_count = transaction_count + 1, ' +
        '  last_transaction_at = :occurred_at, ' +
        '  updated_at = NOW() ' +
        'WHERE account_id = :id';

    Query.ParamByName('delta').AsCurrency := ADelta;
    if ADelta < 0 then
      Query.ParamByName('abs_delta').AsCurrency := Abs(ADelta);
    Query.ParamByName('occurred_at').AsDateTime := AOccurredAt;
    Query.ParamByName('id').AsString := AAccountId;
    Query.ExecSQL;

    // Record transaction history
    Query.SQL.Text :=
      'INSERT INTO account_transactions ' +
      '  (account_id, transaction_type, amount, balance_after, ' +
      '   description, reference_id, occurred_at) ' +
      'VALUES (:id, :type, :amount, :balance_after, ' +
      '  :desc, :ref_id, :occurred_at)';
    Query.ParamByName('id').AsString := AAccountId;
    Query.ParamByName('type').AsString := ATransactionType;
    Query.ParamByName('amount').AsCurrency := Abs(ADelta);
    Query.ParamByName('balance_after').AsCurrency := NewBalance;
    Query.ParamByName('desc').AsString := ADescription;
    Query.ParamByName('ref_id').AsString := AReferenceId;
    Query.ParamByName('occurred_at').AsDateTime := AOccurredAt;
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

end.
```

## 6. Complete Example - Event Sourcing in Action

```pascal
// Examples/BankingSystem.pas
program BankingSystem;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes,
  uEventStore, uBankAccountAggregate,
  uSnapshotStore, uSnapshotRepository,
  uAccountBalanceProjection;

var
  EventStore: IEventStore;
  SnapshotStore: ISnapshotStore;
  Repository: TSnapshotRepository<TBankAccountAggregate>;
  Projection: TAccountBalanceProjection;
  Account: TBankAccountAggregate;
  Balance: TAccountBalance;
  History: TAccountTransactionHistory;
  I: Integer;

begin
  WriteLn('=== Event Sourcing Banking System ===');

  // Setup
  EventStore := TPostgresEventStore.Create('postgresql://localhost/bankdb');
  SnapshotStore := TSnapshotStore.Create(nil);
  Projection := TAccountBalanceProjection.Create(nil, nil);

  // Subscribe projection to events
  EventStore.SubscribeAll(
    procedure(AEvent: TStoredEvent)
    begin
      case AEvent.EventType of
        'account.opened': Projection.OnAccountOpened(AEvent);
        'account.money_deposited': Projection.OnMoneyDeposited(AEvent);
        'account.money_withdrawn': Projection.OnMoneyWithdrawn(AEvent);
        'account.closed': Projection.OnAccountClosed(AEvent);
      end;
    end
  );

  Repository := TSnapshotRepository<TBankAccountAggregate>.Create(
    EventStore, SnapshotStore, 10);

  // Open a new account
  Account := TBankAccountAggregate.Open(
    'ACC-001', 1001, 'สมชาย ใจดี', 5000, 'THB');
  Repository.Save(Account);
  Account.Free;

  // Load account and do transactions
  Account := Repository.GetById('ACC-001');
  try
    // Deposit money multiple times
    for I := 1 to 15 do
    begin
      Account.Deposit(1000 * I, Format('ฝากเงินครั้งที่ %d', [I]),
        Format('DEP-%04d', [I]));
    end;

    // Withdraw
    Account.Withdraw(5000, 'ถอนเงินสด', 'WTH-0001');
    Account.Withdraw(3000, 'ชำระค่าบัตรเครดิต', 'WTH-0002');

    // Transfer
    Account.Transfer('ACC-002', 2000, 'TRF-0001');

    // Save (will trigger snapshot at version 50)
    Repository.Save(Account);

    WriteLn('ยอดเงินปัจจุบัน: ', Account.Balance:0:2, ' THB');
  finally
    Account.Free;
  end;

  // Query the Read Model (Projection)
  Balance := Projection.GetBalance('ACC-001');
  try
    WriteLn(#13#10'=== ข้อมูลบัญชีจาก Read Model ===');
    WriteLn('เจ้าของบัญชี: ', Balance.OwnerName);
    WriteLn('ยอดคงเหลือ: ', FormatCurr('฿#,##0.00', Balance.Balance));
    WriteLn('ยอดฝากทั้งหมด: ', FormatCurr('฿#,##0.00', Balance.TotalDeposited));
    WriteLn('ยอดถอนทั้งหมด: ', FormatCurr('฿#,##0.00', Balance.TotalWithdrawn));
    WriteLn('จำนวนธุรกรรม: ', Balance.TransactionCount);
  finally
    Balance.Free;
  end;

  // Event Replay (Rebuild from scratch)
  WriteLn(#13#10'=== Rebuilding Projection from Events ===');
  // Reset and replay all events
  // (Useful for fixing bugs in projections)

  ReadLn;
end.
```

## 7. สรุป Event Sourcing

**ประโยชน์ของ Event Sourcing:**
1. **Full Audit Trail** - รู้ว่าข้อมูลเปลี่ยนอย่างไร เมื่อไหร่ และโดยใคร
2. **Time Travel** - สามารถดูสถานะในอดีตได้
3. **Event Replay** - สร้าง Projection ใหม่ได้ตลอดเวลา
4. **Debugging** - ง่ายต่อการหาสาเหตุของ Bug
5. **Temporal Queries** - Query ข้อมูล ณ เวลาใดก็ได้

**ข้อควรระวัง:**
- Event Schema Evolution (เมื่อ Event เปลี่ยนรูปแบบ)
- ขนาด Event Store ที่ใหญ่ขึ้น
- Eventual Consistency
- ความซับซ้อนมากกว่า Traditional CRUD
