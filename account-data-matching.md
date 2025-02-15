---
назва: Відповідність даних облікового запису
завдання:
- Пояснити ризики безпеки, пов'язані з відсутністю перевірок даних
- Реалізувати перевірки даних за допомогою long-form Rust
- Реалізувати перевірки даних за допомогою обмежень Anchor
---

# Стислий виклад

- Використовуйте **data validation checks**, щоб перевірити, що дані облікового запису відповідають очікуваному значенню. Без належних перевірок даних у інструкції можуть використовуватися неочікувані акаунти.
- Для реалізації перевірок даних в Rust достатньо порівняти дані, збережені на обліковому записі, з очікуваним значенням.

    ```rust
    if ctx.accounts.user.key() != ctx.accounts.user_data.user {
        return Err(ProgramError::InvalidAccountData.into());
    }
    ```
    
- В Anchor ви можете використовувати `constraint` для перевірки того, чи вираз, заданий вами, є істинним. З альтернативи ви можете використовувати `has_one` для перевірки того, що поле цільового облікового запису, збережене на обліковому записі, відповідає ключу облікового запису в структурі `Accounts`.

# Урок

Відповідність даних акаунту означає перевірку даних, що використовується для підтвердження відповідності збережених на акаунті даних очікуваному значенню. Перевірки даних надають можливість додати додаткові обмеження, щоб гарантувати передавання відповідних акаунтів у інструкцію.

Це може бути корисним, коли акаунти, необхідні для інструкції, мають залежності від значень, збережених в інших акаунтах, або якщо інструкція залежить від даних, збережених в акаунті.

### Відсутня перевірка даних

У наведеному нижче прикладі міститься інструкція `update_admin`, яка оновлює поле `admin`, збережене в акаунті `admin_config`.  

В інструкції відсутня перевірка даних, яка б підтверджувала, що акаунт `admin`, який підписує транзакцію, відповідає `admin`, збереженому в акаунті `admin_config`. Це означає, що будь-який акаунт, який підписує транзакцію та передається в інструкцію як `admin`, може оновити акаунт `admin_config`.

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

### Додати перевірку даних

Основний підхід у Rust для вирішення цієї проблеми - просто порівняти переданий ключ `admin` з ключем `admin`, збереженим на обліковому записі `admin_config`, і генерувати помилку, якщо вони не відповідають одне одному.

```rust
if ctx.accounts.admin.key() != ctx.accounts.admin_config.admin {
    return Err(ProgramError::InvalidAccountData.into());
}
```

Додавши перевірку даних, інструкція `update_admin` виконуватиметься лише в тому випадку, якщо підписант `admin` у транзакції збігатиметься з `admin`, збереженим в акаунті `admin_config`.

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

Anchor спрощує це за допомогою обмеження `has_one`. Ви можете використати `has_one`, щоб перенести перевірку даних з логіки інструкції до структури `UpdateAdmin`.

У наведеному нижче прикладі `has_one = admin` вказує, що акаунт `admin`, який підписує транзакцію, повинен збігатися з полем `admin`, збереженим в акаунті `admin_config`. Щоб використати обмеження `has_one`, необхідно, щоб назва поля даних в акаунті відповідала назві у структурі перевірки акаунтів.

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

Альтернативно, ви можете використовувати `constraint`, щоб вручну додати вираз, який повинен бути встановлений на true, щоб виконання продовжилося. Це корисно, коли з якихось причин не можна забезпечити узгодженість іменування або коли потрібен більш складний вираз для повної перевірки вхідних даних.

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

У цій лабораторній роботі ми створимо просту програму "сховище" (“vault”) схожу на ту, яку ми використовували в уроках "Авторизація підписанта" і "Перевірка власника" (Authorization lesson and the Owner Check lesson). Подібно до цих лабораторних робіт, ми покажемо в цьому завданні, як відсутність перевірки даних може призвести до втрати коштів зі сховища.

### 1. Початковий код

Для початку завантажте стартовий код з гілки `starter` з [цього репозиторію](https://github.com/Unboxed-Software/solana-account-data-matching). Початковий код містить програму з двома інструкціями та шаблонним налаштуванням для тестового файлу.

Інструкція `initialize_vault` ініціалізує новий акаунт `Vault` і новий акаунт `TokenAccount`. Акаунт `Vault` зберігатиме адресу токен-акаунту, уповноважений акаунт сховища та акаунт для виведення токенів.

Уповноважений акаунт нового токен-акаунту буде встановлений як `vault`, PDA програми. Це дозволяє обліковому запису `vault` підписувати передачі токенів з токен-акаунту.

Інструкція `insecure_withdraw` переносить всі токени з токен-акаунту акаунту `vault` на токен-акаунт `withdraw_destination`.

Зверніть увагу, що ця інструкція ****має**** перевірку підписанта для `authority` і перевірку права власності для `vault`. Однак, ні в перевірці акаунтів, ні в логіці інструкції немає коду, який би перевіряв, чи збігається акаунт `authority`, переданий в інструкцію, з акаунтом `authority`, що зберігається в `vault`.

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

Файл тесту містить код для виклику інструкції `initialize_vault`, використовуючи гаманець провайдера як `authority`, а потім здійснюється мінт 100 токенів на токен-акаунт `vault`.

Додайте тест для виклику інструкції `insecure_withdraw`. Використовуйте `withdrawDestinationFake` як `withdrawDestination` акаунт та `walletFake` як `authority`. Потім відправте транзакцію, використовуючи `walletFake`.

Оскільки немає перевірок, що звірили б, чи збігається акаунт `authority`, переданий в інструкцію, зі значеннями, збереженими на акаунті `vault`, ініціалізованому в першому тесті, інструкція буде успішно оброблена, і токени будуть переведені на акаунт `withdrawDestinationFake`.

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

Ця інструкція буде ідентична інструкції `insecure_withdraw`, за винятком того, що ми використаємо обмеження `has_one` в структурі перевірки акаунтів (`SecureWithdraw`), щоб перевірити, чи збігається акаунт `authority`, переданий в інструкцію, з акаунтом `authority` на акаунті `vault`. Таким чином, лише правильний акаунт `authority` зможе вивести токени зі сховища.

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

Тепер давайте протестуємо інструкцію `secure_withdraw` з двома тестами: один використовує `walletFake` як уповноважений акаунт, а інший — `wallet` як уповноважений акаунт. Очікується, що перший виклик поверне помилку, а другий — успішно виконання.

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

Запустіть `anchor test`, щоб перевірити, що транзакція, яка використовує неправильний уповноважений акаунт, тепер повертає помилку Anchor Error, тоді як транзакція з правильними акаунтами завершується успішно.

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

Зверніть увагу, що Anchor вказує в логах акаунт, який спричинив помилку (`AnchorError caused by account: vault`).

```bash
✔ Secure withdraw, expect error (77ms)
✔ Secure withdraw (10073ms)
```

Тепер ви закрили цю уразливість в безпеці. Головна схожість більшості вразливостей в тому, що вони дуже прості. Втім, зі збільшенням масштабу і складності ваших програм таких потенційно вразливих місць буде все більше. Варто формувати звичку писати тести, які відправляють інструкції, які *не повинні* працювати. Чим більше, тим краще. Таким чином ви виявите проблеми до розгортання програми.

Якщо ви хочете переглянути кінцевий код, ви можете знайти його в гілці `solution` [репозиторію](https://github.com/Unboxed-Software/solana-account-data-matching/tree/solution).

# Завдання

Так само, як і з іншими уроками цього блоку, ви можете відпрацювати слабкі місця, перевіряючи свій власний код.

Виділіть собі час, щоб переглянути принаймні одну програму та переконайтеся, що встановлено належні перевірки даних для уникнення щоб уникнути вразливостей безпеки.

Пам'ятайте, якщо ви знаходите помилку чи слабке місце в програмі когось іншого, будь ласка, попередьте його! Якщо ви знаходите їх у своїй власній програмі, не забудьте одразу їх виправити.

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=a107787e-ad33-42bb-96b3-0592efc1b92f)!
