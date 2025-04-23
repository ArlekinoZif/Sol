---
назва: Стійкі нонси (Durable Nonces)
завдання:
- Вміти пояснити різницю між стійкими транзакціями та звичайними транзакціями.  
- Створювати та надсилати стійкі транзакції.  
- Орієнтуватися у виняткових випадках, які можуть виникати при роботі зі стійкими транзакціями.
---

# Стислий виклад
- Стійкі транзакції не мають терміну дії, на відміну від звичайних транзакцій, які дійсні лише протягом 150 блоків (~80–90 секунд).  
- Після підписання стійкої транзакції її можна зберегти в базі даних, файлі або надіслати на інший пристрій для подальшого надсилання.  
- Стійка транзакція створюється з використанням нонс-акаунту. Нонс-акаунт зберігає уповноважений акаунт та значення нонса, яке замінює `recent_blockhash`, щоб зробити транзакцію стійкою.  
- Стійкі транзакції мають починатися з інструкції `advanceNonce`, і уповноважений акаунт на нонс має бути підписантом у транзакції.  
- Якщо транзакція завершиться невдало з будь-якої причини, крім `advanceNonce`, нонс все одно буде просунутий, навіть якщо всі інші інструкції буде скасовано.

# Огляд

**Стійкі нонси** — це спосіб обійти обмеження на термін дії звичайних транзакцій. Щоб краще це зрозуміти, спершу розглянемо основні принципи, на яких ґрунтуються звичайні транзакції.

У Solana транзакція складається з трьох основних частин:

1. **Інструкції**: Інструкції — це операції, які ви хочете виконати в блокчейні, наприклад, переказ токенів, створення акаунтів або виклик програми. Вони виконуються в заданому порядку.

2. **Підписи**: Підписи є доказом того, що транзакція була підписана необхідними учасниками/авторизованими особами. Наприклад, якщо ви переказуєте SOL зі свого гаманця на інший, вам потрібно підписати транзакцію, щоб мережа могла перевірити її дійсність.

3. **Останній блокхеш (Recent Blockhash):** Останній блокхеш є унікальним ідентифікатором транзакції. Його використовують для запобігання повторним атакам (replay attacks), коли зловмисник записує транзакцію та намагається повторно її надіслати. Завдяки блокхешу кожна транзакція є унікальною та може бути виконана лише один раз. Останній блокхеш дійсний лише протягом 150 блоків.

У стійких транзакціях перші два пункти залишаються незмінними. Стійкі транзакції можливі завдяки маніпуляціям із останнім блокхешем.

Давайте глибше розглянемо концепцію останнього блокхешу. Щоб краще його зрозуміти, розглянемо проблему, яку він вирішує — [подвійні витрати](https://solana.com/developers/guides/advanced/introduction-to-durable-nonces#double-spend).

Уявіть, що ви купуєте NFT на MagicEden або Tensor. Вам потрібно підписати транзакцію, яка дозволяє програмі маркетплейсу зняти деяку кількість SOL з вашого гаманця. Після підписання транзакції маркетплейс надішле її в мережу. Якщо маркетплейс відправить її знову, без перевірок, ваші кошти можуть списати двічі.

Це відомо як проблема подвійних витрат і вона є однією з основних проблем, які вирішують блокчейни, як-от Solana. Наївним рішенням може бути перевірка всіх транзакцій у минулому на наявність дублікатів підпису транзакції. Однак це практично неможливо, оскільки розмір реєстру Solana перевищує 80 ТБ. Тому для вирішення цієї проблеми Solana використовує останні blockhash.

Останній блокхеш — це SHA-256 хеш розміром 32 байти, обчислений з останнього [entry id](https://solana.com/docs/terminology#blockhash) дійсного блоку за останні 150 блоків. Оскільки цей блокхеш включається в транзакцію до її підписання, можна гарантувати, що підписант дійсно підписав транзакцію не раніше ніж 150 блоків тому. Перевіряти 150 останніх блоків значно ефективніше, ніж переглядати весь реєстр.

Коли транзакція надсилається, валідатори Solana виконують наступне:

1. Перевірте чи підпис транзакції був підтверджений протягом останніх 150 блоків. Якщо є дублікат підпису, транзакція не пройде через помилку.
2. Якщо підпис транзакції не знайдено, система перевіряє останній блокхеш, щоб з'ясувати, чи існує він серед останніх 150 блоків. Якщо його немає, повертається помилка "Blockhash not found" (Blockhash не знайдено). Якщо blockhash знайдено, транзакція переходить до етапу виконання перевірок.

Хоча це рішення чудово працює для більшості випадків, воно має деякі обмеження. Основна проблема полягає в тому, що транзакція має бути підписана й відправлена до мережі протягом 150 блоків або приблизно за ~80-90 секунд. Однак існують випадки, коли потрібно більше 90 секунд для подання транзакції.

[Деякі причини використання стійких нонсів за версією Solana](https://solana.com/developers/guides/advanced/introduction-to-durable-nonces#durable-nonce-applications):  
> 1. **Заплановані транзакції**: Одна з найочевидніших сфер застосування стійких нонсів — це можливість планувати транзакції. Користувачі можуть заздалегідь підписати транзакцію, а надіслати її пізніше, що дозволяє реалізовувати заплановані перекази, взаємодії з контрактами або навіть виконання заздалегідь визначених інвестиційних стратегій.  
> 2. **Гаманці з мультипідписом **: Стійкі нонси дуже корисні для гаманців з мультипідписом, коли одна сторона підписує транзакцію, а інші можуть підтвердити її пізніше. Це дозволяє створити, переглянути й виконати транзакцію в майбутньому без потреби довіряти іншим сторонам.  
> 3. **Програми, що вимагають майбутньої взаємодії**: Якщо програма в Solana вимагає взаємодії в майбутньому (наприклад, вестинг-контракт або заплановане виведення коштів), транзакцію можна підписати заздалегідь, використовуючи стійкий нонс. Це гарантує, що взаємодія з контрактом відбудеться в потрібний час без присутності автора транзакції.  
> 4. **Кросчейн взаємодії**: Коли потрібно взаємодіяти з іншою блокчейн-мережею, і потрібно дочекатися підтверджень, ви можете підписати транзакцію зі стійким нонсом, а потім виконати її після отримання необхідних підтверджень.  
> 5. **Децентралізовані платформи деривативів**: У децентралізованих платформах для торгівлі деривативами складні транзакції можуть вимагати виконання за певних умов. Завдяки стійким нонсами ці транзакції можна підписати наперед і виконати, коли спрацює відповідний тригер.

## Міркування  

Стійкі транзакції потребують уважного ставлення, і саме тому слід завжди ретельно перевіряти транзакції перед підписанням.  

Уявімо, що ви необачно підписали зловмисну стійку транзакцію. Ця транзакція передає 500 SOL зловмиснику та змінює уповноважений акаунт нонсу (акаунт, що може оновлювати значення nonce) на цього зловмисника. Припустимо, у вас ще немає такої суми, але в майбутньому вона з’явиться. Це підступно, оскільки зловмисник може дочекатися слушного моменту й активувати транзакцію, щойно ваш баланс перевищить 500 SOL. І ви вже не згадаєте, на що тоді натиснули. Така транзакція може залишатися неактивною днями, тижнями або навіть роками.

Цей приклад не має на меті викликати паніку, а лише нагадує про потенційні ризики. Саме тому варто зберігати у гарячих гаманцях лише ті кошти, які ви готові втратити, і не використовувати холодні гаманці для підписання транзакцій.  

##  Стійкі нонси — спосіб подолати короткий термін дії звичайної транзакції:

Стійкі нонси — це спосіб підписати транзакцію поза мережею та зберігати її до моменту, коли вона буде готова до надсилання в мережу. Це дає змогу створювати стійкі транзакції.

Стійкі нонси, які мають довжину 32 байти (зазвичай подаються у вигляді рядків у форматі base58), використовуються замість останнього блокхешу, щоб зробити кожну транзакцію унікальною (запобігаючи подвійним витратам), водночас усуваючи обмеження на строк дії для невиконаної транзакції.

Якщо нонси використовуються замість останнього блокхешу, першою інструкцією транзакції має бути інструкція `nonceAdvance`, яка змінює або просуває нонс. Це гарантує, що кожна транзакція, підписана з використанням нонсу як останнього блокхешу, буде унікальною.

Важливо зазначити, що стійкі нонси вимагають [унікальних механізмів у межах Solana](https://docs.solanalabs.com/implemented-proposals/durable-tx-nonces) для своєї роботи, тому до них застосовуються особливі правила, які зазвичай не діють. Ми розглянемо це детальніше у технічному аспекті.  

## Стійкі нонси в деталях

Стійкі транзакції відрізняються від звичайних транзакцій наступним чином:

1. Стійкі нонси замінюють звичайний блокхеш на нонс. Цей нонс зберігається в `nonce account` і використовується тільки в одній транзакції. Нонс є унікальним блокхешем.   
2. Кожна довговічна транзакція повинна починатися з `nonce advance instruction` (інструкції просування нонсу), яка змінює нонс в `nonce account`. Це гарантує, що нонс буде унікальним і не може бути використаний повторно в іншій транзакції.

Нонс-акаунт — це акаунт, який зберігає декілька значень:
1. значення нонсу: значення нонсу, яке буде використане в транзакції.
2. уповноважений акаунт: публічний ключ, який може змінювати значення нонсу.
3. калькулятор комісії: калькулятор комісії для транзакції.

Знову ж таки, кожна стійка транзакція повинна починатися з інструкції `nonce advance instruction`, і `authority` має бути підписантом.

Наостанок, існує спеціальне правило: якщо стійка транзакція зазнає невдачі через будь-яку інструкцію, окрім `nonce advance instruction`, нонс все одно буде змінено (просунуто вперед), тоді як решта транзакції буде відкочена назад. Ця поведінка є унікальною виключно для стійких нонсів.

## Операції з стійкими нонсами

Для роботи з стійкими нонсами в пакеті `@solana/web3.js` доступні кілька помічників та констант:
1. `SystemProgram.nonceInitialize`: Інструкція для створення нового нонс-акаунту.  
2. `SystemProgram.nonceAdvance`: Інструкція для зміни нонсу в нонс-акаунті.  
3. `SystemProgram.nonceWithdraw`: Інструкція для виведення коштів з нонс-акаунту. Щоб видалити нонс-акаунт, необхідно вивести всі кошти з нього.  
4. `SystemProgram.nonceAuthorize`: Інструкція для зміни права власності нонс-акаунту.  
5. `NONCE_ACCOUNT_LENGTH`: Константа, що представляє довжину даних нонс-акаунту.  
6. `NonceAccount`: Клас, який представляє нонс-акаунт. Він містить статичну функцію `fromAccountData`, що дозволяє отримати об'єкт нонс-акаунту на основі даних акаунту.

Розглянемо детально кожну з допоміжних функцій.

### `nonceInitialize`

Ця інструкція використовується для створення нового нонс-акаунту. Вона приймає два параметри:  
1. `noncePubkey`: публічний ключ нонс-акаунту.  
2. `authorizedPubkey`: публічний ключ власника нонс-акаунту.  

Ось приклад коду:  

```ts
// 1. Generate/get a keypair for the nonce account, and the authority.
const [nonceKeypair, nonceAuthority] = makeKeypairs(2); // from '@solana-developers/helpers'

const tx = new Transaction().add(
  // 2. Allocate the account and transfer funds to it (the least amount is 0.0015 SOL)
  SystemProgram.createAccount({
    fromPubkey: payer.publicKey,
    newAccountPubkey: nonceKeypair.publicKey,
    lamports: 0.0015 * LAMPORTS_PER_SOL,
    space: NONCE_ACCOUNT_LENGTH,
    programId: SystemProgram.programId,
  }),
  // 3. Initialize the nonce account using the `SystemProgram.nonceInitialize` instruction.
  SystemProgram.nonceInitialize({
    noncePubkey: nonceKeypair.publicKey,
    authorizedPubkey: nonceAuthority.publicKey,
  }),
);

// send the transaction
await sendAndConfirmTransaction(connection, tx, [payer, nonceKeypair]);
```

Системна програма подбає про встановлення значення нонсу всередині нонс-акаунту.


### `nonceAdvance`

Ця інструкція використовується для зміни значення нонсу в нонс-акаунті, вона приймає два параметри:

1. `noncePubkey`: публічний ключ нонс-акаунту.  
2. `authorizedPubkey`: публічний ключ власника нонс-акаунту.  

Ось приклад коду:  

```ts
const instruction = SystemProgram.nonceAdvance({
  authorizedPubkey: nonceAuthority.publicKey,
  noncePubkey: nonceKeypair.publicKey,
});
```

Ви побачите цю інструкцію, як першу інструкцію в будь-якій стійкій транзакції. Але це не означає, що її потрібно використовувати тільки як першу інструкцію стійкої транзакції. Ви завжди можете викликати цю функцію і вона автоматично анулює будь-яку стійку транзакцію, прив'язану до її попереднього значення нонсу.

### `nonceWithdraw`

Ця інструкція використовується для виведення коштів із нонс-акаунту. Вона приймає чотири параметри:  
1. `noncePubkey`: публічний ключ нонс-акаунту.  
2. `toPubkey`: публічний ключ акаунту, який отримає кошти.  
3. `lamports`: кількість лампортів, що будуть виведені.  
4. `ʼauthorizedPubkey`: публічний ключ акаунту з правом власності на нонс-акаунт.  

Ось приклад коду:  

```ts
const instruction = SystemProgram.nonceWithdraw({
  noncePubkey: nonceKeypair.publicKey,
  toPubkey: payer.publicKey,
  lamports: amount,
  authorizedPubkey: nonceAuthority.publicKey,
});
```

Цю інструкцію також можна використовувати для закриття акаунта з nonce, виводячи всі кошти з нього.


### `nonceAuthorize`

Ця інструкція використовується для зміни уповноваженого акаунту нонсу, вона приймає три параметри:
1. `noncePubkey`: публічний ключ нонс-акаунту.
2. `authorizedPubkey`: публічний ключ поточного уповноваженого акаунту нонсу.
3. `newAuthorizedPubkey`: публічний ключ нового уповноваженого акаунту нонсу.

Ось приклад коду для цього:  
```ts
const instruction = SystemProgram.nonceAuthorize({
  noncePubkey: nonceKeypair.publicKey,
  authorizedPubkey: nonceAuthority.publicKey,
  newAuthorizedPubkey: newAuthority.publicKey,
});
```

## Як використовувати стійкі нонси  

Тепер, коли ми розглянули нонс-акаунти і різні операції з ними, поговоримо про те, як їх використовувати.  

Ми обговоримо:  

1. Отримання нонс-акаунту.  
2. Використання нонсу у транзакції для створення стійкої транзакції.  
3. Відправка довготривалої транзакції.  

### Отримання нонс-акаунту

Ми можемо отримати нонс-акаунт, щоб дізнатися значення нонсу, прочитавши дані акаунту та здійснивши їх серіалізацію.

```ts
const nonceAccount = await connection.getAccountInfo(nonceKeypair.publicKey);

const nonce = NonceAccount.fromAccountData(nonceAccount.data); 
```

### Використання нонсу в транзакції для створення стійкої транзакції

Щоб створити повноцінну стійку транзакцію, нам потрібно:

1. Використати значення нонсу замість останнього блокхешу.  
2. Додати інструкцію `nonceAdvance` як першу інструкцію в транзакції.  
3. Підписати транзакцію уповноваженим акаунтом нонс-акаунту.  

Після створення та підписання транзакції ми можемо її серіалізувати та закодувати у формат base58, після чого зберегти цей рядок у сховище для подальшої відправки.

```ts
  // Assemble the durable transaction
  const durableTx = new Transaction();
  durableTx.feePayer = payer.publicKey;

  // use the nonceAccount's stored nonce as the recentBlockhash
  durableTx.recentBlockhash = nonceAccount.nonce;

  // make a nonce advance instruction
  durableTx.add(
    SystemProgram.nonceAdvance({
      authorizedPubkey: nonceAuthority.publicKey,
      noncePubkey: nonceKeypair.publicKey,
    }),
  );

  // Add any instructions you want to the transaction in this case we are just doing a transfer
  durableTx.add(
    SystemProgram.transfer({
      fromPubkey: payer.publicKey,
      toPubkey: recipient.publicKey,
      lamports: 0.1 * LAMPORTS_PER_SOL,
    }),
  );

  // sign the tx with the nonce authority's keypair
  durableTx.sign(payer, nonceAuthority);

  // once you have the signed tx, you can serialize it and store it in a database, or send it to another device.
  // You can submit it at a later point.
  const serializedTx = base58.encode(durableTx.serialize({ requireAllSignatures: false }));
```

### відправлення стійкої транзакції:  

Тепер, коли у нас є транзакція, закодована у форматі base58, ми можемо її декодувати та відправити.

```ts
const tx = base58.decode(serializedTx);
const sig = await sendAndConfirmRawTransaction(connection, tx as Buffer);
```

## Деякі важливі нетипові ситуації

Є кілька моментів, які потрібно враховувати при роботі з стійкими транзакціями:
1. Якщо транзакція не вдається через інструкцію, відмінну від інструкції `nonce advance`.
2. Якщо транзакція не вдається через інструкцію `nonce advance`.
   
### Якщо транзакція не вдається через інструкцію, відмінну від інструкції `nonce advance`

Зазвичай, коли транзакція не проходить, всі інструкції в транзакції повертаються до початкового стану. Але у випадку тривалої транзакції, якщо яка-небудь інструкція не вдається і це не інструкція `nonce advance`, сам nonce все одно буде оновлений, а всі інші інструкції будуть скасовані. Ця функція розроблена для безпеки, щоб гарантувати, що після невдалої транзакції її не вийде використати повторно.

Попередньо підписані, без терміну придатності, тривалі транзакції подібні до підписаних чеків. Вони можуть бути небезпечними в певних ситуаціях. Ця додаткова функція безпеки фактично "анулює" чек, якщо з ним неправильно поводитися.

### Якщо транзакція не вдається через інструкцію `nonce advance`

Якщо транзакція не виконується через інструкцію `nonce advance`, то вся транзакція буде скасована. Це означає, що нонс не буде оновлений.

# Лабораторна робота

У цій лабораторній роботі ми навчимося створювати стійку транзакцію. Ми зосередимося на тому, що можна і що не можна робити з нею. Окрім цього, розглянемо деякі крайні випадки та способи їх обробки.

## 0. Почнемо

Спершу склонуємо початковий код

```bash
git clone https://github.com/Unboxed-Software/solana-lab-durable-nonces
cd Solana-lab-durable-nonces
git checkout starter
npm install
```

У початковому коді ви знайдете файл у `test/index.ts` з тестовим шаблоном. Увесь наш код будемо писати саме тут.

Ми будемо використовувати локальний валідатор для цієї лабораторної роботи. Однак, якщо бажаєте, можете використовувати devnet. (Якщо виникають проблеми з отриманням токенів через ейрдроп на devnet, скористайтеся [Solana Faucet](https://faucet.solana.com/)).

Щоб запустити локальний валідатор, вам потрібно його встановити. Якщо він ще не встановлений, зверніться до [інструкції з встановлення Solana CLI](https://docs.solanalabs.com/cli/install). Після встановлення CLI у вас з'явиться доступ до `solana-test-validator`.

У окремому терміналі запустіть команду:
```bash
solana-test-validator
```

У файлі `test/index.ts` ви побачите п’ять тестів, які допоможуть нам краще зрозуміти роботу стійких нонс-значень.  

Ми детально розглянемо кожен із тестових випадків.

## 1. Створення нонс-акаунту

Перш ніж писати будь-які тести, створимо допоміжну функцію під назвою `createNonceAccount`. Цю функцію розмістимо перед блоком `describe`.  

Вона приймає такі параметри:
- `Connection`: з’єднання, яке буде використовуватись  
- `payer`: акаунт платника  
- `nonceKeypair`: пара ключів для нонс-акаунту  
- `authority`: уповноважений акаунт для нонсу

Вона буде виконувати наступне:
1. Збирати та відправляти транзакцію, яка:
   1. Визначить акаунт, що стане нонс-акаунтом.
   2. Ініціалізує нонс-акаунт за допомогою інструкції `SystemProgram.nonceInitialize`.
2. Отримає дані нонс-акаунту.
3. Серіалізує дані нонс-акаунту та поверне їх.

Вставте наступний код десь вище блоку `describe`.
```ts
async function createNonceAccount(
  connection: Connection,
  payer: Keypair,
  nonceKeypair: Keypair,
  authority: PublicKey,
){
  // 2. Assemble and submit a transaction that will:
  const tx = new Transaction().add(
    // 2.1. Allocate the account that will be the nonce account.
    SystemProgram.createAccount({
      fromPubkey: payer.publicKey,
      newAccountPubkey: nonceKeypair.publicKey,
      lamports: 0.0015 * LAMPORTS_PER_SOL,
      space: NONCE_ACCOUNT_LENGTH,
      programId: SystemProgram.programId,
    }),
    // 2.2. Initialize the nonce account using the `SystemProgram.nonceInitialize` instruction.
    SystemProgram.nonceInitialize({
      noncePubkey: nonceKeypair.publicKey,
      authorizedPubkey: authority,
    }),
  );

  const sig = await sendAndConfirmTransaction(connection, tx, [payer, nonceKeypair]);
  console.log(
    'Creating Nonce TX:',
    `https://explorer.solana.com/tx/${sig}?cluster=custom&customUrl=http%3A%2F%2Flocalhost%3A8899`,
  );

  // 3. Fetch the nonce account.
  const accountInfo = await connection.getAccountInfo(nonceKeypair.publicKey);
  // 4. Serialize the nonce account data and return it.
  return NonceAccount.fromAccountData(accountInfo!.data);
};
```

## 2. Тест: Створення та відправка стійкої транзакції

Щоб створити та надіслати стійку транзакцію, потрібно виконати такі кроки:

1. Створити стійку транзакцію:  
   1. Створити нонс-акаунт.  
   2. Створити нову транзакцію.  
   3. Встановити `recentBlockhash` як значення нонсу.  
   4. Додати інструкцію `nonceAdvance` як першу інструкцію в транзакції.  
   5. Додати інструкцію переказу (тут можна додати будь-яку інструкцію).  
   6. Підписати транзакцію усіма необхідними парами ключів, не забувши додати підпис уповноваженого акаунту нонсу.  
   7. Серіалізувати транзакцію та закодувати її.  
   8. Тепер у вас є стійка транзакція — ви можете зберегти її в базі даних, файлі або передати кудись ще.
2. Надіслати стійку транзакцію:  
   1. Декодувати серіалізовану транзакцію.  
   2. Надіслати її за допомогою функції `sendAndConfirmRawTransaction`.
      
Ми можемо зібрати все це разом у нашому першому тесті:
```ts
it('Creates a durable transaction and submits it', async () => {
  const payer = await initializeKeypair(connection, {
    airdropAmount: 3 * LAMPORTS_PER_SOL,
    minimumBalance: 1 * LAMPORTS_PER_SOL,
  });

  // 1. Create a Durable Transaction.
  const [nonceKeypair, recipient] = makeKeypairs(2);

  // 1.1 Create the nonce account.
  const nonceAccount = await createNonceAccount(connection, payer, nonceKeypair, payer.publicKey);

  // 1.2 Create a new Transaction.
  const durableTx = new Transaction();
  durableTx.feePayer = payer.publicKey;

  // 1.3 Set the recentBlockhash to be the nonce value.
  durableTx.recentBlockhash = nonceAccount.nonce;

  // 1.4 Add the `nonceAdvance` instruction as the first instruction in the transaction.
  durableTx.add(
    SystemProgram.nonceAdvance({
      authorizedPubkey: payer.publicKey,
      noncePubkey: nonceKeypair.publicKey,
    }),
  );

  // 1.5 Add the transfer instruction (you can add any instruction you want here).
  durableTx.add(
    SystemProgram.transfer({
      fromPubkey: payer.publicKey,
      toPubkey: recipient.publicKey,
      lamports: 0.1 * LAMPORTS_PER_SOL,
    }),
  );

  // 1.6 Sign the transaction with the keyPairs that need to sign it, and make sure to add the nonce authority as a signer as well.
  // In this particular example the nonce auth is the payer, and the only signer needed for our transfer instruction is the payer as well, so the payer here as a sign is sufficient.
  durableTx.sign(payer);

  // 1.7 Serialize the transaction and encode it.
  const serializedTx = base58.encode(durableTx.serialize({ requireAllSignatures: false }));
  // 1.8 At this point you have a durable transaction, you can store it in a database or a file or send it somewhere else, etc.
  // ----------------------------------------------------------------

  // 2. Submit the durable transaction.
  // 2.1 Decode the serialized transaction.
  const tx = base58.decode(serializedTx);

  // 2.2 Submit it using the `sendAndConfirmRawTransaction` function.
  const sig = await sendAndConfirmRawTransaction(connection, tx as Buffer, {
    skipPreflight: true,
  });

  console.log(
    'Transaction Signature:',
    `https://explorer.solana.com/tx/${sig}?cluster=custom&customUrl=http%3A%2F%2Flocalhost%3A8899`,
  );
});
```

## 3. Тест: Транзакція не виконується, якщо нонс було змінено

Оскільки ми використовуємо нонс замість останнього блокхешу, система перевірятиме, чи відповідає наданий нами нонс значенню в `nonce_account`. Крім того, у кожній транзакції потрібно додавати інструкцію `nonceAdvance` як першу. Це гарантує, що після успішного виконання транзакції нонс зміниться, і ніхто не зможе повторно її надіслати.

Ось що ми протестуємо:
1. Створимо стійку транзакцію, як у попередньому кроці.
2. Змусимо нонс змінитися (використаємо `nonceAdvance`).
3. Спробуємо надіслати транзакцію — вона має завершитися помилкою.
   
```ts
it('Fails if the nonce has advanced', async () => {
  const payer = await initializeKeypair(connection, {
    airdropAmount: 3 * LAMPORTS_PER_SOL,
    minimumBalance: 1 * LAMPORTS_PER_SOL,
  });

  const [nonceKeypair, nonceAuthority, recipient] = makeKeypairs(3);

  // 1. Create a Durable Transaction.
  const nonceAccount = await createNonceAccount(connection, payer, nonceKeypair, nonceAuthority.publicKey);

  const durableTx = new Transaction();
  durableTx.feePayer = payer.publicKey;

  // use the nonceAccount's stored nonce as the recentBlockhash
  durableTx.recentBlockhash = nonceAccount.nonce;

  // make a nonce advance instruction
  durableTx.add(
    SystemProgram.nonceAdvance({
      authorizedPubkey: nonceAuthority.publicKey,
      noncePubkey: nonceKeypair.publicKey,
    }),
  );

  durableTx.add(
    SystemProgram.transfer({
      fromPubkey: payer.publicKey,
      toPubkey: recipient.publicKey,
      lamports: 0.1 * LAMPORTS_PER_SOL,
    }),
  );

  // sign the tx with both the payer and nonce authority's keypair
  durableTx.sign(payer, nonceAuthority);

  // once you have the signed tx, you can serialize it and store it in a database, or send it to another device
  const serializedTx = base58.encode(durableTx.serialize({ requireAllSignatures: false }));

  // 2. Advance the nonce
  const nonceAdvanceSig = await sendAndConfirmTransaction(
    connection,
    new Transaction().add(
      SystemProgram.nonceAdvance({
        noncePubkey: nonceKeypair.publicKey,
        authorizedPubkey: nonceAuthority.publicKey,
      }),
    ),
    [payer, nonceAuthority],
  );

  console.log(
    'Nonce Advance Signature:',
    `https://explorer.solana.com/tx/${nonceAdvanceSig}?cluster=custom&customUrl=http%3A%2F%2Flocalhost%3A8899`,
  );

  const tx = base58.decode(serializedTx);

  // 3. Try to submit the transaction, and it should fail.
  await assert.rejects(sendAndConfirmRawTransaction(connection, tx as Buffer));
});
```

## Тест 4: Нонс-акаунт змінюється, навіть якщо транзакція не вдалася

Важливим нетиповим випадком, який варто врахувати, є те, що навіть якщо транзакція не вдалася з будь-якої причини, окрім інструкції зміни нонсу, сам нонс все одно зміниться. Ця функція розроблена з метою безпеки, щоб гарантувати, що після підписання транзакції користувачем, навіть якщо вона не виконується, ця стійка транзакція більше не може бути використана.

Наступний код демонструє цей випадок. Ми спробуємо створити стійку транзакцію для переведення 50 SOL від платника до одержувача. Однак у платника недостатньо SOL для виконання переказу, тому транзакція не вдасться, але нонс все одно зміниться.

```ts
it('Advances the nonce account even if the transaction fails', async () => {
  const TRANSFER_AMOUNT = 50;
  const payer = await initializeKeypair(connection, {
    airdropAmount: 3 * LAMPORTS_PER_SOL,
    minimumBalance: 1 * LAMPORTS_PER_SOL,
  });

  const [nonceKeypair, nonceAuthority, recipient] = makeKeypairs(3);

  // Create the nonce account
  const nonceAccount = await createNonceAccount(connection, payer, nonceKeypair, nonceAuthority.publicKey);
  const nonceBeforeAdvancing = nonceAccount.nonce;

  console.log('Nonce Before Advancing:', nonceBeforeAdvancing);

  // Assemble a durable transaction that will fail

  const balance = await connection.getBalance(payer.publicKey);

  // making sure that we don't have 50 SOL in the account
  assert(
    balance < TRANSFER_AMOUNT * LAMPORTS_PER_SOL,
    `Too much balance, try to change the transfer amount constant 'TRANSFER_AMOUNT' at the top of the function to be more than ${ balance / LAMPORTS_PER_SOL }`,
  );

  const durableTx = new Transaction();
  durableTx.feePayer = payer.publicKey;

  // use the nonceAccount's stored nonce as the recentBlockhash
  durableTx.recentBlockhash = nonceAccount.nonce;

  // make a nonce advance instruction
  durableTx.add(
    SystemProgram.nonceAdvance({
      authorizedPubkey: nonceAuthority.publicKey,
      noncePubkey: nonceKeypair.publicKey,
    }),
  );

  // Transfer 50 sols instruction
  // This will fail because the account doesn't have enough balance
  durableTx.add(
    SystemProgram.transfer({
      fromPubkey: payer.publicKey,
      toPubkey: recipient.publicKey,
      lamports: TRANSFER_AMOUNT * LAMPORTS_PER_SOL,
    }),
  );

  // sign the tx with both the payer and nonce authority's keypair
  durableTx.sign(payer, nonceAuthority);

  // once you have the signed tx, you can serialize it and store it in a database, or send it to another device
  const serializedTx = base58.encode(durableTx.serialize({ requireAllSignatures: false }));

  const tx = base58.decode(serializedTx);

  // assert the promise to throw an error
  await assert.rejects(
    sendAndConfirmRawTransaction(connection, tx as Buffer, {
      // If we don't skip preflight this transaction will never reach the network, and the library will reject it and throw an error, therefore it will fail but the nonce will not advance
      skipPreflight: true,
    }),
  );

  const nonceAccountAfterAdvancing = await connection.getAccountInfo(nonceKeypair.publicKey);
  const nonceAfterAdvancing = NonceAccount.fromAccountData(nonceAccountAfterAdvancing!.data).nonce;

  // We can see that even though the transitions failed, the nonce has advanced
  assert.notEqual(nonceBeforeAdvancing, nonceAfterAdvancing);
});
```

Зверніть увагу, що ми встановлюємо параметр `skipPreflight: true` у функції `sendAndConfirmRawTransaction`. Цей крок дуже важливий, оскільки без нього транзакція ніколи не досягне мережі. Натомість бібліотека відхилить її та викине помилку, що призведе до того, що нонс не зміниться.

Однак це не вся історія. У наступному тесті ми побачимо сценарій, коли навіть якщо транзакція не вдалася, нонс не зміниться.

## 5. Тест: Нонс-акаунт не зміниться, якщо транзакція не вдалася через інструкцію зміни нонсу

Для того, щоб nonce змінився, інструкція `nonceAdvance` повинна бути виконана успішно. Тому якщо транзакція не вдалася з будь-якої причини, пов'язаної з цією інструкцією, nonce не зміниться.

Правильно сформульована інструкція `nonceAdvance` може не спрацювати тільки в тому випадку, якщо уповноважений на зміну nonce не підписав транзакцію.

Давайте подивимося на це в дії.

```ts
it('The nonce account will not advance if the transaction fails because the nonce auth did not sign the transaction', async () => {
  const payer = await initializeKeypair(connection, {
    airdropAmount: 3 * LAMPORTS_PER_SOL,
    minimumBalance: 1 * LAMPORTS_PER_SOL,
  });

  const [nonceKeypair, nonceAuthority, recipient] = makeKeypairs(3);

  // Create the nonce account
  const nonceAccount = await createNonceAccount(connection, payer, nonceKeypair, nonceAuthority.publicKey);
  const nonceBeforeAdvancing = nonceAccount.nonce;

  console.log('Nonce before submitting:', nonceBeforeAdvancing);

  // Assemble a durable transaction that will fail

  const durableTx = new Transaction();
  durableTx.feePayer = payer.publicKey;

  // use the nonceAccount's stored nonce as the recentBlockhash
  durableTx.recentBlockhash = nonceAccount.nonce;

  // make a nonce advance instruction
  durableTx.add(
    SystemProgram.nonceAdvance({
      authorizedPubkey: nonceAuthority.publicKey,
      noncePubkey: nonceKeypair.publicKey,
    }),
  );

  durableTx.add(
    SystemProgram.transfer({
      fromPubkey: payer.publicKey,
      toPubkey: recipient.publicKey,
      lamports: 0.1 * LAMPORTS_PER_SOL,
    }),
  );

  // sign the tx with the payer keypair
  durableTx.sign(payer);

  // once you have the signed tx, you can serialize it and store it in a database, or send it to another device
  const serializedTx = base58.encode(durableTx.serialize({ requireAllSignatures: false }));

  const tx = base58.decode(serializedTx);

  // assert the promise to throw an error
  await assert.rejects(
    sendAndConfirmRawTransaction(connection, tx as Buffer, {
      skipPreflight: true,
    }),
  );

  const nonceAccountAfterAdvancing = await connection.getAccountInfo(nonceKeypair.publicKey);
  const nonceAfterAdvancing = NonceAccount.fromAccountData(nonceAccountAfterAdvancing!.data).nonce;

  // We can see that the nonce did not advance, because the error was in the nonce advance instruction
  assert.equal(nonceBeforeAdvancing, nonceAfterAdvancing);
});
```

## 6. Тест: Підписати транзакцію, а потім змінити уповноваження для nonce

Останній тест, який ми розглянемо, полягає в створенні тривалої транзакції. Спробуйте надіслати її з неправильним уповноваженням nonce (це призведе до помилки). Потім змініть уповноваженя nonce і надішліть транзакцію з правильним — вона буде успішною.

```ts
it('Submits after changing the nonce auth to an already signed address', async () => {
  const payer = await initializeKeypair(connection, {
    airdropAmount: 3 * LAMPORTS_PER_SOL,
    minimumBalance: 1 * LAMPORTS_PER_SOL,
  });

  const [nonceKeypair, nonceAuthority, recipient] = makeKeypairs(3);

  // Create the nonce account
  const nonceAccount = await createNonceAccount(connection, payer, nonceKeypair, nonceAuthority.publicKey);
  const nonceBeforeAdvancing = nonceAccount.nonce;

  console.log('Nonce before submitting:', nonceBeforeAdvancing);

  // Assemble a durable transaction that will fail

  const durableTx = new Transaction();
  durableTx.feePayer = payer.publicKey;

  // use the nonceAccount's stored nonce as the recentBlockhash
  durableTx.recentBlockhash = nonceAccount.nonce;

  // make a nonce advance instruction
  durableTx.add(
    SystemProgram.nonceAdvance({
      // The nonce auth is not the payer at this point in time, so the transaction will fail
      // But in the future we can change the nonce auth to be the payer and submit the transaction whenever we want
      authorizedPubkey: payer.publicKey,
      noncePubkey: nonceKeypair.publicKey,
    }),
  );

  durableTx.add(
    SystemProgram.transfer({
      fromPubkey: payer.publicKey,
      toPubkey: recipient.publicKey,
      lamports: 0.1 * LAMPORTS_PER_SOL,
    }),
  );

  // sign the tx with the payer keypair
  durableTx.sign(payer);

  // once you have the signed tx, you can serialize it and store it in a database, or send it to another device
  const serializedTx = base58.encode(durableTx.serialize({ requireAllSignatures: false }));

  const tx = base58.decode(serializedTx);

  // assert the promise to throw an error
  // It will fail because the nonce auth is not the payer
  await assert.rejects(
    sendAndConfirmRawTransaction(connection, tx as Buffer, {
      skipPreflight: true,
    }),
  );

  const nonceAccountAfterAdvancing = await connection.getAccountInfo(nonceKeypair.publicKey);
  const nonceAfterAdvancing = NonceAccount.fromAccountData(nonceAccountAfterAdvancing!.data).nonce;

  // We can see that the nonce did not advance, because the error was in the nonce advance instruction
  assert.equal(nonceBeforeAdvancing, nonceAfterAdvancing);

  // Now we can change the nonce auth to be the payer
  const nonceAuthSig = await sendAndConfirmTransaction(
    connection,
    new Transaction().add(
      SystemProgram.nonceAuthorize({
        noncePubkey: nonceKeypair.publicKey,
        authorizedPubkey: nonceAuthority.publicKey,
        newAuthorizedPubkey: payer.publicKey,
      }),
    ),
    [payer, nonceAuthority],
  );

  console.log(
    'Nonce Auth Signature:',
    `https://explorer.solana.com/tx/${nonceAuthSig}?cluster=custom&customUrl=http%3A%2F%2Flocalhost%3A8899`,
  );

  // At any time in the future we can submit the transaction and it will go through
  const txSig = await sendAndConfirmRawTransaction(connection, tx as Buffer, {
    skipPreflight: true,
  });

  console.log(
    'Transaction Signature:',
    `https://explorer.solana.com/tx/${txSig}?cluster=custom&customUrl=http%3A%2F%2Flocalhost%3A8899`,
  );
});
```

## 8. Запустіть тести

Нарешті, давайте запустимо тести:
```bash
npm start
```

Переконайтесь, що всі тести проходять успішно.

Вітаємо! Тепер ви знаєте, як працюють тривалі (durable) nonce.

# Завдання

Напишіть програму, яка створює тривалу транзакцію та зберігає її у файл, а потім створіть окрему програму, яка читає файл з цією тривалою транзакцією та відправляє її в мережу.
