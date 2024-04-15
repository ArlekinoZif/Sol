---
заголовок: Відповідність даних облікового запису
мета:
- Пояснити ризики безпеки, пов'язані з відсутністю перевірок валідації даних
- Реалізувати перевірки валідації даних за допомогою довгого long-form Rust
- Реалізувати перевірки валідації даних за допомогою обмежень Anchor
---

# Стислий виклад

- Використовуйте **data validation checks**, щоб перевірити, що дані облікового запису відповідають очікуваному значенню. Без відповідних перевірок валідації даних можуть, в інструкції бути використані неочікувані облікові записи.
- Для реалізації перевірок валідації даних в Rust достатньо порівняти дані, збережені на обліковому записі, з очікуваним значенням.

    ```rust
    if ctx.accounts.user.key() != ctx.accounts.user_data.user {
        return Err(ProgramError::InvalidAccountData.into());
    }
    ```
    
- В Anchor ви можете використовувати `constraint` для перевірки того, чи вираз, заданий вами, є істинним. З альтернативи ви можете використовувати `has_one` для перевірки того, що поле цільового облікового запису, збережене на обліковому записі, відповідає ключу облікового запису в структурі `Accounts`.

# Огляд

Account data matching refers to data validation checks used to verify the data stored on an account matches an expected value. Data validation checks provide a way to include additional constraints to ensure the appropriate accounts are passed into an instruction. 

This can be useful when accounts required by an instruction have dependencies on values stored in other accounts or if an instruction is dependent on the data stored in an account.

### Missing data validation check

The example below includes an `update_admin` instruction that updates the `admin` field stored on an `admin_config` account. 

The instruction is missing a data validation check to verify the `admin` account signing the transaction matches the `admin` stored on the `admin_config` account. This means any account signing the transaction and passed into the instruction as the `admin` account can update the `admin_config` account.


Відповідність даних облікового запису вказує на перевірки валідації даних, які використовуються для перевірки того, чи дані, збережені на обліковому записі, відповідають очікуваному значенню. Перевірки валідації даних надають можливість включити додаткові обмеження, щоб забезпечити передачу відповідних облікових записів в інструкцію.

Це може бути корисно, коли облікові записи, які необхідні для виконання інструкції, залежать від значень, що зберігаються в інших облікових записах, або якщо виконання інструкції залежить від даних, які зберігаються в обліковому записі.

### Відсутність валідації даних

У наведеному нижче прикладі є інструкція `update_admin`, яка оновлює поле `admin`, збережене на обліковому записі `admin_config`.

В інструкції відсутня валідація даних для перевірки того, що обліковий запис `admin`, який підписує транзакцію, відповідає `admin`, збереженому на обліковому записі `admin_config`. Це означає, що будь-який обліковий запис, який підписує транзакцію і передається в інструкцію як обліковий запис `admin`, може оновлювати обліковий запис `admin_config`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### Додати перевірку валідації даних

Основний підхід у Rust для вирішення цієї проблеми - просто порівняти переданий ключ `admin` з ключем `admin`, збереженим на обліковому записі `admin_config`, і генерувати помилку, якщо вони не відповідають одне одному.

```rust
if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
    return Err(ProgramError::InvalidAccountData.into());
}
```

Додавши валідацію даних, інструкція `update_admin` буде виконуватися лише в тому випадку, якщо підписник `admin` транзакції відповідає `admin`, збереженому на обліковому записі `admin_config`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
      if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
            return Err(ProgramError::InvalidAccountData.into());
        }
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(mut)]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

### Використайте обмеження Anchor

Anchor спрощує, використовуючи обмеження `has_one`. Ви можете використовувати обмеження `has_one`, щоб перемістити валідацію даних з логіки інструкції в структуру `UpdateAdmin`.

У наведеному нижче прикладі `has_one = admin` вказує, що обліковий запис `admin`, який підписує транзакцію, повинен відповідати полю `admin`, збереженому на обліковому записі `admin_config`. Для використання обмеження `has_one` конвенція найменування поля даних на обліковому записі повинна бути узгодженою з конвенцією найменування на обліковому записі структури валідації.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod data_validation {
    use super::*;
    ...
    pub fn update_admin(ctx: Context<UpdateAdmin>) -> Result<()> {
        ctx.accounts.admin_config.admin = ctx.accounts.new_admin.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        has_one = admin
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}

#[account]
pub struct AdminConfig {
    admin: Pubkey,
}
```

Замість цього ви можете використовувати `constraint`, щоб вручну додати вираз, який повинен взяти участь в процесі виконання, із виразу true, щоб виконання могло продовжитися. Це корисно, коли, з якої-небудь причини, неможливо забезпечити послідовність імен, або коли для повної перевірки вхідних даних вам потрібен більш складний вираз.

```rust
#[derive(Accounts)]
pub struct UpdateAdmin<'info> {
    #[account(
        mut,
        constraint = admin_config.admin == admin.key()
    )]
    pub admin_config: Account<'info, AdminConfig>,
    #[account(mut)]
    pub admin: Signer<'info>,
    pub new_admin: SystemAccount<'info>,
}
```

# Лабораторна робота

У цьому лабораторному завданні ми створимо просту програму "сховище" (“vault”) схожу на ту, яку ми використовували в уроках "Авторизація підписувача" і "Перевірка власника" (Authorization lesson and the Owner Check lesson). Подібно до цих лабораторних робіт, ми покажемо в цьому завданні, як відсутність перевірки валідації даних може призвести до втратити коштів зі сховища.

### 1. Стартовий код

Для початку завантажте стартовий код з гілки `starter` з [цього репозиторію](https://github.com/Unboxed-Software/solana-account-data-matching). Стартовий код включає програму з двома інструкціями та заготовкою для файлу тестів.

Інструкція `initialize_vault` ініціалізує новий обліковий запис `Vault` та новий обліковий запис `TokenAccount`. Обліковий запис `Vault` буде зберігати адресу облікового запису токена, авторитет сховища та обліковий запис для виводу.

Авторитет нового облікового запису токена буде встановлено як `vault`, PDA програми. Це дозволяє обліковому запису `vault` підписувати трансфери токенів з облікового запису токена.

Інструкція `insecure_withdraw` перекладає всі токени в обліковому записі токена `vault` до облікового запису токена `withdraw_destination`. 

Зверніть увагу, що ця інструкція має перевірку підпису для `authority` та перевірку власника для `vault`. Однак ніде в логіці перевірки облікового запису чи логіці інструкції немає коду, який перевіряє, що обліковий запис `authority`, переданий в інструкцію, відповідає обліковому запису `authority` в `vault`.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;

    pub fn initialize_vault(ctx: Context<InitializeVault>) -> Result<()> {
        ctx.accounts.vault.token_account = ctx.accounts.token_account.key();
        ctx.accounts.vault.authority = ctx.accounts.authority.key();
        ctx.accounts.vault.withdraw_destination = ctx.accounts.withdraw_destination.key();
        Ok(())
    }

    pub fn insecure_withdraw(ctx: Context<InsecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeVault<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 32 + 32 + 32,
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        init,
        payer = authority,
        token::mint = mint,
        token::authority = vault,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct InsecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}

#[account]
pub struct Vault {
    token_account: Pubkey,
    authority: Pubkey,
    withdraw_destination: Pubkey,
}
```

### 2. Тест інструкції `insecure_withdraw` 

Щоб довести, що це проблема, давайте напишемо тест, де обліковий запис, який не є `authority` сховища, намагається зняти кошти з сховища.

У тестовому файлі вже є код для виклику інструкції `initialize_vault`, використовуючи гаманець постачальника як `authority`, а потім надсилає 100 токенів на рахунок `vault` токена.

Додайте тест для виклику інструкції `insecure_withdraw`. Використовуйте `withdrawDestinationFake` як рахунок `withdrawDestination`, а `walletFake` як `authority`. Потім відправте транзакцію, використовуючи `walletFake`.

Оскільки немає перевірок, які б визначали, що обліковий запис `authority`, переданий в інструкцію, відповідає значенням, збереженим на сховищі, інструкція буде успішно оброблена, і токени будуть передані на рахунок `withdrawDestinationFake`.

```tsx
describe("account-data-matching", () => {
  ...
  it("Insecure withdraw", async () => {
    const tx = await program.methods
      .insecureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestinationFake,
        authority: walletFake.publicKey,
      })
      .transaction()

    await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Запустіть `anchor test`, щоб переконатися, що обидві транзакції завершаться успішно.

```bash
account-data-matching
  ✔ Initialize Vault (811ms)
  ✔ Insecure withdraw (403ms)
```

### 3. Додайте інструкцію `secure_withdraw` 

Давайте реалізуємо безпечну версію цієї інструкції під назвою `secure_withdraw`.

Ця інструкція буде ідентичною інструкції `insecure_withdraw`, за винятком того, що ми використаємо обмеження `has_one` в структурі перевірки облікових записів (`SecureWithdraw`), щоб перевірити, що обліковий запис `authority`, переданий до інструкції, відповідає обліковому запису `authority` на обліковому записі `vault`. Таким чином, тільки правильний обліковий запис може знімати кошти з хранилища.

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod account_data_matching {
    use super::*;
    ...
    pub fn secure_withdraw(ctx: Context<SecureWithdraw>) -> Result<()> {
        let amount = ctx.accounts.token_account.amount;

        let seeds = &[b"vault".as_ref(), &[*ctx.bumps.get("vault").unwrap()]];
        let signer = [&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            token::Transfer {
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.withdraw_destination.to_account_info(),
            },
            &signer,
        );

        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SecureWithdraw<'info> {
    #[account(
        seeds = [b"vault"],
        bump,
        has_one = token_account,
        has_one = authority,
        has_one = withdraw_destination,

    )]
    pub vault: Account<'info, Vault>,
    #[account(
        mut,
        seeds = [b"token"],
        bump,
    )]
    pub token_account: Account<'info, TokenAccount>,
    #[account(mut)]
    pub withdraw_destination: Account<'info, TokenAccount>,
    pub token_program: Program<'info, Token>,
    pub authority: Signer<'info>,
}
```

### 4. Тест інструкції `secure_withdraw` 

Тепер давайте протестуємо інструкцію `secure_withdraw` двома тестами: один, який використовує `walletFake` як підтвердження, і один, який використовує `wallet` як підтвердження. Ми очікуємо, що перший виклик поверне помилку, а другий буде успішним.

```tsx
describe("account-data-matching", () => {
  ...
  it("Secure withdraw, expect error", async () => {
    try {
      const tx = await program.methods
        .secureWithdraw()
        .accounts({
          vault: vaultPDA,
          tokenAccount: tokenPDA,
          withdrawDestination: withdrawDestinationFake,
          authority: walletFake.publicKey,
        })
        .transaction()

      await anchor.web3.sendAndConfirmTransaction(connection, tx, [walletFake])
    } catch (err) {
      expect(err)
      console.log(err)
    }
  })

  it("Secure withdraw", async () => {
    await spl.mintTo(
      connection,
      wallet.payer,
      mint,
      tokenPDA,
      wallet.payer,
      100
    )

    await program.methods
      .secureWithdraw()
      .accounts({
        vault: vaultPDA,
        tokenAccount: tokenPDA,
        withdrawDestination: withdrawDestination,
        authority: wallet.publicKey,
      })
      .rpc()

    const balance = await connection.getTokenAccountBalance(tokenPDA)
    expect(balance.value.uiAmount).to.eq(0)
  })
})
```

Виконайте команду `anchor test`, щоб переконатися, що транзакція, яка використовує невірний обліковий запис для підтвердження, тепер повертає помилку Anchor, тоді як транзакція, яка використовує правильні облікові записи, завершується успішно.

```bash
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: SecureWithdraw',
'Program log: AnchorError caused by account: vault. Error Code: ConstraintHasOne. Error Number: 2001. Error Message: A has one constraint was violated.',
'Program log: Left:',
'Program log: DfLZV18rD7wCQwjYvhTFwuvLh49WSbXFeJFPQb5czifH',
'Program log: Right:',
'Program log: 5ovvmG5ntwUC7uhNWfirjBHbZD96fwuXDMGXiyMwPg87',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 10401 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7d1'
```

Важливо відзначити, що Anchor вказує в логах обліковий запис, який спричинив помилку (`AnchorError caused by account: vault`).

```bash
✔ Secure withdraw, expect error (77ms)
✔ Secure withdraw (10073ms)
```

Тепер ви закрили цю уразливість в безпеці. Головна схожість більшості вразливостей в тому, що вони дуже прості. Втім, зі збільшенням масштабу і складності ваших програм таких потенційно вразливих місць буде все більше. Варто формувати звичку писати тести, які відправляють інструкції, які *не повинні* працювати. Чим більше, тим краще. Таким чином ви виявите проблеми до розгортання.

Якщо ви хочете переглянути кінцевий код, ви можете знайти його в гілці `solution` [репозиторію](https://github.com/Unboxed-Software/solana-account-data-matching/tree/solution).

# Виклик

Так само, як і з іншими уроками цього блоку, ви можете відпрацювати слабкі місця, перевіряючи свій власний код.

Виділіть собі час, щоб переглянути принаймні одну програму та переконайтеся, що встановлено належні перевірки даних для уникнення зловживань з боку безпеки.

Пам'ятайте, якщо ви знаходите помилку чи слабке місце в програмі когось іншого, будь ласка, попередьте його! Якщо ви знаходите їх у своїй власній програмі, не забудьте виправити їх одразу.

## Завершили лабораторну роботу?

Вивантажте свій код на GitHub і [розкажіть нам, що ви думаєте про цей урок](https://form.typeform.com/to/IPH0UGz7#answers-lesson=a107787e-ad33-42bb-96b3-0592efc1b92f)!
