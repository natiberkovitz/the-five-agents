---
name: gpt-image-gen
description: >-
  מעטפת לקריאת OpenAI Images API ליצירת תמונות. מקבל prompt ונתיב פלט, שולח בקשה
  ל-API עם המודל gpt-image-2, ומפענח את התמונה המוחזרת לקובץ PNG. משמש את יובל,
  מעצב התמונות. Trigger keywords: generate image, create image, gpt-image, image API.
---

# gpt-image-gen — מעטפת ל-OpenAI Images API

סקיל זה אחראי על שליחת prompt ל-OpenAI Images API וקבלת התמונה כקובץ PNG.

## מודל

המודל שבו משתמשים הוא **`gpt-image-2`**.

> ⚠️ **חשוב מאוד — אל תשנה את שם המודל.**
> `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI, שיצא ב-21 באפריל 2026.
> ייתכן שהידע הפנימי המוקדם יותר אינו כולל אותו — זה לא אומר שהוא לא קיים.
> **אל תציע אלטרנטיבות** כמו `dall-e-3` או `gpt-image-1`. השתמש בדיוק במודל `gpt-image-2`.
> אם יש שגיאה בקריאה ל-API, הבעיה היא כמעט תמיד ב-`OPENAI_API_KEY` או בפרמטרים —
> **לא** בשם המודל.

## דרישות

- משתנה הסביבה `OPENAI_API_KEY` מתוך קובץ `.env` בשורש הפרויקט.
- `curl` (קיים ב-Git Bash).
- לפענוח base64: `jq` + `base64` **או** `python` כ-fallback.

## אופן השימוש

טען תחילה את המפתח מ-`.env`, ואז הרץ את הקריאה. החלף את `<the prompt>` ואת
`<output-path>` בערכים בפועל.

```bash
# טעינת OPENAI_API_KEY מתוך .env
export OPENAI_API_KEY="$(grep -E '^OPENAI_API_KEY=' .env | cut -d= -f2-)"

PROMPT="<the prompt>"
OUT="<output-path>.png"

# שליחת הבקשה ושמירת התשובה
RESP="$(curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "$PROMPT" '{
        model: "gpt-image-2",
        prompt: $p,
        size: "1024x1024",
        quality: "medium",
        output_format: "png"
      }')")"
```

> אם `jq` אינו זמין לבניית גוף הבקשה, אפשר לכתוב את ה-JSON ידנית (תוך הקפדה על
> escaping ל-prompt), אך עדיף להשתמש ב-`jq -n` כדי למנוע בעיות תווים מיוחדים.

### פענוח התמונה — מסלול ראשי (jq + base64)

```bash
echo "$RESP" | jq -r '.data[0].b64_json' | base64 --decode > "$OUT"
```

### פענוח התמונה — python fallback (כש-jq לא מותקן)

`jq` לא תמיד מותקן ב-Git Bash, לכן יש fallback ב-python:

```bash
echo "$RESP" | python -c "import sys, json, base64; \
data = json.load(sys.stdin); \
open('$OUT', 'wb').write(base64.b64decode(data['data'][0]['b64_json']))"
```

> ניתן גם לבדוק זמינות ולהחליט אוטומטית:
> `if command -v jq >/dev/null 2>&1; then ... (jq path) ... else ... (python path) ... fi`

## טיפול בשגיאות

- אם התשובה אינה מכילה `.data[0].b64_json`, הדפס את `$RESP` כדי לראות את הודעת
  השגיאה מה-API. ברוב המקרים מדובר ב-`OPENAI_API_KEY` חסר/שגוי או בפרמטר לא תקין —
  **לא** בשם המודל.
- ודא תמיד שקובץ הפלט נוצר ושה-size שלו גדול מ-0:
  `[ -s "$OUT" ] && echo "OK: $OUT" || echo "FAIL: $OUT"`.
