# 📸 تطبيق ويب: تمديد صورة مربعة إلى مستطيل بالذكاء الاصطناعي (Outpainting)

هذا مشروع كامل (واجهة + API) يعمل على **Next.js 14** مع **TailwindCSS**. ترفع صورة مربعة، تختار نسبة/حجم الإخراج (مستطيل أفقي أو عمودي)، وهو ينشئ **قماش أكبر + قناع Mask** تلقائيًا ويرسلها إلى **OpenAI Images (gpt-image-1)** لملء الفراغات بتفاصيل واقعية.

> ✅ مصمم للاستخدام السريع: انسخ الشفرة كما هي إلى مشروع Next.js جديد.
>
> 🧠 ممكن تبدل المزوّد لاحقًا (Stability/Replicate) من نفس الواجهة.

---

## المتطلبات

* Node 18+
* حساب OpenAI ومفتاح API في متغير البيئة `OPENAI_API_KEY`

> ملاحظة: نموذج **gpt-image-1** يدعم أحجام مستطيلة مثل `1536x1024` (أفقي) و`1024x1536` (عمودي).

---

## 1) تهيئة مشروع Next + Tailwind

```bash
npx create-next-app@latest ai-outpaint --ts --eslint --tailwind --app --src-dir --import-alias "@/*"
cd ai-outpaint
npm i form-data
```

Tailwind جاهز مع القالب.

أنشئ ملف `.env.local`:

```
OPENAI_API_KEY=sk-................
NEXT_PUBLIC_MAX_SIZE=1536x1024
```

> يمكنك تغيير `NEXT_PUBLIC_MAX_SIZE` لاحقًا.

---

## 2) مكوّن الواجهة: أداة رفع + ضبط الحجم + توليد القناع

ضع الملف التالي في: `src/app/page.tsx`

```tsx
'use client';
import React, { useRef, useState, useMemo } from 'react';

// واجهة بسيطة بدون مكتبات خارجية لسهولة النسخ
// إن رغبت، يمكنك إحلال shadcn/ui لاحقًا.

type SizeOption = '1024x1024' | '1536x1024' | '1024x1536';

type ExpandMode =
  | 'CENTER' // تمديد حول الصورة من كل الجهات
  | 'LEFT'   // نُلصق الأصل يمينًا ونمد لليسار
  | 'RIGHT'  // نُلصق الأصل يسارًا ونمد لليمين
  | 'TOP'    // نُلصق الأصل أسفلًا ونمد للأعلى
  | 'BOTTOM';// نُلصق الأصل أعلى ونمد للأسفل

const ALLOWED_SIZES: SizeOption[] = ['1024x1024', '1536x1024', '1024x1536'];

function parseSize(s: SizeOption) {
  const [w, h] = s.split('x').map(Number);
  return { w, h };
}

function drawFeatheredMask(
  ctx: CanvasRenderingContext2D,
  keepRect: { x: number; y: number; w: number; h: number },
  canvasW: number,
  canvasH: number,
  feather: number
) {
  // القناع (Mask): مناطق شفافة = مناطق يريد الذكاء توليدها
  // نملأ أولاً القناع بالأسود (مع ألفا=1) ليبقى الجزء الأصلي دون تعديل
  ctx.clearRect(0, 0, canvasW, canvasH);
  ctx.fillStyle = 'rgba(0,0,0,1)';
  ctx.fillRect(0, 0, canvasW, canvasH);

  // نجعل المناطق خارج المستطيل الأصلي شفافة (ألفا=0)
  // مع تدرّج حواف (feather) لدمج أنعم
  const grd = ctx.createLinearGradient(keepRect.x, 0, keepRect.x + keepRect.w, 0);
  // لا نستخدم التدرج هنا بشكل كامل، سنقص بمنطقة مسار مع حواف ناعمة لاحقًا

  // طريقة أسهل: نرسم مستطيلاً أوسع قليلاً ثم نستخدم shadow لعمل feather
  ctx.save();
  ctx.globalCompositeOperation = 'destination-out';
  ctx.shadowColor = 'black';
  ctx.shadowBlur = feather; // يحدد نعومة الحافة
  ctx.fillStyle = 'rgba(0,0,0,1)';
  ctx.fillRect(keepRect.x, keepRect.y, keepRect.w, keepRect.h);
  ctx.restore();
}

function placeOriginal(
  mode: ExpandMode,
  targetW: number,
  targetH: number,
  imgW: number,
  imgH: number
) {
  // إرجاع موضع الصورة الأصلية على القماش الكبير
  let x = 0, y = 0;
  switch (mode) {
    case 'CENTER':
      x = Math.floor((targetW - imgW) / 2);
      y = Math.floor((targetH - imgH) / 2);
      break;
    case 'LEFT':
      x = targetW - imgW;
      y = Math.floor((targetH - imgH) / 2);
      break;
    case 'RIGHT':
      x = 0;
      y = Math.floor((targetH - imgH) / 2);
      break;
    case 'TOP':
      x = Math.floor((targetW - imgW) / 2);
      y = targetH - imgH;
      break;
    case 'BOTTOM':
      x = Math.floor((targetW - imgW) / 2);
      y = 0;
      break;
  }
  return { x, y };
}

export default function Page() {
  const [file, setFile] = useState<File | null>(null);
  const [prompt, setPrompt] = useState('');
  const [size, setSize] = useState<SizeOption>('1536x1024');
  const [mode, setMode] = useState<ExpandMode>('CENTER');
  const [feather, setFeather] = useState(24);
  const [busy, setBusy] = useState(false);
  const [resultUrl, setResultUrl] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);

  const imgRef = useRef<HTMLImageElement | null>(null);

  const onFile = (f: File) => {
    setFile(f);
    const url = URL.createObjectURL(f);
    const img = new Image();
    img.onload = () => {
      imgRef.current = img;
    };
    img.src = url;
  };

  const createCanvasAndMask = async (): Promise<{ base: Blob; mask: Blob } | null> => {
    if (!imgRef.current) return null;
    const img = imgRef.current;

    const { w: targetW, h: targetH } = parseSize(size);

    // نفترض الصورة مربعة. لو كانت غير مربعة سنقلمها داخليًا لتناسب أقصر ضلع
    const side = Math.min(img.width, img.height);

    // نقتطع مربّعًا من المنتصف
    const sx = Math.floor((img.width - side) / 2);
    const sy = Math.floor((img.height - side) / 2);

    // نرسم القماش الكبير
    const baseCanvas = document.createElement('canvas');
    baseCanvas.width = targetW;
    baseCanvas.height = targetH;
    const bctx = baseCanvas.getContext('2d')!;

    // خلفية شفافة
    bctx.clearRect(0, 0, targetW, targetH);

    // نحسب قياس الصورة الأصلية داخل الهدف مع الحفاظ على المربّع (بدون تمديد)
    const imgSize = Math.min(targetW, targetH);

    const pos = placeOriginal(mode, targetW, targetH, imgSize, imgSize);

    // نرسم الجزء المقتطع من الأصل داخل القماش الكبير
    bctx.drawImage(img, sx, sy, side, side, pos.x, pos.y, imgSize, imgSize);

    // نصنع قناعًا بنفس الأبعاد
    const maskCanvas = document.createElement('canvas');
    maskCanvas.width = targetW;
    maskCanvas.height = targetH;
    const mctx = maskCanvas.getContext('2d')!;

    drawFeatheredMask(mctx, { x: pos.x, y: pos.y, w: imgSize, h: imgSize }, targetW, targetH, feather);

    const base = await new Promise<Blob>((res) => baseCanvas.toBlob((b) => res(b!), 'image/png'));
    const mask = await new Promise<Blob>((res) => maskCanvas.toBlob((b) => res(b!), 'image/png'));

    return { base, mask };
  };

  const handleSubmit = async () => {
    try {
      setError(null);
      setResultUrl(null);
      if (!file) {
        setError('حمّل صورة أولاً');
        return;
      }
      if (!prompt.trim()) {
        setError('اكتب وصفًا بسيطًا للمشهد لنتائج أدق (مثال: غرفة معيشة دافئة بخلفية نباتات).');
        return;
      }

      const cm = await createCanvasAndMask();
      if (!cm) {
        setError('تعذّر تجهيز القناع.');
        return;
      }
      const form = new FormData();
      form.append('image', cm.base, 'base.png');
      form.append('mask', cm.mask, 'mask.png');
      form.append('prompt', prompt);
      form.append('size', size);

      setBusy(true);
      const res = await fetch('/api/outpaint', { method: 'POST', body: form });
      setBusy(false);

      if (!res.ok) {
        const t = await res.text();
        setError(`فشل الطلب: ${t}`);
        return;
      }
      const blob = await res.blob();
      const url = URL.createObjectURL(blob);
      setResultUrl(url);
    } catch (e: any) {
      setBusy(false);
      setError(e?.message || 'خطأ غير متوقع');
    }
  };

  return (
    <div className="min-h-screen bg-neutral-50 text-neutral-900">
      <div className="max-w-5xl mx-auto p-6 space-y-6">
        <header className="flex items-center justify-between">
          <h1 className="text-2xl md:text-3xl font-semibold">🖼️ تمديد الصور بالذكاء الاصطناعي</h1>
        </header>

        <section className="grid md:grid-cols-2 gap-6">
          <div className="bg-white rounded-2xl shadow p-4 space-y-4">
            <div>
              <label className="block text-sm font-medium mb-2">الصورة الأصلية</label>
              <div className="border-2 border-dashed rounded-xl p-6 text-center">
                <input
                  type="file"
                  accept="image/*"
                  onChange={(e) => e.target.files && onFile(e.target.files[0])}
                />
                <p className="text-xs text-neutral-500 mt-2">الأفضل رفع صورة مربعة؛ إن لم تكن، سنقصّ مربعًا من المنتصف تلقائيًا.</p>
              </div>
            </div>

            <div className="grid grid-cols-2 gap-3">
              <div>
                <label className="block text-sm font-medium mb-2">حجم الناتج</label>
                <select
                  value={size}
                  onChange={(e) => setSize(e.target.value as SizeOption)}
                  className="w-full rounded-lg border p-2"
                >
                  {ALLOWED_SIZES.map((s) => (
                    <option key={s} value={s}>{s}</option>
                  ))}
                </select>
                <p className="text-xs text-neutral-500 mt-1">يدعم gpt-image-1: 1024x1024، 1536x1024، 1024x1536</p>
              </div>

              <div>
                <label className="block text-sm font-medium mb-2">اتجاه التمديد</label>
                <select
                  value={mode}
                  onChange={(e) => setMode(e.target.value as ExpandMode)}
                  className="w-full rounded-lg border p-2"
                >
                  <option value="CENTER">حول الصورة (مستحسن)</option>
                  <option value="RIGHT">تمديد يمين</option>
                  <option value="LEFT">تمديد يسار</option>
                  <option value="TOP">تمديد للأعلى</option>
                  <option value="BOTTOM">تمديد للأسفل</option>
                </select>
              </div>
            </div>

            <div>
              <label className="block text-sm font-medium mb-2">نعومة الحواف (Feather)</label>
              <input
                type="range"
                min={0}
                max={64}
                value={feather}
                onChange={(e) => setFeather(Number(e.target.value))}
                className="w-full"
              />
              <div className="text-xs text-neutral-500">قيمة أعلى = دمج أنعم بين الأصل والمناطق المولدة</div>
            </div>

            <div>
              <label className="block text-sm font-medium mb-2">وصف المشهد (Prompt)</label>
              <textarea
                className="w-full border rounded-lg p-2 min-h-[84px]"
                placeholder="مثال: غرفة نوم مودرن بإضاءة دافئة ونباتات داخلية، خامات خشب وأقمشة طبيعية"
                value={prompt}
                onChange={(e) => setPrompt(e.target.value)}
              />
              <p className="text-xs text-neutral-500 mt-1">اكتب وصفًا مختصرًا يوجّه التوليد في المساحات الجديدة فقط.</p>
            </div>

            <button
              onClick={handleSubmit}
              disabled={busy || !file}
              className={`w-full rounded-xl py-3 font-medium ${busy || !file ? 'bg-neutral-300' : 'bg-black text-white hover:opacity-90'}`}
            >
              {busy ? 'جاري التوليد…' : 'مدّد الصورة'}
            </button>

            {error && (
              <div className="text-sm text-red-600">{error}</div>
            )}
          </div>

          <div className="bg-white rounded-2xl shadow p-4">
            <h3 className="text-sm font-medium mb-2">المعاينة</h3>
            <div className="aspect-video bg-neutral-100 rounded-xl flex items-center justify-center overflow-hidden">
              {resultUrl ? (
                // النتيجة
                // eslint-disable-next-line @next/next/no-img-element
                <img src={resultUrl} alt="result" className="w-full h-auto" />
              ) : file ? (
                // المعاينة الأولية للصورة الأصلية
                // eslint-disable-next-line @next/next/no-img-element
                <img src={URL.createObjectURL(file)} alt="preview" className="h-full" />
              ) : (
                <div className="text-neutral-400 text-sm">لا توجد نتيجة بعد</div>
              )}
            </div>
          </div>
        </section>
      </div>
    </div>
  );
}
```

> الواجهة تولد **صورتين** على المتصفح: `base.png` (القماش الأكبر مع الصورة الأصلية) و`mask.png` (مناطق شفافة حيث نريد التوليد). ثم ترسلها إلى `/api/outpaint`.

---

## 3) مسار API: استدعاء OpenAI Image Edits

ضع الملف التالي في: `src/app/api/outpaint/route.ts`

```ts
import { NextRequest } from 'next/server';

export const runtime = 'nodejs';
export const dynamic = 'force-dynamic';

export async function POST(req: NextRequest) {
  try {
    const apiKey = process.env.OPENAI_API_KEY;
    if (!apiKey) {
      return new Response('Missing OPENAI_API_KEY', { status: 500 });
    }

    const form = await req.formData();
    const image = form.get('image') as File | null;
    const mask = form.get('mask') as File | null;
    const prompt = (form.get('prompt') as string) || '';
    const size = (form.get('size') as string) || '1536x1024';

    if (!image || !mask) {
      return new Response('image/mask missing', { status: 400 });
    }

    // نعيد تشكيل بيانات الفورم كما هي لإرسالها لـ OpenAI
    const upstream = new FormData();
    upstream.append('model', 'gpt-image-1');
    upstream.append('prompt', prompt);
    upstream.append('size', size);
    upstream.append('image', image, 'base.png');
    upstream.append('mask', mask, 'mask.png');
    upstream.append('response_format', 'b64_json');

    const res = await fetch('https://api.openai.com/v1/images/edits', {
      method: 'POST',
      headers: { Authorization: `Bearer ${apiKey}` },
      body: upstream,
    });

    if (!res.ok) {
      const text = await res.text();
      return new Response(text, { status: res.status });
    }

    const json = await res.json();
    const b64 = json?.data?.[0]?.b64_json as string | undefined;
    if (!b64) {
      return new Response('No image returned', { status: 502 });
    }

    const buf = Buffer.from(b64, 'base64');
    return new Response(buf, {
      status: 200,
      headers: {
        'Content-Type': 'image/png',
        'Cache-Control': 'no-store',
      },
    });
  } catch (e: any) {
    return new Response(e?.message || 'Server error', { status: 500 });
  }
}
```

> **مهم:** واجهة OpenAI لميزة *Edits* تستخدم صورة + قناع بنفس المقاس. القناع يجب أن يجعل **المناطق المراد توليدها شفافة**، وتبقى المناطق غير الشفافة محفوظة كما هي.

---

## 4) تحسينات اختيارية

* **منع الملفات الكبيرة**: PNG كبير قد يتخطى 4MB. يمكنك قبل الإرسال تصغير `targetW/targetH` أو التحويل إلى JPEG (لكن أقنعة OpenAI تتطلب PNG).
* **Feather متطور**: يمكنك رسم حواف تدريجية باتجاه الخارج فقط (ليس داخل المنطقة الأصلية) لمزج أنعم.
* **سجل النُسخ**: خزّن نسخ النتائج في Supabase أو S3.
* **مزوّد بديل**: أضف مسار `/api/outpaint-stability` إن رغبت باستخدام Stability أو Replicate. (انظر المثال أدناه)

### مثال مسار بديل لـ Replicate (SDXL Outpainting)

```ts
// src/app/api/outpaint-replicate/route.ts
import { NextRequest } from 'next/server';

export async function POST(req: NextRequest) {
  try {
    const key = process.env.REPLICATE_API_TOKEN;
    if (!key) return new Response('Missing REPLICATE_API_TOKEN', { status: 500 });

    const form = await req.formData();
    const image = form.get('image') as File | null;
    const mask = form.get('mask') as File | null;
    const prompt = (form.get('prompt') as string) || '';

    if (!image || !mask) return new Response('image/mask missing', { status: 400 });

    const imgB64 = Buffer.from(await image.arrayBuffer()).toString('base64');
    const maskB64 = Buffer.from(await mask.arrayBuffer()).toString('base64');

    const resp = await fetch('https://api.replicate.com/v1/predictions', {
      method: 'POST',
      headers: { 'Authorization': `Token ${key}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({
        version: 'a542ccf352995f3c41f0bcfaef641daa3058bf2b00e08e04feb0295334ab9804',
        input: {
          prompt,
          image: `data:image/png;base64,${imgB64}`,
          mask: `data:image/png;base64,${maskB64}`,
          // إعدادات إضافية حسب النموذج
        }
      })
    });

    if (!resp.ok) return new Response(await resp.text(), { status: resp.status });
    const data = await resp.json();

    // استعلم حتى الاكتمال
    let poll = data;
    while (poll.status === 'starting' || poll.status === 'processing' || poll.status === 'queued') {
      await new Promise(r => setTimeout(r, 1500));
      const r2 = await fetch(`https://api.replicate.com/v1/predictions/${poll.id}`, { headers: { Authorization: `Token ${key}` } });
      poll = await r2.json();
    }

    if (poll.status !== 'succeeded') return new Response(JSON.stringify(poll), { status: 502 });

    const [url] = poll.output as string[];
    const img = await fetch(url);
    const buf = Buffer.from(await img.arrayBuffer());
    return new Response(buf, { headers: { 'Content-Type': 'image/png' } });
  } catch (e: any) {
    return new Response(e?.message || 'Server error', { status: 500 });
  }
}
```

> يتطلب مفتاح `REPLICATE_API_TOKEN`.

---

## 5) نصائح للوصف (Prompt) للحصول على نتائج نظيفة

* صف الخلفية فقط (المساحات الجديدة) بدلًا من إعادة وصف العنصر الرئيسي.
* حدد الإضاءة والجو (مثال: *ضوء دافئ مسائي، ظلال ناعمة*).
* اذكر الخامات أو النمط (مثال: *خشب طبيعي، أسلوب فوتوغرافي واقعي*).

## 6) تشغيل محليًا

```bash
npm run dev
# افتح http://localhost:3000
```

---

## 7) نشر سريع على Vercel

* اربط مستودع GitHub.
* أضف `OPENAI_API_KEY` في إعدادات Environment.
* انشر.

---

## لقطات شاشة ذهنية لكيف يشتغل التطبيق

1. تختار صورة مربعة.
2. تختار حجم الإخراج: أفقي 1536x1024 أو عمودي 1024x1536.
3. تضبط اتجاه التمديد (يمين/يسار/أعلى/أسفل/حول).
4. تكتب وصفًا بسيطًا للمشهد.
5. التطبيق يصنع قناعًا شفافًا للمناطق الجديدة ويرسل كل شيء للـ API.
6. ترجع لك صورة مستطيلة مكتملة.

---

## ملاحظات دقيقة مهمّة

* القناع **يجب** أن يكون بنفس أبعاد الصورة المرسلة للـ API.
* المناطق الشفافة في القناع = مناطق يُعاد توليدها.
* إن كانت النتيجة "حادة" عند الحافة، ارفع قيمة `Feather`.
* التوليد قد يعيد لمس جزء بسيط من الحافة — هذا طبيعي لتحسين الدمج.

موفّق! 🎉 إذا تحب، أقدر أعدّل الواجهة لك لتطابق ستايل معيّن أو أضيف قوالب جاهزة لنِسَب شائعة للسوشال ميديا (قصص، بانرات… إلخ).
