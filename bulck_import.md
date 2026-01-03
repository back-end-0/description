# ماذا يحدث لبيانات Excel بعد قراءتها؟

## الخطوة 1: تحويل Excel إلى CSV وتحميله للجدول المؤقت

```
Excel File → CSV → system_products_import_temp
```

البيانات تُحمَّل إلى جدول مؤقت `system_products_import_temp` باستخدام `LOAD DATA INFILE`.

---

## الخطوة 2: تحليل الأسماء وتحويلها إلى IDs

الـ Stored Procedure يأخذ الأسماء النصية ويحولها إلى IDs:

| من Excel (نص) | يُحوَّل إلى | من جدول |
|---------------|------------|---------|
| `category` | `resolved_category_id` | `categories` |
| `brand` | `resolved_brand_id` | `brands` |
| `unit` | `resolved_unit_id` | `product_measuring` |
| `color` | `resolved_color_code` | `colors` |

```sql
UPDATE system_products_import_temp t
SET
    t.resolved_category_id = (SELECT MIN(id) FROM categories WHERE LOWER(TRIM(name)) = LOWER(TRIM(t.category))),
    t.resolved_brand_id = (SELECT MIN(id) FROM brands WHERE LOWER(TRIM(name)) = LOWER(TRIM(t.brand))),
    ...
```

---

## الخطوة 3: التحقق من صحة البيانات (Validation)

يتم فحص كل صف وتسجيل الأخطاء في عمود `import_errors`:

### الحقول المطلوبة
- ❌ `name_en` فارغ → "Name (EN) is required"
- ❌ `name_ar` فارغ → "Name (AR) is required"
- ❌ `category` فارغ → "Category is required"
- ❌ `brand` فارغ → "Brand is required"
- ❌ `unit` فارغ → "Unit is required"
- ❌ `color` فارغ → "Color is required"
- ❌ `refundable` فارغ → "Refundable is required"
- ❌ `shipping_type` فارغ → "Shipping type is required"

### التحقق من المراجع (Foreign Keys)
- ❌ التصنيف غير موجود → "Invalid category: xyz"
- ❌ البراند غير موجود → "Invalid brand: xyz"
- ❌ الوحدة غير موجودة → "Invalid unit: xyz"
- ❌ اللون غير موجود → "Invalid color: xyz"

### التحقق من القيم
- ❌ `height < 0` → "Height must be >= 0"
- ❌ `refundable` ليس yes/no → "Refundable must be yes or no"
- ❌ `shipping_type` ليس free/flat_rate → "Shipping type must be free or flat_rate"

### التحقق من التكرار
- ❌ نفس الـ SKU مكرر في الملف → "Duplicate SKU in file"
- ❌ منتج بنفس الاسم والبراند موجود (بدون SKU) → "Product with same name and brand already exists"

---

## الخطوة 4: إدخال/تحديث المنتجات

### السيناريو أ: المنتج له SKU موجود في النظام → **تحديث**

```
إذا SKU موجود في system_products → UPDATE المنتج الموجود
```

```sql
UPDATE system_products sp
INNER JOIN system_products_import_temp t ON sp.sku = t.sku
SET
    sp.name = t.name_en,
    sp.category_id = t.resolved_category_id,
    sp.brand_id = t.resolved_brand_id,
    ...
```

### السيناريو ب: المنتج بدون SKU لكن الاسم+البراند موجود → **تحديث**

```
إذا (name + brand) موجود في system_products → UPDATE المنتج الموجود
```

### السيناريو ج: المنتج جديد تماماً → **إضافة**

```
إذا SKU غير موجود AND (name + brand) غير موجود → INSERT منتج جديد
```

```sql
INSERT INTO system_products (name, sku, category_id, brand_id, ...)
SELECT t.name_en, t.sku, t.resolved_category_id, t.resolved_brand_id, ...
FROM system_products_import_temp t
WHERE NOT EXISTS (SELECT 1 FROM system_products sp WHERE sp.sku = t.sku)
  AND NOT EXISTS (SELECT 1 FROM system_products sp WHERE sp.name = t.name_en AND sp.brand_id = t.resolved_brand_id)
```

---

## الخطوة 5: إضافة/تحديث الترجمات

لكل منتج يتم إنشاء سجلين في `product_translations`:

### الترجمة الإنجليزية (lang = 'en')
```sql
INSERT INTO product_translations (product_id, lang, name, description, meta_title, meta_description)
VALUES (123, 'en', 'Product Name EN', 'Description EN', 'Meta Title', 'Meta Desc')
ON DUPLICATE KEY UPDATE name = VALUES(name), ...
```

### الترجمة العربية (lang = 'sa')
```sql
INSERT INTO product_translations (product_id, lang, name, description, meta_title, meta_description)
VALUES (123, 'sa', 'اسم المنتج', 'الوصف', 'عنوان ميتا', 'وصف ميتا')
ON DUPLICATE KEY UPDATE name = VALUES(name), ...
```

---

## الخطوة 6: معالجة الصور (في PHP بعد الـ Stored Procedure)

إذا كان هناك `seller_id` و `foldername` في Excel:

```
uploads/generic_photos/{env}/{seller_id}/{foldername}/ → صور المنتج
```

1. جلب جميع الصور من S3 للمجلد المحدد
2. إنشاء سجلات في جدول `uploads`
3. ربط الصور بالمنتج في `system_products.photos` و `system_products.thumbnail_img`

---

## ملخص تدفق البيانات

```
┌─────────────────────────────────────────────────────────────────┐
│                          Excel                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              system_products_import_temp (جدول مؤقت)           │
│  •Convert name to IDs                                           │
│  • Data verification                                            │
│  • import_errors                                                │
└─────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
                   true                   false
                    │                       │
                    ▼                       ▼
    ┌───────────────────────────┐   ┌────────────────────────┐
    │    Check if it exists     │   │  Ignored and recorded  │
    │    • SKU                  │   │ User error             │
    │    • name + brand         │   └────────────────────────┘
    └───────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
    existing               not existing
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│    UPDATE     │       │    INSERT     │
│system_products│       │system_products│
└───────────────┘       └───────────────┘
        │                       │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │ product_translations  │
        │  • lang = 'en'        │
        │  • lang = 'sa'        │
        └───────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │  image processing (S3)│
        │  • uploads            │
        │  • photos             │
        │  • thumbnail_img      │
        └───────────────────────┘
```

---

## الجداول المتأثرة

| الجدول | العملية |
|--------|---------|
| `system_products_import_temp` | INSERT (مؤقت) |
| `system_products` | INSERT / UPDATE |
| `product_translations` | INSERT / UPDATE |
| `uploads` | INSERT (للصور) |
| `import_logs` | UPDATE (حالة الاستيراد) |
