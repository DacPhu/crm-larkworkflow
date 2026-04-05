# VIG Academy — Lark CRM Course Payment Workflow

Automated course registration & payment funnel:
**Landing Page → n8n Webhook → Lark Base CRM → SePay QR Email → 3-day check → Reminder / Confirmation**

---

## File Structure

```
crm-lark-workflow/
├── workflow/
│   └── lark-crm-payment-workflow.json   ← Import into n8n
└── landing-page/
    └── index.html                        ← Open in browser / deploy
```

---

## Prerequisites

- n8n running at `http://localhost:5678` (Docker Compose in `~/Desktop/test/n8n/`)
- A Lark / Feishu account with Lark Base enabled
- SMTP email credentials (Gmail App Password recommended)

---

## Step 1 — Lark Base Setup

### 1a. Create Lark app
1. Go to [open.larksuite.com](https://open.larksuite.com) → Create App
2. Note the **App ID** and **App Secret**
3. Enable permissions: `bitable:app` (read/write)
4. Publish the app

### 1b. Get tenant_access_token
```bash
curl -X POST https://open.larksuite.com/open-apis/auth/v3/tenant_access_token/internal \
  -H "Content-Type: application/json" \
  -d '{"app_id":"YOUR_APP_ID","app_secret":"YOUR_APP_SECRET"}'
```
Copy the `tenant_access_token` from the response.

> ⚠️ Token expires every **2 hours**. Set a reminder or add a secondary n8n cron workflow to refresh it.

### 1c. Create Lark Base table
Create a Base with these columns:

| Column name       | Type          |
|-------------------|---------------|
| Họ và tên         | Text          |
| Email             | Email         |
| Số điện thoại     | Phone         |
| Khóa học          | Single select |
| Giá               | Number        |
| Mã thanh toán     | Text          |
| Trạng thái        | Single select |
| Ngày đăng ký      | DateTime      |
| Ngày kích hoạt    | DateTime      |
| Segment           | Text          |

**Trạng thái** options: `Chờ thanh toán` · `Đã thanh toán` · `Đã nhắc nhở` · `Đã kích hoạt`

Note the **app_token** (from the Base URL: `...applink.larksuite.com/client/...`) and **table_id** (from Base → table URL).

---

## Step 2 — Import Workflow into n8n

1. Open `http://localhost:5678`
2. Menu (top-left) → **Import from File** → select `workflow/lark-crm-payment-workflow.json`
3. The workflow will appear with 13 nodes

---

## Step 3 — Configure Credentials

In n8n → **Settings → Credentials**:

### A. HTTP Bearer Auth (for Lark)
- Name: `Lark Bearer Auth`
- Token: `<your tenant_access_token>`

### B. SMTP (for email)
- Host: `smtp.gmail.com`
- Port: `587`
- User: your Gmail address
- Password: [Gmail App Password](https://myaccount.google.com/apppasswords)

Assign these credentials to the relevant nodes in the workflow.

---

## Step 4 — Set Environment Variables

Add to `~/Desktop/test/n8n/docker-compose.yaml` under `environment:`:

```yaml
environment:
  - LARK_APP_TOKEN=bascnXXXXXXXX      # Lark Base app token
  - LARK_TABLE_ID=tblXXXXXXXX         # Lark Base table ID
  - SENDER_EMAIL=hello@vigacademy.vn
  - BANK_CODE=MB                       # Bank code for SePay QR
  - BANK_NAME=MB Bank
  - BANK_ACCOUNT=123456789             # Bank account number
  - HOTLINE=0901234567
  - COURSE_PORTAL_URL=https://learn.vigacademy.vn
```

Restart n8n after changes:
```bash
cd ~/Desktop/test/n8n && docker-compose restart
```

---

## Step 5 — Configure Landing Page

Edit the `CONFIG` object in `landing-page/index.html`:

```js
const CONFIG = {
  webhookUrl: 'http://localhost:5678/webhook/course-registration',
  // Update to production URL when deploying, e.g.:
  // webhookUrl: 'https://n8n.vigacademy.vn/webhook/course-registration',
  courses: [
    // Edit course names and prices as needed
  ]
};
```

---

## Step 6 — Activate Workflow & Test

1. In n8n, toggle the workflow to **Active**
2. Open the landing page: `open ~/Desktop/test/crm-lark-workflow/landing-page/index.html`
3. Fill and submit the registration form
4. Check n8n execution: `http://localhost:5678/workflow/executions`

### Manual curl test
```bash
curl -X POST http://localhost:5678/webhook/course-registration \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Nguyen Van A",
    "email": "test@example.com",
    "phone": "0901234567",
    "course": "Digital Marketing Toàn Diện",
    "price": 2990000
  }'
```

---

## Workflow Flow

```
[Landing Page Form]
        │ POST JSON
        ▼
[1] Webhook - Form Registration
        ├──▶ [2] Respond to Webhook (immediate 200 OK)
        └──▶ [3] Set - Normalize Data
                   (generates paymentCode = VIG{timestamp}{phone_last4})
                        │
                        ▼
              [4] Lark Base - Create Contact Record
                        │
                        ▼
              [5] Set - Store Lark Record ID
                        │
                        ▼
              [6] Email - Send QR Payment
                   (SePay QR: https://qr.sepay.vn/{BANK}/{ACCOUNT}?amount=X&content=CODE)
                        │
                        ▼
              [7] Wait - 3 Days
                        │
                        ▼
              [8] Lark Base - Get Payment Status
                        │
                        ▼
              [9] IF - Trạng thái == "Đã thanh toán"?
                  ┌────────────────┴────────────────┐
                TRUE                              FALSE
                  │                                  │
                  ▼                                  ▼
     [11] Email - Confirmation          [10] Email - Reminder
                  │                                  │
                  ▼                                  ▼
     [12] Lark Update → "Đã kích hoạt" [13] Lark Update → "Đã nhắc nhở"
          Segment → "Học viên active"
```

---

## SePay QR Format

```
https://qr.sepay.vn/{BANK_CODE}/{ACCOUNT_NUMBER}?amount={AMOUNT}&content={PAYMENT_CODE}
```

Example:
```
https://qr.sepay.vn/MB/123456789?amount=2990000&content=VIG202504051234
```

Where `paymentCode` = `VIG` + timestamp + last 4 digits of phone (e.g. `VIG202504051234`).

---

## Testing Payment Status Logic

To test the **paid branch** without waiting 3 days:
1. Temporarily set the Wait node to 1 minute
2. Submit a test registration
3. In Lark Base, manually update the record's **Trạng thái** to `Đã thanh toán`
4. Wait for execution to resume — confirmation email should arrive

To test the **unpaid/reminder branch**: leave the status as `Chờ thanh toán`.

---

## Production Deployment

### n8n
- Use [n8n Cloud](https://n8n.io/cloud/) or deploy on VPS with nginx + SSL
- Update `allowedOrigins` in Webhook node to your landing page domain

### Landing page
```bash
# Netlify (drag & drop the landing-page/ folder)
# or Vercel:
npx vercel ~/Desktop/test/crm-lark-workflow/landing-page --prod
# or copy to any web server
```

---

## Lark Token Refresh (Important)

`tenant_access_token` expires every 2 hours. Create a second n8n workflow:
- **Trigger**: Cron every 90 minutes
- **Node**: HTTP Request to `POST https://open.larksuite.com/open-apis/auth/v3/tenant_access_token/internal` with `app_id` and `app_secret`
- **Node**: Update the `Lark Bearer Auth` credential programmatically (or store token elsewhere)
