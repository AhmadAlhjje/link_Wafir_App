# ميزان — النظام المالي

Backend مالي مبني بـNode.js 20 وTypeScript وExpress وMySQL 8 وSequelize وفق Clean Architecture. العميل هو الحساب المالي، و`journal_entries` هو مصدر الحقيقة الوحيد للأرصدة.

## المتطلبا

- Node.js 20+
- MySQL 8+
- قاعدة بيانات فارغة بترميز `utf8mb4`

## التشغيل

```bash
npm install
cp .env.example .env
npm run db:migrate
npm run db:seed
npm run dev
```

على Windows PowerShell استخدم `Copy-Item .env.example .env`. اضبط اتصال MySQL و`JWT_SECRET` بطول 32 محرفًا على الأقل قبل التشغيل.

لإنشاء مدير أول اختياريًا، اضبط `INITIAL_ADMIN_NAME` و`INITIAL_ADMIN_USERNAME` و`INITIAL_ADMIN_PASSWORD` قبل `npm run db:seed`. تُشفّر كلمة المرور بـbcrypt ولا تُخزن كنص صريح.

## المعمارية

- `src/domain`: الكيانات والقيم المالية والقواعد الخالصة.
- `src/application`: حالات الاستخدام ومنافذ repositories والخدمات.
- `src/infrastructure`: Sequelize وMySQL وJWT وbcrypt وPino.
- `src/presentation`: Express وZod والمسارات والـmiddleware.
- `src/container`: Composition Root وربط الاعتماديات.
- `database`: migrations وseeders وإعداد Sequelize CLI.

اتجاه الاعتماد هو Presentation → Application → Domain. تنفذ Infrastructure المنافذ الداخلية، ولا تستخدم طبقات Domain أوApplication أوPresentation استعلامات Sequelize.

## قاعدة البيانات

الجداول التجارية المعتمدة فقط:

`admins`, `client_groups`, `clients`, `currencies`, `movement_types`, `movements`, `journal_entries`, `transfer_details`, `exchange_details`, `receipt_payment_details`, `notifications`.

لا يوجد جدول accounts؛ صف `clients` هو الحساب المالي. لا تحفظ الأرصدة في clients، بل تُحسب من قيود الحركات `POSTED`.

تطور المخطط يتم عبر migrations فقط:

```bash
npm run db:migrate
npm run db:migrate:undo
npm run db:seed
```

لا يستخدم المشروع `sync({ force: true })` أو `sync({ alter: true })`، ولا يستخدم `ON DELETE CASCADE` للسجل المالي.

## القاعدة المالية

- `US`: لنا على العميل.
- `THEM`: علينا للعميل.
- الصافي: `totalUS - totalTHEM`.
- القبض: `THEM`.
- الدفع: `US`.
- التصريف: العملة المستلمة من العميل `THEM`، والمسلّمة إليه `US`.
- الحسابات المالية تستخدم `decimal.js` وقيم DECIMAL النصية، وليس JavaScript Number.

كل حركة مالية تنفذ ذريًا بواسطة `SequelizeUnitOfWork`. أي فشل في الرأس أو التفاصيل أو القيود يؤدي إلى rollback كامل.

## الحركات

- الحوالة: movement + transfer detail + journal entries.
- التسوية والمتعددة: movement + journal entries.
- القبض والدفع: movement + receipt/payment detail + journal entry.
- التصريف: movement + exchange detail + journal entries.
- الإلغاء يغيّر الحالة دون حذف.
- العكس يقلب أطراف الحركة **نفسها** في مكانها (`reversed_at/reversed_by`)؛ التعديل (`PUT /movements/<type>/:id`) يعيد كتابة الحركة نفسها بنفس الرقم.
- أرقام الحركات متسلسلة **1، 2، 3…** (`nextNumber` بقفل `FOR UPDATE` داخل المعاملة)؛ هجرة `20260917000100` تعيد ترقيم القديم.
- سعر صرف الدولار قابل للتعديل؛ التقييم بالدولار يستعمل السعر المخزّن لكل عملة بما فيها `USD`.

## المكاتب والترخيص

باك اند واحد لكل المكاتب: جدول `offices` (كود المكتب + الترخيص) وعمود `office_id` في كل جدول تجاري، والعزل تلقائي عبر `tenancy-hooks`. المكاتب تُنشأ من **لوحة التحكم** (`../dashboard`) عبر `/api/v1/platform/*` بمفتاح `PLATFORM_API_KEY`؛ الإنشاء يولّد الكود ويبذر العملات والصناديق النظامية ومدير المكتب. تسجيل الدخول: كود المكتب + اسم المستخدم + كلمة المرور. الترخيص (`ACTIVE`/`SUSPENDED`/`EXPIRED` + `expires_at` + `message`) يُغيَّر من اللوحة ويصل القفل/رفعه لأجهزة المكتب خلال ≤`LICENSE_POLL_SECONDS` ثوانٍ عبر SSE؛ الطلبات ترجع `403 LICENSE_*`. **حالة الإداري تُفحص مع كل طلب** (`authenticate`): إداري معطَّل ⇒ `403 ADMIN_DEACTIVATED` حتى لو كان توكنه صالحاً، ويُبثّ حدث SSE `account {isActive}` لأجهزته لحظة التعطيل/التفعيل؛ `GET /auth/me` يخدم شاشة القفل في التطبيق. البيانات الموجودة قبل الهجرة تُنسب إلى المكتب الافتراضي (`DEFAULT_OFFICE_CODE` أو كود يُطبع في سجل الهجرة).

## أهم المسارات

- `GET /api/v1/license?office=<code>` — حالة ترخيص مكتب (بلا مصادقة)
- `POST /api/v1/auth/login` — `{officeCode | deviceKey, fullName, password}`: كود المكتب **لمرة واحدة** عند أول دخول على جهاز (يُصدَر `deviceKey` دائم في الرد ويتبدّل الكود فوراً — الجديد يظهر في اللوحة فقط)؛ الدخول اللاحق بـ `deviceKey`. جهاز مُلغى ⇒ `404 DEVICE_NOT_FOUND`. جدول `office_devices` (المفتاح مجزَّأ SHA-256).
- `GET /api/v1/license` — بترويسة `X-Device-Key` (التطبيق) أو `?office=CODE`.
- **حد الحركات** (2026-09-22): `offices.movement_limit` (null = بلا حد) و`movements_used` (يزيد بالإضافة فقط — التعديل/الإلغاء/العكس لا يمرّ بـ create). بلوغه **لا يقفل التطبيق**: مسارات الإضافة `POST /movements/*` فقط ترد `403 LICENSE_LIMIT_REACHED` (`requireMovementQuota`)، والعرض/التعديل/الحذف يعملان. الحد من `PUT /platform/offices/:id/license` بـ `movementLimit`، والتصفير من اللوحة فقط `POST /platform/offices/:id/movements/reset`. `GET /dashboard` يعيد `totalMovementsCount` (العدّاد) و`movementLimit`. سند القبض/الدفع يقبل `cashBoxClientId` اختيارياً (حساب صندوق) يُحفظ في `second_client_id` للعرض/السجل — القيد يبقى على حساب العميل كما هو.
- `GET/DELETE /api/v1/platform/offices/:id/devices[/:deviceId]` — أجهزة المكتب وإلغاؤها.
- `DELETE /api/v1/platform/offices/:id` (2026-09-23) — حذف مكتب بكل بياناته: يتطلب `{ confirm: "<كود المكتب>" }` حرفياً، ويحذف في معاملة واحدة القيود وتفاصيل الحركات والإشعارات والحركات والعملاء والمجموعات والعملات والإداريين والأجهزة ثم صفّ المكتب (قيود FK هي RESTRICT فالترتيب إلزامي)، ويحذف ملف اللوغو ويحدّث مراقب الترخيص.
- **كشف «كل العملات» لحساب مدوَّر** (إصلاح 2026-09-26): كشف كل العملات يمرّر `currencyId = undefined`، وSequelize يرمي عند وجود قيمة undefined داخل `where` — فكان الكشف يفشل بـ 500 «خطأ في الخادم» لكل حساب له `last_rollover_at` (لأن الرصيد الافتتاحي يُحسب عندها دائماً). الحلّ: `clientCurrencyWhere()` في `infrastructure/database/sequelize/where.ts` تحذف المفتاح بدل تمرير undefined، وحدّا الطيّ `fromAt/beforeAt` صارا كائن `Date` لا نصّ ISO (نصّ `…T…Z` غير صالح للمقارنة في MySQL)، وألغيت التسمية `rows` في الاستعلام المجمَّع (كلمة محجوزة في MariaDB). يحرسه `tests/unit/AllCurrenciesStatementWhere.test.ts`.
- **طيّ ما قبل تدوير الأرصدة** (2026-09-23): `GET /clients/:id/statement?collapse_rollover=true` يبدأ الفترة من لحظة `clients.last_rollover_at` ويعيد `rolledOver { at, count, totalUs, totalThem, balance, byCurrency? }` = سطر واحد يجمع كل ما قبلها (يساوي الرصيد الافتتاحي تماماً فلا ازدواج ولا فجوة)؛ و`&scope=ROLLED_OVER` يعيد تلك القيود نفسها (للمراجعة فقط، بترقيم صفحات — `limit ≤ 100` كبقية المسارات وإلا 422). المرشِّحان `fromAt`/`beforeAt` في `JournalFilters` يعملان بدقّة الثانية بخلاف `date_from/date_to`.
- **المبالغ بمنزلتين** (2026-09-24): `AMOUNT_SCALE = 2` — كل مبلغ وارد يُقرَّب في مخطّطات zod عبر `amountString` (`toAmount`) قبل الحساب والتخزين، وكل مبلغ محسوب يعود بمنزلتين (`ZERO_AMOUNT = '0.00'`). **أسعار الصرف وحدها** تبقى بعشر منازل (`RATE_SCALE`) لأنها معامِل لا مبلغ (سعر مثل 0.0000666 يصبح صفراً لو قُرِّب). أعمدة القاعدة تبقى DECIMAL(30,10) فلا هجرة؛ السجلات الأقدم بدقّة أعلى تُعرض مقرَّبة، وبقايا أقلّ من قرش تُعدّ صفراً (وإلا تعذّرت أرشفة الحساب).
- **إعلان إيقاف المنصّة** (2026-09-23): جدول `platform_notice` (صفّ واحد) — `GET/PUT /api/v1/platform/notice` من اللوحة `{ isActive, title?, message }`، و`GET /api/v1/system/notice` بلا مصادقة للتطبيق. مفعَّلاً يرفض **تسجيل الدخول** بـ `403 SYSTEM_NOTICE` (قبل فحص المكتب وكلمة المرور)، ولا يُفحص في `authenticate` عمداً: الجلسة الجارية تُكمل عملها ولا تُقاطع — الرسالة تظهر عند إقلاع التطبيق أو عند دخول جديد فقط. الهجرة `20260923000100-platform-notice.cjs`.
- `GET /api/v1/clients/:id/statement` — `currency_id` **اختياري**: بلا عملة يعيد كشفاً لكل العملات (عملة لكل قيد، `totalsByCurrency`، بلا رصيد جارٍ/افتتاحي/ختامي — قرار المستخدم 2026-09-22).
- **جلسة واحدة لكل حساب** (2026-09-22): «الحساب مفتوح» = له بثّ SSE حيّ من جهاز (`NotificationHub.claim/release` عند فتح/إغلاق `/notifications/stream`؛ الجهاز من `deviceId` في التوكن). دخول أو أي طلب من جهاز آخر أثناء ذلك ⇒ `409 ACCOUNT_IN_USE` («الحساب مفتوح على حاسوب آخر»)؛ إغلاق التطبيق يقطع البثّ فيتحرّر الحساب خلال ثوانٍ، والجهاز نفسه لا يحجب نفسه.
- الأهمية (`importance`) قابلة للتعديل في العملات الأساسية أيضاً.
- `GET /api/v1/auth/me` — الحساب الحالي ومكتبه (معطَّل ⇒ `403 ADMIN_DEACTIVATED`)
- `PUT/DELETE /api/v1/platform/offices/:id/logo` — لوغو المكتب من لوحة التحكم (multipart `logo`، ≤20MB PNG/JPG/WEBP؛ الملف تحت `uploads/offices`، ويُبثّ حدث SSE `office` بالمعلومات العامة `{id,code,name,address,phone,logoUrl,logoVersion}`؛ نفس الشكل يعود مع الدخول و`GET /license`). الهجرة `20260922000100-office-logo.cjs`.
- `/api/v1/platform/*` — واجهة لوحة التحكم (مفتاح `X-Platform-Key`)
- `POST /api/v1/auth/login` — `{ officeCode, fullName, password }`
- `GET|POST|PATCH /api/v1/admins`
- `GET|POST|PATCH /api/v1/client-groups`
- `GET|POST|PATCH /api/v1/clients`
- `GET|POST|PATCH /api/v1/currencies`
- `GET|PATCH /api/v1/movement-types`
- `POST /api/v1/movements/transfers`
- `POST /api/v1/movements/settlements`
- `POST /api/v1/movements/multi`
- `POST /api/v1/movements/receipts`
- `POST /api/v1/movements/payments`
- `POST /api/v1/movements/exchanges`
- `GET /api/v1/movements/:id`
- `POST /api/v1/movements/:id/cancel`
- `POST /api/v1/movements/:id/reverse`
- `GET /api/v1/journal`
- `GET /api/v1/clients/:id/statement`
- `GET /api/v1/notifications`
- `PATCH /api/v1/notifications/:id/read`

جميع المسارات، عدا تسجيل الدخول وhealth، تتطلب JWT والصلاحية المناسبة. الاستجابات الخاطئة موحدة ولا تكشف أخطاء Sequelize أو SQL.

## كشف الحساب

يدعم العملة، الفترة، نوع الحركة، الصفحة، والحد. يحسب الرصيد الافتتاحي من القيود السابقة لـ`date_from`، ثم الرصيد الجاري بترتيب ثابت:

`movement_date, movement_time, movement_id, line_no`.

## السجلات والأمان

- Helmet وCORS وrate limiting.
- Pino logging مع `requestId` وإخفاء password وhash وJWT وAuthorization.
- تحويل أخطاء uniqueness وforeign key والتحقق والاتصال إلى أخطاء API آمنة.
- لا يوجد hard delete للحركات أو العملاء أو العملات.

## الجودة والاختبارات

```bash
npm run typecheck
npm run lint
npm run format:check
npm test
npm run build
```

الاختبارات الحالية تغطي المال والدقة، الحسابات، اتجاهات القبض والدفع والتصريف، إنشاء الحركات، rollback المنطقي، كشف الحساب، الإلغاء والعكس، إدارة البيانات، وإخفاء كلمات المرور وخريطة أخطاء قاعدة البيانات.

اختبارات التكامل وE2E التي تتصل بـMySQL تحتاج قاعدة اختبار مستقلة عبر `DB_TEST_NAME`. لا تشغّلها على قاعدة الإنتاج.

## المرجع الحاكم

يجب قراءة `role.md` قبل أي تعديل متعلق بالأعمال أو قاعدة البيانات. يمنع إضافة جدول أو حقل أعمال أو تغيير قاعدة مالية دون موافقة صريحة.

## النشر على VPS بـ Docker

`Dockerfile` (صورة تشغيل خفيفة مع sequelize-cli) + `docker-compose.yml` (المنفذ 5002، مجلد دائم `uploads_data` لأيقونات العملات) + `docker/entrypoint.sh` (ينتظر القاعدة عبر `docker/wait-for-db.cjs` ثم يشغّل الهجرات تلقائياً ثم الخادم). القاعدة هي حاوية MariaDB التي تنشئها لوحة التحكم (`dashboard/server`) على الشبكة المشتركة `wafeer-net` (`docker network create wafeer-net` مرة واحدة، ثم شغّل لوحة التحكم أولاً). الإعدادات من `.env` (انسخ `.env.example`)؛ `DB_NAME/DB_USER/DB_PASSWORD` و`PLATFORM_API_KEY` يجب أن تطابق `dashboard/server/.env`. عند أول تشغيل على قاعدة فارغة تُنشئ الهجرة مكتباً افتراضياً بكوده في السجل (أو `DEFAULT_OFFICE_CODE`)؛ المكاتب الحقيقية تُنشأ من لوحة التحكم. أنواع الحركات الستة (بيانات مرجعية) تضمنها الهجرة `20260922000200-ensure-movement-types.cjs` فلا حاجة لتشغيل seeders على الخادم. الخادم الحالي: `163.245.221.210:5002` (افتحه في الجدار الناري لتطبيق وفير المكتبي).

### النسخ الاحتياطي على الخادم
`docker/backup.sh` يأخذ لقطة متسقة لقاعدة البيانات (كل المكاتب) + مجلد `uploads` إلى `/opt/backups/<التاريخ>/` ويحذف ما هو أقدم من 14 يوماً؛ يُجدوَل بـ cron يومياً. `docker/restore.sh <مجلد النسخة>` يستعيدها (يستبدل القاعدة كاملة). انسخ `/opt/backups` دورياً خارج الخادم (rclone/scp) — نسخة على نفس الجهاز لا تحمي من ضياعه.
