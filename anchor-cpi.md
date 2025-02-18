---
назва: Anchor CPIs та Помилки
завдання:
- Виконати Міжпрограмні виклики (Cross Program Invocations (CPI)) з програми Anchor.
- Використати `cpi` для генерації допоміжних функцій для виклику інструкцій у вже існуючих програмах Anchor.
- Використати `invoke` та `invoke_signed` для CPI, коли допоміжні функції CPI недоступні.
- Створити і використати кастомні помилки Anchor 
---

# Стислий виклад

- Anchor надає спрощений спосіб створення CPI за допомогою **`CpiContext`**.
- Функція `cpi` в Anchor генерує допоміжні функції для CPI, які дозволяють викликати інструкції у вже існуючих програмах Anchor. Це надає зручний і автоматизований спосіб взаємодії з іншими програмами, спрощуючи процес розробки та використання міжпрограмних викликів.
- Якщо у вас немає доступу до допоміжних функцій для CPI, ви все ще можете використовувати функції `invoke` та `invoke_signed` безпосередньо.
- Макрос-атрибут `error_code` використовується для створення кастомних помилок Anchor

# Урок

Якщо згадати [перший урок з CPI](cpi), то ви пам'ятаєте, що конструювання CPI може бути складним у стандартному Rust. Однак Anchor трохи спрощує цей процес, особливо якщо програма, яку ви викликаєте, також є програмою Anchor, і ви маєте доступ до її бібліотеки.

На цьому уроці ви навчитесь конструювати CPI за допомогою Anchor. Ви також дізнаєтеся, як створювати кастомні помилки у програмі Anchor, щоб мати можливість писати більш вдосконалені програми на базі Anchor.

## Міжпрограмні виклики (CPI) з використанням Anchor

Нагадаємо, міжпрограмні виклики (CPI) дозволяють програмам викликати інструкції інших програм за допомогою функцій `invoke` чи `invoke_signed`. Це дозволяє новим програмам будувати на основі існуючих програм (ми називаємо це композабельністю).

Хоча створення CPI безпосередньо за допомогою `invoke` чи `invoke_signed` робочий варіант, Anchor також надає спрощений спосіб створювати CPI, використовуючи `CpiContext`.

У цьому уроці ви будете використовувати бібліотеку `anchor_spl` для створення CPI до програми SPL Token. Ви можете [дослідити, що є доступним у бібліотеці `anchor_spl`](https://docs.rs/anchor-spl/latest/anchor_spl/#).

### `CpiContext`

Перший крок у створенні CPI - це створення екземпляру `CpiContext`. `CpiContext` дуже схожий на `Context`, тип першого аргументу, який вимагається функціями інструкцій Anchor. Вони обидва оголошені в одному і тому ж модулі і мають подібні функціональні можливості.

Тип `CpiContext` визначає необов'язкові вхідні дані для міжпрограмних викликів:

- `accounts` - список акаунтів, необхідних для виклику інструкції
- `remaining_accounts` - будь-які залишкові акаунти
- `program` - ідентифікатор програми, яка викликається
- `signer_seeds` - якщо PDA підписує, то для отримання PDA необхідно включити seeds

```rust
pub struct CpiContext<'a, 'b, 'c, 'info, T>
where
    T: ToAccountMetas + ToAccountInfos<'info>,
{
    pub accounts: T,
    pub remaining_accounts: Vec<AccountInfo<'info>>,
    pub program: AccountInfo<'info>,
    pub signer_seeds: &'a [&'b [&'c [u8]]],
}
```

Використайте `CpiContext::new` для конструювання нового екземпляра, коли передаєте початковий підпис транзакції.

```rust
CpiContext::new(cpi_program, cpi_accounts)
```

```rust
pub fn new(
        program: AccountInfo<'info>,
        accounts: T
    ) -> Self {
    Self {
        accounts,
        program,
        remaining_accounts: Vec::new(),
        signer_seeds: &[],
    }
}
```

Використайте `CpiContext::new_with_signer` для створення нового екземпляра, коли підписуєте від імені програми з публічним ключем PDA (Program Derived Address) для CPI.

```rust
CpiContext::new_with_signer(cpi_program, cpi_accounts, seeds)
```

```rust
pub fn new_with_signer(
    program: AccountInfo<'info>,
    accounts: T,
    signer_seeds: &'a [&'b [&'c [u8]]],
) -> Self {
    Self {
        accounts,
        program,
        signer_seeds,
        remaining_accounts: Vec::new(),
    }
}
```

### CPI акаунти 

Одна з головних особливостей `CpiContext`, яка спрощує міжпрограмні виклики, - це те, що аргумент `accounts` є параметром загального типу, що дозволяє передавати будь-який об'єкт, який використовує трейти `ToAccountMetas` та `ToAccountInfos<'info>`.

Ці трейти додаються макрос-атрибутом `#[derive(Accounts)]`, який ви використовували раніше при створенні структур для представлення акаунтів інструкцій. Це означає, що ви можете використовувати схожі структури із `CpiContext`.

Це полегшує організацію коду та забезпечує типову безпеку.

### Виклик інструкції в іншій програмі Anchor

Коли програма, яку ви викликаєте, є програмою Anchor з опублікованою бібліотекою, Anchor може генерувати будівельники інструкцій та допоміжні функції CPI для вас.

Просто визначте залежність вашої програми від програми, яку ви викликаєте, у файлі `Cargo.toml` вашої програми наступним чином:

```
[dependencies]
callee = { path = "../callee", features = ["cpi"]}
```

Додаючи `features = ["cpi"]`, ви активуєте функцію `cpi`, і ваша програма отримує доступ до модулю `callee::cpi`.

Модуль `cpi` викладає інструкції `callee` як функції Rust, які приймають аргументи `CpiContext` та будь-які додаткові дані інструкції. Ці функції мають той же формат, що й функції інструкцій у ваших програмах Anchor, тільки з `CpiContext` замість `Context`. Модуль `cpi` також надає структури акаунтів, необхідні для виклику інструкцій.

Наприклад, якщо у `callee` є інструкція `do_something`, для якої потрібні акаунти, визначені в структурі `DoSomething`, ви можете викликати `do_something` наступним чином:

```rust
use anchor_lang::prelude::*;
use callee;
...

#[program]
pub mod lootbox_program {
    use super::*;

    pub fn call_another_program(ctx: Context<CallAnotherProgram>, params: InitUserParams) -> Result<()> {
        callee::cpi::do_something(
            CpiContext::new(
                ctx.accounts.callee.to_account_info(),
                callee::DoSomething {
                    user: ctx.accounts.user.to_account_info()
                }
            )
        )
        Ok(())
    }
}
...
```

### Виклик інструкції на програму, яка не є програмою Anchor

Коли програма, яку ви викликаєте, не є програмою Anchor, існують два можливі варіанти:

1. Є можливість, що розробники програми опублікували бібліотеку із власними допоміжними функціями для виклику їх програми. Наприклад, бібліотека `anchor_spl` надає допоміжні функції, які практично ідентичні за точкою виклику тому, що ви отримаєте з модулем `cpi` програми Anchor. Наприклад, ви можете створити токен за допомогою [допоміжної функції  `mint_to`](https://docs.rs/anchor-spl/latest/src/anchor_spl/token.rs.html#36-58) та використовувати структуру акаунтів [`MintTo`](https://docs.rs/anchor-spl/latest/anchor_spl/token/struct.MintTo.html).
   ```rust
    token::mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::MintTo {
                mint: ctx.accounts.mint_account.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint_authority").unwrap()],
            ]]
        ),
        amount,
    )?;
    ```
2. Якщо для програми, інструкцію(ї) якої потрібно викликати, немає допоміжного модуля, ви можете скористатися `invoke` та `invoke_signed`. Насправді, вихідний код допоміжної функції  `mint_to`, на яку було посилання вище, показує приклад використання `invoke_signed`, коли передається `CpiContext`. Ви можете притримуватись подібного шаблону, якщо вирішите використовувати структуру акаунтів та `CpiContext` для організації та підготовки вашого CPI.
    ```rust
    pub fn mint_to<'a, 'b, 'c, 'info>(
        ctx: CpiContext<'a, 'b, 'c, 'info, MintTo<'info>>,
        amount: u64,
    ) -> Result<()> {
        let ix = spl_token::instruction::mint_to(
            &spl_token::ID,
            ctx.accounts.mint.key,
            ctx.accounts.to.key,
            ctx.accounts.authority.key,
            &[],
            amount,
        )?;
        solana_program::program::invoke_signed(
            &ix,
            &[
                ctx.accounts.to.clone(),
                ctx.accounts.mint.clone(),
                ctx.accounts.authority.clone(),
            ],
            ctx.signer_seeds,
        )
        .map_err(Into::into)
    }
    ```

## Помилки в Anchor

Ми вже достатньо розібралися в Anchor, тож можемо розібратися з тим, як створювати кастомні помилки.

У кінці кінців, всі програми повертають один і той же тип помилки: [`ProgramError`](https://docs.rs/solana-program/latest/solana_program/program_error/enum.ProgramError.html). Однак, коли ви пишете програму з використанням Anchor, ви можете використовувати `AnchorError` як абстракцію поверх `ProgramError`. Ця абстракція надає додаткову інформацію при невдачі програми, зокрема:

- Назву та номер помилки
- Місце в коді, де була викинута помилка
- Обліковий запис, який порушив обмеження

```rust
pub struct AnchorError {
    pub error_name: String,
    pub error_code_number: u32,
    pub error_msg: String,
    pub error_origin: Option<ErrorOrigin>,
    pub compared_values: Option<ComparedValues>,
}
```

Помилки Anchor можна розділити на дві категорії:

- Внутрішні помилки Anchor: це помилки, які фреймворк повертає з власного коду.
- Кастомні помилки: це помилки, які ви, розробник, можете створити.

Ви можете додати унікальні для своєї програми помилки, використовуючи атрибут `error_code`. Просто додайте цей атрибут до власного типу `enum`. Потім ви можете використовувати варіанти `enum` як помилки у вашій програмі. Додатково, ви можете додати повідомлення про помилку до кожного варіанта, використовуючи атрибут `msg`. Клієнти можуть відображати це повідомлення про помилку у разі її виникнення.

```rust
#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

Щоб повернути власну помилку, ви можете використовувати макроси [err](https://docs.rs/anchor-lang/latest/anchor_lang/macro.err.html) або [error](https://docs.rs/anchor-lang/latest/anchor_lang/prelude/macro.error.html) з функції інструкції. Вони додають інформацію про файл і рядок до помилки, яка потім логується Anchor для допомоги вам при дебагінгу коду.

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        if data.data >= 100 {
            return err!(MyError::DataTooLarge);
        }
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

Альтернативно, ви можете використовувати макрос [require](https://docs.rs/anchor-lang/latest/anchor_lang/macro.require.html), щоб спростити повернення помилок. Код вище може бути перероблений наступний чином:

```rust
#[program]
mod hello_anchor {
    use super::*;
    pub fn set_data(ctx: Context<SetData>, data: MyAccount) -> Result<()> {
        require!(data.data < 100, MyError::DataTooLarge);
        ctx.accounts.my_account.set_inner(data);
        Ok(())
    }
}

#[error_code]
pub enum MyError {
    #[msg("MyAccount may only hold data below 100")]
    DataTooLarge
}
```

# Лабораторна робота

Давайте закріпимо концепції, які ми розглянули у цьому уроці, розширивши програму обзору фільмів з попередніх уроків.

У цьому завданні ми оновимо програму для того, щоб вона могла створювати токени для користувачів, які затверджують новий обзор фільму.

### 1. Початковий код

Для початку ми використаємо останній стан програми Anchor Movie Review з попереднього уроку. Так що, якщо ви щойно завершили цей урок, ви готові йти вперед. Якщо ви тільки починаєте, не хвилюйтеся, ви можете [завантажити початковий код](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-pdas). Ми будемо використовувати гілку `solution-pdas` як наш стартовий пункт.

### 2. Додайте залежності в `Cargo.toml`

Перш ніж ми розпочнемо, нам потрібно активувати функцію `init-if-needed` та додати бібліотеку `anchor-spl` до залежностей в `Cargo.toml`. Якщо вам потрібно освіжити пам'ять щодо функції `init-if-needed`, перегляньте [урок Anchor PDAs and Accounts](anchor-pdas).

```rust
[dependencies]
anchor-lang = { version = "0.25.0", features = ["init-if-needed"] }
anchor-spl = "0.25.0"
```

### 3. Ініціалізуйте токен винагороди

Далі перейдіть до файлу `lib.rs` та створіть інструкцію для ініціалізації нового токен мінта. Це буде токен, який мінтиться кожного разу, коли користувач залишає відгук. Зверніть увагу, що нам не потрібно включати жодної специфічної логіки інструкції, оскільки ініціалізацію можна повністю обробити через обмеження Anchor.

```rust
pub fn initialize_token_mint(_ctx: Context<InitializeMint>) -> Result<()> {
    msg!("Token mint initialized");
    Ok(())
}
```

Тепер реалізуйте тип контексту `InitializeMint` та створіть список акаунтів та обмежень, яких вимагає інструкція. Тут ми ініціалізуємо новий акаунт `Mint`, використовуючи PDA із рядком "mint" як seed. Зверніть увагу, що ми можемо використовувати той самий PDA як для адреси акаунту `Mint`, так і для уповноваженого мінт-акаунту. Використання PDA як уповноваженого мінт-акаунту дозволяє нашій програмі підписувати транзакції мінта токенів.

Для ініціалізації облікового запису `Mint` нам знадобляться `token_program`, `rent`, та `system_program` в списку акаунтів.

```rust
#[derive(Accounts)]
pub struct InitializeMint<'info> {
    #[account(
        init,
        seeds = ["mint".as_bytes()],
        bump,
        payer = user,
        mint::decimals = 6,
        mint::authority = mint,
    )]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
    pub system_program: Program<'info, System>
}
```

Можливо, є деякі обмеження, які ви ще не бачили. Додавання `mint::decimals` та `mint::authority` разом з `init` гарантує, що акаунт ініціалізується як новий токен-мінт з відповідними значеннями для десяткових знаків та встановленим уповноваженим акаунтом.

### 4. Помилка Anchor

Далі створимо помилку Anchor, яку ми будемо використовувати при перевірці `rating`, переданого інструкцією `add_movie_review` або `update_movie_review`.

```rust
#[error_code]
enum MovieReviewError {
    #[msg("Rating must be between 1 and 5")]
    InvalidRating
}
```

### 5. Оновіть інструкцію `add_movie_review`

Тепер, коли ми виконали певні підготовчі роботи, давайте оновимо інструкцію `add_movie_review` та тип контексту `AddMovieReview`, щоб мінтити токени рецензентові.

Далі оновіть тип контексту `AddMovieReview`, щоб додати наступні акаунти:

- `token_program` - ми будемо використовувати Token Program для мінту токенів
- `mint` - мінт-акаунт для токенів, які ми будемо мінтити користувачам під час додавання обзору фільму
- `token_account` - пов'язаний токен-акаунт для зазначеного вище `mint` та рецензента
- `associated_token_program` - потрібний, оскільки ми будемо використовувати обмеження `associated_token` для акаунту `token_account`
- `rent` - потрібний, оскільки ми використовуємо обмеження `init-if-needed` для акаунту `token_account` 

```rust
#[derive(Accounts)]
#[instruction(title: String, description: String)]
pub struct AddMovieReview<'info> {
    #[account(
        init,
        seeds=[title.as_bytes(), initializer.key().as_ref()],
        bump,
        payer = initializer,
        space = 8 + 32 + 1 + 4 + title.len() + 4 + description.len()
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
    // ADDED ACCOUNTS BELOW
    pub token_program: Program<'info, Token>,
    #[account(
        seeds = ["mint".as_bytes()]
        bump,
        mut
    )]
    pub mint: Account<'info, Mint>,
    #[account(
        init_if_needed,
        payer = initializer,
        associated_token::mint = mint,
        associated_token::authority = initializer
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub rent: Sysvar<'info, Rent>
}
```

Знову ж таки, деякі з вказаних вище обмежень можуть бути вам невідомі. Обмеження `associated_token::mint` і `associated_token::authority`, разом з обмеженням `init_if_needed`, забезпечують, що якщо акаунт ще не був ініціалізований, він буде ініціалізований як токен-акаунт, пов’язаний із вказаними мінт-акаунтом і уповноваженим акаунтом.

Далі давайте оновимо інструкцію `add_movie_review`, щоб виконати наступне:

- Перевірити, що `rating` є допустимим. Якщо це не є допустимий рейтинг, поверніть помилку `InvalidRating`.
- Виконати CPI до інструкції `mint_to` токен-програми, використовуючи PDA уповноваженого мінт-акаунту як підписанта. Зазначте, що ми будемо мінтити 10 токенів користувачеві, але ми повинні скоригувати для десяткових розрядів мінта, зробивши це `10*10^6`.

На щастя, ми можемо використовувати пакет `anchor_spl`, щоб отримати доступ до допоміжних функцій та типів, таких як `mint_to` та `MintTo`, для створення нашого CPI до токен-програми. `mint_to` приймає `CpiContext` та ціле число як аргументи, де ціле число представляє кількість токенів для мінта. `MintTo` може бути використаний для списку акаунтів, які потрібні для інструкції мінта.

```rust
pub fn add_movie_review(ctx: Context<AddMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account created");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.reviewer = ctx.accounts.initializer.key();
    movie_review.title = title;
    movie_review.description = description;
    movie_review.rating = rating;

    mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                authority: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                mint: ctx.accounts.mint.to_account_info()
            },
            &[&[
                "mint".as_bytes(),
                &[*ctx.bumps.get("mint").unwrap()]
            ]]
        ),
        10*10^6
    )?;

    msg!("Minted tokens");

    Ok(())
}
```

### 6. Оновлення інструкції `update_movie_review`

Тут ми лише додаємо перевірку того, що `rating` є допустимим.

```rust
pub fn update_movie_review(ctx: Context<UpdateMovieReview>, title: String, description: String, rating: u8) -> Result<()> {
    msg!("Movie review account space reallocated");
    msg!("Title: {}", title);
    msg!("Description: {}", description);
    msg!("Rating: {}", rating);

    require!(rating >= 1 && rating <= 5, MovieReviewError::InvalidRating);

    let movie_review = &mut ctx.accounts.movie_review;
    movie_review.description = description;
    movie_review.rating = rating;

    Ok(())
}
```

### 7. Тест

Це всі зміни, які нам потрібно зробити в програмі! Тепер давайте оновимо наші тести.

Почніть з того, щоб переконатися, що ваші імпорти та функція `describe` виглядають так:

```typescript
import * as anchor from "@project-serum/anchor"
import { Program } from "@project-serum/anchor"
import { expect } from "chai"
import { getAssociatedTokenAddress, getAccount } from "@solana/spl-token"
import { AnchorMovieReviewProgram } from "../target/types/anchor_movie_review_program"

describe("anchor-movie-review-program", () => {
  // Configure the client to use the local cluster.
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)

  const program = anchor.workspace
    .AnchorMovieReviewProgram as Program<AnchorMovieReviewProgram>

  const movie = {
    title: "Just a test movie",
    description: "Wow what a good movie it was real great",
    rating: 5,
  }

  const [movie_pda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from(movie.title), provider.wallet.publicKey.toBuffer()],
    program.programId
  )

  const [mint] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from("mint")],
    program.programId
  )
...
}
```

Зробивши це, додайте тест для інструкції `initializeTokenMint`:

```typescript
it("Initializes the reward token", async () => {
    const tx = await program.methods.initializeTokenMint().rpc()
})
```

Зверніть увагу, що нам не потрібно додавати `.accounts`, оскільки вони можуть бути виведені, включаючи акаунт `mint` (за умови, що у вас включено виведення seed).

Далі оновіть тест для інструкції `addMovieReview`. Основні додатки такі:
1. Отримати адресу пов’язаного токен-акаунту, яку потрібно передати в інструкцію як акаунт, що не може бути виведений автоматично.  
2. Перевірити в кінці тесту, що пов’язаний токен-акаунт містить 10 токенів.

```typescript
it("Movie review is added`", async () => {
  const tokenAccount = await getAssociatedTokenAddress(
    mint,
    provider.wallet.publicKey
  )
  
  const tx = await program.methods
    .addMovieReview(movie.title, movie.description, movie.rating)
    .accounts({
      tokenAccount: tokenAccount,
    })
    .rpc()
  
  const account = await program.account.movieAccountState.fetch(movie_pda)
  expect(movie.title === account.title)
  expect(movie.rating === account.rating)
  expect(movie.description === account.description)
  expect(account.reviewer === provider.wallet.publicKey)

  const userAta = await getAccount(provider.connection, tokenAccount)
  expect(Number(userAta.amount)).to.equal((10 * 10) ^ 6)
})
```

Після цього тест для `updateMovieReview` або тест для `deleteMovieReview` не потребує жодних змін.

На цьому етапі виконайте команду `anchor test`, і ви повинні побачити наступний вивід

```console
anchor-movie-review-program
    ✔ Initializes the reward token (458ms)
    ✔ Movie review is added (410ms)
    ✔ Movie review is updated (402ms)
    ✔ Deletes a movie review (405ms)

  5 passing (2s)
```

Якщо вам потрібно більше часу для освоєння концепцій цього уроку або ви застрягли, зверніть увагу на [код рішення](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-add-tokens). Зверніть увагу, що рішення для цього завдання розташоване в гілці `solution-add-tokens`.

# Завдання

Щоб застосувати те, що ви вивчили про CPI в цьому уроці, подумайте, як ви могли б їх включити в програму Student Intro. Ви можете зробити щось схоже з тим, що ми робили в лабораторній роботі, і додати функціонал для мінту токенів користувачам під час їхнього представлення.

Спробуйте це зробити самостійно, якщо можете! Але якщо ви застрягли, звертайтеся до цього [рішення](https://github.com/Unboxed-Software/anchor-student-intro-program/tree/cpi-challenge). Варто відмітити, що ваш код може виглядати трошки інакше, ніж рішення, в залежності від вашої реалізації.

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=21375c76-b6f1-4fb6-8cc1-9ef151bc5b0a)!
