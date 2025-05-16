---
назва: Використання кастомних ончейн-програм
завдання:
- Створення транзакцій для кастомних ончейн-програм
---

# Стислий виклад

У Solana є кілька ончейн-програм, які ви можете використовувати. Інструкції, що використовують ці програми, повинні містити дані у кастомному форматі, визначеному програмою.

# Урок
### Інструкції

У попередніх розділах ми використовували:

- Функцію `SystemProgram.transfer()` з `@solana/web3.js` для створення інструкції для системної програми з переказу SOL.
- Функції `mintTo()` та `transfer()` з `@solana/spl-token` для створення інструкцій до токен-програми для мінта та переказу токенів.
- Функцію `createCreateMetadataAccountV3Instruction()` з `@metaplex-foundation/mpl-token-metadata@2` для створення інструкцій до Metaplex для створення метаданих токена.

Однак при роботі з іншими програмами вам доведеться створювати інструкції вручну. За допомогою `@solana/web3.js` ви можете створювати інструкції за допомогою конструктора `TransactionInstruction`:

```typescript
const instruction = new TransactionInstruction({
  programId: PublicKey;
  keys: [ 
    {
      pubkey: Pubkey,
      isSigner: boolean,
      isWritable: boolean,
    },
  ],
  data?: Buffer;
});
```

`TransactionInstruction()` приймає 3 поля:

- Поле `programId` є досить зрозумілим: це публічний ключ (також званий 'адресою' або 'ID програми') програми.

- `keys` — це масив акаунтів та їх використання під час транзакції. Ви повинні знати поведінку програми, до якої ви звертаєтесь, і забезпечити наявність усіх необхідних акаунтів у масиві.
  - `pubkey` — публічний ключ акаунту
  - `isSigner` — булеве значення, яке вказує, чи є акаунт підписантом транзакції
  - `isWritable` — булеве значення, яке вказує, чи буде акаунт змінюватися під час виконання транзакції

- необов'язковий `Buffer`, який містить дані для передачі програмі. Ми поки що ігноруватимемо поле `data`, але повернемося до нього в наступному уроці.

Після створення інструкції ми додаємо її до транзакції, надсилаємо на наш RPC для обробки та підтвердження, а потім переглядаємо підпис транзакції.

```typescript
const transaction = new web3.Transaction().add(instruction)

const signature = await web3.sendAndConfirmTransaction(
  connection,
  transaction,
  [payer],
);

console.log(`✅ Success! Transaction signature is: ${signature}`);
```

### Solana Explorer

![Solana Explorer, встановлений на Devnet](../assets/solana-explorer-devnet.png)

Усі транзакції в блокчейні є публічно доступними для перегляду на [Solana Explorer](http://explorer.solana.com). Наприклад, ви можете взяти підпис, який повертається функцією `sendAndConfirmTransaction()` у наведеному вище прикладі, знайти цей підпис у Solana Explorer і побачити:

- коли вона відбулася
- у якому блоці була включена
- комісію за транзакцію
- та багато іншого!

![Solana Explorer з деталями про транзакцію](../assets/solana-explorer-transaction-overview.png)

# Лабораторна робота 

### Написання транзакцій для програми лічильника ping

Ми створимо скрипт для надсилання ping до ончейн-програми, яка збільшує лічильник щоразу, коли її пінгують. Ця програма розміщена в Solana Devnet за адресою `ChT1B39WKLS8qUrkLvFDXMhEJ4F1XZzwUNHUt4AU9aVa`. Програма зберігає свої дані в окремому акаунті за адресою `Ah9K7dQ8EHaZqcAsgBW8w37yN2eAy3koFmUn4x3CJtod`.

![Solana зберігає програми та дані в окремих акаунтах](../assets/pdas-global-state.svg)

### 1. Базова структура

Ми почнемо з використання тих самих пакетів і файлу `.env`, які ми створили раніше в [Вступі до запису даних](./intro-to-writing-data).

Назвіть файл `send-ping-transaction.ts`:

```typescript
import * as web3 from "@solana/web3.js";
import "dotenv/config"
import { getKeypairFromEnvironment, airdropIfRequired } from "@solana-developers/helpers";

const payer = getKeypairFromEnvironment('SECRET_KEY')
const connection = new web3.Connection(web3.clusterApiUrl('devnet'))

const newBalance = await airdropIfRequired(
  connection,
  payer.publicKey,
  1 * web3.LAMPORTS_PER_SOL,
  0.5 * web3.LAMPORTS_PER_SOL,
);
```

Це підключиться до Solana Devnet і, за потреби, запитає тестові лампорти.

### 2. Програма Ping

Тепер давайте взаємодіяти з програмою Ping! Для цього потрібно:

1. створити транзакцію
2. створити інструкцію
3. додати інструкцію до транзакції
4. надіслати транзакцію

Пам’ятайте, що найскладніше тут — це правильно включити всю необхідну інформацію в інструкції. Ми знаємо адресу програми, яку викликаємо. Також ми знаємо, що ця програма записує дані в окремий акаунт, адресу якого ми також маємо. Додаймо рядкові версії обох адрес як константи на початку файлу:

```typescript
const PING_PROGRAM_ADDRESS = new web3.PublicKey('ChT1B39WKLS8qUrkLvFDXMhEJ4F1XZzwUNHUt4AU9aVa')
const PING_PROGRAM_DATA_ADDRESS =  new web3.PublicKey('Ah9K7dQ8EHaZqcAsgBW8w37yN2eAy3koFmUn4x3CJtod')
```

Тепер давайте створимо нову транзакцію, а потім ініціалізуємо `PublicKey` для акаунту програми та ще один — для акаунту з даними.

```typescript
const transaction = new web3.Transaction()
const programId = new web3.PublicKey(PING_PROGRAM_ADDRESS)
const pingProgramDataId = new web3.PublicKey(PING_PROGRAM_DATA_ADDRESS)
```

Далі давайте створимо інструкцію. Пам’ятайте, що інструкція повинна включати публічний ключ програми Ping, а також масив із усіма акаунтами, з яких буде здійснюватися читання або в які буде виконуватися запис. У цьому прикладі потрібен лише акаунт з даними, згаданий вище.

```typescript
const transaction = new web3.Transaction()
const programId = new web3.PublicKey(PING_PROGRAM_ADDRESS)
const pingProgramDataId = new web3.PublicKey(PING_PROGRAM_DATA_ADDRESS)

const instruction = new web3.TransactionInstruction({
  keys: [
    {
      pubkey: pingProgramDataId,
      isSigner: false,
      isWritable: true
    },
  ],
  programId
})
```

Далі давайте додамо цю інструкцію до транзакції, яку ми створили. Потім викликаємо `sendAndConfirmTransaction()`, передавши підключення, транзакцію та платника. Наприкінці давайте виведемо результат цього виклику функції, щоб ми могли перевірити його в Solana Explorer.

```typescript
transaction.add(instruction)

const signature = await web3.sendAndConfirmTransaction(
  connection,
  transaction,
  [payer]
)

console.log(`✅ Transaction completed! Signature is ${signature}`)
```

### 3. Запустіть клієнт пінгів і перевірте Solana Explorer

Тепер запустіть код за допомогою наступної команди:

```bash
npx esrun send-ping-transaction.ts
```

Це може зайняти кілька секунд, але ви повинні побачити довгий рядок, виведений у консолі, подібний до наступного:

```
✅ Transaction completed! Signature is 55S47uwMJprFMLhRSewkoUuzUs5V6BpNfRx21MpngRUQG3AswCzCSxvQmS3WEPWDJM7bhHm3bYBrqRshj672cUSG
```

Скопіюйте підпис транзакції. Відкрийте браузер і перейдіть за посиланням [https://explorer.solana.com/?cluster=devnet](https://explorer.solana.com/?cluster=devnet) (параметр у кінці URL гарантує, що ви переглядаєте транзакції саме в Devnet, а не в Mainnet). Вставте підпис у рядок пошуку у верхній частині Solana Explorer (переконайтеся, що ви підключені до Devnet), і натисніть Enter. Ви побачите всі деталі транзакції. Якщо прокрутити сторінку до самого низу, з’явиться розділ `Program Logs`, де буде показано, скільки разів програму було пінговано, включно з вашим пінгом.

![Solana Explorer з логами виклику програми Ping](../assets/solana-explorer-ping-result.png)

Прокрутіть сторінку в Explorer і зверніть увагу на те, що ви бачите:
- У розділі **Account Input(s)** ви побачите:
  - Адресу вашого платника — з нього буде списано 5000 лампортів за транзакцію
  - Адресу програми для підрахунку пінгів
  - Адресу акаунту з даними цієї програми
- У розділі **Instruction** буде одна інструкція без жодних даних — програма для підрахунку пінгів досить проста і не потребує додаткових даних.
- У **Program Instruction Logs** відображаються логи від цієї програми.

[//]: # "TODO: з цього вийшло б гарне інтерактивне запитання–відповідь, коли ми розмістимо цей контент на solana.com і зможемо легко додавати більше інтерактивного контенту."

Якщо ви хочете полегшити перегляд транзакцій у Solana Explorer у майбутньому, просто змініть свій `console.log` на таке:

```typescript
console.log(`You can view your transaction on Solana Explorer at:\nhttps://explorer.solana.com/tx/${signature}?cluster=devnet`)
```

And just like that you’re calling programs on the Solana network and writing data onchain!

In the next few lessons, you’ll learn how to

1. Send transactions safely from the browser instead of running a script
2. Add custom data to your instructions
3. Deserialize data from the chain

# Challenge

Go ahead and create a script from scratch that will allow you to transfer SOL from one account to another on Devnet. Be sure to print out the transaction signature so you can look at it on Solana Explorer.

If you get stuck feel free to glance at the [solution code](https://github.com/Unboxed-Software/solana-ping-client).


## Completed the lab?

Push your code to GitHub and [tell us what you thought of this lesson](https://form.typeform.com/to/IPH0UGz7#answers-lesson=e969d07e-ae85-48c3-976f-261a22f02e52)!
