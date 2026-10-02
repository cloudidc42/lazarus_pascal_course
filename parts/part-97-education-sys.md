# ตอนที่ 97: Education Systems กับ Pascal

## บทนำ: ระบบการศึกษาด้วย Pascal

ระบบบริหารการศึกษาต้องการ multi-user, grading, reporting และ parent communication

## 1. School Management System

```pascal
// uSchoolManagement.pas - ระบบบริหารโรงเรียน
unit uSchoolManagement;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils, Math;

type
  TStudentStatus = (ssActive, ssInactive, ssGraduated,
    ssTransferred, ssSuspended);
  TGradeLevel = 1..12;

  TStudent = class
  private
    FStudentId: string;   // รหัสนักเรียน
    FFirstName: string;
    FLastName: string;
    FDateOfBirth: TDateTime;
    FGradeLevel: TGradeLevel;
    FClassroom: string;
    FStatus: TStudentStatus;
    FParentName: string;
    FParentPhone: string;
    FParentEmail: string;
    FAddress: string;
    FEnrollDate: TDateTime;

    function GetAge: Integer;
    function GetFullName: string;

  public
    constructor Create(const AStudentId, AFirstName,
      ALastName: string; AGradeLevel: TGradeLevel);

    property StudentId: string read FStudentId;
    property FullName: string read GetFullName;
    property FirstName: string read FFirstName write FFirstName;
    property LastName: string read FLastName write FLastName;
    property DateOfBirth: TDateTime
      read FDateOfBirth write FDateOfBirth;
    property Age: Integer read GetAge;
    property GradeLevel: TGradeLevel
      read FGradeLevel write FGradeLevel;
    property Classroom: string read FClassroom write FClassroom;
    property Status: TStudentStatus read FStatus write FStatus;
    property ParentName: string read FParentName write FParentName;
    property ParentPhone: string read FParentPhone write FParentPhone;
    property ParentEmail: string read FParentEmail write FParentEmail;
  end;

  TSubject = class
  public
    SubjectId: string;
    SubjectCode: string;   // e.g., TH11
    SubjectName: string;
    SubjectNameThai: string;
    Credits: Integer;
    GradeLevel: TGradeLevel;
    IsRequired: Boolean;
    TeacherId: string;
  end;

  TGradeRecord = class
  public
    StudentId: string;
    SubjectId: string;
    Semester: Integer;     // 1 or 2
    AcademicYear: Integer;
    Score: Double;         // 0-100
    GradePoint: Double;    // 0.0-4.0
    LetterGrade: string;   // A, B+, B, C+, C, D+, D, F
    Attendance: Double;    // % attendance
    IsIncomplete: Boolean;
    TeacherComment: string;

    class function ScoreToGrade(AScore: Double): string; static;
    class function GradeToPoint(const AGrade: string): Double; static;
  end;

  TTeacher = class
  public
    TeacherId: string;
    FirstName: string;
    LastName: string;
    FullName: string;
    SubjectSpecialty: string;
    ClassroomAssigned: string;
    Phone: string;
    Email: string;
    EmploymentDate: TDateTime;
    Qualification: string;
  end;

  TSchoolManagementSystem = class
  private
    FStudents: TObjectDictionary<string, TStudent>;
    FTeachers: TObjectDictionary<string, TTeacher>;
    FSubjects: TObjectDictionary<string, TSubject>;
    FGrades: TObjectList<TGradeRecord>;
    FAcademicYear: Integer;
    FCurrentSemester: Integer;

    function GenerateStudentId: string;
    function CalculateGPA(const AStudentId: string;
      AYear, ASemester: Integer): Double;

  public
    constructor Create(AAcademicYear, ASemester: Integer);
    destructor Destroy; override;

    // Student Management
    function EnrollStudent(const AFirstName, ALastName: string;
      AGrade: TGradeLevel; ADateOfBirth: TDateTime): TStudent;
    function GetStudent(const AStudentId: string): TStudent;
    function GetStudentsByGrade(AGrade: TGradeLevel): TList<TStudent>;

    // Grades
    procedure RecordGrade(const AStudentId, ASubjectId: string;
      AScore, AAttendance: Double;
      const AComment: string = '');
    function GetGrades(const AStudentId: string;
      AYear: Integer = -1): TList<TGradeRecord>;
    function CalculateSemesterGPA(const AStudentId: string): Double;
    function CalculateCumulativeGPA(const AStudentId: string): Double;

    // Reports
    function GenerateReportCard(const AStudentId: string): TStringList;
    function GenerateClassRanking(AGrade: TGradeLevel): TStringList;
    function GenerateAttendanceReport(
      const AClassroom: string): TStringList;

    // Statistics
    function GetGradeDistribution(const ASubjectId: string): TStringList;
    function GetTopStudents(AGrade: TGradeLevel;
      ATopN: Integer = 10): TList<TStudent>;
  end;

implementation

class function TGradeRecord.ScoreToGrade(AScore: Double): string;
begin
  if AScore >= 80 then Result := 'A'
  else if AScore >= 75 then Result := 'B+'
  else if AScore >= 70 then Result := 'B'
  else if AScore >= 65 then Result := 'C+'
  else if AScore >= 60 then Result := 'C'
  else if AScore >= 55 then Result := 'D+'
  else if AScore >= 50 then Result := 'D'
  else Result := 'F';
end;

class function TGradeRecord.GradeToPoint(const AGrade: string): Double;
begin
  if AGrade = 'A'  then Result := 4.0
  else if AGrade = 'B+' then Result := 3.5
  else if AGrade = 'B'  then Result := 3.0
  else if AGrade = 'C+' then Result := 2.5
  else if AGrade = 'C'  then Result := 2.0
  else if AGrade = 'D+' then Result := 1.5
  else if AGrade = 'D'  then Result := 1.0
  else Result := 0.0; // F
end;

function TSchoolManagementSystem.CalculateGPA(
  const AStudentId: string; AYear, ASemester: Integer): Double;
var
  Grade: TGradeRecord;
  Subject: TSubject;
  TotalPoints: Double;
  TotalCredits: Integer;
begin
  TotalPoints := 0;
  TotalCredits := 0;

  for Grade in FGrades do
  begin
    if Grade.StudentId <> AStudentId then Continue;
    if (AYear > 0) and (Grade.AcademicYear <> AYear) then Continue;
    if (ASemester > 0) and (Grade.Semester <> ASemester) then Continue;
    if Grade.IsIncomplete then Continue;

    Subject := FSubjects[Grade.SubjectId];
    if Assigned(Subject) then
    begin
      TotalPoints := TotalPoints +
        Grade.GradePoint * Subject.Credits;
      Inc(TotalCredits, Subject.Credits);
    end;
  end;

  if TotalCredits = 0 then
    Result := 0
  else
    Result := TotalPoints / TotalCredits;
end;

function TSchoolManagementSystem.GenerateReportCard(
  const AStudentId: string): TStringList;
var
  Student: TStudent;
  Grade: TGradeRecord;
  Subject: TSubject;
  Grades: TList<TGradeRecord>;
begin
  Result := TStringList.Create;
  Student := GetStudent(AStudentId);

  Result.Add('==================================');
  Result.Add('          REPORT CARD');
  Result.Add(Format('Academic Year: %d  Semester: %d',
    [FAcademicYear, FCurrentSemester]));
  Result.Add('==================================');
  Result.Add(Format('Student: %s', [Student.FullName]));
  Result.Add(Format('ID: %s', [Student.StudentId]));
  Result.Add(Format('Grade: %d  Classroom: %s',
    [Student.GradeLevel, Student.Classroom]));
  Result.Add('');
  Result.Add(Format('%-30s %6s %5s %8s %5s',
    ['Subject', 'Score', 'Grade', 'Attend', 'GPA']));
  Result.Add(StringOfChar('-', 60));

  Grades := GetGrades(AStudentId, FAcademicYear);
  try
    for Grade in Grades do
    begin
      if Grade.Semester <> FCurrentSemester then Continue;

      Subject := FSubjects[Grade.SubjectId];
      if Assigned(Subject) then
        Result.Add(Format('%-30s %6.1f %5s %7.0f%% %5.1f',
          [Subject.SubjectName,
           Grade.Score,
           Grade.LetterGrade,
           Grade.Attendance,
           Grade.GradePoint]));
    end;
  finally
    Grades.Free;
  end;

  Result.Add(StringOfChar('-', 60));
  Result.Add(Format('Semester GPA: %.2f',
    [CalculateSemesterGPA(AStudentId)]));
  Result.Add(Format('Cumulative GPA: %.2f',
    [CalculateCumulativeGPA(AStudentId)]));
end;

end.
```

## 2. Quiz Engine (LMS)

```pascal
// uQuizEngine.pas - Quiz/Assessment Engine
unit uQuizEngine;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils;

type
  TQuestionType = (qtMultipleChoice, qtTrueFalse,
    qtShortAnswer, qtEssay, qtFillBlank, qtMatching);

  TAnswer = class
  public
    AnswerId: string;
    Text: string;
    IsCorrect: Boolean;
    Explanation: string;
    Points: Double;        // Partial credit
  end;

  TQuestion = class
  private
    FQuestionId: string;
    FType: TQuestionType;
    FText: string;
    FImageUrl: string;
    FAnswers: TObjectList<TAnswer>;
    FPoints: Double;
    FDifficulty: Integer;  // 1=Easy, 2=Medium, 3=Hard
    FTags: TStringList;
    FTimeLimit: Integer;   // seconds, 0=no limit

  public
    constructor Create(const AId: string; AType: TQuestionType);
    destructor Destroy; override;

    procedure AddAnswer(const AText: string; AIsCorrect: Boolean;
      const AExplanation: string = ''; APoints: Double = 0);

    function GetCorrectAnswers: TList<TAnswer>;
    function GetMaxPoints: Double;
    function Validate: Boolean; // Check question is properly set up

    property QuestionId: string read FQuestionId;
    property QuestionType: TQuestionType read FType;
    property Text: string read FText write FText;
    property Points: Double read FPoints write FPoints;
    property Difficulty: Integer read FDifficulty write FDifficulty;
    property Answers: TObjectList<TAnswer> read FAnswers;
    property TimeLimit: Integer read FTimeLimit write FTimeLimit;
  end;

  TQuiz = class
  private
    FQuizId: string;
    FTitle: string;
    FDescription: string;
    FQuestions: TObjectList<TQuestion>;
    FTimeLimit: Integer;    // Total quiz time in minutes
    FPassScore: Double;     // % required to pass
    FMaxAttempts: Integer;
    FShuffleQuestions: Boolean;
    FShuffleAnswers: Boolean;
    FShowCorrectAnswers: Boolean;
    FIsPublished: Boolean;
    FCreatedBy: string;
    FSubjectId: string;

  public
    constructor Create(const ATitle, ACreatedBy: string);
    destructor Destroy; override;

    procedure AddQuestion(const AQuestion: TQuestion);
    procedure RemoveQuestion(const AQuestionId: string);
    function GetTotalPoints: Double;
    function GetQuestionCount: Integer;

    property QuizId: string read FQuizId;
    property Title: string read FTitle write FTitle;
    property Questions: TObjectList<TQuestion> read FQuestions;
    property PassScore: Double read FPassScore write FPassScore;
    property TimeLimit: Integer read FTimeLimit write FTimeLimit;
  end;

  // Student Response
  TStudentResponse = class
  public
    ResponseId: string;
    QuestionId: string;
    SelectedAnswers: TStringList;  // Answer IDs
    TextResponse: string;          // For essay/short answer
    IsCorrect: Boolean;
    PointsEarned: Double;
    TimeSpent: Integer;            // seconds

    constructor Create;
    destructor Destroy; override;
  end;

  // Quiz Attempt
  TQuizAttempt = class
  private
    FAttemptId: string;
    FQuizId: string;
    FStudentId: string;
    FStartTime: TDateTime;
    FEndTime: TDateTime;
    FResponses: TObjectList<TStudentResponse>;
    FScore: Double;
    FMaxScore: Double;
    FPercentage: Double;
    FPassed: Boolean;
    FStatus: string;  // 'in_progress', 'completed', 'timed_out'

  public
    constructor Create(const AQuizId, AStudentId: string);
    destructor Destroy; override;

    procedure RecordResponse(const AQuestionId: string;
      const ASelectedAnswers: TArray<string>;
      const ATextResponse: string = '');
    procedure Submit;
    procedure TimeOut;

    function GetTimeElapsed: Integer; // seconds
    function GetTimeRemaining(AQuizTimeLimit: Integer): Integer;

    property AttemptId: string read FAttemptId;
    property Score: Double read FScore;
    property Percentage: Double read FPercentage;
    property Passed: Boolean read FPassed;
    property Status: string read FStatus;
    property Responses: TObjectList<TStudentResponse> read FResponses;
  end;

  // Quiz Grader
  TQuizGrader = class
  private
    FQuiz: TQuiz;
    procedure GradeMultipleChoice(const AResponse: TStudentResponse;
      const AQuestion: TQuestion);
    procedure GradeTrueFalse(const AResponse: TStudentResponse;
      const AQuestion: TQuestion);
    procedure GradeFillBlank(const AResponse: TStudentResponse;
      const AQuestion: TQuestion);

  public
    constructor Create(const AQuiz: TQuiz);

    procedure GradeAttempt(const AAttempt: TQuizAttempt);
    function GenerateReport(const AAttempt: TQuizAttempt): TStringList;
    function GetClassAnalysis(const AAttempts: TList<TQuizAttempt>): TStringList;
  end;

implementation

procedure TQuizGrader.GradeAttempt(const AAttempt: TQuizAttempt);
var
  Response: TStudentResponse;
  Question: TQuestion;
  TotalScore: Double;
  MaxScore: Double;
begin
  TotalScore := 0;
  MaxScore := 0;

  for Response in AAttempt.FResponses do
  begin
    // Find corresponding question
    Question := nil;
    for var Q in FQuiz.Questions do
      if Q.QuestionId = Response.QuestionId then
      begin
        Question := Q;
        Break;
      end;

    if not Assigned(Question) then Continue;

    MaxScore := MaxScore + Question.GetMaxPoints;

    // Grade based on type
    case Question.QuestionType of
      qtMultipleChoice:
        GradeMultipleChoice(Response, Question);

      qtTrueFalse:
        GradeTrueFalse(Response, Question);

      qtFillBlank:
        GradeFillBlank(Response, Question);

      qtEssay:
      begin
        // Essay requires manual grading
        Response.IsCorrect := False;
        Response.PointsEarned := 0; // Will be manually set
      end;
    end;

    TotalScore := TotalScore + Response.PointsEarned;
  end;

  AAttempt.FScore := TotalScore;
  AAttempt.FMaxScore := MaxScore;
  if MaxScore > 0 then
    AAttempt.FPercentage := (TotalScore / MaxScore) * 100
  else
    AAttempt.FPercentage := 0;

  AAttempt.FPassed := AAttempt.FPercentage >= FQuiz.FPassScore;
end;

procedure TQuizGrader.GradeMultipleChoice(
  const AResponse: TStudentResponse;
  const AQuestion: TQuestion);
var
  CorrectAnswers: TList<TAnswer>;
  AllCorrect: Boolean;
  SelectedCount: Integer;
begin
  CorrectAnswers := AQuestion.GetCorrectAnswers;
  try
    // All correct answers must be selected, and no incorrect ones
    AllCorrect := True;

    for var CorrectAns in CorrectAnswers do
      if AResponse.SelectedAnswers.IndexOf(CorrectAns.AnswerId) < 0 then
      begin
        AllCorrect := False;
        Break;
      end;

    if AllCorrect then
    begin
      // Check no incorrect answers selected
      SelectedCount := AResponse.SelectedAnswers.Count;
      if SelectedCount = CorrectAnswers.Count then
      begin
        AResponse.IsCorrect := True;
        AResponse.PointsEarned := AQuestion.Points;
      end
      else
      begin
        // Partial: selected extra wrong answers
        AResponse.IsCorrect := False;
        AResponse.PointsEarned := 0;
      end;
    end
    else
    begin
      AResponse.IsCorrect := False;
      AResponse.PointsEarned := 0;
    end;
  finally
    CorrectAnswers.Free;
  end;
end;

function TQuizGrader.GenerateReport(
  const AAttempt: TQuizAttempt): TStringList;
var
  Response: TStudentResponse;
  Question: TQuestion;
begin
  Result := TStringList.Create;

  Result.Add('=== Quiz Result Report ===');
  Result.Add(Format('Student: %s', [AAttempt.FStudentId]));
  Result.Add(Format('Quiz: %s', [FQuiz.Title]));
  Result.Add(Format('Score: %.1f / %.1f (%.1f%%)',
    [AAttempt.Score, AAttempt.FMaxScore, AAttempt.Percentage]));
  Result.Add(Format('Status: %s',
    [IfThen(AAttempt.Passed, 'PASSED', 'FAILED')]));
  Result.Add(Format('Time: %d minutes',
    [AAttempt.GetTimeElapsed div 60]));
  Result.Add('');
  Result.Add('=== Question Details ===');

  var QNum := 1;
  for Response in AAttempt.Responses do
  begin
    // Find question
    Question := nil;
    for var Q in FQuiz.Questions do
      if Q.QuestionId = Response.QuestionId then
      begin
        Question := Q;
        Break;
      end;

    if not Assigned(Question) then Continue;

    Result.Add(Format('Q%d. %s', [QNum, Question.Text]));
    if Response.IsCorrect then
      Result.Add(Format('  [CORRECT] +%.1f points', [Response.PointsEarned]))
    else
      Result.Add(Format('  [INCORRECT] +0 / %.1f points',
        [Question.GetMaxPoints]));

    Inc(QNum);
  end;
end;

end.
```

## 3. Grade Calculation System

```pascal
// uGradeCalc.pas - Thai Education Grade Calculation
unit uGradeCalc;

{$mode objfpc}{$H+}

interface

uses SysUtils, Math;

type
  TGradeBreakdown = record
    Midterm: Double;     // น้ำหนัก: 30%
    Final: Double;       // น้ำหนัก: 40%
    Quizzes: Double;     // น้ำหนัก: 20%
    Assignments: Double; // น้ำหนัก: 10%
    TotalScore: Double;
    LetterGrade: string;
    GradePoint: Double;
    Remark: string;
  end;

  TGradeCalculator = class
  private
    FMidtermWeight: Double;
    FFinalWeight: Double;
    FQuizWeight: Double;
    FAssignmentWeight: Double;
    FMinAttendance: Double; // % required

    function ApplyAttendancePenalty(AScore, AAttendance: Double): Double;

  public
    constructor Create;

    procedure SetWeights(AMidterm, AFinal,
      AQuiz, AAssignment: Double);

    function Calculate(
      AMidterm, AFinal, AQuizAvg, AAssignmentAvg: Double;
      AAttendance: Double = 100): TGradeBreakdown;

    function CalculateQuizAverage(
      const AScores: TArray<Double>;
      ADropLowest: Integer = 0): Double;

    // Thai standard grading
    class function ToLetterGrade(AScore: Double): string; static;
    class function ToGradePoint(const AGrade: string): Double; static;
    class function GradeDescription(const AGrade: string): string; static;
  end;

implementation

constructor TGradeCalculator.Create;
begin
  inherited Create;
  // Default Thai university weights
  FMidtermWeight := 0.30;
  FFinalWeight := 0.40;
  FQuizWeight := 0.20;
  FAssignmentWeight := 0.10;
  FMinAttendance := 80.0;
end;

function TGradeCalculator.Calculate(
  AMidterm, AFinal, AQuizAvg, AAssignmentAvg: Double;
  AAttendance: Double): TGradeBreakdown;
var
  Total: Double;
begin
  // Calculate weighted total
  Total := (AMidterm * FMidtermWeight) +
           (AFinal * FFinalWeight) +
           (AQuizAvg * FQuizWeight) +
           (AAssignmentAvg * FAssignmentWeight);

  // Apply attendance penalty
  if AAttendance < FMinAttendance then
    Total := ApplyAttendancePenalty(Total, AAttendance);

  Result.Midterm := AMidterm;
  Result.Final := AFinal;
  Result.Quizzes := AQuizAvg;
  Result.Assignments := AAssignmentAvg;
  Result.TotalScore := Total;
  Result.LetterGrade := ToLetterGrade(Total);
  Result.GradePoint := ToGradePoint(Result.LetterGrade);
  Result.Remark := GradeDescription(Result.LetterGrade);

  if AAttendance < FMinAttendance then
    Result.Remark := Result.Remark +
      Format(' (มาเรียนต่ำกว่า %.0f%%)', [FMinAttendance]);
end;

function TGradeCalculator.ApplyAttendancePenalty(
  AScore, AAttendance: Double): Double;
begin
  if AAttendance < 60 then
    Result := 0 // Fail automatically
  else if AAttendance < FMinAttendance then
    // Deduct 5 points per 10% below minimum
    Result := AScore - 5 * Trunc((FMinAttendance - AAttendance) / 10)
  else
    Result := AScore;

  if Result < 0 then Result := 0;
end;

class function TGradeCalculator.ToLetterGrade(AScore: Double): string;
begin
  // Thai 8-level grading system
  if AScore >= 80 then Result := 'A'
  else if AScore >= 75 then Result := 'B+'
  else if AScore >= 70 then Result := 'B'
  else if AScore >= 65 then Result := 'C+'
  else if AScore >= 60 then Result := 'C'
  else if AScore >= 55 then Result := 'D+'
  else if AScore >= 50 then Result := 'D'
  else Result := 'F';
end;

class function TGradeCalculator.GradeDescription(
  const AGrade: string): string;
begin
  if AGrade = 'A'  then Result := 'ดีเยี่ยม (Excellent)'
  else if AGrade = 'B+' then Result := 'ดีมาก (Very Good)'
  else if AGrade = 'B'  then Result := 'ดี (Good)'
  else if AGrade = 'C+' then Result := 'ค่อนข้างดี (Fairly Good)'
  else if AGrade = 'C'  then Result := 'พอใช้ (Fair)'
  else if AGrade = 'D+' then Result := 'อ่อน (Poor)'
  else if AGrade = 'D'  then Result := 'อ่อนมาก (Very Poor)'
  else Result := 'ตก (Fail)';
end;

end.
```

## 4. Main Application

```pascal
// EducationApp/SchoolApp.pas - School Management Main App
program SchoolApp;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes,
  uSchoolManagement, uQuizEngine, uGradeCalc;

procedure DemoGradeCalculation;
var
  Calc: TGradeCalculator;
  Grade: TGradeBreakdown;
begin
  WriteLn('=== Grade Calculation Demo ===');

  Calc := TGradeCalculator.Create;
  try
    // Student with: Midterm=75, Final=82, Quiz avg=88, Assignment=90, Attendance=95%
    Grade := Calc.Calculate(75, 82, 88, 90, 95);

    WriteLn(Format('Midterm (30%%): %.1f', [Grade.Midterm]));
    WriteLn(Format('Final (40%%): %.1f', [Grade.Final]));
    WriteLn(Format('Quizzes (20%%): %.1f', [Grade.Quizzes]));
    WriteLn(Format('Assignments (10%%): %.1f', [Grade.Assignments]));
    WriteLn(Format('Total Score: %.1f', [Grade.TotalScore]));
    WriteLn(Format('Letter Grade: %s (%s)',
      [Grade.LetterGrade, Grade.Remark]));
    WriteLn(Format('Grade Point: %.1f', [Grade.GradePoint]));
  finally
    Calc.Free;
  end;
end;

procedure DemoQuizSystem;
var
  Quiz: TQuiz;
  Q: TQuestion;
  Grader: TQuizGrader;
  Attempt: TQuizAttempt;
  Report: TStringList;
begin
  WriteLn(#13#10'=== Quiz System Demo ===');

  Quiz := TQuiz.Create('Pascal Basics Quiz', 'teacher01');
  try
    Quiz.PassScore := 60;
    Quiz.TimeLimit := 30;

    // Question 1
    Q := TQuestion.Create('q1', qtMultipleChoice);
    Q.Text := 'Pascal ถูกพัฒนาโดยใคร?';
    Q.Points := 10;
    Q.AddAnswer('Niklaus Wirth', True, 'Pascal พัฒนาโดย Niklaus Wirth ในปี 1970');
    Q.AddAnswer('Bjarne Stroustrup', False);
    Q.AddAnswer('Guido van Rossum', False);
    Q.AddAnswer('Dennis Ritchie', False);
    Quiz.AddQuestion(Q);

    // Question 2
    Q := TQuestion.Create('q2', qtTrueFalse);
    Q.Text := 'Pascal เป็น Type-safe language';
    Q.Points := 5;
    Q.AddAnswer('True', True, 'Pascal มี strong type system');
    Q.AddAnswer('False', False);
    Quiz.AddQuestion(Q);

    Grader := TQuizGrader.Create(Quiz);
    try
      Attempt := TQuizAttempt.Create(Quiz.QuizId, 'student001');
      try
        Attempt.RecordResponse('q1', ['ans_1']); // Correct
        Attempt.RecordResponse('q2', ['ans_true']); // Correct
        Attempt.Submit;

        Grader.GradeAttempt(Attempt);

        Report := Grader.GenerateReport(Attempt);
        try
          for var Line in Report do
            WriteLn(Line);
        finally
          Report.Free;
        end;
      finally
        Attempt.Free;
      end;
    finally
      Grader.Free;
    end;
  finally
    Quiz.Free;
  end;
end;

procedure DemoSchoolManagement;
var
  School: TSchoolManagementSystem;
  Student: TStudent;
  Report: TStringList;
begin
  WriteLn(#13#10'=== School Management Demo ===');

  School := TSchoolManagementSystem.Create(2024, 1);
  try
    // Enroll students
    Student := School.EnrollStudent(
      'สมชาย', 'ใจดี', 10,
      EncodeDate(2009, 5, 15));
    Student.Classroom := '10/1';
    Student.ParentPhone := '081-234-5678';

    WriteLn('Enrolled: ', Student.FullName,
      ' (ID: ', Student.StudentId, ')');

    // Record grades
    School.RecordGrade(Student.StudentId, 'TH11',
      78.5, 92, 'มีความพยายามดี');
    School.RecordGrade(Student.StudentId, 'MA11',
      85.0, 95, 'เก่งคณิต');
    School.RecordGrade(Student.StudentId, 'EN11',
      72.0, 88, 'ต้องพัฒนาทักษะการเขียน');

    // Generate report card
    Report := School.GenerateReportCard(Student.StudentId);
    try
      for var Line in Report do
        WriteLn(Line);
    finally
      Report.Free;
    end;
  finally
    School.Free;
  end;
end;

begin
  WriteLn('Education System Demo');
  WriteLn('=====================');

  DemoGradeCalculation;
  DemoQuizSystem;
  DemoSchoolManagement;

  WriteLn(#13#10'Press Enter to exit...');
  ReadLn;
end.
```

## 5. สรุป Education Systems

**Key Components:**
1. **Student Information System (SIS)** - ข้อมูลนักเรียน, enrollment
2. **Learning Management System (LMS)** - Courses, content, quizzes
3. **Grade Management** - Thai 8-level grading, GPA calculation
4. **Attendance Tracking** - รายงานการขาด/เข้าเรียน
5. **Parent Portal** - แจ้งผู้ปกครอง

**Thai Education Standards:**
- หลักสูตรแกนกลาง พ.ศ. 2551 (Core Curriculum 2008)
- ระบบ GPA 8 ระดับ (A ถึง F)
- ข้อกำหนด 80% attendance
- มาตรฐาน สทศ. (NIETS) สำหรับ ONET/TPAT

**Integration:**
- SchoolMIS - ระบบ MIS โรงเรียน
- Thai Student ID System
- ระบบ GPAX ของ สพฐ.
- LINE/email notification สำหรับผู้ปกครอง
