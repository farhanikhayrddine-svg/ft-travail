/* server.js - مشروع FT travail
   يستقبل الطلبات، يحفظ البيانات، يرسل البريد والواتساب تلقائي */
const express = require('express');
const multer = require('multer');
const path = require('path');
const bodyParser = require('body-parser');
const nodemailer = require('nodemailer');
const basicAuth = require('express-basic-auth');
const fs = require('fs');
const cors = require('cors');
const twilio = require('twilio');
const Database = require('better-sqlite3');

// إعداد السيرفر
const app = express();
app.use(cors());
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());
// إتاحة الوصول للمجلدات الثابتة (uploads و public)
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));
app.use(express.static(path.join(__dirname, 'public')));

// قاعدة بيانات
const db = new Database('db.sqlite');
db.prepare(`CREATE TABLE IF NOT EXISTS applicants (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    firstname TEXT,
    lastname TEXT,
    address TEXT,
    country TEXT,
    birthdate TEXT,
    education TEXT,
    email TEXT,
    whatsapp TEXT,
    idFront TEXT,
    idBack TEXT,
    status TEXT
)`).run();

// إعدادات ملفات الهوية (Multer)
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        // إنشاء مجلد uploads إذا لم يكن موجودًا
        if (!fs.existsSync('uploads')) {
            fs.mkdirSync('uploads');
        }
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => cb(null, Date.now() + '_' + file.originalname)
});
const upload = multer({ storage });

// قراءة متغيرات البيئة
require('dotenv').config();
const ADMIN_USER = process.env.ADMIN_USER || 'admin';
const ADMIN_PASS = process.env.ADMIN_PASS || 'admin';
const PORT = process.env.PORT || 10000;

// إعداد خدمة البريد الإلكتروني (SMTP)
const transporter = nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port: process.env.SMTP_PORT,
    secure: process.env.SMTP_SECURE === 'true',
    auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS
    }
});

// إعداد خدمة الرسائل (Twilio)
const client = twilio(process.env.TWILIO_SID, process.env.TWILIO_TOKEN);

// المسار: استقبال طلب التقديم
app.post('/apply', upload.fields([{name:'idFront'},{name:'idBack'}]), (req,res)=>{
    const files = req.files;
    const body = req.body;
    
    if (!files.idFront || !files.idBack) {
        return res.status(400).send('الرجاء إرفاق صورة الهوية الأمامية والخلفية.');
    }

    db.prepare(`INSERT INTO applicants (firstname, lastname, address, country, birthdate, education, email, whatsapp, idFront, idBack, status)
        VALUES (?,?,?,?,?,?,?,?,?,?,?)`)
      .run(body.firstname, body.lastname, body.address, body.country, body.birthdate, body.education, body.email, body.whatsapp,
        files.idFront[0].filename, files.idBack[0].filename, 'pending');
        
    res.redirect('/thanks.html');
});

// المسار: لوحة الإدارة (حماية بـ Basic Auth)
app.use('/admin.html', basicAuth({users:{[ADMIN_USER]:ADMIN_PASS}, challenge:true}));

// المسار: جلب بيانات المتقدمين
app.get('/api/applicants', (req,res)=>{
    const rows = db.prepare('SELECT * FROM applicants').all();
    res.json(rows);
});

// المسار: تحديث حالة المتقدم وإرسال الإشعارات
app.post('/api/applicants/:id/status', async (req,res)=>{
    const {status} = req.body;
    const id = req.params.id;
    const applicant = db.prepare('SELECT * FROM applicants WHERE id=?').get(id);
    
    if(!applicant) return res.status(404).send('Applicant not found');
    
    db.prepare('UPDATE applicants SET status=? WHERE id=?').run(status,id);

    // إرسال بريد
    let subject, text;
    if(status==='accepted'){
        subject = 'تهانينا — قبول طلب التوظيف';
        text = `تهانينا ${applicant.firstname}! أنت مقبول في العمل معنا. سنتواصل معك على الواتساب خلال 24 ساعة.`;
   
        // إرسال واتساب (فقط إذا كانت البيانات متوفرة)
        if(applicant.whatsapp && process.env.TWILIO_SID && process.env.TWILIO_WHATSAPP_FROM){
            try {
                await client.messages.create({
                    from: 'whatsapp:' + process.env.TWILIO_WHATSAPP_FROM,
                    to: 'whatsapp:' + applicant.whatsapp,
                    body: text
                });
            } catch (error) {
                console.error('Twilio Error:', error.message);
                // يمكن أن يحدث هذا إذا كان الرقم غير صحيح أو غير مفعل في Twilio
            }
        }
    }else{
        subject = 'نتيجة طلب التوظيف';
        text = `مرحبًا ${applicant.firstname}, نعتذر، أنت غير مؤهل للعمل معنا.`;
    }

    transporter.sendMail({
        from: process.env.EMAIL_FROM,
        to: applicant.email,
        subject,
        text
    }).catch(err => console.error('Nodemailer Error:', err.message));
    
    res.send('Status updated');
});

// بدء تشغيل الخادم
app.listen(PORT, ()=>console.log(`Server running on http://localhost:${PORT}`));
{
  "name": "ft-travail-app",
  "version": "1.0.0",
  "description": "Application for managing job applications with automated email and WhatsApp notifications.",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "better-sqlite3": "^8.4.0",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "multer": "^1.4.5-lts.1",
    "nodemailer": "^6.9.4",
    "express-basic-auth": "^1.2.0",
    "body-parser": "^1.20.2",
    "twilio": "^4.6.0",
    "dotenv": "^16.3.1"
  }
}
# إعدادات الخادم
PORT=10000

# بيانات دخول لوحة الإدارة
ADMIN_USER=admin_user
ADMIN_PASS=khayrii123@

# إعدادات البريد الإلكتروني (مثال على Gmail)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=yourfarhanikhayrddine@gmail.com
SMTP_PASS=khayri123@
EMAIL_FROM="FT travail <farhanikhayrddine@gmail.com>"

# إعدادات Twilio (لرسائل WhatsApp)
TWILIO_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_WHATSAPP_FROM=+21692527066 # رقم واتساب Twilio الخاص بك
/node_modules
/db.sqlite
.env
# FT travail - نظام طلبات التوظيف التلقائي

هذا مشروع Node.js يستخدم Express لإدارة طلبات التوظيف، وحفظ بيانات المتقدمين وصور هوياتهم، وإرسال إشعارات تلقائية عبر البريد الإلكتروني والواتساب عند تحديث الحالة.

## 🚀 التقنيات المستخدمة
* **الخادم:** Node.js و Express.
* **قاعدة البيانات:** Better SQLite3.
* **معالجة الملفات:** Multer لحفظ صور الهوية.
* **البريد الإلكتروني:** Nodemailer.
* **الواتساب:** Twilio API.
* **الحماية:** express-basic-auth للوحة الإدارة.

## 🛠️ خطوات التشغيل

### 1. الإعداد الأولي
1.  استنسخ المستودع (Clone) أو قم بتنزيله.
2.  انتقل إلى المجلد الرئيسي للمشروع.
3.  ثبت التبعيات: `npm install`

### 2. إعداد المتغيرات البيئية
1.  أنشئ ملف باسم **`.env`** في المجلد الجذر.
2.  انسخ محتوى ملف `.env.example` إليه واملأ بياناتك السرية (كلمات المرور وتوكنات API).

### 3. تشغيل الخادم
* **تشغيل الإنتاج:** `npm start`
* **تشغيل التطوير (إذا كان لديك Nodemon):** `npm run dev`

سيتم تشغيل الخادم على المنفذ المحدد.

## 🔑 الوصول
* **نموذج التقديم:** `http://localhost:10000/index.html`
* **لوحة الإدارة:** `http://localhost:10000/admin.html` (تتطلب مصادقة)
* 

<!DOCTYPE html>
<html lang="ar">
<head>
<meta charset="UTF-8">
<title>تقديم طلب وظيفة - FT travail</title>
</head>
<body>
<h2>نموذج طلب الوظيفة</h2>
<form action="/apply" method="post" enctype="multipart/form-data">
  الاسم: <input type="text" name="firstname" required><br>
  اللقب: <input type="text" name="lastname" required><br>
  العنوان الحالي: <input type="text" name="address" required><br>
  البلد: <input type="text" name="country" required><br>
  تاريخ الولادة: <input type="date" name="birthdate" required><br>
  المستوى التعليمي: <input type="text" name="education" required><br>
  الايميل: <input type="email" name="email" required><br>
  رقم الواتساب: <input type="text" name="whatsapp" placeholder="مثال: +9665xxxxxxxx"><br>
  صورة الهوية (الجهة الأمامية): <input type="file" name="idFront" accept="image/*" required><br>
  صورة الهوية (الجهة الخلفية): <input type="file" name="idBack" accept="image/*" required><br>
  <button type="submit">إرسال الطلب</button>
</form>
</body>
</html>
<!DOCTYPE html>
<html lang="ar">
<head><meta charset="UTF-8"><title>شكراً</title></head>
<body>
<h2>تم استلام طلبك</h2>
<p>طلبكم تحت الدراسة، في ظرف أقل من 24 ساعة سيتم الإجابة على الإيميل، شكراً.</p>
</body>
</html>
<!DOCTYPE html>
<html lang="ar">
<head>
    <meta charset="UTF-8">
    <title>لوحة الإدارة</title>
    <style>
        body { font-family: sans-serif; }
        .applicant-card { border: 1px solid #ccc; padding: 10px; margin-bottom: 10px; }
        .applicant-card button { margin-left: 10px; padding: 5px 10px; cursor: pointer; }
        .accepted { color: green; font-weight: bold; }
        .rejected { color: red; font-weight: bold; }
    </style>
</head>
<body>
<h2>لوحة الإدارة - FT travail</h2>
<div id="applicants"></div>

<script>
    const baseUrl = window.location.origin;

    async function loadApplicants(){
        const div = document.getElementById('applicants');
        div.innerHTML = 'جاري التحميل...';
        
        try {
            // ملاحظة: يتم تمرير بيانات Basic Auth عبر المتصفح تلقائياً
            const res = await fetch('/api/applicants');
            if (res.status === 401) {
                div.innerHTML = 'الرجاء تحديث الصفحة وإدخال اسم المستخدم وكلمة المرور.';
                return;
            }
            const data = await res.json();
            
            div.innerHTML = '';
            
            if (data.length === 0) {
                div.innerHTML = 'لا يوجد متقدمون حاليًا.';
                return;
            }

            data.forEach(a=>{
                const card = document.createElement('div');
                card.className = 'applicant-card';
                
                let statusText = '';
                if (a.status === 'accepted') {
                    statusText = `<span class="accepted">مقبول</span>`;
                } else if (a.status === 'rejected') {
                    statusText = `<span class="rejected">مرفوض</span>`;
                } else {
                    statusText = 'قيد الانتظار';
                }

                card.innerHTML = `
                    <h3>${a.firstname} ${a.lastname} (${statusText})</h3>
                    <p><strong>الإيميل:</strong> ${a.email}</p>
                    <p><strong>الواتساب:</strong> ${a.whatsapp || 'غير متوفر'}</p>
                    <p><strong>البلد:</strong> ${a.country} - <strong>المستوى التعليمي:</strong> ${a.education}</p>
                    <p>
                        <a href="${baseUrl}/uploads/${a.idFront}" target="_blank">صورة الهوية (أمام)</a> | 
                        <a href="${baseUrl}/uploads/${a.idBack}" target="_blank">صورة الهوية (خلف)</a>
                    </p>
                    <button onclick="updateStatus(${a.id},'accepted')">قبول وإرسال إشعار</button>
                    <button onclick="updateStatus(${a.id},'rejected')">رفض وإرسال إشعار</button>
                `;
                div.appendChild(card);
            });
        } catch (error) {
            console.error('Error loading applicants:', error);
            div.innerHTML = 'حدث خطأ أثناء تحميل البيانات.';
        }
    }

    async function updateStatus(id, status){
        if (!confirm(`هل أنت متأكد من تغيير حالة المتقدم رقم ${id} إلى ${status === 'accepted' ? 'قبول' : 'رفض'}؟ سيتم إرسال إشعار.`)) {
            return;
        }

        const buttonText = status === 'accepted' ? 'قبول' : 'رفض';
        alert(`جاري تحديث الحالة وإرسال الإشعارات. يرجى الانتظار...`);

        try {
            const res = await fetch('/api/applicants/'+id+'/status',{
                method:'POST', 
                headers:{'Content-Type':'application/json'}, 
                body:JSON.stringify({status})
            });
            
            if (res.ok) {
                alert(`تم تحديث الحالة بنجاح إلى ${buttonText} وإرسال الإشعارات.`);
                loadApplicants();
            } else {
                const errorText = await res.text();
                alert(`فشل تحديث الحالة. ${errorText}`);
            }

        } catch (error) {
            alert('حدث خطأ في الاتصال بالخادم.');
        }
    }

    loadApplicants();
</script>
</body>
</html>
