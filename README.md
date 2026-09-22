# RUPrint Submission & Approval System

Rubot প্রজেক্টের **Verification & Submission Engine (Pillar 1)** এর আওতায় বানানো দ্বিতীয় automation, এবং RUPrint venture-এর content collection funnel-এর প্রথম ধাপ। উদ্দেশ্য: RU/IER students-দের কাছ থেকে course notes/PDF collect করা, human-approve করা, এবং approve হলে automatically সঠিক Course + Category অনুযায়ী Google Drive folder-এ organize করা।

## Core Pattern

```
Form Submission → Rule-based Check → Human Approval (Telegram) → Database Update (Supabase) → File Organize (Drive)
```

## Data Flow

```
[Apps Script: createFirstYearCoursesFolders]
   - Google Drive-এ মূল folder + 6 course × 7 category
     = 42টা subfolder তৈরি করে (one-time setup)
        │
        ▼
[Apps Script: populateFolderLookup]
   - ওই 42টা folder স্ক্যান করে Google Sheet-এ
     Course | Category | FolderID | MatchKey বসিয়ে দেয়
        ▼
[Google Sheet: RUPrint Folder Lookup]  ← reference data, standalone

[Apps Script: createRUPrintForm]
   - Google Form + Response Sheet বানায়
   - File Upload question ম্যানুয়ালি যোগ করতে হয়
     (Apps Script দিয়ে File Upload question বানানো যায় না —
      এটা Google-এর সীমাবদ্ধতা)
        │
        ▼
[Google Form: RUPrint Content Submission]
   │  (নাম, Dept, Course, Category, Semester,
   │   File Upload, Contact Number, Consent)
   ▼
[Google Sheet: RUPrint Content Submission (Responses)]
        │  (n8n প্রতি ১ মিনিটে নতুন row চেক করে)
        ▼
[n8n Trigger: New Form Response]
        ▼
[IF Node: Validate Submission]
   - Name/Course/Category/File খালি না?
   - Consent == true?
        │
   ┌────┴────┐
  Pass       Fail
   │           │
   ▼           ▼
[Supabase   [No Operation]
 Insert]
 status=
 "pending"
   │
   ▼
[Telegram: Notify Admin]
   - নাম, Course, Category, File link সহ মেসেজ
   - Inline button: ✅ Approve / ❌ Reject
   │
   ▼
[Telegram Trigger → Parse Callback Data → IF: approve/reject]
   │
   ┌────┴────┐
Approve    Reject
   │           │
   ▼           ▼
[Lookup:    [Supabase
 Folder      Update:
 Lookup      status=
 Sheet-এ     rejected]
 Course+
 Category
 মিলিয়ে
 FolderID
 বের করা]
   │
   ▼
[Google Drive: Move File]
   - form-এর default upload folder থেকে
     সঠিক course/category folder-এ move
   │
   ▼
[Supabase Update:
 status=approved,
 target_folder_id=FolderID]
   │
   └─────┬─────┘
         ▼
[Telegram: Edit Message — Approved/Rejected দেখায়]
```

## Supabase Table: `ruprint_submissions`

```sql
create table ruprint_submissions (
  id uuid primary key default gen_random_uuid(),
  contributor_name text not null,
  department text not null,
  course text not null,
  category text not null,
  semester text,
  file_url text not null,
  phone text,
  consent boolean not null default false,
  status text not null default 'pending',
  target_folder_id text,
  telegram_message_id text,
  submitted_at timestamptz default now(),
  decided_at timestamptz
);

alter table ruprint_submissions disable row level security;
```

## Folder Structure (Google Drive)

```
IER First Semester All Course Content/
├── 1. Introduction to Education/
│   ├── Syllabus & Course Info/
│   ├── Instructor Guidelines/
│   ├── Class Notes & Materials/
│   ├── Previous Year Questions/
│   ├── Reference Books & PDFs/
│   ├── Important Questions/
│   └── Model Answers/
├── 2. History of Education/  (একই 7টা subfolder)
├── 3. Bangla/
├── 4. Bangladesh Studies: History, Culture & Heritage/
├── 5. Introduction to Sociology/
└── 6. Principles of Economics/
```

## Apps Scripts (folder/lookup/form বানানোর জন্য, one-time setup)

### 1. Folder structure বানানো

```javascript
function createFirstYearCoursesFolders() {
  var mainFolderName = "IER First Semester All Course Content";
  
  var courseNames = [
    "1. Introduction to Education",
    "2. History of Education",
    "3. Bangla",
    "4. Bangladesh Studies: History, Culture & Heritage",
    "5. Introduction to Sociology",
    "6. Principles of Economics"
  ];
  
  var subFolderNames = [
    "Syllabus & Course Info",
    "Instructor Guidelines",
    "Class Notes & Materials",
    "Previous Year Questions",
    "Reference Books & PDFs",
    "Important Questions",
    "Model Answers"
  ];

  var mainFolder = DriveApp.createFolder(mainFolderName);
  
  for (var i = 0; i < courseNames.length; i++) {
    var courseFolder = mainFolder.createFolder(courseNames[i]);
    for (var j = 0; j < subFolderNames.length; j++) {
      courseFolder.createFolder(subFolderNames[j]);
    }
  }
}
```

### 2. Lookup Sheet populate করা

আগে একটা blank Google Sheet বানাও নাম "RUPrint Folder Lookup" দিয়ে, তার URL কপি করে নিচের script-এর `sheetUrl` variable-এ বসাও:

```javascript
function populateFolderLookup() {
  var mainFolderName = "IER First Semester All Course Content";
  var sheetUrl = "PASTE_YOUR_URL_HERE";
  
  var folders = DriveApp.getFoldersByName(mainFolderName);
  if (!folders.hasNext()) {
    Logger.log("Main folder পাওয়া যায়নি: " + mainFolderName);
    return;
  }
  var mainFolder = folders.next();
  
  var sheet = SpreadsheetApp.openByUrl(sheetUrl).getActiveSheet();
  
  sheet.clear();
  sheet.appendRow(["Course", "Category", "FolderID", "MatchKey"]);
  
  var courseFolders = mainFolder.getFolders();
  while (courseFolders.hasNext()) {
    var courseFolder = courseFolders.next();
    var courseName = courseFolder.getName();
    
    var categoryFolders = courseFolder.getFolders();
    while (categoryFolders.hasNext()) {
      var categoryFolder = categoryFolders.next();
      var categoryName = categoryFolder.getName();
      var folderId = categoryFolder.getId();
      var matchKey = courseName + "|" + categoryName;
      
      sheet.appendRow([courseName, categoryName, folderId, matchKey]);
    }
  }
  
  Logger.log("সম্পন্ন! মোট row: " + (sheet.getLastRow() - 1));
}
```

### 3. Google Form বানানো

```javascript
function createRUPrintForm() {
  var form = FormApp.create("RUPrint Content Submission");
  form.setDescription("IER প্রথম সেমিস্টার কোর্সের নোট/ম্যাটেরিয়াল জমা দেওয়ার ফর্ম");

  form.addTextItem().setTitle("নাম").setRequired(true);

  form.addListItem()
    .setTitle("Department")
    .setChoiceValues(["IER"])
    .setRequired(true);

  form.addListItem()
    .setTitle("Course")
    .setChoiceValues([
      "1. Introduction to Education",
      "2. History of Education",
      "3. Bangla",
      "4. Bangladesh Studies: History, Culture & Heritage",
      "5. Introduction to Sociology",
      "6. Principles of Economics"
    ])
    .setRequired(true);

  form.addListItem()
    .setTitle("Category")
    .setChoiceValues([
      "Syllabus & Course Info",
      "Instructor Guidelines",
      "Class Notes & Materials",
      "Previous Year Questions",
      "Reference Books & PDFs",
      "Important Questions",
      "Model Answers"
    ])
    .setRequired(true);

  form.addTextItem().setTitle("Semester").setRequired(true);

  // File Upload question script দিয়ে বানানো যায় না —
  // Form বানানোর পর ম্যানুয়ালি Add question → File upload

  form.addTextItem().setTitle("Contact Number").setRequired(true);

  form.addCheckboxItem()
    .setTitle("Copyright Consent")
    .setChoiceValues(["আমি নিশ্চিত করছি এটি আমার নিজের নোট অথবা বৈধভাবে শেয়ারযোগ্য কনটেন্ট, কোনো কপিরাইটেড বই/প্রকাশনার স্ক্যান না"])
    .setRequired(true);

  var ss = SpreadsheetApp.create("RUPrint Content Submission (Responses)");
  form.setDestination(FormApp.DestinationType.SPREADSHEET, ss.getId());

  Logger.log("Form Edit URL: " + form.getEditUrl());
  Logger.log("Form Published URL: " + form.getPublishedUrl());
  Logger.log("Response Sheet URL: " + ss.getUrl());
}
```

Run করার পর Form Edit URL দিয়ে Form খুলে **ম্যানুয়ালি "File Upload" question যোগ করতে হবে** (Semester আর Contact Number-এর মাঝে বসালে ক্রম ঠিক থাকবে)।

## Credentials লাগবে

- **Google Sheets Trigger** (OAuth2) — Form response sheet পড়ার জন্য
- **Google Drive OAuth** — File move করার জন্য
- **Supabase API** — Host (`https://<project-id>.supabase.co`) + Secret Key (`service_role` key)
- **Telegram API** — Bot token

## Setup করার সিকোয়েন্স

1. Apps Script দিয়ে Drive folder structure বানাও (script #1)
2. Blank Google Sheet বানাও "RUPrint Folder Lookup" নামে, script #2 দিয়ে populate করো
3. Apps Script দিয়ে Form বানাও (script #3), তারপর Form-এ গিয়ে ম্যানুয়ালি File Upload question যোগ করো
4. Supabase-এ উপরের SQL চালিয়ে table বানাও, RLS disable করো
5. Telegram bot + Chat ID (আগে থেকে আছে হলে reuse করো)
6. n8n-এ Credential Manager-এ Google Sheets, Google Drive, Supabase, Telegram credential যোগ করো
7. n8n-এ নতুন workflow বানাও, AI assistant-কে নিচের prompt দাও
8. Node ধরে ধরে credential assign করো; Lookup matching আর file ID extraction অংশ manually verify করো
9. Workflow Publish + Active করো, test submission দিয়ে end-to-end verify করো

## n8n AI Assistant Prompt

```
আমি একটা n8n workflow বানাতে চাই, RUPrint নামে একটা academic 
resource collection ও approval system-এর জন্য। ধাপগুলো:

1. Trigger: Google Sheets Trigger, "On row added"। এই Sheet একটা 
   Google Form-এর response sheet, columns: নাম, Department, Course, 
   Category, Semester, File Upload, Contact Number, Copyright Consent, 
   Timestamp।

2. IF node "Validate Submission": চেক করবে নাম, Course, Category, 
   File Upload খালি না, এবং Copyright Consent checked/true। পাস করলে 
   true branch, নাহলে No Operation node দিয়ে থেমে যাবে।

3. true branch-এ Supabase node, operation "Insert", table 
   "ruprint_submissions", field mapping: contributor_name=নাম, 
   department=Department, course=Course, category=Category, 
   semester=Semester, file_url=File Upload, phone=Contact Number, 
   consent=Copyright Consent, status="pending" (fixed)। Insert-এর 
   response-এর id সেভ রাখো পরের ধাপের জন্য।

4. Telegram node, "Send Message" আমার chat-এ, ফরম্যাট:
   "📚 নতুন RUPrint Submission
   নাম: {{নাম}}
   Course: {{Course}}
   Category: {{Category}}
   File: {{File Upload}}"
   
   Inline keyboard: Button "✅ Approve" callback_data="approve_<id>", 
   Button "❌ Reject" callback_data="reject_<id>"

5. নতুন trigger branch: Telegram Trigger, updates: "callback_query"

6. Code node: callback_data থেকে action আর id আলাদা করো (underscore 
   দিয়ে split করে)

7. IF node: action == "approve" নাকি "reject"

8. Reject branch: Supabase Update node, table "ruprint_submissions", 
   filter id দিয়ে, status="rejected", decided_at=current timestamp

9. Approve branch:
   a. Google Sheets node: "RUPrint Folder Lookup" Sheet থেকে row 
      খোঁজো যেখানে MatchKey column সমান হয় (Course + "|" + Category)। 
      FolderID column রিটার্ন করাও।
   b. Google Drive node, operation "Move File": file-টা হবে 
      submission-এর File Upload লিংক থেকে, destination folder হবে 
      উপরের ধাপ থেকে পাওয়া FolderID।
   c. Supabase Update node: status="approved", 
      target_folder_id=FolderID, decided_at=current timestamp।

10. সবশেষে প্রতিটা branch-এ Telegram node দিয়ে মূল message edit করে 
    confirmation দেখাও ("✅ Approved হয়েছে" / "❌ Rejected হয়েছে")।

সব node-এর নাম স্পষ্ট রাখো। Credentials আমি নিজে পরে select করব।
```

## Common Issues

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| Google Sheets Trigger-এ "No results" | Form-এর সাথে এখনো কোনো Sheet link করা হয়নি | Form → Responses ট্যাব → Sheets আইকন → Create |
| Apps Script-এ `addFileUploadItem is not a function` | Google Apps Script দিয়ে File Upload question বানানো যায় না (Google-এর সীমাবদ্ধতা) | Script দিয়ে বাকি field বানিয়ে, Form-এ গিয়ে ম্যানুয়ালি File Upload question যোগ করতে হয় |
| Supabase credential "Couldn't connect" | Host field-এ শুধু Project ID বসানো হয়েছিল, পুরো URL না | `https://<project-id>.supabase.co` ফরম্যাটে বসাতে হবে |
| Telegram-এ approval message আসছে না | (ক) Workflow Active করা হয়নি, (খ) n8n workspace sleep/offline (trial hosting-এ হয়) | Active toggle চেক করো; workspace অফলাইন হলে কিছুক্ষণ অপেক্ষা করে রিফ্রেশ করো |
| Course/Category dropdown-এ lookup fail | Form-এর dropdown option আর Lookup Sheet-এর Course/Category column-এর বানান/স্পেসিং না মেলা | সবসময় Lookup Sheet থেকে copy-paste করে dropdown option বসাতে হবে, নতুন করে টাইপ না করে |

## Design Principles

- **Rule-based check** — Course/Category structured dropdown data, AI/LLM লাগে না। ফাইলের কনটেন্ট আসলেই useful/legitimate কিনা — এটা judgment-based, তাই human review-ই দিচ্ছে এই সিদ্ধান্ত।
- **Human approval প্রায় ১০০% submission-এ** — pilot stage-এ কনটেন্ট quality নিশ্চিত করতে।
- **Course-only granularity না নিয়ে Course+Category (৪২ variant)** — pilot-এই পুরো real structure নিয়ে শুরু করা হয়েছে, যাতে পরে reorganize করা না লাগে।
- **File "move" (copy না)** — duplicate storage এড়ানোর জন্য।
