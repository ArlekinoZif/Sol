---
title: Вступ до розробки клієнтської частини з Anchor
завдання:
- Використовувати IDL для взаємодії з програмою Solana з клієнта  
- Пояснити об'єкт Anchor `Provider`  
- Пояснити об'єкт Anchor `Program`  
- Використовувати `MethodsBuilder` в Anchor для створення інструкцій і транзакцій  
- Використовувати Anchor для отримання акаунтів  
- Налаштувати фронтенд для виклику інструкцій за допомогою Anchor і IDL
---

# Стислий виклад

- **IDL** — це файл, що представляє структуру Solana-програми. Програми, написані та зібрані з використанням Anchor, автоматично генерують відповідний IDL. IDL розшифровується як Interface Description Language (Мова Опису Інтерфейсу)
- `@coral-xyz/anchor` — це Typescript-клієнт, який містить усе необхідне для взаємодії з Anchor-програмами.  
- Об'єкт **Anchor `Provider`** поєднує `connection` до кластера та вказаний `wallet`, щоб забезпечити підписання транзакцій.  
- Об'єкт **Anchor `Program`** надає спеціальний API для взаємодії з конкретною програмою. Ви створюєте екземпляр `Program`, використовуючи IDL програми та `Provider`.  
- **Anchor `MethodsBuilder`** надає простий інтерфейс через `Program` для створення інструкцій та транзакцій.
  
# Урок

Anchor спрощує процес взаємодії з Solana-програмами з боку клієнта, надаючи файл Interface Description Language (IDL), що відображає структуру програми. Використання IDL разом із Typescript-бібліотекою Anchor (`@coral-xyz/anchor`) забезпечує спрощений формат для створення інструкцій та транзакцій.

```tsx
// sends transaction
await program.methods
  .instructionName(instructionDataInputs)
  .accounts({})
  .signers([])
  .rpc()
```

Це працює з будь-якого Typescript-клієнта, незалежно від того, чи це фронтенд, чи інтеграційні тести. У цьому уроці ми розглянемо, як використовувати `@coral-xyz/anchor` для спрощення взаємодії вашої програми з клієнтської сторони.

## Структура клієнтської частини Anchor

Почнемо з основної структури Typescript-бібліотеки Anchor. Основним об'єктом, який ви будете використовувати, є об'єкт `Program`. Екземпляр `Program` представляє конкретну Solana-програму і надає спеціальний API для читання та запису до програми.

Щоб створити екземпляр `Program`, вам знадобляться наступні елементи:

- IDL — файл, що представляє структуру програми
- `Connection` — з'єднання з кластером
- `Wallet` — пара ключів за замовчуванням, що використовується для оплати та підписання транзакцій
- `Provider` — об'єднує `Connection` до Solana-кластера та `Wallet`
- `ProgramId` — ончейн-адреса програми

![Anchor structure](../assets/anchor-client-structure.png)

Вищезгадане зображення показує, як кожен з цих елементів об'єднується для створення екземпляра `Program`. Ми розглянемо кожен з них окремо, щоб краще зрозуміти, як все взаємодіє між собою.

### Interface Description Language (IDL)

Коли ви будуєте програму на Anchor, Anchor генерує як JSON, так і Typescript файл, що представляє IDL вашої програми. IDL відображає структуру програми і може бути використаний клієнтом для визначення, як взаємодіяти з конкретною програмою.

Хоча це не є автоматичним процесом, ви також можете згенерувати IDL для нативної Solana-програми за допомогою інструментів, таких як [shank](https://github.com/metaplex-foundation/shank) від Metaplex.

Щоб зрозуміти, яку інформацію надає IDL, ось IDL для програми для підрахунку, яку ви створювали раніше:

```json
{
  "version": "0.1.0",
  "name": "counter",
  "instructions": [
    {
      "name": "initialize",
      "accounts": [
        { "name": "counter", "isMut": true, "isSigner": true },
        { "name": "user", "isMut": true, "isSigner": true },
        { "name": "systemProgram", "isMut": false, "isSigner": false }
      ],
      "args": []
    },
    {
      "name": "increment",
      "accounts": [
        { "name": "counter", "isMut": true, "isSigner": false },
        { "name": "user", "isMut": false, "isSigner": true }
      ],
      "args": []
    }
  ],
  "accounts": [
    {
      "name": "Counter",
      "type": {
        "kind": "struct",
        "fields": [{ "name": "count", "type": "u64" }]
      }
    }
  ]
}
```

Переглядаючи IDL, можна побачити, що ця програма містить дві інструкції (`initialize` та `increment`).

Зверніть увагу, що окрім вказання інструкцій, також вказуються акаунти та вхідні дані для кожної інструкції. Інструкція `initialize` вимагає три акаунти:

1. `counter` — новий акаунт, що ініціалізується в інструкції  
2. `user` — платник за транзакцію та ініціалізацію  
3. `systemProgram` — системна програма, яка викликається для ініціалізації нового акаунту

Інструкція `increment` вимагає два акаунти:  

1. `counter` – існуючий акаунт, у якому збільшується поле `count`.  
2. `user` – платник у транзакції.  

Переглядаючи IDL, можна побачити, що в обох інструкціях `user` є обов'язковим підписантом, оскільки прапорець `isSigner` встановлений на `true`. Крім того, жодна з інструкцій не вимагає додаткових даних інструкції, оскільки розділ `args` порожній для обох.

Далі в розділі `accounts` можна побачити, що програма містить один тип акаунту з іменем `Counter`, що має єдине поле `count` типу `u64`.

Хоча IDL не надає деталей виконання для кожної інструкції, ми можемо отримати загальне уявлення про те, як очікується, що інструкції будуть побудовані для ончейн-програми, і побачити структуру акаунтів програми.

Незалежно від того, як ви його отримуєте, вам *потрібен* файл IDL для взаємодії з програмою за допомогою пакету `@coral-xyz/anchor`. Щоб використовувати IDL, вам потрібно включити файл IDL у ваш проєкт, а потім імпортувати його. 

```tsx
import idl from "./idl.json"
```

### Provider

Перед тим як створити об'єкт `Program` за допомогою IDL, вам спочатку потрібно створити об'єкт Anchor `Provider`.

Об'єкт `Provider` поєднує дві речі:

- `Connection` — з'єднання з Solana-кластером (наприклад, localhost, devnet, mainnet)
- `Wallet` — вказана адреса, що використовується для оплати та підписання транзакцій

Об'єкт `Provider` може надсилати транзакції до блокчейну Solana від імені `Wallet`, додаючи підпис гаманця до вихідних транзакцій. Коли використовуєте фронтенд із провайдером Solana-гаманця, усі вихідні транзакції повинні бути схвалені користувачем через його розширення гаманця в браузері.

Налаштування `Wallet` та `Connection` виглядатиме приблизно так:

```tsx
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react"

const { connection } = useConnection()
const wallet = useAnchorWallet()
```

Щоб налаштувати з'єднання, ви можете використовувати хук `useConnection` з `@solana/wallet-adapter-react`, щоб отримати `Connection` до Solana-кластера.

Зверніть увагу, що об'єкт `Wallet`, наданий хуком `useWallet` з `@solana/wallet-adapter-react`, не сумісний з об'єктом `Wallet`, який очікує Anchor `Provider`. Однак, `@solana/wallet-adapter-react` також надає хук `useAnchorWallet`.

Для порівняння, ось `AnchorWallet` з хука `useAnchorWallet`:

```tsx
export interface AnchorWallet {
  publicKey: PublicKey
  signTransaction(transaction: Transaction): Promise<Transaction>
  signAllTransactions(transactions: Transaction[]): Promise<Transaction[]>
}
```

А ось `WalletContextState` з хука `useWallet`:

```tsx
export interface WalletContextState {
  autoConnect: boolean
  wallets: Wallet[]
  wallet: Wallet | null
  publicKey: PublicKey | null
  connecting: boolean
  connected: boolean
  disconnecting: boolean
  select(walletName: WalletName): void
  connect(): Promise<void>
  disconnect(): Promise<void>
  sendTransaction(
    transaction: Transaction,
    connection: Connection,
    options?: SendTransactionOptions
  ): Promise<TransactionSignature>
  signTransaction: SignerWalletAdapterProps["signTransaction"] | undefined
  signAllTransactions:
    | SignerWalletAdapterProps["signAllTransactions"]
    | undefined
  signMessage: MessageSignerWalletAdapterProps["signMessage"] | undefined
}
```

`WalletContextState` надає значно більше функціональності порівняно з `AnchorWallet`, але для налаштування об'єкта `Provider` необхідно використовувати саме `AnchorWallet`.

Для створення об'єкта `Provider` використовується `AnchorProvider` з `@coral-xyz/anchor`.

Конструктор `AnchorProvider` приймає три параметри:

- `connection` — з'єднання з Solana-кластером
- `wallet` — об'єкт `Wallet`
- `opts` — необов'язковий параметр, що вказує параметри підтвердження, з використанням налаштувань за замовчуванням, якщо вони не надані

Після створення об'єкта `Provider`, ви можете встановити його як стандартний провайдер за допомогою `setProvider`.

```tsx
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react"
import { AnchorProvider, setProvider } from "@coral-xyz/anchor"

const { connection } = useConnection()
const wallet = useAnchorWallet()
const provider = new AnchorProvider(connection, wallet, {})
setProvider(provider)
```

### Program

Якщо у вас є IDL та провайдер, ви можете створити екземпляр `Program`. Конструктор вимагає три параметри:

- `idl` — IDL як тип `Idl`
- `programId` — ончейн-адреса програми у вигляді `string` або `PublicKey`
- `Provider` — провайдер, розглянутий у попередньому розділі
  
Об'єкт `Program` створює спеціалізований API, за допомогою якого ви можете взаємодіяти з Solana-програмою. Цей API є універсальним інструментом для всього, що стосується комунікації з ончейн-програмами. Серед іншого, ви можете надсилати транзакції, отримувати десеріалізовані акаунти, декодувати дані інструкцій, підписуватися на зміни акаунтів і слухати події. Ви також можете [дізнатися більше про клас `Program`](https://coral-xyz.github.io/anchor/ts/classes/Program.html#constructor).

Щоб створити об'єкт `Program`, спочатку імпортуйте `Program` та `Idl` з `@coral-xyz/anchor`. `Idl` — це тип, який можна використовувати при роботі з Typescript.

Далі вкажіть `programId` програми. Ми повинні явно вказати `programId`, оскільки може бути кілька програм з однаковою структурою IDL (наприклад, якщо одна і та сама програма розгорнута кілька разів за різними адресами). При створенні об'єкта `Program` буде використано стандартний `Provider`, якщо інший не вказано.

Весь процес налаштування виглядатиме приблизно так:

```tsx
import idl from "./idl.json"
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react"
import {
  Program,
  Idl,
  AnchorProvider,
  setProvider,
} from "@coral-xyz/anchor"

const { connection } = useConnection()
const wallet = useAnchorWallet()

const provider = new AnchorProvider(connection, wallet, {})
setProvider(provider)

const programId = new PublicKey("JPLockxtkngHkaQT5AuRYow3HyUv5qWzmhwsCPd653n")
const program = new Program(idl as Idl, programId)
```

## Anchor `MethodsBuilder`

Після налаштування об'єкта `Program` ви можете використовувати Anchor Methods Builder для створення інструкцій та транзакцій, що стосуються програми. `MethodsBuilder` використовує IDL, щоб забезпечити спрощений формат для побудови транзакцій, які викликають інструкції програми.

Зверніть увагу, що для взаємодії з програмою з клієнтської сторони використовується стилістика іменування в camel case, порівняно з стилістикою snake case, яка використовується при написанні програми на Rust.

Основний формат `MethodsBuilder` виглядатиме так:

```tsx
// sends transaction
await program.methods
  .instructionName(instructionDataInputs)
  .accounts({})
  .signers([])
  .rpc()
```

Крок за кроком, ви:

1. Викликаєте `methods` за допомогою `program` — це API побудови для створення викликів інструкцій, що стосуються IDL програми.
2. Викликаєте ім'я інструкції як `.instructionName(instructionDataInputs)` — просто викликаєте інструкцію за допомогою синтаксису крапки та імені інструкції, передаючи будь-які аргументи інструкції як значення, розділені комами.
3. Викликаєте `accounts` — за допомогою синтаксису крапки викликаєте `.accounts`, передаючи об'єкт з кожним акаунтом, який інструкція очікує на основі IDL.
4. За потреби викликаєте `signers` — за допомогою синтаксису крапки викликаєте `.signers`, передаючи масив додаткових підписантів, необхідних для інструкції.
5. Викликаєте `rpc` — цей метод створює та надсилає підписану транзакцію з вказаною інструкцією та повертає `TransactionSignature`. Коли використовується `.rpc`, `Wallet` з `Provider` автоматично додається як підписант і не потрібно вказувати його явно.
   
Зверніть увагу, що якщо інструкція не потребує додаткових підписантів, окрім `Wallet`, вказаного в `Provider`, рядок `.signer([])` можна упустити.

Також можна безпосередньо побудувати транзакцію, замінивши `.rpc()` на `.transaction()`. Це створить об'єкт `Transaction`, використовуючи вказану інструкцію.

```tsx
// creates transaction
const transaction = await program.methods
  .instructionName(instructionDataInputs)
  .accounts({})
  .transaction()

await sendTransaction(transaction, connection)
```

Подібним чином, ви можете використовувати той самий формат для побудови інструкції за допомогою `.instruction()` і потім вручну додавати інструкції до нової транзакції. Це створить об'єкт `TransactionInstruction`, використовуючи вказану інструкцію.

```tsx
// creates first instruction
const instructionOne = await program.methods
  .instructionOneName(instructionOneDataInputs)
  .accounts({})
  .instruction()

// creates second instruction
const instructionTwo = await program.methods
  .instructionTwoName(instructionTwoDataInputs)
  .accounts({})
  .instruction()

// add both instruction to one transaction
const transaction = new Transaction().add(instructionOne, instructionTwo)

// send transaction
await sendTransaction(transaction, connection)
```

Підсумовуючи, Anchor `MethodsBuilder` надає спрощений і більш гнучкий спосіб взаємодії з ончейн-програмами. Ви можете побудувати інструкцію, транзакцію або побудувати й надіслати транзакцію, використовуючи в основному один і той самий формат, без необхідності вручну серіалізувати або десеріалізувати акаунти чи дані інструкцій.

## Отримання програмних акаунтів 

Об'єкт `Program` також дозволяє легко отримувати та фільтрувати акаунти програми. Просто викликайте `account` на `program`, а потім вказуйте ім'я типу акаунту, як це зазначено в IDL. Anchor потім десеріалізує та повертає всі акаунти відповідно до специфікацій.

Нижче наведено приклад того, як можна отримати всі існуючі акаунти `counter` для програми Counter.

```tsx
const accounts = await program.account.counter.all()
```

Ви також можете застосувати фільтр за допомогою `memcmp`, вказавши `offset` та `bytes` для фільтрації. 

У наведеному прикладі отримуються всі акаунти `counter` зі значенням `count` рівним 0. Зверніть увагу, що `offset` 8 — це 8 байт, які використовує Anchor для ідентифікації типів акаунтів. 9-й байт — це місце, де починається поле `count`. Ви можете звернутися до IDL, щоб побачити, що наступний байт зберігає поле `count` типу `u64`. Anchor потім фільтрує та повертає всі акаунти з відповідними байтами в тій самій позиції.

```tsx
const accounts = await program.account.counter.all([
    {
        memcmp: {
            offset: 8,
            bytes: bs58.encode((new BN(0, 'le')).toArray()),
        },
    },
])
```

Альтернативно, ви також можете отримати десеріалізовані дані акаунту для конкретного акаунту за допомогою `fetch`, якщо ви знаєте адресу акаунту, який шукаєте.

```tsx
const account = await program.account.counter.fetch(ACCOUNT_ADDRESS)
```

Аналогічно, ви можете отримати кілька акаунтів за допомогою `fetchMultiple`.

```tsx
const accounts = await program.account.counter.fetchMultiple([ACCOUNT_ADDRESS_ONE, ACCOUNT_ADDRESS_TWO])
```

# Лабораторна робота  

Давайте попрактикуємося разом, створюючи фронтенд для програми Counter з минулого уроку. Як нагадування, програма Counter має дві інструкції:

- `initialize` — ініціалізує новий акаунт `Counter` та встановлює значення `count` на `0`
- `increment` — збільшує значення `count` на існуючому акаунті `Counter`

### 1. Завантажте початковий код

Завантажте [початковий код для цього проєкту](https://github.com/Unboxed-Software/anchor-ping-frontend/tree/starter). Після того, як ви завантажите початковий код, ознайомтесь з його структурою. Встановіть залежності за допомогою команди `npm install`, а потім запустіть додаток командою `npm run dev`.

Цей проєкт — це проста програма на Next.js. Вона містить `WalletContextProvider`, який ми створили на уроці про [гаманці](https://github.com/Unboxed-Software/solana-course/blob/main/content/interact-with-wallets), файл `idl.json` для програми Counter, а також компоненти `Initialize` та `Increment`, які ми будемо створювати протягом цього заняття. У початковому коді також вказано `programId` програми, яку ми будемо викликати.

### 2. `Initialize`

Для початку давайте завершимо налаштування для створення об'єкта `Program` у компоненті `Initialize.tsx`.

Пам'ятайте, нам потрібен екземпляр `Program`, щоб використовувати `MethodsBuilder` з Anchor для виклику інструкцій нашої програми. Для цього нам знадобляться гаманець Anchor і з'єднання, які ми можемо отримати за допомогою хуків `useAnchorWallet` і `useConnection`. Давайте також створимо `useState`, щоб зберігати екземпляр програми.

```tsx
export const Initialize: FC<Props> = ({ setCounter }) => {
  const [program, setProgram] = useState("")

  const { connection } = useConnection()
  const wallet = useAnchorWallet()

  ...
}
```

З цим ми можемо перейти до створення фактичного екземпляру `Program`. Давайте зробимо це в `useEffect`.

Спочатку нам потрібно або отримати стандартний провайдер, якщо він вже існує, або створити його, якщо його немає. Ми можемо зробити це, викликавши `getProvider` всередині блоку `try/catch`.  Якщо виникає помилка, це означає, що стандартного провайдера немає, і нам потрібно створити новий.

Коли ми отримаємо провайдера, ми можемо створити екземпляр `Program`.

```tsx
useEffect(() => {
  let provider: anchor.Provider

  try {
    provider = anchor.getProvider()
  } catch {
    provider = new anchor.AnchorProvider(connection, wallet, {})
    anchor.setProvider(provider)
  }

  const program = new anchor.Program(idl as anchor.Idl, PROGRAM_ID)
  setProgram(program)
}, [])
```

Тепер, коли ми завершили налаштування Anchor, ми можемо фактично викликати інструкцію `initialize` програми. Ми зробимо це всередині функції `onClick`.

По-перше, нам потрібно згенерувати нову `Keypair` для нового акаунту `Counter`, оскільки ми ініціалізуємо акаунт вперше.

Тоді ми можемо використовувати `MethodsBuilder` з Anchor для створення та відправлення нової транзакції. Пам'ятайте, що Anchor може визначити деякі акаунти, які необхідні, такі як акаунти `user` і `systemAccount`. Однак він не може визначити акаунт `counter`, оскільки ми генеруємо його динамічно, тому вам потрібно додати його за допомогою `.accounts`. Також потрібно додати цю пару ключів як підписанта через `.signers`. Наприкінці, ви можете використати `.rpc()`, щоб надіслати транзакцію до гаманця користувача.

Коли транзакція буде оброблена, викличте `setUrl` з URL для експлорера, а потім викликайте `setCounter`, передаючи counter-акаунт.

```tsx
const onClick = async () => {
  const sig = await program.methods
    .initialize()
    .accounts({
      counter: newAccount.publicKey,
      user: wallet.publicKey,
      systemAccount: anchor.web3.SystemProgram.programId,
    })
    .signers([newAccount])
    .rpc()

    setTransactionUrl(`https://explorer.solana.com/tx/${sig}?cluster=devnet`)
    setCounter(newAccount.publicKey)
}
```

### 3. `Increment`

Далі, давайте перейдемо до компонента `Increment.tsx`. Так само, як і раніше, завершіть налаштування для створення об'єкта `Program`. Окрім виклику `setProgram`, хук `useEffect` повинен також викликати `refreshCount`.

Додайте наступний код для початкового налаштування:

```tsx
export const Increment: FC<Props> = ({ counter, setTransactionUrl }) => {
  const [count, setCount] = useState(0)
  const [program, setProgram] = useState<anchor.Program>()
  const { connection } = useConnection()
  const wallet = useAnchorWallet()

  useEffect(() => {
    let provider: anchor.Provider

    try {
      provider = anchor.getProvider()
    } catch {
      provider = new anchor.AnchorProvider(connection, wallet, {})
      anchor.setProvider(provider)
    }

    const program = new anchor.Program(idl as anchor.Idl, PROGRAM_ID)
    setProgram(program)
    refreshCount(program)
  }, [])
  ...
}
```

Далі давайте використаємо `MethodsBuilder` з Anchor, щоб побудувати нову інструкцію для виклику інструкції `increment`. Знову ж таки, Anchor може визначити акаунт `user` з гаманця, тому нам потрібно лише додати акаунт `counter`.

```tsx
const incrementCount = async () => {
  const sig = await program.methods
    .increment()
    .accounts({
      counter: counter,
      user: wallet.publicKey,
    })
    .rpc()

  setTransactionUrl(`https://explorer.solana.com/tx/${sig}?cluster=devnet`)
}
```

### 4. Відображення правильного значення лічильника

Тепер, коли ми можемо ініціалізувати програму лічильника та збільшувати значення, нам потрібно зробити так, щоб наш інтерфейс відображав значення, яке зберігається в count-акаунті.

Ми покажемо, як спостерігати за змінами акаунтів у наступному уроці, але поки що в нас є кнопка, яка викликає `refreshCount`, щоб ви могли натискати її та показувати нове значення після кожного виклику `increment`.

Всередині `refreshCount` давайте використаємо `program`, щоб отримати count-акаунт, а потім використаємо `setCount`, щоб встановити значення лічильника, яке зберігається в програмі.

```tsx
const refreshCount = async (program) => {
  const counterAccount = await program.account.counter.fetch(counter)
  setCount(counterAccount.count.toNumber())
}
```

Дуже просто з Anchor!

### 5. Тестування фронтенду

На цьому етапі все має працювати! Ви можете протестувати фронтенд, запустивши команду `npm run dev`.

1. Підключіть свій гаманець, і ви повинні побачити кнопку `Initialize Counter`.
2. Натисніть на кнопку `Initialize Counter`, а потім підтвердіть транзакцію.
3. Потім ви повинні побачити посилання внизу екрану на Solana Explorer для транзакції `initialize`. Також повинні з'явитися кнопки `Increment Counter`, `Refresh Count` та значення рахунку.
4. Натисніть на кнопку `Increment Counter`, а потім підтвердіть транзакцію.
5. Почекати кілька секунд і натиснути `Refresh Count`. Значення лічильника має збільшитися на екрані.

![Gif of Anchor Frontend Demo](../assets/anchor-frontend-demo.gif)

Щоб переглянути журнали програми для кожної транзакції - натисніть на посилання!

![Initialize Program Log](../assets/anchor-frontend-initialize.png)

![Increment Program Log](../assets/anchor-frontend-increment.png)

Вітаємо! Тепер ви знаєте, як налаштувати фронтенд для виклику програми в Solana за допомогою Anchor IDL.

Якщо вам потрібно більше часу, щоб розібратися з цим проектом і краще засвоїти концепції, можете переглянути [код рішення у гілці `solution-increment`](https://github.com/Unboxed-Software/anchor-ping-frontend/tree/solution-increment) перед тим, як продовжити.

# Завдання

Тепер ваша черга створити щось самостійно. Спираючись на те, що ми зробили в лабораторній роботі, спробуйте створити новий компонент у фронтенді, який реалізує кнопку для зменшення значення лічильника.

Перш ніж створювати компонент у фронтенді, вам спершу потрібно:  

1. Створити та розгорнути нову програму, яка реалізує інструкцію `decrement`.  
2. Оновити файл IDL у фронтенді, додавши до нього IDL із вашої нової програми.  
3. Оновити `programId`, вказавши ідентифікатор вашої нової програми.

Якщо вам потрібна допомога, можете звернутися до [цього прикладу програми](https://github.com/Unboxed-Software/anchor-counter-program/tree/solution-decrement).

Спробуйте зробити це самостійно! Але якщо у вас виникнуть труднощі, можете звернутися до [коду рішення](https://github.com/Unboxed-Software/anchor-ping-frontend/tree/solution-decrement).

## Завершили Лабораторну роботу?

Поділіться своїм кодом на GitHub та [поділіться своїми враженнями про цей урок](https://form.typeform.com/to/IPH0UGz7#answers-lesson=774a4023-646d-4394-af6d-19724a6db3db)!
