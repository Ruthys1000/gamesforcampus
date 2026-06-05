# מרכז פיתוח משחקי AI לקמפוס הדיגיטלי

כלי ווב לבניית משחקים לימודיים שרצים בתוך מודל (Moodle) — עם פרומפט מוכן, וולידטור אוטומטי ותהליך שליחה מובנה.

## הרקע

הקמפוס הדיגיטלי של מרכז ההדרכה מוסיף פעילות חדשה: **משחק AI** — ספריית משחקים לימודיים שנבנו בבינה מלאכותית. כל מנהל קורס יוכל לבחור משחק מהספרייה ולהוסיפו לקורס שלו. כאשר המשחק רץ בתוך הקמפוס, המערכת מזריקה לתוכו שאלות מותאמות לתוכן הקורס, עוקבת אחרי ציונים ומאפשרת תחרות בין חניכים.

כדי שמשחק יעבוד בסביבה הזו, הוא חייב לעמוד ב-20 דרישות טכניות מדויקות — תקשורת עם הפלטפורמה דרך `postMessage`, קבלת שאלות, שליחת ציונים, ניהול חיים וטיימר. הכלי הזה בנוי כדי להנחות מפתחים לעמוד בכל הדרישות האלה מהרגע הראשון, ולחסוך איטרציות ארוכות של שגיאות.

---

## איך הכלי עובד

### שלב 1 — בנה עם AI
בחר סוג משחק מגלריית ההשראה (8 ז'אנרים: ירי אסטרואידים, ריצת מכשולים, הגנת בסיס, ספייס אינווידרס ועוד). תאר את הקונספט — האווירה, העוצמה, הרעיון — והכלי מייצר פרומפט מלא שמכווין את ה-AI (Claude / Cursor / Bolt) לבנות משחק שעומד בכל הדרישות. מדביקים, בונים, מקבלים קובץ HTML.

### שלב 2 — בדוק ותקן
גוררים את קובץ ה-HTML לוולידטור. הוא מריץ 20 בדיקות אוטומטיות ומציג ציון מיידי עם פירוט מה עבר ומה לא. אם יש כשלים — הוולידטור מייצר פרומפט תיקון ממוקד לחזור אל ה-AI. 2–4 סבבים זה נורמלי. אפשר גם לראות את המשחק בתצוגה מקדימה בגודל מובייל ישר בדף.

### שלב 3 — שלח
לאחר שהמשחק עובר וולידציה (כולל 5 בדיקות ידניות — בדיקה על מובייל, ניסיון גמר משחק, ניקיון הקונסול), מופיעה טופס שליחה. פרטי המפתח + שם המשחק → פותח אפליקציית מייל עם הכל מלא, מחכה רק לצירוף הקובץ.

---

## דרישות טכניות (20 בדיקות)

הוולידטור בודק אוטומטית:

| # | דרישה |
|---|-------|
| 1 | שליחת אירוע `GAME_START` בכל התחלה/הפעלה מחדש |
| 2 | שליחת `SCORE_UPDATE` בזמן אמת על כל תשובה נכונה |
| 3 | שליחת `SCORE_UPDATE` עם `isfinished: true` בסיום המשחק (פעם אחת בלבד) |
| 4 | קבלת שאלות, חיים, טיימר ומשוב מהפלטפורמה |
| 5 | סדר אתחול נכון: משתני config → `applyMoodleConfig` → משתני game state |
| 6 | 7 משתני config מוצהרים לפני `applyMoodleConfig` |
| 7 | מיפוי שאלות תומך בשני פורמטי Moodle |
| 8 | בדיקת טיימר עם `!== undefined` (כדי ש-0 יעבוד) |
| 9 | `showFeedback` נבדק עם `'showFeedback' in cfg` (לא truthy) |
| 10 | משך משוב ותוכן משוב נלקחים מה-config |
| 11 | לוגיקת טיימר: ספירה עולה כש-0, ספירה יורדת + game over כשיש ערך |
| 12 | שדה timer בדיווח = שניות אמיתיות (לא 0 hardcoded) |
| 13 | חיים מתחילים מערך ה-config |
| 14 | תשובה שגויה = אבדן חיים; 0 חיים = game over |
| 15 | אין חזרה על שאלות בתוך מחזור; איפוס pool לאחר מיצוי |
| 16 | הצגת/הסתרת משוב לפי config עם תזמון duration |
| 17 | Restart מאפס את כל ה-state בלי `location.reload()` |
| 18 | RTL עברית + viewport meta tag למובייל |
| 19 | ללא `alert`/`confirm`/`prompt` וללא `localStorage`/`sessionStorage` |
| 20 | תמיכה ב-touch events ו/או עיצוב רספונסיבי |

---

## אינטגרציית Moodle — פירוט טכני

המשחק רץ בתוך `<iframe>` בתוך הקמפוס. התקשורת היא דו-כיוונית:

### קבלת הגדרות (Platform → Game)
```javascript
window.moodleGameConfig = {
  questions: [
    { q: "טקסט שאלה", options: ["א","ב","ג","ד"], correct: 0, explanation: "הסבר" }
  ],
  lives: 3,
  timer: 60,
  showFeedback: true,
  feedbackDuration: 3,
  correctFeedback: "מצוין!",
  incorrectFeedback: "טעות!"
}
```

### שליחת אירועים (Game → Platform)
```javascript
// בכל התחלה
window.parent.postMessage({ type: 'GAME_START' }, '*');

// על תשובה נכונה
window.parent.postMessage({ type: 'SCORE_UPDATE', score: 10, timer: 45, isfinished: false }, '*');

// בסיום
window.parent.postMessage({ type: 'SCORE_UPDATE', score: 80, timer: 12, isfinished: true }, '*');
```

### מבנה קוד נדרש (סדר חובה)
```javascript
// 1. משתני config עם ברירות מחדל
let questionsSource = [ /* שאלות דמה בעברית */ ];
let MAX_LIVES = 3, GAME_TIMER = 0, SHOW_FEEDBACK = false;
let FEEDBACK_DURATION = 3, CORRECT_FEEDBACK = "נכון!", INCORRECT_FEEDBACK = "טעות!";

// 2. פונקציית apply
function applyMoodleConfig(cfg) {
  if (!cfg) return;
  if (cfg.questions?.length) questionsSource = cfg.questions;
  if (cfg.lives !== undefined) MAX_LIVES = cfg.lives;
  if (cfg.timer !== undefined) GAME_TIMER = cfg.timer;  // !== לא truthy!
  if ('showFeedback' in cfg) SHOW_FEEDBACK = cfg.showFeedback;  // in, לא truthy!
  // ...
}

// 3. קריאה מהפלטפורמה
try { applyMoodleConfig(window.parent.moodleGameConfig); } catch(e) {}

// 4. משתני game state (אחרי apply!)
let lives = MAX_LIVES;
```

---

## מבנה הפרויקט

```
index.html   — כל הכלי: CSS + HTML + JavaScript בקובץ אחד
README.md    — המסמך הזה
```

הכלי פועל כדף סטטי בלבד — אין שרת, אין בסיס נתונים. פרוס על GitHub Pages בכתובת:
`ruthys1000.github.io/gamesforcampus`

---

## פיתוח והרצה מקומית

```bash
# שכפול
git clone https://github.com/ruthys1000/gamesforcampus.git

# פתח ישירות בדפדפן
open index.html
```

אין תלויות חיצוניות להתקנה. ספריית Lucide Icons נטענת מ-CDN.

---

פותח עבור מרכז ההדרכה של מה"ד — קמפוס דיגיטלי.
