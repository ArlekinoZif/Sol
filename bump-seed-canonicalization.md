---
назва: Bump Seed Canonicalization
завдання:
- Пояснити вразливості, пов'язані з використанням PDA, похідних без канонічного bump.
- Ініціалізувати PDA за допомогою обмежень `seeds` і `bump` Anchor, щоб автоматично використовувати канонічний bump.
- Використати обмеження `seeds` і `bump` Anchor, щоб забезпечити використання канонічного bump в майбутніх інструкціях при похідних PDA.
---

# Стислий виклад

- Функція [**`create_program_address`**](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.create_program_address) отримує PDA без пошуку **канонічного bump**. Це означає, що існує кілька дійсних bump, кожен з яких дає різні адреси.
- Використання [**`find_program_address`**](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.find_program_address) гарантує, що для отримання адреси буде використано найвищий дійсний bump або канонічний bump, що забезпечує детермінований спосіб знаходження адреси за певними seed.
- Під час ініціалізації ви можете використовувати обмеження `seeds` та `bump` в Anchor, щоб забезпечити, що отримання PDA в структурі перевірки акаунтів завжди використовуватиме канонічний bump.
- Anchor дозволяє вам **вказати bump** за допомогою обмеження `bump = <some_bump>`, коли ви перевіряєте PDA.
- Оскільки `find_program_address` може бути дороговартісним, найкраща практика — зберігати отриманий bump у полі даних акаунту для подальшого використання при повторному отриманні адреси для перевірки.
    ```rust
    #[derive(Accounts)]
    pub struct VerifyAddress<'info> {
    	#[account(
        	seeds = [DATA_PDA_SEED.as_bytes()],
    	    bump = data.bump
    	)]
    	data: Account<'info, Data>,
    }
    ```

# Урок

Bump seeds — це число від 0 до 255 включно, яке використовується для забезпечення того, щоб адреса, отримана за допомогою [`create_program_address`](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.create_program_address), була дійсною PDA. **Канонічний bump** — це найвищий bump, який дає дійсну PDA. Стандарт в Solana — це *завжди використовувати канонічний bump* при отриманні PDA, як для безпеки, так і для зручності.

## Небезпечне виведення PDA за допомогою `create_program_address`

Задано набір seed, функція `create_program_address` генерує дійсну PDA приблизно в 50% випадків. Bump seed — це додатковий байт, який додається як seed для "підвищення" отриманої адреси в дійсну область. Оскільки існує 256 можливих bump seeds і функція генерує дійсні PDA приблизно в 50% випадків, існує багато дійсних bump для заданого набору вхідних seed.

Це може спричинити плутанину при пошуку акаунтів, коли використовуються seed для відображення між відомими даними та акаунтами. Використання канонічного bump як стандарту гарантує, що ви завжди зможете знайти правильний акаунт. Більш важливо, що це дозволяє уникнути вразливостей безпеки, спричинених відкритою можливістю для використання кількох bumps.

У наведеному нижче прикладі інструкція `set_value` використовує `bump`, який був переданий, як дані інструкції для виведення PDA. Після цього інструкція виводить PDA за допомогою функції `create_program_address` і перевіряє, чи співпадає `address` з публічним ключем акаунту `data`.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod bump_seed_canonicalization_insecure {
    use super::*;

    pub fn set_value(ctx: Context<BumpSeed>, key: u64, new_value: u64, bump: u8) -> Result<()> {
        let address =
            Pubkey::create_program_address(&[key.to_le_bytes().as_ref(), &[bump]], ctx.program_id).unwrap();
        if address != ctx.accounts.data.key() {
            return Err(ProgramError::InvalidArgument.into());
        }

        ctx.accounts.data.value = new_value;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct BumpSeed<'info> {
    data: Account<'info, Data>,
}

#[account]
pub struct Data {
    value: u64,
}
```

Хоча інструкція отримує PDA та перевіряє переданий акаунт, що є перевагою, вона дозволяє викликачеві передати довільний bump. Залежно від контексту вашої програми, це може призвести до небажаної поведінки або потенційної вразливості.

Якщо співставлення seed було призначено для забезпечення однозначного відношення між PDA та користувачем, наприклад, ця програма не буде належним чином це забезпечувати. Користувач може викликати програму кілька разів з багатьма дійсними "bump", кожен з яких створює різний PDA.

## Рекомендоване похідне створення за допомогою `find_program_address`

Простий спосіб обійти цю проблему полягає в тому, щоб програма очікувала лише канонічне значення `bump` та використовувала `find_program_address` для похідного створення PDA.

Функція [`find_program_address`](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.find_program_address) *завжди використовує канонічне значення `bump`*. Ця функція ітерується через виклики `create_program_address`, починаючи з bump, що дорівнює 255 та зменшуючи його на одиницю з кожною ітерацією. Як тільки знайдено дійсну адресу, функція повертає як отриману PDA, так і канонічне значення bump, яке використовувалося для її створення.

Це забезпечує однозначне відображення між введеними seed та адресою, яку вони генерують.

```rust
pub fn set_value_secure(
    ctx: Context<BumpSeed>,
    key: u64,
    new_value: u64,
    bump: u8,
) -> Result<()> {
    let (address, expected_bump) =
        Pubkey::find_program_address(&[key.to_le_bytes().as_ref()], ctx.program_id);

    if address != ctx.accounts.data.key() {
        return Err(ProgramError::InvalidArgument.into());
    }
    if expected_bump != bump {
        return Err(ProgramError::InvalidArgument.into());
    }

    ctx.accounts.data.value = new_value;
    Ok(())
}
```

## Використайте обмеження `seeds` та `bump` в Anchor.

Anchor забезпечує зручний спосіб отримання PDA в структурі перевірки акаунтів, використовуючи обмеження `seeds` та `bump`. Ці обмеження можна навіть поєднати з обмеженням `init`, щоб ініціалізувати акаунт за потрібною адресою. Щоб захистити програму від вразливості, про яку ми говорили протягом цього уроку, Anchor навіть не дозволяє ініціалізувати акаунт за PDA, використовуючи будь-що, окрім канонічного bump. Замість цього вона використовує `find_program_address` для отримання PDA та подальшої ініціалізації.

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod bump_seed_canonicalization_recommended {
    use super::*;

    pub fn set_value(ctx: Context<BumpSeed>, _key: u64, new_value: u64) -> Result<()> {
        ctx.accounts.data.value = new_value;
        Ok(())
    }
}

// initialize account at PDA
#[derive(Accounts)]
#[instruction(key: u64)]
pub struct BumpSeed<'info> {
  #[account(mut)]
  payer: Signer<'info>,
  #[account(
    init,
    seeds = [key.to_le_bytes().as_ref()],
    // derives the PDA using the canonical bump
    bump,
    payer = payer,
    space = 8 + 8
  )]
  data: Account<'info, Data>,
  system_program: Program<'info, System>
}

#[account]
pub struct Data {
    value: u64,
}
```

Якщо ви не ініціалізуєте акаунт, ви все ще можете перевіряти PDA, використовуючи обмеження `seeds` та `bump`. Це просто перераховує PDA і порівнює отриману адресу з адресою переданого акаунту.

У цьому випадку Anchor *дозволяє* вам вказати bump, який слід використовувати для отримання PDA з `bump = <some_bump>`. Тут мета полягає не в тому, щоб ви використовували довільні bump, а в тому, щоб дозволити вам оптимізувати програму. Ітеративний характер `find_program_address` робить його дорогим, тому найкраще зберігати канонічний bump у даних акаунту PDA при ініціалізації PDA, що дозволяє посилатися на збережений bump при перевірці PDA в наступних інструкціях.

Коли ви вказуєте bump для використання, Anchor використовує `create_program_address` з наданим bump замість `find_program_address`. Цей шаблон збереження bump у даних акаунту гарантує, що ваша програма завжди використовує канонічний bump без погіршення продуктивності.

```rust
use anchor_lang::prelude::*;

declare_id!("CVwV9RoebTbmzsGg1uqU1s4a3LvTKseewZKmaNLSxTqc");

#[program]
pub mod bump_seed_canonicalization_recommended {
    use super::*;

    pub fn set_value(ctx: Context<BumpSeed>, _key: u64, new_value: u64) -> Result<()> {
        ctx.accounts.data.value = new_value;
        // store the bump on the account
        ctx.accounts.data.bump = *ctx.bumps.get("data").unwrap();
        Ok(())
    }

    pub fn verify_address(ctx: Context<VerifyAddress>, _key: u64) -> Result<()> {
        msg!("PDA confirmed to be derived with canonical bump: {}", ctx.accounts.data.key());
        Ok(())
    }
}

// initialize account at PDA
#[derive(Accounts)]
#[instruction(key: u64)]
pub struct BumpSeed<'info> {
  #[account(mut)]
  payer: Signer<'info>,
  #[account(
    init,
    seeds = [key.to_le_bytes().as_ref()],
    // derives the PDA using the canonical bump
    bump,
    payer = payer,
    space = 8 + 8 + 1
  )]
  data: Account<'info, Data>,
  system_program: Program<'info, System>
}

#[derive(Accounts)]
#[instruction(key: u64)]
pub struct VerifyAddress<'info> {
  #[account(
    seeds = [key.to_le_bytes().as_ref()],
    // guranteed to be the canonical bump every time
    bump = data.bump
  )]
  data: Account<'info, Data>,
}

#[account]
pub struct Data {
    value: u64,
    // bump field
    bump: u8
}
```

Якщо ви не вказуєте bump в обмеженні `bump`, Anchor все одно буде використовувати `find_program_address` для отримання PDA з використанням канонічного bump. Як наслідок, вашій інструкції буде надано змінний обсяг обчислювального бюджету. Програми, які вже ризикують перевищити свій обчислювальний бюджет, повинні використовувати це обмеження обережно, оскільки існує ймовірність того, що бюджет програми може бути непередбачувано перевищено.

З іншого боку, якщо вам потрібно лише перевірити PDA, що передається, без ініціалізації акаунту, вам доведеться або дозволити Anchor визначити канонічний bump, або піддати вашу програму непотрібним ризикам. У такому разі, будь ласка, використовуйте канонічний bump, незважаючи на незначне зниження продуктивності.

# Лабораторна робота

Щоб продемонструвати можливість використання вразливостей без перевірки канонічного bump, давайте попрацюємо з програмою, яка дозволяє кожному користувачеві програми виконати "claim" винагороди вчасно.

### 1. Налаштування

Для початку, отримайте код з гілки `starter` з [цього репозиторію](https://github.com/Unboxed-Software/solana-bump-seed-canonicalization/tree/starter).

Зверніть увагу, що у програмі є дві інструкції та один тест у директорії `tests`.

Інструкції у програмі:

1. `create_user_insecure`
2. `claim_insecure`

Інструкція `create_user_insecure` просто створює новий акаунт за адресою PDA, який походить від публічного ключа підписанта та переданого значення bump.

Інструкція `claim_insecure` мінтить 10 токенів для користувача, а потім позначає винагороди акаунта як отримані, щоб уникнути повторних заявок.

Однак програма не перевіряє, що PDA використовує канонічний bump.

Перегляньте програму, щоб зрозуміти, що вона робить, перед тим як продовжувати.

### 2. Тести незахищених інструкцій

Оскільки інструкції не вимагають явно, щоб `user` PDA використовував канонічний bump, зловмисник може створити кілька акаунтів на один гаманець і отримати більше винагород, ніж дозволено.

Тест у директорії `tests` створює нову пару ключів під назвою `attacker`, щоб представити зловмисника. Потім він пройде через всі можливі bump і викличе `create_user_insecure` та `claim_insecure`. В кінці тест очікує, що зловмисник зміг отримати нагороди кілька разів та заробив більше, ніж 10 токенів на одного користувача.

```typescript
it("Attacker can claim more than reward limit with insecure instructions", async () => {
    const attacker = Keypair.generate()
    await safeAirdrop(attacker.publicKey, provider.connection)
    const ataKey = await getAssociatedTokenAddress(mint, attacker.publicKey)

    let numClaims = 0

    for (let i = 0; i < 256; i++) {
      try {
        const pda = createProgramAddressSync(
          [attacker.publicKey.toBuffer(), Buffer.from([i])],
          program.programId
        )
        await program.methods
          .createUserInsecure(i)
          .accounts({
            user: pda,
            payer: attacker.publicKey,
          })
          .signers([attacker])
          .rpc()
        await program.methods
          .claimInsecure(i)
          .accounts({
            user: pda,
            mint,
            payer: attacker.publicKey,
            userAta: ataKey,
          })
          .signers([attacker])
          .rpc()

        numClaims += 1
      } catch (error) {
        if (
          error.message !== "Invalid seeds, address must fall off the curve"
        ) {
          console.log(error)
        }
      }
    }

    const ata = await getAccount(provider.connection, ataKey)

    console.log(
      `Attacker claimed ${numClaims} times and got ${Number(ata.amount)} tokens`
    )

    expect(numClaims).to.be.greaterThan(1)
    expect(Number(ata.amount)).to.be.greaterThan(10)
})
```

Запустіть `anchor test`, щоб переконатися, що цей тест пройде, показуючи успішність атаки. Оскільки тест викликає інструкції для кожного дійсного bump, він займає трохи часу, будь ласка, будьте терплячими.

```bash
  bump-seed-canonicalization
Attacker claimed 129 times and got 1290 tokens
    ✔ Attacker can claim more than reward limit with insecure instructions (133840ms)
```

### 3. Створення безпечних інструкції

Давайте продемонструємо усунення цієї вразливості, створивши дві нові інструкції:

1. `create_user_secure`
2. `claim_secure`

Перш ніж ми напишемо перевірку акаунту або логіку інструкцій, давайте створимо новий тип користувача `UserSecure`. Цей новий тип додасть канонічний bump як поле в структурі.

```rust
#[account]
pub struct UserSecure {
    auth: Pubkey,
    bump: u8,
    rewards_claimed: bool,
}
```

Далі створимо структури перевірки акаунту для кожної з нових інструкцій. Вони будуть дуже схожі на небезпечні версії, але Anchor буде відповідальним за виведення та десеріалізацію PDA.

```rust
#[derive(Accounts)]
pub struct CreateUserSecure<'info> {
    #[account(mut)]
    payer: Signer<'info>,
    #[account(
        init,
        seeds = [payer.key().as_ref()],
        // derives the PDA using the canonical bump
        bump,
        payer = payer,
        space = 8 + 32 + 1 + 1
    )]
    user: Account<'info, UserSecure>,
    system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct SecureClaim<'info> {
    #[account(
        seeds = [payer.key().as_ref()],
        bump = user.bump,
        constraint = !user.rewards_claimed @ ClaimError::AlreadyClaimed,
        constraint = user.auth == payer.key()
    )]
    user: Account<'info, UserSecure>,
    #[account(mut)]
    payer: Signer<'info>,
    #[account(
        init_if_needed,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = payer
    )]
    user_ata: Account<'info, TokenAccount>,
    #[account(mut)]
    mint: Account<'info, Mint>,
    /// CHECK: mint auth PDA
    #[account(seeds = ["mint".as_bytes().as_ref()], bump)]
    pub mint_authority: UncheckedAccount<'info>,
    token_program: Program<'info, Token>,
    associated_token_program: Program<'info, AssociatedToken>,
    system_program: Program<'info, System>,
    rent: Sysvar<'info, Rent>,
}
```

Наостанок реалізуємо логіку для двох нових інструкцій. Інструкція `create_user_secure` повинна просто встановити значення полів `auth`, `bump` і `rewards_claimed` у даних акаунту `user`.

```rust
pub fn create_user_secure(ctx: Context<CreateUserSecure>) -> Result<()> {
    ctx.accounts.user.auth = ctx.accounts.payer.key();
    ctx.accounts.user.bump = *ctx.bumps.get("user").unwrap();
    ctx.accounts.user.rewards_claimed = false;
    Ok(())
}
```

Інструкція `claim_secure` повинна мінтити 10 токенів користувачу та встановлювати поле `rewards_claimed` акаунту `user` в `true`.

```rust
pub fn claim_secure(ctx: Context<SecureClaim>) -> Result<()> {
    token::mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                mint: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.user_ata.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            &[&[
                    b"mint".as_ref(),
                &[*ctx.bumps.get("mint_authority").unwrap()],
            ]],
        ),
        10,
    )?;

    ctx.accounts.user.rewards_claimed = true;

    Ok(())
}
```

### 4. Протестуйте безпечні інструкції

Давайте продовжимо і напишемо тест, щоб показати, що тепер зловмисник не може клеймити більше одного разу за допомогою нових інструкцій.

Зверніть увагу, що якщо ви почнете перебирати кілька PDA, як у старому тесті, ви навіть не зможете передати неканонічний bump до інструкцій. Однак ви все ще можете перебирати різні PDA і в кінці перевірити, що стався тільки 1 клейм на загальну суму 10 токенів. Ваш кінцевий тест буде виглядати приблизно так:

```typescript
it.only("Attacker can only claim once with secure instructions", async () => {
    const attacker = Keypair.generate()
    await safeAirdrop(attacker.publicKey, provider.connection)
    const ataKey = await getAssociatedTokenAddress(mint, attacker.publicKey)
    const [userPDA] = findProgramAddressSync(
      [attacker.publicKey.toBuffer()],
      program.programId
    )

    await program.methods
      .createUserSecure()
      .accounts({
        payer: attacker.publicKey,
      })
      .signers([attacker])
      .rpc()

    await program.methods
      .claimSecure()
      .accounts({
        payer: attacker.publicKey,
        userAta: ataKey,
        mint,
        user: userPDA,
      })
      .signers([attacker])
      .rpc()

    let numClaims = 1

    for (let i = 0; i < 256; i++) {
      try {
        const pda = createProgramAddressSync(
          [attacker.publicKey.toBuffer(), Buffer.from([i])],
          program.programId
        )
        await program.methods
          .createUserSecure()
          .accounts({
            user: pda,
            payer: attacker.publicKey,
          })
          .signers([attacker])
          .rpc()

        await program.methods
          .claimSecure()
          .accounts({
            payer: attacker.publicKey,
            userAta: ataKey,
            mint,
            user: pda,
          })
          .signers([attacker])
          .rpc()

        numClaims += 1
      } catch {}
    }

    const ata = await getAccount(provider.connection, ataKey)

    expect(Number(ata.amount)).to.equal(10)
    expect(numClaims).to.equal(1)
})
```

```bash
  bump-seed-canonicalization
Attacker claimed 119 times and got 1190 tokens
    ✔ Attacker can claim more than reward limit with insecure instructions (128493ms)
    ✔ Attacker can only claim once with secure instructions (1448ms)
```

Якщо ви використовуєте Anchor для всіх похідних PDA, досить просто уникнути цього конкретного експлойта. Однак, якщо ви будете робити щось "нестандартне", будьте обережні та розробіть вашу програму так, щоб вона використовувала канонічний bump!

Якщо ви хочете переглянути остаточний код рішення, ви можете знайти його в гілці `solution` [в тому ж репозиторії](https://github.com/Unboxed-Software/solana-bump-seed-canonicalization/tree/solution).

# Завдання

Так само, як і з іншими уроками цього розділу, ваша можливість попрактикуватися в уникненні цієї вразливості полягає в аудиті власних або інших програм.

Приділіть час, щоб переглянути принаймні одну програму та переконайтеся, що всі похідні та перевірки PDA використовують канонічний bump.

Пам'ятайте, якщо ви виявите помилку або вразливість у чужій програмі, будь ласка, повідомте про це її автора! Якщо ви виявите таке у своїй програмі, не забудьте виправити помилку негайно.

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=d3f6ca7a-11c8-421f-b7a3-d6c08ef1aa8b)!
