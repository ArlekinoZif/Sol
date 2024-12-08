---
заголовок: Виклики між програмами (Cross Program Invocations)
мета:
- Пояснити, що таке виклики між програмами (CPI).  
- Описати, як створювати та використовувати CPI.  
- Пояснити, як програма забезпечує підпис для PDA (програмно визначеного облікового запису).  
- Уникати поширених помилок і вирішувати типові проблеми, пов’язані з CPI.  
---

# Стислий виклад  

- **Виклик між програмами (CPI)** — це звернення однієї програми до іншої з викликом конкретної інструкції у програмі, до якої звертаються.  
- CPI здійснюються за допомогою команд `invoke` або `invoke_signed`. Остання використовується, щоб програми могли надавати підписи для PDA, якими вони володіють.  
- CPI забезпечують повну взаємодію програм у екосистемі Solana, оскільки всі публічні інструкції однієї програми можуть викликатися іншою програмою через CPI.  
- Оскільки ми не контролюємо облікові записи та дані, що передаються до програми, важливо перевіряти всі параметри, що передаються через CPI, щоб забезпечити безпеку програми.  

# Урок

## Що таке CPI?

Виклик між програмами (CPI) — це прямий виклик однієї програми до іншої. Так само, як будь-який клієнт може звертатися до будь-якої програми через JSON RPC, будь-яка програма може напряму викликати іншу програму. Єдина вимога для виклику інструкції в іншій програмі — це правильне створення цієї інструкції. Ви можете здійснювати CPI до вбудованих програм, інших програм, які ви створили, або сторонніх програм. CPI фактично перетворює всю екосистему Solana на гігантський API, який розробник може використовувати на свій розсуд.  


CPIs за своєю структурою схожі на інструкції, які ви створюєте на стороні клієнта. Проте існують певні особливості та відмінності залежно від того, чи ви використовуєте `invoke`, чи `invoke_signed`. Ми розглянемо обидва методи пізніше в цьому уроці.  

## Як викликати CPI

CPI можна викликати за допомогою функцій [`invoke`](https://docs.rs/solana-program/1.10.19/solana_program/program/fn.invoke.html) або [`invoke_signed`](https://docs.rs/solana-program/1.10.19/solana_program/program/fn.invoke_signed.html) із бібліотеки `solana_program`.  

- Функція `invoke` дозволяє передати оригінальний підпис транзакції, який був наданий вашій програмі.  
- Функція `invoke_signed` використовується для того, щоб ваша програма могла "підписуватися" від імені своїх PDA (програмно визначених акаунтів).  

```rust
// Used when there are not signatures for PDAs needed
pub fn invoke(
    instruction: &Instruction,
    account_infos: &[AccountInfo<'_>]
) -> ProgramResult

// Used when a program must provide a 'signature' for a PDA, hence the signer_seeds parameter
pub fn invoke_signed(
    instruction: &Instruction,
    account_infos: &[AccountInfo<'_>],
    signers_seeds: &[&[&[u8]]]
) -> ProgramResult
```

CPI передає права доступу від програми-ініціатора до викликаної програми. Якщо обліковий запис був позначений як підписант (signer) або доступний для запису (writable) в початковій програмі, ці права залишаються і у викликаній програмі.

Важливо розуміти, що саме ви, як розробник, вирішуєте, які облікові записи передавати в CPI. CPI можна уявити як створення нової інструкції з нуля, використовуючи лише ті дані, які були передані у вашу програму.  

### CPI з `invoke`

```rust
invoke(
    &Instruction {
        program_id: calling_program_id,
        accounts: accounts_meta,
        data,
    },
    &account_infos[account1.clone(), account2.clone(), account3.clone()],
)?;
```

- `program_id` - the public key of the program you are going to invoke
- `account` - a list of account metadata as a vector. You need to include every account that the invoked program will read or write
- `data` - a byte buffer representing the data being passed to the callee program as a vector

The `Instruction` type has the following definition:

```rust
pub struct Instruction {
    pub program_id: Pubkey,
    pub accounts: Vec<AccountMeta>,
    pub data: Vec<u8>,
}
```


Залежно від програми, до якої ви звертаєтеся, може існувати бібліотека (crate) із допоміжними функціями для створення об'єкта `Instruction`. Багато розробників і організацій створюють загальнодоступні бібліотеки разом зі своїми програмами, які спрощують виклики до цих програм. Це схоже на бібліотеки TypeScript, які ми використовували у цьому курсі (наприклад, [@solana/web3.js](https://solana-labs.github.io/solana-web3.js/), [@solana/spl-token](https://solana-labs.github.io/solana-program-library/token/js/)). Наприклад, у практичній частині цього уроку ми використовуватимемо бібліотеку `spl_token` для створення інструкцій випуску токенів.  

В інших випадках вам доведеться створювати екземпляр `Instruction` вручну.  

Хоча поле `program_id` досить просте, поля `accounts` і `data` потребують пояснення.  

Поля `accounts` і `data` мають тип `Vec` (вектор). Для створення вектора ви можете використовувати макрос [`vec`](https://doc.rust-lang.org/std/macro.vec.html) із нотацією масиву, наприклад:  

```rust
let v = vec![1, 2, 3];
assert_eq!(v[0], 1);
assert_eq!(v[1], 2);
assert_eq!(v[2], 3);
```


Поле `accounts` у структурі `Instruction` очікує вектор типу [`AccountMeta`](https://docs.rs/solana-program/latest/solana_program/instruction/struct.AccountMeta.html). Структура `AccountMeta` має таке визначення:  


```rust
pub struct AccountMeta {
    pub pubkey: Pubkey,
    pub is_signer: bool,
    pub is_writable: bool,
}
```

Поєднуючи ці дві частини, це виглядатиме ось так:  

```rust
use solana_program::instruction::AccountMeta;

vec![
    AccountMeta::new(account1_pubkey, true), // metadata for a writable, signer account
    AccountMeta::read_only(account2_pubkey, false), // metadata for a read-only, non-signer account
    AccountMeta::read_only(account3_pubkey, true), // metadata for a read-only, signer account
    AccountMeta::new(account4_pubkey, false), // metadata for a writable, non-signer account
]
```


Останнє поле об'єкта інструкції — це дані, які представлені байт буфером. Ви можете створити байт буфер у Rust, знову використовуючи макрос `vec`, який має реалізовану функцію для створення вектора певної довжини. Після того, як ви ініціалізуєте порожній вектор, ви будете створювати байт буфер так само, як і на стороні клієнта. Необхідно визначити, які дані потрібні викликаній програмі, та формат серіалізації, що використовується, і написати код відповідно до цього. Не соромтеся ознайомитись з деякими [можливостями макросу `vec`](https://doc.rust-lang.org/alloc/vec/struct.Vec.html#), які доступні вам.


```rust
let mut vec = Vec::with_capacity(3);
vec.push(1);
vec.push(2);
vec.extend_from_slice(&number_variable.to_le_bytes());
```

Метод [`extend_from_slice`](https://doc.rust-lang.org/alloc/vec/struct.Vec.html#method.extend_from_slice) ймовірно новий для вас. Це метод для векторів, який приймає зріз (slice) як вхід, ітерує через цей зріз, клонує кожен елемент і потім додає його до `Vec`.

### Передача списку облікових записів

Окрім інструкції,  `invoke` та `invoke_signed` також потребують списку об'єктів `account_info`. Так само, як і список об'єктів `AccountMeta`, який ви додали до інструкції, потрібно включити всі облікові записи, які програма, до якої ви звертаєтесь, буде читати чи змінювати.

До моменту виконання CPI у вашій програмі ви вже повинні були отримати всі об'єкти `account_info`, які були передані у вашу програму, та зберегти їх у змінних. Ви будете створювати список об'єктів `account_info` для CPI, вибираючи, які з цих облікових записів копіювати та передавати далі.

Ви можете скопіювати кожен об'єкт `account_info`, який потрібно передати в CPI, використовуючи трейт [`Clone`](https://docs.rs/solana-program/1.10.19/solana_program/account_info/struct.AccountInfo.html#impl-Clone), який реалізовано для структури `account_info` у бібліотеці `solana_program`. Цей трейт `Clone` повертає копію екземпляра [`account_info`](https://docs.rs/solana-program/1.10.19/solana_program/account_info/struct.AccountInfo.html).

```rust
&[first_account.clone(), second_account.clone(), third_account.clone()]
```

### CPI з `invoke`

Створивши інструкцію та список облікових записів, ви можете виконати виклик до `invoke`.

```rust
invoke(
    &Instruction {
        program_id: calling_program_id,
        accounts: accounts_meta,
        data,
    },
    &[account1.clone(), account2.clone(), account3.clone()],
)?;
```

Немає потреби включати підпис, оскільки runtime Solana передає оригінальний підпис, який був переданий у вашу програму. Пам'ятайте, що `invoke` не працюватиме, якщо потрібно підписати від імені PDA. Для цього вам слід використовувати `invoke_signed`.

### CPI з `invoke_signed`

Використання `invoke_signed` трохи відрізняється, оскільки є додаткове поле, яке вимагає наявність seed, що використовуються для створення будь-яких PDA, які повинні підписати транзакцію. Ви можете пам'ятати з попередніх уроків, що PDA не знаходяться на кривій Ed25519 і, отже, не мають відповідного секретного ключа. Вам уже казали, що програми можуть надавати підписи для своїх PDA, але ви ще не дізналися, як це насправді відбувається — до цього моменту. Програми надають підписи для своїх PDA за допомогою функції `invoke_signed`. Перші два поля в `invoke_signed` такі ж, як і в `invoke`, але є додаткове поле `signers_seeds`, яке тут відіграє важливу роль.

```rust
invoke_signed(
    &instruction,
    accounts,
    &[&["First addresses seed"],
        &["Second addresses first seed",
        "Second addresses second seed"]],
)?;
```

Хоча PDA не мають власних секретних ключів, вони можуть використовуватися програмою для видачі інструкції, що включає PDA як підписанта (signer). Єдиний спосіб для runtime перевірити, що PDA належить викликаючій програмі, — це коли викликаюча програма надає сід, що використовуються для створення адреси, у полі `signers_seeds`.

Solana runtime внутрішньо викликає метод [`create_program_address`](https://docs.rs/solana-program/1.4.4/solana_program/pubkey/struct.Pubkey.html#method.create_program_address), використовуючи надані seed та `program_id` викликаючої програми. Потім він порівнює результат з адресами, які були надані в інструкції. Якщо хоча б одна з адрес співпадає, то runtime розуміє, що програма, пов'язана з цією адресою, є викликаючою і таким чином має право бути підписантом.


## Найкращі практики та поширені помилки

### Перевірка безпеки

Є кілька поширених помилок і важливих аспектів, які мають значення для безпеки та надійності вашої програми і їх слід пам'ятати при використанні CPI.

Як вже відомо, ми не маємо контролю над тим, яка інформація передається в наші програми. З цієї причини важливо завжди перевіряти `program_id`, акаунти та дані, що передаються в CPI. Без цих перевірок безпеки хтось може надіслати транзакцію, яка викликає інструкцію в абсолютно іншій програмі, ніж очікувалося.

На щастя, в функції `invoke_signed` є вбудовані перевірки будь-яких PDA, що позначені як підписанти (signer). Всі інші акаунти та `instruction_data` повинні бути перевірені в коді вашої програми перед виконанням CPI. Також важливо переконатися, що ви націлені на правильну інструкцію в програмі, яку ви викликаєте. Найпростіший спосіб зробити це — ознайомитися з вихідним кодом програми, яку ви будете викликати, так само, як ви б це робили, складаючи інструкцію з боку клієнта.

### Поширені помилки

Існують кілька поширених помилок, які ви можете побачити під час виконання CPI. Зазвичай це означає, що ви створюєте CPI з неправильною інформацією. Наприклад, вам може трапитися повідомлення про помилку, подібне до цього:

```text
EF1M4SPfKcchb6scq297y8FPCaLvj5kGjwMzjTM68wjA's signer privilege escalated
Program returned error: "Cross-program invocation with unauthorized signer or writable account"
```

Це повідомлення трохи оманливе, оскільки "signer privilege escalated" не виглядає як проблема, але насправді це означає, що ви неправильно підписуєте адресу, зазначену в повідомленні. Якщо ви використовуєте `invoke_signed` і отримуєте цю помилку, це, ймовірно, означає, що наданий вами seed неправильні. Ви також можете знайти [приклад транзакції, яка завершилася з цією помилкою](https://explorer.solana.com/tx/3mxbShkerH9ZV1rMmvDfaAhLhJJqrmMjcsWzanjkARjBQurhf4dounrDCUkGunH1p9M4jEwef9parueyHVw6r2Et?cluster=devnet).

Інша подібна помилка виникає, коли акаунт, в який записується дані, не позначено як `writable` у структурі `AccountMeta`.

```text
2qoeXa9fo8xVHzd2h9mVcueh6oK3zmAiJxCTySM5rbLZ's writable privilege escalated
Program returned error: "Cross-program invocation with unauthorized signer or writable account"
```

Пам'ятайте, що будь-який акаунт повинен бути вказаний як доступний для запису (writable), якщо його дані можуть бути змінені програмою під час виконання. Спроба запису в акаунт, який не позначено як writable, призведе до помилки транзакції. Запис у акаунт, який не належить програмі, також призведе до помилки транзакції. Будь-який акаунт, баланс лампортів якого може бути змінений програмою під час виконання, повинен бути позначений як writable. Спроба змінити лампорти в акаунті, який не позначений як writable, викличе помилку транзакції. Хоча віднімання лампортів від акаунту, який не належить програмі, призведе до помилки транзакції, додавання лампортів до будь-якого акаунту дозволено, якщо він є змінним (mutable).

Щоб побачити в дії, перегляньте [цю транзакцію в експлорері](https://explorer.solana.com/tx/ExB9YQJiSzTZDBqx4itPaa4TpT8VK4Adk7GU5pSoGEzNz9fa7PPZsUxssHGrBbJRnCvhoKgLCWnAycFB7VYDbBg?cluster=devnet).

## Чому CPI важливі?

CPI є дуже важливою функцією в екосистемі Solana, і вона дозволяє всім програмам цієї екосистеми взаємодіяти між собою. Коли мова йде про розробку, CPI дозволяє не винаходити заново колесо. Дає можливість будувати нові протоколи та додатки поверх вже створених, як будівельні блоки або деталі Lego. Важливо пам'ятати, що CPI — це вулиця з двостороннім рухом, і це стосується будь-яких програм, які ви розгортаєте! Якщо ви створите щось класне і корисне, розробники матимуть можливість будувати поверх вашого досягнення або підключити ваш протокол до свого. Можливість компонувати — це важлива частина того, що робить криптовалюту такою унікальною, і саме CPI роблять це можливим на Solana.


Ще один важливий аспект CPI — це можливість для програм підписувати транзакції для своїх PDA. Як ви, мабуть, помітили, PDA часто використовуються в розробці на Solana, тому що вони дозволяють програмам контролювати конкретні адреси таким чином, що жоден зовнішній користувач не може створити транзакцію з дійсними підписами для цих адрес. Це може бути *дуже* корисно для багатьох додатків у Web3 (наприклад, DeFi, NFT і т. д.). Без CPI PDA не були б настільки корисними, оскільки не було б способу для програми підписати транзакції, що включають ці адреси — це фактично перетворює їх на чорні діри (якщо щось відправити на PDA, не буде жодного способу витягнути це без CPI!).

# Лабораторна робота

Тепер давайте перейдемо до практики використанням CPI, зробивши деякі доповнення до програми Movie Review. Якщо ви приєдналися до цього уроку без проходження попередніх, нагадаю, що програма Movie Review дозволяє користувачам подавати рецензії на фільми та зберігати їх у акаунтах PDA.

У минулому уроці ми додали можливість залишати коментарі до рецензій інших користувачів, використовуючи PDA. У цьому уроці ми зосередимося на тому, щоб програма мінтила токен кожного разу, коли створюється рецензія або коментар.

Щоб це реалізувати, нам потрібно буде викликати інструкцію `MintTo` з програми SPL Token через CPI. Якщо вам потрібен додатковий матеріал про токени, токен мінти та їх випуск, перегляньте урок про [Token Program](./token-program), перш ніж продовжити роботу над лабораторною роботою.

### 1. Візьміть початковий код та додайте залежності

Щоб розпочати, ми будемо використовувати фінальний стан програми Movie Review з попереднього уроку про PDA. Тож, якщо ви щойно завершили цей урок, ви вже готові до роботи. Якщо ви лише приєднуєтеся зараз, не хвилюйтеся, ви можете [завантажити стартовий код тут](https://github.com/Unboxed-Software/solana-movie-program/tree/solution-add-comments). Ми будемо використовувати гілку `solution-add-comments` як нашу стартову точку.

### 2. Додайте залежності до `Cargo.toml`

Перш ніж ми почнемо, потрібно додати дві нові залежності до файлу `Cargo.toml` в гілці `[dependencies]`. Ми будемо використовувати бібліотеки (crate) `spl-token` та `spl-associated-token-account` на додаток до існуючих залежностей.

```text
spl-token = { version="~3.2.0", features = [ "no-entrypoint" ] }
spl-associated-token-account = { version="=1.0.5", features = [ "no-entrypoint" ] }
```

Після додавання вищезазначеного, запустіть команду `cargo check` у вашій консолі, щоб Cargo вирішив залежності та переконався, що ви готові продовжити. Залежно від вашого середовища, можливо, вам доведеться змінити версії бібліотек перед тим, як рухатися далі.

### 3. Додайте необхідні акаунти в `add_movie_review`

Оскільки ми хочемо, щоб під час створення відгуку створювався токен, потрібно додати відповідну логіку до функції `add_movie_review`. Оскільки ми будемо здійснювати мінт токенів, інструкція `add_movie_review` вимагає передачі декількох нових акаунтів:

- `token_mint` — токен-мінт адреса  
- `mint_auth` — адреса уповноважена (authority) створювати токени
- `user_ata` — асоційований токен-акаунт користувача для цього мінта (куди будуть мінтитися токени)  
- `token_program` — адреса токен-програми  

Ми почнемо з додавання цих нових акаунтів у ту частину функції, яка перебирає передані акаунти:

```rust
// Inside add_movie_review
msg!("Adding movie review...");
msg!("Title: {}", title);
msg!("Rating: {}", rating);
msg!("Description: {}", description);

let account_info_iter = &mut accounts.iter();

let initializer = next_account_info(account_info_iter)?;
let pda_account = next_account_info(account_info_iter)?;
let pda_counter = next_account_info(account_info_iter)?;
let token_mint = next_account_info(account_info_iter)?;
let mint_auth = next_account_info(account_info_iter)?;
let user_ata = next_account_info(account_info_iter)?;
let system_program = next_account_info(account_info_iter)?;
let token_program = next_account_info(account_info_iter)?;
```

Для нової функціональності не потрібні додаткові дані `instruction_data`, тому немає необхідності змінювати спосіб десеріалізації даних. Єдине, що потрібно — це додаткові акаунти.

### 4. Мінт токенів рецензента в `add_movie_review`

Перед тим як заглибитися в логіку створення токенів, давайте імпортуємо адресу програми Token та константу `LAMPORTS_PER_SOL` в верхній частині файлу.

```rust
// Inside processor.rs
use solana_program::native_token::LAMPORTS_PER_SOL;
use spl_associated_token_account::get_associated_token_address;
use spl_token::{instruction::initialize_mint, ID as TOKEN_PROGRAM_ID};
```

Тепер можемо перейти до логіки, яка оброблятиме сам токен мінт! Ми додамо це в кінець функції `add_movie_review`, перед тим, як буде повернено `Ok(())`.

Мінти токенів вимагає підпису від уповноваженої адреси. Оскільки програма повинна мати змогу мінтити токени, уповноважена адреса має бути акаунтом, для якого програма може підписувати транзакції. Іншими словами, це має бути акаунт PDA, що належить програмі.

Також ми будемо структурувати наш токен-мінт так, щоб акаунт мінта був акаунтом PDA, який ми можемо отримати детермінованим способом. Це дозволить нам завжди перевіряти, чи є акаунт `token_mint`, переданий в програму, очікуваним акаунтом.

Давайте визначимо адреси token mint і mint authority, використовуючи функцію `find_program_address` з seed “token_mint” і “token_auth” відповідно.

```rust
// Mint tokens here
msg!("deriving mint authority");
let (mint_pda, _mint_bump) = Pubkey::find_program_address(&[b"token_mint"], program_id);
let (mint_auth_pda, mint_auth_bump) =
    Pubkey::find_program_address(&[b"token_auth"], program_id);
```

Наступним кроком буде виконання перевірок безпеки для кожного з нових акаунтів, переданих програмі. Завжди пам'ятайте про важливість перевірки акаунтів!

```rust
if *token_mint.key != mint_pda {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *mint_auth.key != mint_auth_pda {
    msg!("Mint passed in and mint derived do not match");
    return Err(ReviewError::InvalidPDA.into());
}

if *user_ata.key != get_associated_token_address(initializer.key, token_mint.key) {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *token_program.key != TOKEN_PROGRAM_ID {
    msg!("Incorrect token program");
    return Err(ReviewError::IncorrectAccountError.into());
}
```

Останнім кроком буде виконання CPI до функції `mint_to` токен-програми з використанням правильних акаунтів через `invoke_signed`. Крейт `spl_token` надає допоміжну функцію `mint_to` для створення інструкції для мінта. Це зручно, оскільки нам не потрібно вручну створювати всю інструкцію з нуля. Натомість ми можемо просто передати необхідні аргументи функції. Ось підпис функції:

```rust
// Inside the token program, returns an Instruction object
pub fn mint_to(
    token_program_id: &Pubkey,
    mint_pubkey: &Pubkey,
    account_pubkey: &Pubkey,
    owner_pubkey: &Pubkey,
    signer_pubkeys: &[&Pubkey],
    amount: u64,
) -> Result<Instruction, ProgramError>
```

Потім ми передаємо копії акаунтів `token_mint`, `user_ata` та `mint_auth`. І, що найважливіше для цього уроку, ми надаємо sedd для пошуку адреси `token_mint`, включаючи bump seed.

```rust
msg!("Minting 10 tokens to User associated token account");
invoke_signed(
    // Instruction
    &spl_token::instruction::mint_to(
        token_program.key,
        token_mint.key,
        user_ata.key,
        mint_auth.key,
        &[],
        10*LAMPORTS_PER_SOL,
    )?,
    // Account_infos
    &[token_mint.clone(), user_ata.clone(), mint_auth.clone()],
    // Seeds
    &[&[b"token_auth", &[mint_auth_bump]]],
)?;

Ok(())
```

Зверніть увагу, що ми використовуємо `invoke_signed`, а не `invoke`. Програма Token вимагає, щоб акаунт `mint_auth` підписав цю транзакцію. Оскільки акаунт `mint_auth` є PDA, тільки програма, від якої він пішов, може підписувати від його імені. Коли викликається `invoke_signed`, середовище виконання Solana викликає `create_program_address` з наданими sedd і bump, а потім порівнює похідну адресу з усіма адресами наданих об'єктів `AccountInfo`. Якщо будь-яка з адрес збігається з похідною адресою, середовище виконання знає, що відповідний акаунт є PDA цієї програми і що програма підписує цю транзакцію для цього акаунта.

На цьому етапі інструкція `add_movie_review` повинна бути повністю функціональною і буде мінтити десять токенів рецензенту при публікації огляду.

### 5. Повторіть для `add_comment`

Наші оновлення до функції `add_comment` будуть майже ідентичними тим, що ми зробили для функції `add_movie_review` вище. Єдина різниця полягає в тому, що ми змінимо кількість токенів, які будуть мінтитися для коментаря, з десяти на п'ять, щоб додавання рецензій мало більшу вагу, ніж коментарі. Спочатку оновіть акаунти, додавши чотири додаткові акаунти, як це було в функції `add_movie_review`.

```rust
// Inside add_comment
let account_info_iter = &mut accounts.iter();

let commenter = next_account_info(account_info_iter)?;
let pda_review = next_account_info(account_info_iter)?;
let pda_counter = next_account_info(account_info_iter)?;
let pda_comment = next_account_info(account_info_iter)?;
let token_mint = next_account_info(account_info_iter)?;
let mint_auth = next_account_info(account_info_iter)?;
let user_ata = next_account_info(account_info_iter)?;
let system_program = next_account_info(account_info_iter)?;
let token_program = next_account_info(account_info_iter)?;
```

Наступним кроком, перейдіть до низу функції `add_comment`, перед `Ok(())`. Потім проведіть деривацію акаунтів для token mint та mint authority accounts. Пам'ятайте, що обидва акаунти є PDA, які отримують нове значення з seed "token_mint" та "token_authority" відповідно.

```rust
// Mint tokens here
msg!("deriving mint authority");
let (mint_pda, _mint_bump) = Pubkey::find_program_address(&[b"token_mint"], program_id);
let (mint_auth_pda, mint_auth_bump) =
    Pubkey::find_program_address(&[b"token_auth"], program_id);
```

Наступним кроком є перевірка, що кожен з нових акаунтів є правильним акаунтом.

```rust
if *token_mint.key != mint_pda {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *mint_auth.key != mint_auth_pda {
    msg!("Mint passed in and mint derived do not match");
    return Err(ReviewError::InvalidPDA.into());
}

if *user_ata.key != get_associated_token_address(commenter.key, token_mint.key) {
    msg!("Incorrect token mint");
    return Err(ReviewError::IncorrectAccountError.into());
}

if *token_program.key != TOKEN_PROGRAM_ID {
    msg!("Incorrect token program");
    return Err(ReviewError::IncorrectAccountError.into());
}
```

Нарешті, використовуйте `invoke_signed`, щоб надіслати інструкцію `mint_to` до Token програми, надіславши п'ять токенів коментатору.

```rust
msg!("Minting 5 tokens to User associated token account");
invoke_signed(
    // Instruction
    &spl_token::instruction::mint_to(
        token_program.key,
        token_mint.key,
        user_ata.key,
        mint_auth.key,
        &[],
        5 * LAMPORTS_PER_SOL,
    )?,
    // Account_infos
    &[token_mint.clone(), user_ata.clone(), mint_auth.clone()],
    // Seeds
    &[&[b"token_auth", &[mint_auth_bump]]],
)?;

Ok(())
```

### 6. Налаштування Token Mint

Ми написали весь код, необхідний для мінту токенів рецензентам і коментаторам, але весь цей код припускає, що існує токен мінт, який був створений за допомогою seed (початкового значення) 'token_mint'. Для того, щоб це працювало, ми налаштуємо додаткову інструкцію для ініціалізації токен мінту. Вона буде написана таким чином, щоб її можна було викликати лише один раз, і не має великого значення, хто її викликає.

Зважаючи на те, що протягом цього уроку ми вже кілька разів детально розглядали всі концепції, пов'язані з PDA та CPI, ми пройдемо цю частину з меншою кількістю пояснень, ніж у попередніх кроках. Почнімо з того, щоб додати четвертий варіант інструкції до enum `MovieInstruction` у файлі `instruction.rs`.

```rust
pub enum MovieInstruction {
    AddMovieReview {
        title: String,
        rating: u8,
        description: String,
    },
    UpdateMovieReview {
        title: String,
        rating: u8,
        description: String,
    },
    AddComment {
        comment: String,
    },
    InitializeMint,
}
```

Не забудьте додати її до оператора `match` у функції `unpack` в тому ж файлі під варіантом `3`.

```rust
impl MovieInstruction {
    pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
        let (&variant, rest) = input
            .split_first()
            .ok_or(ProgramError::InvalidInstructionData)?;
        Ok(match variant {
            0 => {
                let payload = MovieReviewPayload::try_from_slice(rest).unwrap();
                Self::AddMovieReview {
                    title: payload.title,
                    rating: payload.rating,
                    description: payload.description,
                }
            }
            1 => {
                let payload = MovieReviewPayload::try_from_slice(rest).unwrap();
                Self::UpdateMovieReview {
                    title: payload.title,
                    rating: payload.rating,
                    description: payload.description,
                }
            }
            2 => {
                let payload = CommentPayload::try_from_slice(rest).unwrap();
                Self::AddComment {
                    comment: payload.comment,
                }
            }
            3 => Self::InitializeMint,
            _ => return Err(ProgramError::InvalidInstructionData),
        })
    }
}
```

У функції `process_instruction` у файлі `processor.rs` додайте нову інструкцію до оператора `match` і викликайте функцію `initialize_token_mint`.

```rust
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let instruction = MovieInstruction::unpack(instruction_data)?;
    match instruction {
        MovieInstruction::AddMovieReview {
            title,
            rating,
            description,
        } => add_movie_review(program_id, accounts, title, rating, description),
        MovieInstruction::UpdateMovieReview {
            title,
            rating,
            description,
        } => update_movie_review(program_id, accounts, title, rating, description),
        MovieInstruction::AddComment { comment } => add_comment(program_id, accounts, comment),
        MovieInstruction::InitializeMint => initialize_token_mint(program_id, accounts),
    }
}
```

Останнім кроком буде оголошення та реалізація функції `initialize_token_mint`. Ця функція виконуватиме три основні дії:

1. Визначатиме адреси для token mint та mint authority за допомогою PDA.
2. Створюватиме акаунт для token mint.
3. Ініціалізуватиме token mint, тобто налаштовуватиме його для використання, задаючи необхідні параметри, як-от кількість десяткових знаків для токенів.

Ми не будемо детально пояснювати кожен крок, але варто ознайомитись із кодом, оскільки створення та ініціалізація токен мінту передбачають використання CPI. Якщо вам потрібно повторити теми про токени та мінти, перегляньте урок про [Token Program](./token-program).

```rust
pub fn initialize_token_mint(program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();

    let initializer = next_account_info(account_info_iter)?;
    let token_mint = next_account_info(account_info_iter)?;
    let mint_auth = next_account_info(account_info_iter)?;
    let system_program = next_account_info(account_info_iter)?;
    let token_program = next_account_info(account_info_iter)?;
    let sysvar_rent = next_account_info(account_info_iter)?;

    let (mint_pda, mint_bump) = Pubkey::find_program_address(&[b"token_mint"], program_id);
    let (mint_auth_pda, _mint_auth_bump) =
        Pubkey::find_program_address(&[b"token_auth"], program_id);

    msg!("Token mint: {:?}", mint_pda);
    msg!("Mint authority: {:?}", mint_auth_pda);

    if mint_pda != *token_mint.key {
        msg!("Incorrect token mint account");
        return Err(ReviewError::IncorrectAccountError.into());
    }

    if *token_program.key != TOKEN_PROGRAM_ID {
        msg!("Incorrect token program");
        return Err(ReviewError::IncorrectAccountError.into());
    }

    if *mint_auth.key != mint_auth_pda {
        msg!("Incorrect mint auth account");
        return Err(ReviewError::IncorrectAccountError.into());
    }

    let rent = Rent::get()?;
    let rent_lamports = rent.minimum_balance(82);

    invoke_signed(
        &system_instruction::create_account(
            initializer.key,
            token_mint.key,
            rent_lamports,
            82,
            token_program.key,
        ),
        &[
            initializer.clone(),
            token_mint.clone(),
            system_program.clone(),
        ],
        &[&[b"token_mint", &[mint_bump]]],
    )?;

    msg!("Created token mint account");

    invoke_signed(
        &initialize_mint(
            token_program.key,
            token_mint.key,
            mint_auth.key,
            Option::None,
            9,
        )?,
        &[token_mint.clone(), sysvar_rent.clone(), mint_auth.clone()],
        &[&[b"token_mint", &[mint_bump]]],
    )?;

    msg!("Initialized token mint");

    Ok(())
}
```

### 7. Побудуйте і розгорніть

Зараз ми готові до того, щоб побудувати і розгорнути нашу програму! Щоб побудувати програму, запустіть команду `cargo build-bpf`, а потім команду, яка буде повернена, вона має виглядати приблизно так: `solana program deploy <PATH>`.

Перед тим, як ви почнете тестувати, чи додаються токени при стовренні огляду чи коментаря, вам потрібно ініціалізувати токен мінт програми. Ви можете використовувати для цього [цей скрипт](https://github.com/Unboxed-Software/solana-movie-token-client). Після того, як ви клонували цей репозиторій, замініть `PROGRAM_ID` в файлі `index.ts` на ідентифікатор вашої програми. Потім запустіть команди `npm install` і `npm start`. Скрипт передбачає, що ви розгортаєте програму на Devnet. Якщо ви розгортаєте локально, переконайтесь, що налаштували скрипт відповідно.

Після ініціалізації токен мінта ви можете використовувати [фронтенд для Movie Review](https://github.com/Unboxed-Software/solana-movie-frontend/tree/solution-add-tokens) для тестування додавання рецензій та коментарів. Знову ж таки, код припускає, що ви працюєте в Devnet, тому дійте відповідно.

Після відправлення рецензії ви повинні побачити 10 нових токенів у своєму гаманці! Коли ви додаєте коментар, ви отримаєте 5 токенів. Вони не матимуть красивого імені чи зображення, оскільки ми не додавали метаданих до токену, але суть зрозуміла.

Якщо вам потрібно більше часу для вивчення концепцій цього уроку або ви зіштовхнулися з проблемами, не соромтеся [переглянути код рішення](https://github.com/Unboxed-Software/solana-movie-program/tree/solution-add-tokens). Зверніть увагу, що рішення для цього лабораторного завдання знаходиться в гілці `solution-add-tokens`.

# Завдання

Щоб застосувати те, що ви вивчили про CPI в цьому уроці, подумайте, як ви могли б впровадити їх у програму Student Intro. Ви могли б зробити щось схоже на те, що ми робили в лабораторрій роботі, і додати функціональність для створення токенів користувачам, коли вони представляються. Або, якщо ви почуваєтеся амбіційно, подумайте, як ви могли б використати всі знання, здобуті на курсі, щоб створити щось абсолютно нове з нуля.

Чудовим прикладом було б створити децентралізовану версію Stack Overflow. Програма може використовувати токени для визначення загального рейтингу користувача, карбувати токени, коли даються правильні відповіді на питання, дозволяти користувачам голосувати за відповіді тощо. Все це можливо, і тепер ви маєте навички та знання для створення чогось подібного самостійно!

Вітаємо вас із завершенням Модулю 4! Не соромтеся [поділитися швидким відгуком](https://airtable.com/shrOsyopqYlzvmXSC?prefill_Module=Module%204), щоб ми могли продовжити вдосконалювати курс.


## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=ade5d386-809f-42c2-80eb-a6c04c471f53)!
