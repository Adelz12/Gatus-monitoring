# Gatus Monitoring

مراقبة صحة الخدمات الداخلية في الوقت الفعلي، باستخدام [Gatus](https://github.com/TwiN/gatus) عبر Docker.

---

## 📁 هيكلية المشروع

```
gatus/
├── docker-compose.yml   # وصفة تشغيل الحاوية
├── config/
│   └── config.yaml       # إعدادات المراقبة (endpoints, security, storage)
├── .env                   # متغيرات حساسة (غير مرفوعة على Git)
└── .gitignore
```


---

## 🚀 التشغيل

```bash
docker compose up -d
```

الداشبورد يصبح متاحاً على:
```
http://<server-ip>:8080
```

---

## 🔒 الأمان

تسجيل الدخول محمي بـ Basic Authentication، والقيم تُقرأ من `.env` بدل كتابتها مباشرة داخل `config.yaml`:

```yaml
security:
  basic:
    username: "${GATUS_USERNAME}"
    password-bcrypt-base64: "${GATUS_PASSWORD_BCRYPT_BASE64}"
```

### تغيير كلمة المرور
```bash
# 1. توليد bcrypt hash
docker run --rm -it caddy:latest caddy hash-password

# 2. تحويله إلى base64 بسطر واحد
echo -n '<bcrypt_hash>' | base64 -w0

# 3. تحديث القيمة داخل .env، ثم إعادة التشغيل
docker compose restart
```

---

## 💾 التخزين الدائم

سجل الفحوصات محفوظ عبر SQLite، بحيث لا يُفقد التاريخ عند إعادة تشغيل أو إعادة إنشاء الحاوية:

```yaml
storage:
  type: sqlite
  path: /data/data.db
```

---

## ➕ إضافة نقطة مراقبة جديدة

أضف endpoint جديد داخل `config/config.yaml`:

```yaml
  - name: my-service
    group: internal
    url: "http://<ip>:<port>"
    interval: 30s
    conditions:
      - "[STATUS] == 200"
```

ثم أعد التشغيل:
```bash
docker compose restart
```

---

## 🔔 الإشعارات (Alerting)

Gatus يدعم إرسال تنبيهات تلقائية عند فشل أي endpoint (Slack, Discord, Telegram, Email، وغيرها). يُضاف قسم `alerting` في `config.yaml`، ثم يُربط بكل endpoint:

```yaml
alerting:
  slack:
    webhook-url: "${SLACK_WEBHOOK_URL}"

endpoints:
  - name: my-service
    url: "http://<ip>:<port>"
    conditions:
      - "[STATUS] == 200"
    alerts:
      - type: slack
        failure-threshold: 2     # تنبيه بعد فشلين متتاليين
        success-threshold: 1     # يُعتبر متعافياً بعد نجاح واحد
        send-on-resolved: true   # إرسال تنبيه عند التعافي أيضاً
```

مزودو التنبيهات الآخرون (Email, Telegram, Discord...) يُعدّون بنفس الطريقة — راجع [توثيق Gatus الرسمي](https://github.com/TwiN/gatus#alerting).

---

## 📌 ملاحظات

- Gatus لا يملك واجهة إعدادات (Settings UI) — أي تعديل يتم فقط عبر `config.yaml` ثم `docker compose restart`.
- عند نقل الإعداد لسيرفر آخر: انقل فقط `docker-compose.yml` و `config/`، بدون `data/`، ومع كلمة مرور مختلفة لكل سيرفر.
