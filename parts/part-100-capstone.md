# ตอนที่ 100: Capstone Project - Enterprise Management System

## บทนำ: โปรเจคสรุปครบถ้วน

ระบบ Enterprise Management ที่ครอบคลุมทุกทักษะที่เรียนมาตลอด 100 ตอน

## 1. System Architecture Overview

```
Enterprise Management System (EMS)
===================================

┌─────────────────────────────────────────────────────────┐
│                    Frontend Layer                        │
│  Lazarus LCL Desktop Client  │  REST API (JSON)          │
│  Web Dashboard (HTML/JS)     │  Mobile API               │
└────────────────┬────────────────────────────────────────┘
                 │ HTTP/REST
┌────────────────▼────────────────────────────────────────┐
│                    API Gateway                           │
│  Authentication  │  Rate Limiting  │  Routing            │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│                  Business Services                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │Auth/IAM  │ │HR/Employee│ │ Finance  │ │  CRM     │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │Projects  │ │Documents │ │ Reports  │ │Notifications│  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│                    Data Layer                            │
│  PostgreSQL (Main)  │  Redis (Cache)  │  Minio (Files)   │
└─────────────────────────────────────────────────────────┘

Tech Stack:
  Backend:  Free Pascal 3.2, fphttpserver, PostgreSQL 16
  Cache:    Redis 7
  Files:    MinIO (S3-compatible)
  Auth:     JWT + Refresh Tokens
  Deploy:   Docker, docker-compose
  Monitor:  Prometheus + Grafana
```

## 2. Project Structure

```
EMS/
├── src/
│   ├── api/
│   │   ├── uAPIServer.pas        # HTTP Server, routing
│   │   ├── uAPIGateway.pas       # Auth middleware, rate limit
│   │   ├── uAPIResponse.pas      # Standardized responses
│   │   └── controllers/
│   │       ├── uAuthController.pas
│   │       ├── uEmployeeController.pas
│   │       ├── uProjectController.pas
│   │       ├── uCRMController.pas
│   │       ├── uFinanceController.pas
│   │       └── uReportController.pas
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── uEmployee.pas
│   │   │   ├── uProject.pas
│   │   │   ├── uCustomer.pas
│   │   │   └── uTransaction.pas
│   │   ├── repositories/
│   │   │   ├── uIRepository.pas
│   │   │   ├── uEmployeeRepository.pas
│   │   │   └── ...
│   │   └── services/
│   │       ├── uAuthService.pas
│   │       ├── uPayrollService.pas
│   │       └── ...
│   ├── infrastructure/
│   │   ├── uDatabase.pas
│   │   ├── uRedisCache.pas
│   │   ├── uJWTAuth.pas
│   │   ├── uEmailService.pas
│   │   └── uFileStorage.pas
│   └── shared/
│       ├── uLogger.pas
│       ├── uConfig.pas
│       ├── uMoney.pas
│       └── uDIContainer.pas
├── tests/
│   ├── TestAuth.pas
│   ├── TestEmployee.pas
│   ├── TestPayroll.pas
│   └── TestIntegration.pas
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_employees.sql
│   └── ...
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── docker-compose.prod.yml
├── EMS.lpi
└── EMS.pas (main program)
```

## 3. Core Application Setup

```pascal
// EMS.pas - Main Program
program EMS;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes,
  uConfig, uLogger, uDatabase, uDIContainer,
  uAPIServer, uAPIGateway,
  uAuthController, uEmployeeController,
  uProjectController, uCRMController,
  uFinanceController, uReportController;

var
  Container: TDIContainer;
  Server: TAPIServer;

procedure SetupDependencies;
begin
  Container := TDIContainer.GetInstance;

  // Infrastructure
  Container.RegisterSingleton<ILogger, TConsoleFileLogger>;
  Container.RegisterSingleton<IDatabase, TPostgresDatabase>;
  Container.RegisterSingleton<ICacheService, TRedisCache>;
  Container.RegisterSingleton<IEmailService, TSMTPEmailService>;
  Container.RegisterSingleton<IFileStorage, TMinioStorage>;

  // Repositories
  Container.Register<IUserRepository, TUserRepository>;
  Container.Register<IEmployeeRepository, TEmployeeRepository>;
  Container.Register<IProjectRepository, TProjectRepository>;
  Container.Register<ICustomerRepository, TCustomerRepository>;
  Container.Register<ITransactionRepository, TTransactionRepository>;

  // Domain Services
  Container.Register<IAuthService, TAuthService>;
  Container.Register<IPayrollService, TPayrollService>;
  Container.Register<IProjectService, TProjectService>;
  Container.Register<ICRMService, TCRMService>;
  Container.Register<IReportService, TReportService>;

  // Controllers
  Container.Register<TAuthController, TAuthController>;
  Container.Register<TEmployeeController, TEmployeeController>;
  Container.Register<TProjectController, TProjectController>;
  Container.Register<TCRMController, TCRMController>;
  Container.Register<TFinanceController, TFinanceController>;
  Container.Register<TReportController, TReportController>;
end;

procedure SetupDatabase;
var
  DB: IDatabase;
  Migrator: TDatabaseMigrator;
begin
  DB := Container.Resolve<IDatabase>;
  DB.Connect;

  Migrator := TDatabaseMigrator.Create(DB);
  try
    Migrator.RunPendingMigrations('./migrations/');
  finally
    Migrator.Free;
  end;
end;

procedure SetupRoutes(AServer: TAPIServer);
var
  Auth: TAuthController;
  Employee: TEmployeeController;
  Project: TProjectController;
  CRM: TCRMController;
  Finance: TFinanceController;
  Reports: TReportController;
begin
  Auth := Container.Resolve<TAuthController>;
  Employee := Container.Resolve<TEmployeeController>;
  Project := Container.Resolve<TProjectController>;
  CRM := Container.Resolve<TCRMController>;
  Finance := Container.Resolve<TFinanceController>;
  Reports := Container.Resolve<TReportController>;

  // Auth routes (public)
  AServer.POST('/api/v1/auth/login', @Auth.Login);
  AServer.POST('/api/v1/auth/refresh', @Auth.RefreshToken);
  AServer.POST('/api/v1/auth/logout', @Auth.Logout);

  // Protected routes (require JWT)
  AServer.UseMiddleware(@JWTAuthMiddleware);

  // HR/Employee
  AServer.GET('/api/v1/employees', @Employee.List);
  AServer.GET('/api/v1/employees/:id', @Employee.GetById);
  AServer.POST('/api/v1/employees', @Employee.Create);
  AServer.PUT('/api/v1/employees/:id', @Employee.Update);
  AServer.DELETE('/api/v1/employees/:id', @Employee.Delete);
  AServer.GET('/api/v1/employees/:id/payslips', @Employee.GetPayslips);

  // Projects
  AServer.GET('/api/v1/projects', @Project.List);
  AServer.GET('/api/v1/projects/:id', @Project.GetById);
  AServer.POST('/api/v1/projects', @Project.Create);
  AServer.PUT('/api/v1/projects/:id', @Project.Update);
  AServer.GET('/api/v1/projects/:id/tasks', @Project.GetTasks);
  AServer.POST('/api/v1/projects/:id/tasks', @Project.AddTask);
  AServer.PUT('/api/v1/projects/:id/tasks/:taskId', @Project.UpdateTask);
  AServer.GET('/api/v1/projects/:id/gantt', @Project.GetGanttData);

  // CRM
  AServer.GET('/api/v1/customers', @CRM.ListCustomers);
  AServer.GET('/api/v1/customers/:id', @CRM.GetCustomer);
  AServer.POST('/api/v1/customers', @CRM.CreateCustomer);
  AServer.GET('/api/v1/leads', @CRM.ListLeads);
  AServer.POST('/api/v1/leads', @CRM.CreateLead);
  AServer.PUT('/api/v1/leads/:id/convert', @CRM.ConvertLead);

  // Finance
  AServer.GET('/api/v1/transactions', @Finance.ListTransactions);
  AServer.POST('/api/v1/transactions', @Finance.CreateTransaction);
  AServer.GET('/api/v1/invoices', @Finance.ListInvoices);
  AServer.POST('/api/v1/invoices', @Finance.CreateInvoice);

  // Reports
  AServer.GET('/api/v1/reports/financial', @Reports.FinancialReport);
  AServer.GET('/api/v1/reports/hr-summary', @Reports.HRSummary);
  AServer.GET('/api/v1/reports/project-status', @Reports.ProjectStatus);
  AServer.GET('/api/v1/reports/dashboard', @Reports.Dashboard);

  // Health check
  AServer.GET('/health', @HandleHealth);
end;

begin
  WriteLn('Enterprise Management System v1.0');
  WriteLn('Initializing...');

  TAppConfiguration.Initialize('./config/app.json');

  SetupDependencies;
  SetupDatabase;

  var Logger := Container.Resolve<ILogger>;
  Logger.Info('Dependencies and database initialized');

  Server := TAPIServer.Create(
    TAppConfiguration.GetInstance.Get('HTTP_HOST', '0.0.0.0'),
    TAppConfiguration.GetInstance.GetInt('HTTP_PORT', 8080)
  );
  try
    SetupRoutes(Server);

    Logger.Info(Format('Server starting on port %d',
      [TAppConfiguration.GetInstance.GetInt('HTTP_PORT', 8080)]));

    Server.Start;
  except
    on E: Exception do
    begin
      Logger.Error('Fatal error: ' + E.Message);
      ExitCode := 1;
    end;
  end;

  Server.Free;
end.
```

## 4. Employee/HR Module

```pascal
// domain/entities/uEmployee.pas
unit uEmployee;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DateUtils, Generics.Collections, uMoney;

type
  TEmployeeStatus = (esActive, esOnLeave, esSuspended, esTerminated);
  TEmploymentType = (etFullTime, etPartTime, etContract, etIntern);

  TDepartment = class
  public
    Id: string;
    Name: string;
    NameThai: string;
    ManagerId: string;
    ParentDepartmentId: string;
    Budget: TMoney;
    CostCenter: string;
  end;

  TPosition = class
  public
    Id: string;
    Title: string;
    TitleThai: string;
    DepartmentId: string;
    Level: Integer;      // Job level 1-10
    MinSalary: TMoney;
    MaxSalary: TMoney;
    IsManagement: Boolean;
  end;

  TLeaveBalance = record
    AnnualLeave: Integer;
    SickLeave: Integer;
    PersonalLeave: Integer;
    MaternityLeave: Integer;
    Year: Integer;
  end;

  TEmployee = class
  private
    FId: string;
    FEmployeeCode: string;   // รหัสพนักงาน
    FFirstName: string;
    FLastName: string;
    FFirstNameThai: string;
    FLastNameThai: string;
    FNationalId: string;
    FDateOfBirth: TDateTime;
    FHireDate: TDateTime;
    FTerminationDate: TDateTime;
    FDepartmentId: string;
    FPositionId: string;
    FManagerId: string;
    FStatus: TEmployeeStatus;
    FEmploymentType: TEmploymentType;
    FBaseSalary: TMoney;
    FBankAccount: string;
    FBankName: string;
    FPhone: string;
    FEmail: string;
    FWorkEmail: string;
    FAddress: string;
    FLeaveBalance: TLeaveBalance;

    function GetFullName: string;
    function GetYearsOfService: Double;

  public
    constructor Create(const ACode, AFirstName, ALastName: string);

    function IsEligibleForBonus: Boolean;
    function CalculateLeaveEntitlement(AYear: Integer): TLeaveBalance;

    property Id: string read FId;
    property EmployeeCode: string read FEmployeeCode;
    property FullName: string read GetFullName;
    property FirstName: string read FFirstName write FFirstName;
    property LastName: string read FLastName write FLastName;
    property NationalId: string read FNationalId write FNationalId;
    property HireDate: TDateTime read FHireDate write FHireDate;
    property DepartmentId: string read FDepartmentId write FDepartmentId;
    property PositionId: string read FPositionId write FPositionId;
    property ManagerId: string read FManagerId write FManagerId;
    property Status: TEmployeeStatus read FStatus write FStatus;
    property BaseSalary: TMoney read FBaseSalary write FBaseSalary;
    property Phone: string read FPhone write FPhone;
    property WorkEmail: string read FWorkEmail write FWorkEmail;
    property YearsOfService: Double read GetYearsOfService;
    property LeaveBalance: TLeaveBalance read FLeaveBalance write FLeaveBalance;
  end;

  // Payroll
  TPayrollItem = class
  public
    Type_: string;       // 'salary', 'allowance', 'bonus', 'deduction', 'tax', 'sso'
    Description: string;
    Amount: TMoney;
    IsDeduction: Boolean;
  end;

  TPayslip = class
  private
    FPayslipId: string;
    FEmployeeId: string;
    FPeriodMonth: Integer;
    FPeriodYear: Integer;
    FPayDate: TDateTime;
    FItems: TObjectList<TPayrollItem>;
    FGrossSalary: TMoney;
    FTotalDeductions: TMoney;
    FNetSalary: TMoney;
    FStatus: string;    // 'draft', 'approved', 'paid'

  public
    constructor Create(const AEmployeeId: string;
      AMonth, AYear: Integer);
    destructor Destroy; override;

    procedure AddItem(const AType, ADescription: string;
      const AAmount: TMoney; AIsDeduction: Boolean = False);
    procedure Calculate;
    function ToHTML: string;

    property PayslipId: string read FPayslipId;
    property EmployeeId: string read FEmployeeId;
    property GrossSalary: TMoney read FGrossSalary;
    property NetSalary: TMoney read FNetSalary;
    property Items: TObjectList<TPayrollItem> read FItems;
  end;

  TPayrollService = class(TInterfacedObject, IPayrollService)
  private
    FEmployeeRepo: IEmployeeRepository;
    FPayslipRepo: IPayslipRepository;
    FTaxCalc: TPersonalIncomeTax;
    FLogger: ILogger;

    function CalculateSSO(const ASalary: TMoney): TMoney;
    function CalculatePVD(const AEmployee: TEmployee;
      const ASalary: TMoney): TMoney;
    function CalculateTax(const AEmployee: TEmployee;
      const AMonthlyGross: TMoney): TMoney;

  public
    constructor Create(
      const AEmployeeRepo: IEmployeeRepository;
      const APayslipRepo: IPayslipRepository);

    function ProcessPayroll(AMonth, AYear: Integer): TList<TPayslip>;
    function GeneratePayslip(const AEmployeeId: string;
      AMonth, AYear: Integer): TPayslip;
    function ApprovePayroll(const APayslipIds: TArray<string>): Boolean;
    function ExportToBankFile(const APayslipIds: TArray<string>;
      const ABankFormat: string): TBytes;
  end;

implementation

procedure TPayslip.Calculate;
var
  Item: TPayrollItem;
begin
  FGrossSalary := TMoney.Zero;
  FTotalDeductions := TMoney.Zero;

  for Item in FItems do
  begin
    if not Item.IsDeduction then
      FGrossSalary := FGrossSalary + Item.Amount
    else
      FTotalDeductions := FTotalDeductions + Item.Amount;
  end;

  FNetSalary := FGrossSalary - FTotalDeductions;
end;

function TPayrollService.GeneratePayslip(
  const AEmployeeId: string; AMonth, AYear: Integer): TPayslip;
var
  Employee: TEmployee;
begin
  Employee := FEmployeeRepo.GetById(AEmployeeId);
  if not Assigned(Employee) then
    raise Exception.CreateFmt('Employee not found: %s', [AEmployeeId]);

  if Employee.Status <> esActive then
    raise Exception.CreateFmt('Employee %s is not active',
      [Employee.EmployeeCode]);

  Result := TPayslip.Create(AEmployeeId, AMonth, AYear);

  // 1. Basic Salary
  Result.AddItem('salary', 'เงินเดือน', Employee.BaseSalary);

  // 2. Allowances (from employee record)
  // ... add meal, transport, position allowances

  // 3. SSO Contribution (employee portion = 5% of salary, max 750)
  var SSODeduction := CalculateSSO(Employee.BaseSalary);
  Result.AddItem('sso', 'ประกันสังคม (ฝ่ายลูกจ้าง)',
    SSODeduction, True);

  // 4. PVD (if applicable)
  var PVDDeduction := CalculatePVD(Employee, Employee.BaseSalary);
  if not PVDDeduction.IsZero then
    Result.AddItem('pvd', 'กองทุนสำรองเลี้ยงชีพ',
      PVDDeduction, True);

  // 5. Calculate gross first
  Result.Calculate;

  // 6. Income Tax (withholding)
  var TaxDeduction := CalculateTax(Employee, Result.GrossSalary);
  if not TaxDeduction.IsZero then
    Result.AddItem('tax', 'ภาษีหัก ณ ที่จ่าย',
      TaxDeduction, True);

  // Final calculation
  Result.Calculate;

  FLogger.Info(Format('Payslip generated: %s, Net: %s',
    [Employee.EmployeeCode, Result.NetSalary.ToFormattedString]));
end;

function TPayrollService.CalculateSSO(const ASalary: TMoney): TMoney;
const
  SSO_RATE = 0.05;
  SSO_MAX_BASE = 15000;  // THB
  SSO_MAX = 750;         // 15000 * 5% = 750
var
  Base: TMoney;
begin
  if ASalary > TMoney.Create(SSO_MAX_BASE) then
    Base := TMoney.Create(SSO_MAX_BASE)
  else
    Base := ASalary;

  Result := Base * SSO_RATE;
end;

end.
```

## 5. Project Management Module (Gantt)

```pascal
// domain/entities/uProject.pas - Project + Gantt
unit uProject;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils;

type
  TTaskStatus = (tsNotStarted, tsInProgress, tsCompleted,
    tsOnHold, tsCancelled);
  TPriority = (prLow, prMedium, prHigh, prCritical);

  TProjectTask = class
  private
    FId: string;
    FProjectId: string;
    FName: string;
    FDescription: string;
    FAssignedTo: string;    // Employee ID
    FStartDate: TDateTime;
    FDueDate: TDateTime;
    FCompletedDate: TDateTime;
    FEstimatedHours: Double;
    FActualHours: Double;
    FStatus: TTaskStatus;
    FPriority: TPriority;
    FProgress: Integer;     // 0-100%
    FDependsOn: TList<string>; // Task IDs this depends on
    FParentTaskId: string;  // For subtasks

  public
    constructor Create(const AProjectId, AName: string);
    destructor Destroy; override;

    function IsOverdue: Boolean;
    function GetSlackDays: Integer;  // Buffer days

    property Id: string read FId;
    property Name: string read FName write FName;
    property StartDate: TDateTime read FStartDate write FStartDate;
    property DueDate: TDateTime read FDueDate write FDueDate;
    property Status: TTaskStatus read FStatus write FStatus;
    property Progress: Integer read FProgress write FProgress;
    property DependsOn: TList<string> read FDependsOn;
  end;

  TProject = class
  private
    FId: string;
    FCode: string;
    FName: string;
    FDescription: string;
    FClientId: string;
    FManagerId: string;
    FStartDate: TDateTime;
    FEndDate: TDateTime;
    FBudget: TMoney;
    FActualCost: TMoney;
    FStatus: string;
    FTasks: TObjectList<TProjectTask>;
    FTeamMembers: TList<string>; // Employee IDs

  public
    constructor Create(const ACode, AName: string);
    destructor Destroy; override;

    procedure AddTask(const ATask: TProjectTask);
    function GetCriticalPath: TList<TProjectTask>;
    function CalculateCompletion: Double; // 0-100%
    function GetGanttData: string;        // JSON for Gantt chart
    function IsOnTrack: Boolean;

    property Id: string read FId;
    property Name: string read FName write FName;
    property Tasks: TObjectList<TProjectTask> read FTasks;
    property Completion: Double read CalculateCompletion;
  end;

implementation

function TProject.GetGanttData: string;
var
  Task: TProjectTask;
  Rows: TStringList;
  Row: string;
const
  StatusNames: array[TTaskStatus] of string =
    ('notstarted', 'inprogress', 'completed',
     'onhold', 'cancelled');
begin
  Rows := TStringList.Create;
  try
    for Task in FTasks do
    begin
      Row := Format(
        '{"id":"%s","name":"%s","start":"%s","end":"%s",' +
        '"progress":%d,"status":"%s","assignee":"%s",' +
        '"dependencies":[%s]}',
        [Task.Id, Task.Name,
         FormatDateTime('yyyy-mm-dd', Task.StartDate),
         FormatDateTime('yyyy-mm-dd', Task.DueDate),
         Task.Progress,
         StatusNames[Task.Status],
         Task.AssignedTo,
         '"' + String.Join('","',
           Task.DependsOn.ToArray) + '"']);
      Rows.Add(Row);
    end;

    Result := Format(
      '{"project":{"id":"%s","name":"%s","start":"%s","end":"%s",' +
      '"completion":%.1f},"tasks":[%s]}',
      [FId, FName,
       FormatDateTime('yyyy-mm-dd', FStartDate),
       FormatDateTime('yyyy-mm-dd', FEndDate),
       CalculateCompletion,
       String.Join(',', Rows.ToStringArray)]);
  finally
    Rows.Free;
  end;
end;

function TProject.CalculateCompletion: Double;
var
  Task: TProjectTask;
  TotalProgress: Integer;
begin
  if FTasks.Count = 0 then
  begin
    Result := 0;
    Exit;
  end;

  TotalProgress := 0;
  for Task in FTasks do
    Inc(TotalProgress, Task.Progress);

  Result := TotalProgress / FTasks.Count;
end;

end.
```

## 6. Database Migrations

```sql
-- migrations/001_create_base_tables.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users (Authentication)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'user',
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Departments
CREATE TABLE departments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL,
    name_thai VARCHAR(100),
    manager_id UUID,
    parent_id UUID REFERENCES departments(id),
    cost_center VARCHAR(20),
    budget DECIMAL(15, 2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Employees
CREATE TABLE employees (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    employee_code VARCHAR(20) UNIQUE NOT NULL,
    user_id UUID REFERENCES users(id),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    first_name_thai VARCHAR(100),
    last_name_thai VARCHAR(100),
    national_id VARCHAR(20) UNIQUE,
    date_of_birth DATE,
    hire_date DATE NOT NULL,
    termination_date DATE,
    department_id UUID REFERENCES departments(id),
    position_id UUID,
    manager_id UUID REFERENCES employees(id),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    employment_type VARCHAR(20) NOT NULL DEFAULT 'full_time',
    base_salary DECIMAL(12, 2) NOT NULL DEFAULT 0,
    bank_account VARCHAR(20),
    bank_name VARCHAR(100),
    phone VARCHAR(20),
    email VARCHAR(255),
    work_email VARCHAR(255),
    address TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    client_id UUID,
    manager_id UUID REFERENCES employees(id),
    start_date DATE,
    end_date DATE,
    budget DECIMAL(15, 2),
    actual_cost DECIMAL(15, 2) DEFAULT 0,
    status VARCHAR(30) DEFAULT 'planning',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Project Tasks
CREATE TABLE project_tasks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
    parent_task_id UUID REFERENCES project_tasks(id),
    name VARCHAR(200) NOT NULL,
    description TEXT,
    assigned_to UUID REFERENCES employees(id),
    start_date DATE,
    due_date DATE,
    completed_date DATE,
    estimated_hours DECIMAL(6, 2),
    actual_hours DECIMAL(6, 2) DEFAULT 0,
    status VARCHAR(30) DEFAULT 'not_started',
    priority VARCHAR(20) DEFAULT 'medium',
    progress INTEGER DEFAULT 0 CHECK (progress BETWEEN 0 AND 100),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Task Dependencies
CREATE TABLE task_dependencies (
    task_id UUID REFERENCES project_tasks(id),
    depends_on_id UUID REFERENCES project_tasks(id),
    PRIMARY KEY (task_id, depends_on_id)
);

-- Customers (CRM)
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    name_thai VARCHAR(200),
    tax_id VARCHAR(20),
    type_ VARCHAR(20) DEFAULT 'company', -- company, individual
    industry VARCHAR(100),
    website VARCHAR(255),
    phone VARCHAR(20),
    email VARCHAR(255),
    address TEXT,
    account_manager_id UUID REFERENCES employees(id),
    status VARCHAR(20) DEFAULT 'active',
    credit_limit DECIMAL(15, 2) DEFAULT 0,
    payment_terms INTEGER DEFAULT 30,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Transactions (Finance)
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    reference VARCHAR(50) UNIQUE NOT NULL,
    type_ VARCHAR(30) NOT NULL, -- revenue, expense, payroll, invoice
    date DATE NOT NULL,
    debit_account VARCHAR(10),
    credit_account VARCHAR(10),
    amount DECIMAL(15, 2) NOT NULL,
    description TEXT,
    department_id UUID REFERENCES departments(id),
    project_id UUID REFERENCES projects(id),
    employee_id UUID REFERENCES employees(id),
    customer_id UUID REFERENCES customers(id),
    posted_by UUID REFERENCES users(id),
    is_posted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_employees_department ON employees(department_id);
CREATE INDEX idx_employees_manager ON employees(manager_id);
CREATE INDEX idx_employees_status ON employees(status);
CREATE INDEX idx_project_tasks_project ON project_tasks(project_id);
CREATE INDEX idx_project_tasks_assigned ON project_tasks(assigned_to);
CREATE INDEX idx_transactions_date ON transactions(date);
CREATE INDEX idx_transactions_type ON transactions(type_);
```

## 7. Docker Deployment

```dockerfile
# docker/Dockerfile
FROM ubuntu:22.04 AS builder

RUN apt-get update && apt-get install -y \
    fpc \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
COPY . .

RUN fpc -O3 -Xs \
    -k"-rpath /usr/lib/x86_64-linux-gnu" \
    -Fl/usr/lib/x86_64-linux-gnu \
    EMS.pas

# Runtime image
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    libssl3 \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /build/EMS /app/EMS
COPY config/ /app/config/
COPY migrations/ /app/migrations/

RUN useradd -r -s /bin/false ems
USER ems

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["/app/EMS"]
```

```yaml
# docker/docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=ems_db
      - DB_USER=ems_user
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - JWT_SECRET=${JWT_SECRET}
      - SMTP_HOST=${SMTP_HOST}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - ems-network

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ems_db
      POSTGRES_USER: ems_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ems_user -d ems_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - ems-network

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - ems-network

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_PASSWORD}
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    networks:
      - ems-network

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - ems-network

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - ems-network

networks:
  ems-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  minio_data:
  grafana_data:
```

## 8. API Response Standards

```pascal
// api/uAPIResponse.pas - Standardized API Responses
unit uAPIResponse;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils, fpjson, fphttpclient;

type
  TAPIResponse = class
  public
    class function Success(const AData: TJSONData;
      const AMessage: string = 'Success'): string;
    class function SuccessWithPaging(const AData: TJSONArray;
      ATotal, APage, APageSize: Integer): string;
    class function Error(const AMessage: string;
      AStatusCode: Integer = 400;
      const ACode: string = 'ERROR'): string;
    class function ValidationError(
      const AErrors: TStringList): string;
    class function NotFound(
      const AResource: string = 'Resource'): string;
    class function Unauthorized: string;
    class function Forbidden: string;
    class function InternalError(
      const ADebug: string = ''): string;
  end;

implementation

class function TAPIResponse.Success(const AData: TJSONData;
  const AMessage: string): string;
var
  Resp: TJSONObject;
begin
  Resp := TJSONObject.Create;
  try
    Resp.Add('success', True);
    Resp.Add('message', AMessage);
    Resp.Add('data', AData);
    Resp.Add('timestamp',
      FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', Now));
    Result := Resp.AsJSON;
  finally
    Resp.Free;
  end;
end;

class function TAPIResponse.SuccessWithPaging(
  const AData: TJSONArray;
  ATotal, APage, APageSize: Integer): string;
var
  Resp, Meta: TJSONObject;
begin
  Resp := TJSONObject.Create;
  try
    Resp.Add('success', True);
    Resp.Add('data', AData);

    Meta := TJSONObject.Create;
    Meta.Add('total', ATotal);
    Meta.Add('page', APage);
    Meta.Add('page_size', APageSize);
    Meta.Add('total_pages', (ATotal + APageSize - 1) div APageSize);
    Resp.Add('meta', Meta);

    Result := Resp.AsJSON;
  finally
    Resp.Free;
  end;
end;

class function TAPIResponse.Error(const AMessage: string;
  AStatusCode: Integer; const ACode: string): string;
var
  Resp: TJSONObject;
begin
  Resp := TJSONObject.Create;
  try
    Resp.Add('success', False);
    Resp.Add('error', TJSONObject.Create([
      'code', ACode,
      'message', AMessage,
      'status', AStatusCode
    ]));
    Result := Resp.AsJSON;
  finally
    Resp.Free;
  end;
end;

end.
```

## 9. Dashboard Report API

```pascal
// api/controllers/uReportController.pas
unit uReportController;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpclient, fpjson,
  uAPIResponse, uReportService;

type
  TReportController = class
  private
    FReportService: IReportService;

  public
    constructor Create(const AReportService: IReportService);

    procedure Dashboard(ARequest: TRequest; AResponse: TResponse);
    procedure FinancialReport(ARequest: TRequest; AResponse: TResponse);
    procedure HRSummary(ARequest: TRequest; AResponse: TResponse);
    procedure ProjectStatus(ARequest: TRequest; AResponse: TResponse);
  end;

implementation

procedure TReportController.Dashboard(
  ARequest: TRequest; AResponse: TResponse);
var
  Data: TJSONObject;
  DateParam: string;
  AsOfDate: TDateTime;
begin
  DateParam := ARequest.QueryFields.Values['as_of'];
  if DateParam = '' then
    AsOfDate := Now
  else
    AsOfDate := ISO8601ToDate(DateParam);

  Data := TJSONObject.Create;
  try
    // KPIs
    var KPIs := FReportService.GetDashboardKPIs(AsOfDate);
    Data.Add('kpis', TJSONObject.Create([
      'total_employees', KPIs.TotalEmployees,
      'active_projects', KPIs.ActiveProjects,
      'monthly_revenue', KPIs.MonthlyRevenue.Amount,
      'monthly_expenses', KPIs.MonthlyExpenses.Amount,
      'net_profit', KPIs.NetProfit.Amount,
      'outstanding_invoices', KPIs.OutstandingInvoices,
      'tasks_overdue', KPIs.TasksOverdue,
      'new_customers_this_month', KPIs.NewCustomers
    ]));

    // Revenue trend (last 12 months)
    var RevTrend := FReportService.GetRevenueTrend(12);
    var TrendArray := TJSONArray.Create;
    for var Point in RevTrend do
    begin
      var P := TJSONObject.Create;
      P.Add('month', FormatDateTime('yyyy-mm', Point.Date));
      P.Add('revenue', Point.Revenue.Amount);
      P.Add('expenses', Point.Expenses.Amount);
      TrendArray.Add(P);
    end;
    Data.Add('revenue_trend', TrendArray);

    // Top projects by completion
    var TopProjects := FReportService.GetTopProjects(5);
    var ProjectsArray := TJSONArray.Create;
    for var Proj in TopProjects do
    begin
      var P := TJSONObject.Create;
      P.Add('name', Proj.Name);
      P.Add('completion', Proj.Completion);
      P.Add('status', Proj.Status);
      P.Add('is_overdue', Proj.IsOverdue);
      ProjectsArray.Add(P);
    end;
    Data.Add('top_projects', ProjectsArray);

    AResponse.ContentType := 'application/json';
    AResponse.Code := 200;
    AResponse.Content := TAPIResponse.Success(Data);
  except
    on E: Exception do
    begin
      Data.Free;
      AResponse.Code := 500;
      AResponse.Content := TAPIResponse.InternalError(E.Message);
    end;
  end;
end;

end.
```

## 10. Running the System

```bash
#!/bin/bash
# scripts/start.sh - Quick start script

echo "=== Enterprise Management System ==="
echo "Starting services..."

# Check prerequisites
command -v docker >/dev/null 2>&1 || { echo "Docker required"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "docker-compose required"; exit 1; }

# Create .env if not exists
if [ ! -f .env ]; then
  cat > .env << EOF
DB_PASSWORD=$(openssl rand -base64 32)
JWT_SECRET=$(openssl rand -base64 64)
REDIS_PASSWORD=$(openssl rand -base64 32)
MINIO_USER=admin
MINIO_PASSWORD=$(openssl rand -base64 32)
GRAFANA_PASSWORD=admin123
SMTP_HOST=smtp.gmail.com
EOF
  echo "Created .env with random secrets"
fi

# Start services
docker-compose -f docker/docker-compose.yml up -d

echo "Waiting for services to be healthy..."
sleep 10

# Check health
curl -s http://localhost:8080/health | python3 -m json.tool

echo ""
echo "=== Services Running ==="
echo "API:       http://localhost:8080"
echo "Grafana:   http://localhost:3000"
echo "MinIO:     http://localhost:9001"
echo "Prometheus: http://localhost:9090"
echo ""
echo "Default admin: admin / admin123"
echo "Change password immediately!"
```

## 11. สรุป Capstone Project

**สิ่งที่ระบบนี้รวบรวม:**

| Module | Pascal Skills Used |
|--------|-------------------|
| Auth/JWT | Crypto, HTTP, JSON |
| HR/Payroll | Money arithmetic, Tax calc |
| Projects/Gantt | Algorithms, JSON generation |
| CRM | Repository pattern, Search |
| Finance | Double-entry accounting |
| Reports | SQL aggregation, JSON |
| Infrastructure | Connection pooling, Caching |
| API | REST, Middleware chain |
| Deployment | Docker, CI/CD |
| Monitoring | Prometheus metrics |

**Complexity metrics (approximate):**
- Lines of Code: ~15,000 - 20,000
- Tables: 20+
- REST Endpoints: 50+
- Test Coverage Target: >80%

**การเริ่มต้น (Step by step):**
1. Clone/create project structure
2. Setup database migrations
3. Implement core domain entities
4. Add repositories (data access)
5. Build service layer (business logic)
6. Create API controllers
7. Write unit tests
8. Docker packaging
9. Add monitoring
10. Deploy to server

**Congratulations!** คุณได้เรียนรู้ Pascal/Lazarus ครบ 100 ตอนแล้ว ตั้งแต่ basic syntax จนถึง Enterprise Application Development!

---
*จบหลักสูตร Lazarus/Pascal Programming - 100 ตอน*
*พัฒนาโดย AI Assistant สำหรับผู้เรียน Pascal ภาษาไทย*
