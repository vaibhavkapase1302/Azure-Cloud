![Image](https://www.msp360.com/resources/wp-content/uploads/2017/10/1-07.png)

![Image](https://media2.dev.to/dynamic/image/width%3D1000%2Cheight%3D420%2Cfit%3Dcover%2Cgravity%3Dauto%2Cformat%3Dauto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fvla6bvobjknfda2cvawm.png)

---

# 🌐 Azure Storage — EVERYTHING IN ONE PLACE

## 1️⃣ What is Azure Storage?

**Microsoft Azure Storage** is a **storage platform**, not one service.

You **always start** with a **Storage Account**.

---

## 2️⃣ Storage Account (TOP LEVEL)

A **Storage Account** is:

* The **container of containers**
* A **security + billing boundary**
* Required for all storage services

```
Azure Storage
   |
   v
Storage Account
```

---

## 3️⃣ What lives INSIDE a Storage Account

```
Storage Account
│
├── Settings
│   └── Static website        (CONFIG ONLY)
│        ├── Index document = index.html
│        └── Error document = 404.html
│
├── Data storage              (ACTUAL DATA)
│   ├── Containers            ← Blob Storage
│   │    ├── $web             ← Static website files
│   │    │    ├── index.html
│   │    │    └── images/
│   │    └── backups
│   │
│   ├── File shares           ← File Storage
│   │
│   ├── Queues                ← Queue Storage
│   │
│   └── Tables                ← Table Storage
│
└── Other settings
```

---

## 4️⃣ Static Website — how it REALLY works

### Configuration (NO UPLOAD HERE)

```
Storage Account → Static website
```

* Tells Azure:

  * Which file is the homepage
  * Which file is the error page

### File Upload (REAL FILES)

```
Storage Account → Containers → $web
```

* Upload:

  * `index.html`
  * `css/style.css`
  * `js/app.js`

---

## 5️⃣ Website Request Flow (VERY IMPORTANT)

```
Browser
   |
   v
https://<name>.web.core.windows.net
   |
   v
Storage Account
   |
   v
Blob Storage
   |
   v
$web container
   |
   v
index.html
```

---

