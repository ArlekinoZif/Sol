---
заголовок: стандартний стан акаунта
цілі:  
- Створити мінт-акаунт, який за замовчуванням є "замороженим".  
- Пояснити варіанти використання стандартного стану акаунта.  
- Експериментувати з правилами розширення.  
---
# Стислий огляд

- Розширення `default state` дозволяє розробникам створювати нові токен-акаунти для мінта відразу "замороженими". Для їх розморожування та використання потрібно виконати додаткові дії в окремих сервісах.  
- Існує три стани токен-акаунтів: **Ініціалізований**, **Неініціалізований** і **Заморожений**, які визначають, як можна взаємодіяти з акаунтом.  
- Коли токен-акаунт заморожений, його баланс не може змінюватися.  
- Лише адреса **`freezeAuthority`** може заморожувати та розморожувати токен-акаунт.  
- `default state` можна оновити за допомогою команди `updateDefaultAccountState`.  
- У лабораторній роботі продемонстровано створення мінта з розширенням **`default state`** та створення нового токен-акаунта, який автоматично переходить у стан "заморожено" після створення. Робота включає тести, які перевіряють коректність роботи розширення як під час мінта, так і під час передачі токенів у замороженому та розмороженому станах.  

# Огляд

Розширення `default state` дозволяє розробникам обмежувати налаштування так, щоб всі нові токен-акаунти перебували в одному з двох станів: "Ініціалізовано" (Initialized) або "Заморожено" (Frozen). Найбільш корисною є можливість автоматично встановлювати стан "Заморожено". Коли токен-акаунт заморожений, його баланс не може змінюватися: до нього не можна додати токени, з нього не можна передавати токени або спалювати їх. Розморозити заморожений акаунт може тільки власник `freezeAuthority`.

Уявіть, що ви розробник гри на Solana і хочете, щоб тільки гравці вашої гри взаємодіяли з внутрішньоігровими токенами. Ви можете змусити гравців зареєструватися у грі, щоб розморозити свій токен-акаунт, після чого вони зможуть грати та торгувати з іншими гравцями. ЯСаме завдяки завдяки розширенню `default state` усі нові токен-акаунти будуть створені у замороженому стані.

### Типи станів

Існує три типи станів у розширенні "стан за замовчуванням" для токен-акаунтів:

- Неініціалізований (Uninitialized): Цей стан означає, що токен-акаунт було створено, але він ще не був ініціалізований за допомогою Token Program.
- Ініціалізований (Initialized): Акаунт у стані "Ініціалізований" належним чином налаштований через Token Program. Це означає, що для нього вказано конкретний mint і призначено власника.
- Заморожений (Frozen): Заморожений акаунт тимчасово відключений від виконання певних операцій, таких як передача та створення токенів.

```ts
/** Token account state as stored by the program */
export enum AccountState {
    Uninitialized = 0,
    Initialized = 1,
    Frozen = 2,
}
```

Однак розширення `default state` працює лише з двома останніми станами: `Initialized` і `Frozen`. Коли ви заморожуєте акаунт, його стан стає `Frozen`, а коли розморожуєте — `Initialized`.

## Додавання початкового стану акаунта 

Ініціалізація мінту з комісією за переказ включає три інструкції:

- `SystemProgram.createAccount`  
- `createInitializeTransferFeeConfigInstruction`  
- `createInitializeMintInstruction`  

Перша інструкція `SystemProgram.createAccount` виділяє місце на блокчейні для мінт-акаунта. Ця інструкція виконує три основні завдання:

- Виділяє `space` (простір) для акаунта.
- Переказує `lamports` для оренди.
- Призначає до програми власника.

Щоб отримати розмір мінт-акаунта, ми викликаємо функцію `getMintLen`, а для того, щоб отримати лампорти, необхідні для місця, ми викликаємо функцію `getMinimumBalanceForRentExemption`.

```tsx
const mintLen = getMintLen([ExtensionType.DefaultAccountState]);
// Minimum lamports required for Mint Account
const lamports = await connection.getMinimumBalanceForRentExemption(mintLen);

const createAccountInstruction = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: mintKeypair.publicKey,
  space: mintLen,
  lamports,
  programId: TOKEN_2022_PROGRAM_ID,
});
```

Друга інструкція `createInitializeDefaultAccountStateInstruction` ініціалізує розширення початкового стану.

```tsx
const initializeDefaultAccountStateInstruction =
  createInitializeDefaultAccountStateInstruction(
    mintKeypair.publicKey, // Mint
    defaultState, // Default State
    TOKEN_2022_PROGRAM_ID,
  );
```

Третя інструкція `createInitializeMintInstruction` ініціалізує мінт.

```tsx
const initializeMintInstruction = createInitializeMintInstruction(
  mintKeypair.publicKey,
  decimals,
  payer.publicKey,
  payer.publicKey,
  TOKEN_2022_PROGRAM_ID,
);
```

Останнім кроком є додавання всіх цих інструкцій до транзакції та відправка в блокчейн.

```ts
const transaction = new Transaction().add(
  createAccountInstruction,
  initializeDefaultAccountStateInstruction,
  initializeMintInstruction,
);

return await sendAndConfirmTransaction(
  connection,
  transaction,
  [payer, mintKeypair],
);
```

## Оновлення початкового стану акаунту

Ви завжди можете змінити початковий стан акаунту, якщо у вас для цього є відповідне право. Для цього просто викликайте функцію `updateDefaultAccountState`.

```ts
/**
 * Update the default account state on a mint
 *
 * @param connection     Connection to use
 * @param payer          Payer of the transaction fees
 * @param mint        Mint to modify
 * @param state        New account state to set on created accounts
 * @param freezeAuthority          Freeze authority of the mint
 * @param multiSigners   Signing accounts if `freezeAuthority` is a multisig
 * @param confirmOptions Options for confirming the transaction
 * @param programId      SPL Token program account
 *
 * @return Signature of the confirmed transaction
 */
export async function updateDefaultAccountState(
    connection: Connection,
    payer: Signer,
    mint: PublicKey,
    state: AccountState,
    freezeAuthority: Signer | PublicKey,
    multiSigners: Signer[] = [],
    confirmOptions?: ConfirmOptions,
    programId = TOKEN_2022_PROGRAM_ID
): Promise<TransactionSignature>
```

## Оновлення Freeze Authority

Останнім кроком ви можете захотіти оновити `freezeAuthority`, щоб призначити інший акаунт для керування заморожуванням і розморожуванням. Наприклад, якщо ви хочете, щоб заморожування і розморожування оброблялися програмою, ви можете зробити це, викликавши функцію `setAuthority`, додавши правильні акаунти та вказавши `authorityType`, в даному випадку це буде `AuthorityType.FreezeAccount`.

```ts
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
  currentAuthority, 
  AuthorityType.FreezeAccount,
  newAuthority, 
  [],
  undefined,
  TOKEN_2022_PROGRAM_ID
)
```

# Лабораторна робота

У цій лабораторній роботі ми створимо мінт, для якого всі нові токен акаунти будуть початково заморожені за допомогою розширення `default state`. Потім ми напишемо тести, щоб перевірити, чи працює розширення згідно з очікуваннями, намагаючись здійснити мінт і трансфер токенів у замороженому та розмороженому станах.

### 1. Налаштування середовища

Щоб розпочати, створіть порожній каталог із назвою `default-account-state` та перейдіть до нього. Ми будемо ініціалізувати абсолютно новий проєкт. Виконайте команду `npm init` і дотримуйтесь підказок.

Далі необхідно додати залежності. Виконайте наступну команду, щоб встановити потрібні пакети:

```bash
npm i @solana-developers/helpers @solana/spl-token @solana/web3.js esrun dotenv typescript
```

Створіть каталог із назвою `src`. У цьому каталозі створіть файл із назвою `index.ts`. У цьому файлі ми будемо виконувати перевірки цього розширення. Вставте наступний код у `index.ts`:

```ts
import { AccountState, TOKEN_2022_PROGRAM_ID, getAccount, mintTo, thawAccount, transfer, createAccount } from "@solana/spl-token";
import { Connection, Keypair, PublicKey } from "@solana/web3.js";
// import { createTokenExtensionMintWithDefaultState } from "./mint-helper"; // This will be uncommented later
import { initializeKeypair, makeKeypairs } from '@solana-developers/helpers';

const connection = new Connection("http://127.0.0.1:8899", "confirmed");
const payer = await initializeKeypair(connection);

const [mintKeypair, ourTokenAccountKeypair, otherTokenAccountKeypair] = makeKeypairs(3)
const mint = mintKeypair.publicKey;
const decimals = 2;
const defaultState = AccountState.Frozen;

const ourTokenAccount = ourTokenAccountKeypair.publicKey;

// To satisfy the transferring tests
const otherTokenAccount = otherTokenAccountKeypair.publicKey;

const amountToMint = 1000;
const amountToTransfer = 50;

// CREATE MINT WITH DEFAULT STATE

// CREATE TEST TOKEN ACCOUNTS

// TEST: MINT WITHOUT THAWING

// TEST: MINT WITH THAWING

// TEST: TRANSFER WITHOUT THAWING

// TEST: TRANSFER WITH THAWING
```

Файл `index.ts` створює з'єднання із зазначеним вузлом валідатора та викликає `initializeKeypair`. Він також містить кілька змінних, які ми будемо використовувати протягом цього уроку. Саме в `index.ts` ми в кінцевому підсумку викликатимемо решту нашого скрипта після того, як його напишемо.

Якщо ви зіткнетеся з помилкою в `initializeKeypair`, пов'язаною з airdrop, виконайте наступний крок.

### 2. Запустіть валідатор ноду

У рамках цього посібника ми будемо запускати власну валідатор ноду.

В іншому терміналі виконайте наступну команду: `solana-test-validator`. Це запустить ноду і виведе на екран кілька ключів та значень. Значення, яке нам потрібно отримати і використовувати у з'єднанні, — це JSON RPC URL, який у цьому випадку дорівнює `http://127.0.0.1:8899`. Потім ми використовуємо його у з'єднанні, щоб вказати використання локального RPC URL.

```tsx
const connection = new Connection("http://127.0.0.1:8899", "confirmed");
```

Альтернативно, якщо ви хочете використовувати testnet або devnet, імпортуйте `clusterApiUrl` з `@solana/web3.js` і передайте його у з'єднання наступним чином:

```tsx
const connection = new Connection(clusterApiUrl('devnet'), 'confirmed');
```

### 3. Допоміжні функції

Коли ми вставили код `index.ts`, ми додали наступні допоміжні функції:

- `initializeKeypair`: Ця функція створює пару ключів для `payer` і також виконує airdrop деякої кількості SOL.  
- `makeKeypairs`: Ця функція створює пари ключів без airdrop'у SOL.


Додатково, у нас є декілька акаунтів:

- `payer`: Використовується для оплати та виступає як уповноважений для всіх операцій.  
- `mintKeypair`: Наш мінт-акаунт, який матиме розширення `default state`.  
- `ourTokenAccountKeypair`: Токен-акаунт, який належить `payer`, і який ми використовуватимемо для тестування.  
- `otherTokenAccountKeypair`: Ще один токен-акаунт, який буде використовуватись для тестування.  

### 4. Створення Mint із Default Account State  

При створенні мінт-акаунту з початковим станом (default state), необхідно виконати інструкцію для створення акаунту, ініціалізувати default account state для mint-акаунту та ініціалізувати сам mint.  

Створіть асинхронну функцію з назвою `createTokenExtensionMintWithDefaultState` у файлі `src/mint-helpers.ts`. Ця функція створюватиме mint таким чином, що всі нові токен-акаунти матимуть стан "заморожений" (frozen) на початку. Функція прийматиме такі аргументи:  

- `connection`: Об'єкт підключення.  
- `payer`: Той, хто оплачує транзакцію.  
- `mintKeypair`: Keypair для нового mint.  
- `decimals`: Десяткові знаки.  
- `defaultState`: Початковий стан, наприклад: `AccountState.Frozen`.  

Перший крок у створенні mint — це резервування простору в Solana за допомогою методу `SystemProgram.createAccount`. Для цього необхідно вказати keypair платника (обліковий запис, який фінансуватиме створення та надасть SOL для звільнення від орендної плати), публічний ключ нового mint-акаунту (`mintKeypair.publicKey`), простір, необхідний для зберігання інформації про mint у блокчейні, кількість SOL (lamports), необхідних для звільнення облікового запису від орендної плати, і ID токен-програми, яка керуватиме цим mint-акаунтом (`TOKEN_2022_PROGRAM_ID`).

```tsx
const mintLen = getMintLen([ExtensionType.DefaultAccountState]);
// Minimum lamports required for Mint Account
const lamports = await connection.getMinimumBalanceForRentExemption(mintLen);

const createAccountInstruction = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: mintKeypair.publicKey,
  space: mintLen,
  lamports,
  programId: TOKEN_2022_PROGRAM_ID,
});
```

Після створення mint-акаунту наступним кроком є його ініціалізація з використанням початкового стану. Функція `createInitializeDefaultAccountStateInstruction` використовується для створення інструкції, яка дозволяє mint встановлювати `defaultState` для будь-яких нових токен-акаунтів.

```tsx
const initializeDefaultAccountStateInstruction =
  createInitializeDefaultAccountStateInstruction(
    mintKeypair.publicKey,
    defaultState,
    TOKEN_2022_PROGRAM_ID,
  );
```

Далі додамо інструкцію для mint, викликавши `createInitializeMintInstruction` і передавши необхідні аргументи. Ця функція надається пакетом SPL Token і створює інструкцію транзакції, яка ініціалізує новий mint.

```tsx
const initializeMintInstruction = createInitializeMintInstruction(
  mintKeypair.publicKey,
  decimals,
  payer.publicKey, // Designated Mint Authority
  payer.publicKey, //  Designated Freeze Authority
  TOKEN_2022_PROGRAM_ID,
);
```

Нарешті, додамо всі інструкції до транзакції та відправимо її в блокчейн:

```tsx
const transaction = new Transaction().add(
  createAccountInstruction,
  initializeDefaultAccountStateInstruction,
  initializeMintInstruction,
);

return await sendAndConfirmTransaction(
  connection,
  transaction,
  [payer, mintKeypair],
);
```

Підсумуючи, фінальний файл `src/mint-helpers.ts` виглядатиме так:

```ts
import {
  AccountState,
  ExtensionType,
  TOKEN_2022_PROGRAM_ID,
  createInitializeDefaultAccountStateInstruction,
  createInitializeMintInstruction,
  getMintLen,
} from "@solana/spl-token";
import {
  Connection,
  Keypair,
  SystemProgram,
  Transaction,
  sendAndConfirmTransaction,
} from "@solana/web3.js";

/**
 * Creates the token mint with the default state
 * @param connection
 * @param payer
 * @param mintKeypair
 * @param decimals
 * @param defaultState
 * @returns signature of the transaction
 */
export async function createTokenExtensionMintWithDefaultState(
  connection: Connection,
  payer: Keypair,
  mintKeypair: Keypair,
  decimals: number = 2,
  defaultState: AccountState
): Promise<string> {
  const mintLen = getMintLen([ExtensionType.DefaultAccountState]);
  // Minimum lamports required for Mint Account
  const lamports = await connection.getMinimumBalanceForRentExemption(mintLen);

  const createAccountInstruction = SystemProgram.createAccount({
    fromPubkey: payer.publicKey,
    newAccountPubkey: mintKeypair.publicKey,
    space: mintLen,
    lamports,
    programId: TOKEN_2022_PROGRAM_ID,
  });

  const initializeDefaultAccountStateInstruction =
    createInitializeDefaultAccountStateInstruction(
      mintKeypair.publicKey,
      defaultState,
      TOKEN_2022_PROGRAM_ID
    );

  const initializeMintInstruction = createInitializeMintInstruction(
    mintKeypair.publicKey,
    decimals,
    payer.publicKey, // Designated Mint Authority
    payer.publicKey, //  Designated Freeze Authority
    TOKEN_2022_PROGRAM_ID
  );

  const transaction = new Transaction().add(
    createAccountInstruction,
    initializeDefaultAccountStateInstruction,
    initializeMintInstruction
  );

  return await sendAndConfirmTransaction(connection, transaction, [
    payer,
    mintKeypair,
  ]);
}
```

### 6. Налаштування Тестів  

Тепер, коли ми маємо можливість створювати мінт із початковим станом для всіх нових токен-акаунтів, давайте напишемо кілька тестів, щоб перевірити, як це працює.

### 6.1 Створення Мінту з Початковим Станом 

Спочатку створимо мінт із початковим станом `frozen`. Для цього викликаємо функцію `createTokenExtensionMintWithDefaultState`, яку ми щойно створили, у нашому файлі `index.ts`:

```ts
// CREATE MINT WITH DEFAULT STATE
await createTokenExtensionMintWithDefaultState(
  connection,
  payer,
  mintKeypair,
  decimals,
  defaultState
);
```

### 6.2 Створення Тестових Токен-Акаунтів  

Тепер створимо два нових токен-акаунти для тестування. Це можна зробити за допомогою виклику хелпера `createAccount`, який надається бібліотекою SPL Token. Ми використаємо пари ключів, створені на початку: `ourTokenAccountKeypair` та `otherTokenAccountKeypair`.

```tsx
// CREATE TEST TOKEN ACCOUNTS
// Transferring from account
await createAccount(
  connection,
  payer,
  mint,
  payer.publicKey,
  ourTokenAccountKeypair,
  undefined,
  TOKEN_2022_PROGRAM_ID
);
// Transferring to account
await createAccount(
  connection,
  payer,
  mint,
  payer.publicKey,
  otherTokenAccountKeypair,
  undefined,
  TOKEN_2022_PROGRAM_ID
);
```

### 7 Тести

Тепер давайте напишемо кілька тестів, щоб показати взаємодії з розширенням `default state`.

Ми напишемо чотири тести загалом:

1. Мінт без розморожування акаунту отримувача
2. Мінт з розморожуванням акаунту отримувача
3. Трансфер без розморожування акаунту отримувача
4. Трансфер з розморожуванням акаунту отримувача

Ці тести перевірять, як працює стан "frozen" для токен-акаунтів при спробі мінтити чи передавати токени.

### 7.1 Мінт без розморожування рахунку отримувача

Цей тест спробує замінтити токен на `ourTokenAccount` без розморожування рахунку. Очікується, що тест завершиться з помилкою, оскільки рахунок буде заморожений під час спроби виконати мінт. Пам'ятайте, що коли токен-акаунт заморожений, його баланс не може змінюватися.

Для цього давайте оберемо виклик функції `mintTo` в блок `try-catch` і виведемо відповідне повідомлення про результат.

```tsx
// TEST: MINT WITHOUT THAWING
try {
  // Attempt to mint without thawing
  await mintTo(
    connection,
    payer,
    mint,
    ourTokenAccount,
    payer.publicKey,
    amountToMint,
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID
  );

  console.error("Should not have minted...");
} catch (error) {
  console.log(
    "✅ - We expected this to fail because the account is still frozen."
  );
}
```

Протестуйте, запустивши скрипт:
```bash
npx esrun src/index.ts
```

Ми повинні побачити наступну помилку виведену в терміналі, що означає, що розширення працює як очікується: `✅ - We expected this to fail because the account is still frozen.`

### 7.2 Мінт після розморожування акаунта отримувача

Цей тест спробує викоанти мінт токен після того, як акаунт токена буде розморожений. Очікується, що тест пройде успішно, оскільки акаунт буде розморожений.

Ми можемо створити цей тест, викликавши функцію `thawAccount`, а потім — `mintTo`:

```tsx
// TEST: MINT WITH THAWING
// Unfreeze frozen token
await thawAccount(
  connection, 
  payer,
  ourTokenAccount,
  mint, 
  payer.publicKey,
  undefined,
  undefined, 
  TOKEN_2022_PROGRAM_ID
);
// Mint tokens to tokenAccount
await mintTo(
  connection,
  payer,
  mint,
  ourTokenAccount,
  payer.publicKey,
  amountToMint,
  undefined,
  undefined,
  TOKEN_2022_PROGRAM_ID
);

const ourTokenAccountWithTokens = await getAccount(connection, ourTokenAccount, undefined, TOKEN_2022_PROGRAM_ID);

console.log(
  `✅ - The new account balance is ${Number(ourTokenAccountWithTokens.amount)} after thawing and minting.`
);
```

Запустість скрипт, все має пройти успішно.
```bash
npx esrun src/index.ts
```

### 7.3 Передача без розморожування акаунта отримувача

Тепер, коли ми протестували мінт, давайте протестуємо передачу токенів, як заморожених, так і не заморожених. Спочатку протестуємо передачу без розморожування акаунта отримувача. Не забувайте, що за замовчуванням акаунт `otherTokenAccountKeypair` заморожений через розширення.

Знову ж таки, ми очікуємо, що цей тест не пройде, оскільки акаунт `otherTokenAccountKeypair` заморожений і його баланс не може змінюватися.

Для тестування обернемо функцію `transfer` в блок `try-catch`:

```tsx
// TEST: TRANSFER WITHOUT THAWING
try {

  await transfer(
    connection,
    payer,
    ourTokenAccount,
    otherTokenAccount,
    payer,
    amountToTransfer,
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID
  )

  console.error("Should not have minted...");
} catch (error) {
  console.log(
    "✅ - We expected this to fail because the account is still frozen."
  );
}
```

Запустіть тест і перевірте результат:
```bash
npx esrun src/index.ts
```

### 7.4 Передача з розморожуванням акаунта отримувача

Останній тест, який ми створимо, перевіряє передачу токенів після розморожування акаунта отримувача, до якого ми будемо передавати токени. Цей тест повинен пройти успішно, оскільки всі токен акаунти будуть розморожені.

Ми зробимо це, викликавши `thawAccount`, а потім `transfer`:

```tsx
// TEST: TRANSFER WITH THAWING
// Unfreeze frozen token
await thawAccount(
  connection,
  payer,
  otherTokenAccount,
  mint,
  payer.publicKey,
  undefined,
  undefined,
  TOKEN_2022_PROGRAM_ID
);

await transfer(
  connection,
  payer,
  ourTokenAccount,
  otherTokenAccount,
  payer,
  amountToTransfer,
  undefined,
  undefined,
  TOKEN_2022_PROGRAM_ID
);

const otherTokenAccountWithTokens = await getAccount(
  connection,
  otherTokenAccount,
  undefined,
  TOKEN_2022_PROGRAM_ID
);

console.log(
  `✅ - The new account balance is ${Number(
    otherTokenAccountWithTokens.amount
  )} after thawing and transferring.`
);
```

проведіть тести останній раз:
```bash
npx esrun src/index.ts
```

Основні моменти, які потрібно запам'ятати:

- Розширення `default state` накладає стандартний стан на *всі* нові токен акаунти.
- Баланс замороженого акаунта не може змінюватися.

**Вітаємо!** Ми щойно створили та протестували мінт, використовуючи розширення початкового стану!

# Задача:
Додайте тести для спроби спалити токени з заморожених та розморожених токен-акаунтів (підказка: один тест зазнає невдачі, а інший — успіху).

Щоб вам було легше почати:
```ts
// TEST: Burn tokens in frozen account
await freezeAccount(
  connection,
  payer,
  ourTokenAccount,
  mint,
  payer.publicKey,
  [],
  undefined,
  TOKEN_2022_PROGRAM_ID,
)

await burn(
  connection,
  payer,
  ourTokenAccount,
  mint,
  payer.publicKey,
  1,
  [],
  undefined,
  TOKEN_2022_PROGRAM_ID,
)
```
