---
назва: Група, Вказівник Групи, Учасник, Вказівник Учасника (Group, Group Pointer, Member, Member Pointer)
завдання:  
 - Створити NFT колекцію за допомогою розширень group, group pointer, member, member pointer.
 - Оновити уповноважений акаунт та максимальний розмір групи.
---

# Стислий виклад  

- 'token groups' зазвичай використовуються для реалізації NFT-колекцій.  
- Розширення `group pointer` встановлює акаунт групи на мінт токена для зберігання інформації про групу токенів.  
- Розширення `group` дозволяє зберігати дані групи безпосередньо в мінт-акаунті.  
- Розширення `member pointer` встановлює індивідуальний акаунт учасника на мінт токена для зберігання інформації про членство токена в групі.  
- Розширення `member` дозволяє зберігати дані учасника безпосередньо в мінт-акаунті.  

# Огляд  
SPL-токени цінні самі по собі, але їх можна поєднувати для розширення функціональності. Це можна зробити в програмі Token Extensions, комбінуючи розширення `group`, `group pointer`, `member` та `member pointer`. Найпоширеніший сценарій використання цих розширень — створення колекції NFT.

Щоб створити колекцію NFT, нам потрібні дві частини: NFT "колекція" та всі NFT у цій колекції. Ми можемо зробити це повністю за допомогою розширень токенів. NFT "колекція" може бути єдиним мінт-акаунтом, що поєднує розширення `metadata`, `metadata pointer`, `group` і `group pointer`. А кожен окремий NFT у колекції може бути одним із мінт-акаунтів у масиві, що поєднує розширення `metadata`, `metadata pointer`, `member` і `member pointer`.

Хоча колекції NFT є поширеним прикладом використання, групи та учасники можуть бути застосовані до будь-якого типу токенів.

Коротке зауваження щодо різниці між `group pointer` і `group`. Розширення `group pointer` зберігає адресу будь-якого акаунта ончейн, який відповідає [Token-Group Interface](https://github.com/solana-labs/solana-program-library/tree/master/token-group/interface). В той час як розширення `group` зберігає дані Token-Group Interface безпосередньо в мінт-акаунті. Як правило, ці розширення використовуються разом, де `group pointer` вказує на сам мінт. Те ж саме стосується `member pointer` і `member`, але для даних учасника.

УВАГА: Група може мати багато учасників, але учасник може належати лише одній групі.

## Група та вказівник групи (Group і Group Pointer)

Розширення `group` та `group pointer` визначають групу токенів. Ось як виглядають дані ончейн:

- `update_authority`: Уповноважений акаунт, що може підписувати транзакцію для оновлення групи.
- `mint`: Мінт-акаунт групового токена.
- `size`: Поточна кількість учасників групи.
- `max_size`: Максимальна кількість учасників групи.
  
```rust
type OptionalNonZeroPubkey = Pubkey; // if all zeroes, interpreted as `None`
type PodU32 = [u8; 4];
type Pubkey = [u8; 32];

/// Type discriminant: [214, 15, 63, 132, 49, 119, 209, 40]
/// First 8 bytes of `hash("spl_token_group_interface:group")`
pub struct TokenGroup {
    /// The authority that can sign to update the group
    pub update_authority: OptionalNonZeroPubkey,
    /// The associated mint, used to counter spoofing to be sure that group
    /// belongs to a particular mint
    pub mint: Pubkey,
    /// The current number of group members
    pub size: PodU32,
    /// The maximum number of group members
    pub max_size: PodU32,
}
```

### Створення мінт-акаунту з розширенням group і group pointer  

Створення мінт акаунту з `group` і `group pointer` включає чотири інструкції:  
- `SystemProgram.createAccount`  
- `createInitializeGroupPointerInstruction`  
- `createInitializeMintInstruction`  
- `createInitializeGroupInstruction`

Перша інструкція `SystemProgram.createAccount` виділяє простір на блокчейні для мінт-акаунту. Однак, як і у випадку з усіма мінт-акаунтами Token Extensions Program, нам потрібно обчислити розмір і вартість мінт-акаунту. Це можна зробити за допомогою `getMintLen` і `getMinimumBalanceForRentExemption`. У цьому випадку ми викликаємо `getMintLen`, використовуючи тільки `ExtensionType.GroupPointer`. Потім додаємо `TOKEN_GROUP_SIZE` до довжини мінт-акаунту, щоб врахувати дані групи.

Щоб отримати розмір мінт-акаунту та створити інструкцію, виконайте наступне:

```ts
// get mint length
const extensions = [ExtensionType.GroupPointer]
const mintLength = getMintLen(extensions) + TOKEN_GROUP_SIZE;

const mintLamports = await connection.getMinimumBalanceForRentExemption(mintLength)

const createAccountInstruction = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: mintKeypair.publicKey,
  space: mintLength,
  lamports: mintLamports,
  programId: TOKEN_2022_PROGRAM_ID,
})
```

Друга інструкція `createInitializeGroupPointerInstruction` ініціалізує group pointer. Вона приймає мінт-акаунт, необов’язковий уповноважений акаунт, що може встановлювати адресу групи, адресу яка зберігає групу, і програму-власника як аргументи.

```ts
const initializeGroupPointerInstruction = createInitializeGroupPointerInstruction(
  mintKeypair.publicKey,
  payer.publicKey,
  mintKeypair.publicKey,
  TOKEN_2022_PROGRAM_ID
)
```

Третя інструкція `createInitializeMintInstruction` ініціалізує мінт-акаунт.

```ts
const initializeMintInstruction = createInitializeMintInstruction(
  mintKeypair.publicKey,
  decimals,
  payer.publicKey,
  payer.publicKey,
  TOKEN_2022_PROGRAM_ID
)
```

Четверта інструкція `createInitializeGroupInstruction` фактично ініціалізує групу і зберігає конфігурацію на акаунті групи.

```ts
const initializeGroupInstruction = createInitializeGroupInstruction({
  group: mintKeypair.publicKey,
  maxSize: maxMembers,
  mint: mintKeypair.publicKey,
  mintAuthority: payer.publicKey,
  programId: TOKEN_2022_PROGRAM_ID,
  updateAuthority: payer.publicKey,
})      
```

Нарешті, ми додаємо інструкції до транзакції і відправляємо її в мережу Solana.
```ts
const mintTransaction = new Transaction().add(
  createAccountInstruction,
  initializeGroupPointerInstruction,
  initializeMintInstruction,
  initializeGroupInstruction
)

const signature = await sendAndConfirmTransaction(
  connection,
  mintTransaction,
  [payer, mintKeypair],
  { commitment: 'finalized' }
)
```

## Оновлення уповноваженого акаунту групи

Для оновлення уповноваженого акаунту групи, нам просто потрібно використати функцію `tokenGroupUpdateGroupAuthority`.

```ts
import { tokenGroupUpdateGroupAuthority } from "@solana/spl-token"

const signature = await tokenGroupUpdateGroupAuthority(
  connection, //connection - Connection to use
  payer, // payer - Payer for the transaction fees
  mint.publicKey, // mint - Group mint
  oldAuthority, // account - Public key of the old update authority
  newAuthority, // account - Public key of the new update authority
  undefined, // multiSigners - Signing accounts if `authority` is a multisig
  { commitment: 'finalized' }, // confirmOptions - Options for confirming thr transaction
  TOKEN_2022_PROGRAM_ID // programId - SPL Token program account
)
```

## Оновлення максимального розміру групи

Для оновлення максимального розміру групи, нам просто потрібно використати функцію `tokenGroupUpdateGroupMaxSize`.

```ts
import { tokenGroupUpdateGroupMaxSize } from "@solana/spl-token"

const signature = tokenGroupUpdateGroupMaxSize(
  connection, //connection - Connection to use
  payer, // payer - Payer for the transaction fees
  mint.publicKey, // mint - Group mint
  updpateAuthority, // account - Update authority of the group
  4, // maxSize - new max size of the group
  undefined, // multiSigners — Signing accounts if `authority` is a multisig
  { commitment: "finalized" }, // confirmOptions - Options for confirming thr transaction
  TOKEN_2022_PROGRAM_ID // programId - SPL Token program account
)
```

## Учасник і Вказівник на Учасника (Member і Member Pointer)  

Розширення `member` і `member pointer` визначають участь токена в групі. Ончейн-дані містять:  

- `mint`: Мінт-акаунт токена-учасника.  
- `group`: Адреса акаунту групи.  
- `member_number`: Номер учасника (індекс у межах групи).

```rust
/// Type discriminant: [254, 50, 168, 134, 88, 126, 100, 186]
/// First 8 bytes of `hash("spl_token_group_interface:member")`
pub struct TokenGroupMember {
    /// The associated mint, used to counter spoofing to be sure that member
    /// belongs to a particular mint
    pub mint: Pubkey,
    /// The pubkey of the `TokenGroup`
    pub group: Pubkey,
    /// The member number
    pub member_number: PodU32,
}
```

### Створення мінт-акаунту з розширеннями Member pointer
Створення мінт-акаунту з розширеннями `member pointer` і `member` включає чотири інструкції:  

- `SystemProgram.createAccount`  
- `createInitializeGroupMemberPointerInstruction`  
- `createInitializeMintInstruction`  
- `createInitializeMemberInstruction`  

Перша інструкція `SystemProgram.createAccount` виділяє місце в блокчейні для мінт акаунта. Однак, як і для всіх мінт-акаунтів у Token Extensions Program, потрібно розрахувати його розмір і вартість. Це можна зробити за допомогою `getMintLen` і `getMinimumBalanceForRentExemption`. У цьому випадку викликаємо `getMintLen` із `ExtensionType.GroupMemberPointer`. Потім додаємо `TOKEN_GROUP_MEMBER_SIZE` до довжини мінт-акаунту, щоб урахувати дані про учасника.  

Щоб отримати довжину мінт-акаунту та створити інструкцію для створення акаунту, виконайте наступне:
```ts
// get mint length
const extensions = [ExtensionType.GroupMemberPointer];
const mintLength = getMintLen(extensions) + TOKEN_GROUP_MEMBER_SIZE;

const mintLamports = await connection.getMinimumBalanceForRentExemption(mintLength);

const createAccountInstruction = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: mintKeypair.publicKey,
  space: mintLength,
  lamports: mintLamports,
  programId: TOKEN_2022_PROGRAM_ID,
});
```
Друга інструкція `createInitializeGroupMemberPointerInstruction` ініціалізує вказівник на учасника групи. Вона приймає такі аргументи: мінт-акаунт, необов’язковий уповноважений акаунт, що може встановлювати адресу групи, адресу, яка зберігає групу, та програму-власника.

```ts
const initializeGroupMemberPointerInstruction = createInitializeGroupMemberPointerInstruction(
  mintKeypair.publicKey,
  payer.publicKey,
  mintKeypair.publicKey,
  TOKEN_2022_PROGRAM_ID
);
```
Третя інструкція `createInitializeMintInstruction` ініціалізує мінт-акаунт.

```ts
const initializeMintInstruction = createInitializeMintInstruction(
  mintKeypair.publicKey,
  decimals,
  payer.publicKey,
  payer.publicKey,
  TOKEN_2022_PROGRAM_ID
);
```
Четверта інструкція `createInitializeMemberInstruction` фактично ініціалізує учасника та зберігає конфігурацію в акаунті учасника. Ця функція приймає адресу групи як аргумент і асоціює учасника з цією групою.

```ts
const initializeMemberInstruction = createInitializeMemberInstruction({
  group: groupAddress,
  groupUpdateAuthority: payer.publicKey,
  member: mintKeypair.publicKey,
  memberMint: mintKeypair.publicKey,
  memberMintAuthority: payer.publicKey,
  programId: TOKEN_2022_PROGRAM_ID,
});
```

Нарешті, ми додаємо інструкції до транзакції та відправляємо її в мережу Solana.

```ts
const mintTransaction = new Transaction().add(
  createAccountInstruction,
  initializeGroupMemberPointerInstruction,
  initializeMintInstruction,
  initializeMemberInstruction
);

const signature = await sendAndConfirmTransaction(
  connection,
  mintTransaction,
  [payer, mintKeypair],
  { commitment: 'finalized' }
);
```

## Отримання даних про групу та учасника  

### Отримання стану `group pointer`  
Щоб отримати стан `group pointer` для мінт-акаунту, потрібно отримати акаунт за допомогою `getMint`, а потім проаналізувати ці дані за допомогою функції `getGroupPointerState`. Це поверне нам структуру `GroupPointer`.

```ts
/** GroupPointer as stored by the program */
export interface GroupPointer {
  /** Optional authority that can set the group address */
  authority: PublicKey | null;
  /** Optional account address that holds the group */
  groupAddress: PublicKey | null;
}
```

Щоб отримати дані `GroupPointer`, викличте наступне:

```ts
const groupMint = await getMint(connection, mint, "confirmed", TOKEN_2022_PROGRAM_ID);

const groupPointerData: GroupPointer = getGroupPointerState(groupMint);
```

### Отримання стану групи
Щоб отримати стан групи для мінт-акаунту, потрібно отримати акаунт за допомогою `getMint`, а потім обробити ці дані за допомогою функції `getTokenGroupState`. Це поверне структуру `TokenGroup`.

```ts
export interface TokenGroup {
  /** The authority that can sign to update the group */
  updateAuthority?: PublicKey;
  /** The associated mint, used to counter spoofing to be sure that group belongs to a particular mint */
  mint: PublicKey;
  /** The current number of group members */
  size: number;
  /** The maximum number of group members */
  maxSize: number;
}
```

Щоб отримати дані `TokenGroup`, викликайте наступне:

```ts
const groupMint = await getMint(connection, mint, "confirmed", TOKEN_2022_PROGRAM_ID);

const groupData: TokenGroup = getTokenGroupState(groupMint);
```

### Отримання стану вказівника учасника групи
Щоб отримати стан `member pointer` для мінт-акаунту, ми отримуємо мінт-акаунт за допомогою `getMint` і потім розбираємо його за допомогою `getGroupMemberPointerState`. Це повертає нам структуру `GroupMemberPointer`.

```ts
/** GroupMemberPointer as stored by the program */
export interface GroupMemberPointer {
  /** Optional authority that can set the member address */
  authority: PublicKey | null;
  /** Optional account address that holds the member */
  memberAddress: PublicKey | null;
}
```

Щоб отримати дані `GroupMemberPointer`, викликайте наступне:

```ts
const memberMint = await getMint(connection, mint, "confirmed", TOKEN_2022_PROGRAM_ID);

const memberPointerData = getGroupMemberPointerState(memberMint);
```

### Отримати стан учасника групи
Щоб отримати стан `member` для мінт-акаунту, ми отримуємо мінт за допомогою `getMint`, а потім розбираємо ці дані за допомогою `getTokenGroupMemberState`. Це повертає структуру `TokenGroupMember`.

```ts
export interface TokenGroupMember {
  /** The associated mint, used to counter spoofing to be sure that member belongs to a particular mint */
  mint: PublicKey;
  /** The pubkey of the `TokenGroup` */
  group: PublicKey;
  /** The member number */
  memberNumber: number;
}
```

Щоб отримати дані `TokenGroupMember`, викликайте наступне:

```ts
const memberMint = await getMint(connection, mint, "confirmed", TOKEN_2022_PROGRAM_ID);
const memberData = getTokenGroupMemberState(memberMint);
```

# Лабораторна робота
У цій лабораторній роботі ми створимо колекцію NFT Cool Cats, використовуючи розширення `group`, `group pointer`, `member` та `member pointer` разом з розширеннями `metadata` та `metadata pointer`.

Колекція NFT Cool Cats матиме групу NFT з трьома учасниками NFT всередині (наприклад три різні картинки).

### 1. Початок роботи
Для початку клонуйте гілку `starter` [цього](https://github.com/Unboxed-Software/solana-lab-group-member) репозиторію.

```bash
git clone https://github.com/Unboxed-Software/solana-lab-group-member.git
cd solana-lab-group-member
git checkout starter
npm install
```

Цей код у гілці `starter` містить наступне:

- `index.ts`: створює об'єкт підключення та викликає функцію `initializeKeypair`. Тут ми будемо писати наш скрипт.
- `assets`: папка, яка містить зображення для нашої NFT колекції.
- `helper.ts`: допоміжні функції для завантаження метаданих.
  
### 2. Запуск ноди-валідатора

Для цілей цього посібника ми запустимо власну ноду-валідатор.

У окремому терміналі виконайте таку команду: `solana-test-validator`. Вона запустить ноду та виведе кілька ключів і значень. Значення, яке нам потрібно використати для підключення, — це JSON RPC URL, який у цьому випадку дорівнює `http://127.0.0.1:8899`. Далі ми використовуємо цю URL-адресу при створенні підключення, щоб указати, що слід використовувати локальний RPC.

`const connection = new Connection("http://127.0.0.1:8899", "confirmed");`

Після правильного налаштування валідатора ви можете запустити `index.ts` і переконатися, що все працює.

```bash
npx esrun src/index.ts
```

### 3. Налаштування метаданих групи  
Перед створенням нашої групи NFT потрібно підготувати та завантажити метадані групи. Для цього ми використовуємо devnet Irys (Arweave) для завантаження зображення та метаданих. Ця функціональність уже реалізована у файлі `helpers.ts`.

### Для зручності цього уроку ми надали ресурси для NFT у директорії `assets`.  

Якщо хочете використовувати власні файли та метадані — будь ласка!  

Щоб підготувати метадані для групи, потрібно виконати такі кроки:  
1. Відформатувати метадані для завантаження, використовуючи інтерфейс `LabNFTMetadata` із `helper.ts`.  
2. Викликати функцію `uploadOffChainMetadata` із `helpers.ts`.  
3. Відформатувати все, включно з отриманим `uri` з попереднього кроку.

Нам потрібно відформатувати метадані для завантаження (`LabNFTMetadata`), завантажити зображення та метадані (`uploadOffChainMetadata`), а потім усе відформатувати у інтерфейс `TokenMetadata` з бібліотеки `@solana/spl-token-metadata`.  

**Примітка:** Ми використовуємо devnet Irys, який дозволяє безкоштовно завантажувати файли розміром до 100 КБ.

```ts
// Create group metadata

const groupMetadata: LabNFTMetadata = {
  mint: groupMintKeypair,
  imagePath: "assets/collection.png",
  tokenName: "cool-cats-collection",
  tokenDescription: "Collection of Cool Cat NFTs",
  tokenSymbol: "MEOW",
  tokenExternalUrl: "https://solana.com/",
  tokenAdditionalMetadata: {},
  tokenUri: "",
}

// Upload off-chain metadata
groupMetadata.tokenUri = await uploadOffChainMetadata(
  payer,
  groupMetadata
)

// Format group token metadata
const collectionTokenMetadata: TokenMetadata = {
  name: groupMetadata.tokenName,
  mint: groupMintKeypair.publicKey,
  symbol: groupMetadata.tokenSymbol,
  uri: groupMetadata.tokenUri,
  updateAuthority: payer.publicKey,
  additionalMetadata: Object.entries(
    groupMetadata.tokenAdditionalMetadata || []
  ).map(([trait_type, value]) => [trait_type, value]),
}
```

Можете запустити скрипт і переконатися, що все завантажується коректно.

```bash
npx esrun src/index.ts
```

### 3. Створення мінт-акаунту з group і group pointer 
Створимо групу NFT, створивши мінт-акаунт із розширеннями `metadata`, `metadata pointer`, `group` і `group pointer`.  

Цей NFT є візуальним представленням нашої колекції.  

Спочатку визначимо вхідні дані для нашої нової функції `createTokenGroup`:

- `connection`: з'єднання з блокчейном  
- `payer`: пара ключів, яка оплачує транзакцію  
- `mintKeypair`: пара ключів мінт-акаунту
- `decimals`: кількість знаків після коми у мінт-акаунті (0 для NFT)  
- `maxMembers`: максимальна кількість учасників у групі  
- `metadata`: метадані для мінт-акаунту групи

```ts
export async function createTokenGroup(
  connection: Connection,
  payer: Keypair,
  mintKeypair: Keypair,
  decimals: number,
  maxMembers: number,
  metadata: TokenMetadata
): Promise<TransactionSignature>
```

Щоб створити наш NFT, ми збережемо метадані безпосередньо в мінт акаунті, використовуючи розширення `metadata` та `metadata pointer`. Також збережемо інформацію про групу за допомогою розширень `group` та `group pointer`.  

Для створення групи NFT нам потрібні такі інструкції:

- `SystemProgram.createAccount`: Виділяє простір у Solana для мінт-акаунту. Ми можемо отримати `mintLength` і `mintLamports`, використовуючи `getMintLen` та `getMinimumBalanceForRentExemption` відповідно.  
- `createInitializeGroupPointerInstruction`: Ініціалізує `group pointer`
- `createInitializeMetadataPointerInstruction`: Ініціалізує `metadata pointer` 
- `createInitializeMintInstruction`: Ініціалізує мінт-акаунт
- `createInitializeGroupInstruction`: Ініціалізує групу
- `createInitializeInstruction`: Ініціалізує метадані

Нарешті, нам потрібно додати всі ці інструкції до транзакції, надіслати її в мережу Solana і отримати підпис. Це можна зробити, викликавши `sendAndConfirmTransaction`.

```ts
import {
  sendAndConfirmTransaction,
  Connection,
  Keypair,
  SystemProgram,
  Transaction,
  TransactionSignature,
} from "@solana/web3.js"

import {
  ExtensionType,
  createInitializeMintInstruction,
  getMintLen,
  TOKEN_2022_PROGRAM_ID,
  createInitializeGroupInstruction,
  createInitializeGroupPointerInstruction,
  TYPE_SIZE,
  LENGTH_SIZE,
  createInitializeMetadataPointerInstruction,
  TOKEN_GROUP_SIZE,
} from "@solana/spl-token"
import {
  TokenMetadata,
  createInitializeInstruction,
  pack,
} from "@solana/spl-token-metadata"

export async function createTokenGroup(
  connection: Connection,
  payer: Keypair,
  mintKeypair: Keypair,
  decimals: number,
  maxMembers: number,
  metadata: TokenMetadata
): Promise<TransactionSignature> {

  const extensions: ExtensionType[] = [
    ExtensionType.GroupPointer,
    ExtensionType.MetadataPointer,
  ]

  const metadataLen = TYPE_SIZE + LENGTH_SIZE + pack(metadata).length + 500
  const mintLength = getMintLen(extensions)
  const totalLen = mintLength + metadataLen + TOKEN_GROUP_SIZE

  const mintLamports =
    await connection.getMinimumBalanceForRentExemption(totalLen)

  const mintTransaction = new Transaction().add(
    SystemProgram.createAccount({
      fromPubkey: payer.publicKey,
      newAccountPubkey: mintKeypair.publicKey,
      space: mintLength,
      lamports: mintLamports,
      programId: TOKEN_2022_PROGRAM_ID,
    }),
    createInitializeGroupPointerInstruction(
      mintKeypair.publicKey,
      payer.publicKey,
      mintKeypair.publicKey,
      TOKEN_2022_PROGRAM_ID
    ),
    createInitializeMetadataPointerInstruction(
      mintKeypair.publicKey,
      payer.publicKey,
      mintKeypair.publicKey,
      TOKEN_2022_PROGRAM_ID
    ),
    createInitializeMintInstruction(
      mintKeypair.publicKey,
      decimals,
      payer.publicKey,
      payer.publicKey,
      TOKEN_2022_PROGRAM_ID
    ),
    createInitializeGroupInstruction({
      group: mintKeypair.publicKey,
      maxSize: maxMembers,
      mint: mintKeypair.publicKey,
      mintAuthority: payer.publicKey,
      programId: TOKEN_2022_PROGRAM_ID,
      updateAuthority: payer.publicKey,
    }),
    createInitializeInstruction({
      metadata: mintKeypair.publicKey,
      mint: mintKeypair.publicKey,
      mintAuthority: payer.publicKey,
      name: metadata.name,
      programId: TOKEN_2022_PROGRAM_ID,
      symbol: metadata.symbol,
      updateAuthority: payer.publicKey,
      uri: metadata.uri,
    })
  )

  const signature = await sendAndConfirmTransaction(
    connection,
    mintTransaction,
    [payer, mintKeypair]
  )

  return signature
}
```

Тепер, коли у нас є функція, викличемо її у файлі `index.ts`.

```ts
// Create group
const signature = await createTokenGroup(
  connection,
  payer,
  groupMintKeypair,
  decimals,
  maxMembers,
  collectionTokenMetadata
)

console.log(`Created collection mint with metadata:\n${getExplorerLink("tx", signature, 'localnet')}\n`)
```

Перед тим як запустити скрипт, отримаємо щойно створену групу NFT і виведемо її вміст. Зробимо це у файлі `index.ts`:

```ts
// Fetch the group
const groupMint = await getMint(connection, groupMintKeypair.publicKey, "confirmed", TOKEN_2022_PROGRAM_ID);
const fetchedGroupMetadata = await getTokenMetadata(connection, groupMintKeypair.publicKey);
const metadataPointerState = getMetadataPointerState(groupMint);
const groupData = getGroupPointerState(groupMint);

console.log("\n---------- GROUP DATA -------------\n");
console.log("Group Mint: ", groupMint.address.toBase58());
console.log("Metadata Pointer Account: ", metadataPointerState?.metadataAddress?.toBase58());
console.log("Group Pointer Account: ", groupData?.groupAddress?.toBase58());
console.log("\n--- METADATA ---\n");
console.log("Name: ", fetchedGroupMetadata?.name);
console.log("Symbol: ", fetchedGroupMetadata?.symbol);
console.log("Uri: ", fetchedGroupMetadata?.uri);
console.log("\n------------------------------------\n");
```

Тепер можемо запустити скрипт і побачити створену групу NFT.

```bash
npx esrun src/index.ts
```

### 4. Налаштування метаданих учасника NFT  

Тепер, коли ми створили групу NFT, можемо створити NFT учасників. Але перед цим потрібно підготувати їхні метадані.  

Процес такий самий, як і для групи NFT:  

1. Форматуємо метадані для завантаження, використовуючи інтерфейс `LabNFTMetadata` із `helper.ts`.  
2. Викликаємо `uploadOffChainMetadata` із `helpers.ts`.  
3. Форматуємо все, включаючи отриманий `uri`, у `TokenMetadata` із бібліотеки `@solana/spl-token-metadata`.

Оскільки у нас є три учасники, ми повторимо кожен крок для кожного з них.  

Спочатку визначимо метадані для кожного учасника:
```ts
// Define member metadata
const membersMetadata: LabNFTMetadata[] = [
  {
    mint: cat0Mint,
    imagePath: "assets/cat_0.png",
    tokenName: "Cat 1",
    tokenDescription: "Adorable cat",
    tokenSymbol: "MEOW",
    tokenExternalUrl: "https://solana.com/",
    tokenAdditionalMetadata: {},
    tokenUri: "",
  },
  {
    mint: cat1Mint,
    imagePath: "assets/cat_1.png",
    tokenName: "Cat 2",
    tokenDescription: "Sassy cat",
    tokenSymbol: "MEOW",
    tokenExternalUrl: "https://solana.com/",
    tokenAdditionalMetadata: {},
    tokenUri: "",
  },
  {
    mint: cat2Mint,
    imagePath: "assets/cat_2.png",
    tokenName: "Cat 3",
    tokenDescription: "Silly cat",
    tokenSymbol: "MEOW",
    tokenExternalUrl: "https://solana.com/",
    tokenAdditionalMetadata: {},
    tokenUri: "",
  },
];
```

Тепер пройдемося по кожному учаснику і завантажимо їхні метадані.

```ts
// Upload member metadata
for (const member of membersMetadata) {
  member.tokenUri = await uploadOffChainMetadata(
    payer,
    member
  )
}
```

Нарешті, відформатуємо метадані для кожного учасника у формат `TokenMetadata`:  

Примітка: Нам потрібно зберегти пару ключів, оскільки вона знадобиться для створення NFT учасників.

```ts
// Format token metadata
const memberTokenMetadata: { mintKeypair: Keypair, metadata: TokenMetadata }[] = membersMetadata.map(member => ({
  mintKeypair: member.mint,
  metadata: {
    name: member.tokenName,
    mint: member.mint.publicKey,
    symbol: member.tokenSymbol,
    uri: member.tokenUri,
    updateAuthority: payer.publicKey,
    additionalMetadata: Object.entries(member.tokenAdditionalMetadata || []).map(([trait_type, value]) => [trait_type, value]),
  } as TokenMetadata
}))
```

### 5. Створення NFT учасників 
Як і з групою NFT, нам потрібно створити NFT учасників. Давайте зробимо це в новому файлі під назвою `create-member.ts`. Він буде дуже схожий на файл `create-group.ts`, але замість розширень `group` і `group pointer` ми використовуватимемо розширення `member` і `member pointer`.

Спочатку визначимо вхідні параметри для нашої нової функції `createTokenMember`:  

- `connection`: З'єднання з блокчейном  
- `payer`: Пара ключів, яка оплачує транзакцію  
- `mintKeypair`: Пара ключів мінт-акаунту 
- `decimals`: Кількість десяткових знаків для мінта (0 для NFT)  
- `metadata`: Метадані для мінт-акаунту групи  
- `groupAddress`: Адреса акаунту групи — в даному випадку це сам мінт-акаунт групи.
  
```ts
export async function createTokenMember(
  connection: Connection,
  payer: Keypair,
  mintKeypair: Keypair,
  decimals: number,
  metadata: TokenMetadata,
  groupAddress: PublicKey
): Promise<TransactionSignature>
```

Так само, як і для group NFT, потрібні такі інструкції:  
- `SystemProgram.createAccount`: виділяє місце на Solana для мінт-акаунту. `mintLength` і `mintLamports` можна отримати за допомогою `getMintLen` і `getMinimumBalanceForRentExemption`.  
- `createInitializeGroupMemberPointerInstruction`: ініціалізує member pointer.  
- `createInitializeMetadataPointerInstruction`: ініціалізує metadata pointer.  
- `createInitializeMintInstruction`: ініціалізує мінт.  
- `createInitializeMemberInstruction`: ініціалізує member.  
- `createInitializeInstruction`: ініціалізує metadata.
  
Ці інструкції потрібно додати до транзакції, відправити в мережу Solana і повернути підпис. Для цього викликаємо `sendAndConfirmTransaction`.

```ts
import {
  sendAndConfirmTransaction,
  Connection,
  Keypair,
  SystemProgram,
  Transaction,
  TransactionSignature,
  PublicKey,
} from "@solana/web3.js"

import {
  ExtensionType,
  createInitializeMintInstruction,
  getMintLen,
  TOKEN_2022_PROGRAM_ID,
  TYPE_SIZE,
  LENGTH_SIZE,
  createInitializeMetadataPointerInstruction,
  TOKEN_GROUP_MEMBER_SIZE,
  createInitializeGroupMemberPointerInstruction,
  createInitializeMemberInstruction,
} from "@solana/spl-token"
import {
  TokenMetadata,
  createInitializeInstruction,
  pack,
} from "@solana/spl-token-metadata"

export async function createTokenMember(
  connection: Connection,
  payer: Keypair,
  mintKeypair: Keypair,
  decimals: number,
  metadata: TokenMetadata,
  groupAddress: PublicKey
): Promise<TransactionSignature> {

  const extensions: ExtensionType[] = [
    ExtensionType.GroupMemberPointer,
    ExtensionType.MetadataPointer,
  ]

  const metadataLen = TYPE_SIZE + LENGTH_SIZE + pack(metadata).length
  const mintLength = getMintLen(extensions)
  const totalLen = mintLength + metadataLen + TOKEN_GROUP_MEMBER_SIZE

  const mintLamports =
    await connection.getMinimumBalanceForRentExemption(totalLen)

  const mintTransaction = new Transaction().add(
    SystemProgram.createAccount({
      fromPubkey: payer.publicKey,
      newAccountPubkey: mintKeypair.publicKey,
      space: mintLength,
      lamports: mintLamports,
      programId: TOKEN_2022_PROGRAM_ID,
    }),
    createInitializeGroupMemberPointerInstruction(
      mintKeypair.publicKey,
      payer.publicKey,
      mintKeypair.publicKey,
      TOKEN_2022_PROGRAM_ID
    ),
    createInitializeMetadataPointerInstruction(
      mintKeypair.publicKey,
      payer.publicKey,
      mintKeypair.publicKey,
      TOKEN_2022_PROGRAM_ID
    ),
    createInitializeMintInstruction(
      mintKeypair.publicKey,
      decimals,
      payer.publicKey,
      payer.publicKey,
      TOKEN_2022_PROGRAM_ID
    ),
    createInitializeMemberInstruction({
      group: groupAddress,
      groupUpdateAuthority: payer.publicKey,
      member: mintKeypair.publicKey,
      memberMint: mintKeypair.publicKey,
      memberMintAuthority: payer.publicKey,
      programId: TOKEN_2022_PROGRAM_ID,
    }),
    createInitializeInstruction({
      metadata: mintKeypair.publicKey,
      mint: mintKeypair.publicKey,
      mintAuthority: payer.publicKey,
      name: metadata.name,
      programId: TOKEN_2022_PROGRAM_ID,
      symbol: metadata.symbol,
      updateAuthority: payer.publicKey,
      uri: metadata.uri,
    })
  )

  const signature = await sendAndConfirmTransaction(
    connection,
    mintTransaction,
    [payer, mintKeypair]
  )

  return signature
}
```

Давайте додамо нашу нову функцію в `index.ts` і викличемо її для кожного учасника.
```ts
// Create member mints
for (const memberMetadata of memberTokenMetadata) {

  const signature = await createTokenMember(
    connection,
    payer,
    memberMetadata.mintKeypair,
    decimals,
    memberMetadata.metadata,
    groupMintKeypair.publicKey
  )

  console.log(`Created ${memberMetadata.metadata.name} NFT:\n${getExplorerLink("tx", signature, 'localnet')}\n`)
}
```

Давайте отримаємо наші щойно створені NFT-учасники групи та виведемо їхній вміст.

```ts
for (const member of membersMetadata) {
  const memberMint = await getMint(connection, member.mint.publicKey, "confirmed", TOKEN_2022_PROGRAM_ID);
  const memberMetadata = await getTokenMetadata(connection, member.mint.publicKey);
  const metadataPointerState = getMetadataPointerState(memberMint);
  const memberPointerData = getGroupMemberPointerState(memberMint);
  const memberData = getTokenGroupMemberState(memberMint);

  console.log("\n---------- MEMBER DATA -------------\n");
  console.log("Member Mint: ", memberMint.address.toBase58());
  console.log("Metadata Pointer Account: ", metadataPointerState?.metadataAddress?.toBase58());
  console.log("Group Account: ", memberData?.group?.toBase58());
  console.log("Member Pointer Account: ", memberPointerData?.memberAddress?.toBase58());
  console.log("Member Number: ", memberData?.memberNumber);
  console.log("\n--- METADATA ---\n");
  console.log("Name: ", memberMetadata?.name);
  console.log("Symbol: ", memberMetadata?.symbol);
  console.log("Uri: ", memberMetadata?.uri);
  console.log("\n------------------------------------\n");
}
```

Останнім кроком давайте запустимо скрипт і побачимо всю нашу колекцію NFT!
```bash
npx esrun src/index.ts
```

Ось і все! Якщо виникають труднощі, не соромтеся переглянути гілку [`solution`](https://github.com/Unboxed-Software/solana-lab-group-member/tree/solution) у репозиторії.

# Завдання  
Створіть свою власну колекцію NFT, використовуючи розширення `group`, `group pointer`, `member` та `member pointer`.
