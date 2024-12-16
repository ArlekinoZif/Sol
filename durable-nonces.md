---
назва: Номери Тривалого Використання (Durable Nonces)
цілі:
- Розуміти різницю між тривалими транзакціями та звичайними транзакціями.
- Створювати та надсилати тривалі транзакції.
- Вміти розв'язувати нетипов проблеми, які можуть виникнути при роботі з тривалими транзакціями.
---

# Стислий огляд
- Тривалі транзакції не мають дати завершення, на відміну від звичайних транзакцій, які мають термін дії 150 блоків (~80-90 секунд).  
- Після підписання тривалої транзакції її можна зберегти в базі даних або файлі, чи надіслати на інший пристрій для подальшого виконання.  
- Тривала транзакція створюється за допомогою nonce-акаунта. Nonce-акаунт зберігає авторизацію і значення nonce, яке замінює останній blockhash для створення тривалої транзакції.  
- Тривалі транзакції повинні починатися з інструкції `advanceNonce`, і nonce authority повинна бути підписантом у транзакції.  
- Якщо транзакція завершується невдало з будь-якої причини, крім виконання інструкції advanceNonce, nonce все одно буде оновлено, хоча всі інші інструкції будуть скасовані.

# Огляд

Durable Nonce — це спосіб обійти термін дії звичайних транзакцій. Щоб краще зрозуміти це, почнемо з основних понять, пов'язаних зі звичайними транзакціями.

У Solana транзакція складається з трьох основних частин:

1. **Інструкції**: Інструкції — це операції, які ви хочете виконати в блокчейні, наприклад, переказ токенів, створення акаунтів або виклик програми. Вони виконуються в заданому порядку.

2. **Підписи**: Підписи є доказом того, що транзакція була підписана необхідними учасниками/авторизованими особами. Наприклад, якщо ви переказуєте SOL зі свого гаманця на інший, вам потрібно підписати транзакцію, щоб мережа могла перевірити її дійсність.

3. **Останній Blockhash**: Останній blockhash — це унікальний ідентифікатор кожної транзакції. Він використовується для запобігання повторним атакам (replay attacks), коли зловмисник записує транзакцію та намагається надіслати її знову. Останній blockhash гарантує, що кожна транзакція унікальна та може бути надіслана лише один раз. Останній blockhash дійсний лише протягом 150 блоків.

У тривалих транзакціях перші два пункти залишаються незмінними. Тривалі транзакції можливі завдяки маніпуляціям із останнім blockhash.

Давайте глибше розглянемо концепцію останнього blockhash. Щоб краще його зрозуміти, розглянемо проблему, яку він вирішує — [подвійні витрати](https://solana.com/developers/guides/advanced/introduction-to-durable-nonces#double-spend).

Уявіть, що ви купуєте NFT на MagicEden або Tensor. Вам потрібно підписати транзакцію, яка дозволяє програмі маркетплейсу зняти деяку кількість SOL з вашого гаманця. Після підписання транзакції маркетплейс надішле її в мережу. Якщо маркетплейс відправить її знову, без перевірок, ваші кошти можуть списати двічі.

Це відомо як проблема подвійних витрат і вона є однією з основних проблем, які вирішують блокчейни, як-от Solana. Наївним рішенням може бути перевірка всіх транзакцій у минулому на наявність дублікатів підпису транзакції. Однак це практично неможливо, оскільки розмір реєстру Solana перевищує 80 ТБ. Тому для вирішення цієї проблеми Solana використовує останні blockhash.

Останній blockhash — це 32-байтовий SHA-256-хеш останнього [entry id](https://solana.com/docs/terminology#blockhash) валідного блоку за останні 150 блоків. Оскільки цей blockhash є частиною транзакції до її підписання, ми можемо гарантувати, що підписувач підписав її протягом останніх 150 блоків. Перевірка 150 блоків набагато реальніша, ніж перевірка всього реєстру.

Коли транзакція надсилається, валідатори Solana виконують наступне:

1. Перевірте чи підпис транзакції був підтверджений протягом останніх 150 блоків. Якщо є дублікат підпису, транзакція не пройде через помилку.

2. Якщо підпис транзакції не знайдено, система перевіряє останній blockhash, щоб з'ясувати, чи існує він серед останніх 150 блоків. Якщо його немає, повертається помилка "Blockhash not found" (Blockhash не знайдено). Якщо blockhash знайдено, транзакція переходить до етапу виконання перевірок.

Хоча це рішення чудово працює для більшості випадків, воно має деякі обмеження. Основна проблема полягає в тому, що транзакція має бути підписана й відправлена до мережі протягом 150 блоків або приблизно за ~80-90 секунд. Однак існують випадки, коли потрібно більше 90 секунд для подання транзакції.

[Деякі причини використання Durable Nonces від Solana](https://solana.com/developers/guides/advanced/introduction-to-durable-nonces#durable-nonce-applications):  

> 1. **Заплановані транзакції**: Durable Nonces дозволяють створювати заплановані транзакції. Користувачі можуть попередньо підписати транзакцію і відправити її пізніше, що дозволяє здійснювати заплановані перекази, взаємодії з контрактами або навіть виконання заздалегідь визначених інвестиційних стратегій.  
> 2. **Мультипідписні гаманці**: Durable Nonces особливо корисні для гаманців з мультипідписом, де одна сторона підписує транзакцію, а інші підтверджують її пізніше. Це забезпечує можливість пропонувати, перевіряти та виконувати транзакції без необхідності довіряти одній стороні.  
> 3. **Програми, що вимагають майбутньої взаємодії**: Якщо програма на Solana вимагає взаємодії в майбутньому (наприклад, договір на вестинг або відкладений випуск коштів), транзакцію можна попередньо підписати за допомогою Durable Nonce. Це гарантує, що взаємодія з контрактом відбудеться в правильний час, без необхідності присутності автора транзакції.  
> 4. **Міжланцюгові взаємодії**: Коли необхідно взаємодіяти з іншою блокчейн-мережею, і це вимагає підтвердження, транзакцію можна підписати з Durable Nonce і виконати після отримання потрібних підтверджень.  
> 5. **Децентралізовані платформи деривативів**: У децентралізованих платформах деривативів складні транзакції можуть вимагати виконання на основі певних тригерів. Завдяки Durable Nonces ці транзакції можуть бути попередньо підписані та виконані, коли тригерна умова буде виконана.  

## Міркування  

Тривалі транзакції потребують уважного ставлення, і саме тому слід завжди ретельно перевіряти транзакції перед підписанням.  

Уявіть, що ви випадково підписали шкідливу тривалу транзакцію. Ця транзакція передає 500 SOL зловмиснику та змінює nonce уповноваження на його користь. Припустимо, що на даний момент у вас немає такої суми, але в майбутньому вона з'явиться. Це може бути вкрай небезпечним, адже зловмисник чекатиме, поки ваш баланс перевищить 500 SOL, щоб реалізувати цю транзакцію. Ви навіть можете не пам’ятати, на що погоджувалися. Вона може залишатися неактивною днями, тижнями чи навіть роками.  

Цей приклад не має на меті викликати паніку, а лише нагадує про потенційні ризики. Саме тому варто зберігати у гарячих гаманцях лише ті кошти, які ви готові втратити, і не використовувати холодні гаманці для підписання транзакцій.  

##   Щоб подолати короткий термін дії звичайної транзакції:  

**Durable nonces** дозволяють підписувати транзакції поза мережею (off-chain) та зберігати їх до моменту, коли вони будуть готові до відправлення в мережу. Це дає змогу створювати **довговічні транзакції** (durable transactions).  

Durable nonces — це 32-байтні значення (зазвичай представлені у вигляді рядків, закодованих у форматі base58), які використовуються замість recent blockhash, щоб зробити кожну транзакцію унікальною (для запобігання подвійним витратам) і водночас усунути часові обмеження для невиконаних транзакцій.  

Якщо замість recent blockhash використовується nonce, першою інструкцією транзакції має бути інструкція `nonceAdvance`, яка змінює або оновлює nonce. Це гарантує, що кожна транзакція, підписана з використанням nonce як recent blockhash, буде унікальною.  

Важливо зазначити, що **durable nonces** вимагають [унікальних механізмів у межах Solana](https://docs.solanalabs.com/implemented-proposals/durable-tx-nonces) для своєї роботи, тому до них застосовуються особливі правила, які зазвичай не діють. Ми розглянемо це детальніше у технічному аспекті.  

## Довговічні нонси в деталях

Довговічні транзакції відрізняються від звичайних транзакцій наступним чином:

1. **Довговічні нонси** замінюють звичайний блокхеш на нонс. Цей нонс зберігається в **`nonce account`** і використовується тільки в одній транзакції. Нонс є унікальним блокхешем.   
2. Кожна довговічна транзакція повинна починатися з **`nonce advance instruction`** (інструкції просування нонсу), яка змінює нонс в **`nonce account`**. Це гарантує, що нонс буде унікальним і не може бути використаний повторно в іншій транзакції.

Ці правила забезпечують стабільність та безпеку довговічних транзакцій, дозволяючи їм залишатися чинними до того часу, поки вони не будуть готові до подачі в мережу.

Nonce account — це акаунт, який містить кілька значень:
1. **nonce value**: значення нонсу, яке буде використано в транзакції.
2. **authority**: публічний ключ, який має право змінювати значення нонсу.
3. **fee calculator**: калькулятор комісії для транзакції.

Знову ж таки, кожна довговічна транзакція повинна починатися з інструкції `nonce advance instruction`, і `authority` має бути підписантом (signer).

Наостанок, існує спеціальне правило: якщо довговічна транзакція зазнає невдачі через будь-яку інструкцію, окрім `nonce advance instruction`, нонс все одно буде змінено (просунуто вперед), тоді як решта транзакції буде відкочена назад. Ця поведінка є унікальною виключно для довговічних нонсів.

## Операції з довговічними нонсами

Для роботи з довговічними нонсами в пакеті `@solana/web3.js` доступні кілька помічників та констант:

1. **`SystemProgram.nonceInitialize`**: Інструкція для створення нового нонс-акаунту.  
2. **`SystemProgram.nonceAdvance`**: Інструкція для зміни нонсу в нонс-акаунті.  
3. **`SystemProgram.nonceWithdraw`**: Інструкція для виведення коштів з нонс-акаунту. Щоб видалити акаунт нонсу, необхідно вивести всі кошти з нього.  
4. **`SystemProgram.nonceAuthorize`**: Інструкція для зміни права власності нонс-акаунту.  
5. **`NONCE_ACCOUNT_LENGTH`**: Константа, що представляє довжину даних нонс-акаунту.  
6. **`NonceAccount`**: Клас, який представляє нонс-акаунт. Він містить статичну функцію `fromAccountData`, що дозволяє отримати об'єкт нонс-акаунту на основі даних акаунта.

Розглянемо детально кожну з допоміжних функцій.

### `nonceInitialize`

Ця інструкція використовується для створення нового нонс-акаунту. Вона приймає два параметри:  
1. **`noncePubkey`**: публічний ключ нонс-акаунту.  
2. **`authorizedPubkey`**: публічний ключ власника нонс-акаунту.  

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

Програма автоматично встановить значення nonce всередині нонс-акаунту.

### `nonceAdvance`

Ця інструкція використовується для зміни значення nonce в нонс-акаунті. Вона приймає два параметри:  

1. **`noncePubkey`**: публічний ключ нонс-акаунту.  
2. **`authorizedPubkey`**: публічний ключ власника нонс-акаунту.  

Ось приклад коду:  

```ts
const instruction = SystemProgram.nonceAdvance({
  authorizedPubkey: nonceAuthority.publicKey,
  noncePubkey: nonceKeypair.publicKey,
});
```

Цю інструкцію можна побачити як першу інструкцію в будь-якій durable-транзакції. Але це не означає, що її потрібно використовувати виключно як першу інструкцію durable-транзакції. Ви завжди можете викликати цю функцію, і вона автоматично зробить недійсними всі durable-транзакції, прив’язані до попереднього значення nonce.

### `nonceWithdraw`

Ця інструкція використовується для виведення коштів із нонс-акаунту. Вона приймає чотири параметри:  

1. **`noncePubkey`**: публічний ключ нонс-акаунту.  
2. **`toPubkey`**: публічний ключ акаунту, який отримає кошти.  
3. **`lamports`**: кількість лампортів, що будуть виведені.  
4. **`authorizedPubkey`**: публічний ключ акаунту з правом власності на нонс-акаунт.  

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

Ця інструкція використовується для зміни авторитету акаунта з nonce. Вона приймає три параметри:  

1. **`noncePubkey`**: публічний ключ акаунта з nonce.  
2. **`authorizedPubkey`**: публічний ключ поточного авторитету акаунта з nonce.  
3. **`newAuthorizedPubkey`**: публічний ключ нового авторитету акаунта з nonce.  

Ось приклад коду для цього:  
```ts
const instruction = SystemProgram.nonceAuthorize({
  noncePubkey: nonceKeypair.publicKey,
  authorizedPubkey: nonceAuthority.publicKey,
  newAuthorizedPubkey: newAuthority.publicKey,
});
```

## Як використовувати durable nonces  

Тепер, коли ми розглянули нонс-акаунти і різні операції з ним, поговоримо про те, як їх використовувати.  

Ми обговоримо:  

1. Отримання нонс-акаунту.  
2. Використання nonce у транзакції для створення довготривалої транзакції.  
3. Відправка довготривалої транзакції.  

### Отримання нонс-акаунту  

Ми можемо отримати дані нонс-акаунту, щоб дізнатися поточне значення nonce, запросивши інформацію про акаунт і перетворивши її у зручний для роботи формат:  

```ts
const nonceAccount = await connection.getAccountInfo(nonceKeypair.publicKey);

const nonce = NonceAccount.fromAccountData(nonceAccount.data); 
```

### Використання nonce у транзакції для створення довговічної транзакції  

Щоб створити повноцінну довговічну транзакцію, необхідно виконати такі дії:  

1. Використати значення nonce замість останнього blockhash.  
2. Додати інструкцію `nonceAdvance` як першу інструкцію в транзакції.  
3. Підписати транзакцію уповноваженим ключем нонс-акаунту.  

Після створення та підписання транзакції її можна серіалізувати й закодувати у формат Base58-строки. Цей рядок можна зберегти у сховищі для подальшого надсилання.
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

### відправлення довготривалої транзакції:  

Тепер, коли у нас є транзакція, закодована у форматі base58, ми можемо її декодувати та відправити.

```ts
const tx = base58.decode(serializedTx);
const sig = await sendAndConfirmRawTransaction(connection, tx as Buffer);
```

## Деякі важливі нетипові ситуації

Є кілька моментів, які потрібно враховувати при роботі з довготривалими транзакціями:

1. Якщо транзакція не вдається через інструкцію, відмінну від інструкції `nonce advance`.
2. Якщо транзакція не вдається через інструкцію `nonce advance`.
   
### Якщо транзакція не вдається через інструкцію, відмінну від інструкції `nonce advance`

Зазвичай, коли транзакція не проходить, всі інструкції в транзакції повертаються до початкового стану. Але у випадку тривалої транзакції, якщо яка-небудь інструкція не вдається і це не інструкція `nonce advance`, сам nonce все одно буде оновлений, а всі інші інструкції будуть скасовані. Ця функція розроблена для безпеки, щоб гарантувати, що після невдалої транзакції її не вийде використати повторно.

Попередньо підписані, без терміну придатності, тривалі транзакції подібні до підписаних чеків. Вони можуть бути небезпечними в певних ситуаціях. Ця додаткова функція безпеки фактично "анулює" чек, якщо з ним неправильно поводитися.

### Якщо транзакція не вдається через інструкцію `nonce advance`

Якщо транзакція не виконується через інструкцію `nonce advance`, то вся транзакція буде скасована. Це означає, що nonce не буде оновлений.

# Лабораторна робота

У цій лабораторній роботі ми навчимося створювати довготривалу транзакцію. Ми зосередимося на тому, що можна і що не можна робити з нею. Окрім цього, розглянемо деякі крайні випадки та способи їх обробки.

## 0. Почнемо

Спершу склонуємо початковий код

```bash
git clone https://github.com/Unboxed-Software/solana-lab-durable-nonces
cd Solana-lab-durable-nonces
git checkout starter
npm install
```

У початковому коді ви знайдете файл у `test/index.ts` з тестовим шаблоном. Увесь наш код будемо писати саме тут.

Ми будемо використовувати локального валідатора для цієї лабораторної роботи. Однак, якщо бажаєте, можете використовувати devnet. (Якщо виникають проблеми з отриманням токенів через airdrop на devnet, скористайтеся [Solana Faucet](https://faucet.solana.com/)).

Щоб запустити локального валідатора, вам потрібно його встановити. Якщо він ще не встановлений, зверніться до [інструкції з встановлення Solana CLI](https://docs.solanalabs.com/cli/install). Після встановлення CLI у вас з'явиться доступ до `solana-test-validator`.

У окремому терміналі запустіть команду:

```bash
solana-test-validator
```

У файлі `test/index.ts` ви побачите п’ять тестів, які допоможуть нам краще зрозуміти роботу тривалих nonce-значень.  

Ми детально розглянемо кожен із тестових випадків.

## 1. Створення nonce-акаунту

Перш ніж писати будь-які тести, створимо допоміжну функцію під назвою `createNonceAccount`. Цю функцію розмістимо перед блоком `describe`.  

Функція прийматиме такі параметри:
- `Connection`: з’єднання, яке буде використовуватися.
- `payer`: платник.
- `nonceKeypair`: пара ключів nonce.
- `authority`: обліковий запис, який матиме право керування nonce.

Ця функція допоможе нам створити nonce-акаунт для подальшого використання в тестах.

Вона буде виконувати наступне:

1. Збирати та відправляти транзакцію, яка:
   1. Визначить акаунт, що стане nonce-акаунтом.
   2. Ініціалізує nonce-акаунт за допомогою інструкції `SystemProgram.nonceInitialize`.
2. Отримає дані nonce-акаунту.
3. Серіалізує дані nonce-акаунту та поверне їх.

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

## 2. Тест: Створення та відправка тривалої транзакції

Щоб створити та відправити тривалу транзакцію, потрібно виконати такі кроки:

1. **Створення тривалої транзакції:**
   1. Створити нонс-акаунту.
   2. Створити нову транзакцію.
   3. Встановити `recentBlockhash` як значення nonce.
   4. Додати інструкцію `nonceAdvance` як першу інструкцію в транзакції.
   5. Додати інструкцію переказу (можна додати будь-яку інструкцію на ваш розсуд).
   6. Підпишіть транзакцію за допомогою пар ключів, які повинні її підписати, і не забудьте також додати уповноважений нонс як підписанта.
   7. Серіалізувати транзакцію та закодувати її.
   8. На цьому етапі у вас є тривала транзакція, яку можна зберегти у базі даних, файлі або відправити в інше місце.

2. **Відправка тривалої транзакції:**
   1. Декодувати серіалізовану транзакцію.
   2. Відправити її за допомогою функції `sendAndConfirmRawTransaction`.
      
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

## Тест 3: Транзакція не виконується, якщо nonce було змінено

Оскільки ми використовуємо nonce замість останнього blockhash, система перевірятиме, чи відповідає наданий нами nonce значенню в `nonce_account`. Крім того, у кожній транзакції потрібно додавати інструкцію `nonceAdvance` як першу. Це гарантує, що після успішного виконання транзакції nonce зміниться, і ніхто не зможе повторно її надіслати.

Ось що ми протестуємо:
1. Створимо durable-транзакцію, як у попередньому кроці.
2. Змусимо nonce змінитися (використаємо `nonceAdvance`).
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

Важливим нетиповим випадком, який варто врахувати, є те, що навіть якщо транзакція не вдалася з будь-якої причини, окрім інструкції зміни nonce, сам nonce все одно зміниться. Ця функція розроблена з метою безпеки, щоб гарантувати, що після підписання транзакції користувачем, навіть якщо вона не виконується, ця durable транзакція більше не може бути використана.

Наступний код демонструє цей випадок. Ми спробуємо створити durable транзакцію для переведення 50 SOL від платника до одержувача. Однак у платника недостатньо SOL для виконання переказу, тому транзакція не вдасться, але nonce все одно зміниться.

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

Зверніть увагу, що ми встановлюємо параметр `skipPreflight: true` у функції `sendAndConfirmRawTransaction`. Цей крок дуже важливий, оскільки без нього транзакція ніколи не досягне мережі. Натомість бібліотека відхилить її та викине помилку, що призведе до того, що nonce не зміниться.

Однак це не вся історія. У наступному тесті ми побачимо сценарій, коли навіть якщо транзакція не вдалася, nonce не зміниться.

## 5. Тест: Нонс-акаунт не зміниться, якщо транзакція не вдалася через інструкцію зміни nonce

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
