# ตอนที่ 95: Financial Applications กับ Pascal

## บทนำ: Pascal สำหรับระบบการเงิน

ความแม่นยำทางตัวเลขเป็นสิ่งสำคัญที่สุดในระบบการเงิน Pascal มี Currency type และ Decimal arithmetic ที่เหมาะสม

## 1. Decimal Arithmetic และ Money Type

```pascal
// uMoney.pas - Decimal Money Type (ป้องกัน floating point errors)
unit uMoney;

{$mode objfpc}{$H+}

interface

uses SysUtils, Math;

type
  // ใช้ Int64 เพื่อ precision (เก็บเป็น satang/cents)
  TMoney = record
  private
    FCents: Int64;    // Amount in smallest unit (satang สำหรับ THB)
    FCurrency: string;

    function GetAmount: Double; inline;

  public
    class function Create(AAmount: Double;
      const ACurrency: string = 'THB'): TMoney; static;
    class function FromCents(ACents: Int64;
      const ACurrency: string = 'THB'): TMoney; static;
    class function Zero(const ACurrency: string = 'THB'): TMoney; static;

    // Arithmetic
    class operator Add(const A, B: TMoney): TMoney;
    class operator Subtract(const A, B: TMoney): TMoney;
    class operator Multiply(const M: TMoney; Factor: Double): TMoney;
    class operator Divide(const M: TMoney; Divisor: Double): TMoney;
    class operator Equal(const A, B: TMoney): Boolean;
    class operator NotEqual(const A, B: TMoney): Boolean;
    class operator GreaterThan(const A, B: TMoney): Boolean;
    class operator LessThan(const A, B: TMoney): Boolean;
    class operator GreaterThanOrEqual(const A, B: TMoney): Boolean;
    class operator LessThanOrEqual(const A, B: TMoney): Boolean;

    // Allocation (split without losing cents)
    procedure Allocate(const ARatios: array of Integer;
      out AParts: TArray<TMoney>);

    function Abs_: TMoney;
    function Negate: TMoney;
    function IsZero: Boolean;
    function IsNegative: Boolean;
    function IsPositive: Boolean;
    function ToString: string;
    function ToFormattedString: string;

    property Cents: Int64 read FCents;
    property Currency: string read FCurrency;
    property Amount: Double read GetAmount;
  end;

implementation

class function TMoney.Create(AAmount: Double;
  const ACurrency: string): TMoney;
begin
  Result.FCurrency := ACurrency;
  // Round to 2 decimal places to avoid floating point drift
  Result.FCents := Round(AAmount * 100);
end;

class function TMoney.FromCents(ACents: Int64;
  const ACurrency: string): TMoney;
begin
  Result.FCurrency := ACurrency;
  Result.FCents := ACents;
end;

function TMoney.GetAmount: Double;
begin
  Result := FCents / 100.0;
end;

class operator TMoney.Add(const A, B: TMoney): TMoney;
begin
  if A.FCurrency <> B.FCurrency then
    raise Exception.CreateFmt(
      'Cannot add %s and %s', [A.FCurrency, B.FCurrency]);

  Result.FCurrency := A.FCurrency;
  Result.FCents := A.FCents + B.FCents;
end;

class operator TMoney.Multiply(const M: TMoney; Factor: Double): TMoney;
begin
  Result.FCurrency := M.FCurrency;
  Result.FCents := Round(M.FCents * Factor);
end;

class operator TMoney.Divide(const M: TMoney; Divisor: Double): TMoney;
begin
  if Divisor = 0 then
    raise Exception.Create('Division by zero');
  Result.FCurrency := M.FCurrency;
  Result.FCents := Round(M.FCents / Divisor);
end;

procedure TMoney.Allocate(const ARatios: array of Integer;
  out AParts: TArray<TMoney>);
var
  TotalRatio, I: Integer;
  Allocated: Int64;
  Remainder: Int64;
begin
  TotalRatio := 0;
  for var R in ARatios do
    Inc(TotalRatio, R);

  SetLength(AParts, Length(ARatios));
  Allocated := 0;

  // Allocate all but last part
  for I := 0 to Length(ARatios) - 2 do
  begin
    AParts[I] := TMoney.FromCents(
      FCents * ARatios[I] div TotalRatio, FCurrency);
    Inc(Allocated, AParts[I].FCents);
  end;

  // Last part gets the remainder (ensures no money is lost)
  AParts[High(AParts)] := TMoney.FromCents(
    FCents - Allocated, FCurrency);
end;

function TMoney.ToFormattedString: string;
var
  Abs_: Int64;
  Baht, Satang: Int64;
begin
  Abs_ := System.Abs(FCents);
  Baht := Abs_ div 100;
  Satang := Abs_ mod 100;

  if FCents < 0 then
    Result := '-'
  else
    Result := '';

  Result := Result + Format('%s %s.%2.2d',
    [FCurrency,
     FormatFloat('#,##0', Baht),
     Satang]);
end;

end.
```

## 2. Accounting Ledger System

```pascal
// uLedger.pas - Double-Entry Accounting
unit uLedger;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils, uMoney;

type
  TAccountType = (atAsset, atLiability, atEquity,
    atRevenue, atExpense);

  TAccount = class
  private
    FId: string;
    FName: string;
    FAccountType: TAccountType;
    FBalance: TMoney;
    FParentId: string;

    function GetNormalBalance: Integer; // +1 debit, -1 credit

  public
    constructor Create(const AId, AName: string;
      AType: TAccountType; const ACurrency: string = 'THB');

    procedure Debit(const AAmount: TMoney);
    procedure Credit(const AAmount: TMoney);

    property Id: string read FId;
    property Name: string read FName;
    property AccountType: TAccountType read FAccountType;
    property Balance: TMoney read FBalance;
    property ParentId: string read FParentId write FParentId;
  end;

  TJournalEntry = class
  private
    FId: string;
    FDate: TDateTime;
    FDescription: string;
    FReference: string;
    FDebitAccount: string;
    FCreditAccount: string;
    FAmount: TMoney;
    FIsPosted: Boolean;

  public
    constructor Create(
      const ADebitAccountId, ACreditAccountId: string;
      const AAmount: TMoney;
      const ADescription: string;
      ADate: TDateTime = 0);

    property Id: string read FId;
    property Date: TDateTime read FDate;
    property Description: string read FDescription;
    property Reference: string read FReference write FReference;
    property DebitAccount: string read FDebitAccount;
    property CreditAccount: string read FCreditAccount;
    property Amount: TMoney read FAmount;
    property IsPosted: Boolean read FIsPosted;
  end;

  TGeneralLedger = class
  private
    FAccounts: TObjectDictionary<string, TAccount>;
    FEntries: TObjectList<TJournalEntry>;
    FFiscalYear: Integer;
    FCurrency: string;
    FLogger: ILogger;

    procedure ValidateEntry(const AEntry: TJournalEntry);
    function IsBalanced: Boolean;

  public
    constructor Create(AFiscalYear: Integer;
      const ACurrency: string = 'THB');
    destructor Destroy; override;

    // Chart of Accounts
    procedure AddAccount(const AAccount: TAccount);
    function GetAccount(const AId: string): TAccount;
    procedure SetupDefaultChartOfAccounts;

    // Journal Entries
    function RecordEntry(const AEntry: TJournalEntry): string;
    procedure PostEntry(const AEntryId: string);
    procedure PostAllPendingEntries;
    procedure VoidEntry(const AEntryId, AReason: string);

    // Financial Reports
    function GetTrialBalance: TStringList;
    function GetIncomeStatement(AStartDate, AEndDate: TDateTime): TStringList;
    function GetBalanceSheet(AAsOfDate: TDateTime): TStringList;
    function GetCashFlowStatement(AStartDate, AEndDate: TDateTime): TStringList;

    // Queries
    function GetAccountTransactions(const AAccountId: string;
      AStartDate, AEndDate: TDateTime): TList<TJournalEntry>;
    function GetNetIncome(AStartDate, AEndDate: TDateTime): TMoney;
    function GetTotalAssets: TMoney;
    function GetTotalLiabilities: TMoney;
    function GetEquity: TMoney;
  end;

implementation

constructor TAccount.Create(const AId, AName: string;
  AType: TAccountType; const ACurrency: string);
begin
  inherited Create;
  FId := AId;
  FName := AName;
  FAccountType := AType;
  FBalance := TMoney.Zero(ACurrency);
end;

function TAccount.GetNormalBalance: Integer;
begin
  // Assets and Expenses have debit normal balance
  // Liabilities, Equity, Revenue have credit normal balance
  case FAccountType of
    atAsset, atExpense: Result := 1;
    atLiability, atEquity, atRevenue: Result := -1;
  else
    Result := 1;
  end;
end;

procedure TAccount.Debit(const AAmount: TMoney);
begin
  // Debit increases assets/expenses, decreases liabilities/equity/revenue
  FBalance := FBalance + (AAmount * GetNormalBalance);
end;

procedure TAccount.Credit(const AAmount: TMoney);
begin
  // Credit is opposite of debit
  FBalance := FBalance - (AAmount * GetNormalBalance);
end;

procedure TGeneralLedger.SetupDefaultChartOfAccounts;
begin
  // Assets
  AddAccount(TAccount.Create('1000', 'Cash', atAsset));
  AddAccount(TAccount.Create('1100', 'Accounts Receivable', atAsset));
  AddAccount(TAccount.Create('1200', 'Inventory', atAsset));
  AddAccount(TAccount.Create('1500', 'Fixed Assets', atAsset));
  AddAccount(TAccount.Create('1510', 'Accumulated Depreciation', atAsset));

  // Liabilities
  AddAccount(TAccount.Create('2000', 'Accounts Payable', atLiability));
  AddAccount(TAccount.Create('2100', 'Accrued Expenses', atLiability));
  AddAccount(TAccount.Create('2500', 'Long-term Debt', atLiability));

  // Equity
  AddAccount(TAccount.Create('3000', 'Common Stock', atEquity));
  AddAccount(TAccount.Create('3100', 'Retained Earnings', atEquity));

  // Revenue
  AddAccount(TAccount.Create('4000', 'Sales Revenue', atRevenue));
  AddAccount(TAccount.Create('4100', 'Service Revenue', atRevenue));

  // Expenses
  AddAccount(TAccount.Create('5000', 'Cost of Goods Sold', atExpense));
  AddAccount(TAccount.Create('5100', 'Salaries Expense', atExpense));
  AddAccount(TAccount.Create('5200', 'Rent Expense', atExpense));
  AddAccount(TAccount.Create('5300', 'Utilities Expense', atExpense));
  AddAccount(TAccount.Create('5400', 'Depreciation Expense', atExpense));
end;

function TGeneralLedger.RecordEntry(const AEntry: TJournalEntry): string;
begin
  ValidateEntry(AEntry);
  FEntries.Add(AEntry);
  Result := AEntry.Id;
  FLogger.Info(Format('Journal Entry recorded: %s - %s (%s)',
    [AEntry.Id, AEntry.Description, AEntry.Amount.ToFormattedString]));
end;

procedure TGeneralLedger.PostEntry(const AEntryId: string);
var
  Entry: TJournalEntry;
  DebitAcc, CreditAcc: TAccount;
begin
  for Entry in FEntries do
    if Entry.Id = AEntryId then
    begin
      DebitAcc := GetAccount(Entry.DebitAccount);
      CreditAcc := GetAccount(Entry.CreditAccount);

      DebitAcc.Debit(Entry.Amount);
      CreditAcc.Credit(Entry.Amount);

      FLogger.Info(Format('Posted: Dr %s / Cr %s = %s',
        [DebitAcc.Name, CreditAcc.Name,
         Entry.Amount.ToFormattedString]));
      Exit;
    end;

  raise Exception.CreateFmt('Entry not found: %s', [AEntryId]);
end;

function TGeneralLedger.GetTrialBalance: TStringList;
var
  Account: TAccount;
  TotalDebit, TotalCredit: TMoney;
begin
  Result := TStringList.Create;
  TotalDebit := TMoney.Zero;
  TotalCredit := TMoney.Zero;

  Result.Add(Format('%-10s %-30s %15s %15s',
    ['Account', 'Name', 'Debit', 'Credit']));
  Result.Add(StringOfChar('-', 72));

  for Account in FAccounts.Values do
  begin
    var Debit := TMoney.Zero;
    var Credit := TMoney.Zero;

    if Account.Balance.IsPositive then
      Debit := Account.Balance
    else
      Credit := Account.Balance.Abs_;

    if not Debit.IsZero or not Credit.IsZero then
    begin
      Result.Add(Format('%-10s %-30s %15s %15s',
        [Account.Id, Account.Name,
         Debit.ToFormattedString,
         Credit.ToFormattedString]));

      TotalDebit := TotalDebit + Debit;
      TotalCredit := TotalCredit + Credit;
    end;
  end;

  Result.Add(StringOfChar('-', 72));
  Result.Add(Format('%-10s %-30s %15s %15s',
    ['', 'TOTAL',
     TotalDebit.ToFormattedString,
     TotalCredit.ToFormattedString]));

  if not (TotalDebit = TotalCredit) then
    Result.Add('*** WARNING: Trial balance does not balance! ***');
end;

function TGeneralLedger.GetIncomeStatement(
  AStartDate, AEndDate: TDateTime): TStringList;
var
  Account: TAccount;
  TotalRevenue, TotalExpense: TMoney;
begin
  Result := TStringList.Create;
  TotalRevenue := TMoney.Zero;
  TotalExpense := TMoney.Zero;

  Result.Add('INCOME STATEMENT');
  Result.Add(Format('Period: %s to %s',
    [DateToStr(AStartDate), DateToStr(AEndDate)]));
  Result.Add('');
  Result.Add('REVENUE');

  for Account in FAccounts.Values do
    if Account.AccountType = atRevenue then
    begin
      Result.Add(Format('  %-30s %15s',
        [Account.Name, Account.Balance.ToFormattedString]));
      TotalRevenue := TotalRevenue + Account.Balance;
    end;

  Result.Add(Format('Total Revenue: %s', [TotalRevenue.ToFormattedString]));
  Result.Add('');
  Result.Add('EXPENSES');

  for Account in FAccounts.Values do
    if Account.AccountType = atExpense then
    begin
      Result.Add(Format('  %-30s %15s',
        [Account.Name, Account.Balance.ToFormattedString]));
      TotalExpense := TotalExpense + Account.Balance;
    end;

  Result.Add(Format('Total Expenses: %s', [TotalExpense.ToFormattedString]));
  Result.Add('');

  var NetIncome := TotalRevenue - TotalExpense;
  Result.Add(Format('NET INCOME: %s', [NetIncome.ToFormattedString]));
end;

end.
```

## 3. Tax Calculation

```pascal
// uTaxCalculation.pas - Thailand Tax Calculation
unit uTaxCalculation;

{$mode objfpc}{$H+}

interface

uses SysUtils, Math, uMoney;

type
  TFilingStatus = (fsIndividual, fsJointSpouse);

  TIncomeTaxBracket = record
    MinIncome: TMoney;
    MaxIncome: TMoney;
    Rate: Double;
  end;

  TPersonalIncomeTax = class
  private
    FBrackets: array of TIncomeTaxBracket;
    FStandardDeduction: Double; // % of income
    FPersonalAllowance: TMoney;
    FSpouseAllowance: TMoney;
    FChildAllowance: TMoney;

    procedure InitThailandBrackets2024;

  public
    constructor Create;

    function CalculateTax(
      AGrossIncome: TMoney;
      ADeductions: TMoney;
      AStatus: TFilingStatus = fsIndividual;
      ANumChildren: Integer = 0;
      AHasSpouse: Boolean = False): TMoney;

    function CalculateEffectiveRate(
      AGrossIncome, ATaxPaid: TMoney): Double;

    function GetTaxBreakdown(
      AGrossIncome: TMoney;
      ADeductions: TMoney): TStringList;
  end;

  // VAT Calculation
  TVATCalculator = class
  private
    FRate: Double; // 0.07 for Thailand 7% VAT

  public
    constructor Create(ARate: Double = 0.07);

    function AddVAT(const AExcludingVAT: TMoney): TMoney;
    function ExtractVAT(const AIncludingVAT: TMoney): TMoney;
    function GetVATAmount(const AExcludingVAT: TMoney): TMoney;
  end;

implementation

procedure TPersonalIncomeTax.InitThailandBrackets2024;
begin
  // Thailand Personal Income Tax 2024
  // (Progressive rates from Revenue Department)
  SetLength(FBrackets, 7);

  // 0% for first 150,000 THB
  FBrackets[0].MinIncome := TMoney.Zero;
  FBrackets[0].MaxIncome := TMoney.Create(150000);
  FBrackets[0].Rate := 0.0;

  // 5% for 150,001 - 300,000 THB
  FBrackets[1].MinIncome := TMoney.Create(150001);
  FBrackets[1].MaxIncome := TMoney.Create(300000);
  FBrackets[1].Rate := 0.05;

  // 10% for 300,001 - 500,000 THB
  FBrackets[2].MinIncome := TMoney.Create(300001);
  FBrackets[2].MaxIncome := TMoney.Create(500000);
  FBrackets[2].Rate := 0.10;

  // 15% for 500,001 - 750,000 THB
  FBrackets[3].MinIncome := TMoney.Create(500001);
  FBrackets[3].MaxIncome := TMoney.Create(750000);
  FBrackets[3].Rate := 0.15;

  // 20% for 750,001 - 1,000,000 THB
  FBrackets[4].MinIncome := TMoney.Create(750001);
  FBrackets[4].MaxIncome := TMoney.Create(1000000);
  FBrackets[4].Rate := 0.20;

  // 25% for 1,000,001 - 2,000,000 THB
  FBrackets[5].MinIncome := TMoney.Create(1000001);
  FBrackets[5].MaxIncome := TMoney.Create(2000000);
  FBrackets[5].Rate := 0.25;

  // 30% for 2,000,001 - 5,000,000 THB
  FBrackets[6].MinIncome := TMoney.Create(2000001);
  FBrackets[6].MaxIncome := TMoney.Create(5000000);
  FBrackets[6].Rate := 0.30;

  // Personal allowances
  FPersonalAllowance := TMoney.Create(60000);  // 60,000 THB
  FSpouseAllowance := TMoney.Create(60000);
  FChildAllowance := TMoney.Create(30000);     // 30,000 per child
end;

function TPersonalIncomeTax.CalculateTax(
  AGrossIncome: TMoney;
  ADeductions: TMoney;
  AStatus: TFilingStatus;
  ANumChildren: Integer;
  AHasSpouse: Boolean): TMoney;
var
  Allowances: TMoney;
  NetIncome: TMoney;
  TaxBracket: TIncomeTaxBracket;
  Tax: TMoney;
  BracketIncome: TMoney;
begin
  // Calculate allowances
  Allowances := FPersonalAllowance;

  if AHasSpouse then
    Allowances := Allowances + FSpouseAllowance;

  Allowances := Allowances +
    (FChildAllowance * ANumChildren);

  // Standard expense deduction (50%, max 100,000)
  var StdDeduction := AGrossIncome * 0.50;
  if StdDeduction > TMoney.Create(100000) then
    StdDeduction := TMoney.Create(100000);

  // Net taxable income
  NetIncome := AGrossIncome - StdDeduction - Allowances - ADeductions;

  if NetIncome.IsNegative or NetIncome.IsZero then
  begin
    Result := TMoney.Zero;
    Exit;
  end;

  // Progressive tax calculation
  Tax := TMoney.Zero;
  for TaxBracket in FBrackets do
  begin
    if NetIncome <= TaxBracket.MinIncome then
      Break;

    if NetIncome > TaxBracket.MaxIncome then
      BracketIncome := TaxBracket.MaxIncome - TaxBracket.MinIncome
    else
      BracketIncome := NetIncome - TaxBracket.MinIncome + TMoney.FromCents(1);

    Tax := Tax + (BracketIncome * TaxBracket.Rate);
  end;

  Result := Tax;
end;

function TPersonalIncomeTax.GetTaxBreakdown(
  AGrossIncome: TMoney;
  ADeductions: TMoney): TStringList;
var
  Tax: TMoney;
  EffRate: Double;
begin
  Result := TStringList.Create;

  Tax := CalculateTax(AGrossIncome, ADeductions);
  EffRate := CalculateEffectiveRate(AGrossIncome, Tax);

  Result.Add('=== Personal Income Tax Calculation ===');
  Result.Add(Format('Gross Income:     %s',
    [AGrossIncome.ToFormattedString]));
  Result.Add(Format('Deductions:       %s',
    [ADeductions.ToFormattedString]));
  Result.Add(Format('Tax:              %s',
    [Tax.ToFormattedString]));
  Result.Add(Format('Effective Rate:   %.2f%%', [EffRate * 100]));
  Result.Add(Format('Net After Tax:    %s',
    [(AGrossIncome - Tax).ToFormattedString]));
end;

end.
```

## 4. Financial Reports Generator

```pascal
// FinancialApp/FinancialReport.pas - Complete Financial App
program FinancialReport;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes, DateUtils,
  uMoney, uLedger, uTaxCalculation;

procedure DemoAccounting;
var
  Ledger: TGeneralLedger;
  Entry: TJournalEntry;
  Report: TStringList;
begin
  Ledger := TGeneralLedger.Create(2024);
  try
    Ledger.SetupDefaultChartOfAccounts;

    // Record business transactions

    // 1. Owner invests 1,000,000 THB
    Entry := TJournalEntry.Create(
      '1000',  // Dr: Cash
      '3000',  // Cr: Common Stock
      TMoney.Create(1000000),
      'Owner investment');
    Ledger.RecordEntry(Entry);
    Ledger.PostEntry(Entry.Id);

    // 2. Buy inventory 200,000 THB
    Entry := TJournalEntry.Create(
      '1200',  // Dr: Inventory
      '1000',  // Cr: Cash
      TMoney.Create(200000),
      'Purchase inventory');
    Ledger.RecordEntry(Entry);
    Ledger.PostEntry(Entry.Id);

    // 3. Sell goods 300,000 THB
    Entry := TJournalEntry.Create(
      '1000',  // Dr: Cash
      '4000',  // Cr: Sales Revenue
      TMoney.Create(300000),
      'Sales');
    Ledger.RecordEntry(Entry);
    Ledger.PostEntry(Entry.Id);

    // 4. COGS 150,000 THB
    Entry := TJournalEntry.Create(
      '5000',  // Dr: COGS
      '1200',  // Cr: Inventory
      TMoney.Create(150000),
      'Cost of goods sold');
    Ledger.RecordEntry(Entry);
    Ledger.PostEntry(Entry.Id);

    // 5. Pay salaries 50,000 THB
    Entry := TJournalEntry.Create(
      '5100',  // Dr: Salaries
      '1000',  // Cr: Cash
      TMoney.Create(50000),
      'Monthly salaries');
    Ledger.RecordEntry(Entry);
    Ledger.PostEntry(Entry.Id);

    // Print reports
    WriteLn('=== TRIAL BALANCE ===');
    Report := Ledger.GetTrialBalance;
    try
      for var Line in Report do
        WriteLn(Line);
    finally
      Report.Free;
    end;

    WriteLn;
    WriteLn('=== INCOME STATEMENT ===');
    Report := Ledger.GetIncomeStatement(
      EncodeDate(2024, 1, 1),
      EncodeDate(2024, 12, 31));
    try
      for var Line in Report do
        WriteLn(Line);
    finally
      Report.Free;
    end;
  finally
    Ledger.Free;
  end;
end;

procedure DemoTaxCalculation;
var
  TaxCalc: TPersonalIncomeTax;
  Report: TStringList;
begin
  TaxCalc := TPersonalIncomeTax.Create;
  try
    // Employee earning 1,200,000 THB/year
    // with 100,000 in additional deductions
    Report := TaxCalc.GetTaxBreakdown(
      TMoney.Create(1200000),  // Gross income
      TMoney.Create(100000)    // Additional deductions (e.g., RMF, SSF)
    );
    try
      WriteLn;
      for var Line in Report do
        WriteLn(Line);
    finally
      Report.Free;
    end;
  finally
    TaxCalc.Free;
  end;
end;

procedure DemoMoneyArithmetic;
var
  Price, VAT, Total: TMoney;
  Parts: TArray<TMoney>;
  VATCalc: TVATCalculator;
begin
  WriteLn;
  WriteLn('=== Money Arithmetic ===');

  Price := TMoney.Create(1299.99);
  VATCalc := TVATCalculator.Create(0.07);
  try
    VAT := VATCalc.GetVATAmount(Price);
    Total := VATCalc.AddVAT(Price);

    WriteLn(Format('Price (ex VAT): %s', [Price.ToFormattedString]));
    WriteLn(Format('VAT (7%%): %s', [VAT.ToFormattedString]));
    WriteLn(Format('Total (inc VAT): %s', [Total.ToFormattedString]));

    // Split bill 3 ways
    Total.Allocate([1, 1, 1], Parts);
    WriteLn(Format('Split 3 ways: %s, %s, %s',
      [Parts[0].ToFormattedString,
       Parts[1].ToFormattedString,
       Parts[2].ToFormattedString]));
  finally
    VATCalc.Free;
  end;
end;

begin
  WriteLn('Financial Application Demo');
  WriteLn('===========================');

  DemoMoneyArithmetic;
  DemoAccounting;
  DemoTaxCalculation;

  WriteLn;
  WriteLn('Press Enter to exit...');
  ReadLn;
end.
```

## 5. สรุป Financial Applications

**Key Principles:**
1. **Never use Float for money** - ใช้ Integer (cents) หรือ Currency type
2. **Double-entry bookkeeping** - ทุก transaction ต้อง balanced
3. **Audit trail** - ห้ามลบ transactions, ใช้ reversal แทน
4. **Currency isolation** - ห้าม mix currencies โดยตรง
5. **Rounding rules** - ระบุ rounding method ชัดเจน (ROUND_HALF_UP)

**Pascal Currency type:**
- `Currency` = 64-bit fixed-point (4 decimal places)
- เหมาะสำหรับ quick calculations
- TMoney class ด้านบนเหมาะสำหรับ production

**Thai Accounting Standards:**
- TAS (Thai Accounting Standards) - ตามมาตรฐานบัญชีไทย
- TFRS (Thai Financial Reporting Standards) - ตามมาตรฐาน IFRS
- Revenue Department formats สำหรับ Tax Returns
