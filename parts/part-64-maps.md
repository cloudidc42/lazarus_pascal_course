# ตอนที่ 64: Maps Integration ใน Lazarus/Pascal

## บทนำ

การรวม Map เข้ากับแอปพลิเคชัน Lazarus ทำได้หลายวิธี บทนี้จะครอบคลุมการใช้ OpenStreetMap ผ่าน WebView, การจัดการพิกัด, geocoding, routing และการแสดง markers

---

## 64.1 ใช้ WebView กับ Leaflet.js

วิธีที่ง่ายที่สุดในการแสดง Map คือใช้ TWebBrowser/TChromiumEmbedded เปิด HTML ที่มี Leaflet.js

### ไฟล์ HTML สำหรับ Map

```html
<!-- map.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Lazarus Map</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        #map { width: 100vw; height: 100vh; }
        .custom-popup { min-width: 200px; }
        .custom-popup h3 { color: #333; margin-bottom: 5px; }
        .custom-popup p { color: #666; font-size: 13px; }
        .marker-cluster-small { background-color: rgba(181,226,140,0.6); }
        .marker-cluster-medium { background-color: rgba(241,211,87,0.6); }
        .marker-cluster-large { background-color: rgba(253,156,115,0.6); }
    </style>
</head>
<body>
    <div id="map"></div>
    
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // สร้าง map
        var map = L.map('map').setView([13.7563, 100.5018], 12); // กรุงเทพฯ
        
        // Tile layer - OpenStreetMap
        var osmLayer = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '© OpenStreetMap contributors',
            maxZoom: 19
        });
        
        // Tile layer - Satellite (Esri)
        var satelliteLayer = L.tileLayer(
            'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
            { attribution: 'Tiles © Esri' }
        );
        
        osmLayer.addTo(map);
        
        // Layer control
        var baseLayers = {
            "OpenStreetMap": osmLayer,
            "Satellite": satelliteLayer
        };
        L.control.layers(baseLayers).addTo(map);
        
        // ตัวแปรสำหรับ markers
        var markers = {};
        var routes = {};
        var currentRoute = null;
        
        // ฟังก์ชันที่ Pascal จะเรียกผ่าน WebBrowser
        
        // เพิ่ม Marker
        function addMarker(id, lat, lng, title, info, color) {
            if (!color) color = 'blue';
            
            var icon = L.divIcon({
                className: 'custom-marker',
                html: '<div style="background:' + color + ';width:20px;height:20px;border-radius:50%;border:2px solid white;"></div>',
                iconSize: [20, 20],
                iconAnchor: [10, 10]
            });
            
            var marker = L.marker([lat, lng], {icon: icon});
            marker.bindPopup(
                '<div class="custom-popup">' +
                '<h3>' + title + '</h3>' +
                '<p>' + info + '</p>' +
                '<p><small>พิกัด: ' + lat.toFixed(6) + ', ' + lng.toFixed(6) + '</small></p>' +
                '</div>'
            );
            
            marker.on('click', function() {
                // ส่งข้อมูลกลับ Pascal
                if (window.external && window.external.onMarkerClick) {
                    window.external.onMarkerClick(id, lat, lng, title);
                }
            });
            
            markers[id] = marker;
            marker.addTo(map);
            return id;
        }
        
        // ลบ Marker
        function removeMarker(id) {
            if (markers[id]) {
                map.removeLayer(markers[id]);
                delete markers[id];
            }
        }
        
        // ล้าง Markers ทั้งหมด
        function clearMarkers() {
            for (var id in markers) {
                map.removeLayer(markers[id]);
            }
            markers = {};
        }
        
        // ย้ายแผนที่ไปพิกัดที่กำหนด
        function moveTo(lat, lng, zoom) {
            if (!zoom) zoom = 15;
            map.setView([lat, lng], zoom);
        }
        
        // วาดเส้นทาง
        function drawRoute(points, color) {
            if (currentRoute) {
                map.removeLayer(currentRoute);
            }
            if (!color) color = '#0044FF';
            
            var latlngs = points.map(function(p) { return [p.lat, p.lng]; });
            currentRoute = L.polyline(latlngs, {
                color: color,
                weight: 4,
                opacity: 0.8
            });
            currentRoute.addTo(map);
            map.fitBounds(currentRoute.getBounds());
        }
        
        // วาด Circle
        function drawCircle(lat, lng, radius, color) {
            L.circle([lat, lng], {
                color: color || 'red',
                fillColor: color || '#f03',
                fillOpacity: 0.2,
                radius: radius
            }).addTo(map);
        }
        
        // วาด Polygon
        function drawPolygon(points, color) {
            var latlngs = points.map(function(p) { return [p.lat, p.lng]; });
            L.polygon(latlngs, {
                color: color || 'green',
                fillOpacity: 0.2
            }).addTo(map);
        }
        
        // รับตำแหน่งปัจจุบัน
        function getCurrentLocation() {
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    function(pos) {
                        var lat = pos.coords.latitude;
                        var lng = pos.coords.longitude;
                        moveTo(lat, lng, 15);
                        addMarker('current', lat, lng, 'ตำแหน่งของคุณ', 'Here you are!', 'green');
                        
                        if (window.external && window.external.onLocationReceived) {
                            window.external.onLocationReceived(lat, lng);
                        }
                    },
                    function(err) {
                        console.error('Geolocation error:', err);
                    }
                );
            }
        }
        
        // คลิกที่แผนที่
        map.on('click', function(e) {
            if (window.external && window.external.onMapClick) {
                window.external.onMapClick(e.latlng.lat, e.latlng.lng);
            }
        });
        
        // เริ่มต้น
        console.log('Map initialized');
    </script>
</body>
</html>
```

---

## 64.2 Pascal Component สำหรับ Map

```pascal
unit map_component;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls,
  OleServer,     // สำหรับ WebBrowser
  SHDocVw;       // TWebBrowser (Windows)

type
  TMapMarker = record
    ID: string;
    Lat, Lng: Double;
    Title: string;
    Info: string;
    Color: string;
  end;

  TMapClickEvent = procedure(Sender: TObject; Lat, Lng: Double) of object;
  TMarkerClickEvent = procedure(Sender: TObject; const ID: string; 
    Lat, Lng: Double; const Title: string) of object;

  TMapControl = class(TWinControl)
  private
    FBrowser: TWebBrowser;
    FMarkers: TList;
    FOnMapClick: TMapClickEvent;
    FOnMarkerClick: TMarkerClickEvent;
    FZoom: Integer;
    FLatitude, FLongitude: Double;
    FMapFile: string;
    
    procedure SetLatitude(const Value: Double);
    procedure SetLongitude(const Value: Double);
    procedure SetZoom(const Value: Integer);
    
  protected
    procedure CreateWnd; override;
    procedure Resize; override;
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    // โหลด Map
    procedure LoadMap(const AHTMLFile: string = '');
    
    // Marker operations
    function AddMarker(const AID: string; ALat, ALng: Double;
      const ATitle, AInfo: string; const AColor: string = 'blue'): string;
    procedure RemoveMarker(const AID: string);
    procedure ClearMarkers;
    
    // Navigation
    procedure MoveTo(ALat, ALng: Double; AZoom: Integer = 0);
    procedure FitBounds(ALatMin, ALngMin, ALatMax, ALngMax: Double);
    
    // Drawing
    procedure DrawRoute(const APoints: array of TMapMarker; const AColor: string = '');
    procedure DrawCircle(ALat, ALng, ARadius: Double; const AColor: string = 'red');
    
    // Geocoding (ต้องการ internet)
    function Geocode(const AAddress: string; out ALat, ALng: Double): Boolean;
    
    // JavaScript execution
    procedure ExecuteScript(const AScript: string);
    function EvalScript(const AScript: string): string;
    
    property Latitude: Double read FLatitude write SetLatitude;
    property Longitude: Double read FLongitude write SetLongitude;
    property Zoom: Integer read FZoom write SetZoom;
    property OnMapClick: TMapClickEvent read FOnMapClick write FOnMapClick;
    property OnMarkerClick: TMarkerClickEvent read FOnMarkerClick write FOnMarkerClick;
  end;

implementation

uses
  ActiveX, ComObj;

constructor TMapControl.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FMarkers := TList.Create;
  FZoom := 12;
  FLatitude := 13.7563;   // กรุงเทพฯ
  FLongitude := 100.5018;
end;

destructor TMapControl.Destroy;
begin
  ClearMarkers;
  FMarkers.Free;
  inherited Destroy;
end;

procedure TMapControl.CreateWnd;
begin
  inherited CreateWnd;
  
  FBrowser := TWebBrowser.Create(Self);
  FBrowser.Parent := Self;
  FBrowser.Align := alClient;
  FBrowser.OnDocumentComplete := @OnDocumentComplete;
  
  LoadMap;
end;

procedure TMapControl.Resize;
begin
  inherited Resize;
  if Assigned(FBrowser) then
  begin
    FBrowser.Width := Width;
    FBrowser.Height := Height;
  end;
end;

procedure TMapControl.LoadMap(const AHTMLFile: string);
var
  MapFile: string;
begin
  if AHTMLFile <> '' then
    MapFile := AHTMLFile
  else
  begin
    // สร้างไฟล์ HTML ชั่วคราว
    MapFile := GetTempDir + 'lazarus_map.html';
    // เขียน HTML template ลงไฟล์
    // (ใช้ content จาก section 64.1)
    WriteMapHTML(MapFile);
  end;
  
  FMapFile := MapFile;
  FBrowser.Navigate('file://' + MapFile);
end;

procedure TMapControl.ExecuteScript(const AScript: string);
begin
  if Assigned(FBrowser) and Assigned(FBrowser.Document) then
  begin
    // รัน JavaScript
    (FBrowser.Document as IHTMLDocument2).parentWindow.execScript(
      WideString(AScript), 'JScript');
  end;
end;

function TMapControl.AddMarker(const AID: string; ALat, ALng: Double;
  const ATitle, AInfo: string; const AColor: string): string;
var
  Script: string;
  Marker: ^TMapMarker;
begin
  New(Marker);
  Marker^.ID := AID;
  Marker^.Lat := ALat;
  Marker^.Lng := ALng;
  Marker^.Title := ATitle;
  Marker^.Info := AInfo;
  Marker^.Color := AColor;
  FMarkers.Add(Marker);
  
  Script := Format('addMarker("%s", %s, %s, "%s", "%s", "%s")',
    [AID, 
     FormatFloat('0.######', ALat),
     FormatFloat('0.######', ALng),
     ATitle, AInfo, AColor]);
     
  ExecuteScript(Script);
  Result := AID;
end;

procedure TMapControl.RemoveMarker(const AID: string);
begin
  ExecuteScript(Format('removeMarker("%s")', [AID]));
end;

procedure TMapControl.ClearMarkers;
var
  i: Integer;
begin
  for i := 0 to FMarkers.Count - 1 do
    Dispose(PMapMarker(FMarkers[i]));
  FMarkers.Clear;
  ExecuteScript('clearMarkers()');
end;

procedure TMapControl.MoveTo(ALat, ALng: Double; AZoom: Integer);
var
  Z: Integer;
begin
  FLatitude := ALat;
  FLongitude := ALng;
  if AZoom > 0 then
    FZoom := AZoom;
    
  Z := FZoom;
  ExecuteScript(Format('moveTo(%s, %s, %d)',
    [FormatFloat('0.######', ALat),
     FormatFloat('0.######', ALng), Z]));
end;

function TMapControl.Geocode(const AAddress: string; out ALat, ALng: Double): Boolean;
var
  URL: string;
  Response: string;
begin
  Result := False;
  ALat := 0;
  ALng := 0;
  
  // ใช้ Nominatim (OpenStreetMap geocoder) - ฟรี
  URL := 'https://nominatim.openstreetmap.org/search?q=' + 
    URLEncode(AAddress) + '&format=json&limit=1';
    
  // HTTP GET request (ต้องใช้ synapse หรือ indy)
  // Response := HTTPGet(URL);
  // ParseJSON(Response, ALat, ALng);
  
  WriteLn('Geocoding: ', AAddress);
  WriteLn('URL: ', URL);
  // จำลองผลลัพธ์
  ALat := 13.7563;
  ALng := 100.5018;
  Result := True;
end;

procedure TMapControl.DrawRoute(const APoints: array of TMapMarker; 
  const AColor: string);
var
  PointsJS: string;
  i: Integer;
begin
  PointsJS := '[';
  for i := 0 to High(APoints) do
  begin
    if i > 0 then PointsJS := PointsJS + ',';
    PointsJS := PointsJS + Format('{lat:%s,lng:%s}',
      [FormatFloat('0.######', APoints[i].Lat),
       FormatFloat('0.######', APoints[i].Lng)]);
  end;
  PointsJS := PointsJS + ']';
  
  ExecuteScript(Format('drawRoute(%s, "%s")', [PointsJS, AColor]));
end;

procedure TMapControl.DrawCircle(ALat, ALng, ARadius: Double; const AColor: string);
begin
  ExecuteScript(Format('drawCircle(%s, %s, %s, "%s")',
    [FormatFloat('0.######', ALat),
     FormatFloat('0.######', ALng),
     FormatFloat('0.##', ARadius),
     AColor]));
end;

procedure TMapControl.SetLatitude(const Value: Double);
begin
  FLatitude := Value;
  MoveTo(FLatitude, FLongitude);
end;

procedure TMapControl.SetLongitude(const Value: Double);
begin
  FLongitude := Value;
  MoveTo(FLatitude, FLongitude);
end;

procedure TMapControl.SetZoom(const Value: Integer);
begin
  FZoom := Value;
  MoveTo(FLatitude, FLongitude, FZoom);
end;

end.
```

---

## 64.3 Geocoding Service

```pascal
unit geocoding;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, HTTPSend, SSL_OpenSSL,
  fpjson, jsonparser;

type
  TGeoLocation = record
    Lat: Double;
    Lng: Double;
    DisplayName: string;
    Country: string;
    City: string;
    PostCode: string;
    Found: Boolean;
  end;

  TRouteInfo = record
    Distance: Double;  // เมตร
    Duration: Double;  // วินาที
    Points: array of TGeoLocation;
  end;

  TGeocoder = class
  private
    FUserAgent: string;
    FCache: TStringList;
    
    function HTTPGet(const AURL: string): string;
    function ParseNominatim(const AJSON: string): TGeoLocation;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    // Forward geocoding (ที่อยู่ -> พิกัด)
    function Geocode(const AAddress: string): TGeoLocation;
    function GeocodeThailand(const AAddress: string): TGeoLocation;
    
    // Reverse geocoding (พิกัด -> ที่อยู่)
    function ReverseGeocode(ALat, ALng: Double): TGeoLocation;
    
    // หาระยะทาง
    function Distance(ALat1, ALng1, ALat2, ALng2: Double): Double;
    
    // แปลงพิกัดเป็น DMS format
    class function ToDMS(ADecimal: Double; AIsLat: Boolean): string;
    
    // แปลง DMS เป็น Decimal
    class function FromDMS(const ADMS: string): Double;
    
    property UserAgent: string read FUserAgent write FUserAgent;
  end;

implementation

uses
  Math;

constructor TGeocoder.Create;
begin
  inherited Create;
  FUserAgent := 'LazarusPascalApp/1.0';
  FCache := TStringList.Create;
end;

destructor TGeocoder.Destroy;
begin
  FCache.Free;
  inherited Destroy;
end;

function TGeocoder.HTTPGet(const AURL: string): string;
var
  HTTP: THTTPSend;
begin
  Result := '';
  HTTP := THTTPSend.Create;
  try
    HTTP.Headers.Add('User-Agent: ' + FUserAgent);
    HTTP.Headers.Add('Accept: application/json');
    
    if HTTP.HTTPMethod('GET', AURL) then
    begin
      if HTTP.ResultCode = 200 then
      begin
        SetLength(Result, HTTP.Document.Size);
        HTTP.Document.Read(Result[1], HTTP.Document.Size);
      end;
    end;
  finally
    HTTP.Free;
  end;
end;

function TGeocoder.ParseNominatim(const AJSON: string): TGeoLocation;
var
  Parser: TJSONParser;
  Data: TJSONData;
  Obj: TJSONObject;
  Arr: TJSONArray;
  Address: TJSONObject;
begin
  Result.Found := False;
  Result.Lat := 0;
  Result.Lng := 0;
  
  if AJSON = '' then Exit;
  
  Parser := TJSONParser.Create(AJSON, []);
  try
    Data := Parser.Parse;
    try
      if Data is TJSONArray then
      begin
        Arr := TJSONArray(Data);
        if Arr.Count > 0 then
        begin
          Obj := TJSONObject(Arr[0]);
          Result.Lat := StrToFloatDef(Obj.Get('lat', '0'), 0);
          Result.Lng := StrToFloatDef(Obj.Get('lon', '0'), 0);
          Result.DisplayName := Obj.Get('display_name', '');
          
          if Obj.Find('address') <> nil then
          begin
            Address := TJSONObject(Obj.Find('address'));
            Result.Country := Address.Get('country', '');
            Result.City := Address.Get('city', Address.Get('town', 
              Address.Get('village', '')));
            Result.PostCode := Address.Get('postcode', '');
          end;
          
          Result.Found := True;
        end;
      end;
    finally
      Data.Free;
    end;
  finally
    Parser.Free;
  end;
end;

function TGeocoder.Geocode(const AAddress: string): TGeoLocation;
var
  URL, CachedValue: string;
  Response: string;
  CacheKey: string;
begin
  CacheKey := LowerCase(AAddress);
  
  // ตรวจสอบ cache
  CachedValue := FCache.Values[CacheKey];
  if CachedValue <> '' then
  begin
    // Parse cached value
    Result.Found := True;
    // ... parse from cache
    Exit;
  end;
  
  URL := 'https://nominatim.openstreetmap.org/search?q=' + 
    StringReplace(AAddress, ' ', '+', [rfReplaceAll]) + 
    '&format=json&limit=1&accept-language=th';
    
  Response := HTTPGet(URL);
  Result := ParseNominatim(Response);
  
  if Result.Found then
  begin
    // บันทึก cache
    FCache.Values[CacheKey] := FormatFloat('0.######', Result.Lat) + 
      ',' + FormatFloat('0.######', Result.Lng);
  end;
end;

function TGeocoder.GeocodeThailand(const AAddress: string): TGeoLocation;
begin
  // เพิ่ม "Thailand" เพื่อให้ผลลัพธ์แม่นยำขึ้น
  Result := Geocode(AAddress + ', Thailand');
end;

function TGeocoder.ReverseGeocode(ALat, ALng: Double): TGeoLocation;
var
  URL, Response: string;
  Parser: TJSONParser;
  Data: TJSONData;
  Obj: TJSONObject;
  Address: TJSONObject;
begin
  Result.Found := False;
  Result.Lat := ALat;
  Result.Lng := ALng;
  
  URL := Format('https://nominatim.openstreetmap.org/reverse?lat=%s&lon=%s&format=json',
    [FormatFloat('0.######', ALat), FormatFloat('0.######', ALng)]);
    
  Response := HTTPGet(URL);
  
  if Response = '' then Exit;
  
  Parser := TJSONParser.Create(Response, []);
  try
    Data := Parser.Parse;
    try
      if Data is TJSONObject then
      begin
        Obj := TJSONObject(Data);
        Result.DisplayName := Obj.Get('display_name', '');
        
        if Obj.Find('address') <> nil then
        begin
          Address := TJSONObject(Obj.Find('address'));
          Result.Country := Address.Get('country', '');
          Result.City := Address.Get('city', Address.Get('town', ''));
          Result.PostCode := Address.Get('postcode', '');
        end;
        
        Result.Found := True;
      end;
    finally
      Data.Free;
    end;
  finally
    Parser.Free;
  end;
end;

function TGeocoder.Distance(ALat1, ALng1, ALat2, ALng2: Double): Double;
const
  R = 6371000; // รัศมีโลก (เมตร)
var
  Phi1, Phi2: Double;
  DPhi, DLambda: Double;
  A, C: Double;
begin
  // Haversine formula
  Phi1 := DegToRad(ALat1);
  Phi2 := DegToRad(ALat2);
  DPhi := DegToRad(ALat2 - ALat1);
  DLambda := DegToRad(ALng2 - ALng1);
  
  A := Sin(DPhi/2) * Sin(DPhi/2) +
       Cos(Phi1) * Cos(Phi2) *
       Sin(DLambda/2) * Sin(DLambda/2);
       
  C := 2 * ArcTan2(Sqrt(A), Sqrt(1-A));
  Result := R * C;
end;

class function TGeocoder.ToDMS(ADecimal: Double; AIsLat: Boolean): string;
var
  Degrees, Minutes: Integer;
  Seconds: Double;
  Dir: string;
begin
  if ADecimal < 0 then
  begin
    if AIsLat then Dir := 'S' else Dir := 'W';
    ADecimal := Abs(ADecimal);
  end
  else
  begin
    if AIsLat then Dir := 'N' else Dir := 'E';
  end;
  
  Degrees := Trunc(ADecimal);
  Minutes := Trunc((ADecimal - Degrees) * 60);
  Seconds := ((ADecimal - Degrees) * 60 - Minutes) * 60;
  
  Result := Format('%d°%d''%.2f"%s', [Degrees, Minutes, Seconds, Dir]);
end;

class function TGeocoder.FromDMS(const ADMS: string): Double;
var
  Deg, Min: Integer;
  Sec: Double;
  Dir: Char;
  S: string;
begin
  // Parse format: "13°45'22.68"N" or "100°30'6.48"E"
  S := ADMS;
  Dir := S[Length(S)];
  
  // Extract values (simplified)
  Result := 0;
  // ... implementation
  
  if (Dir = 'S') or (Dir = 'W') then
    Result := -Result;
end;

end.
```

---

## 64.4 โปรแกรมตัวอย่างสมบูรณ์: Location App

```pascal
unit location_app;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, StdCtrls,
  ExtCtrls, ComCtrls, Dialogs, Menus,
  OleServer, SHDocVw,
  geocoding;

type
  TLocation = record
    ID: Integer;
    Name: string;
    Address: string;
    Lat: Double;
    Lng: Double;
    Category: string;
    Phone: string;
    Note: string;
  end;

  TLocationApp = class(TForm)
    // Layout
    pnlLeft: TPanel;
    pnlRight: TPanel;
    Splitter1: TSplitter;
    
    // Left panel - controls
    pnlSearch: TPanel;
    edtSearch: TEdit;
    btnSearch: TButton;
    btnCurrentLocation: TButton;
    
    // Location list
    lvLocations: TListView;
    
    // Location form
    pnlLocationForm: TPanel;
    lblName: TLabel;
    edtName: TEdit;
    lblAddress: TLabel;
    edtAddress: TEdit;
    lblCategory: TLabel;
    cmbCategory: TComboBox;
    lblPhone: TLabel;
    edtPhone: TEdit;
    lblNote: TLabel;
    memoNote: TMemo;
    btnSave: TButton;
    btnDelete: TButton;
    btnGeocode: TButton;
    btnRoute: TButton;
    
    // Map (WebBrowser)
    WebBrowser: TWebBrowser;
    
    // Status
    StatusBar: TStatusBar;
    pnlCoords: TPanel;
    lblCoords: TLabel;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnSearchClick(Sender: TObject);
    procedure btnCurrentLocationClick(Sender: TObject);
    procedure lvLocationsSelectItem(Sender: TObject; Item: TListItem; Selected: Boolean);
    procedure btnSaveClick(Sender: TObject);
    procedure btnDeleteClick(Sender: TObject);
    procedure btnGeocodeClick(Sender: TObject);
    procedure btnRouteClick(Sender: TObject);
    procedure WebBrowserDocumentComplete(ASender: TObject; const pDisp: IDispatch; var URL: OleVariant);
    
  private
    FLocations: array of TLocation;
    FGeocoder: TGeocoder;
    FSelectedID: Integer;
    FMapReady: Boolean;
    FNextID: Integer;
    
    procedure LoadMapHTML;
    procedure SetupListView;
    procedure RefreshList;
    procedure DisplayLocation(const ALocation: TLocation);
    procedure AddMarkerToMap(const ALocation: TLocation);
    procedure ShowAllMarkers;
    procedure ExecuteJS(const AScript: string);
    function FindLocation(AID: Integer): Integer;
    procedure SaveLocations;
    procedure LoadLocations;
    procedure LoadSampleLocations;
    
  end;

const
  DATA_FILE = 'locations.dat';

implementation

{$R *.lfm}

procedure TLocationApp.FormCreate(Sender: TObject);
begin
  Caption := 'Location App - แอปพลิเคชันแผนที่';
  Width := 1100;
  Height := 700;
  
  FGeocoder := TGeocoder.Create;
  FSelectedID := -1;
  FMapReady := False;
  FNextID := 1;
  
  SetupListView;
  
  // ตั้งค่า ComboBox categories
  cmbCategory.Items.Clear;
  cmbCategory.Items.Add('ร้านอาหาร');
  cmbCategory.Items.Add('โรงแรม');
  cmbCategory.Items.Add('ห้างสรรพสินค้า');
  cmbCategory.Items.Add('โรงพยาบาล');
  cmbCategory.Items.Add('สถานศึกษา');
  cmbCategory.Items.Add('สถานที่ท่องเที่ยว');
  cmbCategory.Items.Add('อื่นๆ');
  cmbCategory.ItemIndex := 0;
  
  // โหลดข้อมูล
  if FileExists(DATA_FILE) then
    LoadLocations
  else
    LoadSampleLocations;
    
  // โหลด Map
  LoadMapHTML;
  
  lblCoords.Caption := 'คลิกที่แผนที่เพื่อดูพิกัด';
end;

procedure TLocationApp.FormDestroy(Sender: TObject);
begin
  SaveLocations;
  FGeocoder.Free;
end;

procedure TLocationApp.SetupListView;
begin
  lvLocations.ViewStyle := vsReport;
  lvLocations.RowSelect := True;
  lvLocations.ReadOnly := True;
  
  with lvLocations.Columns.Add do begin Caption := 'ชื่อ'; Width := 150; end;
  with lvLocations.Columns.Add do begin Caption := 'หมวดหมู่'; Width := 100; end;
  with lvLocations.Columns.Add do begin Caption := 'ที่อยู่'; Width := 200; end;
end;

procedure TLocationApp.LoadMapHTML;
var
  HTMLFile: string;
  HTML: TStringList;
begin
  HTMLFile := GetTempDir + 'location_map.html';
  
  HTML := TStringList.Create;
  try
    HTML.Add('<!DOCTYPE html>');
    HTML.Add('<html><head>');
    HTML.Add('<meta charset="UTF-8">');
    HTML.Add('<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />');
    HTML.Add('<style>* { margin: 0; padding: 0; } #map { width: 100vw; height: 100vh; }</style>');
    HTML.Add('</head><body>');
    HTML.Add('<div id="map"></div>');
    HTML.Add('<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>');
    HTML.Add('<script>');
    HTML.Add('var map = L.map("map").setView([13.7563, 100.5018], 12);');
    HTML.Add('L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {');
    HTML.Add('  attribution: "© OpenStreetMap contributors"');
    HTML.Add('}).addTo(map);');
    HTML.Add('var markers = {};');
    HTML.Add('');
    HTML.Add('function addMarker(id, lat, lng, name, info, cat) {');
    HTML.Add('  var colors = {');
    HTML.Add('    "ร้านอาหาร": "#E74C3C",');
    HTML.Add('    "โรงแรม": "#3498DB",');
    HTML.Add('    "ห้างสรรพสินค้า": "#9B59B6",');
    HTML.Add('    "โรงพยาบาล": "#2ECC71",');
    HTML.Add('    "สถานศึกษา": "#F39C12"');
    HTML.Add('  };');
    HTML.Add('  var color = colors[cat] || "#95A5A6";');
    HTML.Add('  var icon = L.divIcon({');
    HTML.Add('    className: "",');
    HTML.Add('    html: "<div style=''background:" + color + ";width:12px;height:12px;border-radius:50%;border:2px solid white;box-shadow:0 0 4px rgba(0,0,0,0.5)''></div>",');
    HTML.Add('    iconSize: [16, 16], iconAnchor: [8, 8]');
    HTML.Add('  });');
    HTML.Add('  var m = L.marker([lat, lng], {icon: icon});');
    HTML.Add('  m.bindPopup("<b>" + name + "</b><br><i>" + cat + "</i><br>" + info);');
    HTML.Add('  markers[id] = m;');
    HTML.Add('  m.addTo(map);');
    HTML.Add('}');
    HTML.Add('');
    HTML.Add('function removeMarker(id) { if (markers[id]) { map.removeLayer(markers[id]); delete markers[id]; } }');
    HTML.Add('function clearMarkers() { for (var id in markers) map.removeLayer(markers[id]); markers = {}; }');
    HTML.Add('function moveTo(lat, lng, z) { map.setView([lat, lng], z || 15); }');
    HTML.Add('');
    HTML.Add('map.on("click", function(e) {');
    HTML.Add('  document.title = "MAPCLICK:" + e.latlng.lat.toFixed(6) + "," + e.latlng.lng.toFixed(6);');
    HTML.Add('});');
    HTML.Add('</script>');
    HTML.Add('</body></html>');
    
    HTML.SaveToFile(HTMLFile, TEncoding.UTF8);
  finally
    HTML.Free;
  end;
  
  WebBrowser.Navigate('file:///' + StringReplace(HTMLFile, '\', '/', [rfReplaceAll]));
end;

procedure TLocationApp.WebBrowserDocumentComplete(ASender: TObject; 
  const pDisp: IDispatch; var URL: OleVariant);
begin
  FMapReady := True;
  ShowAllMarkers;
  StatusBar.SimpleText := 'แผนที่พร้อมใช้งาน';
end;

procedure TLocationApp.ExecuteJS(const AScript: string);
begin
  if FMapReady and Assigned(WebBrowser.Document) then
  begin
    try
      (WebBrowser.Document as IHTMLDocument2).parentWindow.execScript(
        WideString(AScript), 'JScript');
    except
      // ignore errors
    end;
  end;
end;

procedure TLocationApp.ShowAllMarkers;
var
  i: Integer;
begin
  ExecuteJS('clearMarkers()');
  
  for i := 0 to High(FLocations) do
  begin
    AddMarkerToMap(FLocations[i]);
  end;
end;

procedure TLocationApp.AddMarkerToMap(const ALocation: TLocation);
var
  Script: string;
begin
  if (ALocation.Lat = 0) and (ALocation.Lng = 0) then Exit;
  
  Script := Format('addMarker(%d, %s, %s, "%s", "%s", "%s")',
    [ALocation.ID,
     FormatFloat('0.######', ALocation.Lat),
     FormatFloat('0.######', ALocation.Lng),
     StringReplace(ALocation.Name, '"', '\\"', [rfReplaceAll]),
     StringReplace(ALocation.Address, '"', '\\"', [rfReplaceAll]),
     ALocation.Category]);
     
  ExecuteJS(Script);
end;

procedure TLocationApp.RefreshList;
var
  i: Integer;
  Item: TListItem;
  SearchTerm: string;
begin
  lvLocations.Items.BeginUpdate;
  try
    lvLocations.Items.Clear;
    SearchTerm := LowerCase(edtSearch.Text);
    
    for i := 0 to High(FLocations) do
    begin
      // กรองตาม search term
      if (SearchTerm <> '') and
         (Pos(SearchTerm, LowerCase(FLocations[i].Name)) = 0) and
         (Pos(SearchTerm, LowerCase(FLocations[i].Address)) = 0) then
        Continue;
        
      Item := lvLocations.Items.Add;
      Item.Caption := FLocations[i].Name;
      Item.SubItems.Add(FLocations[i].Category);
      Item.SubItems.Add(FLocations[i].Address);
      Item.Data := Pointer(FLocations[i].ID);
    end;
  finally
    lvLocations.Items.EndUpdate;
  end;
end;

procedure TLocationApp.DisplayLocation(const ALocation: TLocation);
begin
  edtName.Text := ALocation.Name;
  edtAddress.Text := ALocation.Address;
  edtPhone.Text := ALocation.Phone;
  memoNote.Text := ALocation.Note;
  
  var CatIdx := cmbCategory.Items.IndexOf(ALocation.Category);
  if CatIdx >= 0 then
    cmbCategory.ItemIndex := CatIdx
  else
    cmbCategory.ItemIndex := 0;
    
  if (ALocation.Lat <> 0) or (ALocation.Lng <> 0) then
  begin
    lblCoords.Caption := Format('พิกัด: %.6f, %.6f', [ALocation.Lat, ALocation.Lng]);
    ExecuteJS(Format('moveTo(%s, %s, 15)',
      [FormatFloat('0.######', ALocation.Lat),
       FormatFloat('0.######', ALocation.Lng)]));
  end;
end;

procedure TLocationApp.lvLocationsSelectItem(Sender: TObject; Item: TListItem; 
  Selected: Boolean);
var
  ID, Idx: Integer;
begin
  if not Selected then Exit;
  if not Assigned(Item) then Exit;
  
  ID := Integer(Item.Data);
  Idx := FindLocation(ID);
  
  if Idx >= 0 then
  begin
    FSelectedID := ID;
    DisplayLocation(FLocations[Idx]);
    StatusBar.SimpleText := 'เลือก: ' + FLocations[Idx].Name;
  end;
end;

procedure TLocationApp.btnSaveClick(Sender: TObject);
var
  Idx: Integer;
  Loc: TLocation;
begin
  if edtName.Text = '' then
  begin
    ShowMessage('กรุณาใส่ชื่อสถานที่');
    edtName.SetFocus;
    Exit;
  end;
  
  Idx := FindLocation(FSelectedID);
  
  if Idx >= 0 then
  begin
    // อัปเดต
    with FLocations[Idx] do
    begin
      Name := edtName.Text;
      Address := edtAddress.Text;
      Category := cmbCategory.Text;
      Phone := edtPhone.Text;
      Note := memoNote.Text;
    end;
  end
  else
  begin
    // เพิ่มใหม่
    Loc.ID := FNextID;
    Inc(FNextID);
    Loc.Name := edtName.Text;
    Loc.Address := edtAddress.Text;
    Loc.Category := cmbCategory.Text;
    Loc.Phone := edtPhone.Text;
    Loc.Note := memoNote.Text;
    Loc.Lat := 0;
    Loc.Lng := 0;
    
    SetLength(FLocations, Length(FLocations) + 1);
    FLocations[High(FLocations)] := Loc;
    FSelectedID := Loc.ID;
  end;
  
  RefreshList;
  ShowAllMarkers;
  StatusBar.SimpleText := 'บันทึกสำเร็จ: ' + edtName.Text;
end;

procedure TLocationApp.btnDeleteClick(Sender: TObject);
var
  Idx, i: Integer;
begin
  if FSelectedID < 0 then
  begin
    ShowMessage('กรุณาเลือกสถานที่ก่อน');
    Exit;
  end;
  
  if MessageDlg('ยืนยัน', 'ต้องการลบสถานที่นี้หรือไม่?',
    mtConfirmation, [mbYes, mbNo], 0) <> mrYes then Exit;
    
  Idx := FindLocation(FSelectedID);
  if Idx >= 0 then
  begin
    ExecuteJS('removeMarker(' + IntToStr(FSelectedID) + ')');
    
    // ลบออกจาก array
    for i := Idx to High(FLocations) - 1 do
      FLocations[i] := FLocations[i + 1];
    SetLength(FLocations, Length(FLocations) - 1);
    
    FSelectedID := -1;
    RefreshList;
    StatusBar.SimpleText := 'ลบสำเร็จ';
  end;
end;

procedure TLocationApp.btnGeocodeClick(Sender: TObject);
var
  Address: string;
  Location: TGeoLocation;
  Idx: Integer;
begin
  Address := edtAddress.Text;
  if Address = '' then
  begin
    ShowMessage('กรุณาใส่ที่อยู่ก่อน');
    Exit;
  end;
  
  StatusBar.SimpleText := 'กำลังค้นหาพิกัด...';
  Application.ProcessMessages;
  
  Location := FGeocoder.GeocodeThailand(Address);
  
  if Location.Found then
  begin
    lblCoords.Caption := Format('พิกัด: %.6f, %.6f', [Location.Lat, Location.Lng]);
    
    // อัปเดตพิกัดใน location
    Idx := FindLocation(FSelectedID);
    if Idx >= 0 then
    begin
      FLocations[Idx].Lat := Location.Lat;
      FLocations[Idx].Lng := Location.Lng;
      AddMarkerToMap(FLocations[Idx]);
    end;
    
    ExecuteJS(Format('moveTo(%s, %s, 15)',
      [FormatFloat('0.######', Location.Lat),
       FormatFloat('0.######', Location.Lng)]));
       
    StatusBar.SimpleText := 'พบพิกัด: ' + Location.DisplayName;
  end
  else
    StatusBar.SimpleText := 'ไม่พบพิกัดสำหรับ: ' + Address;
end;

procedure TLocationApp.btnRouteClick(Sender: TObject);
var
  FromIdx, ToIdx: Integer;
  Distance: Double;
begin
  if lvLocations.SelCount < 2 then
  begin
    ShowMessage('กรุณาเลือกสถานที่อย่างน้อย 2 แห่ง (Ctrl+Click)');
    Exit;
  end;
  
  // คำนวณระยะทางระหว่างสถานที่ที่เลือก (simplified)
  FromIdx := FindLocation(FSelectedID);
  if FromIdx >= 0 then
  begin
    Distance := 0;
    // ... คำนวณ route
    StatusBar.SimpleText := Format('ระยะทางประมาณ: %.1f กม.', [Distance / 1000]);
  end;
end;

procedure TLocationApp.btnSearchClick(Sender: TObject);
begin
  RefreshList;
end;

procedure TLocationApp.btnCurrentLocationClick(Sender: TObject);
begin
  ExecuteJS('getCurrentLocation()');
end;

function TLocationApp.FindLocation(AID: Integer): Integer;
var
  i: Integer;
begin
  Result := -1;
  for i := 0 to High(FLocations) do
    if FLocations[i].ID = AID then
    begin
      Result := i;
      Exit;
    end;
end;

procedure TLocationApp.SaveLocations;
var
  F: TextFile;
  i: Integer;
begin
  AssignFile(F, DATA_FILE);
  try
    Rewrite(F);
    WriteLn(F, Length(FLocations));
    WriteLn(F, FNextID);
    
    for i := 0 to High(FLocations) do
    begin
      with FLocations[i] do
      begin
        WriteLn(F, ID);
        WriteLn(F, Name);
        WriteLn(F, Address);
        WriteLn(F, Category);
        WriteLn(F, Phone);
        WriteLn(F, FormatFloat('0.######', Lat));
        WriteLn(F, FormatFloat('0.######', Lng));
        WriteLn(F, Note);
        WriteLn(F, '---');
      end;
    end;
    
    CloseFile(F);
  except
    // ignore save errors
  end;
end;

procedure TLocationApp.LoadLocations;
var
  F: TextFile;
  Count, i: Integer;
  Line: string;
begin
  AssignFile(F, DATA_FILE);
  try
    Reset(F);
    ReadLn(F, Count);
    ReadLn(F, FNextID);
    
    SetLength(FLocations, Count);
    
    for i := 0 to Count - 1 do
    begin
      ReadLn(F, FLocations[i].ID);
      ReadLn(F, FLocations[i].Name);
      ReadLn(F, FLocations[i].Address);
      ReadLn(F, FLocations[i].Category);
      ReadLn(F, FLocations[i].Phone);
      ReadLn(F, Line); FLocations[i].Lat := StrToFloatDef(Line, 0);
      ReadLn(F, Line); FLocations[i].Lng := StrToFloatDef(Line, 0);
      ReadLn(F, FLocations[i].Note);
      ReadLn(F, Line);  // '---'
    end;
    
    CloseFile(F);
    RefreshList;
  except
    LoadSampleLocations;
  end;
end;

procedure TLocationApp.LoadSampleLocations;
begin
  SetLength(FLocations, 5);
  
  with FLocations[0] do
  begin
    ID := FNextID; Inc(FNextID);
    Name := 'วัดพระแก้ว';
    Address := 'ถนนหน้าพระลาน แขวงพระบรมมหาราชวัง เขตพระนคร กรุงเทพ';
    Category := 'สถานที่ท่องเที่ยว';
    Phone := '02-224-1833';
    Lat := 13.7500;
    Lng := 100.4914;
    Note := 'วัดสำคัญในพระบรมมหาราชวัง';
  end;
  
  with FLocations[1] do
  begin
    ID := FNextID; Inc(FNextID);
    Name := 'ห้างเซ็นทรัลเวิลด์';
    Address := '4/1-4/2 ถนนราชดำริ แขวงปทุมวัน เขตปทุมวัน กรุงเทพ';
    Category := 'ห้างสรรพสินค้า';
    Phone := '02-021-9999';
    Lat := 13.7469;
    Lng := 100.5391;
    Note := 'ศูนย์การค้าขนาดใหญ่ใจกลางกรุงเทพฯ';
  end;
  
  with FLocations[2] do
  begin
    ID := FNextID; Inc(FNextID);
    Name := 'สนามบินสุวรรณภูมิ';
    Address := '999 ถนนบางนา-ตราด แขวงราชาเทวะ เขตบางพลี สมุทรปราการ';
    Category := 'อื่นๆ';
    Phone := '02-132-1888';
    Lat := 13.6900;
    Lng := 100.7501;
    Note := 'สนามบินนานาชาติหลักของประเทศไทย';
  end;
  
  with FLocations[3] do
  begin
    ID := FNextID; Inc(FNextID);
    Name := 'โรงพยาบาลจุฬาลงกรณ์';
    Address := '1873 ถนนพระราม 4 แขวงปทุมวัน เขตปทุมวัน กรุงเทพ';
    Category := 'โรงพยาบาล';
    Phone := '02-256-4000';
    Lat := 13.7352;
    Lng := 100.5339;
    Note := 'โรงพยาบาลในสังกัดสภากาชาดไทย';
  end;
  
  with FLocations[4] do
  begin
    ID := FNextID; Inc(FNextID);
    Name := 'มหาวิทยาลัยจุฬาลงกรณ์';
    Address := '254 ถนนพญาไท แขวงวังใหม่ เขตปทุมวัน กรุงเทพ';
    Category := 'สถานศึกษา';
    Phone := '02-215-0871';
    Lat := 13.7394;
    Lng := 100.5294;
    Note := 'มหาวิทยาลัยแห่งแรกของประเทศไทย';
  end;
  
  RefreshList;
end;

end.
```

---

## 64.5 Distance Calculator

```pascal
// คำนวณระยะทางระหว่างหลายจุด
procedure CalculateRoute;
var
  Points: array of record
    Name: string;
    Lat, Lng: Double;
  end;
  TotalDistance: Double;
  i: Integer;
  Geo: TGeocoder;
  Dist: Double;
begin
  SetLength(Points, 4);
  Points[0] := (Name: 'กรุงเทพฯ'; Lat: 13.7563; Lng: 100.5018);
  Points[1] := (Name: 'เชียงใหม่'; Lat: 18.7883; Lng: 98.9853);
  Points[2] := (Name: 'ภูเก็ต'; Lat: 7.8804; Lng: 98.3923);
  Points[3] := (Name: 'ขอนแก่น'; Lat: 16.4322; Lng: 102.8236);
  
  Geo := TGeocoder.Create;
  TotalDistance := 0;
  
  try
    WriteLn('=== ระยะทางระหว่างเมือง ===');
    
    for i := 0 to High(Points) - 1 do
    begin
      Dist := Geo.Distance(
        Points[i].Lat, Points[i].Lng,
        Points[i+1].Lat, Points[i+1].Lng
      );
      
      TotalDistance := TotalDistance + Dist;
      
      WriteLn(Format('%s -> %s: %.1f กม.',
        [Points[i].Name, Points[i+1].Name, Dist / 1000]));
    end;
    
    WriteLn('');
    WriteLn('ระยะทางรวม: ' + FormatFloat('#,##0.0', TotalDistance / 1000) + ' กม.');
    
    // แปลงพิกัดเป็น DMS
    WriteLn('');
    WriteLn('พิกัด DMS ของกรุงเทพฯ:');
    WriteLn('  Lat: ' + TGeocoder.ToDMS(Points[0].Lat, True));
    WriteLn('  Lng: ' + TGeocoder.ToDMS(Points[0].Lng, False));
    
  finally
    Geo.Free;
  end;
end;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Leaflet.js + WebBrowser** - แสดง OpenStreetMap ใน Lazarus
2. **Geocoding** - แปลงที่อยู่เป็นพิกัด GPS
3. **Reverse Geocoding** - แปลงพิกัดเป็นที่อยู่
4. **Haversine Formula** - คำนวณระยะทางบนผิวโลก
5. **Location App** - แอปพลิเคชันจัดการสถานที่ครบวงจร

การใช้ WebView + Leaflet.js เป็นวิธีที่ง่ายที่สุดในการรวม Map กับ Lazarus เนื่องจากไม่ต้องพึ่งพา library ภายนอกมาก
