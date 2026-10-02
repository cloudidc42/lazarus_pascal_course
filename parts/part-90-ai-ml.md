# ตอนที่ 90: AI/ML Integration กับ Pascal/Lazarus

## บทนำ: การนำ AI/ML มาใช้ใน Pascal

แม้ Pascal จะไม่ใช่ภาษาหลักสำหรับ ML แต่เราสามารถเรียกใช้ ML Services และ Libraries ได้ผ่าน API หรือ Native Bindings

## 1. เรียก Claude/OpenAI API

```pascal
// uLLMClient.pas - LLM API Client
unit uLLMClient;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpclient, fpjson, Generics.Collections;

type
  TMessageRole = (mrSystem, mrUser, mrAssistant);

  TChatMessage = record
    Role: TMessageRole;
    Content: string;
  end;

  TLLMConfig = record
    ApiKey: string;
    BaseUrl: string;
    Model: string;
    MaxTokens: Integer;
    Temperature: Double;
    TimeoutMs: Integer;
  end;

  TCompletionResult = class
  public
    Content: string;
    FinishReason: string;
    PromptTokens: Integer;
    CompletionTokens: Integer;
    TotalTokens: Integer;
    Model: string;
    RequestId: string;
  end;

  ILLMClient = interface
    function Complete(const AMessages: TArray<TChatMessage>): TCompletionResult;
    function CompleteStream(const AMessages: TArray<TChatMessage>;
      const ACallback: TProc<string>): TCompletionResult;
    function Embed(const AText: string): TArray<Double>;
    function EmbedBatch(const ATexts: TArray<string>): TArray<TArray<Double>>;
  end;

  // Anthropic Claude Client
  TAnthropicClient = class(TInterfacedObject, ILLMClient)
  private
    FConfig: TLLMConfig;
    FHttpClient: TFPHTTPClient;
    FLogger: ILogger;

    function BuildRequestBody(const AMessages: TArray<TChatMessage>): TJSONObject;
    function ParseResponse(const AResponseJson: string): TCompletionResult;
    procedure HandleError(AStatusCode: Integer; const ABody: string);

  public
    constructor Create(const AConfig: TLLMConfig);
    destructor Destroy; override;

    function Complete(const AMessages: TArray<TChatMessage>): TCompletionResult;
    function CompleteStream(const AMessages: TArray<TChatMessage>;
      const ACallback: TProc<string>): TCompletionResult;
    function Embed(const AText: string): TArray<Double>;
    function EmbedBatch(const ATexts: TArray<string>): TArray<TArray<Double>>;
  end;

  // OpenAI Client
  TOpenAIClient = class(TInterfacedObject, ILLMClient)
  private
    FConfig: TLLMConfig;
    FHttpClient: TFPHTTPClient;

  public
    constructor Create(const AConfig: TLLMConfig);
    destructor Destroy; override;

    function Complete(const AMessages: TArray<TChatMessage>): TCompletionResult;
    function CompleteStream(const AMessages: TArray<TChatMessage>;
      const ACallback: TProc<string>): TCompletionResult;
    function Embed(const AText: string): TArray<Double>;
    function EmbedBatch(const ATexts: TArray<string>): TArray<TArray<Double>>;
  end;

implementation

constructor TAnthropicClient.Create(const AConfig: TLLMConfig);
begin
  inherited Create;
  FConfig := AConfig;
  FHttpClient := TFPHTTPClient.Create(nil);
  FHttpClient.AddHeader('x-api-key', FConfig.ApiKey);
  FHttpClient.AddHeader('anthropic-version', '2023-06-01');
  FHttpClient.AddHeader('content-type', 'application/json');
end;

function TAnthropicClient.BuildRequestBody(
  const AMessages: TArray<TChatMessage>): TJSONObject;
var
  MessagesArray: TJSONArray;
  MessageObj: TJSONObject;
  Msg: TChatMessage;
  SystemContent: string;
const
  RoleNames: array[TMessageRole] of string = ('', 'user', 'assistant');
begin
  Result := TJSONObject.Create;
  Result.Add('model', FConfig.Model);
  Result.Add('max_tokens', FConfig.MaxTokens);
  Result.Add('temperature', FConfig.Temperature);

  // Extract system message
  SystemContent := '';
  MessagesArray := TJSONArray.Create;

  for Msg in AMessages do
  begin
    if Msg.Role = mrSystem then
      SystemContent := Msg.Content
    else
    begin
      MessageObj := TJSONObject.Create;
      MessageObj.Add('role', RoleNames[Msg.Role]);
      MessageObj.Add('content', Msg.Content);
      MessagesArray.Add(MessageObj);
    end;
  end;

  if SystemContent <> '' then
    Result.Add('system', SystemContent);

  Result.Add('messages', MessagesArray);
end;

function TAnthropicClient.Complete(
  const AMessages: TArray<TChatMessage>): TCompletionResult;
var
  RequestBody: TJSONObject;
  RequestStr: string;
  ResponseStream: TStringStream;
  StatusCode: Integer;
begin
  RequestBody := BuildRequestBody(AMessages);
  try
    RequestStr := RequestBody.AsJSON;
  finally
    RequestBody.Free;
  end;

  ResponseStream := TStringStream.Create('', CP_UTF8);
  try
    FHttpClient.RequestBody := TStringStream.Create(RequestStr, CP_UTF8);
    try
      FHttpClient.ConnectTimeout := FConfig.TimeoutMs;
      FHttpClient.IOTimeout := FConfig.TimeoutMs;
      FHttpClient.Post(FConfig.BaseUrl + '/messages', ResponseStream);

      StatusCode := FHttpClient.ResponseStatusCode;
      if StatusCode >= 400 then
        HandleError(StatusCode, ResponseStream.DataString);

      Result := ParseResponse(ResponseStream.DataString);
    finally
      FHttpClient.RequestBody.Free;
      FHttpClient.RequestBody := nil;
    end;
  finally
    ResponseStream.Free;
  end;
end;

function TAnthropicClient.ParseResponse(
  const AResponseJson: string): TCompletionResult;
var
  Json: TJSONObject;
  ContentArray: TJSONArray;
  ContentObj: TJSONObject;
  UsageObj: TJSONObject;
begin
  Result := TCompletionResult.Create;

  Json := TJSONObject(GetJSON(AResponseJson));
  try
    Result.RequestId := Json.Get('id', '');
    Result.Model := Json.Get('model', '');
    Result.FinishReason := Json.Get('stop_reason', '');

    ContentArray := TJSONArray(Json.Find('content'));
    if Assigned(ContentArray) and (ContentArray.Count > 0) then
    begin
      ContentObj := TJSONObject(ContentArray[0]);
      Result.Content := ContentObj.Get('text', '');
    end;

    UsageObj := TJSONObject(Json.Find('usage'));
    if Assigned(UsageObj) then
    begin
      Result.PromptTokens := UsageObj.Get('input_tokens', 0);
      Result.CompletionTokens := UsageObj.Get('output_tokens', 0);
      Result.TotalTokens := Result.PromptTokens + Result.CompletionTokens;
    end;
  finally
    Json.Free;
  end;
end;

// OpenAI Embeddings
function TOpenAIClient.Embed(const AText: string): TArray<Double>;
var
  RequestBody: TJSONObject;
  Response: TStringStream;
  Json: TJSONObject;
  DataArray: TJSONArray;
  EmbeddingArray: TJSONArray;
  I: Integer;
begin
  RequestBody := TJSONObject.Create;
  try
    RequestBody.Add('model', 'text-embedding-3-small');
    RequestBody.Add('input', AText);

    Response := TStringStream.Create;
    try
      FHttpClient.RequestBody := TStringStream.Create(RequestBody.AsJSON, CP_UTF8);
      FHttpClient.Post(FConfig.BaseUrl + '/embeddings', Response);

      Json := TJSONObject(GetJSON(Response.DataString));
      try
        DataArray := TJSONArray(Json.Find('data'));
        if Assigned(DataArray) and (DataArray.Count > 0) then
        begin
          EmbeddingArray := TJSONArray(
            TJSONObject(DataArray[0]).Find('embedding'));
          if Assigned(EmbeddingArray) then
          begin
            SetLength(Result, EmbeddingArray.Count);
            for I := 0 to EmbeddingArray.Count - 1 do
              Result[I] := EmbeddingArray.Floats[I];
          end;
        end;
      finally
        Json.Free;
      end;
    finally
      Response.Free;
    end;
  finally
    RequestBody.Free;
  end;
end;

end.
```

## 2. Semantic Search with Vector Embeddings

```pascal
// uVectorSearch.pas - Semantic Search with Embeddings
unit uVectorSearch;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, Math;

type
  TVector = TArray<Double>;

  TVectorDocument = class
  public
    Id: string;
    Content: string;
    Embedding: TVector;
    Metadata: TStringList;

    constructor Create;
    destructor Destroy; override;
  end;

  TSimilarityResult = record
    Document: TVectorDocument;
    Score: Double;
  end;

  IVectorStore = interface
    procedure Add(const ADocument: TVectorDocument);
    procedure AddBatch(const ADocuments: TArray<TVectorDocument>);
    function Search(const AQuery: TVector; ATopK: Integer = 5): TArray<TSimilarityResult>;
    function SearchByText(const AQueryText: string; ATopK: Integer = 5): TArray<TSimilarityResult>;
    procedure Delete(const AId: string);
    function Count: Integer;
  end;

  // In-Memory Vector Store (for small datasets)
  TInMemoryVectorStore = class(TInterfacedObject, IVectorStore)
  private
    FDocuments: TObjectList<TVectorDocument>;
    FEmbedder: ILLMClient;
    FCriticalSection: TCriticalSection;

    function CosineSimilarity(const AV1, AV2: TVector): Double;
    function DotProduct(const AV1, AV2: TVector): Double;
    function Magnitude(const AV: TVector): Double;

  public
    constructor Create(const AEmbedder: ILLMClient);
    destructor Destroy; override;

    procedure Add(const ADocument: TVectorDocument);
    procedure AddBatch(const ADocuments: TArray<TVectorDocument>);
    function Search(const AQuery: TVector; ATopK: Integer): TArray<TSimilarityResult>;
    function SearchByText(const AQueryText: string; ATopK: Integer): TArray<TSimilarityResult>;
    procedure Delete(const AId: string);
    function Count: Integer;
  end;

  // Postgres pgvector Store
  TPgVectorStore = class(TInterfacedObject, IVectorStore)
  private
    FConnection: TSQLConnection;
    FEmbedder: ILLMClient;
    FTableName: string;
    FDimensions: Integer;

    procedure EnsureTable;
    function VectorToSql(const AVector: TVector): string;

  public
    constructor Create(const AConnection: TSQLConnection;
      const AEmbedder: ILLMClient;
      const ATableName: string = 'embeddings';
      ADimensions: Integer = 1536);

    procedure Add(const ADocument: TVectorDocument);
    procedure AddBatch(const ADocuments: TArray<TVectorDocument>);
    function Search(const AQuery: TVector; ATopK: Integer): TArray<TSimilarityResult>;
    function SearchByText(const AQueryText: string; ATopK: Integer): TArray<TSimilarityResult>;
    procedure Delete(const AId: string);
    function Count: Integer;
  end;

implementation

function TInMemoryVectorStore.CosineSimilarity(
  const AV1, AV2: TVector): Double;
var
  Dot, Mag1, Mag2: Double;
begin
  if (Length(AV1) = 0) or (Length(AV1) <> Length(AV2)) then
  begin
    Result := 0;
    Exit;
  end;

  Dot := DotProduct(AV1, AV2);
  Mag1 := Magnitude(AV1);
  Mag2 := Magnitude(AV2);

  if (Mag1 = 0) or (Mag2 = 0) then
    Result := 0
  else
    Result := Dot / (Mag1 * Mag2);
end;

function TInMemoryVectorStore.DotProduct(const AV1, AV2: TVector): Double;
var
  I: Integer;
begin
  Result := 0;
  for I := 0 to High(AV1) do
    Result := Result + AV1[I] * AV2[I];
end;

function TInMemoryVectorStore.Magnitude(const AV: TVector): Double;
var
  Sum: Double;
  V: Double;
begin
  Sum := 0;
  for V in AV do
    Sum := Sum + V * V;
  Result := Sqrt(Sum);
end;

function TInMemoryVectorStore.Search(const AQuery: TVector;
  ATopK: Integer): TArray<TSimilarityResult>;
var
  Results: TList<TSimilarityResult>;
  Doc: TVectorDocument;
  Similarity: Double;
  SR: TSimilarityResult;
begin
  Results := TList<TSimilarityResult>.Create;
  try
    FCriticalSection.Acquire;
    try
      for Doc in FDocuments do
      begin
        Similarity := CosineSimilarity(AQuery, Doc.Embedding);
        SR.Document := Doc;
        SR.Score := Similarity;
        Results.Add(SR);
      end;
    finally
      FCriticalSection.Release;
    end;

    // Sort by score descending
    Results.Sort(TComparer<TSimilarityResult>.Construct(
      function(const A, B: TSimilarityResult): Integer
      begin
        if A.Score > B.Score then Result := -1
        else if A.Score < B.Score then Result := 1
        else Result := 0;
      end
    ));

    // Return top K
    var ActualK := Min(ATopK, Results.Count);
    SetLength(Result, ActualK);
    for var I := 0 to ActualK - 1 do
      Result[I] := Results[I];
  finally
    Results.Free;
  end;
end;

function TInMemoryVectorStore.SearchByText(const AQueryText: string;
  ATopK: Integer): TArray<TSimilarityResult>;
var
  QueryEmbedding: TVector;
begin
  QueryEmbedding := FEmbedder.Embed(AQueryText);
  Result := Search(QueryEmbedding, ATopK);
end;

// pgvector Store
procedure TPgVectorStore.EnsureTable;
var
  Query: TSQLQuery;
begin
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FConnection;
    // Enable pgvector extension
    Query.SQL.Text := 'CREATE EXTENSION IF NOT EXISTS vector';
    Query.ExecSQL;

    // Create table
    Query.SQL.Text := Format(
      'CREATE TABLE IF NOT EXISTS %s (' +
      '  id VARCHAR(100) PRIMARY KEY,' +
      '  content TEXT,' +
      '  embedding vector(%d),' +
      '  metadata JSONB,' +
      '  created_at TIMESTAMPTZ DEFAULT NOW()' +
      ')', [FTableName, FDimensions]);
    Query.ExecSQL;

    // Create index for fast similarity search
    Query.SQL.Text := Format(
      'CREATE INDEX IF NOT EXISTS idx_%s_embedding ' +
      'ON %s USING ivfflat (embedding vector_cosine_ops) ' +
      'WITH (lists = 100)',
      [FTableName, FTableName]);
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

function TPgVectorStore.VectorToSql(const AVector: TVector): string;
var
  Parts: TStringList;
  V: Double;
begin
  Parts := TStringList.Create;
  try
    for V in AVector do
      Parts.Add(FloatToStr(V));
    Result := '[' + String.Join(',', Parts.ToStringArray) + ']';
  finally
    Parts.Free;
  end;
end;

function TPgVectorStore.Search(const AQuery: TVector;
  ATopK: Integer): TArray<TSimilarityResult>;
var
  Query: TSQLQuery;
  Results: TList<TSimilarityResult>;
  SR: TSimilarityResult;
  Doc: TVectorDocument;
begin
  Results := TList<TSimilarityResult>.Create;
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FConnection;
    Query.SQL.Text := Format(
      'SELECT id, content, metadata, ' +
      '  1 - (embedding <=> :query_vec::vector) AS similarity ' +
      'FROM %s ' +
      'ORDER BY embedding <=> :query_vec2::vector ' +
      'LIMIT :top_k', [FTableName]);

    Query.ParamByName('query_vec').AsString := VectorToSql(AQuery);
    Query.ParamByName('query_vec2').AsString := VectorToSql(AQuery);
    Query.ParamByName('top_k').AsInteger := ATopK;
    Query.Open;

    while not Query.EOF do
    begin
      Doc := TVectorDocument.Create;
      Doc.Id := Query.FieldByName('id').AsString;
      Doc.Content := Query.FieldByName('content').AsString;

      SR.Document := Doc;
      SR.Score := Query.FieldByName('similarity').AsFloat;
      Results.Add(SR);
      Query.Next;
    end;

    Result := Results.ToArray;
  finally
    Query.Free;
    Results.Free;
  end;
end;

end.
```

## 3. RAG (Retrieval Augmented Generation)

```pascal
// uRAG.pas - Retrieval Augmented Generation
unit uRAG;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections,
  uLLMClient, uVectorSearch;

type
  TRAGConfig = record
    TopK: Integer;               // Documents to retrieve
    MaxContextLength: Integer;   // Max chars in context
    SimilarityThreshold: Double; // Min similarity score
    SystemPrompt: string;
  end;

  TRAGResponse = class
  public
    Answer: string;
    Sources: TList<TVectorDocument>;
    RelevantDocs: Integer;
    TokensUsed: Integer;

    constructor Create;
    destructor Destroy; override;
  end;

  TRAG = class
  private
    FLlm: ILLMClient;
    FVectorStore: IVectorStore;
    FConfig: TRAGConfig;
    FLogger: ILogger;

    function BuildPrompt(const AQuestion: string;
      const ADocuments: TArray<TSimilarityResult>): TArray<TChatMessage>;
    function FormatContext(const ADocuments: TArray<TSimilarityResult>): string;

  public
    constructor Create(
      const ALlm: ILLMClient;
      const AVectorStore: IVectorStore;
      const AConfig: TRAGConfig);

    function Query(const AQuestion: string): TRAGResponse;

    // Document ingestion
    procedure IngestText(const AId, AText: string;
      const AMetadata: TStringList = nil);
    procedure IngestFile(const AFilePath: string);
    procedure IngestDirectory(const ADirPath: string;
      const APattern: string = '*.txt');
  end;

implementation

function TRAG.Query(const AQuestion: string): TRAGResponse;
var
  SearchResults: TArray<TSimilarityResult>;
  Messages: TArray<TChatMessage>;
  Completion: TCompletionResult;
  SR: TSimilarityResult;
begin
  Result := TRAGResponse.Create;

  FLogger.Info('RAG Query: ' + AQuestion);

  // 1. Retrieve relevant documents
  SearchResults := FVectorStore.SearchByText(AQuestion, FConfig.TopK);

  // 2. Filter by similarity threshold
  var FilteredResults: TList<TSimilarityResult>;
  FilteredResults := TList<TSimilarityResult>.Create;
  try
    for SR in SearchResults do
      if SR.Score >= FConfig.SimilarityThreshold then
        FilteredResults.Add(SR);

    Result.RelevantDocs := FilteredResults.Count;
    FLogger.Debug(Format('Found %d relevant documents', [FilteredResults.Count]));

    if FilteredResults.Count = 0 then
    begin
      // No relevant docs - answer from LLM only
      Messages := [
        ChatMessage(mrSystem, FConfig.SystemPrompt),
        ChatMessage(mrUser, AQuestion)
      ];
    end
    else
    begin
      // Build context from retrieved docs
      Messages := BuildPrompt(AQuestion, FilteredResults.ToArray);

      // Add sources to response
      for SR in FilteredResults do
        Result.Sources.Add(SR.Document);
    end;
  finally
    FilteredResults.Free;
  end;

  // 3. Generate answer
  Completion := FLlm.Complete(Messages);
  try
    Result.Answer := Completion.Content;
    Result.TokensUsed := Completion.TotalTokens;
  finally
    Completion.Free;
  end;
end;

function TRAG.BuildPrompt(const AQuestion: string;
  const ADocuments: TArray<TSimilarityResult>): TArray<TChatMessage>;
var
  Context: string;
  SystemMsg, UserMsg: TChatMessage;
begin
  Context := FormatContext(ADocuments);

  SystemMsg.Role := mrSystem;
  SystemMsg.Content :=
    FConfig.SystemPrompt + #13#10#13#10 +
    'Use the following context to answer the question. ' +
    'If the context does not contain enough information, ' +
    'say so and answer based on your general knowledge.' + #13#10#13#10 +
    'Context:' + #13#10 + Context;

  UserMsg.Role := mrUser;
  UserMsg.Content := AQuestion;

  Result := [SystemMsg, UserMsg];
end;

function TRAG.FormatContext(
  const ADocuments: TArray<TSimilarityResult>): string;
var
  Parts: TStringList;
  I: Integer;
  SR: TSimilarityResult;
begin
  Parts := TStringList.Create;
  try
    for I := 0 to High(ADocuments) do
    begin
      SR := ADocuments[I];
      Parts.Add(Format('[Document %d] (relevance: %.0f%%)',
        [I + 1, SR.Score * 100]));
      Parts.Add(SR.Document.Content);
      Parts.Add('');
    end;
    Result := Parts.Text;
  finally
    Parts.Free;
  end;
end;

procedure TRAG.IngestText(const AId, AText: string;
  const AMetadata: TStringList);
var
  Doc: TVectorDocument;
  Chunks: TArray<string>;
  Chunk: string;
  I: Integer;
begin
  // Split text into chunks
  Chunks := SplitIntoChunks(AText, 500, 50); // 500 chars, 50 overlap

  for I := 0 to High(Chunks) do
  begin
    Chunk := Chunks[I];
    Doc := TVectorDocument.Create;
    try
      Doc.Id := Format('%s_chunk_%d', [AId, I]);
      Doc.Content := Chunk;

      // Generate embedding
      Doc.Embedding := FLlm.Embed(Chunk);

      if Assigned(AMetadata) then
      begin
        Doc.Metadata.AddStrings(AMetadata);
        Doc.Metadata.Values['source_id'] := AId;
        Doc.Metadata.Values['chunk_index'] := IntToStr(I);
      end;

      FVectorStore.Add(Doc);
    except
      Doc.Free;
      raise;
    end;
  end;

  FLogger.Info(Format('Ingested %d chunks from "%s"', [Length(Chunks), AId]));
end;

end.
```

## 4. Image Classification with ONNX Runtime

```pascal
// uImageClassifier.pas - Image Classification with ONNX
unit uImageClassifier;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Graphics, FPImage, FPReadJPEG, FPReadPNG;

type
  TClassificationResult = record
    Label: string;
    Score: Double;
    Index: Integer;
  end;

  TImageClassifier = class
  private
    FModelPath: string;
    FLabels: TArray<string>;
    FInputWidth: Integer;
    FInputHeight: Integer;
    FInputChannels: Integer;

    // ONNX Runtime handles (using ctypes/DLL)
    FOrtEnv: Pointer;
    FOrtSession: Pointer;
    FOrtOptions: Pointer;

    procedure InitOnnxRuntime;
    function PreprocessImage(const AImage: TBitmap): TArray<Single>;
    function Softmax(const ALogits: TArray<Single>): TArray<Single>;
    procedure LoadLabels(const ALabelsFile: string);

  public
    constructor Create(const AModelPath, ALabelsFile: string;
      AInputWidth: Integer = 224;
      AInputHeight: Integer = 224);
    destructor Destroy; override;

    function ClassifyImage(const AImagePath: string): TArray<TClassificationResult>;
    function ClassifyImageData(const AImageData: TArray<Byte>): TArray<TClassificationResult>;
    function ClassifyBitmap(const ABitmap: TBitmap): TArray<TClassificationResult>;
    function GetTopClass(const AImagePath: string): TClassificationResult;
  end;

implementation

// Image classification using ONNX Runtime DLL (onnxruntime.dll/so)
procedure TImageClassifier.InitOnnxRuntime;
type
  TOrtCreateEnv = function(ALogLevel: Integer; ALogId: PChar;
    out AEnv: Pointer): Integer; cdecl;
  TOrtCreateSession = function(AEnv: Pointer; AModelPath: PChar;
    AOptions: Pointer; out ASession: Pointer): Integer; cdecl;
var
  OrtLib: THandle;
  OrtCreateEnv: TOrtCreateEnv;
  OrtCreateSession: TOrtCreateSession;
begin
  {$IFDEF WINDOWS}
  OrtLib := LoadLibrary('onnxruntime.dll');
  {$ELSE}
  OrtLib := LoadLibrary('libonnxruntime.so');
  {$ENDIF}

  if OrtLib = 0 then
    raise Exception.Create('Cannot load ONNX Runtime library');

  OrtCreateEnv := GetProcAddress(OrtLib, 'OrtCreateEnv');
  OrtCreateSession := GetProcAddress(OrtLib, 'OrtCreateSession');

  // Initialize
  OrtCreateEnv(1, 'ImageClassifier', FOrtEnv); // ORT_LOGGING_LEVEL_WARNING = 1
  OrtCreateSession(FOrtEnv, PChar(FModelPath), nil, FOrtSession);
end;

function TImageClassifier.PreprocessImage(
  const AImage: TBitmap): TArray<Single>;
var
  ResizedImg: TBitmap;
  X, Y: Integer;
  Pixel: TColor;
  R, G, B: Byte;
  Idx: Integer;
const
  // ImageNet normalization
  MEAN: array[0..2] of Single = (0.485, 0.456, 0.406);
  STD: array[0..2] of Single = (0.229, 0.224, 0.225);
begin
  // Resize image to model input size
  ResizedImg := TBitmap.Create;
  try
    ResizedImg.Width := FInputWidth;
    ResizedImg.Height := FInputHeight;
    ResizedImg.Canvas.StretchDraw(
      Rect(0, 0, FInputWidth, FInputHeight),
      AImage
    );

    // Convert to float tensor [C, H, W] format
    SetLength(Result, FInputChannels * FInputHeight * FInputWidth);

    for Y := 0 to FInputHeight - 1 do
    begin
      for X := 0 to FInputWidth - 1 do
      begin
        Pixel := ResizedImg.Canvas.Pixels[X, Y];
        R := GetRValue(Pixel);
        G := GetGValue(Pixel);
        B := GetBValue(Pixel);

        // Normalize: (value/255 - mean) / std
        Idx := Y * FInputWidth + X;
        // R channel
        Result[Idx] := ((R / 255.0) - MEAN[0]) / STD[0];
        // G channel
        Result[FInputHeight * FInputWidth + Idx] :=
          ((G / 255.0) - MEAN[1]) / STD[1];
        // B channel
        Result[2 * FInputHeight * FInputWidth + Idx] :=
          ((B / 255.0) - MEAN[2]) / STD[2];
      end;
    end;
  finally
    ResizedImg.Free;
  end;
end;

function TImageClassifier.Softmax(const ALogits: TArray<Single>): TArray<Single>;
var
  MaxVal: Single;
  SumExp: Single;
  I: Integer;
begin
  SetLength(Result, Length(ALogits));

  // Find max for numerical stability
  MaxVal := ALogits[0];
  for I := 1 to High(ALogits) do
    if ALogits[I] > MaxVal then MaxVal := ALogits[I];

  // Compute softmax
  SumExp := 0;
  for I := 0 to High(ALogits) do
  begin
    Result[I] := Exp(ALogits[I] - MaxVal);
    SumExp := SumExp + Result[I];
  end;

  for I := 0 to High(Result) do
    Result[I] := Result[I] / SumExp;
end;

end.
```

## 5. Text Analysis Application

```pascal
// TextAnalyzer/uTextAnalyzer.pas - NLP with LLM
unit uTextAnalyzer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, Generics.Collections, uLLMClient;

type
  TSentiment = (stPositive, stNegative, stNeutral, stMixed);

  TSentimentResult = record
    Sentiment: TSentiment;
    Score: Double; // -1 to 1
    Explanation: string;
    Confidence: Double;
  end;

  TEntityType = (etPerson, etOrganization, etLocation, etDate,
    etAmount, etProduct, etEvent);

  TEntity = class
  public
    Text: string;
    EntityType: TEntityType;
    StartPos: Integer;
    EndPos: Integer;
    Confidence: Double;
  end;

  TTextClassification = class
  public
    Label: string;
    Score: Double;
    SubLabels: TList<string>;

    constructor Create;
    destructor Destroy; override;
  end;

  TTextAnalyzer = class
  private
    FLlm: ILLMClient;
    FSystemPrompt: string;

    function ParseJsonArray(const AJson: string): TJSONArray;
    function CallLlm(const APrompt: string): string;

  public
    constructor Create(const ALlm: ILLMClient);

    // Sentiment Analysis
    function AnalyzeSentiment(const AText: string): TSentimentResult;
    function AnalyzeSentimentBatch(
      const ATexts: TArray<string>): TArray<TSentimentResult>;

    // Named Entity Recognition
    function ExtractEntities(const AText: string): TList<TEntity>;

    // Text Classification
    function Classify(const AText: string;
      const ACategories: TArray<string>): TTextClassification;

    // Text Summarization
    function Summarize(const AText: string;
      AMaxWords: Integer = 100): string;

    // Keyword Extraction
    function ExtractKeywords(const AText: string;
      ATopN: Integer = 10): TArray<string>;

    // Language Detection
    function DetectLanguage(const AText: string): string;

    // Translation
    function Translate(const AText, ATargetLanguage: string): string;
  end;

implementation

function TTextAnalyzer.AnalyzeSentiment(
  const AText: string): TSentimentResult;
var
  Prompt: string;
  Response: string;
  Json: TJSONObject;
  SentimentStr: string;
begin
  Prompt := Format(
    'Analyze the sentiment of the following text. ' +
    'Respond with a JSON object containing:' + #13#10 +
    '- "sentiment": "positive", "negative", "neutral", or "mixed"' + #13#10 +
    '- "score": number from -1 (very negative) to 1 (very positive)' + #13#10 +
    '- "explanation": brief explanation in Thai' + #13#10 +
    '- "confidence": number from 0 to 1' + #13#10#13#10 +
    'Text: "%s"' + #13#10#13#10 +
    'Respond with JSON only, no other text.', [AText]);

  Response := CallLlm(Prompt);

  // Remove markdown code blocks if present
  Response := Response.Replace('```json', '').Replace('```', '').Trim;

  Json := TJSONObject(GetJSON(Response));
  try
    SentimentStr := LowerCase(Json.Get('sentiment', 'neutral'));

    if SentimentStr = 'positive' then Result.Sentiment := stPositive
    else if SentimentStr = 'negative' then Result.Sentiment := stNegative
    else if SentimentStr = 'mixed' then Result.Sentiment := stMixed
    else Result.Sentiment := stNeutral;

    Result.Score := Json.Get('score', 0.0);
    Result.Explanation := Json.Get('explanation', '');
    Result.Confidence := Json.Get('confidence', 0.5);
  finally
    Json.Free;
  end;
end;

function TTextAnalyzer.ExtractEntities(const AText: string): TList<TEntity>;
var
  Prompt: string;
  Response: string;
  JsonArray: TJSONArray;
  EntityObj: TJSONObject;
  Entity: TEntity;
  I: Integer;
const
  EntityTypeNames: array[0..6] of string =
    ('person', 'organization', 'location', 'date',
     'amount', 'product', 'event');
begin
  Result := TList<TEntity>.Create;

  Prompt := Format(
    'Extract named entities from the following text. ' +
    'Return a JSON array of objects with:' + #13#10 +
    '- "text": the entity text' + #13#10 +
    '- "type": one of person, organization, location, date, amount, product, event' + #13#10 +
    '- "confidence": 0 to 1' + #13#10#13#10 +
    'Text: "%s"' + #13#10#13#10 +
    'Return JSON array only.', [AText]);

  Response := CallLlm(Prompt);
  Response := Response.Replace('```json', '').Replace('```', '').Trim;

  try
    JsonArray := TJSONArray(GetJSON(Response));
    try
      for I := 0 to JsonArray.Count - 1 do
      begin
        EntityObj := TJSONObject(JsonArray[I]);
        Entity := TEntity.Create;
        Entity.Text := EntityObj.Get('text', '');
        Entity.Confidence := EntityObj.Get('confidence', 0.5);

        var TypeStr := LowerCase(EntityObj.Get('type', 'person'));
        for var J := 0 to 6 do
          if EntityTypeNames[J] = TypeStr then
          begin
            Entity.EntityType := TEntityType(J);
            Break;
          end;

        Result.Add(Entity);
      end;
    finally
      JsonArray.Free;
    end;
  except
    // Return empty list if parsing fails
  end;
end;

function TTextAnalyzer.Summarize(const AText: string;
  AMaxWords: Integer): string;
var
  Prompt: string;
  Messages: TArray<TChatMessage>;
  Completion: TCompletionResult;
begin
  Prompt := Format(
    'Please summarize the following text in Thai language, ' +
    'using no more than %d words. ' +
    'Focus on the key points and main ideas.' + #13#10#13#10 +
    '%s', [AMaxWords, AText]);

  Messages := [
    ChatMessage(mrSystem,
      'You are a helpful assistant that summarizes text in Thai.'),
    ChatMessage(mrUser, Prompt)
  ];

  Completion := FLlm.Complete(Messages);
  try
    Result := Completion.Content;
  finally
    Completion.Free;
  end;
end;

function TTextAnalyzer.CallLlm(const APrompt: string): string;
var
  Messages: TArray<TChatMessage>;
  Completion: TCompletionResult;
begin
  Messages := [
    ChatMessage(mrSystem, FSystemPrompt),
    ChatMessage(mrUser, APrompt)
  ];

  Completion := FLlm.Complete(Messages);
  try
    Result := Completion.Content;
  finally
    Completion.Free;
  end;
end;

end.
```

## 6. Complete AI-Powered App Example

```pascal
// AIAssistant/AIAssistant.pas - Complete AI Application
program AIAssistant;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes, fpjson,
  uLLMClient, uVectorSearch, uRAG, uTextAnalyzer,
  uConfiguration;

var
  LlmConfig: TLLMConfig;
  LlmClient: ILLMClient;
  VectorStore: IVectorStore;
  RAG: TRAG;
  Analyzer: TTextAnalyzer;
  Input: string;

procedure SetupSystem;
var
  RAGConfig: TRAGConfig;
begin
  // Configure LLM
  LlmConfig.ApiKey := TAppConfiguration.GetInstance.Get('ANTHROPIC_API_KEY');
  LlmConfig.BaseUrl := 'https://api.anthropic.com/v1';
  LlmConfig.Model := 'claude-3-haiku-20240307';
  LlmConfig.MaxTokens := 2048;
  LlmConfig.Temperature := 0.7;
  LlmConfig.TimeoutMs := 30000;

  LlmClient := TAnthropicClient.Create(LlmConfig);

  // Setup Vector Store
  VectorStore := TInMemoryVectorStore.Create(LlmClient);

  // Ingest knowledge base
  RAG := TRAG.Create(LlmClient, VectorStore, RAGConfig);
  RAG.IngestDirectory('./knowledge_base', '*.txt');

  // Text Analyzer
  Analyzer := TTextAnalyzer.Create(LlmClient);
end;

procedure RunInteractiveMode;
begin
  WriteLn('=== AI Assistant (พิมพ์ "quit" เพื่อออก) ===');

  repeat
    Write('คุณ: ');
    ReadLn(Input);

    if LowerCase(Trim(Input)) = 'quit' then Break;

    if Input.StartsWith('/analyze ') then
    begin
      // Analyze sentiment
      var Text := Copy(Input, 10, MaxInt);
      var Sentiment := Analyzer.AnalyzeSentiment(Text);
      WriteLn(Format('Sentiment: %s (Score: %.2f)',
        [SentimentNames[Sentiment.Sentiment], Sentiment.Score]));
      WriteLn('Explanation: ', Sentiment.Explanation);
    end
    else if Input.StartsWith('/summarize ') then
    begin
      var Text := Copy(Input, 12, MaxInt);
      var Summary := Analyzer.Summarize(Text, 50);
      WriteLn('Summary: ', Summary);
    end
    else
    begin
      // RAG Query
      var Response := RAG.Query(Input);
      try
        WriteLn('AI: ', Response.Answer);
        if Response.Sources.Count > 0 then
        begin
          WriteLn(Format('(อ้างอิงจาก %d แหล่งข้อมูล)',
            [Response.Sources.Count]));
        end;
      finally
        Response.Free;
      end;
    end;

    WriteLn;
  until False;
end;

begin
  WriteLn('Initializing AI Assistant...');
  TAppConfiguration.Initialize;
  SetupSystem;
  WriteLn('Ready!');
  WriteLn;
  RunInteractiveMode;

  WriteLn('Goodbye!');
end.
```

## 7. สรุป AI/ML Integration

**วิธีการใช้ AI ใน Pascal Applications:**
1. **LLM APIs** - เรียก Claude/OpenAI สำหรับ Text Generation
2. **Embeddings** - Vector Search สำหรับ Semantic Search
3. **RAG** - Retrieval Augmented Generation
4. **ONNX Runtime** - สำหรับ Image/Audio Processing
5. **External ML Services** - Azure AI, Google Cloud AI, AWS

**Use Cases:**
- Chatbot / Virtual Assistant
- Document Search และ QA
- Sentiment Analysis ของ Customer Reviews
- Auto-categorization ของ Support Tickets
- Image Recognition สำหรับ Product Catalog
- Text Extraction จาก Documents

**Best Practices:**
1. Cache embeddings เพื่อประหยัด API calls
2. Rate limiting สำหรับ API calls
3. Fallback เมื่อ AI ไม่สามารถตอบได้
4. Monitor AI response quality
5. Keep humans in the loop สำหรับ critical decisions
