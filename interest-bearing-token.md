---
назва: Токен із нарахуванням відсотків 
завдання:
- Створити мінт-акаунт із розширенням для нарахування відсотків  
- Пояснити варіанти використання токенів із нарахуванням відсотків  
- Експериментувати з правилами цього розширення
---
# Стислий виклад

- Творці можуть встановлювати процентну ставку та зберігати її безпосередньо в мінт-акаунті.
- Кількість основного токена для токенів із нарахуванням відсотків залишається незмінною.
- Нараховані відсотки можна відображати для UI без необхідності часто змінювати або оновлювати їх для коригування нарахованих відсотків.
- Лабораторна робота демонструє налаштування мінт-акаунту з процентною ставкою. Тестовий випадок також показує, як оновлювати процентну ставку та отримувати ставку з токена.

# Огляд

Токени, чия вартість змінюється з часом, мають практичне застосування в реальному світі, і облігації є яскравим прикладом. Раніше можливість відображати ці динамічні зміни в токенах обмежувалася використанням проксі-контрактів, що вимагало частих ребейсів або оновлень.

Розширення `interest bearing token` допомагає в цьому. Використовуючи розширення `interest bearing token` та функцію `amount_to_ui_amount`, користувачі можуть застосовувати процентну ставку до своїх токенів і отримувати оновлену загальну суму, включаючи відсотки, у будь-який момент.

Розрахунок відсотків здійснюється безперервно з урахуванням часової позначки мережі. Однак, розбіжності в часі мережі можуть призвести до того, що накопичені відсотки будуть трохи меншими, ніж очікувалося, хоча така ситуація є рідкісною.

Важливо зазначити, що цей механізм не генерує нові токени, і відображена сума просто включає накопичені відсотки, що робить зміну суто естетичною. Однак, ця величина зберігається в межах мінт-акаунту, і програми можуть використовувати це для створення функціональності, що виходить за межі чистої естетики.

## Додавання процентної ставки до токена

Ініціалізація токена з відсотками включає три інструкції:

- `SystemProgram.createAccount`
- `createInitializeTransferFeeConfigInstruction`
- `createInitializeMintInstruction`

Перша інструкція `SystemProgram.createAccount` виділяє місце на блокчейні для `mint` акаунту. Ця інструкція виконує три завдання:

- Виділяє `space`
- Переносить `lamports` для оренди
- Призначає собі програму-власника
  
```typescript
SystemProgram.createAccount({
    fromPubkey: payer.publicKey,
    newAccountPubkey: mint,
    space: mintLen,
    lamports: mintLamports,
    programId: TOKEN_2022_PROGRAM_ID,
  }),
```

Друга інструкція `createInitializeInterestBearingMintInstruction` ініціалізує розширення токена з відсотками. Визначальним аргументом, який задає процентну ставку, буде змінна, яку ми створимо, назвавши її `rate`. `Rate` визначається в [базисних пунктах](https://www.investopedia.com/terms/b/basispoint.asp).

```typescript
  createInitializeInterestBearingMintInstruction(
    mint,
    rateAuthority.publicKey,
    rate,
    TOKEN_2022_PROGRAM_ID,
  ),
```

Третя інструкція `createInitializeMintInstruction` ініціалізує мінт-акаунт.

```typescript
 createInitializeMintInstruction(
    mint,
    decimals,
    mintAuthority.publicKey,
    null,
    TOKEN_2022_PROGRAM_ID
  )
```

Коли транзакція з цими трьома інструкціями відправляється, створюється новий токен з відсотками з заданою конфігурацією ставки.

## Отримання накопичених відсотків
Для того, щоб отримати накопичені відсотки по токену на будь-який момент часу, спочатку використовуйте функцію `getAccount` для отримання інформації про токен, включаючи суму та будь-які пов'язані дані, передаючи з'єднання, токен-акаунт платника та відповідний програмний ID, `TOKEN_2022_PROGRAM_ID`.

Далі використовуйте функцію `amountToUiAmount` з отриманою інформацією про токен, а також додатковими параметрами, такими як з'єднання, платник та мінт-акаунт, щоб конвертувати кількість токенів у відповідну кількість для інтерфейсу користувача, що по суті включає будь-які накопичені відсотки.

```typescript
const tokenInfo = await getAccount(connection, payerTokenAccount, undefined, TOKEN_2022_PROGRAM_ID);

/**
 * Get the amount as a string using mint-prescribed decimals
 *
 * @param connection     Connection to use
 * @param payer          Payer of the transaction fees
 * @param mint           Mint for the account
 * @param amount         Amount of tokens to be converted to Ui Amount
 * @param programId      SPL Token program account
 *
 * @return Ui Amount generated
 */
const uiAmount = await amountToUiAmount(
  connection,
  payer,
  mint,
  tokenInfo.amount,
  TOKEN_2022_PROGRAM_ID,
);

console.log("UI Amount: ", uiAmount);
```

Повернуте значення `uiAmount` є рядковим представленням кількості для інтерфейсу користувача і виглядатиме подібно до цього: `0.0000005000001557528245`.

## Оновлення акаунту, що може змінювати ставку
В Solana є допоміжна функція `setAuthority` що дає право новому акаунту змінювати ставку.

Використовуйте функцію `setAuthority` для призначення нового уповноваженого акаунту. Вам потрібно буде вказати `connection`, акаунт, що сплачує комісії за транзакції (payer), токен-акаунт, який потрібно оновити (mint), публічний ключ поточного уповноваженого акаунту, тип уповноваженого акаунту для оновлення (у цьому випадку 7 представляє тип уповноваженого акаунту `InterestRate`), а також публічний ключ нового уповноваженого акаунту.

Після встановлення нового уповноваженого акаунту, використовуйте функцію `updateRateInterestBearingMint` для оновлення процентної ставки для акаунту. Передайте необхідні параметри: `connection`, `payer`, `mint`, публічний ключ нового уповноваженого акаунту, оновлену процентну ставку та програмний ID.

```typescript
/**
 * Assign a new authority to the account
 *
 * @param connection       Connection to use
 * @param payer            Payer of the transaction fees
 * @param account          Address of the account
 * @param currentAuthority Current authority of the specified type
 * @param authorityType    Type of authority to set
 * @param newAuthority     New authority of the account
 * @param multiSigners     Signing accounts if `currentAuthority` is a multisig
 * @param confirmOptions   Options for confirming the transaction
 * @param programId        SPL Token program account
 *
 * @return Signature of the confirmed transaction
 */

await setAuthority(
  connection,
  payer,
  mint,
  rateAuthority,
  AuthorityType.InterestRate, // Rate type (InterestRate)
  otherAccount.publicKey, // new rate authority,
  [],
  undefined,
  TOKEN_2022_PROGRAM_ID
);

await updateRateInterestBearingMint(
  connection,
  payer,
  mint,
  otherAccount, // new rate authority
  10, // updated rate
  undefined,
  undefined,
  TOKEN_2022_PROGRAM_ID,
);
```
# Лабораторна робота

У цій лабораторній роботі ми створюємо токени з відсотками за допомогою програми Token-2022 на Solana. Ми ініціалізуємо ці токени з конкретною процентною ставкою, оновлюємо ставку з відповідним уповноваженим акаунтом і спостерігаємо, як відсотки накопичуються на токенах з часом.

### 1. Налаштування середовища

Для початку створіть порожню директорію з назвою `interest-bearing-token` та перейдіть до неї. Виконайте команду `npm init -y`, щоб ініціалізувати новий проект.

Далі нам потрібно додати залежності. Виконайте наступне для встановлення необхідних пакетів:
```bash
npm i @solana-developers/helpers @solana/spl-token @solana/web3.js esrun dotenv typescript
```

Створіть директорію з назвою `src`. У цій директорії створіть файл з ім'ям `index.ts`. Тут ми будемо перевіряти правила цього розширення. Вставте наступний код у файл `index.ts`:
```ts
import {
  Connection,
  Keypair,
  PublicKey,
} from '@solana/web3.js';

import {
  ExtensionType,
  getMintLen,
  TOKEN_2022_PROGRAM_ID,
  getMint,
  getInterestBearingMintConfigState,
  updateRateInterestBearingMint,
  amountToUiAmount,
  mintTo,
  createAssociatedTokenAccount,
  getAccount,
  AuthorityType
} from '@solana/spl-token';

import { initializeKeypair, makeKeypairs } from '@solana-developers/helpers';

const connection = new Connection("http://127.0.0.1:8899", 'confirmed');
const payer = await initializeKeypair(connection);
const [otherAccount, mintKeypair] = makeKeypairs(2)
const mint = mintKeypair.publicKey;
const rateAuthority = payer;

const rate = 32_767;

// Create an interest-bearing token

// Create an associated token account

// Create the getInterestBearingMint function

// Attempt to update the interest rate

// Attempt to update the interest rate with the incorrect owner

// Log the accrued interest

// Log the interest-bearing mint configuration state

// Update the rate authority and attempt to update the interest rate with the new authority
```
`index.ts` створює з'єднання з вказаною нодою валідатора та викликає `initializeKeypair`. Він також містить кілька змінних, які ми будемо використовувати в решті цієї лабораторної роботи. `index.ts` буде місцем, де ми викликаємо решту нашого скрипта, після того як ми його напишемо.

### 2. Запуск ноди валідатора

Для цієї інструкції ми будемо запускати власну ноду валідатора.

У окремому терміналі виконайте наступну команду: `solana-test-validator`. Це запустить ноду та виведе деякі ключі та значення в логах. Значення, яке нам потрібно отримати та використати в з'єднанні — це JSON RPC URL, яке в цьому випадку виглядає як `http://127.0.0.1:8899`. Потім ми використовуємо це значення у з'єднанні, щоб вказати на використання локального RPC URL.

`const connection = new Connection("http://127.0.0.1:8899", "confirmed");`

### 3. Допоміжні функції

Коли ми вставили код `index.ts` раніше, ми додали наступні допоміжні функції:

- `initializeKeypair`: Ця функція створює пару ключів для `payer` та також виконує ейрдроп 1 тестової SOL на неї.
- `makeKeypairs`: Ця функція створює пару ключів без ейрдропу будь-якої SOL.
  
Крім того, у нас є деякі початкові акаунти:
- `payer`: Використовується для оплати та як уповноважений акаунт для всього
- `mintKeypair`: Наш мінт-акаунт, який буде мати розширення `interest bearing token`
- `otherAccount`: Акаунт, який ми будемо використовувати для спроби оновлення відсотків
- `otherTokenAccountKeypair`: Інший токен, який використовується для тестування

### 4. Створіть токен, на який нараховуються відсотки

Ця функція створюватиме токен таким чином, що всі нові токени будуть створюватися з процентною ставкою. Створіть новий файл у директорії `src` під назвою `token-helper.ts`.

```typescript
import {
  TOKEN_2022_PROGRAM_ID,
  createInitializeInterestBearingMintInstruction,
  createInitializeMintInstruction,
} from "@solana/spl-token";
import {
  sendAndConfirmTransaction,
  Connection,
  Keypair,
  Transaction,
  PublicKey,
  SystemProgram,
} from "@solana/web3.js";

export async function createTokenWithInterestRateExtension(
  connection: Connection,
  payer: Keypair,
  mint: PublicKey,
  mintLen: number,
  rateAuthority: Keypair,
  rate: number,
  mintKeypair: Keypair
) {
  const mintAuthority = payer
  const decimals = 9;
}
```

Ця функція прийматиме наступні аргументи:

- `connection`: Об'єкт з'єднання
- `payer`: Платник для транзакції
- `mint`: Публічний ключ для нового мінт-акаунту
- `rateAuthority`: Пара ключів акаунту, який може змінювати токен, у цьому випадку це буде `payer`
- `rate`: Вибрана процентна ставка для токена. У нашому випадку це буде `32_767`, або 32767, максимальна ставка для розширення токена з відсотками
- `mintKeypair`: Пара ключів для нового мінт-акаунту
  
Коли створюємо токен з нарахуваннями відсотків, ми повинні створити інструкцію для мінт-акаунту, додати інструкцію для нарахування відсотків та ініціалізувати сам мінт. Усередині `createTokenWithInterestRateExtension` в `src/token-helper.ts` вже є кілька змінних, які будуть використані для створення токена з нарахуваннями відсотків. Додайте наступний код після оголошених змінних:

```ts
const extensions = [ExtensionType.InterestBearingConfig];
const mintLen = getMintLen(extensions);
const mintLamports = await connection.getMinimumBalanceForRentExemption(mintLen);

const mintTransaction = new Transaction().add(
  SystemProgram.createAccount({
    fromPubkey: payer.publicKey,
    newAccountPubkey: mint,
    space: mintLen,
    lamports: mintLamports,
    programId: TOKEN_2022_PROGRAM_ID,
  }),
  createInitializeInterestBearingMintInstruction(
    mint,
    rateAuthority.publicKey,
    rate,
    TOKEN_2022_PROGRAM_ID,
  ),
  createInitializeMintInstruction(
    mint,
    decimals,
    mintAuthority.publicKey,
    null,
    TOKEN_2022_PROGRAM_ID
  )
);

await sendAndConfirmTransaction(connection, mintTransaction, [payer, mintKeypair], undefined);
```

Це все для створення токена! Тепер можемо переходити до додавання тестів.
### 5. Створення необхідних акаунтів

Усередині `src/index.ts` початковий код вже містить деякі значення, пов'язані зі створенням токена з нарахуваннями відсотків.

Під існуючою змінною `rate` додайте наступний виклик функції `createTokenWithInterestRateExtension` для створення токена з нарахуваннями відсотків. Нам також потрібно створити асоційований токен-акаунт, в який ми будемо мінтити токени з нарахуваннями відсотків, а також провести деякі тести, щоб перевірити, чи збільшується нарахований відсоток як очікується.

```typescript
const rate = 32_767;

// Create interest bearing token
await createTokenWithInterestRateExtension(
  connection,
  payer,
  mint,
  rateAuthority,
  rate,
  mintKeypair
)

// Create associated token account
const payerTokenAccount = await createAssociatedTokenAccount(
  connection,
  payer,
  mint,
  payer.publicKey,
  undefined,
  TOKEN_2022_PROGRAM_ID
);
```

## 6. Тести

Перш ніж почати писати тести, нам буде корисно мати функцію, яка приймає `mint` і повертає поточну процентну ставку для цього конкретного токена.

Давайте використаємо допоміжну функцію `getInterestBearingMintConfigState`, надану бібліотекою SPL, щоб зробити це. Потім ми створимо функцію, яку будемо використовувати в наших тестах для виведення поточної процентної ставки.

Значення, що повертається цією функцією, є об'єктом з наступними значеннями:

- `rateAuthority`: Пара ключів акаунту, який може змінювати токен
- `initializationTimestamp`: Часова мітка ініціалізації токена з нарахуваннями відсотків
- `preUpdateAverageRate`: Остання ставка перед оновленням
- `lastUpdateTimestamp`: Часова мітка останнього оновлення
- `currentRate`: Поточна процентна ставка

Додайте наступні типи та функцію:

```typescript
// Create getInterestBearingMint function
interface GetInterestBearingMint {
  connection: Connection;
  mint: PublicKey;
}

async function getInterestBearingMint(inputs: GetInterestBearingMint) {
  const { connection, mint } = inputs
  // retrieves information of the mint
  const mintAccount = await getMint(
    connection,
    mint,
    undefined,
    TOKEN_2022_PROGRAM_ID,
  );

	// retrieves the interest state of mint
  const interestBearingMintConfig = await getInterestBearingMintConfigState(
    mintAccount,
  );

  // returns the current interest rate
  return interestBearingMintConfig?.currentRate
}
```

**Оновлення процентної ставки**

Бібліотека Solana SPL надає допоміжну функцію для оновлення процентної ставки токена під назвою `updateRateInterestBearingMint`. Для коректної роботи цієї функції `rateAuthority` цього токена має бути таким самим, як і той акаунт, який створив токен. Якщо `rateAuthority` неправильний, оновлення токена призведе до помилки.

Давайте створимо тест для оновлення ставки з правильним уповноваженим акаунтом. Додайте наступні виклики функцій:

```typescript
// Attempt to update interest rate
const initialRate = await getInterestBearingMint({ connection, mint })
try {
  await updateRateInterestBearingMint(
    connection,
    payer,
    mint,
    payer,
    0, // updated rate
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID,
  );
  const newRate = await getInterestBearingMint({ connection, mint })

  console.log(`✅ - We expected this to pass because the rate has been updated. Old rate: ${initialRate}. New rate: ${newRate}`);
} catch (error) {
  console.error("You should be able to update the interest.");
}
```

Запустіть команду `npx esrun src/index.ts`. Ми повинні побачити наступну помилку, виведену в терміналі, що означатиме, що розширення працює, як очікується, і процентна ставка була оновлена: `✅ - We expected this to pass because the rate has been updated. Old rate: 32767. New rate: 0` (Ми очікували, що це пройде, оскільки ставка була оновлена. Стара ставка: 32767. Нова ставка: 0)

**Оновлення процентної ставки з неправильним `rateAuthority`**

У наступному тесті давайте спробуємо оновити процентну ставку з неправильним `rateAuthority`. Раніше ми створили пару ключів під назвою `otherAccount`. Це буде те, що ми використовуватимемо як `otherAccount`, щоб спробувати змінити процентну ставку.

Під попереднім тестом, який ми створили, додайте наступний код:

```typescript
// Attempt to update the interest rate as the account other than the rate authority.
try {
  await updateRateInterestBearingMint(
    connection,
    otherAccount,
    mint,
    otherAccount, // incorrect authority
    0, // updated rate
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID,
  );
  console.log("You should be able to update the interest.");
} catch (error) {
  console.error(`✅ - We expected this to fail because the owner is incorrect.`);
}
```

Тепер запустіть команду `npx esrun src/index.ts`. Це має призвести до помилки, і в терміналі буде виведено: `✅ - We expected this to fail because the owner is incorrect.` (Ми очікували, що це не вдасться, оскільки власник неправильний)

**Мінт токенів і перевірка процентної ставки**

Отже, ми протестували оновлення процентної ставки. Як перевірити, що нараховані відсотки збільшуються, коли акаунт мінтить більше токенів? Ми можемо використати допоміжні функції `amountToUiAmount` і `getAccount` з бібліотеки SPL для цього.

Давайте створимо цикл, який виконуватиметься 5 разів, змінтить 100 токенів у кожному циклі і виводитиме нову нараховану процентну ставку:

```typescript
// Log accrued interest
{
  // Logs out interest on token
  for (let i = 0; i < 5; i++) {
    const rate = await getInterestBearingMint({ connection, mint });
    await mintTo(
      connection,
      payer,
      mint,
      payerTokenAccount,
      payer,
      100,
      undefined,
      undefined,
      TOKEN_2022_PROGRAM_ID
    );

    const tokenInfo = await getAccount(connection, payerTokenAccount, undefined, TOKEN_2022_PROGRAM_ID);

    // Convert amount to UI amount with accrued interest
    const uiAmount = await amountToUiAmount(
      connection,
      payer,
      mint,
      tokenInfo.amount,
      TOKEN_2022_PROGRAM_ID,
    );

    console.log(`Amount with accrued interest at ${rate}: ${tokenInfo.amount} tokens = ${uiAmount}`);
  }
}
```

Ви повинні побачити щось подібне до наступних логів:

```typescript
Amount with accrued interest at 32767: 100 tokens = 0.0000001000000207670422
Amount with accrued interest at 32767: 200 tokens = 0.0000002000000623011298
Amount with accrued interest at 32767: 300 tokens = 0.0000003000001246022661
Amount with accrued interest at 32767: 400 tokens = 0.00000040000020767045426
Amount with accrued interest at 32767: 500 tokens = 0.0000005000003634233328
```

Ви повинні побачити щось подібне до наступних логів:

**Лог конфігурації мінт-акаунту**

Якщо з якоїсь причини вам потрібно отримати стан конфігурації мінт-акаунту, ми можемо використати функцію `getInterestBearingMintConfigState`, яку ми створили раніше, щоб вивести інформацію про стан мінт-акаунту з нарахуваннями відсотків.

```ts
// Log interest bearing mint config state
const mintAccount = await getMint(
  connection,
  mint,
  undefined,
  TOKEN_2022_PROGRAM_ID,
);

// Get Interest Config for Mint Account
const interestBearingMintConfig = await getInterestBearingMintConfigState(
  mintAccount,
);

console.log(
  "\nMint Config:",
  JSON.stringify(interestBearingMintConfig, null, 2),
);
```

Це повинно вивести щось подібне до цього:

```typescript
Mint Config: {
  "rateAuthority": "Ezv2bZZFTQEznBgTDmaPPwFCg7uNA5KCvMGBNvJvUmS",
  "initializationTimestamp": 1709422265,
  "preUpdateAverageRate": 32767,
  "lastUpdateTimestamp": 1709422267,
  "currentRate": 0
}
```

## Оновлення уповноваженого акаунту і процентної ставки
Перед тим, як завершити цей лабораторний практикум, давайте встановимо новий уповноважений акаунт для токена з нарахуваннями відсотків і спробуємо оновити процентну ставку. Ми зробимо це, використовуючи функцію `setAuthority` і передавши оригінальний уповноважений акаунт, вказавши тип ставки (в цьому випадку це 7 для `InterestRate`), а також публічний ключ нового уповноваженого акаунту.

Після того, як ми встановимо новий уповноважений акаунт, ми можемо спробувати оновити процентну ставку.

```typescript
// Update rate authority and attempt to update interest rate with new authority
try {
  await setAuthority(
    connection,
    payer,
    mint,
    rateAuthority,
    AuthorityType.InterestRate, // Rate type (InterestRate)
    otherAccount.publicKey, // new rate authority,
    [],
    undefined,
    TOKEN_2022_PROGRAM_ID
  );

  await updateRateInterestBearingMint(
    connection,
    payer,
    mint,
    otherAccount, // new authority
    10, // updated rate
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID,
  );

  const newRate = await getInterestBearingMint({ connection, mint })

  console.log(`✅ - We expected this to pass because the rate can be updated with the new authority. New rate: ${newRate}`);
} catch (error) {
  console.error(`You should be able to update the interest with new rate authority.`);
}
```

Це має працювати, і нова процентна ставка повинна бути 10.

Ось і все! Ми щойно створили токен з нарахуваннями відсотків, оновили процентну ставку та вивели оновлений стан токена!

# Завдання

Створіть свій власний токен з нарахуваннями відсотків.
