---
назва: Змінні середовища у програмах Solana
завдання:  
- Визначити функції програми у файлі `Cargo.toml`.  
- Використовувати атрибут `cfg` у Rust для умовної компіляції коду залежно від того, які функції ввімкнені або вимкнені
- Використовувати макрос `cfg!` у Rust для умовної компіляції коду залежно від того, які функції ввімкнені або вимкнені
- Створити інструкцію лише для адміністратора для налаштування акаунту програми, який може використовуватися для збереження значень конфігурації програми
---

# Стислий виклад  

- Не існує готових рішень для створення окремих середовищ в ончейн-програмі, але ви можете досягти чогось подібного до змінних середовища, якщо підійдете до цього креативно.
-   Ви можете використовувати атрибут `cfg` разом із **функціями Rust** (`#[cfg(feature = ...)]`), щоб запускати різний код або надавати різні значення змінних залежно від заданої функції Rust. _Це відбувається під час компіляції й не дозволяє змінювати значення після розгортання програми_.  
-   Так само ви можете використовувати **макрос** `cfg!`, щоб компілювати різні шляхи коду залежно від увімкнених функцій.  
-   Альтернативно, ви можете досягти чогось подібного до змінних середовища, які можна змінювати після розгортання, створюючи акаунти та інструкції, доступні лише уповноваженим акаунтам, що уповноважені вносити зміни.  

# Урок

Однією з труднощів, з якими стикаються інженери у всіх типах розробки програмного забезпечення, є написання тестованого коду та створення окремих середовищ для локальної розробки, тестування, продакшену тощо.

Це може бути особливо складно в розробці програм Solana. Наприклад, уявіть собі створення програми для стейкінгу NFT, яка винагороджує кожен застейканий NFT 10 токенами винагороди на день. Як протестувати можливість отримання винагороди, якщо тести виконуються за кілька сотень мілісекунд, чого недостатньо для отримання винагороди?

Традиційна веброзробка вирішує це частково за допомогою змінних середовища, значення яких можуть відрізнятися в кожному окремому "середовищі". Наразі в Solana-програмах немає формальної концепції змінних середовища. Якби вони були, ви могли б зробити так, щоб у тестовому середовищі винагороди становили 10,000,000 токенів на день. Тоді було б легше тестувати можливість отримання винагород.

На щастя, ви можете досягти подібного функціоналу, якщо проявите креативність. Найкращий підхід, ймовірно, це комбінація двох речей:

1. Маркери функцій Rust, які дозволяють вам вказувати в команді збірки "середовище" збірки, разом із кодом, що відповідним чином коригує певні значення.
2. Акаунти лише для адміністраторів та інструкції програми, які доступні лише авторитету оновлення програми.

## Маркери функцій у Rust

Один із найпростіших способів створення середовищ — це використання функцій (features) Rust. Функції визначаються в секції `[features]` у файлі `Cargo.toml` програми. Ви можете визначити кілька функцій для різних випадків використання.

```toml
[features]
feature-one = []
feature-two = []
```

Важливо зазначити, що наведене вище лише визначає функцію (feature). Щоб увімкнути функцію під час тестування вашої програми, ви можете використовувати маркер `--features` разом із командою `anchor test`.

```bash
anchor test -- --features "feature-one"
```

Ви також можете вказати кілька функцій, розділивши їх комою.

```bash
anchor test -- --features "feature-one", "feature-two"
```

### Умовне включення коду за допомогою атрибута `cfg`

Після визначення функції (feature) ви можете використовувати атрибут `cfg` у своєму коді, щоб умовно компілювати код залежно від того, чи увімкнена певна функція. Це дозволяє включати або виключати певний код із вашої програми.

Синтаксис використання атрибута `cfg` схожий на будь-який інший макрос-атрибут: `#[cfg(feature = [FEATURE_HERE])]`. Наприклад, наступний код компілює функцію `function_for_testing`, якщо увімкнена функція `testing`, і функцію `function_when_not_testing` в іншому випадку:  

```rust
#[cfg(feature = "testing")]
fn function_for_testing() {
    // code that will be included only if the "testing" feature flag is enabled
}

#[cfg(not(feature = "testing"))]
fn function_when_not_testing() {
    // code that will be included only if the "testing" feature flag is not enabled
}
```

Це дозволяє вам увімкнути або вимкнути певний функціонал у вашій програмі Anchor під час компіляції, увімкнувши або вимкнувши функцію.

Не важко уявити, що ви можете використати це для створення різних "середовищ" для різних версій запуску програми. Наприклад, не всі токени є і в Mainnet, і в Devnet. Тому ви можете жорстко закодувати одну адресу токена для розгортання в Mainnet, але жорстко закодувати іншу адресу для розгортання в Devnet і Localnet. Таким чином, ви можете швидко перемикатися між різними середовищами, не вносячи жодних змін у сам код.

Нижче наведено приклад програми Anchor, яка використовує атрибут `cfg`, щоб включати різні адреси токенів для локального тестування порівняно з іншими запусками:

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[cfg(feature = "local-testing")]
pub mod constants {
    use solana_program::{pubkey, pubkey::Pubkey};
    pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("WaoKNLQVDyBx388CfjaVeyNbs3MT2mPgAhoCfXyUvg8");
}

#[cfg(not(feature = "local-testing"))]
pub mod constants {
    use solana_program::{pubkey, pubkey::Pubkey};
    pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v");
}

#[program]
pub mod test_program {
    use super::*;

    pub fn initialize_usdc_token_account(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = payer,
        token::mint = mint,
        token::authority = payer,
    )]
    pub token: Account<'info, TokenAccount>,
    #[account(address = constants::USDC_MINT_PUBKEY)]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}
```

У цьому прикладі атрибут `cfg` використовується для умовної компіляції двох різних реалізацій модуля `constants`. Це дозволяє програмі використовувати різні значення для константи `USDC_MINT_PUBKEY` залежно від того, чи увімкнена функція `local-testing`.

### Умовне включення коду за допомогою макросу `cfg!`

Подібно до атрибута `cfg`, макрос `cfg!` в Rust дозволяє перевіряти значення певних конфігураційних маркерів під час виконання програми. Це може бути корисно, якщо ви хочете виконувати різний код залежно від значень певних налаштувань.

Ви можете використати це, щоб обійти або налаштувати обмеження, пов'язані з часом, які потрібні для програми стейкінгу NFT, про яку ми згадували раніше. Під час виконання тесту можна виконувати код, що надає значно більші винагороди за стейкінг порівняно з виконанням продакшн-версії.

Щоб використати макрос `cfg!` в програмі Anchor, достатньо додати виклик макросу `cfg!` у відповідний вираз:

```rust
#[program]
pub mod my_program {
    use super::*;

    pub fn test_function(ctx: Context<Test>) -> Result<()> {
        if cfg!(feature = "local-testing") {
            // This code will be executed only if the "local-testing" feature is enabled
            // ...
        } else {
            // This code will be executed only if the "local-testing" feature is not enabled
            // ...
        }
        // Code that should always be included goes here
        ...
        Ok(())
    }
}
```

У цьому прикладі функція `test_function` використовує макрос `cfg!` для перевірки значення функції `local-testing` під час виконання програми. Якщо функція `local-testing` увімкнена, виконується перший шлях коду. Якщо функція `local-testing` не увімкнена, виконується другий шлях коду.

## Інструкції лише для адміністраторів

Маркери функцій чудово підходять для налаштування значень і шляхів коду під час компіляції, але вони не дуже корисні, якщо потрібно змінити щось після того, як програма вже розгорнута.

Наприклад, якщо ваша програма для стейкінгу NFT повинна змінити токен винагороди, не буде можливості оновити програму без повторного розгортання. Якби тільки був спосіб для адміністраторів програми оновлювати певні значення... О, це можливо!

По-перше, вам потрібно структурувати програму так, щоб значення, які ви плануєте змінити, зберігалися в акаунті, а не були жорстко закодовані в коді програми.

Далі потрібно забезпечити, щоб цей акаунт міг оновлювати лише відомий уповноважений акаунт програми або, як ми його називаємо, адміністратор. Це означає, що будь-які інструкції, що змінюють дані в цьому акаунті, повинні мати обмеження, що визначають, хто може підписувати інструкцію. Це звучить досить просто в теорії, але є одна основна проблема: як програма визначить, хто є авторизованим адміністратором?

Є кілька рішень, кожне з яких має свої переваги та недоліки:  

1. Жорстко закодувати публічний ключ адміністратора, який можна використовувати в обмеженнях інструкцій лише для адміністраторів.  
2. Зробити уповноважений акаунт програми адміністратором.  
3. Зберігати адміністратора в config-акаунті та встановлювати першого адміністратора за допомогою інструкції `initialize`.  

### Створення config-акаунту

Перший крок — додати до вашої програми те, що ми називаємо "config" акаунтом. Ви можете налаштувати його відповідно до ваших потреб, але ми пропонуємо використовувати єдиний глобальний PDA. У Anchor це просто означає створення структури акаунту та використання одного seed для визначення адреси акаунту.

```rust
pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[account]
pub struct ProgramConfig {
    reward_token: Pubkey,
    rewards_per_day: u64,
}
```

У наведеному прикладі показано гіпотетичний config-акаунт для програми стейкінгу NFT, про яку йшлося протягом уроку. Він зберігає дані, що представляють токен, який слід використовувати для винагород, і кількість токенів, що видаються за кожен день стейкінгу.

Після визначення config-акаунта переконайтеся, що решта вашого коду посилається на цей акаунт під час використання цих значень. Таким чином, якщо дані в акаунті зміняться, програма адаптується відповідно.

### Обмежити оновлення config-акаунту лише для жорстко закодованих адміністраторів

Вам потрібен спосіб ініціалізувати та оновлювати дані config-акаунту. Це означає, що потрібно створити одну або кілька інструкцій, які може викликати лише адміністратор. Найпростіший спосіб реалізувати це — жорстко закодувати публічний ключ адміністратора у вашому коді. Ви можете додати просту перевірку підписанта в інструкцію, яка порівнює підписанта з цим публічним ключем під час перевірки акаунту.

У Anchor обмеження інструкції `update_program_config` для використання лише жорстко закодованим адміністратором може виглядати так:

```rust
#[program]
mod my_program {
    pub fn update_program_config(
        ctx: Context<UpdateProgramConfig>,
        reward_token: Pubkey,
        rewards_per_day: u64
    ) -> Result<()> {
        ctx.accounts.program_config.reward_token = reward_token;
        ctx.accounts.program_config.rewards_per_day = rewards_per_day;

        Ok(())
    }
}

pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[constant]
pub const ADMIN_PUBKEY: Pubkey = pubkey!("ADMIN_WALLET_ADDRESS_HERE");

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = SEED_PROGRAM_CONFIG, bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(constraint = authority.key() == ADMIN_PUBKEY)]
    pub authority: Signer<'info>,
}
```

Перед виконанням логіки інструкції буде виконана перевірка, щоб переконатися, що підписант інструкції збігається з жорстко закодованим `ADMIN_PUBKEY`. Зверніть увагу, що в наведеному прикладі не показано інструкцію, яка ініціалізує config-акаунт, але вона повинна мати подібні обмеження, щоб запобігти тому, щоб зловмисник ініціалізував акаунт з непередбачуваними значеннями.

Хоча цей підхід працює, він вимагає відстеження гаманця адміністратора, а також гаманця для уповноваженого акаунту програми. З кількома додатковими рядками коду ви могли б просто обмежити інструкцію так, щоб її міг викликати тільки уповноважений акаунт програми. Єдина складність — отримати значення уповноваженого акаунту для порівняння.

### Обмежити оновлення config-акаунту уповноваженим акаунтом програми

На щастя, кожна програма має акаунт даних програми, який відповідає типу акаунту `ProgramData` в Anchor і має поле `upgrade_authority_address`. Програма зберігає адресу цього акаунта у своїх даних у полі `programdata_address`.

Отже, окрім двох акаунтів, які потрібні для інструкції в прикладі з жорстко закодованим адміністратором, ця інструкція також вимагає акаунтів `program` та `program_data`.

Ці акаунти повинні мати такі обмеження:

1. Обмеження на `program`, забезпечує, щоб наданий акаунт `program_data` відповідав полю `programdata_address` .  
2. Обмеження на акаунт `program_data`, забезпечує, щоб підписант інструкції відповідав полю `upgrade_authority_address` акаунту `program_data`.  

Коли це завершено, виглядає ось так:  

```rust
...

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = SEED_PROGRAM_CONFIG, bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(constraint = program.programdata_address()? == Some(program_data.key()))]
    pub program: Program<'info, MyProgram>,
    #[account(constraint = program_data.upgrade_authority_address == Some(authority.key()))]
    pub program_data: Account<'info, ProgramData>,
    pub authority: Signer<'info>,
}
```

Знову ж таки, в наведеному прикладі не показано інструкцію, яка ініціалізує config-акаунт, але вона повинна мати ті ж обмеження, щоб зловмисник не зміг ініціалізувати акаунт з непередбачуваними значеннями.

Якщо ви вперше чуєте про акаунт даних програми, варто ознайомитися з [цією статтею на Notion](https://www.notion.so/29780c48794c47308d5f138074dd9838) про запуск програм.

### Обмежити оновлення config-акаунту тільки для зазначеного адміністратора.

Обидва попередні варіанти досить безпечні, але також не гнучкі. Що робити, якщо ви хочете змінити адміністратора на іншого? Для цього можна зберігати адміністратора в config-акаунті.

```rust
pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[account]
pub struct ProgramConfig {
    admin: Pubkey,
    reward_token: Pubkey,
    rewards_per_day: u64,
}
```

Тоді ви можете обмежити свої інструкції "update" перевіркою підписанта, який має співпадати з полем `admin` у config-акаунті.

```rust
...

pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = SEED_PROGRAM_CONFIG, bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(constraint = authority.key() == program_config.admin)]
    pub authority: Signer<'info>,
}
```

Є одна проблема: між запуском програми та ініціалізацією config-акаунту _немає адміністратора_. Це означає, що інструкція для ініціалізації config-акаунту не може бути обмежена тільки для виклику адміністраторами. Отже, зловмисник може викликати її, щоб встановити себе адміністратором.

Хоча це звучить погано, насправді це просто означає, що не слід вважати вашу програму "ініціалізованою", поки ви самі не ініціалізуєте config-акаунт і не перевірите, що адміністратор, вказаний на акаунті, є тим, кого ви очікуєте. Якщо ваш скрипт запуску виконує запуск і одразу ж викликає `initialize`, навряд чи зловмисник навіть знає про існування вашої програми, не кажучи вже про спроби стати адміністратором. Якщо, через якесь неймовірне нещастя, хтось "перехопить" вашу програму, ви можете закрити програму за допомогою уповноваженого акаунта і повторно її запускати.

# Лабораторна робота

Тепер давайте спробуємо це разом. Для цієї лабораторної роботи ми будемо працювати з простою програмою для оплат через USDC. Програма стягує невелику комісію для здійснення переказу. Зверніть увагу, що це дещо штучний приклад, оскільки ви можете робити прямі перекази без посередницького контракту, але він імітує роботу деяких складних DeFi програм.

Під час тестування нашої програми ми швидко зрозуміємо, що вона могла б виграти від гнучкості, яку надає config-акаунт, контрольований адміністратором, і деякі маркери функцій.

### 1. Початковий етап

Завантажте початковий код з гілки `starter` цього [репозиторію](https://github.com/Unboxed-Software/solana-admin-instructions/tree/starter). Код містить програму з однією інструкцією та одним тестом в директорії `tests`.

Швидко пройдемося по тому, як працює програма.

Файл `lib.rs` містить константу для адреси USDC і одну інструкцію `payment`. Інструкція `payment` просто викликає функцію `payment_handler` у файлі `instructions/payment.rs`, де міститься логіка інструкції.

Файл `instructions/payment.rs` містить як функцію `payment_handler`, так і структуру підтвердження акаунтів `Payment`, яка представляє акаунти, необхідні для виконання інструкції `payment`. Функція `payment_handler` обчислює комісію 1% від суми платежу, переказує комісію на вказаний токен-акаунт і передає залишок суми отримувачу платежу.

Нарешті, в директорії `tests` є один тестовий файл `config.ts`, який просто викликає інструкцію `payment` і перевіряє, чи були правильно дебетовані та зараховані баланси відповідних токен-акаунтів.

Перед тим, як продовжити, витратьте кілька хвилин, щоб ознайомитися з цими файлами та їх вмістом.

### 2. Запустіть існуючий тест

Давайте почнемо з запуску існуючого тесту.

Переконайтеся, що ви використовуєте `yarn` або `npm install` для встановлення залежностей з файлу `package.json`. Потім, обов'язково запустіть команду `anchor keys list`, щоб вивести публічний ключ вашої програми в консоль. Це залежить від вашої локальної пари ключів, тому вам потрібно оновити файли `lib.rs` і `Anchor.toml`, щоб використовувати *ваш* ключ.

Нарешті, запустіть команду `anchor test`, щоб почати тест. Тест має завершитися з помилкою і вивести наступний результат:

```
Error: failed to send transaction: Transaction simulation failed: Error processing Instruction 0: incorrect program id for instruction
```

Ця помилка виникає тому, що ми намагаємося використовувати мінт-адресу USDC в Mainnet (жорстко вказану в файлі `lib.rs` програми), але цей токен не існує в локальному середовищі.

### 3. Додавання `local-testing`

Нам потрібен мінт-акаунт, який ми можемо використовувати в локальному середовищі *і* жорстко закодувати в програмі, щоб виправити цю проблему. Оскільки локальне середовище часто скидається під час тестування, вам потрібно зберігати пару ключів, яку ви можете використовувати для відновлення тієї ж адреси мінту щоразу.

Крім того, вам не варто змінювати жорстко закодовану адресу між Mainnet та локальним середовищами, оскільки це може призвести до людської помилки (та й це просто незручно). Тому ми створимо маркер `local-testing`, який при активації змусить програму використовувати наш локальний мінт-акаунт, а в іншому випадку використовувати мінт-акаунт USDC для продакшн.

Згенеруйте нову пару ключів, використовуючи команду `solana-keygen grind`. Виконайте наступну команду, щоб створити пару ключів з публічним ключем, що починається з "env".

```
solana-keygen grind --starts-with env:1
```

Як тільки пара ключів буде знайдена, ви побачите результат, схожий на наступний:

```
Wrote keypair to env9Y3szLdqMLU9rXpEGPqkjdvVn8YNHtxYNvCKXmHe.json
```

Пара ключів записується у файл у вашій робочій директорії. Тепер, коли у нас є мінт-адреса для USDC, давайте змінимо файл `lib.rs`. Використовуйте атрибут `cfg`, щоб визначити константу `USDC_MINT_PUBKEY` залежно від того, чи увімкнена, чи вимкнена функція `local-testing`. Пам'ятайте, що для `local-testing` необхідно встановити константу `USDC_MINT_PUBKEY` з тією, яку ви створили на попередньому кроці, а не копіювати ту, що наведена нижче.

```rust
use anchor_lang::prelude::*;
use solana_program::{pubkey, pubkey::Pubkey};
mod instructions;
use instructions::*;

declare_id!("BC3RMBvVa88zSDzPXnBXxpnNYCrKsxnhR3HwwHhuKKei");

#[cfg(feature = "local-testing")]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("...");

#[cfg(not(feature = "local-testing"))]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v");

#[program]
pub mod config {
    use super::*;

    pub fn payment(ctx: Context<Payment>, amount: u64) -> Result<()> {
        instructions::payment_handler(ctx, amount)
    }
}
```

Далі додайте функцію `local-testing` до файлу `Cargo.toml`, що знаходиться в `/programs`.

```
[features]
...
local-testing = []
```

Далі оновіть тестовий файл `config.ts`, щоб створити мінт за допомогою згенерованої пари ключів. Почніть із видалення константи `mint`.

```typescript
const mint = new anchor.web3.PublicKey(
    "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
);
```

Далі оновіть тест, щоб створити мінт за допомогою пари ключів — це дозволить нам повторно використовувати одну й ту ж адресу мінт-акаунту щоразу при запуску тестів. Не забудьте замінити назву файлу на ту, що була згенерована на попередньому кроці.

```typescript
let mint: anchor.web3.PublicKey

before(async () => {
  let data = fs.readFileSync(
    "env9Y3szLdqMLU9rXpEGPqkjdvVn8YNHtxYNvCKXmHe.json"
  )

  let keypair = anchor.web3.Keypair.fromSecretKey(
    new Uint8Array(JSON.parse(data))
  )

  const mint = await spl.createMint(
    connection,
    wallet.payer,
    wallet.publicKey,
    null,
    0,
    keypair
  )
...
```

Нарешті, запустіть тест із увімкненою функцією `local-testing`.

```
anchor test -- --features "local-testing"
```

Ви повинні побачити такий вивід:

```
config
  ✔ Payment completes successfully (406ms)


1 passing (3s)
```

Бум. Ось так ви використали функції для запуску двох різних шляхів виконання коду для різних середовищ.

### 4. Налаштування програми

Функції чудово підходять для встановлення різних значень під час компіляції, але що робити, якщо ви хочете мати можливість динамічно оновлювати відсоток комісії, що використовується програмою? Давайте зробимо це можливим, створивши акаунт налаштувань програми (Program Config), який дозволить нам оновлювати комісію без необхідності оновлювати саму програму.

Щоб почати, спочатку оновимо файл `lib.rs`, виконавши такі дії:

1. Додамо константу `SEED_PROGRAM_CONFIG`, яка буде використовуватися для створення PDA (Програмно визначеної адреси) для config-акаунту програми.
2. Додамо константу `ADMIN`, яка буде використовуватися як обмеження під час ініціалізації config-акаунту програми. Використайте команду `solana address`, щоб отримати вашу адресу для використання як значення цієї константи.
3. Додамо модуль `state`, який ми реалізуємо найближчим часом.
4. Додамо інструкції `initialize_program_config` і `update_program_config` разом із викликами до їхніх "обробників" (handlers), які ми реалізуємо на наступних етапах.

```rust
use anchor_lang::prelude::*;
use solana_program::{pubkey, pubkey::Pubkey};
mod instructions;
mod state;
use instructions::*;

declare_id!("BC3RMBvVa88zSDzPXnBXxpnNYCrKsxnhR3HwwHhuKKei");

#[cfg(feature = "local-testing")]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("envgiPXWwmpkHFKdy4QLv2cypgAWmVTVEm71YbNpYRu");

#[cfg(not(feature = "local-testing"))]
#[constant]
pub const USDC_MINT_PUBKEY: Pubkey = pubkey!("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v");

pub const SEED_PROGRAM_CONFIG: &[u8] = b"program_config";

#[constant]
pub const ADMIN: Pubkey = pubkey!("...");

#[program]
pub mod config {
    use super::*;

    pub fn initialize_program_config(ctx: Context<InitializeProgramConfig>) -> Result<()> {
        instructions::initialize_program_config_handler(ctx)
    }

    pub fn update_program_config(
        ctx: Context<UpdateProgramConfig>,
        new_fee: u64,
    ) -> Result<()> {
        instructions::update_program_config_handler(ctx, new_fee)
    }

    pub fn payment(ctx: Context<Payment>, amount: u64) -> Result<()> {
        instructions::payment_handler(ctx, amount)
    }
}
```

### 5. Стан Program Config

Тепер визначимо структуру для стану `ProgramConfig`. Цей акаунт буде зберігати адміністратора, токен-акаунт, куди відправляються комісії, і ставку комісії. Ми також визначимо кількість байтів, необхідних для зберігання цієї структури.

Створіть новий файл під назвою `state.rs` у директорії `/src` і додайте такий код:

```rust
use anchor_lang::prelude::*;

#[account]
pub struct ProgramConfig {
    pub admin: Pubkey,
    pub fee_destination: Pubkey,
    pub fee_basis_points: u64,
}

impl ProgramConfig {
    pub const LEN: usize = 8 + 32 + 32 + 8;
}
```

### 6. Додати інструкцію ініціалізації config-акаунта програми

Тепер створимо логіку інструкції для ініціалізації config-акаунту програми. Вона повинна викликатися лише транзакцією, підписаною ключем `ADMIN`, і має встановлювати всі властивості акаунту `ProgramConfig`.

Створіть папку під назвою `program_config` за шляхом `/src/instructions/program_config`. У цій папці будуть зберігатися всі інструкції, пов’язані з config-акаунтом програми.

У папці `program_config` створіть файл під назвою `initialize_program_config.rs` і додайте такий код:

```rust
use crate::state::ProgramConfig;
use crate::ADMIN;
use crate::SEED_PROGRAM_CONFIG;
use crate::USDC_MINT_PUBKEY;
use anchor_lang::prelude::*;
use anchor_spl::token::TokenAccount;

#[derive(Accounts)]
pub struct InitializeProgramConfig<'info> {
    #[account(init, seeds = [SEED_PROGRAM_CONFIG], bump, payer = authority, space = ProgramConfig::LEN)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account( token::mint = USDC_MINT_PUBKEY)]
    pub fee_destination: Account<'info, TokenAccount>,
    #[account(mut, address = ADMIN)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

pub fn initialize_program_config_handler(ctx: Context<InitializeProgramConfig>) -> Result<()> {
    ctx.accounts.program_config.admin = ctx.accounts.authority.key();
    ctx.accounts.program_config.fee_destination = ctx.accounts.fee_destination.key();
    ctx.accounts.program_config.fee_basis_points = 100;
    Ok(())
}
```

### 7. Додайте інструкцію, що дозволить змінювати комісію в Program Config 

Наступним кроком реалізуйте логіку інструкції для оновлення config-акаунту. Інструкція повинна вимагати, щоб підписант відповідав значенню `admin`, яке зберігається в config-акаунті програми.

У папці `program_config` створіть файл із назвою `update_program_config.rs` і додайте до нього наступний код.

```rust
use crate::state::ProgramConfig;
use crate::SEED_PROGRAM_CONFIG;
use crate::USDC_MINT_PUBKEY;
use anchor_lang::prelude::*;
use anchor_spl::token::TokenAccount;

#[derive(Accounts)]
pub struct UpdateProgramConfig<'info> {
    #[account(mut, seeds = [SEED_PROGRAM_CONFIG], bump)]
    pub program_config: Account<'info, ProgramConfig>,
    #[account( token::mint = USDC_MINT_PUBKEY)]
    pub fee_destination: Account<'info, TokenAccount>,
    #[account(
        mut,
        address = program_config.admin,
    )]
    pub admin: Signer<'info>,
    /// CHECK: arbitrarily assigned by existing admin
    pub new_admin: UncheckedAccount<'info>,
}

pub fn update_program_config_handler(
    ctx: Context<UpdateProgramConfig>,
    new_fee: u64,
) -> Result<()> {
    ctx.accounts.program_config.admin = ctx.accounts.new_admin.key();
    ctx.accounts.program_config.fee_destination = ctx.accounts.fee_destination.key();
    ctx.accounts.program_config.fee_basis_points = new_fee;
    Ok(())
}
```

### 8. Додайте mod.rs та оновіть instructions.rs

Далі відкриємо доступ до обробників інструкцій, які ми створили, щоб виклик із lib.rs не спричиняв помилок. Почніть із додавання файлу `mod.rs` у папку `program_config`. Додайте наведений нижче код, щоб зробити два модулі, `initialize_program_config` та `update_program_config`, доступними.

```rust
mod initialize_program_config;
pub use initialize_program_config::*;

mod update_program_config;
pub use update_program_config::*;
```

Тепер оновіть файл `instructions.rs`, розташований за шляхом `/src/instructions.rs`. Додайте наведений нижче код, щоб зробити модулі `program_config` і `payment` доступними.

```rust
mod program_config;
pub use program_config::*;

mod payment;
pub use payment::*;
```

### 9. Оновлення інструкції оплати

Останнім кроком давайте оновимо інструкцію оплати, щоб перевірити, чи збігається акаунт `fee_destination` в інструкції з акаунтом `fee_destination`, що зберігається в конфігураційному акаунті програми. Потім оновимо обчислення комісії в інструкції, щоб вона базувалась на значенні `fee_basis_point`, яке зберігається в config-акаунті програми.

```rust
use crate::state::ProgramConfig;
use crate::SEED_PROGRAM_CONFIG;
use crate::USDC_MINT_PUBKEY;
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount};

#[derive(Accounts)]
pub struct Payment<'info> {
    #[account(
        seeds = [SEED_PROGRAM_CONFIG],
        bump,
        has_one = fee_destination
    )]
    pub program_config: Account<'info, ProgramConfig>,
    #[account(
        mut,
        token::mint = USDC_MINT_PUBKEY
    )]
    pub fee_destination: Account<'info, TokenAccount>,
    #[account(
        mut,
        token::mint = USDC_MINT_PUBKEY
    )]
    pub sender_token_account: Account<'info, TokenAccount>,
    #[account(
        mut,
        token::mint = USDC_MINT_PUBKEY
    )]
    pub receiver_token_account: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    #[account(mut)]
    pub sender: Signer<'info>,
}

pub fn payment_handler(ctx: Context<Payment>, amount: u64) -> Result<()> {
    let fee_amount = amount
        .checked_mul(ctx.accounts.program_config.fee_basis_points)
        .unwrap()
        .checked_div(10000)
        .unwrap();
    let remaining_amount = amount.checked_sub(fee_amount).unwrap();

    msg!("Amount: {}", amount);
    msg!("Fee Amount: {}", fee_amount);
    msg!("Remaining Transfer Amount: {}", remaining_amount);

    token::transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.sender_token_account.to_account_info(),
                authority: ctx.accounts.sender.to_account_info(),
                to: ctx.accounts.fee_destination.to_account_info(),
            },
        ),
        fee_amount,
    )?;

    token::transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.sender_token_account.to_account_info(),
                authority: ctx.accounts.sender.to_account_info(),
                to: ctx.accounts.receiver_token_account.to_account_info(),
            },
        ),
        remaining_amount,
    )?;

    Ok(())
}
```

### 10. Тестування

Тепер, коли ми завершили впровадження нової структури конфігурації програми та інструкцій, перейдемо до тестування оновленої програми. Для початку додайте PDA для config-акаунту програми до тестового файлу.

```typescript
describe("config", () => {
  ...
  const programConfig = findProgramAddressSync(
    [Buffer.from("program_config")],
    program.programId
  )[0]
...
```

Далі оновіть тестовий файл, додавши три нові тести, які перевіряють:

1. Чи config-акаунт програми правильно ініціалізується.  
2. Чи інструкція для платежів працює належним чином.  
3. Чи може адміністратор успішно оновити config-акаунт.  
4. Чи неможливо оновити config-акаунт кимось, хто не є адміністратором.  

Перший тест ініціалізує config-акаунт програми та перевіряє, чи правильно встановлено комісію та чи коректно збережено адміністратора в config-акаунті програми.

```typescript
it("Initialize Program Config Account", async () => {
  const tx = await program.methods
    .initializeProgramConfig()
    .accounts({
      programConfig: programConfig,
      feeDestination: feeDestination,
      authority: wallet.publicKey,
      systemProgram: anchor.web3.SystemProgram.programId,
    })
    .rpc()

  assert.strictEqual(
    (
      await program.account.programConfig.fetch(programConfig)
    ).feeBasisPoints.toNumber(),
    100
  )
  assert.strictEqual(
    (
      await program.account.programConfig.fetch(programConfig)
    ).admin.toString(),
    wallet.publicKey.toString()
  )
})
```

Другий тест перевіряє, чи інструкція для платежів працює правильно: комісія відправляється на акаунт призначення комісії, а залишок переказується отримувачу. Тут ми оновлюємо наявний тест, щоб включити акаунт `programConfig`.

```typescript
it("Payment completes successfully", async () => {
  const tx = await program.methods
    .payment(new anchor.BN(10000))
    .accounts({
      programConfig: programConfig,
      feeDestination: feeDestination,
      senderTokenAccount: senderTokenAccount,
      receiverTokenAccount: receiverTokenAccount,
      sender: sender.publicKey,
    })
    .transaction()

  await anchor.web3.sendAndConfirmTransaction(connection, tx, [sender])

  assert.strictEqual(
    (await connection.getTokenAccountBalance(senderTokenAccount)).value
      .uiAmount,
    0
  )

  assert.strictEqual(
    (await connection.getTokenAccountBalance(feeDestination)).value.uiAmount,
    100
  )

  assert.strictEqual(
    (await connection.getTokenAccountBalance(receiverTokenAccount)).value
      .uiAmount,
    9900
  )
})
```

Третій тест намагається оновити комісію в config-акаунті програми, і це має пройти успішно.

```typescript
it("Update Program Config Account", async () => {
  const tx = await program.methods
    .updateProgramConfig(new anchor.BN(200))
    .accounts({
      programConfig: programConfig,
      admin: wallet.publicKey,
      feeDestination: feeDestination,
      newAdmin: sender.publicKey,
    })
    .rpc()

  assert.strictEqual(
    (
      await program.account.programConfig.fetch(programConfig)
    ).feeBasisPoints.toNumber(),
    200
  )
})
```

Четвертий тест намагається оновити комісію в config-акаунті програми, але адміністратор не є тим, хто збережений у config-акаунті програми, тому це має завершитися невдачею.

```typescript
it("Update Program Config Account with unauthorized admin (expect fail)", async () => {
  try {
    const tx = await program.methods
      .updateProgramConfig(new anchor.BN(300))
      .accounts({
        programConfig: programConfig,
        admin: sender.publicKey,
        feeDestination: feeDestination,
        newAdmin: sender.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [sender])
  } catch (err) {
    expect(err)
  }
})
```

Нарешті, запустіть тест за допомогою наступної команди:

```
anchor test -- --features "local-testing"
```

Ви повинні побачити наступний результат:

```
config
  ✔ Initialize Program Config Account (199ms)
  ✔ Payment completes successfully (405ms)
  ✔ Update Program Config Account (403ms)
  ✔ Update Program Config Account with unauthorized admin (expect fail)

4 passing (8s)
```

І це все! Тепер програмою набагато простіше користуватися надалі. Якщо ви хочете переглянути остаточний код рішення, ви можете знайти його в гілці `solution` [того ж репозиторію](https://github.com/Unboxed-Software/solana-admin-instructions/tree/solution).

# Завдання

Тепер настав час вам зробити це самостійно. Ми згадували, що можна використовувати уповноважений акаунт програми як початкового адміністратора. Спробуйте оновити `initialize_program_config` у лабораторній роботі, щоб лише уповноважений акаунт міг викликати її, замість використання жорстко закодованого `ADMIN`.

Зверніть увагу, що коли ви запускаєте команду `anchor test` на локальній мережі, вона ініціалізує новий тестовий валідатор за допомогою `solana-test-validator`. Цей тестовий валідатор використовує неоновлюваний завантажувач. Нееоновлюваний завантажувач означає, що акаунт `program_data` програми не ініціалізується при запуску валідатора. Ви пам'ятаєте з уроку, що цей акаунт дозволяє нам отримати доступ до уповноваженого акаунту програми.

Щоб обійти це, ви можете додати функцію `deploy` до тестового файлу, яка виконує команду розгортання програми за допомогою оновлюваного завантажувача. Для цього запустіть `anchor test --skip-deploy` і викликайте функцію `deploy` в тесті, щоб виконати команду розгортання після запуску тестового валідатора.

```typescript
import { execSync } from "child_process"

...

const deploy = () => {
  const deployCmd = `solana program deploy --url localhost -v --program-id $(pwd)/target/deploy/config-keypair.json $(pwd)/target/deploy/config.so`
  execSync(deployCmd)
}

...

before(async () => {
  ...
  deploy()
})
```

Наприклад, команда для запуску тесту з використанням фунцій виглядатиме ось так:

```
anchor test --skip-deploy -- --features "local-testing"
```

Спробуйте виконати це самостійно, але якщо виникнуть труднощі — не соромтеся звернутися до гілки `challenge` [цього ж репозиторію](https://github.com/Unboxed-Software/solana-admin-instructions/tree/challenge), щоб переглянути один із можливих варіантів розв’язання.


## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=02a7dab7-d9c1-495b-928c-a4412006ec20)!
