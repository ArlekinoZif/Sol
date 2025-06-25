---
назва: Локальна розробка програм
завдвння:
- Налаштувати локальне середовище для розробки програм Solana, встановивши Solana CLI, Rust та Anchor.
- Переконатися, що Anchor працює "з коробки" без помилок та попереджень.
---

# Стислий виклад 

* Для розробки ончен-програм на вашому комп’ютері потрібні **Solana CLI**, **Rust** та (опційно, але рекомендовано) **Anchor**.
* За допомогою `anchor init` можна створити новий порожній проект Anchor.
* Команда `anchor test` запускає ваші тести та одночасно збирає код.

# Урок

Тут немає теоретичного уроку! Давайте встановимо інструменти Solana CLI, Rust SDK та Anchor, а також створимо тестову програму, щоб переконатися, що наше середовище працює.

# Лабораторна робота

### Додаткові кроки для користувачів Windows

Спершу встановіть [Windows Terminal](https://apps.microsoft.com/detail/9N0DX20HK701) із Microsoft Store.

Потім [встановіть Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install). WSL надає середовище Linux, яке запускається миттєво, коли вам потрібно, і не сповільнює роботу комп’ютера.

Запустіть Windows Terminal, відкрийте сесію 'Ubuntu' всередині терміналу та продовжуйте виконувати наступні кроки.

### Завантаження Rust

Спочатку завантажте Rust, [дотримуючись інструкцій](https://www.rust-lang.org/tools/install):

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Завантаження інструментів Solana CLI

Далі [завантажте інструменти Solana CLI](https://docs.solana.com/cli/install-solana-cli-tools).

```
sh -c "$(curl -sSfL https://release.solana.com/beta/install)"
```

Після цього команда `solana -V` повинна показати `solana-cli 1.18.x` (будь-яке число замість `x` — це нормально).

### Завантаження Anchor

Наостанок [завантажте Anchor](https://www.anchor-lang.com/docs/installation):

```
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force
avm install latest
avm use latest
```

Після цього команда `anchor -V` повинна показати `anchor-cli 0.30.0`.

### Перевірка встановлення Anchor

Створіть тимчасовий проєкт із типовим вмістом за допомогою Anchor і переконайтеся, що він компілюється та проходить тести:

```bash
anchor init temp-project
cd temp-project
anchor test
```

**Команда `anchor test` повинна завершитись без помилок і попереджень**. Однак можуть виникнути певні проблеми — нижче ми покажемо, як їх вирішити:

#### Помилка `package \`solana-program v1.18.12\` cannot be built because it requires rustc 1.75.0 or newer\`

Виконайте команду `cargo add solana-program@"=1.18.x"`, де `x` має відповідати вашій версії `solana-cli`. Потім повторно запустіть `anchor test`.

#### `Error: Unable to read keypair file`

Додайте пару ключів до `.config/solana/id.json`. Ви можете або скопіювати пару ключів з `.env`-файлу (тільки масив чисел) у файл, або скористатися командою `solana-keygen new --no-bip39-passphrase`, щоб створити новий файл із парою ключів. Потім повторно запустіть `anchor test`.

#### Попередження `unused variable: 'ctx'`

Це просто означає, що обробник інструкції `initialize` поки нічого не робить. Ви можете відкрити `programs/favorites/src/lib.rs` і змінити `ctx` на `_ctx`, або просто перейти до наступного кроку.

#### Попередження `No license field in package.json`

Відкрийте `package.json` і додайте `"license": "MIT"` або `"license": "UNLICENSED"` залежно від ваших уподобань.

### Все готово?

Переконайтеся, що `anchor test` завершується успішно - без попереджень і без помилок - перш ніж продовжити.

## Завершили лабораторну роботу?

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=aa0b56d6-02a9-4b36-95c0-a817e2c5b19d)!
