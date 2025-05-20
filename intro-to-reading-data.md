---
назва: Зчитування даних з мережі Solana
завдання:
- Розуміти акаунти та їхні адреси
- Розуміти SOL і лампорти
- Використовувати web3.js для підключення до Solana та зчитування балансу акаунту
---

## Стислий виклад 

* **SOL** — це назва рідного токена Solana. Кожен SOL складається з 1 мільярда **лампортів**.
* **Акаунти** зберігають токени, NFT, програми та дані. Наразі ми зосередимося на акаунтах, які зберігають SOL.
* **Адреси** вказують на акаунти в мережі Solana. Будь-хто може прочитати дані за певною адресою. Більшість адрес також є **публічними ключами**.

# Урок

### Акаунти

Всі дані, що зберігаються в Solana, зберігаються в акаунтах. Акаунти можуть зберігати:

* SOL
* Інші токени, як-от USDC
* NFT
* Програми, наприклад програму для рецензій на фільми, яку ми створюємо в цьому курсі!
* Дані програм, наприклад рецензію на фільм для вищезгаданої програми!

### SOL

SOL — це рідний токен Solana. SOL використовується для оплати комісій за транзакції, оренди акаунтів та іншого. Іноді SOL позначають символом `◎`. Кожна SOL складається з 1 мільярда **лампортів**.

Так само, як фінансові додатки зазвичай виконують обчислення в центах (для USD) чи пенсах (для GBP), додатки Solana зазвичай переказують, витрачають, зберігають і опрацьовують SOL у лампортах, а для відображення користувачам конвертують у повні SOL.

### Адреси

Адреси унікально ідентифікують акаунти. Адреси часто показують як рядки, закодовані у форматі base-58, наприклад `dDCQNnDmNbFVi8cQhKAgXhyhXeJ625tvwsunRyRc7c8`. Більшість адрес у Solana також є **публічними ключами**. Як було згадано в попередньому розділі, той, хто контролює відповідний секретний ключ для адреси, контролює акаунт — наприклад, ця людина може відправляти токени з акаунту.

## Зчитування з блокчейну Solana

### Встановлення

Ми використовуємо npm-пакет `@solana/web3.js` для більшості операцій з Solana. Також встановимо TypeScript і `esrun`, щоб запускати `.ts` файли з командного рядка:

```bash
npm install typescript @solana/web3.js esrun 
```

### Підключення до мережі

Усі взаємодії з мережею Solana за допомогою `@solana/web3.js` відбуваються через об’єкт `Connection`. Об’єкт `Connection` встановлює з’єднання з конкретною мережею Solana, яка називається «кластером». Зараз ми використовуватимемо кластер `Devnet`, а не `Mainnet`. `Devnet` призначений для розробників і тестування, а токени в `Devnet` не мають реальної цінності.

```typescript
import { Connection, clusterApiUrl } from "@solana/web3.js";

const connection = new Connection(clusterApiUrl("devnet"));
console.log(`✅ Connected!`)
```

Запуск цього TypeScript-коду (`npx esrun example.ts`) показує:

```
✅ Connected!
```

### Зчитування з мережі

Щоб зчитати баланс акаунту:

```typescript
import { Connection, PublicKey, clusterApiUrl } from "@solana/web3.js";

const connection = new Connection(clusterApiUrl("devnet"));
const address = new PublicKey('CenYq6bDRB7p73EjsPEpiYN7uveyPUTdXkDkgUduboaN');
const balance = await connection.getBalance(address);

console.log(`The balance of the account at ${address} is ${balance} lamports`); 
console.log(`✅ Finished!`)
```

Повернутий баланс вказаний у *лампортах, як уже згадувалося раніше. Web3.js надає константу `LAMPORTS_PER_SOL` для відображення лампортів у вигляді SOL:

```typescript
import { Connection, PublicKey, clusterApiUrl, LAMPORTS_PER_SOL } from "@solana/web3.js";

const connection = new Connection(clusterApiUrl("devnet"));
const address = new PublicKey('CenYq6bDRB7p73EjsPEpiYN7uveyPUTdXkDkgUduboaN');
const balance = await connection.getBalance(address);
const balanceInSol = balance / LAMPORTS_PER_SOL;

console.log(`The balance of the account at ${address} is ${balanceInSol} SOL`); 
console.log(`✅ Finished!`)
```

Запуск `npx esrun example.ts` покаже щось на кшталт:

```
The balance of the account at CenYq6bDRB7p73EjsPEpiYN7uveyPUTdXkDkgUduboaN is 0.00114144 SOL
✅ Finished!
```

...і от так просто ми зчитуємо дані з блокчейну Solana!

# Лабораторна робота 

Давайте закріпимо вивчене та перевіримо баланс за конкретною адресою.

## Завантаження пари ключів

Згадайте публічний ключ з попереднього розділу.

Створіть новий файл з назвою `check-balance.ts`, підставивши свій публічний ключ замість `<your public key>`.

Скрипт завантажує публічний ключ, підключається до DevNet і перевіряє баланс:

```tsx
import { Connection, LAMPORTS_PER_SOL, PublicKey } from "@solana/web3.js";

const publicKey = new PublicKey("<your public key>");

const connection = new Connection("https://api.devnet.solana.com", "confirmed");

const balanceInLamports = await connection.getBalance(publicKey);

const balanceInSOL = balanceInLamports / LAMPORTS_PER_SOL;

console.log(
  `💰 Finished! The balance for the wallet at address ${publicKey} is ${balanceInSOL}!`
);

```

Збережіть це у файл і запустіть команду `npx esrun check-balance.ts`. Ви побачите щось на кшталт:

```
💰 Finished! The balance for the wallet at address 31ZdXAvhRQyzLC2L97PC6Lnf2yWgHhQUKKYoUo9MLQF5 is 0!
```

## Отримайте SOL у Devnet

У Devnet ви можете отримати безкоштовні SOL для розробки. Уявіть собі Devnet SOL як ігрові гроші — вони виглядають як справжні, але не мають цінності.

[Отримайте Devnet SOL](https://faucet.solana.com/), використавши публічний ключ вашої пари ключів як адресу.

Виберіть будь-яку кількість SOL, яку бажаєте.

## Перевірте свій баланс

Повторно запустіть скрипт. Ви повинні побачити оновлений баланс:

```
💰 Finished! The balance for the wallet at address 31ZdXAvhRQyzLC2L97PC6Lnf2yWgHhQUKKYoUo9MLQF5 is 0.5!
```

## Перевірка балансів інших студентів

Ви можете змінити скрипт, щоб перевіряти баланси будь-якого гаманця.

```tsx
import { Connection, LAMPORTS_PER_SOL, PublicKey } from "@solana/web3.js";

const suppliedPublicKey = process.argv[2];
if (!suppliedPublicKey) {
  throw new Error("Provide a public key to check the balance of!");
}

const connection = new Connection("https://api.devnet.solana.com", "confirmed");

const publicKey = new PublicKey(suppliedPublicKey);

const balanceInLamports = await connection.getBalance(publicKey);

const balanceInSOL = balanceInLamports / LAMPORTS_PER_SOL;

console.log(
  `✅ Finished! The balance for the wallet at address ${publicKey} is ${balanceInSOL}!`
);

```

Поміняйтеся адресами гаманців зі своїми одногрупниками в чаті та перевірте їхні баланси.

```bash
% npx esrun check-balance.ts (some wallet address)
✅ Finished! The balance for the wallet at address 31ZdXAvhRQyzLC2L97PC6Lnf2yWgHhQUKKYoUo9MLQF5 is 3!
```

Перевірте баланс кількох своїх одногрупників.

# Завдання

Змініть скрипт так:

* Додайте обробку некоректних адрес гаманців.
* Змініть підключення до `mainNet` і перевірте кілька відомих Solana-гаманців, наприклад `toly.sol`, `shaq.sol` або `mccann.sol`.

У наступному уроці ми будемо переказувати SOL!

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=8bbbfd93-1cdc-4ce3-9c83-637e7aa57454)!
