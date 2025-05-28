---
title: Вступ до Solana Mobile
завдання:

- Пояснити переваги створення dApp з орієнтацією на мобільні пристрої
- Пояснити загальний принцип роботи Mobile Wallet Adapter (MWA)
- Пояснити основні відмінності між React та React Native
- Створити простий Android Solana dApp за допомогою React Native
---

# Стислий виклад

* Solana Mobile Wallet Adapter (MWA) створює WebSocket-з’єднання між мобільними додатками та мобільними гаманцями, що дозволяє нативним мобільним додаткам надсилати транзакції для підписання
* Найпростіший спосіб почати створювати мобільні Solana-додатки — використати пакети Solana Mobile для React Native: `@solana-mobile/mobile-wallet-adapter-protocol` та `@solana-mobile/mobile-wallet-adapter-protocol-web3js`
* React Native дуже схожий на React, але має кілька особливостей, пов’язаних із мобільними пристроями

# Урок

Solana Mobile Stack (SMS) створений, щоб допомогти розробникам створювати мобільні dApp з безшовним користувацьким досвідом. Він складається з [Mobile Wallet Adapter (MWA)](https://docs.solanamobile.com/getting-started/overview#mobile-wallet-adapter), [Seed Vault](https://docs.solanamobile.com/getting-started/overview#seed-vault) та [Solana dApp Store](https://docs.solanamobile.com/getting-started/overview#solana-dapp-store).

Найважливішим для вашого шляху розробки є Mobile Wallet Adapter (MWA). Найпростіший спосіб почати — використати Mobile Wallet Adapter з React Native для створення простого Android-додатка. Цей урок передбачає, що ви знайомі з React та програмуванням Solana. Якщо це не так, [почніть наш курс з самого початку](./intro-to-cryptography) і повертайтесь сюди, коли будете готові!

## Вступ до Solana Mobile

У цих розділах ми розроблятимемо мобільні додатки, що взаємодіють із мережею Solana. Це відкриває зовсім нову парадигму використання криптовалют і поведінки користувачів.

### Варіанти використання Solana Mobile

Ось кілька прикладів того, що може дати розробка Solana для мобільних пристроїв:

**Мобільний банкінг і трейдинг (DeFi)**

Наразі більшість традиційного банкінгу відбувається у нативних мобільних додатках. Завдяки SMS ви можете банківські операції та трейдинг здійснювати у власних мобільних додатках з вашим власним гаманцем, де ви контролюєте ключі.

**Мобільні ігри з мікроплатежами на Solana**

Мобільні ігри складають близько 50% загальної вартості індустрії відеоігор, переважно завдяки дрібним внутрішньоігровим покупкам. Однак комісії за обробку платежів зазвичай означають мінімальну покупку на рівні \$0.99 USD. З Solana можливі справжні мікроплатежі. Потрібне додаткове життя? Це буде 0.0001 SOL.

**Мобільна електронна комерція**

SMS дає змогу новому поколінню мобільних покупців оплачувати товари прямо зі свого улюбленого гаманця Solana. Уявіть світ, де ваш гаманець Solana працює так само зручно, як Apple Pay.

Підсумовуючи, мобільна крипта відкриває багато можливостей. Давайте зануримось і дізнаємося, як стати частиною цього:

### Чим розробка Solana для нативних мобільних додатків відрізняється від веб-розробки

Взаємодія з гаманцем Solana на мобільних пристроях дещо відрізняється від вебу. Основна функціональність гаманця залишається та сама: гаманець зберігає ваші приватні ключі та використовує їх для підпису й відправлення транзакцій. Щоб уникнути різних інтерфейсів між гаманцями, розробники винесли цю функціональність у стандарт Solana Wallet Adapter. Цей стандарт застосовується у веб. Мобільним аналогом є Mobile Wallet Adapter (MWA).

Різниця між цими двома стандартами зумовлена різною архітектурою веб- і мобільних гаманців. Веб-гаманці — це просто розширення для браузера, які вставляють функції wallet adapter у об'єкт `window` вашої вебсторінки. Це надає сайту доступ до них. Натомість мобільні гаманці — це нативні застосунки на мобільній операційній системі. І немає прямого способу передати функції від одного нативного застосунку до іншого. Mobile Wallet Adapter створений саме для того, щоб будь-який застосунок, написаний будь-якою мовою, міг під'єднуватися до нативного мобільного гаманця.

Ми детально розглянемо специфіку Mobile Wallet Adapter у [наступному уроці](./mwa-deep-dive), але по суті він відкриває WebSocket-з'єднання між застосунками для забезпечення комунікації. Таким чином, окремий застосунок може передати гаманець-застосунку транзакцію для підпису й відправлення, а гаманець-застосунок може у відповідь надсилати відповідні оновлення статусу.

### Підтримувані операційні системи

На момент написання, Mobile Wallet Adapter підтримується лише на Android.

В Android WebSocket-з’єднання може зберігатися між застосунками навіть тоді, коли гаманець-застосунок працює у фоновому режимі.

В iOS тривалість з’єднання між застосунками навмисно обмежується операційною системою. Зокрема, iOS швидко призупиняє з’єднання, коли застосунок переходить у фоновий режим. Це перериває WebSocket-з’єднання MWA. Така поведінка є частиною фундаментального дизайну iOS (ймовірно, для збереження заряду батареї, зменшення використання мережі тощо).

Однак це не означає, що Solana dApp-и взагалі не можуть працювати на iOS. Ви все ще можете створити мобільний вебзастосунок, використовуючи бібліотеку [стандартного wallet adapter](https://github.com/solana-labs/wallet-adapter). Ваші користувачі зможуть встановити мобільний зручний гаманець, наприклад [Glow Wallet](https://glow.app/).

Решта цього уроку буде зосереджена на розробці Android-застосунків із використанням MWA.

### Підтримувані фреймворки

Solana Mobile підтримує низку різних фреймворків. Офіційно підтримуються React Native та нативний Android, а також існують SDK від спільноти для Flutter, Unity та Unreal Engine.

**Solana SDK:**

* [React Native](https://docs.solanamobile.com/react-native/quickstart) (звичайна версія та версія для Expo)
* [Android](https://docs.solanamobile.com/android-native/quickstart)

**SDK від спільноти:**

* [Flutter](https://docs.solanamobile.com/flutter/overview)
* [Unity](https://docs.solanamobile.com/unity/unity_sdk)
* [Unreal Engine](https://docs.solanamobile.com/unreal/unreal_sdk)

Щоб зробити досвід розробки максимально схожим на попередні уроки, ми працюватимемо виключно з React Native.

## Від React до React Native

React Native використовує знайомий веб-фреймворк React для створення мобільних застосунків. Хоча React і React Native дуже схожі на вигляд, між ними є деякі відмінності. Найкращий спосіб зрозуміти ці відмінності — це відчути їх у процесі написання коду. Але щоб дати вам стартову точку, ось список відмінностей, які варто мати на увазі:

* React Native компілюється в нативні застосунки для iOS та Android, тоді як React компілюється в набір вебсторінок.
* У React ви використовуєте JSX для роботи з HTML і CSS. У React Native ви використовуєте схожий синтаксис, але для керування нативними UI-компонентами. Це більше схоже на роботу з бібліотеками інтерфейсу на кшталт Chakra або Tailwind UI. Замість `<div>`, `<p>` і `<img>` ви будете використовувати `<View>`, `<Text>` і `<Image>`.
* Взаємодія з інтерфейсом відрізняється. Замість `onClick` ви використовуватимете `onPress` та інші жести.
* Багато стандартних бібліотек React та Node можуть бути несумісними з React Native. На щастя, існують аналоги найпопулярніших бібліотек, адаптовані для React Native, а також часто можна використовувати поліфіли (polyfills), щоб зробити доступними бібліотеки з Node. Якщо ви не знайомі з поліфілами, ознайомтесь із [документацією MDN](https://developer.mozilla.org/en-US/docs/Glossary/Polyfill). Коротко кажучи, поліфіли — це активні заміни вбудованих Node-бібліотек, які дозволяють використовувати їх у середовищах, де Node не працює.
* Налаштувати середовище розробки в React Native може бути складно. Потрібно встановити Android Studio для компіляції під Android та XCode для iOS. React Native має [дуже детальний гайд](https://reactnative.dev/docs/environment-setup?guide=native) для цього.
* Для звичайної розробки і тестування ви будете використовувати фізичний мобільний пристрій або емулятор, щоб запускати ваш код. Для цього використовується інструмент Metro, який встановлюється за замовчуванням. Цей процес також описаний у гайді з налаштування React Native.
* React Native дає доступ до апаратних можливостей телефону, яких немає у React — наприклад, камера, акселерометр та інше.
* React Native вводить нові конфігураційні файли та каталоги для збірки. Наприклад, папки `ios` і `android` містять специфічну для платформи інформацію. Також є конфігураційні файли на кшталт `Gemfile` та `metro.config.js`. Загалом, не варто змінювати ці налаштування — просто зосередьтеся на написанні коду, стартовою точкою якого буде файл `App.tsx`.

Це правда, що в React Native є своя крива навчання, але якщо ви вже знайомі з React, то шлях до створення мобільних додатків буде набагато коротшим, ніж здається! Спочатку може бути непривично, але вже через кілька годин роботи з React Native ви почнете відчувати себе значно впевненіше. Після [лабораторної з цього уроку](#Лабораторна) ваша впевненість, швидше за все, зросте ще більше.

## Створення Solana dApp з React Native

Solana dApp на React Native майже ідентичні React dApp. Головна різниця полягає у взаємодії з гаманцем. Замість того, щоб гаманець був доступний у браузері, ваш dApp створює сесію MWA (Mobile Wallet Adapter) з обраним додатком-гаманцем через WebSocket. На щастя, ця складність прихована за бібліотекою MWA. Єдина відмінність, яку вам потрібно знати, — що кожного разу, коли треба викликати гаманець, ви використовуватимете функцію `transact`, про яку ми поговоримо трохи пізніше.

![dApp Flow](../assets/basic-solana-mobile-flow.png)

### Читання даних

Читання даних із Solana-кластера в React Native відбувається точно так само, як і в React. Ви використовуєте хук `useConnection`, щоб отримати об’єкт `Connection`. За його допомогою можна отримати інформацію про акаунт. Оскільки читання даних безкоштовне, нам не потрібно насправді підключатися до гаманця.

```tsx
const account = await connection.getAccountInfo(account);
```

Якщо потрібно освіжити пам’ять, перегляньте наш [урок про читання даних з блокчейну](./intro-to-reading-data).

### Підключення до гаманця

Запис даних у блокчейн має відбуватися через транзакцію. Транзакції потрібно підписати одним або кількома приватними ключами та надіслати RPC-провайдеру. Це практично завжди робиться через додаток-гаманець.

Типова взаємодія з гаманцем у вебі відбувається через браузерне розширення. На мобільних пристроях для запуску сесії MWA використовується WebSocket. Конкретно, Android-додатки працюють через intents — децентралізований додаток (dApp) транслює intent із схемою `solana-wallet://`.

![Підключення](../assets/basic-solana-mobile-connect.png)

Коли додаток-гаманець отримує цей intent, він відкриває з’єднання з dApp, що ініціював сесію. Ваш dApp відправляє цей intent за допомогою функції `transact`:

```tsx
transact(async (wallet: Web3MobileWallet) => {
	// Wallet Action code here
}
```

Це дасть вам доступ до об’єкта `Web3MobileWallet`, який можна використовувати для відправлення транзакцій на гаманець. Ще раз: доступ до гаманця завжди має відбуватися через callback функції `transact`.

### Підписання та відправлення транзакцій

Відправлення транзакції відбувається всередині callback функції `transact`. Загальний порядок такий:

1. Встановіть сесію з гаманцем за допомогою `transact`, яка приймає callback функцію `async (wallet: Web3MobileWallet) => {...}`.
2. Всередині callback викличте авторизацію за допомогою `wallet.authorize` або `wallet.reauthorize` — залежно від стану гаманця.
3. Підпишіть транзакцію за допомогою `wallet.signTransactions` або підпишіть і одразу надішліть за допомогою `wallet.signAndSendTransactions`.

![Виконання транзакцій](../assets/basic-solana-mobile-transact.png)

Примітка: Можливо, вам варто створити хук `useAuthorization()` для керування станом авторизації гаманця. Ми потренуємось із цим у [Лабораторній роботі](#lab).

Ось приклад відправки транзакції з використанням MWA:
```tsx
const { authorizeSession } = useAuthorization();
const { connection } = useConnection();

const sendTransactions = (transaction: Transaction)=> {

	transact(async (wallet: Web3MobileWallet) => {
		const latestBlockhashResult = await connection.getLatestBlockhash();
		const authResult = await authorizeSession(wallet);

		const updatedTransaction = new Transaction({
      ...transaction,
      ...latestBlockhashResult,
      feePayer: authResult.publicKey,
    });

		const signature = await wallet.signAndSendTransactions({
      transactions: [transaction],
    });
	})
}
```

### Налагодження

Оскільки у процесі відправки транзакцій задіяні два застосунки, налагодження може бути складним. Зокрема, ви не зможете бачити журнали налагодження гаманця так, як бачите журнали своєї dApp.

На щастя, [Logcat у Android Studio](https://developer.android.com/studio/debug/logcat) дозволяє переглядати журнали всіх застосунків на вашому пристрої.

Якщо ви не хочете використовувати Logcat, інший підхід — використовувати гаманець лише для підпису транзакцій, а надсилати їх уже у своєму коді. Це дає змогу краще налагоджувати транзакції у разі виникнення проблем.

### Реліз

Розгортання мобільних застосунків саме по собі є складним завданням. Для криптозастосунків воно ще складніше. Є дві основні причини цього: безпека користувачів і фінансові стимули.

По-перше, більшість маркетплейсів мобільних застосунків мають політики, що обмежують використання блокчейну. Криптовалюта досі є "дикою картою" з точки зору регулювання. Маркетплейси вважають, що захищають користувачів, дотримуючись суворих вимог щодо застосунків, пов’язаних із блокчейном.

По-друге, якщо ви використовуєте криптовалюту для «покупок» у застосунку, це може розглядатися як спосіб обійти комісію маркетплейсу (яка зазвичай становить від 15 до 30%). Це прямо заборонено політикою більшості маркетплейсів, оскільки вони прагнуть захистити свої джерела доходів.

Це, безумовно, перешкоди, але є й позитивні новини. Ось на що слід звернути увагу для кожного маркетплейсу:

* **App Store (iOS)** – Сьогодні ми говорили лише про Android через технічні обмеження MWA. Але політики Apple одні з найсуворіших і ускладнюють існування Solana dApp-застосунків. Наразі Apple має доволі жорстку анти-крипто політику. Гаманці зазвичай проходять модерацію, але будь-яку функцію, що нагадує покупку за криптовалюту, можуть відхилити.
* **Google Play (Android)** – Google загалом більш лояльний, але є нюанси. Станом на листопад 2023 року Google впроваджує [нові крипто-політики](https://www.theverge.com/2023/7/12/23792720/android-google-play-blockchain-crypto-nft-apps), які чіткіше регламентують, що дозволено, а що ні. Ознайомтеся з ними.
* **Steam** – Взагалі не дозволяє ігри з використанням криптовалюти
  > “побудовані на блокчейн-технології та передбачають випуск або обмін криптовалют чи NFT”
* **Сайти для завантаження / Ваш сайт** – Залежно від цільової платформи, ви можете викладати ваш dApp на власному сайті. Але більшість користувачів з обережністю ставляться до встановлення мобільних застосунків із сайтів.
* **dApp Store (Solana)** – Solana побачила проблеми з дистрибуцією мобільних dApp-застосунків у традиційних маркетплейсах і створила власний. У рамках стеку SMS вони запустили [Solana dApp Store](https://docs.solanamobile.com/getting-started/overview#solana-dapp-store).

## Висновок

Почати розробку мобільних застосунків на Solana досить просто завдяки SMS. Хоча React Native трохи відрізняється від React, код, який вам потрібно писати, більше схожий, ніж відмінний. Основна різниця полягає в тому, що частина коду, яка взаємодіє з гаманцями, буде знаходитись у callback-функції `transact`. Якщо вам потрібно пригадати основи розробки на Solana, зверніться до наших інших уроків.

# Лабораторна робота

Давайте попрактикуємося, створюючи простий мобільний лічильник для Android на React Native. Цей застосунок буде взаємодіяти з програмою лічильника Anchor, яку ми робили у уроці [Intro to client-side Anchor development](https://www.soldev.app/course/intro-to-anchor-frontend). Цей dApp просто відображає лічильник і дозволяє користувачам збільшувати його через Solana програму. У застосунку ми зможемо бачити поточне значення лічильника, підключати гаманець і збільшувати лічильник. Усе це буде працювати на Devnet, а компіляція буде тільки для Android.

Ця програма вже існує та розгорнута в Devnet. Якщо хочете більше контексту, можете переглянути [код розгорнутої програми](https://github.com/Unboxed-Software/anchor-ping-frontend/tree/solution-decrement).

Ми писатимемо цей застосунок на чистому React Native без початкового шаблону. Solana Mobile надає [React Native шаблон](https://docs.solanamobile.com/react-native/react-native-scaffold), який спрощує деякі рутинні налаштування, але немає кращого способу навчитися, ніж зробити щось з нуля.

### 0. Передумови

React Native дає змогу писати мобільні застосунки, використовуючи схожі патерни, як у React. Однак, під капотом наш React-код потрібно скомпілювати у мови та фреймворки, які працюють з рідною ОС пристрою. Для цього потрібно виконати кілька початкових налаштувань:

1. [Налаштуйте середовище розробки React Native](https://reactnative.dev/docs/environment-setup?guide=native#creating-a-new-application). Пройдіть [***всю статтю***](https://reactnative.dev/docs/environment-setup?guide=native#creating-a-new-application), орієнтуючись на Android як цільову ОС. Для зручності нижче наведено основні кроки. Врахуйте, що вихідна стаття може змінюватися з часу написання до моменту вашого читання. Вона є вашим основним джерелом інформації.
   1. Встановіть залежності
   2. Встановіть Android Studio
   3. Налаштуйте змінну середовища **ANDROID\_HOME**
   4. Створіть новий зразковий проект (він потрібен лише для налаштування емулятора)
      1. Якщо виникне помилка `✖ Copying template`, додайте прапорець `--npm` у кінці команди
   
            ```bash
            npx react-native@latest init AwesomeProject
            ✔ Downloading template
            ✖ Copying template
            
            npx react-native@latest init AwesomeProject --npm
            ✔ Downloading template
            ✔ Copying template
            ```
        
   5. Запустіть і скомпілюйте приклад проєкту на вашому емуляторі
6. Встановіть і запустіть Solana fake wallet
   1. Встановіть репозиторій
        
        ```bash
        git clone https://github.com/solana-mobile/mobile-wallet-adapter.git
        ```

   2. В Android Studio, `Open project > Navigate to the cloned directory > Select mobile-wallet-adapter/android`
   3. Після завершення завантаження проекту в Android Studio виберіть `fakewallet` у випадаючому меню конфігурації збірки/запуску у верхньому правому куті
        
        ![Fake Wallet](../assets/basic-solana-mobile-fake-wallet.png)
        
    4. Для налагодження вам знадобиться `Logcat`. Тепер, коли ваш фейковий гаманець запущений на емуляторі, перейдіть до `View -> Tool Windows -> Logcat`. Це відкриє консоль, у якій буде виводитися інформація про роботу фейкового гаманця.
5. (Необов’язково) Встановіть інші Solana-гаманці, як-от Phantom, з Google Play.

Наостанок, якщо виникнуть проблеми з версією Java — вам слід використовувати Java версії 11. Щоб перевірити поточну версію, введіть `java --version` у вашому терміналі.

### 1. Сплануйте структуру застосунку

Перш ніж переходити до написання коду, давайте концептуалізуємо загальну структуру застосунку. Нагадаємо, цей застосунок буде підключатися до програми для підрахунку, яку ми вже задеплоїли на Devnet, і взаємодіяти з нею. Для цього нам знадобиться таке:

* Об'єкт `Connection` для взаємодії з Solana (`ConnectionProvider.tsx`)
* Доступ до нашої програми для підрахунку (`ProgramProvider.tsx`)
* Авторизація гаманця для підпису та надсилання запитів (`AuthProvider.tsx`)
* Текст для відображення значення лічильника (`CounterView.tsx`)
* Кнопка для збільшення значення лічильника (`CounterButton.tsx`)
  
Існуватимуть й інші файли та аспекти, які потрібно врахувати, але це — основні файли, з якими ми працюватимемо та які створюватимемо.

### 2. Створення застосунку

Тепер, коли ми окреслили базове налаштування та структуру, давайте створимо новий застосунок за допомогою наступної команди:

```bash
npx react-native@latest init counter --npm
```

Ця команда створює для нас новий проект React Native під назвою `counter`.

Перевірмо, чи все налаштовано правильно: запустіть застосунок за замовчуванням і відкрийте його на вашому емуляторі Android.

```bash
cd counter
npm run android
```

This should open and run the app in your Android emulator. If you run into problems, check to make sure you’ve accomplished everything in the [prerequisites section](#0-prerequisites).

### 3. Встановіть залежності

Нам потрібно додати залежності для роботи з Solana. [Документація Solana Mobile надає зручний список пакетів](https://docs.solanamobile.com/react-native/setup) та пояснення, навіщо вони потрібні:

* `@solana-mobile/mobile-wallet-adapter-protocol`: API для React Native/Javascript, що забезпечує взаємодію з гаманцями, сумісними з MWA
* `@solana-mobile/mobile-wallet-adapter-protocol-web3js`: зручна обгортка для використання типових примітивів із [@solana/web3.js](https://github.com/solana-labs/solana-web3.js), таких як `Transaction` і `Uint8Array`
* `@solana/web3.js`: веб-бібліотека Solana для взаємодії з мережею Solana через [JSON RPC API](https://docs.solana.com/api/http)
* `react-native-get-random-values`: поліфіл для генерації криптографічно безпечних випадкових значень, потрібний для криптобібліотеки `web3.js` у React Native
* `buffer`: поліфіл для `Buffer`, також необхідний для роботи `web3.js` у середовищі React Native

Окрім цього списку, додамо ще два пакети:
* `@coral-xyz/anchor`: клієнт Anchor для TypeScript
* `assert`: поліфіл, який дозволяє Anchor коректно працювати
* `text-encoding-polyfill`: поліфіл, необхідний для створення об’єкта `Program`

Якщо ви не знайомі з цим: поліфіли активно замінюють бібліотеки, які є в Node за замовчуванням, щоб вони працювали в будь-якому середовищі, де Node не запущений. Ми скоро завершимо налаштування поліфілів. А поки що встановіть залежності за допомогою наступної команди:

```bash
npm install \
  @solana/web3.js \
  @solana-mobile/mobile-wallet-adapter-protocol-web3js \
  @solana-mobile/mobile-wallet-adapter-protocol \
  react-native-get-random-values \
  buffer \
  @coral-xyz/anchor \
  assert \
  text-encoding-polyfill
```

### 4. Створіть ConnectionProvider.tsx

Давайте почнемо додавати функціонал Solana. Створіть нову папку з назвою `components`, а в ній — файл `ConnectionProvider.tsx`. Цей провайдер обгорне весь проект і зробить об’єкт `Connection` доступним по всьому застосунку. Сподіваюся, ви помічаєте закономірність: це ідентично React-патернам, які ми використовували протягом усього курсу.

```tsx
import {Connection, ConnectionConfig} from '@solana/web3.js';
import React, {ReactNode, createContext, useContext, useMemo} from 'react';

export interface ConnectionProviderProps {
  children: ReactNode;
  endpoint: string;
  config?: ConnectionConfig;
}

export interface ConnectionContextState {
  connection: Connection;
}

const ConnectionContext = createContext<ConnectionContextState>(
  {} as ConnectionContextState,
);

export function ConnectionProvider(props: ConnectionProviderProps){
  const {children, endpoint, config = {commitment: 'confirmed'}} = {...props};
  const connection = useMemo(
    () => new Connection(endpoint, config),
    [config, endpoint],
  );

  return (
    <ConnectionContext.Provider value={{connection}}>
      {children}
    </ConnectionContext.Provider>
  );
};

export const useConnection = (): ConnectionContextState =>
  useContext(ConnectionContext);
```

### 5. Створіть AuthProvider.tsx

Наступним компонентом Solana, який нам знадобиться, є провайдер авторизації. Це одна з головних відмінностей між мобільною та веб-розробкою. Те, що ми реалізуємо тут, приблизно відповідає `WalletProvider`, до якого ми звикли у вебзастосунках. Однак, оскільки ми використовуємо Android і його нативно встановлені гаманці, процес підключення та використання їх дещо відрізняється. Найголовніше — нам потрібно дотримуватися протоколу MWA. 

У нашому `AuthProvider` ми реалізуємо таке:

* `accounts`: якщо користувач має кілька гаманців, різні акаунти зберігаються в цьому масиві акаунтів.
* `selectedAccount`: поточний вибраний акаунт для транзакції.
* `authorizeSession(wallet)`: авторизує (або повторно авторизує, якщо токен протерміновано) `wallet` для користувача і повертає акаунт, який буде обраним акаунтом сесії. Змінна `wallet` походить із callback функції `transact`, яку ви викликаєте окремо щоразу, коли хочете взаємодіяти з гаманцем.
* `deauthorizeSession(wallet)`: скасовує авторизацію `wallet`.
* `onChangeAccount`: обробник, який спрацьовує при зміні `selectedAccount`.

Також додамо кілька допоміжних методів:

* `getPublicKeyFromAddress(base64Address)`: створює новий об’єкт Public Key з адреси у форматі Base64, отриманої з об’єкта `wallet`
* `getAuthorizationFromAuthResult`: обробляє результат авторизації, витягує з нього потрібні дані і повертає об’єкт контексту `Authorization`

Усе це ми зробимо доступним через хук `useAuthorization`.

Оскільки цей провайдер практично однаковий у всіх застосунках, ми надамо вам повну реалізацію, яку можна просто скопіювати і вставити. Детальніше про MWA розглянемо у наступних уроках.

Створіть файл `AuthProvider.tsx` у папці `components` та вставте туди наступний код:

```tsx
import {Cluster, PublicKey} from '@solana/web3.js';
import {
  Account as AuthorizedAccount,
  AuthorizationResult,
  AuthorizeAPI,
  AuthToken,
  Base64EncodedAddress,
  DeauthorizeAPI,
  ReauthorizeAPI,
} from '@solana-mobile/mobile-wallet-adapter-protocol';
import {toUint8Array} from 'js-base64';
import {useState, useCallback, useMemo, ReactNode} from 'react';
import React from 'react';

export const AuthUtils = {
  getAuthorizationFromAuthResult: (
    authResult: AuthorizationResult,
    previousAccount?: Account,
  ): Authorization => {
    let selectedAccount: Account;
    if (
      //no wallet selected yet
      previousAccount === null ||
      //the selected wallet is no longer authorized
      !authResult.accounts.some(
        ({address}) => address === previousAccount.address,
      )
    ) {
      const firstAccount = authResult.accounts[0];
      selectedAccount = AuthUtils.getAccountFromAuthorizedAccount(firstAccount);
    } else {
      selectedAccount = previousAccount;
    }
    return {
      accounts: authResult.accounts.map(
        AuthUtils.getAccountFromAuthorizedAccount,
      ),
      authToken: authResult.auth_token,
      selectedAccount,
    };
  },

  getAccountFromAuthorizedAccount: (
    authAccount: AuthorizedAccount,
  ): Account => {
    return {
      ...authAccount,
      publicKey: AuthUtils.getPublicKeyFromAddress(authAccount.address),
    };
  },

  getPublicKeyFromAddress: (address: Base64EncodedAddress) => {
    return new PublicKey(toUint8Array(address));
  },
};

export type Account = Readonly<{
  address: Base64EncodedAddress;
  label?: string;
  publicKey: PublicKey;
}>;

type Authorization = Readonly<{
  accounts: Account[];
  authToken: AuthToken;
  selectedAccount: Account;
}>;

export const AppIdentity = {
  name: 'Solana Counter Incrementor',
};

export type AuthorizationProviderContext = {
  accounts: Account[] | null;
  authorizeSession: (wallet: AuthorizeAPI & ReauthorizeAPI) => Promise<Account>;
  deauthorizeSession: (wallet: DeauthorizeAPI) => void;
  onChangeAccount: (nextSelectedAccount: Account) => void;
  selectedAccount: Account | null;
};

const AuthorizationContext = React.createContext<AuthorizationProviderContext>({
  accounts: null,
  authorizeSession: (_wallet: AuthorizeAPI & ReauthorizeAPI) => {
    throw new Error('Provider not initialized');
  },
  deauthorizeSession: (_wallet: DeauthorizeAPI) => {
    throw new Error('Provider not initialized');
  },
  onChangeAccount: (_nextSelectedAccount: Account) => {
    throw new Error('Provider not initialized');
  },
  selectedAccount: null,
});

export type AuthProviderProps = {
  children: ReactNode;
  cluster: Cluster;
};

export function AuthorizationProvider(props: AuthProviderProps) {
  const {children, cluster} = {...props};
  const [authorization, setAuthorization] = useState<Authorization | null>(
    null,
  );

  const handleAuthorizationResult = useCallback(
    async (authResult: AuthorizationResult): Promise<Authorization> => {
      const nextAuthorization = AuthUtils.getAuthorizationFromAuthResult(
        authResult,
        authorization?.selectedAccount,
      );
      setAuthorization(nextAuthorization);

      return nextAuthorization;
    },
    [authorization, setAuthorization],
  );

  const authorizeSession = useCallback(
    async (wallet: AuthorizeAPI & ReauthorizeAPI) => {
      const authorizationResult = await (authorization
        ? wallet.reauthorize({
            auth_token: authorization.authToken,
            identity: AppIdentity,
          })
        : wallet.authorize({cluster, identity: AppIdentity}));
      return (await handleAuthorizationResult(authorizationResult))
        .selectedAccount;
    },
    [authorization, handleAuthorizationResult],
  );

  const deauthorizeSession = useCallback(
    async (wallet: DeauthorizeAPI) => {
      if (authorization?.authToken === null) {
        return;
      }

      await wallet.deauthorize({auth_token: authorization.authToken});
      setAuthorization(null);
    },
    [authorization, setAuthorization],
  );

  const onChangeAccount = useCallback(
    (nextAccount: Account) => {
      setAuthorization(currentAuthorization => {
        if (
          //check if the account is no longer authorized
          !currentAuthorization?.accounts.some(
            ({address}) => address === nextAccount.address,
          )
        ) {
          throw new Error(`${nextAccount.address} is no longer authorized`);
        }

        return {...currentAuthorization, selectedAccount: nextAccount};
      });
    },
    [setAuthorization],
  );

  const value = useMemo(
    () => ({
      accounts: authorization?.accounts ?? null,
      authorizeSession,
      deauthorizeSession,
      onChangeAccount,
      selectedAccount: authorization?.selectedAccount ?? null,
    }),
    [authorization, authorizeSession, deauthorizeSession, onChangeAccount],
  );

  return (
    <AuthorizationContext.Provider value={value}>
      {children}
    </AuthorizationContext.Provider>
  );
}

export const useAuthorization = () => React.useContext(AuthorizationContext);
```

### 6. Створіть ProgramProvider.tsx

Останнім провайдером, який нам потрібен, є провайдер програми. Він надасть доступ до програми лічильника, з якою ми хочемо взаємодіяти.

Оскільки ми використовуємо клієнт Anchor TS для роботи з нашою програмою, нам потрібен IDL програми. Спочатку створіть у корені папку `models`, а в ній — новий файл `anchor-counter.ts`. Вставте в цей файл вміст [Anchor Counter IDL](../assets/counter-rn-idl.ts).

Далі створіть файл `ProgramProvider.tsx` у папці `components`. Всередині ми створимо провайдера програми, який зробить доступними нашу програму та PDA лічильника:

```tsx
import {AnchorProvider, IdlAccounts, Program, setProvider} from '@coral-xyz/anchor';
import {Keypair, PublicKey} from '@solana/web3.js';
import {AnchorCounter, IDL} from '../models/anchor-counter';
import React, {
  ReactNode,
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState,
} from 'react';
import {useConnection} from './ConnectionProvider';

export type CounterAccount = IdlAccounts<AnchorCounter>['counter'];

export type ProgramContextType = {
  program: Program<AnchorCounter> | null;
  counterAddress: PublicKey | null;
};

export const ProgramContext = createContext<ProgramContextType>({
  program: null,
  counterAddress: null,
});

export type ProgramProviderProps = {
  children: ReactNode;
};

export function ProgramProvider(props: ProgramProviderProps) {
  const { children } = props;
  const {connection} = useConnection();
  const [program, setProgram] = useState<Program<AnchorCounter> | null>(null);
  const [counterAddress, setCounterAddress] = useState<PublicKey | null>(null);

  const setup = useCallback(async () => {
    const programId = new PublicKey(
      'ALeaCzuJpZpoCgTxMjJbNjREVqSwuvYFRZUfc151AKHU',
    );

    const MockWallet = {
      signTransaction: () => Promise.reject(),
      signAllTransactions: () => Promise.reject(),
      publicKey: Keypair.generate().publicKey,
    };

    const provider = new AnchorProvider(connection, MockWallet, {});
    setProvider(provider);

    const programInstance = new Program<AnchorCounter>(
      IDL,
      programId,
      provider,
    );

    const [counterProgramAddress] = PublicKey.findProgramAddressSync(
      [Buffer.from('counter')],
      programId,
    );

    setProgram(programInstance);
    setCounterAddress(counterProgramAddress);
  }, [connection]);

  useEffect(() => {
    setup();
  }, [setup]);

  const value: ProgramContextType = useMemo(
    () => ({
      program,
      counterAddress,
    }),
    [program, counterAddress],
  );

  return (
    <ProgramContext.Provider value={value}>{children}</ProgramContext.Provider>
  );
}

export const useProgram = () => useContext(ProgramContext);
```

### 7. Змініть App.tsx

Тепер, коли у нас є всі провайдери, давайте обгорнемо ними наш застосунок. Ми перепишемо стандартний `App.tsx` з такими змінами:

* Імпортуємо наші провайдери та додамо поліфіли
* Обгорнемо застосунок спочатку в `ConnectionProvider`, потім у `AuthorizationProvider` і наостанок у `ProgramProvider`
* Передамо наш Devnet endpoint у `ConnectionProvider`
* Передамо кластер у `AuthorizationProvider`
* Замінемо стандартний внутрішній `<View>` на `<MainScreen />` — екран, який ми створимо на наступному кроці

```tsx
// Polyfills at the top
import "text-encoding-polyfill";
import "react-native-get-random-values";
import { Buffer } from "buffer";
global.Buffer = Buffer;

import { clusterApiUrl } from "@solana/web3.js";
import { ConnectionProvider } from "./components/ConnectionProvider";
import { AuthorizationProvider } from "./components/AuthProvider";
import { ProgramProvider } from "./components/ProgramProvider";
import { MainScreen } from "./screens/MainScreen"; // Going to make this
import React from "react";

export default function App() {
  const cluster = "devnet";
  const endpoint = clusterApiUrl(cluster);

  return (
    <ConnectionProvider
      endpoint={endpoint}
      config={{ commitment: "processed" }}
    >
      <AuthorizationProvider cluster={cluster}>
        <ProgramProvider>
          <MainScreen />
        </ProgramProvider>
      </AuthorizationProvider>
    </ConnectionProvider>
  );
}
```

### 8. Створіть MainScreen.tsx

Тепер з’єднаємо все разом, щоб створити наш інтерфейс. Створіть нову папку `screens` та в ній файл `MainScreen.tsx`. У цьому файлі ми лише структуруємо екран для відображення двох компонентів, які ще потрібно створити: `CounterView` і `CounterButton`.

Також у цьому файлі ми використовуємо `StyleSheet` з React Native. Це ще одна відмінність від звичайного React. Не хвилюйтеся, він працює дуже схоже на CSS.

Вставте у файл `screens/MainScreen.tsx` наступний код:
```tsx
import { StatusBar, StyleSheet, View } from 'react-native';
import { CounterView } from '../components/CounterView';
import { CounterButton } from '../components/CounterButton';
import React from 'react';

const mainScreenStyles = StyleSheet.create({
  container: {
    height: '100%',
    width: '100%',
    backgroundColor: 'lightgray',
  },

  incrementButtonContainer: {position: 'absolute', right: '5%', bottom: '3%'},
  counterContainer: {
    alignContent: 'center',
    alignItems: 'center',
    justifyContent: 'center',
  },
});

export function MainScreen() {
  return (
    <View style={mainScreenStyles.container}>
      <StatusBar barStyle="light-content" backgroundColor="darkblue" />
      <View
        style={{
          ...mainScreenStyles.container,
          ...mainScreenStyles.counterContainer,
        }}>
        <CounterView />
      </View>
      <View style={mainScreenStyles.incrementButtonContainer}>
        <CounterButton />
      </View>
    </View>
  );
}
```

### 9. Створіть CounterView\.tsx

`CounterView` — це перший із двох файлів, специфічних для нашої програми. Його завдання — отримувати та відслідковувати оновлення акаунту `Counter`. Оскільки тут ми лише слухаємо зміни, робота з MWA не потрібна. Це буде практично ідентично вебзастосунку. Ми використаємо об’єкт `Connection` для прослуховування `programAddress`, вказаного у `ProgramProvider.tsx`. Коли акаунт змінюється — оновлюємо UI.

Вставте у файл `components/CounterView.tsx` наступний код:

```tsx
import {View, Text, StyleSheet} from 'react-native';
import {useConnection} from './ConnectionProvider';
import {useProgram, CounterAccount} from './ProgramProvider';
import {useEffect, useState} from 'react';
import {AccountInfo} from '@solana/web3.js';
import React from 'react';

const counterStyle = StyleSheet.create({
  counter: {
    fontSize: 48,
    fontWeight: 'bold',
    color: 'black',
    textAlign: 'center',
  },
});

export function CounterView() {
  const {connection} = useConnection();
  const {program, counterAddress} = useProgram();
  const [counter, setCounter] = useState<CounterAccount>();

  // Fetch Counter Info
  useEffect(() => {
    if (!program || !counterAddress) return;

    program.account.counter.fetch(counterAddress).then(setCounter);

    const subscriptionId = connection.onAccountChange(
      counterAddress,
      (accountInfo: AccountInfo<Buffer>) => {
        try {
          const data = program.coder.accounts.decode(
            'counter',
            accountInfo.data,
          );
          setCounter(data);
        } catch (e) {
          console.log('account decoding error: ' + e);
        }
      },
    );

    return () => {
      connection.removeAccountChangeListener(subscriptionId);
    };
  }, [program, counterAddress, connection]);

  if (!counter) return <Text>Loading...</Text>;

  return (
    <View>
      <Text>Current counter</Text>
      <Text style={counterStyle.counter}>{counter.count.toString()}</Text>
    </View>
  );
}
```

### 10. Створіть CounterButton.tsx

Нарешті, останній компонент — `CounterButton`. Ця кнопка, що "плаває" на екрані, виконуватиме наступні дії у функції `incrementCounter`:

* Викликає `transact`, щоб отримати доступ до мобільного гаманця
* Авторизує сесію за допомогою `authorizeSession` з хука `useAuthorization`
* Запитує ейрдроп у Devnet, щоб профінансувати транзакцію, якщо недостатньо SOL
* Створює транзакцію `increment`
* Викликає `signAndSendTransactions`, щоб гаманець підписав і надіслав транзакцію

Примітка: фейковий Solana-гаманець, який ми використовуємо, генерує нову пару ключів щоразу при перезапуску застосунку, тому нам потрібно перевіряти наявність коштів і виконувати ейрдроп кожного разу. Це лише для демонстраційних цілей — у продакшн-середовищі так робити не можна.

Створіть файл `CounterButton.tsx` і вставте в нього наступне:

```tsx
import {
  Alert,
  Platform,
  Pressable,
  StyleSheet,
  Text,
  ToastAndroid,
} from 'react-native';
import { useAuthorization } from './AuthProvider';
import { useProgram } from './ProgramProvider';
import { useConnection } from './ConnectionProvider';
import {
  transact,
  Web3MobileWallet,
} from '@solana-mobile/mobile-wallet-adapter-protocol-web3js';
import { LAMPORTS_PER_SOL, Transaction } from '@solana/web3.js';
import { useState } from 'react';
import React from 'react';

const floatingActionButtonStyle = StyleSheet.create({
  container: {
    height: 64,
    width: 64,
    alignItems: 'center',
    borderRadius: 40,
    justifyContent: 'center',
    elevation: 4,
    marginBottom: 4,
    backgroundColor: 'blue',
  },

  text: {
    fontSize: 24,
    color: 'white',
  },
});

export function CounterButton() {
  const {authorizeSession} = useAuthorization();
  const {program, counterAddress} = useProgram();
  const {connection} = useConnection();
  const [isTransactionInProgress, setIsTransactionInProgress] = useState(false);

  const showToastOrAlert = (message: string) => {
    if (Platform.OS === 'android') {
      ToastAndroid.show(message, ToastAndroid.SHORT);
    } else {
      Alert.alert(message);
    }
  };

  const incrementCounter = () => {
    if (!program || !counterAddress) return;

    if (!isTransactionInProgress) {
      setIsTransactionInProgress(true);

      transact(async (wallet: Web3MobileWallet) => {
        const authResult = await authorizeSession(wallet);
        const latestBlockhashResult = await connection.getLatestBlockhash();

        const ix = await program.methods
          .increment()
          .accounts({counter: counterAddress, user: authResult.publicKey})
          .instruction();

        const balance = await connection.getBalance(authResult.publicKey);

        console.log(
          `Wallet ${authResult.publicKey} has a balance of ${balance}`,
        );

        // When on Devnet you may want to transfer SOL manually per session, due to Devnet's airdrop rate limit
        const minBalance = LAMPORTS_PER_SOL / 1000;

        if (balance < minBalance) {
          console.log(
            `requesting airdrop for ${authResult.publicKey} on ${connection.rpcEndpoint}`,
          );
          await connection.requestAirdrop(authResult.publicKey, minBalance * 2);
        }

        const transaction = new Transaction({
          ...latestBlockhashResult,
          feePayer: authResult.publicKey,
        }).add(ix);
        const signature = await wallet.signAndSendTransactions({
          transactions: [transaction],
        });

        showToastOrAlert(`Transaction successful! ${signature}`);
      })
        .catch(e => {
          console.log(e);
          showToastOrAlert(`Error: ${JSON.stringify(e)}`);
        })
        .finally(() => {
          setIsTransactionInProgress(false);
        });
    }
  };

  return (
    <>
      <Pressable
        style={floatingActionButtonStyle.container}
        onPress={incrementCounter}>
        <Text style={floatingActionButtonStyle.text}>+</Text>
      </Pressable>
    </>
  );
}
```

### 11. Збірка та запуск

Тепер час перевірити, що все працює! Зберіть і запустіть застосунок за допомогою наступної команди:

```bash
npm run android
```

Це відкриє застосунок у вашому емуляторі. Натисніть кнопку "+" у нижньому правому куті — відкриється "фейковий гаманець". У "фейковому гаманці" є різні опції для допомоги з налагодженням. Зображення нижче показує, на які кнопки потрібно натиснути, щоб коректно протестувати застосунок:

![Counter App](../assets/basic-solana-mobile-counter-app.png)

Якщо виникнуть проблеми, ось кілька прикладів того, що це може бути, і як це виправити:

* Застосунок не збирається → Вийдіть із Metro за допомогою ctrl+c і спробуйте знову
* Нічого не відбувається після натискання `CounterButton` → Переконайтеся, що у вас встановлено гаманець Solana (наприклад, фейковий гаманець, який ми встановлювали на етапі Підготовки)
* Ви застрягли в нескінченному циклі при виклику `increment` → Ймовірно, ви досягли ліміту на ейрдроп у Devnet. Приберіть частину з ейрдропом у `CounterButton` і вручну надішліть трохи Devnet SOL на адресу вашого гаманця (виводиться в консолі)

Ось і все! Ви створили свій перший Solana Mobile dApp. Якщо десь застрягли, можете переглянути [повний код рішення](https://github.com/Unboxed-Software/solana-react-native-counter) у гілці `main` цього репозиторію.

# Завдання

Ваше завдання на сьогодні — додати до застосунку функцію зменшення лічильника. Просто додайте ще одну кнопку й викличте функцію `decrement` у вашій програмі. Ця інструкція вже існує в програмі та її IDL, тож вам потрібно лише написати клієнтський код для її виклику.

Після того, як спробуєте самостійно, можете переглянути [код рішення у гілці `solution`](https://github.com/Unboxed-Software/solana-react-native-counter/tree/solution).

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=c15928ce-8302-4437-9b1b-9aa1d65af864)!
