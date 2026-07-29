# المُدقِّق — Al-Mudaqiq

تطبيق ويب من ملف واحد لتدقيق النصوص العربية: إملائي، نحوي، تشكيل كامل، إعادة صياغة، استخراج الحوار، وتحويل الفصحى إلى ٢١ لهجة عربية — مدعوم بواجهة Anthropic API مباشرة من المتصفح.

A single-file web app for Arabic proofreading (spelling, grammar, full tashkeel, rewriting, dialogue extraction, and dialect conversion), powered by the Anthropic API directly from the browser.

## التشغيل / Run

افتح `index.html` في المتصفح (أو انشره على أي استضافة ثابتة)، ثم أدخل مفتاح Anthropic API من ⚙ الإعدادات — يُحفظ في متصفحك فقط.

Open `index.html` in a browser (or host it anywhere static), then paste your Anthropic API key in ⚙ Settings — it is stored only in your browser.

## تحسينات الواجهة / Frontend UX enhancements

- **مظهر داكن كامل** مع زر تبديل في الشريط العلوي، يحترم تفضيل النظام (`prefers-color-scheme`) ويُحفظ اختيارك — نظام ألوان رمزي (tokens) موحّد للوضعين بروح المخطوطة العربية.
- **حفظ تلقائي للمسودة**: نصك يبقى بعد إغلاق الصفحة ويُسترجع عند العودة.
- **اختصار لوحة المفاتيح**: Ctrl+Enter (أو ⌘+Enter) يبدأ التحقيق مباشرة من المحرر.
- **زر مسح المحرر** بتأكيد خفيف يمنع الحذف بالخطأ، وزر **إظهار/إخفاء مفتاح API**.
- **إتاحة أفضل**: تسميات ARIA للأزرار الأيقونية، حالات `aria-pressed` للخيارات الثنائية، مناطق إعلان حية للنتائج والأخطاء، وحلقات تركيز واضحة للوحة المفاتيح.
- **حركة مدروسة**: انتقالات دخول للبطاقات والنتائج مع احترام `prefers-reduced-motion`، وتمرير تلقائي إلى النتيجة عند اكتمالها.
- **لمسات بصرية**: شريط علوي لاصق بخلفية ضبابية، أرقام جدولية للعدادات والمؤقتات، مؤشر تبويب نشط في شريط التنقل السفلي، ودعم المساحات الآمنة لهواتف iOS.
