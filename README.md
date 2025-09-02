# AI Outpaint (Next.js + Tailwind)

تمديد الصور (Outpainting) من مربع إلى مستطيل باستخدام واجهة OpenAI Images Edits.

## الاستخدام محليًا
1) ثبّت الاعتمادات: `npm install`
2) أضف `.env.local` في الجذر:
   ```
   OPENAI_API_KEY=sk-... ضع مفتاحك هنا
   ```
3) شغّل: `npm run dev` ثم افتح http://localhost:3000

## النشر على Vercel (مُوصى به)
- ارفع هذا المجلد إلى GitHub.
- من Vercel: New Project → Import من GitHub.
- أضف متغير البيئة OPENAI_API_KEY.
- Deploy.
