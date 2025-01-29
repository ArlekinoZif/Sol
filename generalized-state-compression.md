---
заголовок: Generalized State Compression
цілі:
- Пояснити логіку стиснення стану в Solana  
- Пояснити різницю між деревом Меркля та одночасним деревом Меркля  
- Реалізувати загальне стиснення стану в базових програмах Solana
---

# Підсумок
- Стиснення стану на Solana найчастіше використовується для стиснених NFT, але його можна застосовувати і для довільних даних.
- Стиснення стану зменшує обсяг даних, які потрібно зберігати ончейн, використовуючи дерева Меркля.
- Дерева Меркля зберігають один хеш, що представляє ціле бінарне дерево хешів. Кожен лист дерева Меркля — це хеш даних цього листа.
- Одночасні дерева Меркля — це спеціалізована версія дерев Меркля, що дозволяє виконувати одночасні оновлення.
- Оскільки дані у програмі зі стисненим станом не зберігаються ончейн, вам потрібно використовувати індексатори для підтримки кешу даних офф-чейн, а потім перевіряти ці дані з ончейн-деревом Меркля.

# Урок

Раніше ми обговорювали стиснення стану в контексті стиснених NFT. На момент написання стиснуті NFT є найбільш поширеним випадком використання стиснення стану, але стиснення стану можна застосовувати в будь-якій програмі. У цьому уроці ми обговоримо стиснення стану в більш загальних термінах, щоб ви могли застосувати це до будь-якої з ваших програм.

## Теоретичний огляд стиснення стану

У традиційних програмах дані серіалізуються (зазвичай за допомогою borsh), а потім зберігаються безпосередньо в акаунті. Це дозволяє легко читати та записувати дані через програми Solana. Ви можете "довіряти" даним, що зберігаються в акаунтах, оскільки їх не можна змінити, окрім як через механізми, які надає програма.

Стиснення стану фактично стверджує, що найважливішою частиною цього рівняння є те, наскільки "надійними" є дані. Якщо все, що нас цікавить, — це можливість довіряти тим даним, що вони справжні, то ми насправді можемо обійтися ***без*** зберігання даних в акаунті ончейн. Замість цього ми можемо зберігати хеші даних, які можна використовувати для доведення або перевірки цих даних. Хеш даних займає значно менше місця для зберігання, ніж самі дані. Потім ми можемо зберігати самі дані в набагато дешевшому місці та турбуватися про перевірку їх на відповідність хешу ончейн, коли дані будуть доступні.

Структурою даних, яку використовує програма стиснення стану Solana, є спеціальна бінарна структура дерева, відома як **одночасне (concurrent) дерево Меркла**. Ця структура хешує частини даних разом у детермінований спосіб, щоб обчислити єдиний фінальний хеш, який зберігається ончейн. Цей фінальний хеш є значно меншим за розміром, ніж усі початкові дані разом, звідси й назва — "стиснення". Процес відбувається у таких кроках:

1. Візьміть будь-який фрагмент даних.  
2. Створіть хеш для цих даних.  
3. Збережіть цей хеш як “листок” у нижній частині дерева.  
4. Кожну пару листків хешуйте разом, створюючи “гілку”.  
5. Кожну пару гілок хешуйте разом.  
6. Продовжуйте підніматися по дереву, хешуючи сусідні гілки разом.  
7. Діставшись верхівки дерева, отримайте фінальний “кореневий хеш”.  
8. Збережіть кореневий хеш ончейн як доказ, що підтверджує дані в кожному листку.  
9. Кожен, хто хоче перевірити, чи відповідають його дані "джерелу істини", може пройти той самий процес і порівняти фінальний хеш, не зберігаючи всі дані ончейн.

Це включає кілька досить серйозних компромісів у розробці:

1. Оскільки дані більше не зберігаються в ончейн, доступ до них стає складнішим.  
2. Після отримання доступу до даних розробники повинні вирішити, як часто їхні програми перевірятимуть дані на відповідність хешу в блокчейні.  
3. Будь-які зміни даних вимагатимуть надсилання всієї попередньо хешованої інформації *та* нових даних у виклик інструкції. Розробникам, можливо, також доведеться надати додаткові дані, необхідні для перевірки оригінальних даних на відповідність хешу.

Кожен із цих аспектів потрібно враховувати, визначаючи, **чи**, **коли** і **як** впроваджувати стиснення стану у вашу програму.

### Одночасні дерева Меркла  

**Дерево Меркла** — це бінарна структура дерева, яка представляється одним хешем. Кожен лист дерева є хешем своїх внутрішніх даних, а кожна гілка — це хеш хешів дочірніх листків. У свою чергу, гілки також хешуються разом, доки в результаті не залишається один фінальний кореневий хеш.

Оскільки дерево Меркла представляється одним хешем, будь-яка зміна даних листка змінює кореневий хеш. Це створює проблему, коли кілька транзакцій в одному слоті намагаються змінити дані листка. Оскільки ці транзакції повинні виконуватись послідовно, усі, окрім першої, завершаться невдачею, оскільки кореневий хеш і доказ, передані в транзакції, будуть анульовані першою транзакцією, яка виконається. Іншими словами, стандартне дерево Меркла може змінювати лише один листок за слот. У гіпотетичній програмі зі стисненим станом, яка покладається на одне дерево Меркла для свого стану, це суттєво обмежує пропускну здатність.

Цю проблему можна вирішити за допомогою **одночасного дерева Меркла**. Одночасне дерево Меркла — це дерево Меркла, яке зберігає безпечний журнал змін із найновішими змінами разом із їхнім кореневим хешем і доказом для його отримання. Коли кілька транзакцій у тому самому слоті намагаються змінити дані листків, журнал змін можна використовувати як джерело істини, щоб дозволити одночасне внесення змін до дерева.

Іншими словами, тоді як акаунт, який зберігає дерево Меркла, матиме лише кореневий хеш, одночасне дерево Меркла також міститиме додаткові дані, які дозволяють успішно виконувати наступні записи. Це включає:

1. Кореневий хеш — такий самий кореневий хеш, який має стандартне дерево Меркла.  
2. Буфер журналу змін — цей буфер містить дані доказів, що стосуються останніх змін кореневого хеша, щоб наступні записи в тому ж слоті могли бути успішними.  
3. Навіс (canopy) — під час оновлення будь-якого листка необхідно мати весь ланцюжок доказів від цього листка до кореневого хеша. Навіс зберігає проміжні вузли цього ланцюжка, щоб їх не потрібно було щоразу передавати з клієнта в програму.

Як архітектор програми, ви керуєте трьома параметрами, які безпосередньо пов’язані з цими трьома елементами. Ваш вибір визначає розмір дерева, вартість його створення та кількість одночасних змін, які можна вносити до дерева:

1. Максимальна глибина (Max depth)  
2. Максимальний розмір буфера (Max buffer size)  
3. Глибина навісу (Canopy depth)

**Максимальна глибина** (max depth) — це максимальна кількість кроків (перехід між вузлами), необхідних для переходу від будь-якого листка до кореня дерева. Оскільки дерева Меркла є бінарними деревами, кожен листок з’єднаний тільки з одним іншим листком. Тому максимальна глибина може логічно використовуватися для обчислення кількості вузлів (nodes) у дереві за допомогою формули `2 ^ maxDepth`.

**Максимальний розмір буфера** (max buffer size) — це фактично максимальна кількість одночасних змін, які можна внести до дерева в межах одного слота, зберігаючи при цьому кореневий хеш чинним. Коли кілька транзакцій подаються в одному слоті, і кожна з них намагається оновити листя в стандартному дереві Меркла, лише перша, яка виконується, буде чинною. Це тому, що операція "запису" змінює хеш, що зберігається в акаунті. Наступні транзакції в тому ж слоті намагатимуться перевірити свої дані проти тепер уже застарілого хешу. Одночасне дерево Меркла має буфер, який зберігає поточний журнал цих змін. Це дозволяє програмі зі стисненням стану перевіряти кілька записів даних в одному слоті, оскільки вона може звертатися до попередніх хешів у буфері та порівнювати їх з відповідними хешами.

**Глибина навісу** (canopy depth) — це кількість вузлів доказу, які зберігаються ончейн для будь-якого шляху доказу. Для перевірки будь-якого листка потрібен повний шлях доказу для дерева. Повний шлях доказу складається з одного вузла доказу для кожного “шару” дерева, тобто при максимальній глибині 14 буде 14 вузлів доказу. Кожен вузол доказу, що передається в програму, додає 32 байти до транзакції, тому для великих дерев швидко перевищується максимальний ліміт розміру транзакції. Кешування вузлів доказу в ончейн-навісі допомагає покращити сумісність програми.

Кожне з трьох значень — максимальна глибина (max depth), максимальний розмір буфера (max buffer size) та глибина навісу (canopy depth) — має свої компроміси. Збільшення будь-якого з цих параметрів призводить до збільшення розміру акаунту, який використовується для зберігання дерева, а отже, збільшує вартість його створення.

Вибір максимальної глибини (max depth) досить простий, оскільки він безпосередньо пов'язаний із кількістю листків у дереві, тобто обсягом даних, які можна зберігати. Наприклад, якщо вам потрібно 1 мільйон cNFT на одному дереві, де кожен cNFT є листком дерева, вам слід знайти максимальну глибину, яка задовольняє вираз: `2^maxDepth > 1 мільйон`. Відповідь: 20.

Вибір максимального розміру буфера (max buffer size) фактично визначає пропускну здатність: скільки одночасних записів вам потрібно? Чим більший буфер, тим вища пропускна здатність.

Нарешті, глибина навісу (canopy depth) впливатиме на можливість інтеграції вашої програми з іншими. Піонери у сфері компресії стану чітко заявляють, що відсутність навісу — це погана ідея. Наприклад, програма A не зможе викликати вашу стиснену програму B, якщо це призведе до перевищення ліміту розміру транзакції. Не забувайте, що програма A також має свої необхідні акаунти та дані, окрім необхідних шляхів доказу, кожен із яких займає місце в транзакції.

### Доступ до даних у програмі зі стисненням стану

Акаунт із стисненням стану не зберігає самі дані. Натомість він зберігає структуру одночасного дерева Меркла, про яку згадувалося раніше. Сирі дані зберігаються лише у дешевшому **стані реєстру** (ledger state) блокчейну. Це ускладнює доступ до даних, але не робить його неможливим.

The Solana ledger — це список записів, які містять підписані транзакції. У теорії, це можна простежити до самого першого блоку (genesis block). Це фактично означає, що будь-які дані, які коли-небудь були передані у транзакціях, існують у журналі.

Оскільки процес хешування під час стиснення стану відбувається ончейн, всі дані існують у стані реєстру і теоретично можуть бути отримані з оригінальної транзакції шляхом повторного програвання всього стану блокчейну від самого початку. Однак набагато простіше (хоча все ще складно) використовувати **індексатор**, який відстежує та індексує ці дані в міру виконання транзакцій. Це забезпечує наявність поза ланцюгом "кешу" даних, до якого кожен може отримати доступ і згодом верифікувати ці дані з ончейн кореневим хешем.

Цей процес є складним, але з практикою все стане зрозумілим.

## Інструменти для стиснення стану

Теорія, описана вище, є важливою для правильного розуміння стиснення стану. Однак вам не потрібно реалізовувати це з нуля. Видатні інженери вже зробили більшу частину роботи за вас у формі SPL State Compression Program та Noop Program.

### Програми SPL State Compression та Noop

Програма SPL State Compression створена для того, щоб зробити процес створення та оновлення одночасних дерев Меркла повторюваним і сумісним у всій екосистемі Solana. Вона надає інструкції для ініціалізації дерев Меркла, управління листками дерев (тобто додавання, оновлення, видалення даних) та перевірки даних листків.

Програма State Compression також використовує окрему програму «no op», основна мета якої — полегшити індексацію даних листків, записуючи їх у стан реєстру. Коли ви хочете зберігати стиснені дані, ви передаєте їх у програму State Compression, де вони хешуються і відправляються як «подія» до програми Noop. Хеш зберігається у відповідному одночасному дереві Меркла, але необроблені дані залишаються доступними через журнали транзакцій програми Noop.

### Індексація даних для зручного пошуку

За нормальних умов ви зазвичай отримуєте доступ до ончейн-даних, отримуючи відповідний акаунт. Однак при використанні стиснення стану це не так просто.

Як згадувалося вище, дані тепер існують у стані реєстру, а не в акаунті. Найзручніше місце для пошуку повних даних — це логи інструкції Noop. На жаль, хоча ці дані певною мірою існуватимуть у стані реєстру назавжди, вони, ймовірно, стануть недоступними через валідатори після певного періоду часу.

Щоб заощадити місце та забезпечити вищу продуктивність, валідатори не зберігають усі транзакції, починаючи з генезисного блоку. Конкретний період, протягом якого ви зможете отримати доступ до логів інструкцій Noop, пов’язаних із вашими даними, залежатиме від валідатора. Зрештою, ви втратите доступ до них, якщо будете покладатися безпосередньо на логи інструкцій.

Технічно, ви *можете* відтворити стан транзакції, починаючи з генезисного блоку, але середньостатистична команда цього не робитиме, і це, безумовно, не буде ефективним. [Digital Asset Standard (DAS)](https://docs.helius.dev/compression-and-das-api/digital-asset-standard-das-api) був прийнятий багатьма постачальниками RPC для забезпечення ефективних запитів стислих NFT та інших активів. Однак на момент написання статті він не підтримує довільне стиснення стану. Натомість у вас є два основні варіанти:

1. Використовувати постачальника індексації, який створить кастомне рішення для індексації вашої програми, спостерігаючи за подіями, що надсилаються до програми Noop, і зберігаючи відповідні дані офф-чейн.
2. Створити власне псевдо-рішення для індексації, яке зберігатиме дані транзакцій офф-чейн.

Для багатьох dApp варіант 2 є цілком логічним. Для більших за масштабами додатків може знадобитись спиратися на інфраструктурних постачальників для обробки індексації.

## Процес розробки за допомогою стиснення стану

### Створення типів Rust

Як і в типовій програмі на Anchor, одним із перших кроків є визначення типів Rust для вашої програми. Однак типи Rust в традиційній програмі Anchor часто представляють акаунти. У програмі зі стисненням стану, стан вашого акаунта буде зберігати лише дерево Меркла. Більш «корисна» схема даних буде серіалізована та записана до програми Noop.

Цей тип повинен містити всі дані, що зберігаються в листовому вузлі, а також будь-яку контекстуальну інформацію, необхідну для того, щоб зрозуміти ці дані. Наприклад, якщо ви створюєте просту програму для обміну повідомленнями, ваша структура `Message` може виглядати так:

```rust
#[derive(AnchorSerialize)]
pub struct MessageLog {
		leaf_node: [u8; 32], // The leaf node hash
    from: Pubkey,        // Pubkey of the message sender
		to: Pubkey,          // Pubkey of the message recipient
    message: String,     // The message to send
}

impl MessageLog {
    // Constructs a new message log from given leaf node and message
    pub fn new(leaf_node: [u8; 32], from: Pubkey, to: Pubkey, message: String) -> Self {
        Self { leaf_node, from, to, message }
    }
}
```

Щоб було абсолютно зрозуміло, **це не обліковий запис, з якого ви зможете читати дані**. Ваша програма створюватиме екземпляр цього типу з вхідних даних інструкції, а не конструюватиме екземпляр цього типу з даних облікового запису, які вона зчитує. Ми обговоримо, як читати дані, у наступному розділі.

### Ініціалізація нового дерева

Клієнти створюватимуть та ініціалізуватимуть обліковий запис дерева Меркла у двох окремих інструкціях. Перша інструкція просто виділяє обліковий запис, викликаючи System Program. Друга буде інструкцією, яку ви створите в користувацькій програмі для ініціалізації нового облікового запису. Ця ініціалізація фактично зводиться до запису максимальної глибини та розміру буфера для дерева Меркла.

Усе, що потрібно зробити цій інструкції, — це створити CPI для виклику інструкції `init_empty_merkle_tree` у State Compression Program. Оскільки це вимагає максимальну глибину та максимальний розмір буфера, ці параметри потрібно передати як аргументи до інструкції.

Пам’ятайте, що максимальна глибина стосується максимальної кількості переходів від будь-якого листка до кореня дерева. Максимальний розмір буфера стосується обсягу простору, зарезервованого для збереження журналу змін дерева. Цей журнал використовується для забезпечення того, щоб ваше дерево могло підтримувати одночасні оновлення в межах одного блоку.

Наприклад, якщо ми ініціалізуємо дерево для зберігання повідомлень між користувачами, інструкція може виглядати так:

```rust
pub fn create_messages_tree(
    ctx: Context<MessageAccounts>,
    max_depth: u32, // Max depth of the Merkle tree
    max_buffer_size: u32 // Max buffer size of the Merkle tree
) -> Result<()> {
    // Get the address for the Merkle tree account
    let merkle_tree = ctx.accounts.merkle_tree.key();
    // Define the seeds for pda signing
    let signer_seeds: &[&[&[u8]]] = &[
        &[
            merkle_tree.as_ref(), // The address of the Merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ],
    ];

    // Create cpi context for init_empty_merkle_tree instruction.
    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.compression_program.to_account_info(), // The spl account compression program
        Initialize {
            authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the Merkle tree, using a PDA
            merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be initialized
            noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
        },
        signer_seeds // The seeds for pda signing
    );

    // CPI to initialize an empty Merkle tree with given max depth and buffer size
    init_empty_merkle_tree(cpi_ctx, max_depth, max_buffer_size)?;

    Ok(())
}
```

### Додавання хешів до дерева

Після ініціалізації дерева Меркла можна починати додавати хеші даних. Це передбачає передачу некомпресованих даних до інструкції у вашій програмі, яка буде хешувати ці дані, записувати їх у програму Noop і використовувати інструкцію `append` з програми State Compression для додавання хеша до дерева. Далі детально обговорюється, що має виконувати ваша інструкція:

1. Використовуйте функцію `hashv` з бібліотеки `keccak` для хешування даних. У більшості випадків також потрібно хешувати власника або уповноважений акаунт даних, щоб гарантувати, що їх можна змінювати лише відповідним уповноваженим акаунтом.
2. Створіть об'єкт журналу, який представляє дані, які потрібно записати у програму Noop, а потім викличте `wrap_application_data_v1`, щоб виконати CPI до програми Noop з цим об'єктом. Це забезпечує доступність не стиснених даних для будь-якого клієнта, який їх шукає. Для широких випадків використання, таких як cNFT, це будуть індексатори. Ви також можете створити власного клієнта-спостерігача, щоб імітувати роботу індексаторів, але зосереджуючись на вашій конкретній програмі.
3. Створіть та виконайте CPI до інструкції `append` програми State Compression. Ця інструкція приймає хеш, обчислений на першому етапі, та додає його до наступного доступного листка у вашому дереві Меркла. Для цього обовʼязково потрібно мати адресу дерева Меркла, та уповноважений bump, який виконує роль seed-підписанта

Коли все це зібрати разом, використовуючи приклад з обміном повідомленнями, це виглядає приблизно так:

```rust
// Instruction for appending a message to a tree.
pub fn append_message(ctx: Context<MessageAccounts>, message: String) -> Result<()> {
    // Hash the message + whatever key should have update authority
    let leaf_node = keccak::hashv(&[message.as_bytes(), ctx.accounts.sender.key().as_ref()]).to_bytes();
    // Create a new "message log" using the leaf node hash, sender, receipient, and message
    let message_log = MessageLog::new(leaf_node.clone(), ctx.accounts.sender.key().clone(), ctx.accounts.receipient.key().clone(), message);
    // Log the "message log" data using noop program
    wrap_application_data_v1(message_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;
    // Get the address for the Merkle tree account
    let merkle_tree = ctx.accounts.merkle_tree.key();
    // Define the seeds for pda signing
    let signer_seeds: &[&[&[u8]]] = &[
        &[
            merkle_tree.as_ref(), // The address of the Merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ],
    ];
    // Create a new cpi context and append the leaf node to the Merkle tree.
    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.compression_program.to_account_info(), // The spl account compression program
        Modify {
            authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the Merkle tree, using a PDA
            merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be modified
            noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
        },
        signer_seeds // The seeds for pda signing
    );
    // CPI to append the leaf node to the Merkle tree
    append(cpi_ctx, leaf_node)?;
    Ok(())
}
```

### Оновлення хешів

Щоб оновити дані, потрібно створити новий хеш, який замінить хеш на відповідному листі дерева Меркла. Для цього вашій програмі потрібно мати доступ до чотирьох елементів:

1. Індекс листа, який потрібно оновити
2. Кореневий хеш дерева Меркла
3. Оригінальні дані, які ви хочете замінити
4. Оновлені дані

Маючи доступ до цих даних, інструкція програми може виконати подібні кроки, як і для додавання початкових даних до дерева:

1. **Підтвердження акаунту, що може здійснювати оновлення** - Перший крок є новим. В більшості випадків потрібно підтвердити право на оновлення. Це зазвичай включає доведення того, що підписант транзакції `update` є справжнім власником або уповноваженим акаунтом для листка на заданому індексі. Оскільки дані стиснуті у вигляді хешу на листку, ми не можемо просто порівняти публічний ключ `authority` з збереженим значенням. Натомість нам потрібно обчислити попередній хеш, використовуючи старі дані та `authority`, вказаний в структурі валідації акаунту. Потім ми будуємо та видаємо CPI для інструкції `verify_leaf` програми State Compression, використовуючи наш обчислений хеш.
2. **Хешування нових даних** - Цей крок такий самий, як і перший крок при додаванні початкових даних. Використовуйте функцію `hashv` з бібліотеки `keccak`, щоб хешувати нові дані та авторитет оновлення, кожен з яких представлений відповідним байтовим форматом.
3. **Логування нових даних** - Цей крок такий самий, як і другий крок при додаванні початкових даних. Створіть екземпляр структури журналу та викличте `wrap_application_data_v1`, щоб виконати CPI до програми Noop.
4. **Заміна існуючого хешу листка** - Цей крок дещо відрізняється від останнього кроку додавання початкових даних. Створіть і виконайте CPI до інструкції `replace_leaf` програми State Compression. Це використовує старий хеш, новий хеш та індекс листка для заміни даних листка на вказаному індексі новим хешем. Для цього обовʼязково потрібно мати адресу дерева Меркла, та уповноважений bump, який виконує роль seed-підписанта

Поєднавши все це в одну інструкцію, процес виглядатиме так:

```rust
pub fn update_message(
    ctx: Context<MessageAccounts>,
    index: u32,
    root: [u8; 32],
    old_message: String,
    new_message: String
) -> Result<()> {
    let old_leaf = keccak
        ::hashv(&[old_message.as_bytes(), ctx.accounts.sender.key().as_ref()])
        .to_bytes();

    let merkle_tree = ctx.accounts.merkle_tree.key();

    // Define the seeds for pda signing
    let signer_seeds: &[&[&[u8]]] = &[
        &[
            merkle_tree.as_ref(), // The address of the Merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ],
    ];

    // Verify Leaf
    {
        if old_message == new_message {
            msg!("Messages are the same!");
            return Ok(());
        }

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            VerifyLeaf {
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be modified
            },
            signer_seeds // The seeds for pda signing
        );
        // Verify or Fails
        verify_leaf(cpi_ctx, root, old_leaf, index)?;
    }

    let new_leaf = keccak
        ::hashv(&[new_message.as_bytes(), ctx.accounts.sender.key().as_ref()])
        .to_bytes();

    // Log out for indexers
    let message_log = MessageLog::new(new_leaf.clone(), ctx.accounts.sender.key().clone(), ctx.accounts.recipient.key().clone(), new_message);
    // Log the "message log" data using noop program
    wrap_application_data_v1(message_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;

    // replace leaf
    {
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            Modify {
                authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the Merkle tree, using a PDA
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be modified
                noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
            },
            signer_seeds // The seeds for pda signing
        );
        // CPI to append the leaf node to the Merkle tree
        replace_leaf(cpi_ctx, root, old_leaf, new_leaf, index)?;
    }

    Ok(())
}
```

### Видалення хешів

На момент написання програми State Compression не надає явної інструкції `delete`. Замість цього, вам слід оновити дані листка таким чином, щоб ці дані вказували на те, що вони є "видаленими". Конкретні дані залежать від вашого випадку використання та вимог до безпеки. Деякі можуть вирішити встановити всі дані в 0, в той час як інші можуть зберігати статичний рядок, який буде спільним для всіх "видалених" елементів.

### Доступ до даних через клієнта

Обговорення, що було вище, охоплює 3 з 4 стандартних процедур CRUD: створення, оновлення та видалення. Те, що залишилось — одна з найскладніших концепцій у стані компресії: читання даних.

Доступ до даних через клієнта є складним, оскільки дані не зберігаються у форматі, який легко отримати. Хеші даних, що зберігаються в акаунті дерева Меркла, не можуть бути використані для відновлення початкових даних, а дані, записані в програму Noop, не доступні безкінечно.

Ваші найкращі варіанти — це один з двох підходів:

1. Працювати з постачальником індексації для створення індивідуального рішення з індексації для вашої програми, а потім написати код на стороні клієнта, який буде використовувати дані індексації.
2. Створити власний псевдо-індексатор як легшу альтернативу.

Якщо ваш проект справді є децентралізованим, і багато учасників будуть взаємодіяти з вашою програмою іншими способами, окрім вашого власного фронтенду, то варіант 2 може виявитися недостатнім. Однак залежно від масштабів проекту чи того, чи матимете ви контроль над більшістю доступів до програми, це може бути життєздатним підходом.

Немає "правильного" способу це зробити. Є два потенційних підходи:

1. Зберігати сирі дані в базі даних одночасно з їх відправкою до програми, разом із листом, до якого ці дані були хешовані та збережені.
2. Створити сервер, який спостерігає за транзакціями вашої програми, шукає відповідні логи Noop, декодує їх і зберігає.

Ми застосуємо обидва підходи під час написання тестів у лабораторній роботі цього уроку (хоча ми не будемо зберігати дані в базі даних – вони існуватимуть в пам’яті лише під час виконання тестів).

Налаштування для цього є дещо стомлюючим. Маючи певну транзакцію, ви можете отримати її від RPC-провайдера, отримати внутрішні інструкції, пов’язані з програмою Noop, скористатися функцією `deserializeApplicationDataEvent` з пакета `@solana/spl-account-compression` для JS, щоб отримати логи, а потім десеріалізувати їх за допомогою Borsh. Нижче наведено приклад на основі програми для обміну повідомленнями, яку ми розглядали вище.

```tsx
export async function getMessageLog(connection: Connection, txSignature: string) {
  // Confirm the transaction, otherwise the getTransaction sometimes returns null
  const latestBlockHash = await connection.getLatestBlockhash()
  await connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: txSignature,
  })

  // Get the transaction info using the tx signature
  const txInfo = await connection.getTransaction(txSignature, {
    maxSupportedTransactionVersion: 0,
  })

  // Get the inner instructions related to the program instruction at index 0
  // We only send one instruction in test transaction, so we can assume the first
  const innerIx = txInfo!.meta?.innerInstructions?.[0]?.instructions

  // Get the inner instructions that match the SPL_NOOP_PROGRAM_ID
  const noopInnerIx = innerIx.filter(
    (instruction) =>
      txInfo?.transaction.message.staticAccountKeys[
        instruction.programIdIndex
      ].toBase58() === SPL_NOOP_PROGRAM_ID.toBase58()
  )

  let messageLog: MessageLog
  for (let i = noopInnerIx.length - 1; i >= 0; i--) {
    try {
      // Try to decode and deserialize the instruction data
      const applicationDataEvent = deserializeApplicationDataEvent(
        Buffer.from(bs58.decode(noopInnerIx[i]?.data!))
      )

      // Get the application data
      const applicationData = applicationDataEvent.fields[0].applicationData

      // Deserialize the application data into MessageLog instance
      messageLog = deserialize(
        MessageLogBorshSchema,
        MessageLog,
        Buffer.from(applicationData)
      )

      if (messageLog !== undefined) {
        break
      }
    } catch (__) {}
  }

  return messageLog
}
```

## Висновок

Реалізація узагальненої компресії стану може бути складною, але вона цілком можлива завдяки доступним інструментам. Ба більше, ці інструменти та програми будуть лише покращуватись із часом. Якщо ви знайдете рішення, які полегшують процес розробки, обов’язково поділіться ними з спільнотою!

# Лабораторна робота

Давайте попрактикуємось у використанні узагальненої компресії стану, створивши нову програму на Anchor. Ця програма буде використовувати кастомну компресію стану для роботи простого додатка для ведення нотаток.

### 1. Налаштування проєкту

Почніть із ініціалізації програми Anchor:

```bash
anchor init compressed-notes
```

Ми будемо використовувати бібліотеку `spl-account-compression` із увімкненою опцією `cpi`. Додайте її як залежність у файл `programs/compressed-notes/Cargo.toml`.

```toml
[dependencies]
anchor-lang = "0.28.0"
spl-account-compression = { version="0.2.0", features = ["cpi"] }
solana-program = "1.16.0"
```

Ми будемо проводити тестування локально, але нам потрібні програма Compression та програма Noop з Mainnet. Необхідно додати їх до файлу `Anchor.toml` у кореневій директорії, щоб вони були скопійовані до нашого локального кластеру.

```toml
[test.validator]
url = "https://api.mainnet-beta.solana.com"

[[test.validator.clone]]
address = "noopb9bkMVfRPU8AsbpTUg8AQkHtKwMYZiFUjNRtMmV"

[[test.validator.clone]]
address = "cmtDvXumGCrqC1Age74AVPhSRVXJMd8PJS91L8KbNCK"
```

Нарешті, підготуємо файл `lib.rs` для решти Демо. Видаліть інструкцію `initialize` та структуру акаунтів `Initialize`, після чого додайте імпорти, показані в наведеному нижче кодовому фрагменті (переконайтеся, що вказали ***ваш*** ідентифікатор програми):

```rust
use anchor_lang::{
    prelude::*, 
    solana_program::keccak
};
use spl_account_compression::{
    Noop,
    program::SplAccountCompression,
    cpi::{
        accounts::{Initialize, Modify, VerifyLeaf},
        init_empty_merkle_tree, verify_leaf, replace_leaf, append, 
    },
    wrap_application_data_v1, 
};

declare_id!("YOUR_KEY_GOES_HERE");

// STRUCTS GO HERE

#[program]
pub mod compressed_notes {
    use super::*;

	// FUNCTIONS GO HERE
	
}
```

Для решти цієї демонстрації ми будемо вносити зміни безпосередньо у програмний код у файлі `lib.rs`. Це трохи спрощує пояснення. Ви можете змінювати структуру програми на свій розсуд.

Не соромтеся виконати збірку перед тим, як продовжити. Це дозволить переконатися, що ваше середовище працює належним чином, і зменшить час на майбутню роботу.

### 2. Визначення схеми `Note`

Наступним кроком ми визначимо, як виглядає запис у нашій програмі. Записи повинні мати такі властивості:

- `leaf_node` — це масив з 32 байт, що представляє хеш, збережений на вузлі листа.
- `owner` — публічний ключ власника запису.
- `note` — рядкове представлення запису.

```rust
#[derive(AnchorSerialize)]
pub struct NoteLog {
    leaf_node: [u8; 32],  // The leaf node hash
    owner: Pubkey,        // Pubkey of the note owner
    note: String,         // The note message
}

impl NoteLog {
    // Constructs a new note from given leaf node and message
    pub fn new(leaf_node: [u8; 32], owner: Pubkey, note: String) -> Self {
        Self { leaf_node, owner, note }
    }
}
```

У традиційній програмі Anchor це була б структура акаунта, але оскільки ми використовуємо стиснення стану, наші акаунти не відображатимуть наші нативні структури. Оскільки нам не потрібна вся функціональність акаунта, ми можемо використати макрос `AnchorSerialize` замість макроса `account`.

### 3. Визначення акаунтів введення та обмежень

Як не дивно, всі наші інструкції будуть використовувати однакові акаунти. Ми створимо єдину структуру `NoteAccounts` для валідації акаунтів. Вона потребує наступних акаунтів:

- `owner` - це творець та власник запису; має бути підписантом транзакції.
- `tree_authority` - уповноважений акаунт для дерева Меркла; використовується для підписання CPIs, пов'язаних зі стисненням.
- `merkle_tree` - адреса дерева Меркла, яке використовується для зберігання хешів записів; буде неперевіреним, оскільки воно перевіряється програмою State Compression.
- `log_wrapper` - адреса програми Noop.
- `compression_program` - адреса програми State Compression.

```rust
#[derive(Accounts)]
pub struct NoteAccounts<'info> {
    // The payer for the transaction
    #[account(mut)]
    pub owner: Signer<'info>,

    // The pda authority for the Merkle tree, only used for signing
    #[account(
        seeds = [merkle_tree.key().as_ref()],
        bump,
    )]
    pub tree_authority: SystemAccount<'info>,

    // The Merkle tree account
    /// CHECK: This account is validated by the spl account compression program
    #[account(mut)]
    pub merkle_tree: UncheckedAccount<'info>,

    // The noop program to log data
    pub log_wrapper: Program<'info, Noop>,

    // The spl account compression program
    pub compression_program: Program<'info, SplAccountCompression>,
}
```

### 4. Створення інструкції `create_note_tree`

Далі давайте створимо інструкцію `create_note_tree`. Пам’ятайте, що клієнти вже виділили акаунт для дерева Меркла, але використовуватимуть цю інструкцію для його ініціалізації.

Ця інструкція повинна лише побудувати CPI для виклику інструкції `init_empty_merkle_tree` в програмі State Compression. Для цього їй потрібні акаунти, вказані в структурі валідації акаунтів `NoteAccounts`. Також необхідно передати два додаткові аргументи:

1. `max_depth` - максимальна глибина дерева Меркла
2. `max_buffer_size` - максимальний розмір буфера дерева Меркла

Ці значення необхідні для ініціалізації даних на акаунті дерева Меркла. Пам'ятайте, що максимальна глибина вказує на максимальну кількість переходів від будь-якого листка до кореня дерева. Максимальний розмір буфера вказує на кількість місця, зарезервованого для зберігання журналу змін дерева. Цей журнал змін використовується для забезпечення підтримки одночасних оновлень у межах одного блоку.

```rust
#[program]
pub mod compressed_notes {
    use super::*;

    // Instruction for creating a new note tree.
    pub fn create_note_tree(
        ctx: Context<NoteAccounts>,
        max_depth: u32,       // Max depth of the Merkle tree
        max_buffer_size: u32, // Max buffer size of the Merkle tree
    ) -> Result<()> {
        // Get the address for the Merkle tree account
        let merkle_tree = ctx.accounts.merkle_tree.key();

        // Define the seeds for pda signing
        let signer_seeds: &[&[&[u8]]] = &[&[
            merkle_tree.as_ref(), // The address of the Merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ]];

        // Create cpi context for init_empty_merkle_tree instruction.
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            Initialize {
                authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the Merkle tree, using a PDA
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be initialized
                noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
            },
            signer_seeds, // The seeds for pda signing
        );

        // CPI to initialize an empty Merkle tree with given max depth and buffer size
        init_empty_merkle_tree(cpi_ctx, max_depth, max_buffer_size)?;
        Ok(())
    }

    //...
}
```

Переконайтесь, що ваші сіди підписанта для CPI включають як адресу дерева Меркла, так і bump-значення уповноваженого акаунту дерева.

### 5. Створення інструкції `append_note`

Тепер давайте створимо нашу інструкцію `append_note`. Ця інструкція повинна приймати сирий запис як рядок і стискати його в хеш, який ми зберігатимемо в дереві Меркла. Також ми будемо записувати нотатки до програми Noop, щоб усі дані існували в стані ланцюга.

Кроки для цього процесу такі:

1. Use the `hashv` function from the `keccak` crate to hash the note and owner, each as their corresponding byte representation. It’s ***crucial*** that you hash the owner as well as the note. This is how we’ll verify note ownership before updates in the update instruction.
2. Create an instance of the `NoteLog` struct using the hash from step 1, the owner’s public key, and the raw note as a String. Then call `wrap_application_data_v1` to issue a CPI to the Noop program, passing the instance of `NoteLog`. This ensures the entirety of the note (not just the hash) is readily available to any client looking for it. For broad use cases like cNFTs, that would be indexers. You might create your observing client to simulate what indexers are doing but for your own application.
3. Build and issue a CPI to the State Compression Program’s `append` instruction. This takes the hash computed in step 1 and adds it to the next available leaf on your Merkle tree. Just as before, this requires the Merkle tree address and the tree authority bump as signature seeds.

```rust
#[program]
pub mod compressed_notes {
    use super::*;

    //...

    // Instruction for appending a note to a tree.
    pub fn append_note(ctx: Context<NoteAccounts>, note: String) -> Result<()> {
        // Hash the "note message" which will be stored as leaf node in the Merkle tree
        let leaf_node =
            keccak::hashv(&[note.as_bytes(), ctx.accounts.owner.key().as_ref()]).to_bytes();
        // Create a new "note log" using the leaf node hash and note.
        let note_log = NoteLog::new(leaf_node.clone(), ctx.accounts.owner.key().clone(), note);
        // Log the "note log" data using noop program
        wrap_application_data_v1(note_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;
        // Get the address for the Merkle tree account
        let merkle_tree = ctx.accounts.merkle_tree.key();
        // Define the seeds for pda signing
        let signer_seeds: &[&[&[u8]]] = &[&[
            merkle_tree.as_ref(), // The address of the Merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ]];
        // Create a new cpi context and append the leaf node to the Merkle tree.
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.compression_program.to_account_info(), // The spl account compression program
            Modify {
                authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the Merkle tree, using a PDA
                merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be modified
                noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
            },
            signer_seeds, // The seeds for pda signing
        );
        // CPI to append the leaf node to the Merkle tree
        append(cpi_ctx, leaf_node)?;
        Ok(())
    }

    //...
}
```

### 6. Create `update_note` instruction

The last instruction we’ll make is the `update_note` instruction. This should replace an existing leaf with a new hash representing the new updated note data.

For this to work, we’ll need the following parameters:

1. `index` - the index of the leaf we are going to update
2. `root` - the root hash of the Merkle tree
3. `old_note` - the string representation of the old note we’re updating
4. `new_note` - the string representation of the new note we want to update to

Remember, the steps here are similar to `append_note`, but with some minor additions and modifications:

1. The first step is new. We need to first prove that the `owner` calling this function is the true owner of the leaf at the given index. Since the data is compressed as a hash on the leaf, we can’t simply compare the `owner` public key to a stored value. Instead, we need to compute the previous hash using the old note data and the `owner` listed in the account validation struct. We then build and issue a CPI to the State Compression Program’s `verify_leaf` instruction using our computed hash.
2. This step is the same as the first step from creating the `append_note` instruction. Use the `hashv` function from the `keccak` crate to hash the new note and its owner, each as their corresponding byte representation.
3. This step is the same as the second step from creating the `append_note` instruction. Create an instance of the `NoteLog` struct using the hash from step 2, the owner’s public key, and the new note as a string. Then call `wrap_application_data_v1` to issue a CPI to the Noop program, passing the instance of `NoteLog`
4. This step is slightly different than the last step from creating the `append_note` instruction. Build and issue a CPI to the State Compression Program’s `replace_leaf` instruction. This uses the old hash, the new hash, and the leaf index to replace the data of the leaf at the given index with the new hash. Just as before, this requires the Merkle tree address and the tree authority bump as signature seeds.

```rust
#[program]
pub mod compressed_notes {
    use super::*;

    //...

		pub fn update_note(
        ctx: Context<NoteAccounts>,
        index: u32,
        root: [u8; 32],
        old_note: String,
        new_note: String,
    ) -> Result<()> {
        let old_leaf =
            keccak::hashv(&[old_note.as_bytes(), ctx.accounts.owner.key().as_ref()]).to_bytes();

        let merkle_tree = ctx.accounts.merkle_tree.key();

        // Define the seeds for pda signing
        let signer_seeds: &[&[&[u8]]] = &[&[
            merkle_tree.as_ref(), // The address of the Merkle tree account as a seed
            &[*ctx.bumps.get("tree_authority").unwrap()], // The bump seed for the pda
        ]];

        // Verify Leaf
        {
            if old_note == new_note {
                msg!("Notes are the same!");
                return Ok(());
            }

            let cpi_ctx = CpiContext::new_with_signer(
                ctx.accounts.compression_program.to_account_info(), // The spl account compression program
                VerifyLeaf {
                    merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be modified
                },
                signer_seeds, // The seeds for pda signing
            );
            // Verify or Fails
            verify_leaf(cpi_ctx, root, old_leaf, index)?;
        }

        let new_leaf =
            keccak::hashv(&[new_note.as_bytes(), ctx.accounts.owner.key().as_ref()]).to_bytes();

        // Log out for indexers
        let note_log = NoteLog::new(new_leaf.clone(), ctx.accounts.owner.key().clone(), new_note);
        // Log the "note log" data using noop program
        wrap_application_data_v1(note_log.try_to_vec()?, &ctx.accounts.log_wrapper)?;

        // replace leaf
        {
            let cpi_ctx = CpiContext::new_with_signer(
                ctx.accounts.compression_program.to_account_info(), // The spl account compression program
                Modify {
                    authority: ctx.accounts.tree_authority.to_account_info(), // The authority for the Merkle tree, using a PDA
                    merkle_tree: ctx.accounts.merkle_tree.to_account_info(), // The Merkle tree account to be modified
                    noop: ctx.accounts.log_wrapper.to_account_info(), // The noop program to log data
                },
                signer_seeds, // The seeds for pda signing
            );
            // CPI to append the leaf node to the Merkle tree
            replace_leaf(cpi_ctx, root, old_leaf, new_leaf, index)?;
        }

        Ok(())
    }
}
```

### 7. Client test setup

We’re going to write a few tests to ensure that our program works as expected. First, let’s do some setup.

We’ll be using the `@solana/spl-account-compression` package. Go ahead and install it:

```bash
yarn add @solana/spl-account-compression
```

Next, we’re going to give you the contents of a utility file we’ve created to make testing easier. Create a `utils.ts` file in the `tests` directory, add in the below, then we’ll explain it. 

```tsx
import {
  SPL_NOOP_PROGRAM_ID,
  deserializeApplicationDataEvent,
} from "@solana/spl-account-compression"
import { Connection, PublicKey } from "@solana/web3.js"
import { bs58 } from "@coral-xyz/anchor/dist/cjs/utils/bytes"
import { deserialize } from "borsh"
import { keccak256 } from "js-sha3"

class NoteLog {
  leafNode: Uint8Array
  owner: PublicKey
  note: string

  constructor(properties: {
    leafNode: Uint8Array
    owner: Uint8Array
    note: string
  }) {
    this.leafNode = properties.leafNode
    this.owner = new PublicKey(properties.owner)
    this.note = properties.note
  }
}

// A map that describes the Note structure for Borsh deserialization
const NoteLogBorshSchema = new Map([
  [
    NoteLog,
    {
      kind: "struct",
      fields: [
        ["leafNode", [32]], // Array of 32 `u8`
        ["owner", [32]], // Pubkey
        ["note", "string"],
      ],
    },
  ],
])

export function getHash(note: string, owner: PublicKey) {
  const noteBuffer = Buffer.from(note)
  const publicKeyBuffer = Buffer.from(owner.toBytes())
  const concatenatedBuffer = Buffer.concat([noteBuffer, publicKeyBuffer])
  const concatenatedUint8Array = new Uint8Array(
    concatenatedBuffer.buffer,
    concatenatedBuffer.byteOffset,
    concatenatedBuffer.byteLength
  )
  return keccak256(concatenatedUint8Array)
}

export async function getNoteLog(connection: Connection, txSignature: string) {
  // Confirm the transaction, otherwise the getTransaction sometimes returns null
  const latestBlockHash = await connection.getLatestBlockhash()
  await connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: txSignature,
  })

  // Get the transaction info using the tx signature
  const txInfo = await connection.getTransaction(txSignature, {
    maxSupportedTransactionVersion: 0,
  })

  // Get the inner instructions related to the program instruction at index 0
  // We only send one instruction in test transaction, so we can assume the first
  const innerIx = txInfo!.meta?.innerInstructions?.[0]?.instructions

  // Get the inner instructions that match the SPL_NOOP_PROGRAM_ID
  const noopInnerIx = innerIx.filter(
    (instruction) =>
      txInfo?.transaction.message.staticAccountKeys[
        instruction.programIdIndex
      ].toBase58() === SPL_NOOP_PROGRAM_ID.toBase58()
  )

  let noteLog: NoteLog
  for (let i = noopInnerIx.length - 1; i >= 0; i--) {
    try {
      // Try to decode and deserialize the instruction data
      const applicationDataEvent = deserializeApplicationDataEvent(
        Buffer.from(bs58.decode(noopInnerIx[i]?.data!))
      )

      // Get the application data
      const applicationData = applicationDataEvent.fields[0].applicationData

      // Deserialize the application data into NoteLog instance
      noteLog = deserialize(
        NoteLogBorshSchema,
        NoteLog,
        Buffer.from(applicationData)
      )

      if (noteLog !== undefined) {
        break
      }
    } catch (__) {}
  }

  return noteLog
}
```

There are 3 main things in the above file:

1. `NoteLog` - a class representing the note log we’ll find in the Noop program logs. We’ve also added the borsh schema as `NoteLogBorshSchema` for deserialization.
2. `getHash` - a function that creates a hash of the note and note owner so we can compare it to what we find on the Merkle tree
3. `getNoteLog` - a function that looks through the provided transaction’s logs, finds the Noop program logs, then deserializes and returns the corresponding Note log.

### 8. Write client tests

Now that we’ve got our packages installed and utility file ready, let’s dig into the tests themselves. We’re going to create four of them:

1. Create Note Tree - this will create the Merkle tree we’ll be using to store note hashes
2. Add Note - this will call our `append_note` instruction
3. Add Max Size Note - this will call our `append_note` instruction with a note that maxes out the 1232 bytes allowed in a single transaction
4. Update First Note - this will call our `update_note` instruction to modify the first note we added

The first test is mostly just for setup. In the last three tests, we’ll be asserting each time that the note hash on the tree matches what we would expect given the note text and signer.

Let’s start with our imports. There are quite a few from Anchor, `@solana/web3.js`, `@solana/spl-account-compression`, and our own utils file.

```tsx
import * as anchor from "@coral-xyz/anchor"
import { Program } from "@coral-xyz/anchor"
import { CompressedNotes } from "../target/types/compressed_notes"
import {
  Keypair,
  Transaction,
  PublicKey,
  sendAndConfirmTransaction,
  Connection,
} from "@solana/web3.js"
import {
  ValidDepthSizePair,
  createAllocTreeIx,
  SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
  SPL_NOOP_PROGRAM_ID,
  ConcurrentMerkleTreeAccount,
} from "@solana/spl-account-compression"
import { getHash, getNoteLog } from "./utils"
import { assert } from "chai"
```

Next, we’ll want to set up the state variables we’ll be using throughout our tests. This includes the default Anchor setup as well as generating a Merkle tree keypair, the tree authority, and some notes.

```tsx
describe("compressed-notes", () => {
  const provider = anchor.AnchorProvider.env()
  anchor.setProvider(provider)
  const connection = new Connection(
    provider.connection.rpcEndpoint,
    "confirmed" // has to be confirmed for some of the methods below
  )

  const wallet = provider.wallet as anchor.Wallet
  const program = anchor.workspace.CompressedNotes as Program<CompressedNotes>

  // Generate a new keypair for the Merkle tree account
  const merkleTree = Keypair.generate()

  // Derive the PDA to use as the tree authority for the Merkle tree account
  // This is a PDA derived from the Note program, which allows the program to sign for appends instructions to the tree
  const [treeAuthority] = PublicKey.findProgramAddressSync(
    [merkleTree.publicKey.toBuffer()],
    program.programId
  )

	const firstNote = "hello world"
  const secondNote = "0".repeat(917)
  const updatedNote = "updated note"


  // TESTS GO HERE

});
```

Finally, let’s start with the tests themselves. First the `Create Note Tree` test. This test will do two things:

1. Allocate a new account for the Merkle tree with a max depth of 3, max buffer size of 8, and canopy depth of 0
2. Initialize this new account using our program’s `createNoteTree` instruction

```tsx
it("Create Note Tree", async () => {
  const maxDepthSizePair: ValidDepthSizePair = {
    maxDepth: 3,
    maxBufferSize: 8,
  }

  const canopyDepth = 0

  // instruction to create new account with required space for tree
  const allocTreeIx = await createAllocTreeIx(
    connection,
    merkleTree.publicKey,
    wallet.publicKey,
    maxDepthSizePair,
    canopyDepth
  )

  // instruction to initialize the tree through the Note program
  const ix = await program.methods
    .createNoteTree(maxDepthSizePair.maxDepth, maxDepthSizePair.maxBufferSize)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .instruction()

  const tx = new Transaction().add(allocTreeIx, ix)
  await sendAndConfirmTransaction(connection, tx, [wallet.payer, merkleTree])
})
```

Next, we’ll create the `Add Note` test. It should call `append_note` with `firstNote`, then check that the onchain hash matches our computed hash and that the note log matches the text of the note we passed into the instruction.

```tsx
it("Add Note", async () => {
  const txSignature = await program.methods
    .appendNote(firstNote)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .rpc()
  
  const noteLog = await getNoteLog(connection, txSignature)
  const hash = getHash(firstNote, provider.publicKey)
  
  assert(hash === Buffer.from(noteLog.leafNode).toString("hex"))
  assert(firstNote === noteLog.note)
})
```

Next, we’ll create the `Add Max Size Note` test. It is the same as the previous test, but with the second note. 

```tsx
it("Add Max Size Note", async () => {
  // Size of note is limited by max transaction size of 1232 bytes, minus additional data required for the instruction
  const txSignature = await program.methods
    .appendNote(secondNote)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .rpc()
  
  const noteLog = await getNoteLog(connection, txSignature)
  const hash = getHash(secondNote, provider.publicKey)
  
  assert(hash === Buffer.from(noteLog.leafNode).toString("hex"))
  assert(secondNote === noteLog.note)
})
```

Lastly, we’ll create the `Update First Note` test. This is slightly more complex than adding a note. We’ll do the following:

1. Get the Merkle tree root as it’s required by the instruction.
2. Call the `update_note` instruction of our program, passing in the index 0 (for the first note), the Merkle tree root, the first note, and the updated data. Remember, it needs the first note and the root because the program must verify the entire proof path for the note’s leaf before it can be updated. 

```tsx
it("Update First Note", async () => {
  const merkleTreeAccount =
    await ConcurrentMerkleTreeAccount.fromAccountAddress(
      connection,
      merkleTree.publicKey
    )
  
  const rootKey = merkleTreeAccount.tree.changeLogs[0].root
  const root = Array.from(rootKey.toBuffer())

  const txSignature = await program.methods
    .updateNote(0, root, firstNote, updatedNote)
    .accounts({
      merkleTree: merkleTree.publicKey,
      treeAuthority: treeAuthority,
      logWrapper: SPL_NOOP_PROGRAM_ID,
      compressionProgram: SPL_ACCOUNT_COMPRESSION_PROGRAM_ID,
    })
    .rpc()
  
  const noteLog = await getNoteLog(connection, txSignature)
  const hash = getHash(updatedNote, provider.publicKey)
  
  assert(hash === Buffer.from(noteLog.leafNode).toString("hex"))
  assert(updatedNote === noteLog.note)
})
```

That’s it, congrats! Go ahead and run `anchor test` and you should get four passing tests.

If you’re running into issues, feel free to go back through some of the demo or look at the full solution code in the [Compressed Notes repository](https://github.com/unboxed-software/anchor-compressed-notes). 

# Challenge

Now that you’ve practiced the basics of state compression, add a new instruction to the Compressed Notes program. This new instruction should allow users to delete an existing note. keep in mind that you can’t remove a leaf from the tree, so you’ll need to decide what “deleted” looks like for your program. Good luck!

If you'd like a very simple example of a delete function, check out the [`solution` branch on GitHub](https://github.com/Unboxed-Software/anchor-compressed-notes/tree/solution).

## Completed the lab?

Push your code to GitHub and [tell us what you thought of this lesson](https://form.typeform.com/to/IPH0UGz7#answers-lesson=60f6b072-eaeb-469c-b32e-5fea4b72d1d1)!
