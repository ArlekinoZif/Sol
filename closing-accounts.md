---
заголовок: Закриття акаунтів та атаки відновлення  
мета:  
- Пояснити різні вразливості безпеки, пов’язані з неправильним закриттям програмних акаунтів  
- Закривати програмні акаунти безпечно та надійно за допомогою нативного Rust  
- Закривати програмні акаунти безпечно та надійно за допомогою обмеження Anchor `close`   
---

# Стислий виклад

- **Неправильне закриття облікового запису** створює можливість для атак реініціалізації/відродження
- Система виконання Solana **збирає сміття з облікових записів**, коли вони більше не звільняються від орендної плати. Закриття облікових записів передбачає перенесення лампортів, що зберігаються в обліковому записі для звільнення від орендної плати, до іншого облікового запису за вашим вибором.
- Ви можете скористатися обмеженням Anchor `#[account(close = <адреса_для_надсилання_лампортів>)]` для безпечного закриття облікових записів і встановити дискримінатор облікових записів у значення `CLOSED_ACCOUNT_DISCRIMINATOR`.

    ```rust
    #[account(mut, close = receiver)]
    pub data_account: Account<'info, MyData>,
    #[account(mut)]
    pub receiver: SystemAccount<'info>
    ```

# Урок

Хоча це звучить просто, правильне закриття акаунтів може бути непростим завданням. Існує кілька способів, як зловмисник може обійти процедуру закриття акаунта, якщо ви не виконаєте певні кроки.

Щоб краще зрозуміти ці вектори атак, давайте детально розглянемо кожен з цих сценаріїв.

## Небезпечне закриття облікового запису

По суті, закриття облікового запису передбачає перенесення його лампортів до окремого облікового запису, що призводить до того, що Solana починає збирати сміття з першого облікового запису. Це перезавантажує власника від програми-власника до системної програми.

Погляньте на приклад нижче. Інструкція вимагає двох облікових записів:

1. `account_to_close` - акаунт, який необхідно закрити
2. `destination` - акаунт, який має отримати лампорти закритого облікового запису

Логіка програми спрямована на закриття акаунта шляхом простого збільшення лампортів `destination` на суму, що зберігається в `account_to_close`, і встановлення лампортів `account_to_close` в 0. За допомогою цієї програми, після повної обробки транзакції, `account_to_close` буде сміттям, зібраним під час виконання.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod closing_accounts_insecure {
    use super::*;

    pub fn close(ctx: Context<Close>) -> ProgramResult {
        let dest_starting_lamports = ctx.accounts.destination.lamports();

        **ctx.accounts.destination.lamports.borrow_mut() = dest_starting_lamports
            .checked_add(ctx.accounts.account_to_close.to_account_info().lamports())
            .unwrap();
        **ctx.accounts.account_to_close.to_account_info().lamports.borrow_mut() = 0;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Close<'info> {
    account_to_close: Account<'info, Data>,
    destination: AccountInfo<'info>,
}

#[account]
pub struct Data {
    data: u64,
}
```

Однак, збір сміття не відбувається, поки транзакція не завершиться. А оскільки в транзакції може бути кілька інструкцій, це дає зловмиснику можливість викликати інструкцію закриття акаунта, а також включити в транзакцію запит для відшкодування лампортів за звільнення акаунта від орендної плати. В результаті обліковий запис «не буде» очищено від сміття, що відкриває зловмиснику шлях до непередбачуваної поведінки в програмі і навіть до зливу протоколу.

## Безпечне закриття акаунта

Дві найважливіші речі, які ви можете зробити, щоб закрити цю лазівку - це обнулити дані облікового запису та додати дискримінатор облікового запису, який вказує на те, що обліковий запис було закрито. Щоб уникнути непередбачуваної поведінки програми, вам потрібно зробити обидві ці речі.

Обліковий запис з обнуленими даними все ще можна використовувати для деяких речей, особливо якщо це PDA, чия адреса використовується у програмі для верифікації. Однак шкода може бути потенційно обмеженою, якщо зловмисник не зможе отримати доступ до раніше збережених даних.

Однак для додаткового захисту програми закритим обліковим записам слід надати дискримінатор облікових записів, який позначає їх як «закриті», а всі інструкції повинні виконувати перевірку всіх переданих облікових записів, які повертають помилку, якщо обліковий запис позначено як закритий.

Подивіться на приклад нижче. Ця програма переносить лампорти з облікового запису, обнуляє дані облікового запису і встановлює дискримінатор облікового запису в одній інструкції, сподіваючись, що наступна інструкція не зможе знову використати цей обліковий запис до того, як його буде вилучено зі смітника. Якщо не виконати жодної з цих дій, це призведе до вразливості системи безпеки.

```rust
use anchor_lang::prelude::*;
use std::io::Write;
use std::ops::DerefMut;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod closing_accounts_insecure_still_still {
    use super::*;

    pub fn close(ctx: Context<Close>) -> ProgramResult {
        let account = ctx.accounts.account.to_account_info();

        let dest_starting_lamports = ctx.accounts.destination.lamports();

        **ctx.accounts.destination.lamports.borrow_mut() = dest_starting_lamports
            .checked_add(account.lamports())
            .unwrap();
        **account.lamports.borrow_mut() = 0;

        let mut data = account.try_borrow_mut_data()?;
        for byte in data.deref_mut().iter_mut() {
            *byte = 0;
        }

        let dst: &mut [u8] = &mut data;
        let mut cursor = std::io::Cursor::new(dst);
        cursor
            .write_all(&anchor_lang::__private::CLOSED_ACCOUNT_DISCRIMINATOR)
            .unwrap();

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Close<'info> {
    account: Account<'info, Data>,
    destination: AccountInfo<'info>,
}

#[account]
pub struct Data {
    data: u64,
}
```

Зверніть увагу, що у наведеному вище прикладі використовується `CLOSED_ACCOUNT_DISCRIMINATOR` від Anchor. Це просто дискримінатор рахунку, де кожен байт має значення `255`. Дискримінатор не має жодного власного значення, але якщо ви поєднаєте його з перевіркою облікового запису, яка повертає помилки щоразу, коли обліковий запис з цим дискримінатором передається в інструкцію, ви не дасте вашій програмі ненавмисно обробити інструкцію із закритим обліковим записом.

### Ручне відшкодування витрат

Існує ще одна невелика проблема. Хоча практика обнулення даних облікового запису і додавання дискримінатора «closed» запобігає використанню вашої програми, користувач все ще може запобігти видаленню цього акаунта, повернувши його лампорти до завершення інструкції. Це призведе до того, що один або потенційно багато облікових записів перебуватимуть у невизначеному стані, коли їх не можна буде використовувати, але й не можна буде вилучити зі смітника.

Щоб впоратися з цим граничним випадком, ви можете розглянути можливість додавання інструкції, яка дозволить *будь-кому* відшкодовувати кошти з акаунтів, позначених дискримінатором «closed» акаунт. Єдина перевірка облікового запису, яку виконуватиме ця інструкція, - переконатися, що обліковий запис, який вилучається, позначено як закритий. Це може виглядати приблизно так:

```rust
use anchor_lang::__private::CLOSED_ACCOUNT_DISCRIMINATOR;
use anchor_lang::prelude::*;
use std::io::{Cursor, Write};
use std::ops::DerefMut;

...

    pub fn force_defund(ctx: Context<ForceDefund>) -> ProgramResult {
        let account = &ctx.accounts.account;

        let data = account.try_borrow_data()?;
        assert!(data.len() > 8);

        let mut discriminator = [0u8; 8];
        discriminator.copy_from_slice(&data[0..8]);
        if discriminator != CLOSED_ACCOUNT_DISCRIMINATOR {
            return Err(ProgramError::InvalidAccountData);
        }

        let dest_starting_lamports = ctx.accounts.destination.lamports();

        **ctx.accounts.destination.lamports.borrow_mut() = dest_starting_lamports
            .checked_add(account.lamports())
            .unwrap();
        **account.lamports.borrow_mut() = 0;

        Ok(())
    }

...

#[derive(Accounts)]
pub struct ForceDefund<'info> {
    account: AccountInfo<'info>,
    destination: AccountInfo<'info>,
}
```

Оскільки цю інструкцію може викликати будь-хто, це може слугувати стримуючим фактором для спроб відновлення акаунтів, оскільки зловмисник платить за звільнення від оренди акаунта, але будь-хто інший може забрати лампорти з відшкодованого акаунта собі.

Хоча це і не обов'язково, але це може допомогти усунути марнування місця і лампортів, пов'язаних з цими «невизначеними» акаунтами.

## Використання обмеження `close` в Anchor

На щастя, Anchor робить все це набагато простіше за допомогою обмеження `#[account(close = <target_account>)]`. Це обмеження обробляє все, що потрібно для безпечного закриття акаунта:

1. Переносить лампорти облікового запису на заданий `<target_account>`.
2. Обнуляє дані рахунку
3. Встановлює дискримінатор рахунку у значення `CLOSED_ACCOUNT_DISCRIMINATOR`.

Все, що вам потрібно зробити, це додати його в структуру валідації облікового запису, який ви хочете закрити:

```rust
#[derive(Accounts)]
pub struct CloseAccount {
    #[account(
        mut, 
        close = receiver
    )]
    pub data_account: Account<'info, MyData>,
    #[account(mut)]
    pub receiver: SystemAccount<'info>
}
```

Інструкція `force_defund` є необов'язковим доповненням, яке вам доведеться реалізувати самостійно, якщо ви хочете його використовувати.

# Лабораторна робота

Щоб з'ясувати, як зловмисник може скористатися атакою відродження (revival attack), давайте попрацюємо з простою лотерейною програмою, яка використовує стан облікового запису програми для керування участю користувача в лотереї.

## 1. Налаштування

Почніть з отримання коду в гілці `starter` з [наступного репозиторію](https://github.com/Unboxed-Software/solana-closing-accounts/tree/starter).

Код складається з двох інструкцій програми та двох тестів у каталозі `tests`.

Інструкції програми:

1. `enter_lottery`
2. `redeem_rewards_insecure`

Коли користувач викликає `enter_lottery`, програма ініціалізує обліковий запис для зберігання певного стану про участь користувача у лотереї.

Оскільки це спрощений приклад, а не повноцінна лотерейна програма, після того, як користувач зареєструвався у лотереї, він може викликати інструкцію `redeem_rewards_insecure` у будь-який час. Ця команда змінтить користувачеві певну кількість токенів винагород, пропорційну кількості разів, коли користувач брав участь у лотереї. Після мінта винагород програма закриває участь користувача в лотереї.

Приділіть хвилину для ознайомлення з кодом програми. Інструкція `enter_lottery` просто створює обліковий запис на PDA, прив'язаному до користувача, та ініціалізує на ньому деякий стан.

Інструкція `redeem_rewards_insecure` виконує деяку перевірку облікового запису та даних, мінтить токени на даний обліковий запис, а потім закриває лотерейний обліковий запис, видаляючи його лампорти.

Однак, зверніть увагу, що інструкція `redeem_rewards_insecure` лише видаляє лампорти акаунта, залишаючи акаунт відкритим для атак відновлення.

## 2. Протестуйте небезпечну програму

Зловмисник, який успішно утримує свій акаунт від закриття, може викликати `redeem_rewards_insecure` декілька разів, вимагаючи більшу винагороду, ніж йому належить.

Вже написано декілька початкових тестів, які демонструють цю вразливість. Погляньте на файл `closing-accounts.ts` в каталозі `tests`. Там є деякі налаштування у функції `before`, потім тест, який просто створює новий лотерейний запис для `attacker`.

Нарешті, є тест, який демонструє, як зловмисник може зберегти акаунт живим навіть після отримання винагороди, а потім знову вимагати винагороду. Цей тест виглядає наступним чином:

```typescript
it("attacker  can close + refund lottery acct + claim multiple rewards", async () => {
    // claim multiple times
    for (let i = 0; i < 2; i++) {
      const tx = new Transaction()
      // instruction claims rewards, program will try to close account
      tx.add(
        await program.methods
          .redeemWinningsInsecure()
          .accounts({
            lotteryEntry: attackerLotteryEntry,
            user: attacker.publicKey,
            userAta: attackerAta,
            rewardMint: rewardMint,
            mintAuth: mintAuth,
            tokenProgram: TOKEN_PROGRAM_ID,
          })
          .instruction()
      )

      // user adds instruction to refund dataAccount lamports
      const rentExemptLamports =
        await provider.connection.getMinimumBalanceForRentExemption(
          82,
          "confirmed"
        )
      tx.add(
        SystemProgram.transfer({
          fromPubkey: attacker.publicKey,
          toPubkey: attackerLotteryEntry,
          lamports: rentExemptLamports,
        })
      )
      // send tx
      await sendAndConfirmTransaction(provider.connection, tx, [attacker])
      await new Promise((x) => setTimeout(x, 5000))
    }

    const ata = await getAccount(provider.connection, attackerAta)
    const lotteryEntry = await program.account.lotteryAccount.fetch(
      attackerLotteryEntry
    )

    expect(Number(ata.amount)).to.equal(
      lotteryEntry.timestamp.toNumber() * 10 * 2
    )
})
```

Цей тест виконує наступні дії:
1. Викликає `redeem_rewards_insecure` для викупу винагороди користувача
2. У цій же транзакції додає інструкцію повернути користувачеві його `lottery_entry` до того, як вона буде фактично закрита
3. Успішно повторює кроки 1 і 2, викуповуючи винагороди вдруге.

Теоретично ви можете повторювати кроки 1-2 до нескінченності, поки або а) програмі не залишиться більше винагород, або б) хтось не помітить і не виправить експлойт. Очевидно, що це буде серйозною проблемою в будь-якій реальній програмі, оскільки це дозволяє зловмиснику викачати весь фонд винагород.

## 3. Створіть інструкцію `redeem_rewards_secure`

Щоб запобігти цьому, ми створимо нову інструкцію, яка закриває акаунт лотереї безпечно, використовуючи обмеження Anchor `close`. Не соромтеся спробувати це самостійно, якщо хочете.

Нова структура валідації акаунта під назвою `RedeemWinningsSecure` має виглядати наступним чином:

```rust
#[derive(Accounts)]
pub struct RedeemWinningsSecure<'info> {
    // program expects this account to be initialized
    #[account(
        mut,
        seeds = [user.key().as_ref()],
        bump = lottery_entry.bump,
        has_one = user,
        close = user
    )]
    pub lottery_entry: Account<'info, LotteryAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    #[account(
        mut,
        constraint = user_ata.key() == lottery_entry.user_ata
    )]
    pub user_ata: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = reward_mint.key() == user_ata.mint
    )]
    pub reward_mint: Account<'info, Mint>,
    ///CHECK: mint authority
    #[account(
        seeds = [MINT_SEED.as_bytes()],
        bump
    )]
    pub mint_auth: AccountInfo<'info>,
    pub token_program: Program<'info, Token>
}
```

Вона має бути точно такою ж, як і оригінальна структура перевірки акаунта `RedeemWinnings`, за винятком додаткового обмеження `close = user` для акаунта `lottery_entry`. Це вкаже Anchor закрити акаунт шляхом обнулення даних, перенесення його лампортів до акаунта `user` і встановлення дискримінатора акаунта на `CLOSED_ACCOUNT_DISCRIMINATOR`. Останній крок запобігатиме повторному використанню облікового запису, якщо програма вже намагалася його закрити.

Потім ми можемо створити метод `mint_ctx` на новій структурі `RedeemWinningsSecure`, щоб допомогти з мінтом CPI для програми токенів.

```Rust
impl<'info> RedeemWinningsSecure <'info> {
    pub fn mint_ctx(&self) -> CpiContext<'_, '_, '_, 'info, MintTo<'info>> {
        let cpi_program = self.token_program.to_account_info();
        let cpi_accounts = MintTo {
            mint: self.reward_mint.to_account_info(),
            to: self.user_ata.to_account_info(),
            authority: self.mint_auth.to_account_info()
        };

        CpiContext::new(cpi_program, cpi_accounts)
    }
}
```

Нарешті, логіка нової безпечної інструкції повинна виглядати так:

```rust
pub fn redeem_winnings_secure(ctx: Context<RedeemWinningsSecure>) -> Result<()> {

    msg!("Calculating winnings");
    let amount = ctx.accounts.lottery_entry.timestamp as u64 * 10;

    msg!("Minting {} tokens in rewards", amount);
    // program signer seeds
    let auth_bump = *ctx.bumps.get("mint_auth").unwrap();
    let auth_seeds = &[MINT_SEED.as_bytes(), &[auth_bump]];
    let signer = &[&auth_seeds[..]];

    // redeem rewards by minting to user
    mint_to(ctx.accounts.mint_ctx().with_signer(signer), amount)?;

    Ok(())
}
```

Ця логіка просто обчислює винагороду для користувача, який подав заявку, і передає її. Однак, через обмеження `close` у структурі перевірки облікового запису, зловмисник не зможе викликати цю інструкцію декілька разів.

## 4. Тестування програми

Щоб протестувати нашу нову безпечну інструкцію, давайте створимо новий тест, який спробує викликати `redeemingWinningsSecure` двічі. Ми очікуємо, що другий виклик призведе до помилки.

```typescript
it("attacker cannot claim multiple rewards with secure claim", async () => {
    const tx = new Transaction()
    // instruction claims rewards, program will try to close account
    tx.add(
      await program.methods
        .redeemWinningsSecure()
        .accounts({
          lotteryEntry: attackerLotteryEntry,
          user: attacker.publicKey,
          userAta: attackerAta,
          rewardMint: rewardMint,
          mintAuth: mintAuth,
          tokenProgram: TOKEN_PROGRAM_ID,
        })
        .instruction()
    )

    // user adds instruction to refund dataAccount lamports
    const rentExemptLamports =
      await provider.connection.getMinimumBalanceForRentExemption(
        82,
        "confirmed"
      )
    tx.add(
      SystemProgram.transfer({
        fromPubkey: attacker.publicKey,
        toPubkey: attackerLotteryEntry,
        lamports: rentExemptLamports,
      })
    )
    // send tx
    await sendAndConfirmTransaction(provider.connection, tx, [attacker])

    try {
      await program.methods
        .redeemWinningsSecure()
        .accounts({
          lotteryEntry: attackerLotteryEntry,
          user: attacker.publicKey,
          userAta: attackerAta,
          rewardMint: rewardMint,
          mintAuth: mintAuth,
          tokenProgram: TOKEN_PROGRAM_ID,
        })
        .signers([attacker])
        .rpc()
    } catch (error) {
      console.log(error.message)
      expect(error)
    }
})
```

Запустіть `anchor test`, щоб переконатися, що тест пройдено. Результат буде виглядати приблизно так:

```bash
  closing-accounts
    ✔ Enter lottery (451ms)
    ✔ attacker can close + refund lottery acct + claim multiple rewards (18760ms)
AnchorError caused by account: lottery_entry. Error Code: AccountDiscriminatorMismatch. Error Number: 3002. Error Message: 8 byte discriminator did not match what was expected.
    ✔ attacker cannot claim multiple rewards with secure claim (414ms)
```

Зауважте, що це не заважає зловмиснику повністю повернути кошти на свій акаунт - це лише захищає нашу програму від випадкового повторного використання акаунта, коли він має бути закритий. Поки що ми не реалізували інструкцію `force_defund`, але ми могли б це зробити. Якщо ви відчуваєте, що готові до цього, спробуйте самі!

Найпростіший і найбезпечніший спосіб закриття акаунтів - це використання обмеження `close` в Anchor. Якщо вам коли-небудь знадобиться більш кастомна поведінка і ви не зможете використовувати це обмеження, переконайтеся, що ви реплікуєте його функціональність, щоб забезпечити безпеку вашої програми.

Якщо ви хочете поглянути на остаточний код рішення, ви можете знайти його в гілці `solution` у [цьому ж репозиторії] (https://github.com/Unboxed-Software/solana-closing-accounts/tree/solution).

# Виклик

Як і в інших уроках цього розділу, ваша можливість попрактикуватися в уникненні цієї уразливості полягає в аудиті власних або інших програм.

Витратьте трохи часу на перевірку хоча б однієї програми і переконайтеся, що після закриття облікових записів вони не будуть вразливі до атак відновлення.

Пам'ятайте, якщо ви знайшли баг або уразливість в чужій програмі, будь ласка, повідомте про це! Якщо ви знайшли таку у власній програмі, не забудьте негайно виправити її.


## Виконали завдання?

Завантажте свій код на GitHub і [розкажіть нам, що ви думаєте про цей урок] (https://form.typeform.com/to/IPH0UGz7#answers-lesson=e6b99d4b-35ed-4fb2-b9cd-73eefc875a0f)!
