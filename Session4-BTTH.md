# [BTTH] Bài tập thực hành tổng hợp: Session 04

**Kỹ thuật Prompting Nâng cao (Logic & Chuỗi)**
**Sinh viên:** Hà Gia Bách

---

## PHẦN 1. BÁO CÁO THIẾT KẾ GIẢI PHÁP

### 1.1 Bối cảnh & bài toán

Cửa hàng **TechShop** chạy khuyến mãi Black Friday, mỗi giờ nhận hàng trăm đơn do khách nhập tay (tin nhắn tự do). Nhân viên đang phải làm thủ công 4 việc:

1. Đọc đơn, tách sản phẩm, số lượng, mã giảm giá.
2. Tính tiền (áp mã giảm giá có điều kiện, phí ship).
3. Đối chiếu tồn kho.
4. Soạn tin nhắn xác nhận cho khách.

**Mục tiêu:** xây **chuỗi prompt khép kín** tự động hóa 4 việc trên, trong đó:
- **Chain-of-Thought (CoT)** dùng ở bước tính tiền (bước dễ sai logic nhất).
- **Prompt Chaining**: output JSON của bước trước là input của bước sau.
- **Xử lý đứt gãy**: một đơn lỗi không được làm sập cả lô.

### 1.2 Dữ liệu tham chiếu (đưa vào prompt)

**Danh mục & tồn kho**

| SKU | Sản phẩm | Giá (VNĐ) | Tồn |
|---|---|---|---|
| KB-100 | Bàn phím cơ KB-100 | 1.200.000 | 5 |
| HP-300 | Tai nghe HP-300 | 890.000 | 3 |
| MS-20 | Chuột không dây MS-20 | 250.000 | 1 |

**Quy tắc giá**

| Mã | Quy tắc |
|---|---|
| `BF20` | Giảm 20% giá trị hàng nếu hàng ≥ 2.000.000đ, giảm tối đa 500.000đ, hạn 30/11/2026 |
| `FREESHIP` | Miễn phí ship nếu hàng ≥ 500.000đ |
| Ship | Mặc định 30.000đ |

Công thức: `Tổng = Tạm tính − Giảm giá + Ship`

### 1.3 Kiến trúc chuỗi

```
 Đơn thô (text)
      │
      ▼
 ┌──────────────┐  INVALID / JSON hỏng sau 2 lần retry
 │ B1 Trích xuất │ ─────────────────────────────┐
 │ + Kiểm tra    │                              │
 └──────────────┘                              ▼
      │ OK                                ┌──────────┐
      ▼                                   │ DEAD     │
 ┌──────────────┐  lệch với code tính lại │ LETTER   │ → nhân viên xử lý
 │ B2 CoT tính   │ ──► [Code đối chiếu] ──►│ (cách ly)│
 │ tiền          │     dùng số của code    └──────────┘
 └──────────────┘     + gắn cờ                 ▲
      │                                        │
      ▼                                        │
 ┌──────────────┐                              │
 │ B3 Tồn kho   │ ── lỗi ──────────────────────┘
 └──────────────┘
      │ FULL / PARTIAL / OUT
      ▼
 ┌──────────────┐
 │ B4 Soạn tin  │ → Tin nhắn gửi khách
 └──────────────┘
```

### 1.4 Đặc tả Input / Output từng bước (chỉ truyền trường cần thiết để tiết kiệm token)

| Bước | Input | Output (JSON) |
|---|---|---|
| B1 | Đơn thô | `{st, kh, items:[{sku,qty}], code, err:[...]}` với `st` ∈ OK / INVALID |
| B2 | `items`, `code`, bảng giá + quy tắc | `{steps:[...], sub, disc, ship, total}` |
| B3 | `items`, bảng tồn kho | `{kq, lines:[{sku,need,have,short}]}` với `kq` ∈ FULL / PARTIAL / OUT |
| B4 | `kh`, `total`, `kq`, `lines` | `{msg}` |

### 1.5 Bảng dữ liệu dị biệt & cách xử lý

| # | Dị biệt | Xử lý | Lớp xử lý |
|---|---|---|---|
| 1 | Số lượng âm, bằng 0, hoặc không phải số | `NẾU qty ≤ 0 hoặc không phải số nguyên → err, st=INVALID` | Prompt B1 |
| 2 | SKU không có trong danh mục | `NẾU sku không thuộc danh mục → err UNKNOWN_SKU` | Prompt B1 |
| 3 | Mã giảm giá sai chính tả (BF2O) hoặc hết hạn | B1 ghi `code` đúng như khách nhập; B2 xác định mã không hợp lệ → disc = 0, ghi rõ lý do | Prompt B1, B2 |
| 4 | Không đủ điều kiện áp mã (hàng < 2.000.000đ) | B2 lập luận từng bước, disc = 0 | CoT B2 |
| 5 | Tên khách trống | `kh = "Quý khách"` (giá trị mặc định, không chặn đơn) | Prompt B1 |
| 6 | LLM tính sai số học | Code tính lại; lệch thì **dùng số của code** và gắn cờ `fix` | Code |
| 7 | LLM trả JSON hỏng, thừa chữ, hoặc thiếu key | Regex bắt khối `{...}`, kiểm tra key, retry tối đa 2 lần | Code |
| 8 | Hết retry hoặc lỗi bất ngờ (exception) | Đơn vào **dead letter**, lô vẫn chạy tiếp | Code |
| 9 | Tồn kho không đủ | B3 trả PARTIAL / OUT; B4 đưa phương án cho khách, không hứa giao đủ | Prompt B3, B4 |

### 1.6 Tối ưu token

- Mỗi bước chỉ nhận **đúng trường cần** (B3 không cần đọc quy tắc giá, B4 không cần đọc đơn thô).
- Khóa JSON viết tắt (`st`, `kh`, `err`, `sub`, `disc`).
- Bảng dữ liệu tham chiếu viết dạng dòng ngắn `SKU|giá|tồn`, không dùng văn xuôi.
- **Việc cộng/trừ/nhân** giao cho code kiểm lại. LLM chỉ cần lập luận quy tắc, nên không cần few-shot dài.
- Giới hạn độ dài đầu ra: B4 tối đa 60 từ.

---

## PHẦN 2. PROMPT THỰC THI TỪNG BƯỚC

> Ký hiệu `{{...}}` là biến được thay bằng dữ liệu thật khi chạy.

### B1. Trích xuất & kiểm tra

```text
Bạn là bộ kiểm tra đơn hàng TechShop. Chỉ trích xuất và kiểm tra, KHÔNG tính tiền.

SKU hợp lệ: KB-100, HP-300, MS-20
ĐƠN: <<<{{don_tho}}>>>

Quy tắc:
- NẾU qty ≤ 0, không phải số nguyên, hoặc trống → thêm err "BAD_QTY:<sku>", st="INVALID".
- NẾU sku không thuộc danh sách hợp lệ → thêm err "UNKNOWN_SKU:<giá trị>", st="INVALID".
- NẾU không có sản phẩm nào → err "EMPTY_ORDER", st="INVALID".
- NẾU tên khách trống → kh="Quý khách".
- Mã giảm giá: giữ NGUYÊN văn khách nhập, không tự sửa chính tả. Không có mã → code=null.
- Không có lỗi → st="OK", err=[].
- Không đoán dữ liệu thiếu.

Chỉ trả 1 khối JSON, không giải thích:
{"st":"","kh":"","items":[{"sku":"","qty":0}],"code":null,"err":[]}
```

### B2. Tính tiền bằng Chain-of-Thought

```text
Bạn là kế toán cửa hàng TechShop. Tính tiền đơn hàng, suy nghĩ TỪNG BƯỚC.

Giá: KB-100=1200000 | HP-300=890000 | MS-20=250000
Mã BF20: giảm 20% hàng nếu hàng ≥ 2000000, tối đa 500000, hạn 30/11/2026.
Mã FREESHIP: miễn ship nếu hàng ≥ 500000.
Ship mặc định: 30000. Hôm nay: {{ngay}}.
Tổng = tạm tính − giảm + ship.

ĐƠN: items={{items}} code={{code}}

Các bước ghi vào "steps" (mỗi bước 1 dòng, có phép tính):
1) Tạm tính từng dòng và tổng hàng.
2) Mã có thuộc BF20 / FREESHIP không? (so khớp CHÍNH XÁC từng ký tự). Còn hạn không?
3) Tổng hàng có đạt điều kiện tối thiểu của mã không?
4) Tính giảm giá (nhớ trần 500000 với BF20).
5) Tính ship.
6) Tính tổng cuối.

Điều kiện:
- NẾU code=null → giảm = 0.
- NẾU code không khớp mã nào, hết hạn, hoặc chưa đạt điều kiện → giảm = 0 (nếu là FREESHIP thì ship vẫn 30000) và nêu lý do ở bước 2 hoặc 3.
- Chỉ dùng quy tắc ở trên.

Chỉ trả JSON:
{"steps":[""],"sub":0,"disc":0,"ship":0,"total":0}
```

### B3. Kiểm tra tồn kho

```text
Đối chiếu tồn kho. Tồn: KB-100=5 | HP-300=3 | MS-20=1
Đơn: {{items}}

Với mỗi dòng: need=qty, have=min(qty, tồn), short=need−have.
- NẾU mọi short=0 → kq="FULL".
- NẾU có short>0 nhưng còn ít nhất 1 dòng have>0 → kq="PARTIAL".
- NẾU mọi have=0 → kq="OUT".
Chỉ trả JSON: {"kq":"","lines":[{"sku":"","need":0,"have":0,"short":0}]}
```

### B4. Soạn tin nhắn cho khách

```text
Viết tin nhắn xác nhận đơn cho khách, tối đa 60 từ, giọng thân thiện.
Dữ liệu: kh={{kh}} | tổng={{total}}đ | kq={{kq}} | lines={{lines}}

- NẾU kq="FULL" → xác nhận đơn, nêu tổng tiền, hẹn giao.
- NẾU kq="PARTIAL" → nêu rõ món thiếu và số lượng thiếu; đưa 2 lựa chọn: (a) giao phần có sẵn trước, phần thiếu chờ hàng; (b) hủy phần thiếu và hoàn tiền phần đó. Không hứa ngày cụ thể cho phần thiếu.
- NẾU kq="OUT" → xin lỗi, đề nghị chờ hàng hoặc hủy đơn.
- Chỉ dùng số liệu được cung cấp, không tự thêm khuyến mãi.
Chỉ trả JSON: {"msg":""}
```

### Mã điều phối (code sạch, bắt lỗi toàn diện)

```python
import json, re

PRICE = {"KB-100": 1_200_000, "HP-300": 890_000, "MS-20": 250_000}
MAX_RETRY = 2


class StepError(Exception):
    pass


def call_llm(prompt: str) -> str:
    """Thay bằng lời gọi API thật (OpenAI / Gemini / Claude)."""
    raise NotImplementedError


def run_step(name: str, prompt: str, required: set) -> dict:
    """Gọi LLM, ép JSON hợp lệ, kiểm tra key, retry khi lỗi."""
    err = ""
    for i in range(MAX_RETRY + 1):
        try:
            raw = call_llm(prompt + ("\nCHỈ TRẢ JSON HỢP LỆ." if i else ""))
            data = json.loads(re.search(r"\{.*\}", raw, re.S).group())
            missing = required - data.keys()
            if missing:
                raise KeyError(f"thiếu key {missing}")
            return data
        except (json.JSONDecodeError, AttributeError, KeyError, TypeError) as e:
            err = f"{name}: {e}"
    raise StepError(err)


def recompute(items: list, disc: int) -> int:
    """Đối chiếu số học độc lập với kết quả của LLM."""
    return sum(PRICE[i["sku"]] * i["qty"] for i in items) - disc


def process_order(raw_order: str, prompts: dict) -> dict:
    b1 = run_step("B1", prompts["b1"](raw_order), {"st", "items", "err"})
    if b1["st"] != "OK":
        raise StepError(f"INVALID {b1['err']}")

    b2 = run_step("B2", prompts["b2"](b1), {"sub", "disc", "ship", "total"})
    code_sub = recompute(b1["items"], 0)
    code_total = code_sub - b2["disc"] + b2["ship"]
    flag = None
    if code_total != b2["total"] or code_sub != b2["sub"]:
        b2["sub"], b2["total"], flag = code_sub, code_total, "fix_arithmetic"

    b3 = run_step("B3", prompts["b3"](b1), {"kq", "lines"})
    b4 = run_step("B4", prompts["b4"](b1, b2, b3), {"msg"})
    return {"total": b2["total"], "kq": b3["kq"], "msg": b4["msg"], "flag": flag}


def run_batch(orders: list, prompts: dict):
    ok, dead_letter = [], []
    for idx, order in enumerate(orders):
        try:
            ok.append(process_order(order, prompts))
        except Exception as e:  # một đơn lỗi không làm sập cả lô
            dead_letter.append({"idx": idx, "order": order, "reason": str(e)})
    return ok, dead_letter
```

---

## PHẦN 3. KẾT QUẢ CHẠY THỬ

### Test 1: Đơn hợp lệ, có áp mã giảm giá (luồng chính)

**Đơn thô:** *"Mình là Nguyễn Văn An, lấy 1 bàn phím KB-100 và 1 tai nghe HP-300, nhập mã BF20 giúp mình."*

**B1**
```json
{"st":"OK","kh":"Nguyễn Văn An","items":[{"sku":"KB-100","qty":1},{"sku":"HP-300","qty":1}],"code":"BF20","err":[]}
```

**B2**
```json
{
  "steps": [
    "1) KB-100: 1×1.200.000 = 1.200.000; HP-300: 1×890.000 = 890.000; tạm tính = 2.090.000",
    "2) Mã BF20 khớp chính xác; hạn 30/11/2026, hôm nay 18/11/2026 nên còn hạn",
    "3) 2.090.000 ≥ 2.000.000 nên đạt điều kiện",
    "4) 20% × 2.090.000 = 418.000; 418.000 ≤ trần 500.000 nên giảm 418.000",
    "5) Không dùng FREESHIP nên ship 30.000",
    "6) Tổng = 2.090.000 − 418.000 + 30.000 = 1.702.000"
  ],
  "sub": 2090000, "disc": 418000, "ship": 30000, "total": 1702000
}
```
→ **Đối chiếu bằng code:** `1.200.000 + 890.000 − 418.000 + 30.000 = 1.702.000` khớp, không gắn cờ.

**B3**
```json
{"kq":"FULL","lines":[{"sku":"KB-100","need":1,"have":1,"short":0},{"sku":"HP-300","need":1,"have":1,"short":0}]}
```

**B4**
```json
{"msg":"Chào anh Nguyễn Văn An, TechShop đã xác nhận đơn gồm bàn phím KB-100 và tai nghe HP-300. Tổng thanh toán 1.702.000đ (đã áp mã BF20, giảm 418.000đ). Đơn sẽ được giao sớm, cảm ơn anh đã ủng hộ!"}
```

---

### Test 2: Dữ liệu dị biệt (không được làm crash hệ thống)

Lô gồm 3 đơn chạy liên tiếp: đơn #0 hợp lệ (Test 1), đơn #1 bị lỗi, đơn #2 xem Test 3.

**Đơn #1 (thô):** *"Bán cho mình 2 cái XX-999 với MS-20 số lượng -2, mã BF2O."*

**B1**
```json
{
  "st": "INVALID",
  "kh": "Quý khách",
  "items": [{"sku":"XX-999","qty":2},{"sku":"MS-20","qty":-2}],
  "code": "BF2O",
  "err": ["UNKNOWN_SKU:XX-999", "BAD_QTY:MS-20"]
}
```

**Điều phối:** `st != "OK"` → `process_order` ném `StepError` → `run_batch` bắt lại → đơn #1 vào dead letter, **các đơn #0 và #2 vẫn chạy bình thường**.

**Dead letter:**
```json
[{"idx":1,"order":"Bán cho mình 2 cái XX-999 với MS-20 số lượng -2, mã BF2O.","reason":"INVALID ['UNKNOWN_SKU:XX-999', 'BAD_QTY:MS-20']"}]
```

**Kiểm thử đứt gãy kỹ thuật:** nếu B1 trả `"Dạ đây là kết quả: {st: OK"` (JSON hỏng), `json.loads` lỗi → retry với câu `CHỈ TRẢ JSON HỢP LỆ.`; sau 3 lần vẫn hỏng thì `StepError` và đơn vào dead letter. Hệ thống không bị dừng.

**Kiểm thử LLM tính sai:** giả sử B2 trả `total = 1.720.000` (sai). Code tính `1.702.000`, lệch nên ghi đè bằng số của code và đặt `flag = "fix_arithmetic"` để nhân viên rà soát sau.

---

### Test 3: Thiếu tồn kho + mã FREESHIP ở ngưỡng biên

**Đơn thô:** *"Chị Mai đặt 2 chuột MS-20, dùng mã FREESHIP."*

**B1**
```json
{"st":"OK","kh":"Chị Mai","items":[{"sku":"MS-20","qty":2}],"code":"FREESHIP","err":[]}
```

**B2**
```json
{
  "steps": [
    "1) MS-20: 2×250.000 = 500.000; tạm tính = 500.000",
    "2) Mã FREESHIP khớp chính xác",
    "3) 500.000 ≥ 500.000 nên đạt điều kiện (đúng ngưỡng biên)",
    "4) Không có mã giảm % nên giảm = 0",
    "5) FREESHIP hợp lệ nên ship = 0",
    "6) Tổng = 500.000 − 0 + 0 = 500.000"
  ],
  "sub": 500000, "disc": 0, "ship": 0, "total": 500000
}
```
→ Code đối chiếu: khớp.

**B3**
```json
{"kq":"PARTIAL","lines":[{"sku":"MS-20","need":2,"have":1,"short":1}]}
```

**B4**
```json
{"msg":"Chào chị Mai, tổng đơn của chị là 500.000đ (đã miễn phí ship). Hiện MS-20 chỉ còn 1 cái, thiếu 1 cái. Chị chọn giúp shop: (a) giao 1 cái trước, cái còn lại chờ hàng về; hoặc (b) hủy phần thiếu và hoàn lại 250.000đ. Shop chờ phản hồi của chị ạ!"}
```

---

## PHẦN 4. ĐỐI CHIẾU YÊU CẦU ĐỀ BÀI

| Yêu cầu | Đáp ứng |
|---|---|
| Chain-of-Thought | B2: 6 bước lập luận có phép tính, ghi vào `steps` |
| Prompt Chaining khép kín tuần tự | B1 → B2 → B3 → B4, output JSON bước trước làm input bước sau |
| Xử lý đứt gãy | Retry 2 lần, dead letter, đơn lỗi không dừng cả lô (mục 1.5, Test 2) |
| Code sạch | Hàm tách nhỏ, mỗi hàm một nhiệm vụ, có docstring, hằng số đặt tên rõ |
| Tối ưu token | Truyền đúng trường cần, khóa JSON ngắn, bảng dữ liệu dạng dòng, code lo phần số học (mục 1.6) |
| Bắt lỗi dữ liệu dị biệt | 9 dị biệt có điều kiện xử lý ở cả prompt và code (mục 1.5) |
| Báo cáo thiết kế + prompt + kết quả sạch lỗi | Phần 1, 2, 3 |

## Kết luận

Giải pháp kết hợp **LLM cho việc hiểu ngôn ngữ và lập luận quy tắc** với **code cho việc kiểm chứng và điều phối**. Nhờ vậy chuỗi vừa linh hoạt với dữ liệu tự do, vừa an toàn: lỗi bị chặn sớm ở B1, số học được đối chiếu độc lập ở B2, và mọi đơn lỗi đều được cách ly thay vì làm sập hệ thống.
