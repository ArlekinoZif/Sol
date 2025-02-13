---
назва: Вступ до розробки з Anchor 
завдання:
- Використати фреймворк Anchor для створення базової програми
- Описати базову структуру програми Anchor
- Пояснити, як реалізувати базову валідацію акаунтів та перевірки безпеки за допомогою Anchor
---

# Стислий виклад

- **Anchor** — це фреймворк для створення програм на Solana
- **Макроси Anchor** прискорюють процес розробки програм на Solana, абстрагуючи значну кількість шаблонного коду
- Anchor дозволяє створювати **безпечні програми** простіше, виконуючи певні перевірки безпеки, вимагаючи валідації акаунтів та надаючи простий спосіб реалізувати додаткові перевірки

# Урок

## Що таке Anchor?

Anchor робить написання програм на Solana легшим, швидшим та безпечнішим, що робить його "фреймворком за замовчуванням" для розробки на Solana. Він спрощує організацію та розуміння вашого коду, автоматично реалізує загальні перевірки безпеки та усуває значну частину шаблонного коду, який зазвичай супроводжує написання програми на Solana.

## Структура програми Anchor

Anchor використовує макроси та трейти для автоматичного створення шаблонного коду на Rust. Це надає чітку структуру вашій програмі, що дозволяє легше розуміти та працювати з кодом. Основні макроси та атрибути високого рівня:

- `declare_id` — макрос для оголошення ончейн-адреси програми
- `#[program]` — макрос-атрибут, який використовується для позначення модуля, що містить логіку інструкцій програми
- `Accounts` — трейт, що представляє список акаунтів, необхідних для інструкції
- `#[account]` — макрос-атрибут, який використовується для визначення користувацьких типів акаунтів для програми

Давайте розглянемо кожен з них перед тим, як зібрати всі частини разом.

## Оголошення ID вашої програми

Макрос `declare_id` вказує ончейн-адресу програми (тобто `programId`). Коли ви будуєте програму на Anchor вперше, фреймворк генерує нову пару ключів. Це стає парою ключів за замовчуванням, що використовується для деплою програми, якщо не вказана інша. Відповідний публічний ключ має бути використаний як `programId`, зазначений у макросі `declare_id!`.

```rust
declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");
```

## Визначення логіки інструкцій

Макрос-атрибут `#[program]` визначає модуль, що містить усі інструкції вашої програми. Тут ви реалізуєте бізнес-логіку для кожної інструкції у вашій програмі.

Кожна публічна функція в модулі з атрибутом `#[program]` буде оброблятися як окрема інструкція.

Кожна функція інструкції вимагає параметр типу `Context` і може опціонально включати додаткові параметри функції, що представляють дані інструкції. Anchor автоматично оброблятиме десеріалізацію даних інструкції, щоб ви могли працювати з ними (даними) як з типами Rust.

```rust
#[program]
mod program_module_name {
    use super::*;

    pub fn instruction_one(ctx: Context<InstructionAccounts>, instruction_data: u64) -> Result<()> {
		ctx.accounts.account_name.data = instruction_data;
        Ok(())
    }
}
```

### Аргумент `context`  

Аргумент `context` дозволяє отримати доступ до метаданих інструкції та акаунтів у вашому обробнику інструкцій:  

- ID програми (`ctx.program_id`), яка виконується  
- Акаунти, передані в інструкцію (`ctx.accounts`)  
- Залишкові акаунти (`ctx.remaining_accounts`). `remaining_accounts` — це вектор, що містить усі акаунти, передані в інструкцію, але не оголошені в структурі `Accounts`.  
- Значення bumps для будь-яких PDA-акаунтів у структурі `Accounts` (`ctx.bumps`)

## Визначення акаунтів інструкції  

Трейт `Accounts` визначає структуру даних для валідації акаунтів. Структури, що імплементують цей трейт, задають список акаунтів, необхідних для виконання певної інструкції. Ці акаунти стають доступними через `Context` інструкції, що усуває потребу вв ручному переборі та десеріалізації акаунтів.

Зазвичай трейт `Accounts` застосовується через макрос `derive` (наприклад, `#[derive(Accounts)]`). Це реалізує десеріалізатор `Accounts` для вказаної структури та усуває потребу в ручній десеріалізації кожного акаунту.

Реалізації трейту `Accounts` відповідають за виконання всіх необхідних перевірок обмежень, щоб гарантувати, що акаунти відповідають умовам, необхідним для безпечного виконання програми. Обмеження задаються для кожного поля за допомогою макрос-атрибута `#[account(..)]` (докладніше про це далі).

Наприклад, `instruction_one` вимагає аргумент `Context` типу `InstructionAccounts`. Макрос `#[derive(Accounts)]` використовується для реалізації структури `InstructionAccounts`, яка містить три акаунти: `account_name`, `user` і `system_program`.

```rust
#[program]
mod program_module_name {
    use super::*;
    pub fn instruction_one(ctx: Context<InstructionAccounts>, instruction_data: u64) -> Result<()> {
		...
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InstructionAccounts {
    #[account(init, payer = user, space = 8 + 8)]
    pub account_name: Account<'info, AccountStruct>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,

}
```

Коли викликається `instruction_one`, програма:  

- Перевіряє, чи відповідають передані в інструкцію акаунти типам акаунтів, зазначеним у структурі `InstructionAccounts`  
- Перевіряє акаунти на відповідність додатковим зазначеним обмеженням

Якщо будь-який із акаунтів, переданих у `instruction_one`, не проходить валідацію акаунтів або перевірки безпеки, визначені в структурі `InstructionAccounts`, то інструкція завершиться з помилкою ще до виконання логіки програми.

## Валідація акаунтів  

У попередньому прикладі ви могли помітити, що один із акаунтів у `InstructionAccounts` був типу `Account`, інший — типу `Signer`, а ще один — типу `Program`.

Anchor надає кілька типів акаунтів, які можна використовувати для представлення акаунтів. Кожен тип реалізує різну валідацію акаунтів. Ми розглянемо кілька з найбільш поширених типів, з якими ви можете зіткнутися, але обов'язково перегляньте [повний список типів акаунтів](https://docs.rs/anchor-lang/latest/anchor_lang/accounts/index.html).

### `Account`

`Account` — це обгортка навколо `AccountInfo`, яка перевіряє право власності програми та десеріалізує основні дані в тип Rust.

```rust
// Deserializes this info
pub struct AccountInfo<'a> {
    pub key: &'a Pubkey,
    pub is_signer: bool,
    pub is_writable: bool,
    pub lamports: Rc<RefCell<&'a mut u64>>,
    pub data: Rc<RefCell<&'a mut [u8]>>,    // <---- deserializes account data
    pub owner: &'a Pubkey,    // <---- checks owner program
    pub executable: bool,
    pub rent_epoch: u64,
}
```

Згадаємо попередній приклад, де у `InstructionAccounts` було поле `account_name`:

```rust
pub account_name: Account<'info, AccountStruct>
```

Обгортка `Account` тут виконує наступне:

- Десеріалізує дані акаунту у формат типу `AccountStruct`
- Перевіряє, чи власник програми акаунту збігається з власником програми, зазначеним для типу `AccountStruct`

Коли тип акаунту, зазначений в обгортці `Account`, визначено в тій же бібліотеці за допомогою макроса-атрибута `#[account]`, перевірка власності програми здійснюється щодо `programId`, визначеного в макросі `declare_id!`.

Ось перевірки, які виконуються:

```rust
// Checks
Account.info.owner == T::owner()
!(Account.info.owner == SystemProgram && Account.info.lamports() == 0)
```

### `Signer`

Тип `Signer` перевіряє, що вказаний акаунт підписав транзакцію. Інші перевірки на власність або тип не виконуються. Використовувати `Signer` слід лише тоді, коли дані акаунту не потрібні в інструкції.

Для акаунту `user` в попередньому прикладі тип `Signer` вказує, що акаунт `user` повинен бути підписантом інструкції.

Ось перевірка, яка виконується автоматично:

```rust
// Checks
Signer.info.is_signer == true
```

### `Program`

Тип `Program` перевіряє, що акаунт є певною програмою.

Для акаунту `system_program` в попередньому прикладі використовується тип `Program`, щоб вказати, що акаунт має бути системною програмою. Anchor надає тип `System`, який включає `programId` системної програми для перевірки.

Ось перевірки, які виконуються автоматично:

```rust
//Checks
account_info.key == expected_program
account_info.executable == true
```

## Додавання обмежень за допомогою `#[account(..)]`  

Макрос-атрибут `#[account(..)]` використовується для застосування обмежень до акаунтів. Ми розглянемо кілька прикладів обмежень у цьому та наступних уроках, але обов’язково перегляньте повний [список можливих обмежень](https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html).

Згадаємо ще раз поле `account_name` із прикладу `InstructionAccounts`.

```rust
#[account(init, payer = user, space = 8 + 8)]
pub account_name: Account<'info, AccountStruct>,
#[account(mut)]
pub user: Signer<'info>,
```

Зверніть увагу, що атрибут `#[account(..)]` містить три значення, розділені комами:  

- `init` — створює акаунт через CPI до системної програми та ініціалізує його (встановлює дискримінатор акаунту).  
- `payer` — вказує, що платником за ініціалізацію акаунту є акаунт `user`, визначений у структурі.  
- `space` — задає розмір пам’яті, виділеної для акаунту, який у цьому випадку становить `8 + 8` байт. Перші 8 байт відводяться для дискримінатора, який Anchor автоматично додає для ідентифікації типу акаунту. Наступні 8 байтів виділяють місце для даних, що зберігатимуться в акаунті згідно з типом `AccountStruct`.

Для `user` ми використовуємо атрибут `#[account(..)]`, щоб вказати, що цей акаунт є змінним. Акаунт `user` має бути позначений як змінний, оскільки з нього буде списано лампорти для оплати ініціалізації `account_name`.

```rust
#[account(mut)]
pub user: Signer<'info>,
```

Зверніть увагу, що обмеження `init`, застосоване до `account_name`, автоматично включає обмеження `mut`, тому `account_name` і `user` є змінними акаунтами.

## `#[account]`

Атрибут `#[account]` застосовується до структур, що представляють структуру даних акаунту Solana. Він реалізує такі трейти:

- `AccountSerialize`
- `AccountDeserialize`
- `AnchorSerialize`
- `AnchorDeserialize`
- `Clone`
- `Discriminator`
- `Owner`

Ви можете дізнатися більше про [деталі кожного трейту](https://docs.rs/anchor-lang/latest/anchor_lang/attr.account.html). Однак головне, що потрібно знати: атрибут `#[account]` забезпечує серіалізацію та десеріалізацію, а також реалізує трейти дискримінатора та власника акаунту.

Дискримінатор — це унікальний 8-байтовий ідентифікатор типу акаунту, отриманий з перших 8 байт хешу SHA256 від імені цього типу. Перші 8 байтів зарезервовані для дискримінатора акаунту при реалізації трейту серіалізації акаунту (що майже завжди використовується в Anchor-програмах).

У результаті будь-який виклик `try_deserialize` з `AccountDeserialize` перевірятиме цей дискримінатор. Якщо він не збігається, це означає, що передано некоректний акаунт, і десеріалізація завершиться помилкою.

Атрибут `#[account]` також реалізує трейт `Owner` для структури, використовуючи `programId`, оголошений за допомогою `declareId` у бібліотеці, в якій використовується `#[account]`. Іншими словами, всі акаунти, ініціалізовані за допомогою типу акаунту, визначеного за допомогою атрибута `#[account]` в програмі, також належать цій програмі.

Як приклад, давайте подивимося на `AccountStruct`, що використовується в `account_name` з `InstructionAccounts`.

```rust
#[derive(Accounts)]
pub struct InstructionAccounts {
    #[account(init, payer = user, space = 8 + 8)]
    pub account_name: Account<'info, AccountStruct>,
    ...
}

#[account]
pub struct AccountStruct {
    data: u64
}
```

Атрибут `#[account]` гарантує, що він може бути використаний як акаунт в `InstructionAccounts`.

Коли акаунт `account_name` ініціалізується:

- Перші 8 байт встановлюються як дискримінатор `AccountStruct`
- Поле даних акаунту буде відповідати типу `AccountStruct`
- Власником акаунту буде встановлено `programId`, визначений у `declare_id`

## Обʼєднаємо все разом

Коли ви поєднуєте всі ці типи Anchor, ви отримуєте повну програму. Нижче наведено приклад базової Anchor-програми з однією інструкцією, яка:

- Ініціалізує новий акаунт
- Оновлює поле даних акаунту за допомогою даних інструкції, що передаються в інструкцію

```rust
// Use this import to gain access to common anchor features
use anchor_lang::prelude::*;

// Program onchain address
declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

// Instruction logic
#[program]
mod program_module_name {
    use super::*;
    pub fn instruction_one(ctx: Context<InstructionAccounts>, instruction_data: u64) -> Result<()> {
        ctx.accounts.account_name.data = instruction_data;
        Ok(())
    }
}

// Validate incoming accounts for instructions
#[derive(Accounts)]
pub struct InstructionAccounts<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub account_name: Account<'info, AccountStruct>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,

}

// Define custom program account type
#[account]
pub struct AccountStruct {
    data: u64
}
```

Тепер ви готові створити свою власну програму Solana, використовуючи фреймворк Anchor!

# Лабораторна робота

Перед тим, як почати, встановіть Anchor, дотримуючись [кроків з документації Anchor](https://www.anchor-lang.com/docs/installation).

Для цього лабораторного заняття ми створимо просту програму-лічильник з двома інструкціями:

- Перша інструкція ініціалізує акаунт для зберігання нашого лічильника
- Друга інструкція збільшить значення лічильника, що зберігається в акаунті

### 1. Налаштування

Створіть новий проект під назвою `anchor-counter`, запустивши команду `anchor init`:

```console
anchor init anchor-counter
```

Перейдіть у нову директорію, а потім виконайте команду `anchor build`:

```console
cd anchor-counter
anchor build
```

Команда `anchor build` також згенерує пару ключів для вашої нової програми — ключі будуть збережені в директорії `target/deploy`.

Відкрийте файл `lib.rs` і подивіться на `declare_id!`:

```rust
declare_id!("BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr");
```

Запустіть команду `anchor keys sync`:

```console
anchor keys sync
```

Ви побачите, що Anchor оновить обидва:

- Ключ, використаний у `declare_id!()` в `lib.rs`
- Ключ в `Anchor.toml`

Щоб він відповідав ключу, згенерованому під час виконання `anchor build`:

```console
Found incorrect program id declaration in "anchor-counter/programs/anchor-counter/src/lib.rs"
Updated to BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr

Found incorrect program id declaration in Anchor.toml for the program `anchor_counter`
Updated to BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr

All program id declarations are synced.
```

Нарешті, видаліть стандартний код у файлі `lib.rs`, залишивши тільки наступне:

```rust
use anchor_lang::prelude::*;

declare_id!("your-private-key");

#[program]
pub mod anchor_counter {
    use super::*;

}
```

### 2. Додайте інструкцію `initialize`

Спочатку давайте реалізуємо інструкцію `initialize` в межах `#[program]`. Ця інструкція вимагає `Context` типу `Initialize` і не приймає додаткових даних інструкції. В логіці інструкції, в `counter` акаунті встановіть значення поля `count` = `0`.

```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    let counter = &mut ctx.accounts.counter;
    counter.count = 0;
    msg!("Counter Account Created");
    msg!("Current Count: { }", counter.count);
    Ok(())
}
```

### 3. Реалізація `Context` для типу `Initialize`  

Далі, використовуючи макрос `#[derive(Accounts)]`, реалізуємо тип `Initialize`, який визначає та валідує акаунти, що використовуються в інструкції `initialize`. Він повинен містити такі акаунти:  

- `counter` – акаунт лічильника, що ініціалізується в інструкції  
- `user` – платник за ініціалізацію  
- `system_program` – системна програма, необхідна для створення нових акаунтів
  
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### 4. Реалізація `Counter`  

Далі, використовуючи атрибут `#[account]`, визначимо новий тип акаунту `Counter`. Структура `Counter` містить одне поле `count` типу `u64`. Це означає, що всі нові акаунти, ініціалізовані як `Counter`, матимуть відповідну структуру даних. Атрибут `#[account]` також автоматично встановлює дискримінатор для нового акаунту та задає його власником `programId`, визначений у макросі `declare_id!`.

```rust
#[account]
pub struct Counter {
    pub count: u64,
}
```

### 5. Додавання інструкції `increment`  

У межах `#[program]` реалізуємо інструкцію `increment`, яка збільшує `count` після ініціалізації акаунту `counter` першою інструкцією. Ця інструкція потребує `Context` типу `Update` (реалізується в наступному кроці) і не приймає додаткових вхідних даних. У логіці інструкції ми просто збільшуємо поле `count` акаунту `counter` на `1`.

```rust
pub fn increment(ctx: Context<Update>) -> Result<()> {
    let counter = &mut ctx.accounts.counter;
    msg!("Previous counter: {}", counter.count);
    counter.count = counter.count.checked_add(1).unwrap();
    msg!("Counter incremented. Current count: {}", counter.count);
    Ok(())
}
```

### 6. Реалізація `Context` для типу `Update`  

Нарешті, використовуючи макрос `#[derive(Accounts)]`, створимо тип `Update`, який визначає акаунти, необхідні для інструкції `increment`. Він міститиме такі акаунти:  

- `counter` — існуючий акаунт лічильника, значення якого буде збільшено  
- `user` — акаунт, що оплачує комісію за транзакцію  

Також потрібно задати відповідні обмеження за допомогою атрибута `#[account(..)]`.

```rust
#[derive(Accounts)]
pub struct Update<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
    pub user: Signer<'info>,
}
```

### 7. Збірка  

У підсумку, повна програма виглядатиме так:

```rust
use anchor_lang::prelude::*;

declare_id!("BouTUP7a3MZLtXqMAm1NrkJSKwAjmid8abqiNjUyBJSr");

#[program]
pub mod anchor_counter {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = 0;
        msg!("Counter account created. Current count: {}", counter.count);
        Ok(())
    }

    pub fn increment(ctx: Context<Update>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        msg!("Previous counter: {}", counter.count);
        counter.count = counter.count.checked_add(1).unwrap();
        msg!("Counter incremented. Current count: {}", counter.count);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Update<'info> {
    #[account(mut)]
    pub counter: Account<'info, Counter>,
    pub user: Signer<'info>,
}

#[account]
pub struct Counter {
    pub count: u64,
}
```

Запустіть `anchor build`, щоб зібрати програму.  

### 8. Тестування  

Тести в Anchor зазвичай є інтеграційними тестами на TypeScript, які використовують фреймворк тестування Mocha. Ми детальніше розглянемо тестування пізніше, а поки що перейдіть до файлу `anchor-counter.ts` і замініть код тесту за замовчуванням наступним:

```typescript
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { expect } from "chai"
import { AnchorCounter } from "../target/types/anchor_counter"

describe("anchor-counter", () => {
  // Configure the client to use the local cluster.
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace.AnchorCounter as Program<AnchorCounter>

  const counter = anchor.web3.Keypair.generate()

  it("Is initialized!", async () => {})

  it("Incremented the count", async () => {})
})
```

Наведений вище код генерує нову пару ключів для акаунту `counter`, який ми ініціалізуватимемо, і створює шаблони для тестування кожної інструкції.  

Далі створимо перший тест для інструкції `initialize`:

```typescript
it("Is initialized!", async () => {
  // Add your test here.
  const tx = await program.methods
    .initialize()
    .accounts({ counter: counter.publicKey })
    .signers([counter])
    .rpc()

  const account = await program.account.counter.fetch(counter.publicKey)
  expect(account.count.toNumber()).to.equal(0)
})
```

Далі створимо другий тест для інструкції `increment`:

```typescript
it("Incremented the count", async () => {
  const tx = await program.methods
    .increment()
    .accounts({ counter: counter.publicKey, user: provider.wallet.publicKey })
    .rpc()

  const account = await program.account.counter.fetch(counter.publicKey)
  expect(account.count.toNumber()).to.equal(1)
})
```

Нарешті, запустіть `anchor test`, і ви повинні побачити наступне:

```console
anchor-counter
✔ Is initialized! (290ms)
✔ Incremented the count (403ms)


2 passing (696ms)
```

Запуск `anchor test` автоматично запускає локальний тестовий валідатор, розгортає вашу програму та виконує тести mocha. Не переживайте, якщо зараз вам здається, що тести складні — ми детально розглянемо це пізніше.

Вітаємо, ви щойно створили програму Solana, використовуючи фреймворк Anchor! Якщо вам потрібно більше часу для розуміння, ви можете звернутися до [кодового рішення](https://github.com/Unboxed-Software/anchor-counter-program/tree/solution-increment).

# Завдання

Тепер ваша черга створити щось самостійно. Оскільки ми починаємо з простих програм, ваша програма буде майже ідентична тій, що ми щойно створили. Це корисно, щоб досягти того рівня, коли ви зможете написати її з нуля без посилань на попередній код, тому намагайтеся не копіювати та не вставляти тут.

1. Напишіть нову програму, яка ініціалізує акаунт `counter`.
2. Реалізуйте інструкції для `increment` (збільшення) та `decrement` (зменшення).
3. Побудуйте та розгорніть вашу програму так, як ми це робили в лабораторній роботі.
4. Протестуйте новостворену програму та використовуйте Solana Explorer для перевірки логів програми.
As always, get creative with these challenges and take them beyond the basic instructions if you want - and have fun!

Спробуйте зробити це самостійно, якщо зможете! Але якщо виникнуть труднощі, ви можете звернутися до [коду рішення](https://github.com/Unboxed-Software/anchor-counter-program/tree/solution-decrement).

## Завершили Лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=334874b7-b152-4473-b5a5-5474c3f8f3f1)!
