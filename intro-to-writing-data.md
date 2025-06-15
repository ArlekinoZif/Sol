---
назва: Створення транзакцій у мережі Solana
завдання:
- Пояснити, що таке транзакції
- Пояснити, що таке комісія за транзакції
- Використати `@solana/web3.js` для відправлення SOL
- Використати `@solana/web3.js` для підписання транзакцій
- Використати Solana Explorer для перегляду транзакцій
---

# Стислий виклад

Усі зміни ончен-даних відбуваються через **транзакції**. Транзакції здебільшого складаються з набору інструкцій, які викликають програми Solana. Транзакції атомарні — це означає, що вони або успішно виконуються, якщо всі інструкції пройшли коректно, або зазнають невдачі, ніби взагалі не запускалися.

# Урок

## Транзакції є атомарними

Будь-які зміни ончен-даних відбуваються через транзакції, які надсилаються програмам.

Транзакція в Solana подібна до транзакції в інших системах: вона є атомарною. Атомарна означає, що вся транзакція або виконується повністю, або не виконується взагалі.

Уявіть, що ви оплачуєте щось онлайн:

* З вашого рахунку списуються кошти
* Банк переказує гроші продавцю

Обидві ці дії мають відбутися, щоб транзакція вважалась успішною. Якщо якась із них не спрацює, не повинно відбутися жодної — інакше може статися так, що продавець отримає оплату, а з вашого рахунку гроші не спишуться, або ж гроші спишуться, але продавець нічого не отримає.

Атомарна означає: або транзакція відбувається повністю — тобто всі окремі кроки виконуються успішно — або вся транзакція скасовується.

## Транзакції містять інструкції

Кроки всередині транзакції в Solana називаються **інструкціями**.

Кожна інструкція містить:

* масив акаунтів, з яких буде зчитуватися і/або до яких буде записуватися. Саме це робить Solana швидкою — транзакції, які впливають на різні акаунти, обробляються одночасно
* публічний ключ програми, яку потрібно викликати
* дані, що передаються викликаній програмі, у вигляді масиву байтів

Коли транзакція виконується, одна або кілька програм Solana викликаються згідно з інструкціями, включеними до транзакції.

Як і можна очікувати, `@solana/web3.js` надає допоміжні функції для створення транзакцій та інструкцій. Нову транзакцію можна створити за допомогою конструктора `new Transaction()`. Після створення до транзакції можна додати інструкції за допомогою методу `add()`.

Одна з таких допоміжних функцій — це `SystemProgram.transfer()`, яка створює інструкцію для `SystemProgram` з метою переказу певної кількості SOL:

```typescript
const transaction = new Transaction()

const sendSolInstruction = SystemProgram.transfer({
  fromPubkey: sender,
  toPubkey: recipient,
  lamports: LAMPORTS_PER_SOL * amount
})

transaction.add(sendSolInstruction)
```

Функція `SystemProgram.transfer()` потребує:

* публічного ключа, що відповідає акаунту відправника
* публічного ключа, що відповідає акаунту отримувача
* кількості SOL для надсилання в лампортах

`SystemProgram.transfer()` повертає інструкцію для надсилання SOL від відправника до отримувача.

Програма, яка використовується в цій інструкції — це `system` (за адресою `11111111111111111111111111111111`), дані — це кількість SOL для переказу (в лампортах), а акаунти визначаються на основі відправника та отримувача.

Інструкцію можна додати до транзакції.

Після того, як всі інструкції додані, транзакцію потрібно відправити до кластера та підтвердити:

```typescript
const signature = sendAndConfirmTransaction(
  connection,
  transaction,
  [senderKeypair]
)
```

Функція `sendAndConfirmTransaction()` приймає такі параметри:

* підключення до кластера
* транзакцію
* масив пар ключів, які виступатимуть підписантами транзакції — у цьому прикладі у нас лише один підписант: відправник.

## Транзакції мають комісію

Комісії за транзакції закладені в економіку Solana як компенсація мережі валідаторів за ресурси CPU та GPU, необхідні для обробки транзакцій. Комісії в Solana є детермінованими.

Перший підписант у масиві підписантів транзакції відповідає за сплату комісії. Якщо у цього підписанта недостатньо SOL на рахунку для покриття комісії, транзакція буде відхилена з помилкою на кшталт:

```
> Transaction simulation failed: Attempt to debit an account but found no record of a prior credit.
```

Якщо ви отримали цю помилку, це означає, що ваша пара ключів нова і на рахунку немає SOL для оплати комісії за транзакції. Виправимо це, додавши такі рядки одразу після налаштування підключення:

```typescript
await airdropIfRequired(
  connection,
  keypair.publicKey,
  1 * LAMPORTS_PER_SOL,
  0.5 * LAMPORTS_PER_SOL,
);
```

Це зарахує 1 SOL на ваш акаунт, якщо баланс буде менший за 0.5 SOL — ці кошти можна використовувати для тестування.
Це не спрацює в Mainnet, де SOL має реальну вартість, але дуже зручно для локального тестування та роботи в Devnet.

Також можна скористатися командою Solana CLI `solana airdrop 1`, щоб отримати безкоштовні тестові SOL на свій акаунт під час тестування локально або в Devnet.

## Solana Explorer

![Solana Explorer, встановлений на Devnet](../assets/solana-explorer-devnet.png)

Всі транзакції на блокчейні публічно доступні для перегляду на [Solana Explorer](http://explorer.solana.com). Наприклад, ви можете взяти підпис (signature), який повертає функція `sendAndConfirmTransaction()` у наведеному вище прикладі, знайти цей підпис у Solana Explorer і побачити:

* коли відбулася
* у який блок була включена
* розмір комісії за транзакцію
* і багато іншого!

![Solana Explorer із деталями про транзакцію](../assets/solana-explorer-transaction-overview.png)

# Лабораторна робота

Ми створимо скрипт для відправлення SOL іншим студентам.

### 1. Базова структура

Почнемо з використання тих самих пакетів і файлу `.env`, які ми зробили раніше в [Вступі до криптографії](./intro-to-cryptography).

Створіть файл під назвою `transfer.ts`:

```typescript
import {
  Connection,
  Transaction,
  SystemProgram,
  sendAndConfirmTransaction,
  PublicKey,
} from "@solana/web3.js";
import "dotenv/config"
import { getKeypairFromEnvironment } from "@solana-developers/helpers";

const suppliedToPubkey = process.argv[2] || null;

if (!suppliedToPubkey) {
  console.log(`Please provide a public key to send to`);
  process.exit(1);
}

const senderKeypair = getKeypairFromEnvironment("SECRET_KEY");

console.log(`suppliedToPubkey: ${suppliedToPubkey}`);

const toPubkey = new PublicKey(suppliedToPubkey);

const connection = new Connection("https://api.devnet.solana.com", "confirmed");

console.log(
  `✅ Loaded our own keypair, the destination public key, and connected to Solana`
);
```

Запустіть скрипт, щоб переконатися, що він підключається, завантажує вашу пару ключів і завантажує:


```bash
npx esrun transfer.ts (destination wallet address)
```

### Створіть транзакцію та виконайте її

Додайте наступне, щоб завершити транзакцію та відправити її:

```typescript
console.log(
  `✅ Loaded our own keypair, the destination public key, and connected to Solana`
);

const transaction = new Transaction();

const LAMPORTS_TO_SEND = 5000;

const sendSolInstruction = SystemProgram.transfer({
  fromPubkey: senderKeypair.publicKey,
  toPubkey,
  lamports: LAMPORTS_TO_SEND,
});

transaction.add(sendSolInstruction);

const signature = await sendAndConfirmTransaction(connection, transaction, [
  senderKeypair,
]);

console.log(
  `💸 Finished! Sent ${LAMPORTS_TO_SEND} to the address ${toPubkey}. `
);
console.log(`Transaction signature is ${signature}!`);
```
### Експериментуйте!

Відправляйте SOL іншим студентам у класі.

```bash
npx esrun transfer.ts (destination wallet address)
```

# Challenge

Відповідьте на наступні питання:

* Скільки SOL було витрачено на переказ? Скільки це у доларах США?

* Чи можете ви знайти свою транзакцію на https://explorer.solana.com? Пам’ятайте, що ми використовуємо мережу `devnet`.

* Скільки часу займає переказ? 

* Як ви думаєте, що означає статус «confirmed»?

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=dda6b8de-9ed8-4ed2-b1a5-29d7a8a8b415)!
