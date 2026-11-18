# Лаборатори 05 — REST API-ийн тест дизайн

Оюутан: Г. Саранчимэг B232274001  


## 1. Ажиллуулах орчин

Шаардлага: Node.js 16+ болон Newman.

```powershell
node -v
newman -v
node server.js
```

Сервер `http://localhost:3000` дээр асна. Өөр terminal-оос:

```powershell
newman run lab05-collection.json --reporters cli | Tee-Object -FilePath results/newman-pass.txt
$LASTEXITCODE
```

`$LASTEXITCODE = 0` бол PASS. Postman-д `lab05-collection.json`-ийг импортлоод collection variable-ийн `baseUrl`-ийг `http://localhost:3000` хэвээр хэрэглэнэ.

## 2. Лекц 5-ын таван алхам

### 1) Бие даан тестлэгдэх функц

Тестлэгдэх үндсэн функц нь `POST /registrations`. Оролт нь `studentID`, `courseID`; систем оюутан идэвхтэй эсэх, хичээл байгаа эсэх, prerequisite-ууд биелсэн эсэхийг шалгаад бүртгэнэ. `PUT /students/:id`, `PUT /courses/:id` нь зөвхөн тухайн тестийн setup бөгөөд гол функцийн oracle биш.

### 2–3) Сонголтууд ба төлөөлөх утгууд

| Сонголт | Ангилал | Төлөөлөх утга |
|---|---|---|
| `studentID`-ийн байдал | идэвхтэй / идэвхгүй / байхгүй | `S01` / `S03` / `UNKNOWN02` |
| `coursesTaken` | prerequisite-ийг хангасан / хангаагүй | `["CS201"]` / `[]` |
| `courseID`-ийн байдал | байгаа / байхгүй | `C01` / `UNKNOWN04` |
| prerequisite | бүгд үзсэн / огт үзээгүй / зарим нь үзсэн | `["CS201"]` + `["CS201"]`; `[]` + `["CS201"]`; `["CS201","MATH101"]` + `["CS201"]` |
| хүсэлтийн бүтэц | бүрэн / талбар дутуу / JSON буруу | хоёр талбар / `courseID` дутуу / тасарсан JSON |

### 4) Спецификаци

| ID | Тохиолдол | Setup / төлөөлөх утга | Хүлээгдэх HTTP | Хүлээгдэх body |
|---|---|---|---:|---|
| S01 | Happy path | active, CS201; C01 requires CS201 | 201 | `result=OK`, numeric `registrationID` |
| S02 | Оюутан байхгүй | C02 exists; `UNKNOWN02` | 200 | `ERROR_NO_STUDENT` |
| S03 | Оюутан идэвхгүй | S03 inactive; C03 exists | 200 | `ERROR_INACTIVE_STUDENT` |
| S04 | Хичээл байхгүй | S04 active; `UNKNOWN04` | 200 | `ERROR_NO_COURSE` |
| S05 | Prerequisite огт хангаагүй | S05 `[]`; C05 requires CS201 | 200 | `ERROR_PREREQUISITES`, missing CS201 |
| S06 | Бүх prerequisite хангасан | S06 has CS201, MATH101; C06 requires both | 201 | `result=OK`, numeric `registrationID` |
| S07 | Хязгаар: prerequisite хоосон | S07 active; C07 requires `[]` | 201 | `result=OK`, numeric `registrationID` |
| S08 | Хязгаар: coursesTaken хоосон | S08 active, `[]`; C08 requires CS201 | 200 | `ERROR_PREREQUISITES` |
| S09 | Талбар дутуу | `studentID` байгаа, `courseID` байхгүй | 400 | `ERROR_BAD_REQUEST` |
| S10 | JSON буруу | тасарсан JSON | 400 | `ERROR_BAD_JSON` |

Стратеги нь happy path 3, алдааны ангилал бүрээс дор хаяж 1, мөн boundary 2 кейс ашигласан. Иймээс бодит хэрэгжүүлсэн specification 10 байна.

### 5) Бодит тест кейс

`lab05-collection.json`-ийн бүх folder нэг бие даасан тест кейс. Folder дотор эхлээд өөрийн student/course setup PUT-үүд, дараа нь нэг POST, эцэст нь `pm.test` oracle-ууд байна. Setup ID-ууд давхцахгүй тул дараалал болон өмнөх бүртгэлээс хамаарахгүй. `registrationID` нь серверийн ажилласан хугацаанд өсдөг учир яг `1` гэх мэт тогтмол утга шалгаагүй.

## 3. Newman үр дүн

PASS collection-ийг сервер асаалттай үед ажиллуулж, бүх request болон assertion-ийг дахин үүсгэнэ:

```powershell
New-Item -ItemType Directory -Force results | Out-Null
newman run lab05-collection.json --reporters cli 2>&1 | Tee-Object -FilePath results/newman-pass.txt
$LASTEXITCODE # 0
```

Буруу oracle-ийн quality gate-ийг шалгахдаа:

```powershell
newman run lab05-collection-fail.json --reporters cli 2>&1 | Tee-Object -FilePath results/newman-fail.txt
$LASTEXITCODE # 1
```

Серверийг зогсоогоод ажиллуулсан DOWN хувилбар нь interface/network алдаа болохыг харуулна:

```powershell
newman run lab05-collection.json --reporters cli 2>&1 | Tee-Object -FilePath results/newman-down.txt
$LASTEXITCODE # 1; ECONNREFUSED
```

`results/` доторх тоонууд нь Newman-ийн бодит CLI summary-гаас авсан: PASS — 1 iteration, 24 requests, 38 assertions, failed 0, exit code 0; FAIL — 1 iteration, 3 requests, 3 assertions, failed 1, exit code 1; DOWN — 1 iteration, 24 request, бүгд `ECONNREFUSED`, exit code 1. DOWN нь oracle-ийн алдаа биш, сервер ажиллахгүй байгаатай холбоотой interface/network алдаа юм.

## 4. Дүгнэлт

Энэ ажлын хамгийн их бодол шаардсан алхам нь сонголтуудыг зөв ангилах байлаа. Нэг л алдааны жишээ авахад идэвхгүй оюутан, байхгүй оюутан, prerequisite дутуу гэсэн ялгаатай бизнес дүрмүүд орхигдоно. Төлөөлөх утгыг ангилал бүрээс сонгосноор 10 кейсээр үндсэн шийдвэрүүдийг хамруулсан. Boundary кейсүүд болох хоосон prerequisite болон хоосон `coursesTaken` нь массивын логикийн алдааг илрүүлэхэд хэрэгтэй. Спецификацид HTTP status болон `result`-ийг хамтад нь oracle болгосон тул 200 дээр буцсан бизнесийн алдааг амжилт гэж андуурахгүй. Тест бүр setup-аа өөрөө хийдэг учир collection-ийн дараалал өөрчлөгдөхөд тогтвортой. Newman-ийн exit code нь CI quality gate болгон ашиглагдана. Харин DOWN үр дүн нь API ажиллахгүй үеийн орчны алдаа бөгөөд серверийн бизнес oracle-ийн алдаа биш. Бодит API дээр тест дизайн нь зөвхөн request илгээхээс илүүтэй зөв ангилал, зөв төлөөлөх утга, хэмжигдэхүйц хүлээлт гурвыг холбох ажил юм.

## 5. Нэмэлт AI өнцөг

AI-д ижил API-ийн сонголтуудыг ангилуулбал идэвхтэй/идэвхгүй/байхгүй оюутан, байгаа/байхгүй хичээл, prerequisite-ийн төлөвүүдийг зөв санал болгоно. Мөн талбар дутуу болон буруу JSON гэсэн input-format ангиллыг нэмж сануулж чадна. Гэхдээ AI-ийн санал анхандаа `studentID`-ийн тэмдэгтийн хэлбэр зэрэг синтаксийн ангиллыг тусад нь авч үзээгүй. Үүнийг төлөөлөх утгын хүснэгтэд нэмэх шаардлагатай. AI-ийн oracle-уудын status ба `result` хослол зөв байсан ч `registrationID`-г яг тогтмол тоотой харьцуулах эрсдэл гарч болно. Иймээс энэ collection-д ID нь тоо мөн эсэхийг шалгасан. Эцсийн сонголтыг server.js-ийн бодит behavior-тэй тулгаж хүн баталгаажуулсан.
