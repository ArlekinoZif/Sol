---
назва: CPI Захист  
цілі:  
- Пояснити, від чого захищає CPI захист
- Написати код для тестування CPI захисту
---

# Підсумок  

- `CPI Guard` — це розширення токен-акаунтів із програми Token Extensions.  
- Розширення `CPI Guard` забороняє виконання певних дій під час крос-програмних викликів (CPI). Коли цей захист активовано, він забезпечує захист від потенційно шкідливих дій із токен-акаунтом.  
- `CPI Guard` можна увімкнути або вимкнути за потреби.  
- Ці захисні заходи реалізуються безпосередньо в межах самої програми Token Extensions.  

# Огляд  

CPI Guard — це розширення, яке забороняє виконання певних дій під час крос-програмних викликів (CPI), захищаючи користувачів від прихованого підпису на шкідливі дозволи, наприклад, дії, приховані в програмах, які не належать до System або Token програм.  

Конкретний приклад: якщо CPI Guard увімкнено, жоден CPI не може схвалити делегування для токен-акаунта. Це корисно, оскільки, якщо зловмисний CPI викликає `set_delegate`, зміна балансу не буде відразу очевидною, але зловмисник отримає права на переказ і спалювання токенів із цього токен-акаунта. CPI Guard робить це неможливим.  

Користувачі можуть за власним бажанням увімкнути або вимкнути розширення CPI Guard у своєму токен-акаунті. Після активації воно забезпечує наступні ефекти під час CPI:  

- Переказ (Transfer): Підписуючий авторитет має бути власником або раніше встановленим делегатом акаунта.  
- Спалювання (Burn): Підписуючий авторитет має бути власником або раніше встановленим делегатом акаунта.  
- Схвалення (Approve): Заборонено — делегати не можуть бути схвалені під час CPI.  
- Закриття акаунта (Close Account): Отримувач лампортів має бути власником акаунта.  
- Встановлення права на закриття акаунта (Set Close Authority): Заборонено, якщо не відбувається скасування права.  
- Встановлення власника (Set Owner): Завжди заборонено, включаючи випадки поза межами CPI.  

CPI Guard є розширенням токен-акаунта, тобто кожен окремий токен-акаунт у програмі Token Extensions має активувати його індивідуально.

## Як працює CPI Guard  

CPI Guard можна увімкнути або вимкнути для токен-акаунта, який створено з достатнім обсягом пам’яті для цього розширення. Програма `Token Extensions Program` виконує кілька перевірок у логіці, пов’язаній із вищезазначеними діями, щоб визначити, чи дозволяти інструкції продовжувати виконання в контексті CPI Guard. Зазвичай, це працює наступним чином:  

* Перевіряє, чи має акаунт розширення CPI Guard.  
* Перевіряє, чи активовано CPI Guard для токен-акаунта.  
* Перевіряє, чи виконується функція в межах крос-програмного виклику (CPI).  

Хороший спосіб зрозуміти розширення CPI Guard — це уявити його як замок, який можна увімкнути або вимкнути. Захисник використовує [структуру даних `CpiGuard`](https://github.com/solana-labs/solana-program-library/blob/ce8e4d565edcbd26e75d00d0e34e9d5f9786a646/token/program-2022/src/extension/cpi_guard/mod.rs#L24), яка зберігає логічне значення (boolean). Це значення вказує, чи активовано захист, чи ні. Розширення CPI Guard має лише дві інструкції: `Enable` і `Disable`. Вони змінюють це логічне значення.

```rust
pub struct CpiGuard {
    /// Lock privileged token operations from happening via CPI
    pub lock_cpi: PodBool,
}
```
CPI Guard має дві додаткові допоміжні функції, які програма `Token Extensions Program` використовує для визначення, коли CPI Guard активовано і коли інструкція виконується як частина крос-програмного виклику (CPI). Перша функція, `cpi_guard_enabled()`, просто повертає поточне значення поля `CpiGuard.lock_cpi`, якщо розширення існує на акаунті, в іншому випадку повертає false. Інші частини програми можуть використовувати цю функцію для визначення, чи активовано захист.

```rust
/// Determine if CPI Guard is enabled for this account
pub fn cpi_guard_enabled(account_state: &StateWithExtensionsMut<Account>) -> bool {
    if let Ok(extension) = account_state.get_extension::<CpiGuard>() {
        return extension.lock_cpi.into();
    }
    false
}
```

Друга допоміжна функція називається `in_cpi()` і визначає, чи знаходиться поточна інструкція в межах CPI. Функція може визначити, чи знаходиться вона зараз в CPI, викликаючи [`get_stack_height()` з бібліотеки `solana_program`](https://docs.rs/solana-program/latest/solana_program/instruction/fn.get_stack_height.html). Ця функція повертає поточну висоту стеку інструкцій. Інструкції, створені на рівні початкової транзакції, матимуть висоту [`TRANSACTION_LEVEL_STACK_HEIGHT`](https://docs.rs/solana-program/latest/solana_program/instruction/constant.TRANSACTION_LEVEL_STACK_HEIGHT.html) або 1. Перша внутрішня викликана транзакція або CPI матиме висоту `TRANSACTION_LEVEL_STACK_HEIGHT` + 1 і так далі. З цією інформацією ми знаємо, що якщо `get_stack_height()` повертає значення, більше ніж `TRANSACTION_LEVEL_STACK_HEIGHT`, ми зараз знаходимося в CPI! Саме це перевіряє функція `in_cpi()`. Якщо `get_stack_height() > TRANSACTION_LEVEL_STACK_HEIGHT`, вона повертає `True`. В іншому випадку, вона повертає `False`.

```rust
/// Determine if we are in CPI
pub fn in_cpi() -> bool {
    get_stack_height() > TRANSACTION_LEVEL_STACK_HEIGHT
}
```

Використовуючи ці дві допоміжні функції, програма `Token Extensions Program` може легко визначити, чи повинна вона відхилити інструкцію чи ні.

## Перемикання CPI Guard

Щоб увімкнути/вимкнути CPI Guard, токен-акаунт повинен бути ініціалізований для цього конкретного розширення. Потім можна надіслати інструкцію для активації CPI Guard. Це можна зробити тільки з клієнта. _Ви не можете перемикати CPI Guard через CPI_. Інструкція `Enable` [перевіряє, чи була вона викликана через CPI, і повертає помилку, якщо це так](https://github.com/solana-labs/solana-program-library/blob/ce8e4d565edcbd26e75d00d0e34e9d5f9786a646/token/program-2022/src/extension/cpi_guard/processor.rs#L44). Це означає, що лише кінцевий користувач може перемикати CPI Guard.

```rust
// inside process_toggle_cpi_guard()
if in_cpi() {
    return Err(TokenError::CpiGuardSettingsLocked.into());
}
```

Ви можете увімкнути CPI, використовуючи пакунок [`@solana/spl-token`](https://solana-labs.github.io/solana-program-library/token/js/modules.html) для Typescript. Ось приклад.

```typescript
// create token account with the CPI Guard extension
const tokenAccount = tokenAccountKeypair.publicKey;
const extensions = [
  ExtensionType.CpiGuard,
];
const tokenAccountLen = getAccountLen(extensions);
const lamports = await connection.getMinimumBalanceForRentExemption(tokenAccountLen);

const createTokenAccountInstruction = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: tokenAccount,
  space: tokenAccountLen,
  lamports,
  programId: TOKEN_2022_PROGRAM_ID,
});

// create 'enable CPI Guard' instruction
const enableCpiGuardInstruction =
  createEnableCpiGuardInstruction(tokenAccount, owner.publicKey, [], TOKEN_2022_PROGRAM_ID)

const initializeAccountInstruction = createInitializeAccountInstruction(
  tokenAccount,
  mint,
  owner.publicKey,
  TOKEN_2022_PROGRAM_ID,
);

// construct transaction with these instructions
const transaction = new Transaction().add(
  createTokenAccountInstruction,
  initializeAccountInstruction,
  enableCpiGuardInstruction,
);

transaction.feePayer = payer.publicKey;
// Send transaction
await sendAndConfirmTransaction(
  connection,
  transaction,
  [payer, owner, tokenAccountKeypair],
)
```

Ви також можете використовувати допоміжні функції [`enableCpiGuard`](https://solana-labs.github.io/solana-program-library/token/js/functions/enableCpiGuard.html) та [`disableCpiGuard`](https://solana-labs.github.io/solana-program-library/token/js/functions/disableCpiGuard.html) з API `@solana/spl-token` після ініціалізації акаунта.

```typescript
// enable CPI Guard
await enableCpiGuard(
  connection, // connection
  payer, // payer
  userTokenAccount.publicKey, // account
  payer, // owner
  [] // multiSigners
)

// disable CPI Guard
await disableCpiGuard(
  connection, // connection
  payer, // payer
  userTokenAccount.publicKey, // account
  payer, // owner
  [] // multiSigners
)
```

## Захист CPI Guard

### Трансфер

Функція трансферу в CPI Guard запобігає авторизації інструкції трансферу будь-ким, окрім делегата акаунта. Це реалізовано в різних функціях трансферу в програмі `Token Extensions Program`. Наприклад, [переглянувши інструкцію `transfer`](https://github.com/solana-labs/solana-program-library/blob/ce8e4d565edcbd26e75d00d0e34e9d5f9786a646/token/program-2022/src/processor.rs#L428), ми бачимо перевірку, яка поверне помилку, якщо будуть виконані певні умови.

Використовуючи допоміжні функції, про які ми говорили раніше, програма може визначити, чи слід викидати помилку чи ні.

```rust
// inside process_transfer in the token extensions program
if let Ok(cpi_guard) = source_account.get_extension::<CpiGuard>() {
    if cpi_guard.lock_cpi.into() && in_cpi() {
        return Err(TokenError::CpiGuardTransferBlocked.into());
    }
}
```

Цей захист означає, що навіть власник токен-акаунта не може перевести токени з акаунта, якщо інший акаунт є авторизованим делегатом.

### ### Спалювання

Цей захист CPI також забезпечує, що лише делегат акаунта може спалювати токени з токен-акаунта, подібно до захисту під час трансферу.

Функція `process_burn` в програмі `Token Extension Program` працює так само, як і інструкції трансферу. Вона [повертає помилку за тими ж умовами](https://github.com/solana-labs/solana-program-library/blob/ce8e4d565edcbd26e75d00d0e34e9d5f9786a646/token/program-2022/src/processor.rs#L1076).

```rust
// inside process_burn in the token extensions program
if let Ok(cpi_guard) = source_account.get_extension::<CpiGuard>() {
    if cpi_guard.lock_cpi.into() && in_cpi() {
        return Err(TokenError::CpiGuardBurnBlocked.into());
    }
}
```

Цей захист означає, що навіть власник токен-акаунта не може спалювати токени з акаунта, якщо інший акаунт є авторизованим делегатом.

### Підтвердження

CPI Guard запобігає підтвердженню делегата токен-акаунта через CPI. Ви можете затвердити делегата через клієнтську інструкцію, але не через CPI. Функція `process_approve` програми `Token Extension Program` виконує [ті ж самі перевірки, щоб визначити, чи увімкнено захист і чи виконується інструкція в межах CPI](https://github.com/solana-labs/solana-program-library/blob/ce8e4d565edcbd26e75d00d0e34e9d5f9786a646/token/program-2022/src/processor.rs#L583).

Це означає, що кінцевий користувач не ризикує підписати транзакцію, яка в обхід затверджує делегата на їх токен-акаунт без їх відома. Раніше користувач був залежний від свого гаманця, щоб попередити про такі транзакції заздалегідь.

### Закриття

Щоб закрити токен-акаунт через CPI, наявність захисту означає, що програма `Token Extensions Program` перевірить, чи є [акаунт призначення, який отримує лямпорти токен-акаунта, власником акаунта](https://github.com/solana-labs/solana-program-library/blob/ce8e4d565edcbd26e75d00d0e34e9d5f9786a646/token/program-2022/src/processor.rs#L1128).

Ось точний фрагмент коду з функції `process_close_account`.

```rust
if !source_account
    .base
    .is_owned_by_system_program_or_incinerator()
{
    if let Ok(cpi_guard) = source_account.get_extension::<CpiGuard>() {
        if cpi_guard.lock_cpi.into()
            && in_cpi()
            && !cmp_pubkeys(destination_account_info.key, &source_account.base.owner)
        {
            return Err(TokenError::CpiGuardCloseAccountBlocked.into());
        }
    }
...
}
```
Цей захист оберігає користувача від підписання транзакції, яка закриває токен-акаунт, що йому належить, і передає лямпорти цього акаунта на інший акаунт через CPI. Це було б важко виявити з боку кінцевого користувача без перевірки самих інструкцій. Цей захист гарантує, що лямпорти будуть передані лише власнику акаунта під час його закриття через CPI.

### Надання прав закриття (Close Authority)

Захист CPI перешкоджає встановленню прав для закриття акаунта через CPI, однак можна скасувати раніше встановлені права для закриття акаунта. Програма розширень токенів забезпечує це, перевіряючи, чи було передано значення в параметрі `new_authority` функції `process_set_authority`.

```rust
AuthorityType::CloseAccount => {
    let authority = account.base.close_authority.unwrap_or(account.base.owner);
    Self::validate_owner(
        program_id,
        &authority,
        authority_info,
        authority_info_data_len,
        account_info_iter.as_slice(),
    )?;

    if let Ok(cpi_guard) = account.get_extension::<CpiGuard>() {
        if cpi_guard.lock_cpi.into() && in_cpi() && new_authority.is_some() {
            return Err(TokenError::CpiGuardSetAuthorityBlocked.into());
        }
    }

    account.base.close_authority = new_authority;
}
```

Цей захист запобігає тому, щоб користувач підписав транзакцію, яка дає іншому акаунту можливість закрити його Token акаунт непомітно.

### Встановлення власника 

CPI Guard запобігає зміні власника акаунта за будь-яких обставин, як через CPI, так і без нього. Власник акаунта оновлюється в тій самій функції `process_set_authority`, що й авторитет `CloseAccount`, згаданий у попередньому розділі. Якщо інструкція намагається оновити авторитет акаунта з увімкненим CPI Guard, [функція поверне одну з двох можливих помилок](https://github.com/solana-labs/solana-program-library/blob/ce8e4d565edcbd26e75d00d0e34e9d5f9786a646/token/program-2022/src/processor.rs#L662).

Якщо інструкція виконується через CPI, функція поверне помилку `CpiGuardSetAuthorityBlocked`. В іншому випадку буде повернута помилка `CpiGuardOwnerChangeBlocked`.

```rust
if let Ok(cpi_guard) = account.get_extension::<CpiGuard>() {
    if cpi_guard.lock_cpi.into() && in_cpi() {
        return Err(TokenError::CpiGuardSetAuthorityBlocked.into());
    } else if cpi_guard.lock_cpi.into() {
        return Err(TokenError::CpiGuardOwnerChangeBlocked.into());
    }
}
```

Цей захист запобігає зміні власника токен-акаунта в будь-який момент, коли він увімкнений.

# Лабораторна робота

Ця лабораторна робота зосереджена на написанні тестів у TypeScript, але для цього потрібно запускати програму локально проти цих тестів. Тому необхідно виконати кілька кроків, щоб забезпечити відповідне середовище на вашій машині для роботи програми. 

Ончейн-програму вже написано для вас, і вона включена в початковий код лабораторної роботи.

Ончейн-програма містить кілька інструкцій, які демонструють, від чого може захищати CPI Guard. Ми напишемо тести, які викликають ці інструкції як із увімкненим CPI Guard, так і з вимкненим.

Тести розділено на окремі файли в каталозі `/tests`. Кожен файл є окремим модульним тестом, який викликає конкретну інструкцію в нашій програмі та ілюструє дію конкретного CPI Guard.

Програма має п'ять інструкцій: `malicious_close_account`, `prohibited_approve_account`, `prohibited_set_authority`, `unauthorized_burn`, `set_owner`.

Кожна з цих інструкцій виконує CPI до `Token Extensions Program` і намагається виконати дію над вказаним токен-акаунтом, яка потенційно є шкідливою і може бути непомітною для підписанта оригінальної транзакції. Ми не будемо тестувати захист `Transfer`, оскільки він ідентичний захисту `Burn`.

### 1. Перевірка версій Solana/Anchor/Rust

У цій лабораторній роботі ми будемо взаємодіяти з програмою `Token Extensions Program`, і для цього вам необхідно мати версію Solana CLI ≥ 1.18.0.

Щоб перевірити вашу версію, виконайте:
```bash
solana --version
```

Якщо після виконання команди `solana --version` відображається версія, менша за `1.18.0`, ви можете вручну оновити CLI. Зверніть увагу, що на момент написання цього тексту команда `solana-install update` не оновлює CLI до потрібної версії, тому необхідно явно завантажити версію `1.18.0`. Це можна зробити за допомогою наступної команди:

```bash
solana-install init 1.18.0
```

Якщо під час спроби зібрати програму у вас виникає ця помилка, це, ймовірно, означає, що у вас не встановлена правильна версія Solana CLI.

```bash
anchor build
error: package `solana-program v1.18.0` cannot be built because it requires rustc 1.72.0 or newer, while the currently active rustc version is 1.68.0-dev
Either upgrade to rustc 1.72.0 or newer, or use
cargo update -p solana-program@1.18.0 --precise ver
where `ver` is the latest version of `solana-program` supporting rustc 1.68.0-dev
```

Вам також потрібно встановити версію `0.29.0` Anchor CLI. Ви можете виконати оновлення за допомогою avm, скориставшись інструкціями за посиланням: [https://www.anchor-lang.com/docs/avm](https://www.anchor-lang.com/docs/avm).

або просто виконайте команду
```bash
avm install 0.29.0
avm use 0.29.0
```

На момент написання остання версія Anchor CLI — це `0.29.0`.

Тепер ми можемо перевірити нашу версію Rust.

```bash
rustc --version
```

На момент написання використовувалася версія компілятора Rust `1.26.0`. Якщо ви хочете оновити, ви можете зробити це за допомогою `rustup` за інструкцією тут: [Rust Installation](https://doc.rust-lang.org/book/ch01-01-installation.html).

```bash
rustup update
```

Тепер у нас повинні бути встановлені всі правильні версії.

### 2. Візьміть стартовий код і додайте залежності

Давайте візьмемо стартову гілку.

```bash
git clone https://github.com/Unboxed-Software/solana-lab-cpi-guard
cd solana-lab-cpi-guard
git checkout starter
```

### 3. Оновіть Program ID та Anchor Keypair

Перебуваючи в стартовій гілці, виконайте команду:

`anchor keys sync`

Це замінить програмний ID у різних місцях на ваш новий програмний keypair.

Далі вкажіть шлях до keypair розробника у файлі `Anchor.toml`.

```toml
[provider]
cluster = "Localnet"
wallet = "~/.config/solana/id.json"
```

"~/.config/solana/id.json" is the most common keypair path, but if you're unsure, just run:

```bash
solana config get
```

### 4. Підтвердження, що програма компілюється

Давайте побудуємо стартовий код, щоб переконатися, що все налаштовано правильно. Якщо програма не компілюється, будь ласка, поверніться до попередніх кроків і перевірте їх ще раз.

```bash
anchor build
```

Ви можете безпечно ігнорувати build script попередження, вони зникнуть, коли ми додамо необхідний код.

Не соромтесь запустити надані тести, щоб переконатися, що інше середовище розробки налаштоване правильно. Вам потрібно буде встановити залежності для Node, використовуючи `npm` або `yarn`. Тести повинні пройти вдало, але наразі вони нічого не роблять.

```bash
yarn install
anchor test
```

### 5. Створення токена з CPI Guard

Перед тим, як писати тести, давайте створимо допоміжну функцію, яка створюватиме обліковий запис токена з розширенням CPI Guard. Для цього створимо новий файл `tests/token-helper.ts` і нову функцію з назвою `createTokenAccountWithCPIGuard`.

Internally, this function will call:
- `SystemProgram.createAccount`: Allocates space for the token account
- `createInitializeAccountInstruction`: Initializes the token account
- `createEnableCpiGuardInstruction`: Enables the CPI Guard

```ts
import {
  ExtensionType,
  TOKEN_2022_PROGRAM_ID,
  createEnableCpiGuardInstruction,
  createInitializeAccountInstruction,
  getAccountLen,
} from "@solana/spl-token";
import {
  Connection,
  Keypair,
  PublicKey,
  SystemProgram,
  Transaction,
  sendAndConfirmTransaction,
} from "@solana/web3.js";

export async function createTokenAccountWithCPIGuard(
  connection: Connection,
  payer: Keypair,
  owner: Keypair,
  tokenAccountKeypair: Keypair,
  mint: PublicKey,
): Promise<string> {
  const tokenAccount = tokenAccountKeypair.publicKey;

  const extensions = [ExtensionType.CpiGuard];

  const tokenAccountLen = getAccountLen(extensions);
  const lamports = await connection.getMinimumBalanceForRentExemption(
    tokenAccountLen
  );

  const createTokenAccountInstruction = SystemProgram.createAccount({
    fromPubkey: payer.publicKey,
    newAccountPubkey: tokenAccount,
    space: tokenAccountLen,
    lamports,
    programId: TOKEN_2022_PROGRAM_ID,
  });

  const initializeAccountInstruction = createInitializeAccountInstruction(
    tokenAccount,
    mint,
    owner.publicKey,
    TOKEN_2022_PROGRAM_ID
  );

  const enableCpiGuardInstruction = createEnableCpiGuardInstruction(
    tokenAccount,
    owner.publicKey,
    [],
    TOKEN_2022_PROGRAM_ID
  );

  const transaction = new Transaction().add(
    createTokenAccountInstruction,
    initializeAccountInstruction,
    enableCpiGuardInstruction
  );

  transaction.feePayer = payer.publicKey;

  // Send transaction
  return await sendAndConfirmTransaction(connection, transaction, [
    payer,
    owner,
    tokenAccountKeypair,
  ]);
}
```

### 5. Затвердження делегата

Першим CPI Guard, який ми будемо тестувати, є функція затвердження делегата. CPI Guard забороняє затверджувати делегата облікового запису токена з увімкненим CPI Guard через CPI. Важливо зазначити, що можна затвердити делегата на CPI Guard акаунті, але не за допомогою CPI. Для цього потрібно надіслати інструкцію безпосередньо до `Token Extensions Program` з клієнта, а не через іншу програму.

Перед тим, як написати наш тест, нам потрібно ознайомитися з кодом програми, яку ми тестуємо. В цьому випадку звертаємо увагу на нструкцію `prohibited_approve_account`.

```rust
// inside src/lib.rs
pub fn prohibited_approve_account(ctx: Context<ApproveAccount>, amount: u64) -> Result<()> {
    msg!("Invoked ProhibitedApproveAccount");

    msg!(
        "Approving delegate: {} to transfer up to {} tokens.",
        ctx.accounts.delegate.key(),
        amount
    );

    approve(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Approve {
                to: ctx.accounts.token_account.to_account_info(),
                delegate: ctx.accounts.delegate.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        amount,
    )
}
...

#[derive(Accounts)]
pub struct ApproveAccount<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        mut,
        token::token_program = token_program,
        token::authority = authority
    )]
    pub token_account: InterfaceAccount<'info, token_interface::TokenAccount>,
    /// CHECK: delegat to approve
    #[account(mut)]
    pub delegate: AccountInfo<'info>,
    pub token_program: Interface<'info, token_interface::TokenInterface>,
}
```

Якщо ви знайомі з Solana програмами, то ця інструкція виглядатиме досить простою. Інструкція очікує на акаунт `authority` як `Signer` та `token_account`, на який цей `authority` має права.

Далі інструкція викликає інструкцію `Approve` в `Token Extensions Program` і намагається призначити `delegate` як делегата для вказаного `token_account`.

Давайте відкриємо файл `/tests/approve-delegate-example.ts`, щоб почати тестування цієї інструкції. Ознайомтеся з початковим кодом. У нас є платник, деякі тестові ключові пари та функція `airdropIfRequired`, яка буде виконуватися перед тестами.

Коли ви ознайомитесь з початковим кодом, ми можемо перейти до тестів "Approve Delegate". Ми створимо тести, що викликають ту саму інструкцію в нашій цільовій програмі, з увімкненим і вимкненим CPI guard.

Щоб протестувати нашу інструкцію, спершу потрібно створити наш токен-мінт і токен-акаунт з розширеннями.

```typescript
it("stops 'Approve Delegate' when CPI guard is enabled", async () => {

  await createMint(
    provider.connection,
    payer,
    provider.wallet.publicKey,
    undefined,
    6,
    testTokenMint,
    undefined,
    TOKEN_2022_PROGRAM_ID
  )
  await createTokenAccountWithCPIGuard(
    provider.connection,
    payer,
    payer,
    userTokenAccount,
    testTokenMint.publicKey
  )

})
```

Тепер давайте надішлемо транзакцію до нашої програми, яка спробує викликати інструкцію "Approve delegate" у `Token Extensions Program`.

```typescript
// inside "allows 'Approve Delegate' when CPI guard is disabled" test block
try {
  const tx = await program.methods.prohibitedApproveAccount(new anchor.BN(1000))
    .accounts({
      authority: payer.publicKey,
      tokenAccount: userTokenAccount.publicKey,
      delegate: maliciousAccount.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();

  console.log("Your transaction signature", tx);
} catch (e) {
  assert(e.message == "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2d")
  console.log("CPI Guard is enabled, and a program attempted to approve a delegate");
}
```

Зверніть увагу, що ми обертаємо це в блок `try/catch`. Оскільки, ця інструкція повинна завершитися з помилкою, якщо CPI Guard працює правильно. Ми ловимо помилку і перевіряємо, що повідомлення про помилку відповідає очікуваному.

Тепер ми фактично робимо те ж саме для тесту `"дозволяє 'Approve Delegate', коли CPI guard вимкнено"`, за винятком того, що хочемо передати токен-акаунт без включеного CPI Guard. Для цього ми можемо просто вимкнути CPI Guard на `userTokenAccount` і повторно надіслати транзакцію.

```typescript
it("allows 'Approve Delegate' when CPI guard is disabled", async () => {
  await disableCpiGuard(
    provider.connection,
    payer,
    userTokenAccount.publicKey,
    payer,
    [],
    undefined,
    TOKEN_2022_PROGRAM_ID
  )

  await program.methods.prohibitedApproveAccount(new anchor.BN(1000))
    .accounts({
      authority: payer.publicKey,
      tokenAccount: userTokenAccount.publicKey,
      delegate: maliciousAccount.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();
})
```
Ця транзакція успішно виконається, і обліковий запис `delegate` тепер матиме повноваження переводити задану кількість токенів з `userTokenAccount`.

Не соромтеся зберегти свою роботу та запустити команду `anchor test`. Усі тести будуть виконані, але поки що лише ці два тести щось роблять. Обидва тести повинні пройти успішно на цьому етапі.

### 6. Close Account

Інструкція "close account" викликає інструкцію `close_account` в програмі `Token Extensions Program`, що дозволяє закрити вказаний обліковий запис токена. Однак ви маєте можливість визначити, на який обліковий запис мають бути переведені лампорти, отримані за оренду. CPI Guard гарантує, що ці лампорти будуть завжди переведені на обліковий запис власника.

```rust
pub fn malicious_close_account(ctx: Context<MaliciousCloseAccount>) -> Result<()> {
    msg!("Invoked MaliciousCloseAccount");

    msg!(
        "Token account to close : {}",
        ctx.accounts.token_account.key()
    );

    close_account(CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        CloseAccount {
            account: ctx.accounts.token_account.to_account_info(),
            destination: ctx.accounts.destination.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    ))
}

...

#[derive(Accounts)]
pub struct MaliciousCloseAccount<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        mut,
        token::token_program = token_program,
        token::authority = authority
    )]
    pub token_account: InterfaceAccount<'info, token_interface::TokenAccount>,
    /// CHECK: malicious account
    #[account(mut)]
    pub destination: AccountInfo<'info>,
    pub token_program: Interface<'info, token_interface::TokenInterface>,
    pub system_program: Program<'info, System>,
}
```

Нашу програму просто викликає інструкція `close_account`, але потенційно зловмисний клієнт може замінити акаунт власника токену і відправити інший, як обліковий запис `destination`. Це буде важко помітити з боку користувача, якщо лише гаманець не сповістить про це. Якщо CPI Guards увімкнено, програма `Token Extension Program` просто відхилить інструкцію в такому випадку.

Щоб протестувати, відкриємо файл `/tests/close-account-example.ts`. Початковий код тут такий самий, як і в попередньому тесті.

Спочатку створимо наш мінт і токен-акаунт з CPI Guard:

```typescript
it("stops 'Close Account' when CPI guard in enabled", async () => {
  await createMint(
    provider.connection,
    payer,
    provider.wallet.publicKey,
    undefined,
    6,
    testTokenMint,
    undefined,
    TOKEN_2022_PROGRAM_ID
  )
  await createTokenAccountWithCPIGuard(
    provider.connection,
    payer,
    payer,
    userTokenAccount,
    testTokenMint.publicKey
  )
})
```

Тепер надішлемо транзакцію до нашої інструкції `malicious_close_account`. Оскільки ми увімкнули CPI Guard на цьому токен-аккаунті, транзакція має не вдатися. Наш тест перевіряє, що вона не вдалася з очікуваної причини.

```typescript
// inside "stops 'Close Account' when CPI guard in enabled" test block
try {
  const tx = await program.methods.maliciousCloseAccount()
    .accounts({
      authority: payer.publicKey,
      tokenAccount: userTokenAccount.publicKey,
      destination: maliciousAccount.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();

  console.log("Your transaction signature", tx);
} catch (e) {
  assert(e.message == "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2c")
  console.log("CPI Guard is enabled, and a program attempted to close an account without returning lamports to owner");
}
```

Тепер ми можемо вимкнути CPI Guard і надіслати точно таку ж транзакцію в тесті "Close Account without CPI Guard". Ця транзакція повинна успішно виконатись.

```typescript
it("Close Account without CPI Guard", async () => {
  await disableCpiGuard(
    provider.connection,
    payer,
    userTokenAccount.publicKey,
    payer,
    [],
    undefined,
    TOKEN_2022_PROGRAM_ID
  )

  await program.methods.maliciousCloseAccount()
    .accounts({
      authority: payer.publicKey,
      tokenAccount: userTokenAccount.publicKey,
      destination: maliciousAccount.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();
})
```

### 7. Set Authority (Налаштування прав) 

Перехід до інструкції `prohibited_set_authority`, CPI Guard захищає від зміни авторитету `CloseAccount` через CPI.

```rust
pub fn prohibted_set_authority(ctx: Context<SetAuthorityAccount>) -> Result<()> {
    msg!("Invoked ProhibitedSetAuthority");

    msg!(
        "Setting authority of token account: {} to address: {}",
        ctx.accounts.token_account.key(),
        ctx.accounts.new_authority.key()
    );

    set_authority(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            SetAuthority {
                current_authority: ctx.accounts.authority.to_account_info(),
                account_or_mint: ctx.accounts.token_account.to_account_info(),
            },
        ),
        spl_token_2022::instruction::AuthorityType::CloseAccount,
        Some(ctx.accounts.new_authority.key()),
    )
}

#[derive(Accounts)]
pub struct SetAuthorityAccount<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        mut,
        token::token_program = token_program,
        token::authority = authority
    )]
    pub token_account: InterfaceAccount<'info, token_interface::TokenAccount>,
    /// CHECK: delegat to approve
    #[account(mut)]
    pub new_authority: AccountInfo<'info>,
    pub token_program: Interface<'info, token_interface::TokenInterface>,
}
```

Наша інструкція програми просто викликає інструкцію `SetAuthority` і вказує, що ми хочемо встановити права `spl_token_2022::instruction::AuthorityType::CloseAccount` для вказаного токен-аккаунта.

Відкрийте файл `/tests/set-authority-example.ts`. Стартовий код такий самий, як і в попередніх тестах.

Давайте створимо наш токен-мінт і токен-аккаунт з увімкненим CPI Guard. Потім ми можемо надіслати транзакцію до нашої інструкції `prohibited_set_authority`.

```typescript
it("sets authority when CPI guard in enabled", async () => {

  await createMint(
    provider.connection,
    payer,
    provider.wallet.publicKey,
    undefined,
    6,
    testTokenMint,
    undefined,
    TOKEN_2022_PROGRAM_ID
  );
  await createTokenAccountWithCPIGuard(
    provider.connection,
    payer,
    payer,
    userTokenAccount,
    testTokenMint.publicKey
  );

  try {
    const tx = await program.methods
      .prohibtedSetAuthority()
      .accounts({
        authority: payer.publicKey,
        tokenAccount: userTokenAccount.publicKey,
        newAuthority: maliciousAccount.publicKey,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
      })
      .signers([payer])
      .rpc();

    console.log("Your transaction signature", tx);
  } catch (e) {
    assert(
      e.message ==
      "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2e"
    );
    console.log(
      "CPI Guard is enabled, and a program attempted to add or change an authority"
    );
  }
});
```

Для тесту `"Set Authority Example"` ми можемо вимкнути CPI Guard і знову надіслати транзакцію.

```typescript
it("Set Authority Example", async () => {
  await disableCpiGuard(
    provider.connection,
    payer,
    userTokenAccount.publicKey,
    payer,
    [],
    undefined,
    TOKEN_2022_PROGRAM_ID
  )

  await program.methods.prohibtedSetAuthority()
    .accounts({
      authority: payer.publicKey,
      tokenAccount: userTokenAccount.publicKey,
      newAuthority: maliciousAccount.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();
})
```

### 8. Burn (Спалювання)

Наступна інструкція, яку ми будемо тестувати, — це інструкція `unauthorized_burn` з нашої тестової програми. Ця інструкція викликає інструкцію `burn` з програми `Token Extensions Program` і намагається спалити певну кількість токенів з вказаного токен-аккаунта.

CPI Guard гарантує, що це можливо лише в тому випадку, якщо підписувачем є делегат токен-аккаунта.

```rust
pub fn unauthorized_burn(ctx: Context<BurnAccounts>, amount: u64) -> Result<()> {
    msg!("Invoked Burn");

    msg!(
        "Burning {} tokens from address: {}",
        amount,
        ctx.accounts.token_account.key()
    );

    burn(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Burn {
                mint: ctx.accounts.token_mint.to_account_info(),
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        amount,
    )
}

...

#[derive(Accounts)]
pub struct BurnAccounts<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        mut,
        token::token_program = token_program,
        token::authority = authority
    )]
    pub token_account: InterfaceAccount<'info, token_interface::TokenAccount>,
    #[account(
        mut,
        mint::token_program = token_program
    )]
    pub token_mint: InterfaceAccount<'info, token_interface::Mint>,
    pub token_program: Interface<'info, token_interface::TokenInterface>,
}
```
Щоб протестувати, відкрийте файл `tests/burn-example.ts`. Стартовий код такий самий, як і в попередніх тестах, але ми замінили `maliciousAccount` на `delegate`.

Далі ми можемо створити наш мінт і токен-аккаунт з увімкненим CPI Guard.

```typescript
it("stops 'Burn' without a delegate signature", async () => {
  await createMint(
    provider.connection,
    payer,
    provider.wallet.publicKey,
    undefined,
    6,
    testTokenMint,
    undefined,
    TOKEN_2022_PROGRAM_ID
  );

  await createTokenAccountWithCPIGuard(
    provider.connection,
    payer,
    payer,
    userTokenAccount,
    testTokenMint.publicKey
  );
})
```

Now, let's mint some tokens to our test account.

```typescript
// inside "stops 'Burn' without a delegate signature" test block
const mintToTx = await mintTo(
  provider.connection,
  payer,
  testTokenMint.publicKey,
  userTokenAccount.publicKey,
  payer,
  1000,
  undefined,
  undefined,
  TOKEN_2022_PROGRAM_ID
)
```

Тепер давайте підтвердимо делегата для нашого токен-акаунту. Цей токен-акаунт має увімкнений CPI Guard, але ми все ще можемо підтвердити делегата. Це відбувається тому, що ми робимо це, викликаючи `Token Extensions Program` безпосередньо, а не через CPI, як у нашому попередньому прикладі.

```typescript
// inside "stops 'Burn' without a delegate signature" test block
const approveTx = await approve(
  provider.connection,
  payer,
  userTokenAccount.publicKey,
  delegate.publicKey,
  payer,
  500,
  undefined,
  undefined,
  TOKEN_2022_PROGRAM_ID
)
```

Тепер, коли ми призначили делегата для нашого токен-акаунту, ми можемо надіслати транзакцію до нашої програми, щоб спробувати спалити деяку кількість токенів. Ми будемо передавати аккаунт `payer` як авторитет. Цей аккаунт є власником `userTokenAccount`, але оскільки ми призначили аккаунт `delegate` як делегата, CPI Guard запобіжить проходженню цієї транзакції.

```typescript
// inside "stops 'Burn' without a delegate signature" test block
try {
  const tx = await program.methods.unauthorizedBurn(new anchor.BN(500))
    .accounts({
      // payer is not the delegate
      authority: payer.publicKey,
      tokenAccount: userTokenAccount.publicKey,
      tokenMint: testTokenMint.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();

  console.log("Your transaction signature", tx);
} catch (e) {
  assert(e.message == "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2b")
  console.log("CPI Guard is enabled, and a program attempted to burn user funds without using a delegate.");
}
```

Для тесту `"Burn without Delegate Signature Example"` ми просто вимкнемо CPI Guard і знову надішлемо транзакцію.

```typescript
it("Burn without Delegate Signature Example", async () => {
  await disableCpiGuard(
    provider.connection,
    payer,
    userTokenAccount.publicKey,
    payer,
    [],
    undefined,
    TOKEN_2022_PROGRAM_ID
  )

  const tx = await program.methods.unauthorizedBurn(new anchor.BN(500))
    .accounts({
      // payer is not the delegate
      authority: payer.publicKey,
      tokenAccount: userTokenAccount.publicKey,
      tokenMint: testTokenMint.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();
})
```

### 9. Set Owner (Встановлення Власника)

Останній захист, який ми протестуємо для CPI Guard, це захист від зміни власника (SetOwner). Коли CPI Guard увімкнено, ця дія завжди заборонена, навіть поза контекстом CPI. Для тестування ми спробуємо змінити власника токен-аккаунту як з боку клієнта, так і через CPI за допомогою нашої тестової програми.

Ось інструкція програми.

```rust
pub fn set_owner(ctx: Context<SetOwnerAccounts>) -> Result<()> {
    msg!("Invoked SetOwner");

    msg!(
        "Setting owner of token account: {} to address: {}",
        ctx.accounts.token_account.key(),
        ctx.accounts.new_owner.key()
    );

    set_authority(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            SetAuthority {
                current_authority: ctx.accounts.authority.to_account_info(),
                account_or_mint: ctx.accounts.token_account.to_account_info(),
            },
        ),
        spl_token_2022::instruction::AuthorityType::AccountOwner,
        Some(ctx.accounts.new_owner.key()),
    )
}

#[derive(Accounts)]
pub struct SetOwnerAccounts<'info> {
    #[account(mut)]
    pub authority: Signer<'info>,
    #[account(
        mut,
        token::token_program = token_program,
        token::authority = authority
    )]
    pub token_account: InterfaceAccount<'info, token_interface::TokenAccount>,
    /// CHECK: delegat to approve
    #[account(mut)]
    pub new_owner: AccountInfo<'info>,
    pub token_program: Interface<'info, token_interface::TokenInterface>,
}
```

Відкрийте файл `/tests/set-owner-example.ts`. Ми напишемо чотири тести для цього: два для зміни власника без CPI та два для зміни власника через CPI.

Зверніть увагу, що ми видалили `delegate` і додали `firstNonCPIGuardAccount`, `secondNonCPIGuardAccount` та `newOwner`.

Починаючи з першого тесту `"зупиняє 'Set Authority' без CPI на обліковому записі з CPI-guard"`, створимо мінт і токен-аккаунт з CPI-guard.

```typescript
it("stops 'Set Authority' without CPI on a CPI-guarded account", async () => {
  await createMint(
    provider.connection,
    payer,
    provider.wallet.publicKey,
    undefined,
    6,
    testTokenMint,
    undefined,
    TOKEN_2022_PROGRAM_ID
  );

  await createTokenAccountWithCPIGuard(
    provider.connection,
    payer,
    payer,
    userTokenAccount,
    testTokenMint.publicKey
  );
})
```

Далі ми спробуємо надіслати транзакцію до інструкції `set_authority` програми `Token Extensions Program` за допомогою функції `setAuthority` з бібліотеки `@solana/spl-token`.

```typescript
// inside the "stops 'Set Authority' without CPI on a CPI-guarded account" test block
try {
  await setAuthority(
    provider.connection,
    payer,
    userTokenAccount.publicKey,
    payer,
    AuthorityType.AccountOwner,
    newOwner.publicKey,
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID
  )
} catch (e) {
  assert(e.message == "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2f")
  console.log("Account ownership cannot be changed while CPI Guard is enabled.")
}
```

Ця транзакція повинна зазнати невдачі, тому ми обгортаємо виклик у блок `try/catch` і перевіряємо, чи є помилка очікуваною.

Далі ми створимо ще один токен-акаунт без увімкненого CPI Guard і спробуємо зробити те саме.

```typescript
it("Set Authority without CPI on Non-CPI Guarded Account", async () => {
  await createAccount(
    provider.connection,
    payer,
    testTokenMint.publicKey,
    payer.publicKey,
    firstNonCPIGuardAccount,
    undefined,
    TOKEN_2022_PROGRAM_ID
  )

  await setAuthority(
    provider.connection,
    payer,
    firstNonCPIGuardAccount.publicKey,
    payer,
    AuthorityType.AccountOwner,
    newOwner.publicKey,
    undefined,
    undefined,
    TOKEN_2022_PROGRAM_ID
  )
})
```

Цей тест має пройти успішно.

Тепер давайте перевіримо це за допомогою CPI. Для цього нам потрібно просто надіслати транзакцію до інструкції `set_owner` нашої програми.

```typescript
it("[CPI Guard] Set Authority via CPI on CPI Guarded Account", async () => {
  try {
    await program.methods.setOwner()
      .accounts({
        authority: payer.publicKey,
        tokenAccount: userTokenAccount.publicKey,
        newOwner: newOwner.publicKey,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
      })
      .signers([payer])
      .rpc();

  } catch (e) {
    assert(e.message == "failed to send transaction: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x2e")
    console.log("CPI Guard is enabled, and a program attempted to add or change an authority.")
  }
})
```

Останнім кроком ми можемо створити ще один токен-акаунт без увімкненого CPI Guard і передати його до інструкції програми. Цього разу CPI має пройти успішно.

```typescript
it("Set Authority via CPI on Non-CPI Guarded Account", async () => {
  await createAccount(
    provider.connection,
    payer,
    testTokenMint.publicKey,
    payer.publicKey,
    secondNonCPIGuardAccount,
    undefined,
    TOKEN_2022_PROGRAM_ID
  );

  await program.methods.setOwner()
    .accounts({
      authority: payer.publicKey,
      tokenAccount: secondNonCPIGuardAccount.publicKey,
      newOwner: newOwner.publicKey,
      tokenProgram: TOKEN_2022_PROGRAM_ID,
    })
    .signers([payer])
    .rpc();
})
```

І ось і все! Тепер ви можете зберегти свою роботу та запустити команду `anchor test`. Усі тести, які ми написали, повинні пройти успішно.

# Завдання

Напишіть тести для функціональності Transfer.
