# Cash Empire — פרויקט Vercel עם מולטיפלייר

החבילה כוללת את `index.html` של המשחק ואת פונקציית Socket.IO של Vercel. לא מעלים רק את קובץ ה־HTML: צריך לפרוס את **כל תיקיית הפרויקט** כדי שגם `api/[...path].js` והחבילות שב־`package.json` יותקנו.

## לפני הפריסה: הגדרת Redis

Redis נדרש כדי ששחקנים שמתחברים למופעי Function שונים יוכלו לראות את אותו עולם ולשמור פרופילים באופן קבוע. בלי `REDIS_URL`, הפרויקט מציג אזהרה ופועל רק במצב מקומי/מוגבל — אל תשתף אותו כשרת ציבורי בלי להגדיר Redis.

1. פתח את פרויקט Vercel אחרי הייבוא.
2. הוסף מסד Redis דרך Marketplace / Integrations, או השתמש במסד Redis קיים.
3. העתק את כתובת החיבור מסוג Redis URL (בדרך כלל מתחילה ב־`rediss://`).
4. ב־Vercel: **Project → Settings → Environment Variables**.
5. הוסף משתנה בשם `REDIS_URL` עם כתובת החיבור. סמן Production וגם Preview אם תרצה לבדוק שם.
6. בצע Redeploy אחרי הוספת המשתנה.

אל תעלה את סיסמת Redis ל־GitHub ואל תשים אותה בתוך `index.html`.

## דרך מומלצת לפרוס ב־Vercel

1. הורד וחלץ את קובץ ה־ZIP.
2. העלה את **כל הקבצים והתיקיות שבתוך** `Cash_Empire_Vercel_Project` למאגר GitHub חדש. ודא ש־`index.html`, `package.json`, `vercel.json` והתיקייה `api` נמצאים בשורש המאגר, ולא בתוך תיקייה פנימית נוספת.
3. ב־Vercel לחץ **Add New → Project** ובחר את מאגר GitHub.
4. Framework Preset: **Other**. ודא ש־Fluid Compute פעיל ב־Project Settings → Functions (בפרויקטים חדשים הוא פעיל כברירת מחדל). השאר את Build Command ריק אם Vercel מאפשר זאת; אין צורך בבניית frontend.
5. לחץ Deploy.
6. הגדר `REDIS_URL` לפי הסעיף הקודם ובצע Redeploy.
7. פתח את כתובת האתר. פתח את המשחק בשני דפדפנים/חלונות פרטיים, צור שתי דמויות עם שמות וקודים שונים, ובדוק שהן מופיעות זו לזו בעיר.

## פריסה באמצעות CLI (חלופה)

נדרשים Node.js 20 ומעלה וחשבון Vercel. מתוך תיקיית הפרויקט:

```bash
npm install
npx vercel login
npx vercel
npx vercel env add REDIS_URL production
npx vercel --prod
```

כאשר Vercel מבקש ערך ל־`REDIS_URL`, הדבק את כתובת Redis. אם הוספת אותו דרך Dashboard, אין צורך בפקודת `env add`.

## מה יש בחבילה

- משחק Cash Empire הקיים (`index.html`).
- Socket.IO דרך Vercel Function בנתיב `/api/socket-io/socket.io`.
- שחקנים בזמן אמת, מיקומים, צ'אט, הודעות/שיחות ופרופילים.
- Redis Adapter לשידור בין מופעי Function שונים, ושמירת פרופילים ב־Redis.
- ניסיון התחברות מחדש מובנה של Socket.IO; ב־Vercel חיבור WebSocket נסגר כשה־Function מגיעה למגבלת הזמן, ולכן המשחק עשוי להתחבר מחדש.

## מגבלות חשובות

- תמיכת WebSockets של Vercel נמצאת בבטא ציבורית ומשך חיבור ב־Hobby מוגבל למקסימום 300 שניות; Socket.IO צריך להתחבר מחדש לאחר סגירה.
- צריך להגדיר `REDIS_URL` אמיתי. החבילה לא כוללת מסד נתונים או פרטי התחברות, ולא ניתן להפעיל מולטיפלייר גלובלי מלא רק באמצעות `index.html`.
- בורסה וירטואלית/מחירי מניות בתוך המשחק אינם עדיין שוק גלובלי משותף לכל השחקנים; חלק מהנתונים במשחק נשמרים בפרופיל של כל שחקן.
- הקוד נבדק לבדיקת תחביר, אך לא נפרס בפועל לחשבון Vercel ולא נבדק מול מסד Redis אמיתי. אחרי הפריסה יש לבצע את בדיקת שני הדפדפנים למעלה.
