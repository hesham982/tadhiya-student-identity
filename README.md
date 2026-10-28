# اعدادية التضحية للبنين — صفحة هوية الطالب

نظام تسجيل بيانات الطلاب عبر Telegram Mini App مع خلفية Supabase.
تم تقليل البيانات المجمّعة إلى **سنة الميلاد فقط** بدلاً من تاريخ الميلاد الكامل،
وتمت إزالة Firebase ورمز PIN بالكامل.

## محتويات المشروع

```
tadhiya-student-identity.html          الواجهة (ملف واحد قابل للنشر)
supabase/migrations/                   سكربتات قاعدة البيانات والتخزين
supabase/functions/students/index.ts   Edge Function للتحقق والحفظ والقائمة والحذف
PRIVACY.md                             سياسة الخصوصية والاحتفاظ بالبيانات
```

## 1) الواجهة — إعداد الاتصال

افتح `tadhiya-student-identity.html` وابحث عن `APP_CONFIG` (قرب بداية الـ script)،
ثم ضع عنوان مشروعك:

```js
const APP_CONFIG = {
    apiBaseUrl: "https://gjyldmqwzlwbuuorqthc.supabase.co/functions/v1",
    apiKey: "sb_publishable_6OlMuW-2AKuecHPMS0mmWA_yJsbphVP",
    functionName: "students"
};
```

> `sb_publishable_...` مفتاح عام مخصص للمتصفح (مثل anon key) — لا بأس بوجوده في الواجهة.
> لا تضع `service_role key` في الواجهة إطلاقاً. الواجهة تعتمد على `initData`،
> والخادم وحده يملك مفتاح الخدمة.

حدود سنة الميلاد في الواجهة: `1990` حتى السنة الحالية (تُضبط تلقائياً)،
وهي نفس الحدود المطبَّقة على الخادم.

## 2) إنشاء مشروع Supabase

```bash
supabase login
supabase link --project-ref gjyldmqwzlwbuuorqthc

# إنشاء الجدول + حاوية الصور الخاصة
supabase db push
```

إذا كانت لديك قاعدة بيانات قائمة، شغّل أيضاً:

```sql
alter table public.students add column if not exists birth_year smallint
  check (birth_year between 1950 and 2100);
```

ثم رحّل السنة فقط من العمود القديم (إن وجد) واحذف عمود التاريخ الكامل:

```sql
update public.students
   set birth_year = extract(year from student_birth_date)::smallint
 where birth_year is null and student_birth_date is not null;

alter table public.students drop column if exists student_birth_date;
```

### البنية الناتجة

| العنصر | الوصف |
|---|---|
| `public.students` | الاسم الرباعي، `birth_year`، هاتف ولي الأمر، مسار الصورة، معرّف تيليجرام، وقت التسجيل |
| RLS | مفعّل بدون أي policy — لا وصول مباشر لـ anon/authenticated |
| `storage.buckets: student-photos` | **خاصة** (public = false)، حد 2MB، صور فقط |
| الروابط | وكيل صور داخل Edge Function برمز HMAC قابل للإلغاء (ساعة واحدة) |

### مواصفات الصورة

- تُقصّ الصورة تلقائياً إلى إطار **بورتريه 3:4** من منتصف الكاميرا (وليس الصورة كاملة).
- الحد الأقصى 550 بكسل للضلع الأطول → الناتج 413×550 تقريباً، JPEG بجودة 0.75.
- الإطار البيضوي في شاشة الكاميرا يُظهر منطقة الوجه المطلوبة.

### لماذا وكيل الصور وليس رابطاً موقعاً من التخزين؟

روابط Storage الموقعة تبقى مخزّنة مؤقتاً في شبكة CDN بعد حذف الملف، لذا قد تُعرض
صورة طالب محذوف لبعض الوقت. لذلك تُعرض الصور عبر:

```
GET /functions/v1/students?action=photo&id=<uuid>&exp=<unix>&sig=<hmac>
```

- `sig = HMAC-SHA256(مفتاح, "photo:<id>:<exp>")`.
- المفتاح من `PHOTO_URL_SECRET` إن وُجد، وإلا يُشتق من رمز البوت.
- الصورة لا تُقدَّم إلا إذا كان **السجل ما زال موجوداً** في `public.students`،
  لذا يتوقف الرابط فوراً بعد الحذف (404).
- كل الاستجابات بترويسة `Cache-Control: no-store`.

## 3) أسرار Edge Function

احصل على معرّفك الرقمي في تيليجرام من `@userinfobot`، ثم:

```bash
supabase secrets set \
  TELEGRAM_BOT_TOKEN="123456:ABC..." \
  ADMIN_TELEGRAM_IDS="111111111,222222222" \
  PHOTO_URL_SECRET="$(openssl rand -hex 32)"
```

متغيرات اختيارية:

| المتغير | الافتراضي | الوظيفة |
|---|---|---|
| `PHOTO_BUCKET` | `student-photos` | اسم حاوية الصور |
| `MIN_BIRTH_YEAR` | `1990` | الحد الأدنى لسنة الميلاد |
| `PHOTO_URL_SECRET` | مشتق من رمز البوت | مفتاح توقيع روابط الصور (دوّره عند الحاجة) |
| `ALLOWED_ORIGIN` | `*` | أصل الواجهة المسموح (يُفضّل ضبطه بعد النشر) |

## 4) نشر الدالة

```bash
supabase functions deploy students
```

`verify_jwt = false` مضبوط مسبقاً في `supabase/config.toml`، لأن تيليجرام لا يستخدم
JWT الخاص بـ Supabase؛ المصادقة تتم داخل الدالة عبر `initData`.

### نقاط النهاية

| الطلب | من يستطيع | الوظيفة |
|---|---|---|
| `GET ?action=whoami` | أي مستخدم تيليجرام صالح | هل الحساب مشرف؟ |
| `GET ?action=photo&...` | صاحب رابط موقّع غير منتهٍ | عرض الصورة (بدون initData) |
| `GET /students` | المشرف فقط (403 لغيره) | قائمة الطلاب + روابط صور موقعة |
| `POST /students` | أي مستخدم تيليجرام صالح | حفظ طالب جديد |
| `DELETE ?id=<uuid>` | المشرف فقط | حذف السجل + الصورة |

### اختبار سريع

احصل على `initData` من داخل التطبيق (كونسول المتصفح داخل تيليجرام):

```js
window.Telegram.WebApp.initData
```

ثم:

```bash
INIT_DATA="query_id=...&user=...&auth_date=...&hash=..."

# معرفة الصلاحية
curl -s "https://gjyldmqwzlwbuuorqthc.supabase.co/functions/v1/students?action=whoami" \
  -H "Authorization: tma $INIT_DATA"

# القائمة (يجب أن تعمل للمشرف فقط)
curl -s "https://gjyldmqwzlwbuuorqthc.supabase.co/functions/v1/students" \
  -H "Authorization: tma $INIT_DATA"

# سنة ميلاد خارج النطاق (يجب أن تُرفض 400)
curl -s -X POST "https://gjyldmqwzlwbuuorqthc.supabase.co/functions/v1/students" \
  -H "Authorization: tma $INIT_DATA" \
  -H "Content-Type: application/json" \
  -d '{"studentName":"أحمد محمد علي حسن","birthYear":1901,"guardianPhone":"07701234567","photoBase64":"data:image/jpeg;base64,..."}'
```

## 5) النشر والربط بالبوت

النشر الحالي:

| العنصر | القيمة |
|---|---|
| رابط التطبيق | https://hesham982.github.io/tadhiya-student-identity/ |
| المستودع | https://github.com/hesham982/tadhiya-student-identity |
| الدالة | https://gjyldmqwzlwbuuorqthc.supabase.co/functions/v1/students |
| زر القائمة في البوت | «تسجيل الطالب» ← رابط التطبيق |

تحديث الواجهة بعد أي تعديل محلي:

```powershell
Copy-Item 'D:\New folder (2)\tadhiya-student-identity.html' "$env:TEMP\tadhiya-pages\index.html" -Force
cd "$env:TEMP\tadhiya-pages"
git add -A; git commit -m "update"; git push
```

بديل: Cloudflare Pages أو Netlify (ملف واحد فقط). بعد تغيير النطاق اضبط
`ALLOWED_ORIGIN` ثم أعد نشر الدالة:

```bash
supabase secrets set ALLOWED_ORIGIN="https://your-site.pages.dev"
supabase functions deploy students
```

ضبط زر القائمة برمجياً (بدل BotFather):

```bash
curl -s -X POST "https://api.telegram.org/bot$TOKEN/setChatMenuButton" \
  -H "Content-Type: application/json" \
  -d '{"menu_button":{"type":"web_app","text":"تسجيل الطالب","web_app":{"url":"https://hesham982.github.io/tadhiya-student-identity/"}}}'
```

## 6) قائمة الاختبار (المرحلة الرابعة)

- [ ] التسجيل من حساب تيليجرام عادي ينجح، ولا يستطيع رؤية بيانات الآخرين.
- [ ] فتح لوحة الإدارة بحساب غير مشرف يعيد **403** (رسالة رفض واضحة).
- [ ] `birthYear = 1901` أو `2050` يُرفض من الخادم (400) وليس من الواجهة فقط.
- [ ] رفع ملف ليس صورة (مثلاً PDF) يُرفض من الخادم (400) حتى لو غُيّر الـ MIME.
- [ ] صورة أكبر من 2MB تُرفض.
- [ ] طلب بدون `initData` أو بـ `initData` منتهٍ يعيد 401.
- [ ] حذف سجل يحذف الصورة من الحاوية أيضاً، ورابط الصورة القديم يعيد **404**.
- [ ] فتح رابط صورة بعد تعديل `sig` أو `id` يعيد 403.

## 7) ملاحظات أمنية مهمة

- لا تُصدر أي سياسة RLS تسمح لـ `anon` بقراءة `students` أو `storage.objects`.
- تحقق الخادم من توقيع `initData` بـ HMAC-SHA256 وعمر الجلسة (24 ساعة).
- صلاحية المشرف تُدار عبر `ADMIN_TELEGRAM_IDS` فقط، ولا توجد كلمة مرور في الواجهة.
- الصور تُفحص ببصمتها الحقيقية (magic bytes) وليس بالامتداد أو `Content-Type`.
- روابط الصور موقّعة بـ HMAC ومرتبطة بمعرّف السجل، وتُرفض بعد الحذف أو انتهاء المدة.
- احتفظ بـ `service_role key` داخل Supabase فقط.
