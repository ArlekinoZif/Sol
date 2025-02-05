---
назва: Незмінний Власник (Immutable Owner)
завдання:
 - Створення токен-акаунтів з незмінним власником
 - Пояснення випадків використання розширення незмінного власника
 - Експерименти з правилами розширення
---
# Стислий виклад 

- Розширення `immutable owner` гарантує, що після створення токен-акаунта його власник не може бути змінений, що забезпечує захист власності від будь-яких змін.
- Токен-акаунти з цим розширенням можуть мати лише один постійний стан щодо власності: **Immutable**.
- Асоційовані токен-акаунти (ATA) мають розширення `immutable owner`, увімкнене за замовчуванням.
- Розширення `immutable owner` є розширенням токен-акаунта; воно увімкнене для кожного токен-акаунта, а не для мінт-акаунта.

# Огляд

Асоційовані токен-акаунти (ATA) унікально визначені власником та мінт акаунтом, спрощуючи процес ідентифікації правильного токен-акаунта для конкретного власника. Спочатку будь-який токен-акаунт міг змінювати свого власника, навіть ATA. Це призвело до проблем з безпекою, оскільки користувачі могли випадково надіслати кошти на акаунт, який більше не належав очікуваному отримувачу. Це може непомітно призвести до втрати коштів, якщо власник акаунта зміниться.

Розширення `immutable owner`, яке автоматично застосовується до ATA, запобігає будь-яким змінам у власності. Це розширення також можна увімкнути для нових токен-акаунтів, створених через програму токен-розширень, гарантуючи, що як тільки власність встановлено, вона є постійною. Це забезпечує захист акаунтів від несанкціонованого доступу та спроб переведення.

Важливо зазначити, що це розширення є розширенням токен-акаунта, тобто воно застосовується до токен-акаунта, а не до мінт акаунта.

## Створення токен-акаунта з незмінним власником

У всіх токен-акаунтів в програмі токен-розширень розширення immutable owner увімкнене за замовчуванням. Якщо ви хочете створити ATA, ви можете використовувати `createAssociatedTokenAccount`.

Окрім ATA, які мають розширення immutable owner увімкнене за замовчуванням, ви можете вручну увімкнути це розширення на будь-якому токен-акаунті в програмі токен-розширень.

Ініціалізація токен-акаунта з незмінним власником передбачає три інструкції:

- `SystemProgram.createAccount`
- `createInitializeImmutableOwnerInstruction`
- `createInitializeAccountInstruction`

Примітка: Ми припускаємо, що мінт-акаунт вже був створений.

Перша інструкція `SystemProgram.createAccount` виділяє місце на блокчейні для токен-акаунта. Ця інструкція виконує три завдання:

- Виділяє `space`
- Переводить `lamports` для оренди
- Призначає до програми-власника

```typescript
const tokenAccountKeypair = Keypair.generate();
const tokenAccount = tokenAccountKeypair.publicKey;
const extensions = [ExtensionType.ImmutableOwner];

const tokenAccountLen = getAccountLen(extensions);
const lamports = await connection.getMinimumBalanceForRentExemption(tokenAccountLen);

const createTokenAccountInstruction = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: tokenAccount,
  space: tokenAccountLen,
  lamports,
  programId: TOKEN_2022_PROGRAM_ID,
});
```

Друга інструкція `createInitializeImmutableOwnerInstruction` ініціалізує розширення immutable owner.

```typescript
const initializeImmutableOwnerInstruction =
  createInitializeImmutableOwnerInstruction(
    tokenAccount,
    TOKEN_2022_PROGRAM_ID,
  );
```

Третя інструкція `createInitializeAccountInstruction` ініціалізує токен-акаунт.

```typescript
const initializeAccountInstruction = createInitializeAccountInstruction(
  tokenAccount,
  mint,
  owner.publicKey,
  TOKEN_2022_PROGRAM_ID,
);
```

Нарешті, додайте всі ці інструкції до транзакції та надішліть її в блокчейн.
```ts
const transaction = new Transaction().add(
  createTokenAccountInstruction,
  initializeImmutableOwnerInstruction,
  initializeAccountInstruction,
);

transaction.feePayer = payer.publicKey;

return await sendAndConfirmTransaction(
  connection,
  transaction,
  [payer, owner, tokenAccountKeypair],
);
```
Коли транзакція з цими трьома інструкціями надіслана, створюється новий токен-акаунт з розширенням immutable owner.

# Лабораторна робота  

У цій лабораторній роботі ми створимо токен-акаунт з незмінним власником. Потім напишемо тести, щоб перевірити, чи працює розширення належним чином, спробувавши змінити власника токен-акаунта.

### 1. Налаштування середовища  

Щоб розпочати, створіть порожню директорію з назвою `immutable-owner` і перейдіть до неї. Ми ініціалізуємо абсолютно новий проект. Виконайте `npm init -y`, щоб створити проект із параметрами за замовчуванням.

Далі нам потрібно додати залежності. Виконайте наступну команду, щоб встановити необхідні пакети:
```bash
npm i @solana-developers/helpers @solana/spl-token @solana/web3.js esrun dotenv typescript
```

Створіть каталог `src`. У цьому каталозі створіть файл `index.ts`. Тут ми будемо виконувати перевірки правил цього розширення. Вставте наступний код у `index.ts`:

```ts
import { AuthorityType, TOKEN_2022_PROGRAM_ID, createMint, setAuthority } from "@solana/spl-token";
import { Connection, Keypair, LAMPORTS_PER_SOL, PublicKey } from "@solana/web3.js";
import { initializeKeypair, makeKeypairs } from "@solana-developers/helpers";

const connection = new Connection("http://127.0.0.1:8899", 'confirmed');
const payer = await initializeKeypair(connection);

const [otherOwner, mintKeypair, ourTokenAccountKeypair] = makeKeypairs(3)
const ourTokenAccount = ourTokenAccountKeypair.publicKey;
```

### 2. Запуск ноди валідатора  

У рамках цієї інструкції ми запускатимемо власну ноду валідатора.

У окремому терміналі запустіть наступну команду: `solana-test-validator`. Це запустить ноду та збереже в логах деякі ключі й значення. Значення, яке нам потрібно отримати та використати для з'єднання — це JSON RPC URL, в даному випадку це `http://127.0.0.1:8899`. Потім ми використовуємо це значення для вказівки на використання локального RPC URL.

```typescript
const connection = new Connection("http://127.0.0.1:8899", "confirmed");
```

Альтернативно, якщо ви хочете використовувати тестову або devnet-мережу, імпортуйте `clusterApiUrl` з `@solana/web3.js` і передайте його у підключення наступним чином:

```typescript
const connection = new Connection(clusterApiUrl('devnet'), 'confirmed');
```

### 3. Хелпери

Коли ми вставили код з `index.ts` раніше, ми додали наступні хелпери:

- `initializeKeypair`: Ця функція створює пару ключів для `payer` і також надсилає 2 тестові SOL на її рахунок.
- `makeKeypairs`: Ця функція створює пару ключів без надсилання SOL.

Додатково, у нас є кілька початкових акаунтів:
  - `payer`: Використовується для оплати та є авторитетом для всього.
  - `mintKeypair`: Наш мінт-акаунт.
  - `ourTokenAccountKeypair`: Токен-акаунт, що належить `payer` і який ми будемо використовувати для тестування.
  - `otherOwner`: Токен-акаунт, якому ми спробуємо передати власність двох незмінних акаунтів.

### 4. Створення мінт-акаунту  

Давайте створимо мінт-акаунт, який ми використовуватимемо для наших токен-акаунтів.  

У файлі `src/index.ts` вже буде імпортовано необхідні залежності, а також згадані раніше акаунти. Додайте наступну функцію `createMint` до наявного коду:

```typescript
// CREATE MINT
const mint = await createMint(
  connection,
  payer,
  mintKeypair.publicKey,
  null,
  2,
  undefined,
  undefined,
  TOKEN_2022_PROGRAM_ID,
);
```

### 5. Створення токен-акаунту з розширенням immutable owner  

Пам'ятайте, що всі ATAs мають розширення `immutable owner` за замовчуванням. Однак ми створимо токен-акаунт, використовуючи пару ключів. Це вимагає від нас створення акаунту, ініціалізації розширення immutable owner і ініціалізації акаунту.

У директорії `src` створіть новий файл із назвою `token-helper.ts` і додайте в нього функцію `createTokenAccountWithImmutableOwner`. У цій функції ми створимо асоційований токен-акаунт із незмінним власником. Функція прийматиме такі аргументи:

- `connection`: об'єкт підключення  
- `mint`: публічний ключ мінт-акаунту  
- `payer`: акаунт, який оплачує транзакцію  
- `owner`: власник асоційованого токен-акаунту  
- `tokenAccountKeypair`: пара ключів токен-акаунту, пов’язана з токен-акаунтом

```ts
import { ExtensionType, TOKEN_2022_PROGRAM_ID, createInitializeAccountInstruction, createInitializeImmutableOwnerInstruction, getAccountLen } from "@solana/spl-token";
import { Connection, Keypair, PublicKey, SystemProgram, Transaction, sendAndConfirmTransaction } from "@solana/web3.js";

export async function createTokenAccountWithImmutableOwner(
  connection: Connection,
  mint: PublicKey,
  payer: Keypair,
  owner: Keypair,
  tokenAccountKeypair: Keypair
): Promise<string> {

  // Create account instruction

  // Enable immutable owner instruction

  // Initialize account instruction

  // Send to blockchain
  
  return 'TODO Replace with signature';

}
```

Перший крок у створенні токен-акаунту — це резервування місця в Solana за допомогою методу **`SystemProgram.createAccount`**. Для цього потрібно вказати пару ключів платника (акаунт, який фінансуватиме створення і надаватиме SOL для звільнення від оренди), публічний ключ нового токен-акаунту (**`tokenAccountKeypair.publicKey`**), обсяг пам’яті, необхідний для збереження інформації про токен у блокчейні, кількість SOL (лампортів), необхідних для звільнення акаунту від орендної плати, та ID токен-програми, яка керуватиме цим токен-акаунтом (**`TOKEN_2022_PROGRAM_ID`**).

```typescript
// CREATE ACCOUNT INSTRUCTION
const tokenAccount = tokenAccountKeypair.publicKey;

const extensions = [ExtensionType.ImmutableOwner];

const tokenAccountLen = getAccountLen(extensions);
const lamports = await connection.getMinimumBalanceForRentExemption(tokenAccountLen);

const createTokenAccountInstruction = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: tokenAccount,
  space: tokenAccountLen,
  lamports,
  programId: TOKEN_2022_PROGRAM_ID,
});
```

Після створення токен-акаунту наступна інструкція ініціалізує розширення `immutable owner`. Для створення цієї інструкції використовується функція `createInitializeImmutableOwnerInstruction`.

```typescript
// ENABLE IMMUTABLE OWNER INSTRUCTION
const initializeImmutableOwnerInstruction =
  createInitializeImmutableOwnerInstruction(
    tokenAccount,
    TOKEN_2022_PROGRAM_ID,
  );
```

Далі ми додаємо інструкцію ініціалізації акаунту, викликаючи **`createInitializeAccountInstruction`** і передаючи необхідні аргументи. Ця функція надається пакетом SPL Token і створює інструкцію транзакції для ініціалізації нового токен-акаунту.

```typescript
  // INITIALIZE ACCOUNT INSTRUCTION
const initializeAccountInstruction = createInitializeAccountInstruction(
  tokenAccount,
  mint,
  owner.publicKey,
  TOKEN_2022_PROGRAM_ID,
);
```

Тепер, коли інструкції створено, акаунт токена можна створити з розширенням immutable owner.

```typescript
// SEND TO BLOCKCHAIN
const transaction = new Transaction().add(
  createTokenAccountInstruction,
  initializeImmutableOwnerInstruction,
  initializeAccountInstruction,
);

transaction.feePayer = payer.publicKey;

const signature = await sendAndConfirmTransaction(
  connection,
  transaction,
  [payer, owner, tokenAccountKeypair],
);

return signature
```

Тепер, коли ми додали функціональність для `token-helper`, ми можемо створити наші тестові токен-акаунти. Один з двох тестових токен-акаунтів буде створений за допомогою виклику `createTokenAccountWithImmutableOwner`. Інший буде створений за допомогою вбудованої допоміжної функції SPL `createAssociatedTokenAccount`. Ця допоміжна функція створить асоційований токен-акаунт, який за замовчуванням включає незмінного власника. В рамках цього уроку ми будемо тестувати обидва ці підходи.

Поверніться до `index.ts` під змінною mint, створіть наступні два токен-акаунти:

```
// CREATE TEST TOKEN ACCOUNTS: Create explicitly with immutable owner instructions
const createOurTokenAccountSignature = await createTokenAccountWithImmutableOwner(
  connection,
  mint,
  payer,
  payer,
  ourTokenAccountKeypair
);

// CREATE TEST TOKEN ACCOUNTS: Create an associated token account with default immutable owner
const associatedTokenAccount = await createAssociatedTokenAccount(
  connection,
  payer,
  mint,
  payer.publicKey,
  undefined,
  TOKEN_2022_PROGRAM_ID,
);
```

Це все щодо токен-акаунтів! Тепер можемо перейти до тестування, щоб перевірити, чи правильно застосовуються правила розширення, запустивши кілька тестів.

Якщо ви хочете перевірити, чи все працює, запустіть скрипт.
```bash
npx esrun src/index.ts
```

### 6. Тести

**Тест на спробу передачі власності**

Перший токен-акаунт, який створюється, прив'язаний до `ourTokenAccountKeypair`. Ми спробуємо передати власність акаунту до `otherOwner`, який був згенерований раніше. Очікується, що цей тест не вдасться, оскільки новий уповноважений акаунт не є власником акаунту під час його створення.

Додайте наступний код до вашого файлу `src/index.ts`:

```typescript
// TEST TRANSFER ATTEMPT ON IMMUTABLE ACCOUNT
try {
  await setAuthority(
    connection,
    payer,
    ourTokenAccount,
    payer.publicKey,
    AuthorityType.AccountOwner,
    otherOwner.publicKey,
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID
  );

  console.error("You should not be able to change the owner of the account.");

} catch (error) {
  console.log(
    `✅ - We expected this to fail because the account is immutable, and cannot change owner.`
  );
}
```

Тепер ми можемо викликати функцію `setAuthority`, запустивши команду `npx esrun src/index.ts`. Ми повинні побачити наступну помилку в терміналі, що означає, що розширення працює так, як нам потрібно: `✅ - We expected this to fail because the account is immutable, and cannot change owner.` (`✅ - Ми очікували, що це не вдасться, оскільки акаунт є незмінним і не може змінити власника.`)

**Тест на спробу передати право власності до асоційованого токен-акаунта**

Цей тест спробує передати право власності до Асоційованого Токен-акаунта. Очікується, що цей тест також не вдасться, оскільки новий уповноважений акаунт не є власником акаунту під час його створення.

Під попереднім тестом додайте наступний блок try/catch:

```typescript
// TEST TRANSFER ATTEMPT ON ASSOCIATED IMMUTABLE ACCOUNT
try {
  await setAuthority(
    connection,
    payer,
    associatedTokenAccount,
    payer.publicKey,
    AuthorityType.AccountOwner,
    otherOwner.publicKey,
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID
  );

  console.error("You should not be able to change the owner of the account.");

} catch (error) {
  console.log(
    `✅ - We expected this to fail because the associated token account is immutable, and cannot change owner.`
  );
}
```

Тепер ми можемо запустити `npx esrun src/index.ts`. Цей тест повинен вивести повідомлення про помилку, подібну до того, що було в попередньому тесті. Це означає, що обидва наші токен-акаунти дійсно є незмінними і працюють так, як очікується.



Вітаю! Ми щойно створили токен-акаунти та протестували розширення immutable owner! Якщо ви застрягли на будь-якому етапі, ви можете знайти робочий код на гілці `solution` в [цьому репозиторії](https://github.com/Unboxed-Software/solana-lab-immutable-owner/tree/solution).

# Завдання

Створіть свій власний токен-акаунт з незмінним власником.
