---
назва: Розширення Close Mint
завдання:
 - Створити мінт, який можна закрити
 - Описати всі необхідні передумови для закриття мінт-акаунту
---

# Стислий виклад
 - Оригінальна Token Program дозволяла закривати лише токен-акаунти, але не мінт-акаунти.
 - Програма Token Extensions включає розширення `close mint`, яке дозволяє закривати мінт акаунти.
 - Для того, щоб закрити мінт за допомогою розширення `close mint`, кількість токенів у цьому мінті повинна бути 0.
 - `mintCloseAuthority` можна оновити, викликавши функцію `setAuthority`.

# Огляд

Оригінальна програма Token Program дозволяє власникам закривати лише токен-акаунти, але не мінт-акаунти. Тому, якщо ви створюєте мінт, ви ніколи не зможете закрити його акаунт. Це призвело до значних витрат простору на блокчейні. Щоб вирішити цю проблему, у програмі Token Extensions було впроваджено розширення `close mint`. Воно дозволяє закривати мінт-акаунт та повернути лампорти. Єдина умова — пропозиція цього мінта має бути рівним 0.

Це розширення є зручним удосконаленням для розробників, які можуть мати тисячі мінт-акаунтів, що підлягають видаленню, дозволяючи їм повернути кошти. Крім того, це чудова новина для власників NFT, які хочуть спалити свої NFT. Тепер вони зможуть повернути всі витрати, тобто закриття мінт-акаунту, метаданих і токен-акаунтів. Раніше, якщо хтось спалював NFT, він міг повернути лише ренту метаданих і токен-акаунту. Зазначимо, що той, хто спалює NFT, також має бути `mintCloseAuthority`.

Розширення `close mint` додає до мінт-акаунту додаткове поле `mintCloseAuthority`. Це адреса авторитету, який може фактично закрити акаунт.

Наголошую, щоб закрити мінт-акаунт за допомогою цього розширення, його пропозиція (Supply) має бути рівна 0. Тому, якщо будь-яка частина цього токена була створена, її спершу необхідно спалити.

## Створення мінт-акаунту з Close Authority (Правом Закриття)

Ініціалізація мінт-акаунту з розширенням close authority включає виконання трьох інструкцій:
 - `SystemProgram.createAccount` 
 - `createInitializeMintCloseAuthorityInstruction`
 - `createInitializeMintInstruction`

Перша інструкція `SystemProgram.createAccount` виділяє простір у блокчейні для мінт-акаунту. Однак, як і для всіх мінт-акаунтів Token Extensions Program, нам потрібно розрахувати розмір і вартість мінт-акаунту. Це можна зробити за допомогою функцій `getMintLen` та `getMinimumBalanceForRentExemption`. У цьому випадку ми викликаємо `getMintLen` із параметром `ExtensionType.MintCloseAuthority`.

Щоб отримати розмір мінт-акаунту та інструкцію для створення акаунту, виконайте наступне:
```ts
const extensions = [ExtensionType.MintCloseAuthority]
const mintLength = getMintLen(extensions)

const mintLamports =
	await connection.getMinimumBalanceForRentExemption(mintLength)

const createAccountInstruction = SystemProgram.createAccount({
	fromPubkey: payer,
	newAccountPubkey: mint,
	space: mintLength,
	lamports: mintLamports,
	programId: TOKEN_2022_PROGRAM_ID,
})
```

Друга інструкція `createInitializeMintCloseAuthorityInstruction` ініціалізує розширення close authority. Єдиним важливим параметром є `mintCloseAuthority` на другій позиції. Це адреса, яка має право закривати мінт-акаунт.

```ts

const initializeMintCloseAuthorityInstruction = createInitializeMintCloseAuthorityInstruction(
	mint,
	authority,
	TOKEN_2022_PROGRAM_ID
)
```

Остання інструкція `createInitializeMintInstruction` ініціалізує мінт-акаунт.
```ts
const initializeMintInstruction = createInitializeMintInstruction(
	mint,
	decimals,
	payer.publicKey,
	null,
	TOKEN_2022_PROGRAM_ID
)
```

Останнім кроком ми додаємо інструкції до транзакції і відправляємо її в мережу Solana.
```typescript
const mintTransaction = new Transaction().add(
	createAccountInstruction,
	initializeMintCloseAuthorityInstruction,
	initializeMintInstruction,
)

const signature = await sendAndConfirmTransaction(
	connection,
	mintTransaction,
	[payer, mintKeypair],
	{ commitment: 'finalized' }
)
```

Коли транзакція відправляється, створюється новий мінт-акаунт з вказаним close authority.

## Закриття мінт-акаунту з Close Authority

Щоб закрити мінт-акаунт з розширенням `close mint`, потрібно лише викликати функцію `closeAccount`.

Не забувайте, що для закриття мінт-акаунту загальний обсяг (supply) повинен бути рівним 0. Тому, якщо існують токени, їх потрібно спочатку спалити. Це можна зробити за допомогою функції `burn`.

Примітка: Функція `closeAccount` працює як для mint-акаунтів, так і для токен-акаунтів.

```ts
// burn tokens to 0
const burnSignature = await burn(
	connection, // connection - Connection to use
	payer, // payer -Payer of the transaction fees
	sourceAccount, // account - Account to burn tokens from 
	mintKeypair.publicKey, // mint - Mint for the account
	sourceKeypair, // owner - Account owner
	sourceAccountInfo.amount, // amount -  Amount to burn
	[], // multiSigners - Signing accounts if `owner` is a multisig
	{ commitment: 'finalized' }, // confirmOptions - Options for confirming the transaction
	TOKEN_2022_PROGRAM_ID // programId - SPL Token program account
)

// account can be closed as total supply is now 0
await closeAccount(
	connection, // connection - Connection to use
	payer, // payer - Payer of the transaction fees
	mintKeypair.publicKey, // account - Account to close
	payer.publicKey, // destination - Account to receive the remaining balance of the closed account
	payer, // authority - Authority which is allowed to close the account
	[], // multiSigners - Signing accounts if `authority` is a multisig
	{ commitment: 'finalized' }, // confirmOptions - Options for confirming the transaction
	TOKEN_2022_PROGRAM_ID // programIdSPL Token program account
)
```

## Оновлення Close Mint Authority

Щоб змінити `closeMintAuthority`, можна викликати функцію `setAuthority` та передати правильні акаунти, а також `authorityType`, у цьому випадку — `AuthorityType.CloseMint`.

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
  AuthorityType.CloseMint,
  newAuthority, 
  [],
  undefined,
  TOKEN_2022_PROGRAM_ID
)
```

# Лабораторна робота

У цій лабораторній роботі ми створимо мінт-акаунт з розширенням `close mint`. Потім ми випустимо кілька токенів і подивимося, що станеться, коли ми спробуємо закрити його з ненульовим балансом (підказка: транзакція закриття зазнає невдачі). Наприкінці ми спалимо баланс і закриємо акаунт.

## 1. Початок роботи

Для початку створіть порожню директорію з назвою `close-mint` і перейдіть до неї. Ми будемо ініціалізувати новий проект. Запустіть команду `npm init` і дотримуйтесь підказок.

Далі нам потрібно додати залежності. Виконайте наступну команду, щоб встановити необхідні пакети:
```bash
npm i @solana-developers/helpers @solana/spl-token @solana/web3.js esrun dotenv typescript
```

Створіть директорію з назвою `src`. У цій директорії створіть файл з ім'ям `index.ts`. Тут ми будемо перевіряти правила цієї розширеної функціональності. Вставте наступний код у файл `index.ts`:

```ts
import {
	Connection,
	Keypair,
	LAMPORTS_PER_SOL,
} from '@solana/web3.js'
import { initializeKeypair } from '@solana-developers/helpers'
// import { createClosableMint } from './create-mint' // - uncomment this in a later step
import {
	TOKEN_2022_PROGRAM_ID,
	burn,
	closeAccount,
	createAccount,
	getAccount,
	getMint,
	mintTo,
} from '@solana/spl-token'
import dotenv from 'dotenv'
dotenv.config()

/**
 * Create a connection and initialize a keypair if one doesn't already exists.
 * If a keypair exists, airdrop a SOL if needed.
 */
const connection = new Connection("http://127.0.0.1:8899")
const payer = await initializeKeypair(connection)

console.log(`public key: ${payer.publicKey.toBase58()}`)

const mintKeypair = Keypair.generate()
const mint = mintKeypair.publicKey
console.log(
	'\nmint public key: ' + mintKeypair.publicKey.toBase58() + '\n\n'
)

// CREATE A MINT WITH CLOSE AUTHORITY

// MINT TOKEN

// VERIFY SUPPLY

// TRY CLOSING WITH NON ZERO SUPPLY

// BURN SUPPLY

// CLOSE MINT
```

`index.ts` створює підключення до вказаного валідатора та викликає `initializeKeypair`. Це місце, де ми викликаємо решту нашого скрипту, коли він буде написаний.

Запустіть скрипт. Ви повинні побачити публічні ключі `payer` та `mint`, виведені у вашому терміналі.

```bash
npx esrun src/index.ts
```

Якщо ви зіткнулися з помилкою в `initializeKeypair` під час аірдропу, виконайте наступний крок.

### 2. Запустіть ноду валідатора

Для цієї інструкції ми будемо запускати власну ноду валідатора.

У окремому терміналі виконайте наступну команду: `solana-test-validator`. Це запустить ноду та виведе деякі ключі й значення. Значення, яке нам потрібно отримати та використати для підключення, — це JSON RPC URL, в даному випадку `http://127.0.0.1:8899`. Потім ми використовуємо це у підключенні, щоб вказати використання локального RPC URL.

```tsx
const connection = new Connection("http://127.0.0.1:8899", "confirmed");
```

Альтернативно, якщо ви хочете використовувати testnet або devnet, імпортуйте `clusterApiUrl` з `@solana/web3.js` і передайте його до підключення наступним чином:

```tsx
const connection = new Connection(clusterApiUrl('devnet'), 'confirmed');
```

## 3. Створення мінт-акаунту з Close Authority

Давайте створимо мінт-акаунт з можливістю закриття, створивши функцію `createClosableMint` у новому файлі `src/create-mint.ts`.

Щоб створити мінт-акаунт з можливістю закриття, нам потрібно виконати кілька інструкцій.: 

- `getMintLen`: Отримує необхідний простір для мінт-акаунту
- `SystemProgram.getMinimumBalanceForRentExemption`: Показує вартість оренди мінт-акаунту
- `SystemProgram.createAccount`: Створює інструкцію для виділення простору на Solana для мінт-акаунту
- `createInitializeMintCloseAuthorityInstruction`: Створює інструкцію для ініціалізації розширення close mint - цей параметр приймає `closeMintAuthority`.
- `createInitializeMintInstruction`: Створює інструкцію для ініціалізації мінт-акаунту
- `sendAndConfirmTransaction`: Відправляє транзакцію в блокчейн

Ми викличемо всі ці функції по черзі. Але перед цим давайте визначимо вхідні параметри для нашої функції `createClosableMint`:
- `connection: Connection`: Об'єкт з'єднання
- `payer: Keypair`: Платник транзакції
- `mintKeypair: Keypair`: Пара ключів для нового мінт-акаунту
- `decimals: number`: Десяткові знаки мінт-акаунту


Склавши все разом ми отримаємо:
```ts
import {
	sendAndConfirmTransaction,
	Connection,
	Keypair,
	SystemProgram,
	Transaction,
	TransactionSignature,
} from '@solana/web3.js'

import {
	ExtensionType,
	createInitializeMintInstruction,
	getMintLen,
	TOKEN_2022_PROGRAM_ID,
	createInitializeMintCloseAuthorityInstruction,
} from '@solana/spl-token'

export async function createClosableMint(
	connection: Connection,
	payer: Keypair,
	mintKeypair: Keypair,
	decimals: number
): Promise<TransactionSignature> {
	const extensions = [ExtensionType.MintCloseAuthority]
	const mintLength = getMintLen(extensions)

	const mintLamports =
		await connection.getMinimumBalanceForRentExemption(mintLength)

	console.log('Creating a transaction with close mint instruction...')
	const mintTransaction = new Transaction().add(
		SystemProgram.createAccount({
			fromPubkey: payer.publicKey,
			newAccountPubkey: mintKeypair.publicKey,
			space: mintLength,
			lamports: mintLamports,
			programId: TOKEN_2022_PROGRAM_ID,
		}),
		createInitializeMintCloseAuthorityInstruction(
			mintKeypair.publicKey,
			payer.publicKey,
			TOKEN_2022_PROGRAM_ID
		),
		createInitializeMintInstruction(
			mintKeypair.publicKey,
			decimals,
			payer.publicKey,
			null,
			TOKEN_2022_PROGRAM_ID
		)
	)

	console.log('Sending transaction...')
	const signature = await sendAndConfirmTransaction(
		connection,
		mintTransaction,
		[payer, mintKeypair],
		{ commitment: 'finalized' }
	)

	return signature
}
```

Тепер давайте викличемо цю функцію в `src/index.ts`. Спочатку потрібно імпортувати нашу нову функцію. Потім вставити наступний код під відповідним коментарем:

```ts
// CREATE A MINT WITH CLOSE AUTHORITY
const decimals = 9

await createClosableMint(connection, payer, mintKeypair, decimals)
```

Це створить транзакцію з інструкцією для закриття мінт-акаунту.

Не соромтеся запустити це, та перевірити, чи все працює:

```bash
npx esrun src/index.ts
```

## 4. Закриття мінт-акаунту

Ми збираємось закрити мінт-акаунт, але спочатку давайте подивимось, що трапиться, якщо в нас є запас токенів при спробі закрити (підказка: це не вийде).

Для цього ми будемо мінтити кілька токенів, спробуємо закрити, потім спалимо токени та нарешті справді закриємо мінт.

### 4.1 Мінт токена  
У файлі `src/index.ts` створіть акаунт і змінтіть 1 токен на цей акаунт.

Ми можемо досягти цього за допомогою двох функцій: `createAccount` і `mintTo`:
```ts
// MINT TOKEN
/**
 * Creating an account and mint 1 token to that account
*/
console.log('Creating an account...')
const sourceKeypair = Keypair.generate()
const sourceAccount = await createAccount(
	connection,
	payer,
	mint,
	sourceKeypair.publicKey,
	undefined,
	{ commitment: 'finalized' },
	TOKEN_2022_PROGRAM_ID
)

console.log('Minting 1 token...\n\n')
const amount = 1 * LAMPORTS_PER_SOL
await mintTo(
	connection,
	payer,
	mint,
	sourceAccount,
	payer,
	amount,
	[payer],
	{ commitment: 'finalized' },
	TOKEN_2022_PROGRAM_ID
)
```

Тепер ми можемо перевірити, що запас токенів у мінт-акаунті не дорівнює нулю, отримавши інформацію про мінт-акаунт. Під блоком функцій для мінта додайте наступний блок коду:

```ts
// VERIFY SUPPLY
/**
 * Get mint information to verify supply
*/
const mintInfo = await getMint(
	connection,
	mintKeypair.publicKey,
	'finalized',
	TOKEN_2022_PROGRAM_ID
)
console.log("Initial supply: ", mintInfo.supply)
```

Запустимо скрипт і перевіримо початкову кількість токенів:

```bash
npx esrun src/index.ts
```

Ви повинні побачити наступне у вашому терміналі:

```bash
Initial supply:  1000000000n
```

### 4.2 Закриття мінт-акаунту з ненульовою пропозицією

Тепер ми спробуємо закрити мінт-акаунт, коли пропозиція (supply) не дорівнює нулю. Ми знаємо, що це не спрацює, оскільки розширення `close mint` вимагає нульової пропозиції. Тож, щоб побачити повідомлення про помилку, ми обгорнемо функцію `closeAccount` у блок `try catch` і виведемо помилку в лог:
```ts
// TRY CLOSING WITH NON ZERO SUPPLY
/**
 * Try closing the mint account when supply is not 0
 *
 * Should throw `SendTransactionError`
*/
try {
	await closeAccount(
		connection,
		payer,
		mintKeypair.publicKey,
		payer.publicKey,
		payer,
		[],
		{ commitment: 'finalized' },
		TOKEN_2022_PROGRAM_ID
	)
} catch (e) {
	console.log(
		'Close account fails here because the supply is not zero. Check the program logs:',
		(e as any).logs,
		'\n\n'
	)
}
```

Запустіть:
```bash
npx esrun src/index.ts
```

Ми побачимо, що програма виведе помилку разом з логами. Ви повинні побачити наступне::

```bash
Close account fails here because the supply is not zero.
```

### 4.3 Спалення балансу

Тепер ми спалимо всю пропозицію токенів, щоб мати можливість закрити мінт-акаунт. Для цього ми використовуємо функцію `burn`.:

```ts
// BURN SUPPLY
const sourceAccountInfo = await getAccount(
	connection,
	sourceAccount,
	'finalized',
	TOKEN_2022_PROGRAM_ID
)

console.log('Burning the supply...')
const burnSignature = await burn(
	connection,
	payer,
	sourceAccount,
	mintKeypair.publicKey,
	sourceKeypair,
	sourceAccountInfo.amount,
	[],
	{ commitment: 'finalized' },
	TOKEN_2022_PROGRAM_ID
)
```

### 4.4 Закриття мінт-акаунту
Оскільки токени більше не в обігу, ми можемо тепер закрити мінт-акаунт. На цьому етапі ми можемо просто викликати функцію `closeAccount`. Однак, для кращого розуміння цього процесу, ми зробимо наступне:

    - Отримання інформації про мінт-акаунт: Спочатку ми отримуємо та перевіряємо деталі мінт-акаунту, особливо звертаючи увагу на пропозицію (supply), яка на цьому етапі повинна дорівнювати нулю. Це підтверджує, що мінт-акаунт можна закрити.  

    - Перевірка стану акаунту: Далі ми підтверджуємо, що акаунт все ще відкритий і активний.  

    - Закриття акаунту: Після підтвердження, що акаунт відкритий, ми здійснюємо його закриття.

    - Підтвердження закриття: Нарешті, після виклику функції `closeAccount`, ми ще раз перевіряємо стан акаунту, щоб підтвердити, що він був успішно закритий.

Ми можемо виконати всі ці дії за допомогою таких функцій:  
- `getMint`: Отримує мінт-акаунт і десеріалізує його дані.  
- `getAccountInfo`: Отримує мінт-акаунт, щоб перевірити його існування — ми викличемо цю функцію до та після закриття.  
- `closeAccount`: Закриває мінт-акаунт.  

Об’єднуючи все це, отримуємо:

```ts
// CLOSE MINT
const mintInfo = await getMint(
	connection,
	mintKeypair.publicKey,
	'finalized',
	TOKEN_2022_PROGRAM_ID
)

console.log("After burn supply: ", mintInfo.supply)

const accountInfoBeforeClose = await connection.getAccountInfo(mintKeypair.publicKey, 'finalized');

console.log("Account closed? ", accountInfoBeforeClose === null)

console.log('Closing account after burning the supply...')
const closeSignature = await closeAccount(
	connection,
	payer,
	mintKeypair.publicKey,
	payer.publicKey,
	payer,
	[],
	{ commitment: 'finalized' },
	TOKEN_2022_PROGRAM_ID
)

const accountInfoAfterClose = await connection.getAccountInfo(mintKeypair.publicKey, 'finalized');

console.log("Account closed? ", accountInfoAfterClose === null)
```

Запустіть скрипт ще раз.

```bash
npx esrun src/index.ts
```

Ви повинні побачити весь процес: створення мінт-акаунту, який можна закрити, мінт токена, спробу закриття, спалення токена та, нарешті, закриття акаунту.

Це все! Ми успішно створили мінт з правом закриття. Якщо ви застрягли на будь-якому етапі, ви можете знайти робочий код у гілці `solution` [цього репозиторію](https://github.com/Unboxed-Software/solana-lab-close-mint-account/tree/solution).

# Завдання
Спробуйте створити власний мінт-акаунт, здійснити мінти на кілька токен-акаунтів, а потім написати скрипт, який спалить усі ці токени та закриє мінт-акаунт.
