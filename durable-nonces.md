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

## How to use the durable nonces

Now that we learned about the nonce account and its different operations, let's talk about how to use it. 

We'll discuss:

1. Fetching the nonce account
2. Using the nonce in the transaction to make a durable transaction.
3. Submitting a durable transaction.

### Fetching the nonce account

We can fetch the nonce account to get the nonce value by fetching the account and serializing it:

```ts
const nonceAccount = await connection.getAccountInfo(nonceKeypair.publicKey);

const nonce = NonceAccount.fromAccountData(nonceAccount.data); 
```

### Using the nonce in the transaction to make a durable transaction

To build a fully functioning durable transaction, we need the following:

1. Use the nonce value in replacement of the recent blockhash.
2. Add the nonceAdvance instruction as the first instruction in the transaction.
3. Sign the transaction with the authority of the nonce account.

After building and signing the transaction we can serialize it and encode it into a base58 string, and we can save this string in some store to submit it later.

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

### submitting a durable transaction:

Now that we have a base58 encoded transaction, we can decode it and submit it:

```ts
const tx = base58.decode(serializedTx);
const sig = await sendAndConfirmRawTransaction(connection, tx as Buffer);
```

## Some important edge cases

There are a few things that you need to consider when dealing with durable transactions:
1. If the transaction fails due to an instruction other than the nonce advanced instruction.
2. If the transaction fails due to the nonce advanced instruction.

### If the transaction fails due to an instruction other than the nonce advanced instruction

In the normal case of failing transactions, the known behavior is that all the instructions in the transaction will get reverted to the original state. But in the case of a durable transaction, if any instruction fails that is not the advance nonce instruction, the nonce will still get advanced and all other instructions will get reverted. This feature is designed for security purposes, ensuring that once a user signs a transaction, if it fails, it cannot be used again.

Presigned, never expiring, durable transactions are like signed paychecks. They can be dangerous in the right scenarios. This extra safety feature effectively "voids" the paycheck if handled incorrectly.

### If the transaction fails due to the nonce advanced instruction

If a transaction fails because of the advance instruction, the entire transaction is reverted, meaning the nonce does not advance.

# Lab

In this lab, we'll learn how to create a durable transaction. We'll focus on what you can and can't do with it. Additionally, we'll discuss some edge cases and how to handle them.

## 0. Getting started

Let's go ahead and clone our starter code

```bash
git clone https://github.com/Unboxed-Software/solana-lab-durable-nonces
cd Solana-lab-durable-nonces
git checkout starter
npm install
```

In the starter code you will find a file inside `test/index.ts`, with a testing skeleton, we'll write all of our code here.

We're going to use the local validator for this lab. However, feel free to use devnet if you'd like. ( If you have issues airdropping on devnet, check out [Solana's Faucet](https://faucet.solana.com/) )

To run the local validator, you'll need to have it installed, if you don't you can refer to [installing the Solana CLI](https://docs.solanalabs.com/cli/install), once you install the CLI you'll have access to the `solana-test-validator`.

In a separate terminal run:
```bash
solana-test-validator
```

In `test/index.ts` you'll see five tests, these will help us understand durable nonces better.

We'll discuss each test case in depth.

## 1. Create the nonce account

Before we write any tests, let's create a helper function above the `describe` block, called `createNonceAccount`. 

It will take the following parameters:
- `Connection`: Connection to use
- `payer`: The payer
- `nonceKeypair`: The nonce keypair
- `authority`: Authority over the nonce

It will:
1. Assemble and submit a transaction that will:
   1. Allocate the account that will be the nonce account.
   2. Initialize the nonce account using the `SystemProgram.nonceInitialize` instruction.
2. Fetch the nonce account.
3. Serialize the nonce account data and return it.

Paste the following somewhere above the `describe` block.
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

## 2. Test: Create and submit a durable transaction

To create and submit a durable transaction we must follow these steps:

1. Create a Durable Transaction.
  1. Create the nonce account.
  2. Create a new transaction.
  3. Set the `recentBlockhash` to be the nonce value.
  4. Add the `nonceAdvance` instruction as the first instruction in the transaction.
  5. Add the transfer instruction (you can add any instruction you want here).
  6. Sign the transaction with the keypairs that need to sign it, and make sure to add the nonce authority as a signer as well.
  7. Serialize the transaction and encode it.
  8. At this point you have a durable transaction, you can store it in a database or a file or send it somewhere else, etc.
2. Submit the durable transaction.
  1. Decode the serialized transaction.
  2. Submit it using the `sendAndConfirmRawTransaction` function.

We can put all of this together in our first test:
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

## 3. Test: Transaction fails if the nonce has advanced

Because we are using the nonce in place of the recent blockhash, the system will check to ensure that the nonce we provided matches the nonce in the `nonce_account`. Additionally with each transaction, we need to add the `nonceAdvance` instruction as the first instruction. This ensures that if the transaction goes through, the nonce will change, and no one will be able to submit it twice.

Here is what we'll test:
1. Create a durable transaction just like in the previous step.
2. Advance the nonce.
3. Try to submit the transaction, and it should fail.

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

## 4. Test: Nonce account advances even if the transaction fails

An important edge case to be aware of is that even if a transaction fails for any reason other than the nonce advance instruction, the nonce will still advance. This feature is designed for security purposes, ensuring that once a user signs a transaction and it fails, that durable transaction cannot be used again.

The following code demonstrates this use case. We'll attempt to create a durable transaction to transfer 50 SOL from the payer to the recipient. However, the payer doesn't have enough SOL for the transfer, so the transaction will fail, but the nonce will still advance.

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

Notice that we are setting `skipPreflight: true` in the `sendAndConfirmRawTransaction` function. This step is crucial because, without it, the transaction would never reach the network. Instead, the library would reject it and throw an error, leading to a failure where the nonce does not advance.

However, this is not the whole story. In the upcoming test case, we'll discover a scenario where even if the transaction fails, the nonce will not advance.

## 5. Test: Nonce account will not advance if the transaction fails because of the nonce advance instruction

For the nonce to advance, the `advanceNonce` instruction must succeed. Thus, if the transaction fails for any reason related to this instruction, the nonce will not advance.

A well-formatted `nonceAdvance` instruction will only ever fail if the nonce authority did not sign the transaction.

Let's see this in action.

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

## 6. Test sign transaction and then change nonce authority

The last test case we'll go over is creating a durable transaction. Try to send it with the wrong nonce authority (it will fail). Change the nonce authority and send it with the correct one this time and it will succeed.

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

## 8. Run the tests

Finally, let's run the tests:
```bash
npm start
```

Make sure they are all passing. 

And congratulations! You now know how durable nonces work!

# Challenge

Write a program that creates a durable transaction and saves it to a file, then create a separate program that reads the durable transaction file and sends it to the network.
