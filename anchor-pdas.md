---
заголовок: Anchor PDA і облікові записи
цілі:
- Використовуйте обмеження `seeds` та `bump`, щоб працювати з обліковими записами PDA в Anchor.
- Увімкніть та використовуйте обмеження `init_if_needed`.
- Використовуйте обмеження `realloc`, щоб перерозподілити простір на існуючому обліковому записі.
- Використовуйте обмеження `close`, щоб закрити існуючий обліковий запис.
---

# Стислий виклад

- Обмеження `seeds` та `bump` використовуються для ініціалізації та перевірки облікових записів PDA в Anchor.
- Обмеження `init_if_needed` використовується для умовної ініціалізації нового облікового запису.
- Обмеження `realloc` використовується для перерозподілу простору на існуючому обліковому записі.
- Обмеження `close` використовується для закриття облікового запису та повернення оплати за оренду.

# Урок

У цьому уроці ви навчитеся працювати з PDA, перерозподіляти облікові записи та закривати їх в Anchor.

Пам'ятайте, що програми Anchor розділяють логіку інструкцій і перевірку облікових записів. Перевірка облікових записів відбувається переважно в межах структур, які представляють список облікових записів, необхідних для певної інструкції. Кожне поле структури представляє різний обліковий запис, і ви можете налаштувати перевірку облікового запису за допомогою макроса атрибутів #[account(...)].

У цьому уроці ви ознайомитеся з обмеженнями `seeds`, `bump`, `realloc` та `close`, які допомагатимуть вам ініціалізувати та перевірити PDA, перерозподілити облікові записи та закрити їх. Ці обмеження виконують не лише перевірку облікових записів, але й автоматизують повторювані завдання, що в іншому випадку потребували б великої кількості шаблонного коду всередині нашої логіки інструкції.

## PDA з Anchor

Пам'ятайте, що [PDA](https://github.com/Unboxed-Software/solana-course/blob/main/content/pda) визначаються за допомогою списку опціональних seeds, bump seed та ID програми. Anchor надає зручний спосіб перевірити PDA за допомогою обмежень `seeds` та `bump`.

```rust
#[derive(Accounts)]
struct ExampleAccounts {
  #[account(
    seeds = [b"example_seed"],
    bump
  )]
  pub pda_account: Account<'info, AccountType>,
}
```

Під час перевірки облікового запису, Anchor виведе PDA, використовуючи seeds, уточнені в обмеженні `seeds`, та перевірить, що обліковий запис, переданий до інструкції, відповідає знайденому PDA за вказаними `seeds`.

Якщо обмеження `bump` включено без вказівки конкретного bump, Anchor за замовчуванням буде використовувати канонічний bump (перший bump, який призводить до дійсного PDA). У більшості випадків ви повинні використовувати канонічний bump.

Ви можете отримати доступ до інших полів структури з обмеженнями, тому ви можете вказати seeds, що залежать від інших облікових записів, наприклад, від публічного ключа підписника.

Ви також можете посилатися на десеріалізовані дані інструкції, якщо додасте макрос атрибуту `#[instruction(...)]` до структури.

Наприклад, наступний приклад показує список облікових записів, який включає `pda_account` та `user`. `pda_account` обмежений так, що seeds повинні бути рядком "example_seed", публічним ключем `user` та рядком, переданим у інструкцію як `instruction_data`.

```rust
#[derive(Accounts)]
#[instruction(instruction_data: String)]
pub struct Example<'info> {
    #[account(
        seeds = [b"example_seed", user.key().as_ref(), instruction_data.as_ref()],
        bump
    )]
    pub pda_account: Account<'info, AccountType>,
    #[account(mut)]
    pub user: Signer<'info>
}
```

Якщо адреса `pda_account`, надана клієнтом, не відповідає PDA, виведеному за допомогою вказаних seeds та канонічного bump, то перевірка облікового запису буде неуспішною.

### Use PDAs with the `init` constraint

You can combine the `seeds` and `bump` constraints with the `init` constraint to initialize an account using a PDA.

Recall that the `init` constraint must be used in combination with the `payer` and `space` constraints to specify the account that will pay for account initialization and the space to allocate on the new account. Additionally, you must include `system_program` as one of the fields of the account validation struct.

```rust
#[derive(Accounts)]
pub struct InitializePda<'info> {
    #[account(
        init,
        seeds = [b"example_seed", user.key().as_ref()],
        bump,
        payer = user,
        space = 8 + 8
    )]
    pub pda_account: Account<'info, AccountType>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct AccountType {
    pub data: u64,
}
```

При використанні `init` для облікових записів, які не є PDA, Anchor за замовчуванням встановлює власника ініціалізованого облікового запису на поточну програму, яка виконує інструкцію.

Однак, при використанні `init` у поєднанні з `seeds` та `bump`, власник *має* бути виконуючою програмою. Це тому, що ініціалізація облікового запису для PDA вимагає підпису, який може надати лише виконуюча програма. Іншими словами, перевірка підпису для ініціалізації облікового запису PDA буде невдалою, якщо програмний ідентифікатор, використаний для виведення PDA, не відповідає програмному ідентифікатору виконуючої програми.

При визначенні значення `space` для облікового запису, що ініціалізований та належить запущеній програмі Anchor, пам'ятайте, що перші 8 байтів зарезервовані для дискримінатора облікового запису. Це 8-байтове значення, яке Anchor обчислює та використовує для ідентифікації типів облікових записів програми. Ви можете скористатися цим [посиланням](https://www.anchor-lang.com/docs/space), щоб розрахувати, скільки місця ви повинні виділити для облікового запису.

### Seed inference

Список облікових записів для інструкції може стати занадто довгим для деяких програм. Щоб спростити виклик інструкції програми Anchor з боку клієнта, ми можемо увімкнути seed inference.

Seed inference додає інформацію про PDA seeds до IDL, так що Anchor може виводити PDA seeds з існуючої call-site інформації. У попередньому прикладі seeds - це `b"example_seed"` та `user.key()`. Перше є статичним і тому відомим, а друге відоме, оскільки `user` - це підписувач транзакції.

Якщо ви використовуєте seed inference при побудові своєї програми, то якщо ви викликаєте програму за допомогою Anchor, вам не потрібно явно виводити та передавати PDA. Замість цього бібліотека Anchor зробить це за вас.

Ви можете увімкнути seed inference у файлі `Anchor.toml`, вказавши `seeds = true` під `[features]`.

```
[features]
seeds = true
```

### Використовуйте макрос атрибуту `#[instruction(...)]`.

Перш ніж перейти до наступного пункту, давайте коротко розглянемо макрос атрибуту `#[instruction(...)]`. При використанні `#[instruction(...)]` дані інструкції, які ви надаєте у списку аргументів, повинні відповідати та бути в тому ж порядку, що й аргументи інструкції. Ви можете опускати аргументи в кінці списку, що не використовуються, але ви повинні включити всі аргументи до останнього, який ви будете використовувати.

Наприклад, уявіть, що інструкція має аргументи `input_one`, `input_two` та `input_three`. Якщо ваші обмеження облікових записів повинні посилатися на `input_one` та `input_three`, вам потрібно вказати всі три аргументи у макросі атрибуту `#[instruction(...)]`.

Однак, якщо ваші обмеження посилаються лише на `input_one` та `input_two`, ви можете пропустити `input_three`.

```rust
pub fn example_instruction(
    ctx: Context<Example>,
    input_one: String,
    input_two: String,
    input_three: String,
) -> Result<()> {
    ...
    Ok(())
}

#[derive(Accounts)]
#[instruction(input_one:String, input_two:String)]
pub struct Example<'info> {
    ...
}
```

Додатково, ви отримаєте помилку, якщо перерахуєте вхідні дані у неправильному порядку:

```rust
#[derive(Accounts)]
#[instruction(input_two:String, input_one:String)]
pub struct Example<'info> {
    ...
}
```

## Init-if-needed

Anchor надає обмеження `init_if_needed`, яке може бути використане для ініціалізації облікового запису, якщо обліковий запис ще не був ініціалізований.

Ця функція захищена за допомогою флага функцій, щоб ви могли свідомо використовувати її. З погляду безпеки, розумно уникати того, щоб одна інструкція гілкувалася на кілька логічних шляхів. І, як ім'я вказує, `init_if_needed` виконує один з двох можливих кодових шляхів в залежності від стану облікового запису, про який йдеться.

При використанні `init_if_needed` вам потрібно переконатися, що ваша програма належним чином захищена від атак на повторну ініціалізацію. Ви повинні включити перевірки у вашому коді, які перевіряють, що ініціалізований обліковий запис не може бути скинутий до початкових налаштувань після першої ініціалізації.

Щоб використовувати `init_if_needed`, спочатку вам потрібно увімкнути цю функцію в `Cargo.toml`.

```rust
[dependencies]
anchor-lang = { version = "0.25.0", features = ["init-if-needed"] }
```

Після активації функції ви можете включити обмеження в макрос атрибуту `#[account(...)]`. Наведений нижче приклад демонструє використання обмеження `init_if_needed` для ініціалізації нового асоційованого облікового запису токенів, якщо такого ще не існує.

```rust
#[program]
mod example {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init_if_needed,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = payer
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
     #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub rent: Sysvar<'info, Rent>,
}
```

При виклику інструкції `initialize` в попередньому прикладі Anchor перевірить, чи існує `token_account`, і ініціалізує його, якщо такого не існує. Якщо він вже існує, то інструкція продовжить виконуватися без ініціалізації облікового запису. Так само, як з обмеженням `init`, ви можете використовувати `init_if_needed` разом з `seeds` та `bump`, якщо обліковий запис є PDA.

## Realloc

Обмеження `realloc` надає простий спосіб перерозподілу простору для існуючих облікових записів.

Обмеження `realloc` повинно використовуватися в поєднанні з наступними обмеженнями:

- `mut` - обліковий запис повинен бути встановлений як змінний
- `realloc::payer` - обліковий запис для віднімання або додавання лампортів в залежності від того, чи зменшується або збільшується розмір облікового запису
- `realloc::zero` - булеве значення, щоб вказати, чи нова пам'ять повинна бути ініціалізована нулем

Як і з `init`, коли ви використовуєте `realloc`, ви повинні включити `system_program` як один з облікових записів у структурі перевірки облікових записів.

Нижче наведено приклад перерозподілу простору для облікового запису, який зберігає поле `data` типу `String`.

```rust
#[derive(Accounts)]
#[instruction(instruction_data: String)]
pub struct ReallocExample<'info> {
    #[account(
        mut,
        seeds = [b"example_seed", user.key().as_ref()],
        bump,
        realloc = 8 + 4 + instruction_data.len(),
        realloc::payer = user,
        realloc::zero = false,
    )]
    pub pda_account: Account<'info, AccountType>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct AccountType {
    pub data: String,
}
```

Зверніть увагу, що `realloc` встановлено як `8 + 4 + instruction_data.len()`. Це розбивається наступним чином:
- `8` відведено для дискримінатора облікового запису
- `4` відведено на 4 байти простору, який BORSH використовує для зберігання довжини рядка
- `instruction_data.len()` - це довжина самого рядка

Якщо зміна довжини даних облікового запису є додатньою, лампорти будуть перенесені з `realloc::payer` на обліковий запис, щоб забезпечити звільнення від оренди. Так само, якщо зміна є від'ємною, лампорти будуть перенесені з облікового запису назад на `realloc::payer`.

Обмеження `realloc::zero` необхідне для визначення того, чи нова пам'ять повинна бути ініціалізована нулем після перерозподілу. Це обмеження слід встановити в `true` у випадках, коли ви очікуєте, що пам'ять облікового запису буде звужуватися та розширюватися кілька разів. Це дозволить обнулити простір, який в іншому випадку може показатися як застарілі дані.

## Close

Обмеження `close` надає простий та безпечний спосіб закрити існуючий обліковий запис.

Обмеження `close` позначає обліковий запис як закритий в кінці виконання інструкції, встановлюючи його дискримінатор на `CLOSED_ACCOUNT_DISCRIMINATOR` та відправляючи його лампорти на вказаний обліковий запис. Встановлення спеціального варіанту дискримінатора робить атаки на відновлення облікового запису (де наступна інструкція знову додає лампорти звільнення від оренди) неможливими. Якщо хтось спробує знову ініціалізувати обліковий запис, перевірка дискримінатора буде не вдалою, і програма вважатиме ініціалізацію недійсною.

У наведеному нижче прикладі використовується обмеження `close`, щоб закрити `data_account` та відправити лампорти, виділені на оренду, на обліковий запис отримувача.

```rust
pub fn close(ctx: Context<Close>) -> Result<()> {
    Ok(())
}

#[derive(Accounts)]
pub struct Close<'info> {
    #[account(mut, close = receiver)]
    pub data_account: Account<'info, AccountType>,
    #[account(mut)]
    pub receiver: Signer<'info>
}
```

# Лабораторна робота

Давайте пропрацюємо концепції, які ми розглянули в цьому уроці, створивши програму для оглядів фільмів за допомогою фреймворку Anchor.

Ця програма дозволить користувачам:

- Використовувати PDA для ініціалізації нового облікового запису для збереження відгуку
- Оновлювати вміст існуючого облікового запису 
- Закривати існуючий обліковий запис 

### 1. Створіть новий проект Anchor

Для початку створіть новий проект за допомогою команди `anchor init`.

```console
anchor init anchor-movie-review-program
```

Далі перейдіть до файлу `lib.rs` в папці `programs`, і ви побачите наступний початковий код.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod anchor_movie_review_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

Продовжуйте і видаліть інструкцію `initialize` та тип `Initialize`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod anchor_movie_review_program {
    use super::*;

}
```

### 2. `MovieAccountState`

Спочатку використаємо макрос атрибуту `#[account]`, щоб визначити `MovieAccountState`, який буде представляти структуру даних облікових записів. Нагадую, що макрос атрибуту `#[account]` реалізує різні траєкторії, які допомагають з серіалізацією та десеріалізацією облікового запису, встановлюють дискримінатор для облікового запису та встановлюють власника нового облікового запису як ідентифікатор програми, визначений у макросі `declare_id!`.

У кожному обліковому записі ми будемо зберігати:

- `reviewer` - користувач, який створює огляд
- `rating` - рейтинг фільму
- `title` - назва фільму
- `description` - зміст огляду

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod anchor_movie_review_program {
    use super::*;

}

#[account]
pub struct MovieAccountState {
    pub reviewer: Pubkey,    // 32
    pub rating: u8,          // 1
    pub title: String,       // 4 + len()
    pub description: String, // 4 + len()
}
```

### 3. Додайте огляд фільму

Далі реалізуємо інструкцію `add_movie_review`. Ця інструкція буде вимагати `Context` типу `AddMovieReview`, який ми реалізуємо незабаром.

Інструкція буде вимагати ще три додаткові аргументи як дані інструкції, надані оглядачем:

- `title` - назва фільму як `String`
- `description` - деталі огляду як `String`
- `rating` - рейтинг фільму як `u8`

У логіці інструкції ми заповнимо дані нового облікового запису `movie_review` даними інструкції. Ми також встановимо поле `reviewer` як обліковий запис `initializer` з контексту інструкції.

```rust
#[program]
pub mod movie_review{
    use super::*;

    pub fn add_movie_review(
        ctx: Context<AddMovieReview>,
        title: String,
        description: String,
        rating: u8,
    ) -> Result<()> {
        msg!("Movie Review Account Created");
        msg!("Title: {}", title);
        msg!("Description: {}", description);
        msg!("Rating: {}", rating);

        let movie_review = &mut ctx.accounts.movie_review;
        movie_review.reviewer = ctx.accounts.initializer.key();
        movie_review.title = title;
        movie_review.rating = rating;
        movie_review.description = description;
        Ok(())
    }
}
```

Далі створіть структуру `AddMovieReview`, яку ми використовували як загальний параметр у контексті інструкції. Ця структура буде містити перелік облікових записів, які потрібні для інструкції `add_movie_review`.

Пам'ятайте, вам знадобляться наступні макроси:

- Макрос атрибуту `#[derive(Accounts)]` використовується для десеріалізації та перевірки списку облікових записів, вказаних у межах структури.
- Макрос атрибуту `#[instruction(...)]` використовується для доступу до даних інструкції, переданих у межах інструкції.
- Макрос атрибуту `#[account(...)]` потім вказує додаткові обмеження для облікових записів.

Обліковий запис `movie_review` є PDA, який потрібно ініціалізувати, тому ми додамо обмеження `seeds` та `bump`, а також обмеження `init` з необхідними обмеженнями `payer` та `space`.

Для PDA seeds ми використаємо назву фільму та публічний ключ оглядача. Платником для ініціалізації повинен бути оглядач, а простір, виділений для облікового запису, повинен бути достатнім для дискримінатора облікового запису, публічного ключа оглядача та рейтингу, назви та опису огляду фільму.

```rust
#[derive(Accounts)]
#[instruction(title:String, description:String)]
pub struct AddMovieReview<'info> {
    #[account(
        init,
        seeds = [title.as_bytes(), initializer.key().as_ref()],
        bump,
        payer = initializer,
        space = 8 + 32 + 1 + 4 + title.len() + 4 + description.len()
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### 4. Редагування огляду фільму

Далі давайте реалізуємо інструкцію `update_movie_review` з контекстом, чий загальний тип - `UpdateMovieReview`.

Так само, як і раніше, інструкція буде потребувати ще три додаткові аргументи як дані інструкції, надані оглядачем:

- `title` - назва фільму
- `description` - деталі огляду
- `rating` - рейтинг фільму

У логіці інструкції ми оновимо `rating` та `description`, збережені в обліковому записі `movie_review`.

Хоча `title` не використовується у функції інструкції безпосередньо, він нам знадобиться для перевірки облікового запису `movie_review` в наступному кроці.

```rust
#[program]
pub mod anchor_movie_review_program {
    use super::*;

		...

    pub fn update_movie_review(
        ctx: Context<UpdateMovieReview>,
        title: String,
        description: String,
        rating: u8,
    ) -> Result<()> {
        msg!("Movie review account space reallocated");
        msg!("Title: {}", title);
        msg!("Description: {}", description);
        msg!("Rating: {}", rating);

        let movie_review = &mut ctx.accounts.movie_review;
        movie_review.rating = rating;
        movie_review.description = description;

        Ok(())
    }

}
```

Далі створіть структуру `UpdateMovieReview`, щоб визначити облікові записи, які потрібні для інструкції `update_movie_review`.

Оскільки обліковий запис `movie_review` буде вже ініціалізований на цьому етапі, нам вже не потрібно обмеження `init`. Однак, оскільки значення `description` може бути зараз іншим, нам потрібно використовувати обмеження `realloc` для перерозподілу простору на обліковому записі. При цьому нам потрібні обмеження `mut`, `realloc::payer` та `realloc::zero`.

Також нам все ще знадобляться обмеження `seeds` та `bump`, які ми використовували в `AddMovieReview`.

```rust
#[derive(Accounts)]
#[instruction(title:String, description:String)]
pub struct UpdateMovieReview<'info> {
    #[account(
        mut,
        seeds = [title.as_bytes(), initializer.key().as_ref()],
        bump,
        realloc = 8 + 32 + 1 + 4 + title.len() + 4 + description.len(),
        realloc::payer = initializer,
        realloc::zero = true,
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

Зверніть увагу, що обмеження `realloc` встановлене на новий простір, необхідне для облікового запису `movie_review` на основі оновленого значення `description`.

Додатково, обмеження `realloc::payer` вказує, що будь-які додаткові лампорти, необхідні або повернуті, будуть взяті з або надіслані на обліковий запис `initializer`.

Нарешті, ми встановлюємо обмеження `realloc::zero` на `true`, оскільки обліковий запис `movie_review` може бути оновлений декілька разів, або зменшуючи, або збільшуючи простір, виділений для облікового запису.

### 5. Видалити огляд фільму

Нарешті, давайте реалізуємо інструкцію `delete_movie_review`, щоб закрити існуючий обліковий запис `movie_review`.

Ми використовуватимемо контекст, загальний тип якого - `DeleteMovieReview`, і не будемо включати жодних додаткових даних інструкції. Оскільки ми лише закриваємо обліковий запис, насправді нам не потрібна жодна логіка інструкції всередині тіла функції. Закриття буде виконане обмеженням Anchor в типі `DeleteMovieReview`.

```rust
#[program]
pub mod anchor_movie_review_program {
    use super::*;

		...

    pub fn delete_movie_review(_ctx: Context<DeleteMovieReview>, title: String) -> Result<()> {
        msg!("Movie review for {} deleted", title);
        Ok(())
    }

}
```

Далі давайте реалізуємо структуру `DeleteMovieReview`.

```rust
#[derive(Accounts)]
#[instruction(title: String)]
pub struct DeleteMovieReview<'info> {
    #[account(
        mut,
        seeds=[title.as_bytes(), initializer.key().as_ref()],
        bump,
        close=initializer
    )]
    pub movie_review: Account<'info, MovieAccountState>,
    #[account(mut)]
    pub initializer: Signer<'info>,
    pub system_program: Program<'info, System>
}
```

Тут ми використовуємо обмеження `close`, щоб вказати, що ми закриваємо обліковий запис `movie_review` і що оренда повинна бути повернена на обліковий запис `initializer`. Ми також включаємо обмеження `seeds` та `bump` для облікового запису `movie_review` для валідації. Anchor потім обробляє додаткову логіку, необхідну для безпечного закриття облікового запису.

### 6. Тестування 

Програма повинна бути готова до використання! Тепер давайте протестуємо її. Перейдіть до `anchor-movie-review-program.ts` і замініть тестовий код за замовчуванням на наступний.

Тут ми:

- Створюємо значення за замовчуванням для даних інструкції огляду фільму
- Виводимо PDA облікового запису 
- Створюємо заповнювачі для тестів

```typescript
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { assert, expect } from "chai"
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

  const [moviePda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from(movie.title), provider.wallet.publicKey.toBuffer()],
    program.programId
  )

  it("Movie review is added`", async () => {})

  it("Movie review is updated`", async () => {})

  it("Deletes a movie review", async () => {})
})
```

Наступним кроком давайте проведемо перший тест інструкції `addMovieReview`. Зверніть увагу, що ми не додаємо `.accounts`. Це тому, що `Wallet` з `AnchorProvider` автоматично включений як підписувач, Anchor може виводити певні облікові записи, такі як `SystemProgram`, і Anchor також може виводити PDA облікового запису `movieReview` з аргументу `title` і публічного ключа підписувача.

Після виконання інструкції ми отримаємо обліковий запис `movieReview` і перевіряємо, чи дані, збережені на обліковому записі, відповідають очікуваним значенням.

```typescript
it("Movie review is added`", async () => {
  // Add your test here.
  const tx = await program.methods
    .addMovieReview(movie.title, movie.description, movie.rating)
    .rpc()

  const account = await program.account.movieAccountState.fetch(moviePda)
  expect(movie.title === account.title)
  expect(movie.rating === account.rating)
  expect(movie.description === account.description)
  expect(account.reviewer === provider.wallet.publicKey)
})
```

Далі давайте створимо тест для інструкції `updateMovieReview`, використовуючи той же процес, що й раніше.

```typescript
it("Movie review is updated`", async () => {
  const newDescription = "Wow this is new"
  const newRating = 4

  const tx = await program.methods
    .updateMovieReview(movie.title, newDescription, newRating)
    .rpc()

  const account = await program.account.movieAccountState.fetch(moviePda)
  expect(movie.title === account.title)
  expect(newRating === account.rating)
  expect(newDescription === account.description)
  expect(account.reviewer === provider.wallet.publicKey)
})
```

Наступним кроком створимо тест для інструкції `deleteMovieReview`.

```typescript
it("Deletes a movie review", async () => {
  const tx = await program.methods
    .deleteMovieReview(movie.title)
    .rpc()
})
```

Врешті-решт, виконайте команду `anchor test`, і ви повинні побачити такий вивід у консолі.

```console
  anchor-movie-review-program
    ✔ Movie review is added` (139ms)
    ✔ Movie review is updated` (404ms)
    ✔ Deletes a movie review (403ms)


  3 passing (950ms)
```

Якщо вам потрібно більше часу, щоб зрозуміти ці концепції, не соромтеся переглянути [код рішення](https://github.com/Unboxed-Software/anchor-movie-review-program/tree/solution-pdas) перед продовженням.

# Виклик

Тепер ваша черга створити щось самостійно. Озброєні концепціями, введеними у цьому уроці, спробуйте відтворити програму "Student Intro", яку ми використовували раніше, використовуючи фреймворк Anchor.

Програма "Student Intro" - це програма Solana, яка дозволяє студентам представити себе. Програма приймає ім'я користувача та коротке повідомлення як дані інструкції та створює обліковий запис для зберігання цих даних ончейн.

Використовуючи те, що ви вивчили в цьому уроці, розширте цю програму. Програма повинна містити інструкції для:

1. Ініціалізації облікового запису PDA для кожного студента, який зберігає ім'я студента та їх коротке повідомлення.
2. Оновлення повідомлення на існуючому обліковому записі.
3. Закриття існуючого облікового запису.


Спробуйте зробити це самостійно, якщо можете! Але якщо ви застрягнете, не соромтеся посилатися на [код рішення](https://github.com/Unboxed-Software/anchor-student-intro-program).


## Завершили лабораторну? 

Завантажте свій код на GitHub і [розкажіть нам, що ви думали про цей урок.](https://form.typeform.com/to/IPH0UGz7#answers-lesson=f58108e9-94a0-45b2-b0d5-44ada1909105)!
