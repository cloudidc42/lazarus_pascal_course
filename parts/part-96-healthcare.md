# ตอนที่ 96: Healthcare Systems กับ Pascal

## บทนำ: Pascal ในระบบสาธารณสุข

ระบบโรงพยาบาลต้องการความถูกต้อง ความเป็นส่วนตัว และ Audit Trail อย่างเคร่งครัด

## 1. Patient Management System

```pascal
// uPatient.pas - Patient Data Management
unit uPatient;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils, fpjson;

type
  TGender = (gMale, gFemale, gOther, gUnknown);
  TBloodType = (btAPos, btANeg, btBPos, btBNeg,
    btOPos, btONeg, btABPos, btABNeg, btUnknown);

  TContact = record
    Phone: string;
    Email: string;
    Address: string;
    Province: string;
    PostalCode: string;
  end;

  TEmergencyContact = record
    Name: string;
    Relationship: string;
    Phone: string;
  end;

  TAllergy = class
  public
    Allergen: string;          // Drug/Food/Environmental
    AllergenType: string;      // 'drug', 'food', 'environment'
    Reaction: string;          // Reaction description
    Severity: string;          // 'mild', 'moderate', 'severe', 'fatal'
    OnsetDate: TDateTime;
    Verified: Boolean;
  end;

  TVitalSigns = class
  public
    RecordedAt: TDateTime;
    RecordedBy: string;
    TemperatureCelsius: Double;
    SystolicBP: Integer;
    DiastolicBP: Integer;
    HeartRateBPM: Integer;
    RespiratoryRate: Integer;
    OxygenSaturation: Double;  // SpO2 %
    WeightKg: Double;
    HeightCm: Double;
    BMI: Double;

    function FormatBP: string;
    function IsAbnormal: Boolean;
    function ToFHIR: string; // FHIR JSON
  end;

  TPatient = class
  private
    FId: string;              // Hospital Number (HN)
    FNationalId: string;      // Thai ID / Passport
    FFirstName: string;
    FLastName: string;
    FFirstNameEn: string;
    FLastNameEn: string;
    FDateOfBirth: TDateTime;
    FGender: TGender;
    FBloodType: TBloodType;
    FContact: TContact;
    FEmergencyContact: TEmergencyContact;
    FAllergies: TObjectList<TAllergy>;
    FVitalHistory: TObjectList<TVitalSigns>;
    FCreatedAt: TDateTime;
    FUpdatedAt: TDateTime;
    FIsActive: Boolean;

    function GetAge: Integer;
    function GetFullName: string;

  public
    constructor Create(const AHN, ANationalId: string);
    destructor Destroy; override;

    procedure AddAllergy(const AAllergy: TAllergy);
    procedure RecordVitalSigns(const AVitals: TVitalSigns);
    function GetLatestVitals: TVitalSigns;
    function HasAllergyTo(const ADrug: string): Boolean;
    function ToJSON: string;
    class function FromJSON(const AJson: string): TPatient;

    property Id: string read FId;
    property HospitalNumber: string read FId;
    property NationalId: string read FNationalId;
    property FirstName: string read FFirstName write FFirstName;
    property LastName: string read FLastName write FLastName;
    property FullName: string read GetFullName;
    property DateOfBirth: TDateTime read FDateOfBirth write FDateOfBirth;
    property Age: Integer read GetAge;
    property Gender: TGender read FGender write FGender;
    property BloodType: TBloodType read FBloodType write FBloodType;
    property Contact: TContact read FContact write FContact;
    property EmergencyContact: TEmergencyContact
      read FEmergencyContact write FEmergencyContact;
    property Allergies: TObjectList<TAllergy> read FAllergies;
    property IsActive: Boolean read FIsActive write FIsActive;
  end;

implementation

function TPatient.GetAge: Integer;
begin
  Result := YearsBetween(Now, FDateOfBirth);
end;

function TPatient.GetFullName: string;
begin
  Result := FFirstName + ' ' + FLastName;
end;

function TVitalSigns.FormatBP: string;
begin
  Result := Format('%d/%d mmHg', [SystolicBP, DiastolicBP]);
end;

function TVitalSigns.IsAbnormal: Boolean;
begin
  Result :=
    (TemperatureCelsius < 36.0) or (TemperatureCelsius > 38.3) or
    (SystolicBP < 90) or (SystolicBP > 140) or
    (DiastolicBP < 60) or (DiastolicBP > 90) or
    (HeartRateBPM < 60) or (HeartRateBPM > 100) or
    (OxygenSaturation < 95);
end;

function TVitalSigns.ToFHIR: string;
begin
  // HL7 FHIR R4 Observation resource
  Result := Format(
    '{' +
    '"resourceType":"Observation",' +
    '"status":"final",' +
    '"category":[{"coding":[{"system":"http://terminology.hl7.org/CodeSystem/observation-category","code":"vital-signs"}]}],' +
    '"code":{"coding":[{"system":"http://loinc.org","code":"55284-4","display":"Blood pressure"}]},' +
    '"effectiveDateTime":"%s",' +
    '"component":[' +
    '{"code":{"coding":[{"system":"http://loinc.org","code":"8480-6"}]},' +
    '"valueQuantity":{"value":%d,"unit":"mmHg"}},' +
    '{"code":{"coding":[{"system":"http://loinc.org","code":"8462-4"}]},' +
    '"valueQuantity":{"value":%d,"unit":"mmHg"}}' +
    ']}',
    [FormatDateTime('yyyy-mm-dd"T"hh:nn:ss', RecordedAt),
     SystolicBP, DiastolicBP]);
end;

function TPatient.HasAllergyTo(const ADrug: string): Boolean;
var
  Allergy: TAllergy;
begin
  for Allergy in FAllergies do
    if SameText(Allergy.Allergen, ADrug) then
    begin
      Result := True;
      Exit;
    end;
  Result := False;
end;

end.
```

## 2. ICD-10 and Clinical Coding

```pascal
// uClinicalCoding.pas - ICD-10 and CPT/ICD-9-CM
unit uClinicalCoding;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections;

type
  TIcdCode = record
    Code: string;
    Description: string;
    DescriptionThai: string;
    Category: string;
    IsValid: Boolean;
  end;

  TDiagnosis = class
  public
    Code: TIcdCode;
    IsPrimary: Boolean;       // หลัก/รอง
    DiagnosisType: string;    // 'admitting', 'final', 'discharge'
    ClinicalNotes: string;
    DiagnosedAt: TDateTime;
    DiagnosedBy: string;
  end;

  TProcedure = class
  public
    Code: string;             // ICD-9-CM Procedure or CPT
    Description: string;
    PerformedAt: TDateTime;
    PerformedBy: string;
    BodySite: string;
    Duration: Integer;        // minutes
    Notes: string;
  end;

  // ICD-10 Code Validator
  TICD10Validator = class
  private
    FCodeDatabase: TDictionary<string, TIcdCode>;
    procedure LoadDatabase(const AFilePath: string);

  public
    constructor Create(const ACodeDatabasePath: string);
    destructor Destroy; override;

    function Validate(const ACode: string): Boolean;
    function Lookup(const ACode: string): TIcdCode;
    function Search(const AKeyword: string): TList<TIcdCode>;
    function GetCategory(const ACode: string): string;

    // Thai-specific codes
    function IsTropicalDisease(const ACode: string): Boolean;
    function IsChronicDisease(const ACode: string): Boolean;
    function IsInfectiousDisease(const ACode: string): Boolean;
  end;

  // Drug Interaction Checker
  TDrugInteractionChecker = class
  private
    FInteractions: TDictionary<string, TList<string>>;

  public
    constructor Create(const ADataPath: string);
    destructor Destroy; override;

    function CheckInteraction(const ADrug1, ADrug2: string): string;
    function CheckMultiple(const ADrugs: TArray<string>): TStringList;
    function IsContraindicated(const ADrug: string;
      const AAllergies: TList<TAllergy>): Boolean;
  end;

implementation

function TICD10Validator.Validate(const ACode: string): Boolean;
var
  Code: string;
begin
  // ICD-10 format: Letter + 2 digits + optional decimal + more digits
  // e.g., A00.0, Z71, B34.9
  Code := UpperCase(ACode.Replace('.', ''));
  Result := False;

  if Length(Code) < 3 then Exit;
  if not (Code[1] in ['A'..'Z']) then Exit;
  if not (Code[2] in ['0'..'9']) then Exit;
  if not (Code[3] in ['0'..'9']) then Exit;

  Result := True;

  // Check against database if loaded
  if FCodeDatabase.Count > 0 then
    Result := FCodeDatabase.ContainsKey(ACode);
end;

function TICD10Validator.IsTropicalDisease(const ACode: string): Boolean;
var
  Code: string;
begin
  Code := UpperCase(ACode);
  // Tropical diseases in ICD-10
  Result :=
    // Malaria
    Code.StartsWith('B50') or Code.StartsWith('B51') or
    Code.StartsWith('B52') or Code.StartsWith('B53') or
    Code.StartsWith('B54') or
    // Dengue
    Code.StartsWith('A90') or Code.StartsWith('A91') or
    // Cholera
    Code.StartsWith('A00') or
    // Typhoid
    Code.StartsWith('A01') or
    // Leptospirosis
    Code.StartsWith('A27') or
    // Melioidosis
    Code.StartsWith('A24.4');
end;

end.
```

## 3. Electronic Health Records (EHR)

```pascal
// uEHR.pas - Electronic Health Record System
unit uEHR;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils,
  uPatient, uClinicalCoding;

type
  TEncounterType = (etOP, etIP, etER, etICU, etDaySurgery);

  TEncounter = class
  private
    FId: string;
    FPatientId: string;
    FEncounterType: TEncounterType;
    FAdmitDate: TDateTime;
    FDischargeDate: TDateTime;
    FDepartment: string;
    FPhysician: string;
    FDiagnoses: TObjectList<TDiagnosis>;
    FProcedures: TObjectList<TProcedure>;
    FVitals: TObjectList<TVitalSigns>;
    FChiefComplaint: string;
    FHPI: string;           // History of Present Illness
    FPhysicalExam: string;
    FAssessment: string;
    FPlan: string;          // SOAP Plan
    FDischargeNotes: string;

  public
    constructor Create(const APatientId: string;
      AType: TEncounterType);
    destructor Destroy; override;

    procedure AddDiagnosis(const ADiagnosis: TDiagnosis);
    procedure AddProcedure(const AProcedure: TProcedure);
    procedure RecordVitals(const AVitals: TVitalSigns);
    procedure Discharge(const ANotes: string;
      ADischargeDate: TDateTime = 0);

    function IsActive: Boolean;
    function Duration: Integer; // Days
    function ToSOAP: string;
    function ToFHIR: string;    // FHIR Encounter resource

    property Id: string read FId;
    property PatientId: string read FPatientId;
    property EncounterType: TEncounterType read FEncounterType;
    property AdmitDate: TDateTime read FAdmitDate;
    property DischargeDate: TDateTime read FDischargeDate;
    property Department: string read FDepartment write FDepartment;
    property Physician: string read FPhysician write FPhysician;
    property ChiefComplaint: string
      read FChiefComplaint write FChiefComplaint;
    property HPI: string read FHPI write FHPI;
    property Assessment: string read FAssessment write FAssessment;
    property Plan: string read FPlan write FPlan;
    property Diagnoses: TObjectList<TDiagnosis> read FDiagnoses;
  end;

  // Medication Order
  TMedicationOrder = class
  public
    OrderId: string;
    PatientId: string;
    EncounterId: string;
    DrugCode: string;
    DrugName: string;
    Dose: string;
    Route: string;          // PO, IV, IM, SC, etc.
    Frequency: string;      // q8h, qd, bid, tid, etc.
    StartDate: TDateTime;
    EndDate: TDateTime;
    OrderedBy: string;
    VerifiedBy: string;
    Status: string;         // 'ordered', 'verified', 'dispensed', 'administered', 'discontinued'
    Instructions: string;
    IsPRN: Boolean;         // As needed
    MaxDosePer24h: string;

    function IsActive: Boolean;
    function ToDHIS2Format: string;
  end;

  // Lab Result
  TLabResult = class
  public
    OrderId: string;
    PatientId: string;
    TestCode: string;
    TestName: string;
    Result: string;
    Unit_: string;
    ReferenceRange: string;
    IsAbnormal: Boolean;
    AbnormalFlag: string;   // 'H', 'L', 'HH', 'LL', '*'
    CollectedAt: TDateTime;
    ReportedAt: TDateTime;
    ReportedBy: string;
    Verified: Boolean;
  end;

  // Clinic System
  TClinicSystem = class
  private
    FPatients: TObjectDictionary<string, TPatient>;
    FEncounters: TObjectDictionary<string, TEncounter>;
    FMedOrders: TObjectList<TMedicationOrder>;
    FLabResults: TObjectList<TLabResult>;
    FIcdValidator: TICD10Validator;
    FDrugChecker: TDrugInteractionChecker;
    FLogger: IAuditLogger;
    FCriticalSection: TCriticalSection;

    function GenerateHN: string;
    function GenerateEncounterId: string;
    procedure AuditLog(const AAction, AUserId, APatientId,
      ADetails: string);

  public
    constructor Create;
    destructor Destroy; override;

    // Patient Management
    function RegisterPatient(const AFirstName, ALastName,
      ANationalId: string; ADateOfBirth: TDateTime;
      AGender: TGender): TPatient;
    function FindPatient(const AHNOrNationalId: string): TPatient;
    function SearchPatients(const AKeyword: string): TList<TPatient>;

    // Encounters
    function StartEncounter(const APatientId: string;
      AType: TEncounterType;
      const ADepartment, APhysician: string): TEncounter;
    function GetEncounter(const AId: string): TEncounter;
    function GetPatientEncounters(const APatientId: string;
      ALimit: Integer = 10): TList<TEncounter>;

    // Medications
    function OrderMedication(const AEncounterId, ADrugCode,
      ADose, ARoute, AFrequency, AOrderedBy: string): TMedicationOrder;
    function CheckDrugAllergies(const APatientId,
      ADrugCode: string): Boolean;
    function GetActiveMedications(
      const APatientId: string): TList<TMedicationOrder>;

    // Lab
    procedure RecordLabResult(const AResult: TLabResult);
    function GetLabResults(const APatientId: string;
      AStartDate, AEndDate: TDateTime): TList<TLabResult>;
    function GetCriticalValues(
      const APatientId: string): TList<TLabResult>;

    // Reports
    function GetDischareSummary(const AEncounterId: string): string;
    function GetPatientSummary(const APatientId: string): string;
    function GetDailyStatistics(ADate: TDateTime): string;
  end;

implementation

function TClinicSystem.RegisterPatient(
  const AFirstName, ALastName, ANationalId: string;
  ADateOfBirth: TDateTime; AGender: TGender): TPatient;
var
  HN: string;
begin
  FCriticalSection.Acquire;
  try
    // Check for duplicate National ID
    for var P in FPatients.Values do
      if P.NationalId = ANationalId then
        raise Exception.CreateFmt(
          'Patient with National ID %s already registered (HN: %s)',
          [ANationalId, P.Id]);

    HN := GenerateHN;
    Result := TPatient.Create(HN, ANationalId);
    Result.FirstName := AFirstName;
    Result.LastName := ALastName;
    Result.DateOfBirth := ADateOfBirth;
    Result.Gender := AGender;
    FPatients.Add(HN, Result);
  finally
    FCriticalSection.Release;
  end;

  AuditLog('REGISTER_PATIENT', 'system', HN,
    Format('New patient: %s %s', [AFirstName, ALastName]));
end;

function TClinicSystem.OrderMedication(
  const AEncounterId, ADrugCode, ADose, ARoute,
  AFrequency, AOrderedBy: string): TMedicationOrder;
var
  Encounter: TEncounter;
  Patient: TPatient;
  Order: TMedicationOrder;
begin
  Encounter := GetEncounter(AEncounterId);
  Patient := FPatients[Encounter.PatientId];

  // Check drug allergies
  if CheckDrugAllergies(Patient.Id, ADrugCode) then
  begin
    AuditLog('ALLERGY_ALERT', AOrderedBy, Patient.Id,
      Format('Medication %s ordered for patient with known allergy!',
        [ADrugCode]));
    raise Exception.CreateFmt(
      'ALLERGY ALERT: Patient %s has known allergy to %s',
      [Patient.FullName, ADrugCode]);
  end;

  // Check drug interactions with current medications
  var CurrentMeds := GetActiveMedications(Patient.Id);
  try
    var DrugList: TArray<string>;
    SetLength(DrugList, CurrentMeds.Count + 1);
    DrugList[0] := ADrugCode;
    for var I := 0 to CurrentMeds.Count - 1 do
      DrugList[I + 1] := CurrentMeds[I].DrugCode;

    var Interactions := FDrugChecker.CheckMultiple(DrugList);
    try
      if Interactions.Count > 0 then
      begin
        // Log but don't block (physician can override)
        for var Interaction in Interactions do
          AuditLog('DRUG_INTERACTION', AOrderedBy, Patient.Id,
            Interaction);
      end;
    finally
      Interactions.Free;
    end;
  finally
    CurrentMeds.Free;
  end;

  Order := TMedicationOrder.Create;
  Order.OrderId := GenerateId;
  Order.PatientId := Patient.Id;
  Order.EncounterId := AEncounterId;
  Order.DrugCode := ADrugCode;
  Order.Dose := ADose;
  Order.Route := ARoute;
  Order.Frequency := AFrequency;
  Order.OrderedBy := AOrderedBy;
  Order.StartDate := Now;
  Order.Status := 'ordered';
  FMedOrders.Add(Order);

  AuditLog('ORDER_MEDICATION', AOrderedBy, Patient.Id,
    Format('%s %s %s', [ADrugCode, ADose, AFrequency]));

  Result := Order;
end;

function TClinicSystem.GetCriticalValues(
  const APatientId: string): TList<TLabResult>;
var
  Lab: TLabResult;
begin
  Result := TList<TLabResult>.Create;
  for Lab in FLabResults do
    if (Lab.PatientId = APatientId) and
       (Lab.AbnormalFlag = 'HH') or
       (Lab.AbnormalFlag = 'LL') then
      Result.Add(Lab);
end;

end.
```

## 4. HL7 FHIR Integration

```pascal
// uFHIR.pas - HL7 FHIR R4 Interface
unit uFHIR;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, fphttpclient, uPatient;

type
  TFHIRClient = class
  private
    FBaseUrl: string;
    FHttpClient: TFPHTTPClient;
    FBearerToken: string;

    function BuildPatientResource(const APatient: TPatient): TJSONObject;
    function ParsePatient(const AJson: TJSONObject): TPatient;

  public
    constructor Create(const ABaseUrl: string);
    destructor Destroy; override;

    procedure SetBearerToken(const AToken: string);

    // Patient CRUD
    function CreatePatient(const APatient: TPatient): string; // Returns FHIR ID
    function ReadPatient(const AFhirId: string): TPatient;
    function UpdatePatient(const APatient: TPatient): Boolean;
    function SearchPatients(const AName: string): TList<TPatient>;

    // Observation (Vitals)
    function CreateObservation(const APatientId: string;
      const AVitals: TVitalSigns): string;

    // Bundle (batch operations)
    function ExecuteTransaction(const ABundle: TJSONObject): TJSONObject;
  end;

implementation

function TFHIRClient.BuildPatientResource(
  const APatient: TPatient): TJSONObject;
var
  NameArray: TJSONArray;
  NameObj: TJSONObject;
  GivenArray: TJSONArray;
  TelecomArray: TJSONArray;
  PhoneObj: TJSONObject;
const
  GenderNames: array[TGender] of string =
    ('male', 'female', 'other', 'unknown');
begin
  Result := TJSONObject.Create;
  Result.Add('resourceType', 'Patient');

  // Identifier
  var IdArray := TJSONArray.Create;
  var IdObj := TJSONObject.Create;
  IdObj.Add('system', 'urn:oid:2.16.764.1.0.hospital');
  IdObj.Add('value', APatient.HospitalNumber);
  IdArray.Add(IdObj);
  Result.Add('identifier', IdArray);

  // Active
  Result.Add('active', APatient.IsActive);

  // Name
  NameArray := TJSONArray.Create;
  NameObj := TJSONObject.Create;
  NameObj.Add('use', 'official');
  NameObj.Add('family', APatient.LastName);
  GivenArray := TJSONArray.Create;
  GivenArray.Add(APatient.FirstName);
  NameObj.Add('given', GivenArray);
  NameArray.Add(NameObj);
  Result.Add('name', NameArray);

  // Gender
  Result.Add('gender', GenderNames[APatient.Gender]);

  // Birth Date
  Result.Add('birthDate',
    FormatDateTime('yyyy-mm-dd', APatient.DateOfBirth));

  // Telecom
  if APatient.Contact.Phone <> '' then
  begin
    TelecomArray := TJSONArray.Create;
    PhoneObj := TJSONObject.Create;
    PhoneObj.Add('system', 'phone');
    PhoneObj.Add('value', APatient.Contact.Phone);
    PhoneObj.Add('use', 'mobile');
    TelecomArray.Add(PhoneObj);
    Result.Add('telecom', TelecomArray);
  end;
end;

end.
```

## 5. สรุป Healthcare Systems

**Standards สำคัญ:**
1. **HL7 FHIR R4** - Modern healthcare data exchange
2. **ICD-10** - Diagnosis coding (WHO)
3. **ICD-9-CM / ICD-10-PCS** - Procedure coding
4. **SNOMED CT** - Clinical terminology
5. **LOINC** - Lab observation codes
6. **HL7 v2.x** - Legacy messaging (still widely used)

**Security Requirements:**
- PDPA (Personal Data Protection Act) - Thailand
- HIPAA-equivalent for patient data
- End-to-end encryption สำหรับ PHI
- Role-based access control
- Audit logging ทุก access
- Data anonymization สำหรับ research

**Thai Healthcare Standards:**
- มาตรฐาน 43 แฟ้ม (43-file standard) - NHSO
- iMed: Thailand medical record format
- TMC (Thai Medical Council) guidelines
- สปสช. (NHSO) claim format

**Integration Points:**
- สปสช. / NHSO API
- กรมบัญชีกลาง / CGD API  
- Thai Hospital Information System (HIS)
- Laboratory Information System (LIS)
- Picture Archiving and Communication System (PACS)
