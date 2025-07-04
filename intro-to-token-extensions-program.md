---
назва: Вступ до програми Token Extensions
завдання:
- Дізнайтеся про програму Token Extensions
- Дізнайтеся про розширення
---

# Стислий виклад
* Існуюча програма токенів на Solana забезпечує інтерфейси для взаємозамінних (fungible) та невзаємозамінних (non-fungible) токенів. Однак, з появою потреби у нових функціях, було створено кілька форків програми токенів, що ускладнює їхнє впровадження в екосистемі.
* Щоб додавати нові функції токенів без порушення роботи для існуючих користувачів, гаманців та децентралізованих застосунків (dApps), а також забезпечити безпеку існуючих токенів, була розроблена нова програма токенів — Token Extensions Program (також відома як Token-2022).
* Token Extensions Program — це окрема програма з власною адресою, відмінною від оригінальної програми токенів. Вона підтримує ті ж самі функції, а також додаткові через розширення.

# Огляд

Програма Token Extensions, також відома як Token 2022, є надмножиною функціональності, яку надає оригінальна Token Program. Token Program задовольняє більшість потреб для взаємозамінних та невзаємозамінних токенів, використовуючи простий набір інтерфейсів і структур. Хоча вона є простою та продуктивною, згодом спільнота розробників відчула потребу в додаткових можливостях. Це призвело до створення форків Token Program,  що потенційно розділяє екосистему.

Наприклад, припустімо, що університет хоче надіслати NFT-версію диплома у гаманець випускника. Як ми можемо бути впевнені, що цей диплом ніколи не буде передано третій стороні? В поточній **Token Program** це неможливо — нам би довелося додати перевірку в інструкцію передачі, яка відхиляє всі транзакції. Одне з рішень — форкнути **Token Program** і додати цю перевірку. Але це означає створення абсолютно окремої токен-програми. Університету доведеться проводити кампанію, щоб гаманці та децентралізовані застосунки для перевірки дипломів її підтримали. А що, як різні університети захочуть різний функціонал? Доведеться створити якийсь University DAO для вирішення цих внутрішніх суперечок — або навіть кілька таких DAO... Або ж вони можуть просто використати розширення `non-transferable token` у новій Token Extensions Program, яка є базовою програмою Solana, що вже підтримується всією екосистемою.

Саме тому була створена Token Extension Program — щоб значно розширити функціональність і можливість кастомізації найбільш затребуваних функцій з оригінальної Token Program. Вона забезпечує повну підтримку всіх функцій, до яких звикли розробники, і залишає простір для подальших удосконалень. Хоча це окрема програма, дві програми значно легше підтримувати, ніж десятки.

З огляду на це, Token Extensions Program розгорнута за окремою адресою. Навіть якщо інтерфейси цих двох програм однакові, адреси програм не є взаємозамінними за жодних обставин. Тобто токен, створений за допомогою Token Program, не може взаємодіяти з Token Extensions Program. Як наслідок, якщо ми хочемо додати підтримку Token Extensions Program, наш клієнтський застосунок має містити додаткову логіку для розрізнення токенів, що належать кожній з цих програм.

Остання примітка — Token Extensions Program не є повною заміною Token Program. Якщо використання конкретного токена є дуже простим, розширення можуть бути не потрібні. У такому випадку оригінальну Token Program буде трохи доцільніше використовувати, оскільки вона не потребує проходження жодних додаткових перевірок, пов’язаних із розширеннями.

## Розширення

Розширення в Token Extensions Program — це саме розширення. Тобто будь-які додаткові дані, необхідні для розширення, додаються в кінець знайомих нам мінт-акаунтів і токен-акаунтів. Це є ключовим для того, щоб інтерфейси Token Program та Token Extensions Program залишались сумісними.

На момент написання існує [16 розширень](https://spl.solana.com/token-2022/extensions): чотири для токен-акаунтів і дванадцять для мінт-акаунтів.

**Розширення токен-акаунтів** наразі включають:

- **Обов’язкові мемо**
       Це розширення робить обов’язковим наявність мемо у всіх трансфертах, як у традиційних банківських системах.

- **Незмінна власність**
       Власник токен-акаунту зазвичай може передати право власності будь-якій іншій адресі, що корисно в багатьох випадках, але може призводити до вразливостей безпеки, особливо при роботі з Associated Token Accounts (ATA). Щоб уникнути цих проблем, використовується це розширення, яке робить неможливим переназначення власності акаунту.

	Примітка: Усі ATA (Associated Token Accounts) Програми Token Extension Program мають вбудоване розширення незмінної власності.

 - **Стан акаунту за замовчуванням**
        Розробники мінт-акаунтів можуть використовувати це розширення, яке змушує всі нові токен-акаунти бути замороженими. Таким чином, користувачі врешті-решт повинні взаємодіяти з певним сервісом, щоб розморозити свої акаунти і використовувати токени.

 - **CPI Guard**
        Це розширення захищає користувачів від авторизації дій, які для них невидимі, зокрема прихованих програм, що не є ані System, ані Token програмами. Воно це робить, обмежуючи певні дії в міжпрограмних викликах.

**Розширення мінт-акаунтів включають:**

- **Комісії за переказ**
        Програма Token Extension реалізує комісії за переказ на рівні протоколу, віднімаючи певну суму з кожного переказу на рахунок отримувача. Ця зарахована сума недоступна отримувачу і може бути викуплена будь-якою адресою, яку вказує творець мінт-акаунту.

- **Закриття мінт-акаунту**
        У Token Program можна було закривати лише токен-акаунти. Однак завдяки розширенню з правом закриття тепер можна закривати й мінт-акаунти.

	Примітка: Щоб закрити мінт-акаунт, його обіг має дорівнювати 0. Тобто всі токени, випущені цим акаунтом, мають бути спалені.

 - **Токени з нарахуваннями відсотків**
        Токени з постійно змінною вартістю потребують проміжних рішень (проксі), щоб клієнт міг показувати актуальне значення — зазвичай це вимагає регулярного ребейсу або оновлень. Це розширення дозволяє змінити спосіб відображення кількості токенів в інтерфейсі, встановивши процентну ставку для токена та отримуючи його кількість з урахуванням відсотків у будь-який момент. Зверніть увагу: відсотки є лише візуальними й не впливають на фактичну кількість токенів в акаунті.

 - **Невідчужувані токени**
        Це розширення дозволяє створювати токени, які "прив’язані" до свого власника, тобто їх не можна передавати іншим.

 - **Постійний делегат**
        Це розширення дозволяє призначити постійного делегата для мінт-акаунту. Такий уповноважений акаунт має необмежені делеговані повноваження щодо будь-якого токен-акаунту цього мінт-акаунту. Тобто він може спалювати або переказувати будь-яку кількість токенів з будь-якого акаунту. Постійного делегата, наприклад, можуть використовувати програми членства для відкликання токенів доступу або емітенти стейблкоїнів для відкликання балансів, що належать підсанкційним суб'єктам. Це потужне й водночас небезпечне розширення.

 - **Трансфер хук**
        Це розширення дозволяє творцям токенів мати більший контроль над тим, як їхні токени переказуються, завдяки можливості виклику ончейн-функції типу "hook". Творці мають розробити й задеплоїти програму, що реалізує інтерфейс хуку, а потім налаштувати свій мінт-акаунт на використання цієї програми. Після цього при кожному переказі токенів із цього мінт-акаунту викликатиметься трансфер хук.

 - **Вказівник метадати**
        Мінт-акаунт може мати кілька різних акаунтів, які претендують на роль джерела опису цього мінт-акаунту. Це розширення дозволяє творцю токена вказати адресу, що містить канонічну метадату. Вказівник може посилатися на зовнішній акаунт, наприклад, Metaplex metadata акаунт, або — якщо використовується розширення метадати — бути самонаправленим.

 - **Метадата**
        Це розширення дозволяє творцю мінт-акаунту включити метадату токена безпосередньо в мінт-акаунт. Воно завжди використовується разом із розширенням *вказівник метадати*.

- **Вказівник групи**
	Уявіть групу токенів як «колекцію» токенів. Зокрема, в NFT-колекції мінт з розширенням вказівника групи вважається колекційним NFT. Це розширення містить вказівник на акаунт, який відповідає [Token-Group Interface](https://github.com/solana-labs/solana-program-library/tree/master/token-group/interface).

- **Група**
	Це зберігає [інформацію про групу](https://github.com/solana-labs/solana-program-library/tree/master/token-group/interface) безпосередньо в мінт-акаунті. Завжди використовується разом із розширенням вказівника групи.

- **Вказівник учасника**
	Зворотним до вказівника групи є вказівник учасника. Цей вказівник посилається на акаунт, який містить дані учасника, наприклад, до якої групи він належить. В колекції NFT це будуть NFT, що входять до цієї колекції.

- **Учасник**
	Це зберігає інформацію про учасника безпосередньо в мінт-акаунті. Завжди використовується разом із розширенням вказівника учасника.

- **Конфіденційні трансфери**
        Це розширення підвищує приватність транзакцій, не розкриваючи ключові деталі, такі як сума.


Примітка: Ці розширення можна комбінувати в різних варіаціях для створення широкого спектра високофункціональних токенів.

Ми детальніше розглянемо кожне розширення в окремих уроках.

# Речі, які варто враховувати при роботі з Token Program і Token Extension Program

Хоча інтерфейси обох програм залишаються однаковими, це дві різні програми. Їхні `program ID` не є взаємозамінними, і адреси, створені з їх використанням, відрізняються. Якщо ви хочете підтримувати як токени з Token Program, так і токени з Token Extension Program, потрібно додати додаткову логіку як на стороні клієнта, так і в програмі. Ми детально розглянемо ці реалізації в наступних уроках.

# Лабораторна робота 
Тепер протестуємо деякі з цих розширень за допомогою CLI-інструменту `spl-token-cli`.

### 1. Початок роботи
Перш ніж ми зможемо використовувати розширення, потрібно встановити `spl-token-cli`. Дотримуйтесь інструкцій у [цьому гайді](https://spl.solana.com/token#setup). Після встановлення перевірте коректність, виконавши таку команду:

```bash
spl-token --version
```

Примітка: обов’язково дотримуйтесь кожного кроку з [гайда вище](https://spl.solana.com/token#setup), оскільки там також описано, як ініціалізувати локальний гаманець і зробити ейрдроп SOL.

### 2. Створення мінт-акаунту з розширенням права на закриття

Давайте створимо мінт-акаунт з розширенням `close authority`, а потім, щоб показати, що все працює — закриємо цей мінт-акаунт!

Для створення мінт-акаунту з розширенням `close authority` скористаємося CLI:

Це розширення вимагає наступних аргументів:
- `create-token` : інструкція, яку ми хочемо виконати.
- `--program-id` : цей прапорець використовується для вказівки, який program ID використовувати. `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb` — це публічна адреса, за якою розгорнуто Token Extension Program.
- `--enable-close` : цей прапорець вказує, що ми хочемо ініціалізувати мінт з правом закриття.

Виконайте наступну команду:

```bash
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb --enable-close
```

Ми побачимо вивід, схожий на наведений нижче:

```bash
Creating token 3s6mQcPHXqwryufMDwknSmkDjtxwVujfovd5gPQLvKw9 under program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb

Address:  3s6mQcPHXqwryufMDwknSmkDjtxwVujfovd5gPQLvKw9
Decimals:  9

Signature: fGQQ1eAGsnKN11FUcFhGuacpuMTGYwYEfaAVBUys4gvH4pESttRgjVKzTLSfqjeQ5rNXP92qEyBMaFFNTVPMVAD
```

Щоб переглянути деталі про новостворений мінт, можна використати команду `display`. Ця команда покаже актуальні дані про мінт-акаунт, токен-акаунт або мультисиг. Передамо їй адресу мінт-акаунту з попереднього кроку.

```bash
spl-token display <ACCOUNT_ADDRESS>
```

Тепер, коли у нас є мінт-акаунт, ми можемо його закрити за допомогою наступної команди, де `<TOKEN_MINT_ADDRESS>` — це адреса мінт-акаунту, отримана на попередньому кроці.

```bash
spl-token close-mint <TOKEN_MINT_ADDRESS> 
```

Примітка: Закриваючи акаунт, ми повертаємо орендні лампорти, які були на мінт-акаунті. Пам’ятайте, що загальна пропозиція токенів на мінті має бути нульовою.

Як виклик, повторіть цей процес, але перед закриттям мінт-акаунту випустіть трохи токенів, а потім спробуйте його закрити — подивіться, що станеться. (Спойлер: це не вдасться)

### 3. Створення токен-акаунту з незмінним власником

Давайте випробуємо ще одне розширення — цього разу розширення токен-акаунту. Спочатку створимо новий мінт без додаткових розширень, а потім створимо асоційований токен-акаунт із розширенням незмінного власника.

Спершу створимо новий звичайний мінт без додаткових розширень:

```bash
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb 
```

Ви повинні отримати щось схоже на це:
```bash
Creating token FXnaqGm42aQgz1zwjKrwfn4Jk6PJ8cvkkSc8ikMGt6EU under program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb

Address:  FXnaqGm42aQgz1zwjKrwfn4Jk6PJ8cvkkSc8ikMGt6EU
Decimals:  9

Signature: 3tX6FHvE24e8UHqSWbK5HRpBFxtCnDTRHASFZtipKkTzapgMGZEeNJ2zHAHSrSUs8L8wQGnLbvJiLrHuomyps39j
```

Збережіть отриману адресу мінт-акаунту, вона знадобиться нам на наступному кроці.

Тепер давайте здійснимо мінт одного токена на associated token account (ATA), який використовує розширення `immutable owner`. За замовчуванням усі ATA мають увімкнене розширення `immutable owner`. І всі токен-акаунти, створені через CLI, будуть ATA, тож розширення `immutable owner` буде активним.

Цьому розширенню потрібні такі аргументи:
- `create-account`: інструкція, яку ми хочемо виконати.
- `--program-id` (необов’язково): ID програми, яку хочемо використати. Необов’язковий, бо CLI сам визначить програму-власника мінт-акаунту.
- `--owner` (необов’язково): публічний ключ власника гаманця. За замовчуванням це поточний публічний ключ, який можна отримати командою `solana address`.
- `--fee-payer` (необов’язково): пара ключів гаманця, що сплачує комісію за транзакцію. За замовчуванням — поточна пара ключів, яку можна подивитись через `solana config get`.
- `<TOKEN_MINT_ADDRESS>`: це мінт-акаунт, отриманий командою `create-token`.

Виконайте таку команду, щоб створити associated token account з розширенням immutable owner:

```bash
spl-token create-account <TOKEN_MINT_ADDRESS>
```

Після виконання цієї команди ми побачимо вивід, схожий на наведений нижче.

```bash
Creating account F8iDrVskLGwYo53SdJnvBKTpN1C7hobgnPQMq6hLivUn

Signature: 5zX73E2aFVwcsvhCgBSF6AxWqydWYk3KJaTmeS4AY22FwCvgEvnodvJ7fzvBHZptqv3FMz6tbLFR5LbmiUHLUkne
```

Тепер ми можемо створити деякі токени за допомогою функції `mint`. Ось аргументи, які потрібно вказати:
- `mint`: Інструкція
- `<TOKEN_MINT_ADDRESS>`: Адреса мінт-акаунту, отримана на першому кроці
- `<TOKEN_AMOUNT>`: Кількість токенів для створення
- `<RECIPIENT_TOKEN_ACCOUNT_ADDRESS>` (необов’язково): Акаунт токена, який утримуватиме створені токени. За замовчуванням це пов’язаний акаунт токена (ATA) для поточної пари ключів і мінт-акаунту, тобто автоматично використає акаунт із попереднього кроку.

```bash
spl-token mint <TOKEN_MINT_ADDRESS> <TOKEN_AMOUNT>
```

Це призведе до приблизно такого результату:
```bash
Minting 1 tokens
  Token: FXnaqGm42aQgz1zwjKrwfn4Jk6PJ8cvkkSc8ikMGt6EU
  Recipient: 8r9VNjnLqjzrpgkcgCozgvCBDQwWWYUL7RKwatSWnd6B

Signature: 54yREwGCH8YfYXqEf6gRKGou681F8NkToAJZvJqM5qZETJokRkdTb8s8HVkKPeVMQQcc8gCZkq4Kxx3YbLtY9Frk
```

Ви можете скористатися командою `spl-token display`, щоб отримати інформацію про мінт-акаунт та токен-акаунт.



### 4. Створення непередаваного NFT ("soul-bound")

Наостанок, давайте створимо NFT, що не передається — іноді його називають 'soul-bound' NFT. Уявіть його як токен-нагороду, який належить виключно одній особі або акаунту. Для створення цього токена ми використаємо три розширення: вказівник на метадані (metadata pointer), метадані (metadata) і токени, що не передаються (non-transferable token).


З розширенням метаданих ми можемо включити метадані безпосередньо в мінт-акаунт, а розширення токенів, що не передаються, робить токен ексклюзивним для акаунту.

Команда приймає такі аргументи:
* `create-token`: Інструкція, яку ми хочемо виконати.
* `--program-id`: ID програми, яку ми хочемо використати.
* `--decimals`: NFT зазвичай позначаються цілим числом, тому мають 0 знаків після коми.
* `--enable-metadata`: маркер розширення метаданих (ініціалізує розширення метаданих і вказівника на метадані).
* `--enable-non-transferable`: маркер розширення токенів, що не передаються.

Виконайте наступну команду, щоб створити токен, ініціалізований з метаданими та розширеннями, що забороняють передавання.

```bash
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb --decimals 0 --enable-metadata --enable-non-transferable
```

Ми побачимо вивід, схожий на показаний нижче.

```bash
Creating token GVjznwtfPndL9RsBtAYDFT1H8vhQjx8ymAB1rbd17qPr under program TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb
To initialize metadata inside the mint, please run `spl-token initialize-metadata GVjznwtfPndL9RsBtAYDFT1H8vhQjx8ymAB1rbd17qPr <YOUR_TOKEN_NAME> <YOUR_TOKEN_SYMBOL> <YOUR_TOKEN_URI>`, and sign with the mint authority.

Address:  GVjznwtfPndL9RsBtAYDFT1H8vhQjx8ymAB1rbd17qPr
Decimals:  0

Signature: 5EQ95NPTXg5reg9Ybcw9LQRjiWFZvfb9WqJidxu6kKbcKGajp1U999ioToC1qC88KUS4kdUi6rZbibqjgJbzYses
```

Після створення мінт-акаунту з розширенням метаданих, потрібно ініціалізувати метадані, як зазначено у виводі вище. Ініціалізація метаданих приймає такі аргументи:
* Mint address: адреса мінт-акаунту, для якого потрібно ініціалізувати метадані
* `<YOUR_TOKEN_NAME>`: назва токена
* `<YOUR_TOKEN_SYMBOL>`: символ, за яким токен буде ідентифіковано
* `<YOUR_TOKEN_URI>`: URI токена
* `--update-authority` (необов’язково): адреса акаунту з повноваженнями на оновлення метаданих. За замовчуванням використовується поточний публічний ключ

Виконайте наступну команду, щоб ініціалізувати метадані:

```bash
spl-token initialize-metadata <TOKEN_MINT_ADDRESS> MyToken TOK http://my.tokn
```

Тепер давайте подивимося на метадані, виконавши нашу надійну команду `display`.

```bash
spl-token display <TOKEN_MINT_ADDRESS>
```

Далі оновимо метадані для цього мінт-акаунту. Ми будемо оновлювати назву нашого токена. Виконайте наступну команду:

```bash
spl-token update-metadata <TOKEN_MINT_ADDRESS> name MyAwesomeNFT
```

Тепер подивимося, як додати довільне поле до метаданих нашого мінт-акаунту. Ця команда приймає такі аргументи:

* Адреса мінт-акаунту: адреса мінт-акаунту, метадані якого потрібно оновити.
* Назва довільного поля: назва нового довільного поля.
* Значення довільного поля: значення, яке потрібно встановити для нового поля.

Виконайте наступну команду:

```bash
spl-token update-metadata <TOKEN_MINT_ADDRESS> new-field new-value
```

Також ми можемо видалити довільні поля з метаданих мінт-акаунту. Виконайте наступну команду:

```bash
spl-token update-metadata <TOKEN_MINT_ADDRESS> new-field --remove
```

Наостанок зробимо цей NFT дійсно таким, що не можна передати. Для цього ми мінтимо NFT на наш ATA, а потім видаляємо повноваження мінт-акаунту. Таким чином загальна кількість токенів буде дорівнювати одному.

```bash
spl-token create-account <TOKEN_MINT_ADDRESS>
spl-token mint <TOKEN_MINT_ADDRESS> 1
spl-token authorize <TOKEN_MINT_ADDRESS> mint --disable
```

Тепер ми успішно створили NFT, який виключно належить нашому ATA і не підлягає передачі.

Ось і все! Саме так можна використовувати Solana CLI разом із Token Extension Program для роботи з розширеннями. Ми докладніше розглянемо ці розширення в окремих уроках і подивимося, як їх застосовувати програмно.

# Завдання

Спробуйте різні комбінації розширень за допомогою CLI.

Підказка: перегляньте свої опції, викликавши команди з маркером `--help`:
```bash
spl-token --create-token --help
```
