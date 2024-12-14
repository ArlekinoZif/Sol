---
заголовок: Дублікати змінних акаунтів  
цілі: 
- Пояснити ризики безпеки, пов'язані з інструкціями, які вимагають два змінні акаунти одного типу, та пояснити способи їх уникнення  
- Реалізувати перевірку на дублікати змінних акаунтів, використовуючи повний запис на Rust  
- Реалізувати перевірку на дублікати змінних акаунтів за допомогою обмежень Anchor  
---

# Стислий огляд

- Коли інструкція вимагає два змінні акаунти одного типу, зловмисник може передати один і той самий акаунт двічі, що може призвести до його неочікуваної модифікації.  
- Щоб перевірити наявність дублікатів змінних акаунтів у Rust, достатньо порівняти публічні ключі обох акаунтів та викликати помилку, якщо вони збігаються.  

  ```rust
  if ctx.accounts.account_one.key() == ctx.accounts.account_two.key() {
      return Err(ProgramError::InvalidArgument)
  }
  ```

- У Anchor можна використовувати `constraint`, щоб додати до акаунта явне обмеження, яке перевіряє, що він не є таким самим, як інший акаунт.

# Урок

Дублювання змінюваних акаунтів стосується інструкцій, які вимагають два змінювані акаунти одного типу. У таких випадках необхідно перевірити, що ці два акаунти є різними, щоб запобігти передачі одного й того ж акаунта двічі. 

Оскільки програма розглядає кожен акаунт як окремий, передача одного акаунта двічі може призвести до непередбачених змін у другому акаунті. Це може викликати як незначні проблеми, так і катастрофічні наслідки — усе залежить від того, які дані змінює код і як ці акаунти використовуються. У будь-якому разі, це вразливість, про яку має знати кожен розробник.

### Без перевірки

Наприклад, уявіть програму, яка оновлює поле `data` для акаунтів `user_a` та `user_b` в межах однієї інструкції. Значення, яке встановлюється для `user_a`, відрізняється від значення для `user_b`. Без перевірки, що `user_a` та `user_b` є різними акаунтами, програма оновить поле `data` акаунта `user_a`, а потім змінить його вдруге, використовуючи інше значення, припускаючи, що `user_b` — це окремий акаунт.

Ви можете побачити цей приклад у коді нижче. У ньому відсутня перевірка, чи `user_a` і `user_b` є різними акаунтами. Передача одного і того ж акаунта як `user_a` та `user_b` призведе до того, що поле `data` для цього акаунта буде позначено `b`, навіть якщо метою було встановити значення `a` і `b` для окремих акаунтів. Залежно від того, що представляє поле `data`, це може бути незначним ненавмисним побічним ефектом або серйозною проблемою безпеки. Дозвіл на те, щоб `user_a` і `user_b` були одним і тим самим акаунтом, може призвести до:  

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts_insecure {
    use super::*;

    pub fn update(ctx: Context<Update>, a: u64, b: u64) -> Result<()> {
        let user_a = &mut ctx.accounts.user_a;
        let user_b = &mut ctx.accounts.user_b;

        user_a.data = a;
        user_b.data = b;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Update<'info> {
    user_a: Account<'info, User>,
    user_b: Account<'info, User>,
}

#[account]
pub struct User {
    data: u64,
}
```

### Додайте перевірку в інструкцію  

Щоб виправити цю проблему в Rust, просто додайте перевірку в логіку інструкції, яка перевіряє, чи публічний ключ `user_a` не збігається з публічним ключем `user_b`. Якщо вони однакові, повернеться помилка.

```rust
if ctx.accounts.user_a.key() == ctx.accounts.user_b.key() {
    return Err(ProgramError::InvalidArgument)
}
```

Ця перевірка гарантує, що `user_a` та `user_b` не є однаковими акаунтами.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts_secure {
    use super::*;

    pub fn update(ctx: Context<Update>, a: u64, b: u64) -> Result<()> {
        if ctx.accounts.user_a.key() == ctx.accounts.user_b.key() {
            return Err(ProgramError::InvalidArgument.into())
        }
        let user_a = &mut ctx.accounts.user_a;
        let user_b = &mut ctx.accounts.user_b;

        user_a.data = a;
        user_b.data = b;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Update<'info> {
    user_a: Account<'info, User>,
    user_b: Account<'info, User>,
}

#[account]
pub struct User {
    data: u64,
}
```

### Використання Anchor `constraint`

Якщо ви використовуєте Anchor, ще кращим рішенням є додавання перевірки до структури валідації акаунтів замість логіки інструкції.

Ви можете використати атрибут `#[account(..)]` та ключове слово `constraint`, щоб додати умову до акаунта. Ключове слово `constraint` перевіряє вираз, який слідує після нього. Є варіанти: true або false, і якщо вираз оцінюється як false, повертається помилка.

У наступному прикладі перевірка переноситься з логіки інструкції до структури валідації акаунтів шляхом додавання `constraint` до атрибуту `#[account(..)]`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts_recommended {
    use super::*;

    pub fn update(ctx: Context<Update>, a: u64, b: u64) -> Result<()> {
        let user_a = &mut ctx.accounts.user_a;
        let user_b = &mut ctx.accounts.user_b;

        user_a.data = a;
        user_b.data = b;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Update<'info> {
    #[account(constraint = user_a.key() != user_b.key())]
    user_a: Account<'info, User>,
    user_b: Account<'info, User>,
}

#[account]
pub struct User {
    data: u64,
}
```

# Лабораторна робота

Давайте потренуємося, створивши просту програму "Камінь, Ножиці, Папір", щоб продемонструвати, як відсутність перевірки на дубльовані змінювані акаунти може призвести до непередбачуваної поведінки в програмі.

Ця програма ініціалізує акаунти "гравців" та матиме окрему інструкцію, яка потребує два акаунти гравців для початку гри в "Камінь, Ножиці, Папір".

- Інструкція `initialize` для ініціалізації акаунта `PlayerState`.  
- Інструкція `rock_paper_scissors_shoot_insecure`, яка потребує два акаунти `PlayerState`, але не перевіряє, чи є передані акаунти різними.  
- Інструкція `rock_paper_scissors_shoot_secure`, яка працює аналогічно до `rock_paper_scissors_shoot_insecure`, але додає обмеження, яке гарантує, що два акаунти гравців є різними.

### 1. Початок  

Щоб розпочати, завантажте початковий код із гілки `starter` цього [репозиторію](https://github.com/unboxed-software/solana-duplicate-mutable-accounts/tree/starter). Початковий код містить програму з двома інструкціями та базову структуру для файлу тестування.  

Інструкція `initialize` ініціалізує новий акаунт `PlayerState`, який зберігає публічний ключ гравця та поле `choice`, що встановлено у значення `None`.  

Інструкція `rock_paper_scissors_shoot_insecure` потребує два акаунти `PlayerState` та вибір із переліку `RockPaperScissors` для кожного гравця, але не перевіряє, чи є передані акаунти різними. Це означає, що один і той самий акаунт може використовуватися для обох `PlayerState` акаунтів в інструкції.

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        ctx.accounts.new_player.player = ctx.accounts.payer.key();
        ctx.accounts.new_player.choice = None;
        Ok(())
    }

    pub fn rock_paper_scissors_shoot_insecure(
        ctx: Context<RockPaperScissorsInsecure>,
        player_one_choice: RockPaperScissors,
        player_two_choice: RockPaperScissors,
    ) -> Result<()> {
        ctx.accounts.player_one.choice = Some(player_one_choice);

        ctx.accounts.player_two.choice = Some(player_two_choice);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + 32 + 8
    )]
    pub new_player: Account<'info, PlayerState>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct RockPaperScissorsInsecure<'info> {
    #[account(mut)]
    pub player_one: Account<'info, PlayerState>,
    #[account(mut)]
    pub player_two: Account<'info, PlayerState>,
}

#[account]
pub struct PlayerState {
    player: Pubkey,
    choice: Option<RockPaperScissors>,
}

#[derive(Clone, Copy, BorshDeserialize, BorshSerialize)]
pub enum RockPaperScissors {
    Rock,
    Paper,
    Scissors,
}
```

### 2. Тестування інструкції `rock_paper_scissors_shoot_insecure`  

Файл тестування включає код для виклику інструкції `initialize` двічі, щоб створити два акаунти гравців.  

Додайте тест для виклику інструкції `rock_paper_scissors_shoot_insecure`, передаючи `playerOne.publicKey` як для `playerOne`, так і для `playerTwo`.

```typescript
describe("duplicate-mutable-accounts", () => {
	...
	it("Invoke insecure instruction", async () => {
        await program.methods
        .rockPaperScissorsShootInsecure({ rock: {} }, { scissors: {} })
        .accounts({
            playerOne: playerOne.publicKey,
            playerTwo: playerOne.publicKey,
        })
        .rpc()

        const p1 = await program.account.playerState.fetch(playerOne.publicKey)
        assert.equal(JSON.stringify(p1.choice), JSON.stringify({ scissors: {} }))
        assert.notEqual(JSON.stringify(p1.choice), JSON.stringify({ rock: {} }))
    })
})
```

Запустіть команду `anchor test`, щоб переконатися, що транзакція успішно завершується, навіть якщо один і той самий акаунт використовується як два різні в інструкції. Оскільки акаунт `playerOne` використовується в інстукції як для `playerOne`, так і для `playerTwo`, зверніть увагу, що поле `choice`, яке зберігається в акаунті `playerOne`, буде перезаписане та встановлене неправильно, наприклад, як `scissors`.

```bash
duplicate-mutable-accounts
  ✔ Initialized Player One (461ms)
  ✔ Initialized Player Two (404ms)
  ✔ Invoke insecure instruction (406ms)
```

Дозволяти дублікати облікових записів не тільки не має сенсу для гри, але й викликає непередбачувану поведінку. Якщо продовжувати розвивати цю програму, вона матиме лише один вибраний варіант і не зможе порівнювати його з другим. У результаті гра кожного разу закінчуватиметься внічию. Також для користувача не зрозуміло, чи вибір `playerOne` має бути "камінь" чи "ножиці", тому поведінка програми виглядає дивною.

### 3. Додайте інструкцію `rock_paper_scissors_shoot_secure`

Далі поверніться до файлу `lib.rs` і додайте інструкцію `rock_paper_scissors_shoot_secure`, яка використовує макрос `#[account(...)]`, щоб додати додаткову умову `constraint`, яка перевіряє, що `player_one` та `player_two` є різними обліковими записами.

```rust
#[program]
pub mod duplicate_mutable_accounts {
    use super::*;
		...
        pub fn rock_paper_scissors_shoot_secure(
            ctx: Context<RockPaperScissorsSecure>,
            player_one_choice: RockPaperScissors,
            player_two_choice: RockPaperScissors,
        ) -> Result<()> {
            ctx.accounts.player_one.choice = Some(player_one_choice);

            ctx.accounts.player_two.choice = Some(player_two_choice);
            Ok(())
        }
}

#[derive(Accounts)]
pub struct RockPaperScissorsSecure<'info> {
    #[account(
        mut,
        constraint = player_one.key() != player_two.key()
    )]
    pub player_one: Account<'info, PlayerState>,
    #[account(mut)]
    pub player_two: Account<'info, PlayerState>,
}
```

### 7. Тестування інструкції `rock_paper_scissors_shoot_secure`

Щоб протестувати інструкцію `rock_paper_scissors_shoot_secure`, виконаємо її двічі. Спочатку виконаємо інструкцію, використовуючи два різні облікові записи гравців, щоб перевірити, чи працює вона належним чином. Потім виконаємо інструкцію, використовуючи `playerOne.publicKey` для обох облікових записів гравців, і очікуємо, що вона завершиться помилкою.

```typescript
describe("duplicate-mutable-accounts", () => {
	...
    it("Invoke secure instruction", async () => {
        await program.methods
        .rockPaperScissorsShootSecure({ rock: {} }, { scissors: {} })
        .accounts({
            playerOne: playerOne.publicKey,
            playerTwo: playerTwo.publicKey,
        })
        .rpc()

        const p1 = await program.account.playerState.fetch(playerOne.publicKey)
        const p2 = await program.account.playerState.fetch(playerTwo.publicKey)
        assert.equal(JSON.stringify(p1.choice), JSON.stringify({ rock: {} }))
        assert.equal(JSON.stringify(p2.choice), JSON.stringify({ scissors: {} }))
    })

    it("Invoke secure instruction - expect error", async () => {
        try {
        await program.methods
            .rockPaperScissorsShootSecure({ rock: {} }, { scissors: {} })
            .accounts({
                playerOne: playerOne.publicKey,
                playerTwo: playerOne.publicKey,
            })
            .rpc()
        } catch (err) {
            expect(err)
            console.log(err)
        }
    })
})
```

Запустіть команду `anchor test`, щоб переконатися, що інструкція працює належним чином, а використання облікового запису `playerOne` двічі повертає очікувану помилку.

```bash
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: RockPaperScissorsShootSecure',
'Program log: AnchorError caused by account: player_one. Error Code: ConstraintRaw. Error Number: 2003. Error Message: A raw constraint was violated.',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 5104 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7d3'
```

Просте обмеження — це все, що потрібно, щоб закрити цю вразливість. Хоча приклад і є дещо примітивним, але він ілюструє дивну поведінку, яка може виникнути, якщо ви пишете програму за припущенням, що два однакових типи акаунтів будуть різними екземплярами акаунтів, але не вказуєте це обмеження прямо у програмі. Завжди думайте про поведінку, яку ви очікуєте від програми, і чи є це чітко визначеним.

Якщо ви хочете ознайомитись із фінальним кодом рішення, ви можете знайти його в гілці `solution` [репозиторію](https://github.com/Unboxed-Software/solana-duplicate-mutable-accounts/tree/solution).

# Виклик

Як і з іншими уроками в цьому розділі, ваша можливість попрактикуватися в уникненні цієї вразливості безпеки полягає в аудиті власних або чужих програм.

Виділіть час для перевірки хоча б однієї програми і переконайтеся, що будь-які інструкції з двома однаковими типами змінних акаунтів належним чином обмежені, щоб уникнути їх дублювання.

Не забувайте, якщо ви знайдете помилку або вразливість у програмі іншої людини, обов'язково сповістіть їх! Якщо ж ви знайдете помилку у своїй програмі, негайно виправте її.

## Завершили лабораторну?

Завантажте свій код на GitHub і [поділіться своїми враженнями про цей урок](https://form.typeform.com/to/IPH0UGz7#answers-lesson=9b759e39-7a06-4694-ab6d-e3e7ac266ea7)!
