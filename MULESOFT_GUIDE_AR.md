<div dir="rtl" align="right">

# دليل تحدي MuleSoft بالعربي (مبسّط)

> هذا الملف شرح مبسّط بالعربي للملف الإنكليزي `MULESOFT_CHALLENGE_GUIDE.md`.
> المصطلحات التقنية (Flow, Payload, DataWeave...) أبقيناها بالإنكليزي لأنها هي اللي رح تسمعها بالمقابلة.
> الكود كله بالإنكليزي، والشرح بالعربي.

---

## 1. شنو المطلوب منك بالضبط؟ (بجملة وحدة)

> تبني **API صغير** بـ MuleSoft يدير بيانات زبائن (Customers)، وتكتب كود نظيف، وتتعامل مع الأخطاء صح، وتشرح قراراتك بثقة.

**هم مو ينتظرون منك خبير MuleSoft.** هم يريدون يشوفون:

| شنو يقيّمون | شنو يعني لك |
|---|---|
| أساسيات الـ Backend | تعرف HTTP وREST وJSON وStatus Codes. **هذا موجود عندك أصلاً** |
| سرعة التعلّم | تتعلم MuleSoft بأسبوع وتبني شي شغّال |
| طريقة حل المشاكل | لما يخرب شي، تفكر بهدوء وتدبّغ (Debug) بطريقة منظمة |
| وضوح الكود | أسماء واضحة، ملفات مرتبة، ما في تكرار |
| فهم الـ API والـ Integration | تفهم شلون API يكلّم API ثاني وشنو يصير إذا فشل |
| التواصل | تشرح قراراتك ببساطة |

**أهم نصيحة:** الجلسة المباشرة (Live Coding) تختبر إذا المشروع **مشروعك فعلاً** ولا حفظته أو ولّده الـ AI. لذلك افهم كل سطر.

---

## 2. قائمة التسليم (Checklist)

- [ ] مشروع Mule 4 على Anypoint Studio
- [ ] 3 Endpoints:
  - [ ] `GET /customers/{customerId}` ← يرجّع تفاصيل زبون
  - [ ] `POST /customers` ← ينشئ زبون جديد
  - [ ] `GET /customers` ← يرجّع قائمة الزبائن
- [ ] كل شي JSON (طلب وجواب)
- [ ] التحقق (Validation) من الحقول المطلوبة
- [ ] أكواد HTTP صحيحة: `200, 201, 400, 404, 500` (+ `502, 504` من ملفات العينة)
- [ ] البيانات: قائمة بالذاكرة أو ملف JSON (**ما نحتاج Database**)
- [ ] `GET /customers/{customerId}` يكلّم **خدمة خارجية** (Mock Backend) بواسطة **HTTP Request Connector**
  - [ ] رابط الـ Backend في **ملف إعدادات** مو مكتوب بالكود
  - [ ] تحديد **Timeout**
  - [ ] التعامل مع: 404 و4xx و5xx والـ Timeout
- [ ] **DataWeave** لتحويل الطلبات والأجوبة
- [ ] رسائل خطأ موحّدة (Standard Error Response)
- [ ] **Logging** مع **Correlation ID**
- [ ] **ما تطبع بيانات حساسة** (إيميل، موبايل، تاريخ ميلاد...)
- [ ] إعدادات حسب البيئة (dev / test / prod) بملفات Properties
- [ ] مشروع على **Git/GitHub** + ملف **README** (طريقة التشغيل والاختبار)
- [ ] تضيف ملفات الـ JSON اللي أعطوك إياها للـ Repository
- [ ] تخبرهم إنك جاهز للجلسة المباشرة

---

## 3. شنو تقول لك ملفات الـ JSON اللي أعطوك إياها؟

هذي الملفات هي **العقد (Contract)** تبع الـ API. يعني الشكل اللي لازم يطلع منك بالضبط.

### الطلبات (Requests)

| الملف | شنو هو |
|---|---|
| `create-customer-request.json` | طلب صحيح لإنشاء زبون |
| `create-customer-invalid-request.json` | طلب **غلط**: ناقص `lastName` والإيميل غير صحيح. نستخدمه لاختبار الـ 400 |

### الأجوبة (Responses)

| الملف | الكود | الفكرة |
|---|---|---|
| `customer-200.json` | 200 | زبون واحد بكل التفاصيل |
| `customers-200.json` | 200 | قائمة زبائن **بمعلومات مختصرة فقط** + `count` |
| `create-customer-201.json` | 201 | الزبون اللي انخلق + حقول يولّدها السيرفر (`customerId`, `status`, `createdAt`) |
| `error-400.json` | 400 | خطأ تحقق (Validation) مع قائمة `details` |
| `error-404.json` | 404 | الزبون مو موجود |
| `error-500.json` | 500 | خطأ داخلي |
| `error-502.json` | 502 | الـ Backend رجّع خطأ |
| `error-504.json` | 504 | الـ Backend تأخر (Timeout) |

### ملاحظات ذكية (اذكرها بالمقابلة، تعطي انطباع جيد)

1. **كل الأخطاء بنفس الشكل:** `error → code, message, correlationId`، و`details` تظهر **بس** بأخطاء الـ Validation.
2. **كل خطأ فيه `correlationId`** ← نفس الرقم اللي تطبعه بالـ Log. هذا يخلّي الدعم الفني يتتبع الطلب.
3. **جواب القائمة غير جواب الزبون الواحد** (مختصر مقابل كامل) ← تحتاج DataWeave لكل واحد.
4. **الحقول `customerId` و`status` و`createdAt` يولّدها السيرفر** والعميل ما يرسلها.
5. **الـ 502 و504 مو مكتوبين بالإيميل** بس موجودين بالملفات ← لازم تطبقهم.
6. الطلب الغلط (ناقص `lastName` + إيميل غلط) لازم يطلع منه **بالضبط** نفس `details` الموجودة بـ `error-400.json` (خطأين).

---

## 4. المفاهيم الأساسية بشرح بسيط

### 4.1 شنو هو MuleSoft؟

منصة **ربط أنظمة (Integration)**. تبني تطبيقات تستقبل رسالة (من HTTP مثلاً)، تحوّلها، تكلّم أنظمة ثانية، وترجّع جواب.

> تشبيه: تخيل Express/Spring بس بدل ما تكتب كل شي كود، ترسم **خطوات (Flow)** بالـ Studio، وهو يخزنها كملف **XML**.

- **Mule 4**: النسخة الحالية (لا تقرأ شروحات Mule 3، مختلفة).
- **Anypoint Studio**: برنامج التطوير (مبني على Eclipse).
- لازم تتعلم تقرأ **الـ Canvas** (الرسم) و**الـ XML** (الكود الحقيقي). ممكن يطلبون منك تعدل XML مباشرة.

### 4.2 هيكل المشروع

```
src/main/mule/        ← ملفات الـ Flows (XML)
src/main/resources/   ← ملفات الإعدادات، DataWeave، الـ JSON
src/test/munit/       ← الاختبارات (اختياري)
pom.xml               ← المكتبات (Connectors)
```

### 4.3 API-led Connectivity (3 طبقات)

| الطبقة | شنو شغلها | تشبيه Backend |
|---|---|---|
| **Experience API** | مصمّمة لمستخدم معيّن (تطبيق موبايل مثلاً) | Controller |
| **Process API** | منطق العمل وتجميع عدة أنظمة | Service Layer |
| **System API** | غلاف رقيق فوق **نظام واحد** (قاعدة بيانات، Core Banking) | Repository |

**بمشروعك:** تطبيق Mule = الـ API الأمامي، والـ Mock Backend = الـ System API.

> جملة جاهزة للمقابلة: *"التطبيق يعرض عقد واضح للمستهلك ويخفي شكل الـ Backend. إذا تغيّر الـ Backend أغيّر الـ Mapping بمكان واحد فقط."*

### 4.4 الـ Flow والـ Sub-flow والـ Private Flow

| | يبدأ من Listener؟ | عنده Error Handling خاص؟ | متى نستخدمه |
|---|---|---|---|
| **Flow** | نعم | نعم | نقطة دخول رئيسية |
| **Sub-flow** | لا | **لا** (يستخدم تبع اللي ناداه) | خطوات مشتركة صغيرة |
| **Private Flow** | لا | نعم | منطق مشترك يحتاج معالجة أخطاء خاصة |

نناديهم بـ **Flow Reference**. فكّر: Flow = دالة مع try/catch، Sub-flow = دالة مساعدة بدون try/catch.

### 4.5 الـ Mule Event (أهم مفهوم!)

كل رسالة تمر بالـ Flow اسمها **Event**، وتحتوي:

| الجزء | شنو هو | مثال |
|---|---|---|
| `payload` | البيانات الرئيسية | الـ Body تبع الطلب |
| `attributes` | معلومات عن مصدر الرسالة (Metadata) | `attributes.uriParams.customerId` ، `attributes.queryParams` ، `attributes.headers` |
| `vars` | متغيّرات **تضعها أنت** | `vars.customerId` |
| `correlationId` | رقم تتبع الطلب (جاهز) | `correlationId` |
| `error` | موجود **فقط** داخل Error Handler | `error.errorType.identifier` |

> ⚠️ **الفخ الأول والأشهر:** بعد ما ينفّذ **HTTP Request**، تتغير `payload` و`attributes` إلى **جواب الـ Backend**. يعني `attributes.uriParams.customerId` **يختفي**!
> **الحل:** احفظ `customerId` بـ **Set Variable** **قبل** الاتصال بالـ Backend.

**المتغيّرات الجاهزة (Predefined)** اللي ذكروها بالإيميل: `payload`, `attributes`, `vars`, `error`, `correlationId`.

### 4.6 الـ Connectors والـ Global Config

- **Connector** = إضافة تخليك تتكلم مع نظام أو بروتوكول (HTTP, Database, File...).
- **Global Config** = إعدادات مشتركة تُكتب مرة وحدة وتُستخدم بعدة أماكن:
  - إعداد **HTTP Listener** (المنفذ Port).
  - إعداد **HTTP Request** (عنوان الـ Backend + Timeout).
  - إعداد **Configuration Properties** (اسم ملف الإعدادات).

### 4.7 الـ Properties والبيئات (Environments)

- نكتب القيم بملف مثل `config-dev.yaml`:

```yaml
http:
  listener:
    port: "8081"
backend:
  host: "localhost"
  port: "3000"
  responseTimeoutMs: "3000"
```

- نقراها بالـ XML هكذا: `${backend.host}` وبالـ DataWeave هكذا: `p('backend.host')`.
- نختار البيئة وقت التشغيل: `-Denv=dev` (بالـ VM Arguments بالـ Studio).
- **ليش مهم؟** لأن المقيّم يريد: **صفر قيم مكتوبة بالكود** (لا URL ولا Port ولا Timeout).

### 4.8 APIkit أو Listener عادي؟

- **APIkit:** تكتب **مواصفات** (RAML أو OpenAPI) والـ Studio يولّد لك الـ Flows ويتحقق من الطلبات تلقائياً. هذا الأسلوب المحترف.
- **Listener عادي + Choice:** أبسط، بس تسوي كل شي يدوياً.

**توصيتي:** APIkit. وإذا علقت بالـ Day 3، ارجع للأسلوب البسيط.

### 4.9 معالجة الأخطاء (Error Handling) بـ Mule 4

كل خطأ له **نوع** بصيغة `NAMESPACE:IDENTIFIER`:

| النوع | متى يصير |
|---|---|
| `HTTP:NOT_FOUND` | الـ Backend رجّع 404 |
| `HTTP:INTERNAL_SERVER_ERROR` | الـ Backend رجّع 500 |
| `HTTP:TIMEOUT` | الـ Backend تأخر |
| `HTTP:CONNECTIVITY` | الـ Backend طافي / ما نقدر نوصله |
| `APIKIT:BAD_REQUEST` | الطلب ما يطابق المواصفات |
| `ANY` | أي خطأ ثاني |

**الفرق المهم:**

| | On Error **Propagate** | On Error **Continue** |
|---|---|---|
| شنو يسوي | يعالج الخطأ **ويبقى خطأ** (الـ Flow يفشل) | يعالج الخطأ **ويكمل كأنه نجح** |
| الخطر | ما في | ممكن ترجّع 200 بالغلط |
| اخترنا | ✅ هذا | |

**قاعدة:** أول Handler يتطابق **هو اللي ينفذ**. فحط الأنواع المحددة **قبل** `ANY`.

**خطتنا لتحويل الأخطاء (احفظها!):**

| الحالة | الكود | الـ code |
|---|---|---|
| حقول ناقصة أو غلط | 400 | `VALIDATION_ERROR` |
| الزبون مو موجود (404 من الـ Backend) | 404 | `CUSTOMER_NOT_FOUND` |
| الـ Backend رجّع 4xx ثاني | 502 | `DOWNSTREAM_SERVICE_ERROR` |
| الـ Backend رجّع 5xx | 502 | `DOWNSTREAM_SERVICE_ERROR` |
| الـ Backend تأخر | 504 | `DOWNSTREAM_TIMEOUT` |
| الـ Backend طافي | 502 | `DOWNSTREAM_SERVICE_ERROR` |
| أي شي غير متوقع | 500 | `INTERNAL_ERROR` (ما نعرض تفاصيل داخلية أبداً) |

> **سؤال متوقع:** ليش 502 لما الـ Backend يرجّع 4xx؟
> **الجواب:** من وجهة نظر العميل، طلبه سليم. المشكلة بسلسلة الأنظمة الداخلية، فالأصدق نقول "Bad Gateway". الاستثناء الوحيد هو 404 الخاص بالزبون، لأنه فعلاً "غير موجود".

### 4.10 الـ Logging والـ Correlation ID

**الـ Correlation ID** = رقم فريد لكل طلب، يخلّينا نتتبعه عبر كل الأنظمة.

الخطة:
1. إذا العميل أرسل Header اسمه `X-Correlation-ID` نستخدمه، وإذا لا نستخدم اللي يولّده Mule.
2. نطبعه بكل سطر Log.
3. نرسله للـ Backend بنفس الـ Header.
4. نرجعه بـ Header الجواب وداخل جسم الخطأ.

مثال Logger:

```
"[#[correlationId]] GET customer started, customerId=#[vars.customerId]"
```

**ممنوع تطبع:** إيميل، رقم موبايل، تاريخ ميلاد، عنوان، أو الـ Payload كاملاً.
**مسموح تطبع:** correlationId، المسار، الـ customerId، الكود، المدة، **أسماء** الحقول الفاشلة (مو قيمها).

مستويات الـ Log: `INFO` عادي، `WARN` أخطاء العميل (4xx)، `ERROR` أخطاء غير متوقعة (5xx).

### 4.11 الـ HTTP Request Connector

- نحدد: Host, Port, Base Path (كلها من الـ Properties).
- **Response Timeout** (مثلاً 3000 ms) و**Connection Timeout**.
- Method `GET` والمسار `/customers/{id}` مع **URI Parameters** (لا تدمج النصوص يدوياً).
- Headers: `X-Correlation-ID` و`Accept: application/json`.
- **ما تطفّي** التحقق من الاستجابة. Mule يطلع خطأ تلقائياً لما يجي كود غير 2xx، ونحن نستفيد منه بالـ Error Handler.

### 4.12 الـ DataWeave (مفتاح النجاح)

لغة تحويل البيانات. كل سكربت جزئين: **Header** ثم `---` ثم **Body**.

```dataweave
%dw 2.0
output application/json
---
{ name: payload.firstName }
```

تعلّمها بهذا الترتيب:

1. الوصول للحقول: `payload.firstName` ، `payload.address.city`
2. بناء Object `{}` و Array `[]`
3. `default`: `payload.status default "ACTIVE"`
4. `map` و `filter` و `mapObject`
5. النصوص: `upper()` و `++` (دمج)
6. `if / else`
7. التاريخ: `now()` وتنسيقه
8. `uuid()` لتوليد معرّف
9. المتغيرات: `var x = ...`
10. قراءة Property: `p('backend.host')`

**السكربتات الأساسية اللي لازم تكتبها من الذاكرة:**

**(أ) بناء جواب 201:**

```dataweave
%dw 2.0
output application/json
---
{
  customerId: vars.newCustomerId,
  firstName: payload.firstName,
  lastName: payload.lastName,
  email: payload.email,
  mobileNumber: payload.mobileNumber,
  dateOfBirth: payload.dateOfBirth,
  address: payload.address,
  status: "ACTIVE",
  createdAt: now() as String {format: "yyyy-MM-dd'T'HH:mm:ss'Z'"}
}
```

**(ب) جواب القائمة (حقول مختصرة + count):**

```dataweave
%dw 2.0
output application/json
---
{
  customers: payload map (c) -> {
    customerId: c.customerId,
    firstName: c.firstName,
    lastName: c.lastName,
    email: c.email,
    mobileNumber: c.mobileNumber,
    status: c.status
  },
  count: sizeOf(payload)
}
```

**(ج) جواب الخطأ الموحّد:**

```dataweave
%dw 2.0
output application/json
---
{
  error: {
    code: vars.errorCode,
    message: vars.errorMessage,
    correlationId: correlationId
  } ++ (if (vars.errorDetails != null) { details: vars.errorDetails } else {})
}
```

> قاعدة ذهبية: **إذا ما تقدر تشرح سطر DataWeave، لا تخليه بالمشروع.**

### 4.13 وين نخزّن البيانات (بدون Database)؟

| الخيار | ميزة | عيب |
|---|---|---|
| **Object Store** | ميزة حقيقية بـ Mule، الـ POST ثم GET يشتغلون | يتصفّر عند إعادة التشغيل |
| ملف JSON ثابت | أبسط شي | قراءة فقط، الـ POST ما ينحفظ |
| متغيّر عادي | بدون إعداد | يضيع بعد كل طلب (ما يفيد) |

**اخترنا:** Object Store نبدأه من ملف JSON.

---

## 5. خطة الدراسة (7 أيام، 3-4 ساعات باليوم)

> الفكرة: **تعلّم ثم طبّق فوراً.** كل يوم ينتهي بشي يشتغل.

### اليوم 0 (ساعة): التجهيز
- [ ] تنزيل **Anypoint Studio** + Mule Runtime + الـ JDK المطلوب (غالباً Java 17)
- [ ] **Postman** أو curl
- [ ] **Node.js** (للـ Mock Backend)
- [ ] مستودع GitHub فاضي
- [ ] حساب **Anypoint Platform** مجاني

### اليوم 1: الأساسيات + Hello World
- تعلّم: ما هو Mule، الـ Studio، نموذج الـ Event.
- نفّذ:
  - [ ] Flow: HTTP Listener ← Set Payload ← Logger
  - [ ] اقرأ `queryParams` وأرجع `Hello Ahmed`
  - [ ] ضع الـ Port بملف Properties
- **سؤال فحص:** شنو داخل `payload` و`attributes` بعد الـ Listener؟

### اليوم 2: DataWeave
- تعلّم النقاط 1-8 من قسم 4.12.
- نفّذ بالـ **DataWeave Playground**:
  - [ ] حوّل طلب الإنشاء إلى جواب 201.
  - [ ] حوّل قائمة كاملة إلى شكل `customers-200.json`.
  - [ ] مارس `map` و `filter` و `default`.
- **فحص:** اكتب سكربت (أ) و(ب) بدون نظر.

### اليوم 3: هيكل الـ API (3 Endpoints ببيانات تجريبية)
- [ ] اكتب المواصفات (OpenAPI أو RAML) واستعمل ملفات العينة كأمثلة.
- [ ] ولّد الـ Flows بـ APIkit.
- [ ] `GET /customers` و`POST /customers` و`GET /customers/{id}` تشتغل ببيانات تجريبية.
- [ ] جرّبها بـ Postman.

### اليوم 4: الـ Validation ومعالجة الأخطاء
- [ ] تحقق من: `firstName`, `lastName`, `email`, `mobileNumber`.
- [ ] اجمع **كل** الحقول الفاشلة بـ `details[]` (مو أول خطأ بس).
- [ ] Error Handler عام يحوّل الأنواع لـ JSON موحّد.
- [ ] اختبر بالطلب الغلط وقارن مع `error-400.json`.
- **فحص:** اشرح Propagate مقابل Continue بنصف دقيقة.

### اليوم 5: الـ HTTP Request + الـ Mock Backend
- [ ] ابنِ Mock Backend (Node) بمسارات: نجاح، 404، 400، 500، بطيء.
- [ ] ضع Host/Port/Timeout بملف الإعدادات.
- [ ] `GET /customers/{id}`: خزّن `customerId` ثم Logger ثم HTTP Request ثم Transform.
- [ ] اختبر الحالات: 200، 404، 500→502، Timeout→504، الـ Backend طافي→502.
- **فحص:** ليش نحفظ `customerId` بمتغير قبل الاتصال؟

### اليوم 6: الممارسات الهندسية
- [ ] Correlation ID (استقبال/توليد، Log، إرسال للـ Backend، إرجاع).
- [ ] راجع كل الـ Loggers: **لا بيانات شخصية**.
- [ ] ملفات `config-dev/test/prod.yaml` وجرّب `-Denv`.
- [ ] (اختياري) 2-3 اختبارات MUnit.
- [ ] تدرّب على الـ Debugger (Breakpoints).

### اليوم 7: التنظيف والـ README والتمرين
- [ ] اكتب الـ README.
- [ ] أضف `.gitignore`.
- [ ] اختبر "استنساخ نظيف": حمّل مشروعك بمجلد ثاني واتبع الـ README.
- [ ] تدرّب على الأسئلة **بصوت عالٍ**.
- [ ] أخبرهم إنك جاهز.

> **إذا تأخرت:** ركّز بالترتيب: (1) الـ 3 Endpoints (2) الأخطاء وأكواد HTTP (3) HTTP Request والـ Timeout (4) Correlation ID واللوغات الآمنة (5) الـ README. أما APIkit وMUnit والبيئات الثلاث فهي إضافات.

---

## 6. الـ Mock Backend

أسهل خيار لك: **سكربت Node صغير**. يدعم هذي الحالات بمعرّفات خاصة:

| الطلب | شنو يرجّع |
|---|---|
| `GET /backend/customers/CUST-10001` | 200 + بيانات الزبون |
| `GET /backend/customers/CUST-99999` | 404 |
| `GET /backend/customers/CUST-BAD` | 400 |
| `GET /backend/customers/CUST-ERR` | 500 |
| `GET /backend/customers/CUST-SLOW` | 200 بس بعد تأخير أكبر من الـ Timeout |
| لما توقف الـ Mock | خطأ اتصال (`HTTP:CONNECTIVITY`) |

ضعه بالـ Repository بمجلد `mock-backend/` ووثّق طريقة تشغيله بالـ README.

---

## 7. أوامر الاختبار (curl)

```bash
# قائمة الزبائن
curl -i http://localhost:8081/api/customers

# زبون واحد (يروح للـ Mock Backend)
curl -i http://localhost:8081/api/customers/CUST-10001

# زبون غير موجود
curl -i http://localhost:8081/api/customers/CUST-99999

# إنشاء زبون (صحيح)
curl -i -X POST http://localhost:8081/api/customers -H "Content-Type: application/json" -d @samples/requests/create-customer-request.json

# إنشاء زبون (غلط ← 400)
curl -i -X POST http://localhost:8081/api/customers -H "Content-Type: application/json" -d @samples/requests/create-customer-invalid-request.json

# مع Correlation ID
curl -i http://localhost:8081/api/customers/CUST-10001 -H "X-Correlation-ID: test-123"
```

---

## 8. مفاجآت متوقعة بالجلسة المباشرة

| المفاجأة | احتمالها |
|---|---|
| تعديل مشروعك الحالي (إضافة ميزة) | عالي جداً |
| مشروع Mule **جديد** من الصفر | عالي |
| **Code Review** لكود Java / C / C# | متوسط إلى عالي |
| إصلاح Flow مكسور (Debug) | عالي |
| "ليش سويت هذا وماكو بديل؟" | مؤكد |

> **كن صادق بخصوص الـ AI:** مو عيب إنك استخدمته للتعلم، لكن الاختبار هو: هل تقدر تسوي الشي بدونه؟ تدرّب آخر يومين بدون AI.

### 8.1 الـ Code Review (حتى بلغة ما تعرفها)

ما لازم تعرف اللغة بعمق. لازم تعرف **طريقة مراجعة** تصلح لأي لغة، وتقولها بصوت عالٍ:

**6 مراحل:**
1. **افهم:** شنو يسوي هالكود بجملة وحدة؟
2. **الأخطاء (Correctness):** null، شرط غلط، تسريب موارد، استثناءات مبلوعة.
3. **الأمان (Security):** SQL Injection، كلمات سر مكتوبة بالكود، طباعة بيانات شخصية، ما في Validation.
4. **الكود النظيف (Clean Code):** أسماء، دوال طويلة، تكرار، أرقام سحرية (Magic Numbers).
5. **التصميم (SOLID).**
6. **الاختبار والتشغيل:** هل ينختبر؟ هل الأخطاء تنعالج؟

**بالنهاية:** رتّب الملاحظات حسب الخطورة (Blocker / Major / Minor / Nit) وقل كيف تصلح أهم اثنتين.

**SOLID ببساطة:**

| الحرف | المبدأ | مثال مشكلة | الحل |
|---|---|---|---|
| **S** | مسؤولية واحدة | كلاس يتحقق + يتصل بالـ DB + يرسل إيميل | قسّمه لعدة كلاسات |
| **O** | مفتوح للتوسعة مغلق للتعديل | `if/else` طويلة تكبر مع كل نوع جديد | Polymorphism / Strategy |
| **L** | الكلاس الابن يحل محل الأب بدون مشاكل | الابن يرمي `NotSupportedException` | غيّر التصميم |
| **I** | واجهات صغيرة ومحددة | Interface بـ 15 دالة وأغلبها فاضية | قسّمها |
| **D** | اعتمد على Abstraction مو كلاس محدد | `new SqlRepository()` داخل كلاس العمل | Dependency Injection |

**تشبيه Mule:** SRP = كل Flow يسوي شغلة وحدة. DIP = الروابط والإعدادات من Properties مو مكتوبة بالـ Flow.

**أخطاء شائعة حسب اللغة:**
- **C:** `malloc` بدون فحص، `free` ناقص (تسريب)، `strcpy` (Buffer Overflow)، متغيرات بدون تهيئة.
- **C#:** عدم استخدام `using` (Dispose)، `async void`، `.Result` / `.Wait()`، `catch (Exception)` فاضي.
- **Java:** عدم إغلاق الـ Streams (استخدم try-with-resources)، مقارنة `String` بـ `==`، `Optional.get()` بدون فحص.

### 8.2 "افتح مشروعك وأضف ميزة"

استخدم هذي الخطوات كل مرة وقلها بصوت عالٍ:

1. **وضّح المطلوب** بجملة وحدة واسأل سؤال أو سؤالين (شنو الحالات الخاطئة؟).
2. **حدد المكان:** أي ملف XML؟ لازم تعرف خريطة مشروعك.
3. **خطط بصوت عالٍ:** "رح أضيف Flow وDataWeave وProperty وحالة خطأ."
4. **أصغر نسخة شغّالة أولاً:** أرجع جواب ثابت، شغّل، شوف 200. بعدين أضف المنطق.
5. **الإعدادات بالـ Properties** مو بالكود.
6. **أعد الاستخدام:** استعمل الـ Error Handler واللوغ الموجودين.
7. **اختبر** بـ curl، بما فيها حالة فشل واحدة.
8. **لخّص:** "غيّرت 3 ملفات، والمفاضلة كانت كذا."

| طلب متوقع | شنو تغيّر |
|---|---|
| إضافة `PUT /customers/{id}` | المواصفات ← Flow جديد ← Validation ← تحديث التخزين ← حالة 404 |
| إضافة حقل جديد | المواصفات + Validation + DataWeave + التخزين + ملفات العينة + README |
| فلتر `?status=ACTIVE` | `attributes.queryParams.status` ثم `filter` ثم `default` |
| استدعاء Backend ثاني | HTTP Request Config جديد + Properties + تحويل أخطاء |
| تغيير شكل الخطأ | فقط سكربت الـ Error Handler (لهذا مركزناه) |
| Pagination | `limit` و`offset` ثم قص المصفوفة بـ DataWeave |
| بيئة جديدة | `config-<env>.yaml` وتشغيل بـ `-Denv` |

**إذا تجمّدت:** احكِ اللي تفكر فيه. الصمت أسوأ من تخمين غلط.

### 8.3 "ابنِ مشروع جديد من الصفر"

ممكن يعطونك: Orders API، أو تحويل أموال (تحقق: المبلغ > 0، الحسابين مختلفين)، أو ملف CSV ← JSON ← API، أو Scheduler، أو دمج جوابين من API (Scatter-Gather).

**الهيكل الجاهز (احفظه بالترتيب):**
1. مشروع Mule جديد.
2. `global-config.xml`: Configuration Properties + HTTP Listener Config.
3. ملف `config-dev.yaml`.
4. Flow رئيسي: Listener ← Logger ← Set Variable ← المنطق ← Transform ← الجواب.
5. Validation بالبداية (Fail Fast → 400).
6. Error Handler عام بجواب موحّد مع `correlationId`.
7. شغّل واختبر.

**تدرّب:** أعد كتابة هذا الهيكل 3 مرات بمجالات مختلفة (Orders ثم Accounts ثم Products). المرة الثالثة لازم تاخذ ~20 دقيقة.

**إذا شفت شي جديد تماماً** (مثل JMS): قل بصراحة إنك ما استخدمته، ثم افتح docs.mulesoft.com، ابحث عن الـ Quick Start، وجرّب أصغر مثال. هم يريدون يشوفون **كيف تتعلم**.

### 8.4 التواصل أثناء الكتابة

- احكِ **النية** مو ضغطات المفاتيح: "أتحقق أول حتى ما يوصل طلب غلط للـ Backend."
- اذكر المفاضلات: "أقدر أسوي X أو Y، اخترت X لأن..."
- لما يفشل شي: اقرأ الخطأ بصوت عالٍ، كوّن فرضية وحدة، اختبرها. لا تغيّر أشياء عشوائياً.
- عادي تقول: "ما أتذكر الصيغة بالضبط، رح أراجع الـ Docs."
- اسأل أسئلة توضيحية قبل ما تبني.

---

## 9. طريقة الـ Debug (قلها بصوت عالٍ بالجلسة)

1. اقرأ خطأ الـ **Console**: نوع الخطأ + الوصف + أي خطوة فشلت.
2. أضف **Logger** مؤقتاً (احذفه بعدين، لا تترك بيانات شخصية).
3. استخدم **Debugger** بالـ Studio: Breakpoints وشوف payload/attributes/vars.
4. افحص **معاينة Transform Message** ببيانات عينة.
5. افحص الـ **XML** (أخطاء إملائية بـ `config-ref` أو أسماء الـ Flows).
6. أعد المحاولة بـ **curl** لإلغاء تأثير العميل.

**أسباب شائعة:**
- الـ `payload` تغيّرت بعد Connector.
- نسيت `output application/json`.
- نسيت Header `Content-Type`.
- غلط إملائي بمفتاح الـ Property.
- الـ Port مستخدم.
- ترتيب الـ Error Handlers غلط (`ANY` قبل المحدد).

---

## 10. القرارات التصميمية: الإيجابيات والسلبيات

> لكل قرار جاوب بهذا الترتيب: **(1) شنو اخترت؟ (2) شنو البدائل؟ (3) ليش اخترت هذا؟ (4) شنو عيبه؟ (5) شنو تغيّر بالإنتاج؟**
> إذا جاوبت الخمسة، تبين كأنك Senior حتى لو ما عندك خبرة بـ Mule.

### قرار 1: APIkit أو Listener عادي؟

| | APIkit ✅ | Listener عادي |
|---|---|---|
| إيجابيات | معيار الصناعة؛ المواصفات = توثيق + عقد؛ Routing وValidation تلقائي | بسيط؛ تتحكم بكل شي؛ أسرع بالبناء |
| سلبيات | "سحر" مولّد يحتاج فهم؛ أخطاء `APIKIT:*` إضافية | تعيد بناء كل شي؛ ما يكبر؛ ما في عقد |

> **قل:** "اخترت العقد أولاً حتى تكون المواصفات مصدر الحقيقة. الثمن هو هيكل مولّد أكثر، فتأكدت إني أفهم الـ Router وأنواع أخطائه."

### قرار 2: RAML أو OpenAPI؟

| RAML | OpenAPI (OAS 3) |
|---|---|
| مختصر وأصلي لـ MuleSoft | معيار عالمي، يُستخدم خارج Mule أيضاً |
| أقل شيوعاً خارج Mule | مطوّل شوي |

> اختر اللي تكتبه بثقة. قل: "كلاهما صحيح وأقدر أحوّل بينهما."

### قرار 3: التخزين (Object Store ✅ / JSON / متغيّر)

- **Object Store:** ميزة حقيقية؛ POST ثم GET يشتغلون. **العيب:** يتصفّر بإعادة التشغيل، وإعداد إضافي.
- **JSON ثابت:** أبسط. **العيب:** قراءة فقط.
- **متغيّر:** ما يفيد (يضيع بعد كل طلب).

> **قل:** "الـ Brief قال ما نحتاج DB. استخدمت Object Store حتى يكون POST ثم GET متسق، وينتظم من ملف JSON. بالإنتاج يكون هذا System API مدعوم بقاعدة بيانات حقيقية."

### قرار 4: أي Endpoints تروح للـ Backend؟

- **اخترنا:** فقط `GET /{id}` (حسب الـ Brief).
- **عيب:** البيانات موزّعة بين التخزين المحلي والـ Backend.
- **بالإنتاج:** كل شي يمر عبر الـ System API.

### قرار 5: الـ Mock Backend

| Node ✅ | WireMock | Mockoon | API عام |
|---|---|---|---|
| تحكم كامل، تحاكي التأخير والأخطاء | قوي، يحتاج Java | واجهة رسومية | بدون إعداد |
| يحتاج Node عند المقيّم | ملفات Mapping | صعب تخزينه بـ Git | ما تقدر تفرض 5xx أو Timeout |

### قرار 6: YAML أو `.properties`؟

- **YAML ✅:** هرمي ومقروء. **عيب:** أخطاء المسافات (Indentation).
- **.properties:** بسيط ومضمون. **عيب:** مسطّح ومتكرر.
- ممكن تبدّل إذا تعبت من المسافات.

### قرار 7: أسلوب الـ Validation

- **APIkit فقط:** مجاني، بس رسائله عامة وما يعطيك `details[]` المطلوبة.
- **Validation Module:** مقروء، بس **يتوقف عند أول خطأ**.
- **DataWeave مخصص ✅:** يجمع **كل** الأخطاء بالشكل المطلوب بـ `error-400.json`. **عيبه:** كود أكثر.

> **قل:** "الجواب 400 النموذجي يسرد عدة حقول فاشلة، لذلك الفشل عند أول خطأ ما يطابق العقد. أتحقق من كل الحقول بخطوة DataWeave وأرمي خطأ واحد مضبوط."

### قرار 8: Propagate أو Continue؟

- **Propagate ✅:** الخطأ يبقى خطأ (واضح للمراقبة). **عيب:** تحتاج تضبط الكود عبر الـ Listener.
- **Continue:** أبسط. **عيب:** الـ Flow يبين ناجح داخلياً، وسهل ترجّع 200 بالغلط.

### قرار 9: Error Handler عام أو لكل Flow؟

- **عام ✅:** شكل خطأ واحد، ما في تكرار، تغيير الشكل بمكان واحد.
- **لكل Flow:** مرونة، بس تكرار وعدم اتساق.

### قرار 10: تحويل أخطاء الـ Downstream

| الحالة | اخترنا | ليش |
|---|---|---|
| Backend 404 | **404** | العميل طلب شي مو موجود فعلاً |
| Backend 4xx ثاني | **502** | نحن اللي بنينا الطلب للـ Backend، فخطأ 4xx هناك هو مشكلة تكاملنا مو العميل |
| Backend 5xx | **502** | فشل الطرف الأعلى (Gateway) |
| Timeout | **504** | Gateway Timeout (الـ 408 يخص بطء **العميل**) |
| ما نقدر نوصل | **502** (أو 503) | 503 تعني "مؤقت جرّب لاحقاً" |
| غير متوقع | **500** | لا نكشف تفاصيل داخلية |

### قرار 11: 400 أو 422 للـ Validation؟

- **400 ✅:** يطابق الـ Brief والعينات.
- **422:** أدق، بس ما مطلوب.

### قرار 12: مصدر الـ Correlation ID

- **اخترنا:** نقبل `X-Correlation-ID` إذا موجود وصالح، وإلا نستخدم تبع Mule.
- **سلبية:** الثقة بمدخل العميل ← حدّد الطول ونظّف القيمة (لتجنب Log Injection).

### قرار 13: ماذا نسجّل بالـ Log؟

- **اخترنا:** Metadata فقط (correlationId، المسار، customerId، الكود، المدة، **أسماء** الحقول الفاشلة).
- **ليش:** شركة بطاقات/دفع (QiCard) تهتم جداً بالخصوصية.
- **عيب:** الـ Debug أصعب.

### قرار 14: الـ Timeout وإعادة المحاولة

- Response Timeout: **3000ms**، Connection Timeout: **2000ms** (من الـ Properties).
- **ليش:** البحث عن زبون لازم يكون سريع، والفشل السريع يحمي المتصل.
- **Retry:** بدون حالياً. إعادة المحاولة تخفي الفشل وتضاعف الحمل، وبالـ POST ممكن تسبب تكرار. الحل بالإنتاج: Circuit Breaker.

### قرار 15: ليش نحفظ `customerId` بمتغيّر؟

لأن `attributes` تُستبدل بعد الـ Connector. لو قرأتها بعده، تحصل على تبع الـ Backend.

### قرار 16: توليد الـ Customer ID

- **عدّاد `CUST-10001`:** يطابق العينة. **عيب:** يحتاج حالة مشتركة، ممكن يتصادم بعدة Nodes.
- **UUID:** بدون تصادم، بس ما يطابق العينة.
- **قل:** "للعرض عدّاد، وبالإنتاج UUID أو Sequence من قاعدة البيانات."

### قرار 17: تنظيم الملفات

مقسّمة حسب المسؤولية (api، implementation، global-config، error-handling، common). **ليش:** تلاقي أي شي بسرعة بالجلسة المباشرة.

### قرار 18: MUnit وPostman

نضيفهم **بعد** ما يشتغل كل المطلوب. يعطون انطباع نضج هندسي.

### قرار 19: أسئلة منتج مفتوحة (اكتب جوابها بالـ README)

| السؤال | المقترح |
|---|---|
| الحقول المطلوبة بـ POST | `firstName, lastName, email, mobileNumber` |
| فحص الإيميل | Regex بسيط |
| فحص الموبايل | يبدأ بـ `+` وأرقام فقط |
| إيميل مكرر | غير مفحوص (اذكره كتحسين مستقبلي: 409) |
| حقول غير معروفة | تُهمل |

---

## 11. أسئلة المقابلة المتوقعة

**عن MuleSoft:**
- شنو API-led connectivity؟ مثال من مشروعك؟
- الفرق بين Flow وSub-flow وPrivate Flow؟
- شنو داخل الـ Event؟ شنو بـ `attributes` بعد الـ Listener وبعد الـ HTTP Request؟
- ليش خزّنت `customerId` بمتغير؟
- شنو DataWeave؟ الفرق بين `map` و`mapObject` و`pluck`؟
- Propagate مقابل Continue؟
- شلون تحمّل إعدادات بيئة معينة؟
- شنو APIkit؟

**عن قراراتك:**
- ليش 502 مقابل 504؟
- وين يصير الـ Validation وليش؟
- شلون الـ Correlation ID يمر من البداية للنهاية؟
- شنو تغيّر بالإنتاج؟ (قاعدة بيانات حقيقية، Secure Properties، Retry + Circuit Breaker، سياسات API Manager، MUnit، Pagination، Idempotency للـ POST، تسجيل JSON منظّم).
- شنو حدود حلك؟ (يتصفّر عند إعادة التشغيل، بدون Authentication، بدون Pagination، بدون فحص تكرار). **قولها بنفسك، هذي نقطة قوة.**

**أساسيات Backend:**
- الفرق بين 400 و404 و422؟ و401 و403؟ و502 و503 و504؟
- أي طرق HTTP تكون Idempotent؟
- ليش ما نعرض Stack Trace للعميل؟
- ليش ما نسجّل بيانات شخصية؟

---

## 12. مصادر تعلّم (Mule 4 فقط)

- **MuleSoft Trailhead / Training:** مسارات "MuleSoft Basics" و"Anypoint Platform Development: Fundamentals (Mule 4)" (مجانية).
- **docs.mulesoft.com:** Mule Runtime 4.x ← Components, DataWeave, Error Handling, HTTP Connector.
- **DataWeave Playground:** developer.mulesoft.com/learn/dataweave (تمرّن فيه **يومياً**).
- قناة MuleSoft على YouTube.
- **StackOverflow / MuleSoft Community** (وسوم `mule4` و`dataweave`).

**سؤال ممتاز ترسله لهم** (الإيميل شجّعك تسأل):

> "أي نسخة Mule Runtime وStudio تفضّلون أستخدم؟ وهل تفضلون APIkit مع RAML أو OpenAPI؟ وهل في صيغة معيّنة للـ Logging؟"

هذا يبين مبادرة ويشيل التخمين.

---

## 13. قاموس سريع

| المصطلح | المعنى |
|---|---|
| Anypoint Studio | بيئة التطوير لـ Mule |
| Flow | تسلسل خطوات يبدأ من مصدر (Listener) |
| Event Source | اللي يبدأ الـ Flow (HTTP Listener، Scheduler) |
| Processor | خطوة داخل الـ Flow (Logger, Set Variable...) |
| Scope | حاوية خطوات (Try, For Each) |
| Router | تفرّع (Choice, Scatter-Gather) |
| Payload / Attributes / Vars | البيانات / معلومات المصدر / متغيراتك |
| Correlation ID | رقم تتبع الطلب |
| Connector | إضافة للتواصل مع نظام |
| DataWeave | لغة التحويل |
| APIkit | أداة تولّد Flows من المواصفات وتتحقق من الطلبات |
| RAML / OAS | لغات وصف الـ API |
| Exchange | مستودع Anypoint للـ Connectors والمواصفات |
| Object Store | تخزين مفتاح/قيمة بـ Mule |
| MUnit | إطار اختبار Mule |
| CloudHub / RTF | بيئات تشغيل Mule السحابية |
| API Manager | إدارة السياسات (Rate Limit, Client ID) |

---

## 14. طريقة عملنا سوية

- أنت تبني **خطوة خطوة** (حسب قسم 5).
- إذا علقت، ابعث لي: **رسالة الخطأ من الـ Console** + **الـ XML تبع الـ Flow** + **شنو كنت تتوقع**.
- أنا أشرح المفاهيم وأراجع الـ XML والـ DataWeave وأساعدك بالـ Debug.
- **ما أسوي Commit ولا Push**. أنت تسويها يدوياً.
- قبل الجلسة نسوي مقابلة تجريبية (أسئلة قسم 11 + مراجعة كود قسم 8.1).

**ابدأ باليوم 0، ولما تخلّص قولي.** 💪

</div>
