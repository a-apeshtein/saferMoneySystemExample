# SaferMoneySystem – מערכת ניהול התקני תשלום

מערכת `SaferMoneySystem.MoneySystem` נועדה לנהל התקני קבלת תשלום ופדיון מזומן (כגון: CoinChanger, BillRecycler וכו').  
המנהל תומך בפרוטוקולים שונים (`EBDS`, `MDB`, `Paylink`) ומאפשר אינטגרציה פשוטה עם ממשק משתמש לניהול טרנזקציות.

הגישה המומלצת:
- אתחול המערכת באמצעות `MoneySystemManager`
- שימוש ב-`TransactionManager` כתווך עיקרי לביצוע טרנזקציות, האזנה לאירועים והפעלה של התקנים

---

## ⚙️ מחלקה: MoneySystemConfiguration

מחלקת קונפיגורציה להגדרת אילו פרוטוקולים יופעלו באתחול.

**שדות:**
- `UsePaylink`: האם להפעיל את פרוטוקול Paylink
- `UseEbds`: האם להפעיל את פרוטוקול EBDS
- `UseMdbAdaptor`: האם להפעיל את מתאם MDB
- `ReloadDevicesAutomaticly`: האם לבצע ריענון התקנים אוטומטי
- `ReloadDevicesAutomaticlyInterval`: מרווח זמן (בשניות) לריענון אוטומטי

---

## 🧠 מחלקה: MoneySystemManager

מנהלת את טעינת הפרוטוקולים, אתחול התקנים וניהול אירועים.

**אירועים:**
- `CashDeposited` – בעת הפקדת מזומן
- `CashDispensed` – בעת פדיון מזומן
- `Initialized` – כשהאתחול הושלם

**מתודות עיקריות:**
- `InitSystemAsync()` – מאתחל את הפרוטוקולים שנבחרו
- `GetRecylers()` – מחזיר התקנים שמחזירים מזומן
- `GetAcceptors()` – מחזיר התקנים שמקבלים מזומן
- `GetVersion()` – מחזיר גרסת הספרייה

---

## 🔁 מחלקה: TransactionManager

תווך בין ממשק המשתמש להתקני הכסף. אחראית על ניהול אחיד של טרנזקציות.

**שימוש טיפוסי:**
```csharp
var moneySystem = new MoneySystemManager(logger, new MoneySystemConfiguration { UsePaylink = true });
await moneySystem.InitSystemAsync();
var transactionManager = new TransactionManager(new MoneyDeviceFacade(moneySystem, logger), logger);

```


שימושים נפוצים:

```csharp
await transactionManager.EnsureTransactionAsync(amount, CancellationToken.None);
await transactionManager.PayoutAsync(amount);
device.PayoutByValue(amount);
device.Cancel();

```
אירועים:

OnTransactionStarted – התחלת טרנזקציה

OnTransactionCompleted – סיום טרנזקציה

OnTransactionFailed – שגיאה בטרנזקציה


## 🧠 דוגמת קוד פשוטה אך מקיפה

```csharp
class MyLoggerImplementation : SaferMoneySystem.Logging.ILogger
{
    public void Log(SaferMoneySystem.Logging.LogLevel level, string tag, string msg, params object[] args)
    {
        string json = "";
        var text = ($"{DateTime.Now:HH:mm:ss.fff} MoneySystem-{tag} [{level}], {msg}, {json}");
        Console.WriteLine(text);
    }
}

async void RunTheSystem()
{
    var logger = new MyLoggerImplementation();
    var moneySystemInstance = new MoneySystemManager(logger, new MoneySystemConfiguration { UsePaylink = true });

    var transactionManagerInstance = new TransactionManager(
        new MoneyDeviceFacade(moneySystemInstance, logger),
        logger
    );

    moneySystemInstance.Initialized += (_, e) =>
    {
        foreach (var device in moneySystemInstance.Devices)
        {
            Console.WriteLine(device.SerialNumber);

            device.TubeStatusChanged += () =>
            {
                Console.WriteLine($"Tube status changed for: {device.DeviceType}, S/N:{device.SerialNumber}");
            };
        }
    };

    await moneySystemInstance.InitSystemAsync();

    transactionManagerInstance.TransactionChanged += async (_, ev) =>
    {
        if (ev != null)
        {
            Console.WriteLine($"TransactionInfo: Required: {(ev.AmountRequired / 100):0.00} NIS | " +
                              $"Paid: {(ev.AmountPaid / 100):0.00} NIS | " +
                              $"Status: {ev.Status}");
        }

        if (ev?.Status == TransactionStatus.PaymentComplete)
        {
            await transactionManagerInstance.ApproveAsync();
        }

        if (ev?.Status == TransactionStatus.DispensingCompleteSuccesfully)
        {
            // Change dispensing completed
        }
    };



    transactionManagerInstance.StartTransaction(10 * 100);
}

```

💡 למידע נוסף, שילוב עם UI, ולתמיכה טכנית – ניתן לפנות לצוות הפיתוח של Safer.


