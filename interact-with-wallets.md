---
назва: Взаємодія з гаманцями**
завдання:
- Пояснити, що таке гаманці
- Встановити розширення Phantom
- Налаштувати гаманець Phantom на [Devnet](https://api.devnet.solana.com/)
- Використовувати Wallet Adapter для підписання транзакцій користувачами
---

# Стислий виклад

- **Гаманці** зберігають ваш секретний ключ і здійснюють безпечне підписання транзакцій.
- **Апаратні гаманці** зберігають ваш секретний ключ на окремому пристрої.
- **Програмні гаманці** використовують ваш комп'ютер для безпечного зберігання.
- Програмні гаманці часто є **розширеннями для браузера**, які полегшують підключення до вебсайтів.
- **Бібліотека Wallet-Adapter Solana** спрощує підтримку розширень гаманців для браузерів, дозволяючи створювати вебсайти, які можуть запитувати адресу гаманця користувача та пропонувати транзакції для їх підписання.

# Урок

## Гаманці

У попередніх двох уроках ми обговорювали пари ключів. Пари ключів використовуються для пошуку акаунтів і підписання транзакцій. Публічним ключем абсолютно безпечно ділитися, але секретний ключ завжди слід зберігати в безпечному місці. Якщо секретний ключ користувача буде розкритий, зловмисник зможе вивести всі активи з акаунту і виконувати транзакції від імені цього користувача.

Термін "гаманець" відноситься до будь-якого засобу зберігання секретного ключа для його безпеки. Ці варіанти безпечного зберігання можна загалом описати як "апаратні" або "програмні" гаманці. Апаратні гаманці — це пристрої зберігання, які відокремлені від вашого комп'ютера. Програмні гаманці — це додатки, які ви можете встановити на своїх існуючих пристроях.

Програмні гаманці часто мають форму розширення для браузера. Це дозволяє вебсайтам легко взаємодіяти з гаманцем. Такі взаємодії зазвичай обмежуються:

1. Перегляд публічного ключа гаманця (адреси)
2. Надсилання транзакцій для затвердження користувачем
3. Відправка затвердженої транзакції в мережу

Як тільки транзакція надіслана, кінцевий користувач може "підтвердити" транзакцію та відправити її в мережу зі своїм "підписом".

Підписання транзакцій вимагає використання вашого секретного ключа. Дозволяючи сайту надіслати транзакцію до вашого гаманця і дозволяючи гаманцю обробляти підписання, ви гарантуєте, що ваш секретний ключ ніколи не буде розкритий сайту. Замість цього ви ділитесь секретним ключем лише з програмою гаманця.

Якщо ви не створюєте додаток для гаманця, ваш код не повинен запитувати у користувача їхній секретний ключ. Замість цього ви можете попросити користувачів підключитися до вашого сайту, використовуючи надійний гаманець.

## Phantom Wallet

Один з найпоширеніших програмних гаманців в екосистемі Solana — це [Phantom](https://phantom.app). Phantom підтримує кілька популярних браузерів і має мобільний додаток для підключення в будь-якому місці. Ви, ймовірно, захочете, щоб ваші децентралізовані додатки підтримували кілька гаманців, але цей курс буде зосереджено на Phantom.

## WalletAdapter для Solana

WalletAdapter для Solana — це набір модульних пакетів, які ви можете використовувати для спрощення процесу підтримки розширень гаманців для браузера.

Основна функціональність знаходиться в `@solana/wallet-adapter-base` і `@solana/wallet-adapter-react`.

Додаткові пакети надають компоненти для популярних UI фреймворків. В цьому уроці та протягом всього курсу ми будемо використовувати компоненти з `@solana/wallet-adapter-react-ui`.

Нарешті, деякі пакети є адаптерами для конкретних додатків гаманців. Тепер вони більше не є необхідними в більшості випадків — дивіться нижче.

### Встановлення бібліотек Wallet-Adapter

Коли ви додаєте підтримку гаманців до існуючого додатку React, почніть з встановлення відповідних пакетів. Вам знадобляться `@solana/wallet-adapter-base`, `@solana/wallet-adapter-react`. Якщо ви плануєте використовувати надані React компоненти, також потрібно додати `@solana/wallet-adapter-react-ui`.

Усі гаманці, які підтримують [Wallet Standard](https://github.com/wallet-standard/wallet-standard), підтримуються з коробки, і всі популярні гаманці Solana підтримують Wallet Standard. Однак, якщо ви хочете додати підтримку гаманців, які не підтримують цей стандарт, додайте відповідний пакет для них.

```
npm install @solana/wallet-adapter-base \
    @solana/wallet-adapter-react \
    @solana/wallet-adapter-react-ui
```

### Підключення до гаманців

`@solana/wallet-adapter-react` дозволяє зберігати та отримувати стан підключення до гаманця через хуки та контекст-постачальники, а саме:

- `useWallet`
- `WalletProvider`
- `useConnection`
- `ConnectionProvider`

Щоб це працювало належним чином, будь-яке використання `useWallet` і `useConnection` слід обгорнути в `WalletProvider` і `ConnectionProvider`. Один із найкращих способів забезпечити це — обгорнути весь ваш додаток у `ConnectionProvider` і `WalletProvider`:

```tsx
import { NextPage } from "next";
import { FC, ReactNode } from "react";
import {
  ConnectionProvider,
  WalletProvider,
} from "@solana/wallet-adapter-react";
import * as web3 from "@solana/web3.js";

export const Home: NextPage = (props) => {
  const endpoint = web3.clusterApiUrl("devnet");
  const wallets = useMemo(() => [], []);

  return (
    <ConnectionProvider endpoint={endpoint}>
      <WalletProvider wallets={wallets}>
        <p>Put the rest of your app here</p>
      </WalletProvider>
    </ConnectionProvider>
  );
};
```

Зверніть увагу, що `ConnectionProvider` вимагає властивість `endpoint`, а `WalletProvider` — властивість `wallets`. Ми продовжуємо використовувати кінцеву точку для кластера Devnet, і оскільки всі основні гаманці Solana підтримують Wallet Standard, нам не потрібні адаптери для конкретних гаманців.
На цьому етапі ви можете підключитися за допомогою `wallet.connect()`, що змусить гаманець запросити у користувача дозвіл на перегляд його публічного ключа та підтвердження транзакцій.

![wallet connection prompt](../assets/wallet-connect-prompt.png)

Хоча ви могли б реалізувати це в хуку `useEffect`, зазвичай краще надати більш розширену функціональність. Наприклад, ви можете дозволити користувачам вибирати гаманець зі списку підтримуваних додатків або відключатися після підключення.

### `@solana/wallet-adapter-react-ui`

Ви можете створити власні компоненти для цього або скористатися готовими компонентами з `@solana/wallet-adapter-react-ui`. Найпростіший спосіб надати широкий вибір опцій — використати `WalletModalProvider` і `WalletMultiButton`:

```tsx
import { NextPage } from "next";
import { FC, ReactNode } from "react";
import {
  ConnectionProvider,
  WalletProvider,
} from "@solana/wallet-adapter-react";
import {
  WalletModalProvider,
  WalletMultiButton,
} from "@solana/wallet-adapter-react-ui";
import * as web3 from "@solana/web3.js";

const Home: NextPage = (props) => {
  const endpoint = web3.clusterApiUrl("devnet");
  const wallets = useMemo(() => [], []);

  return (
    <ConnectionProvider endpoint={endpoint}>
      <WalletProvider wallets={wallets}>
        <WalletModalProvider>
          <WalletMultiButton />
          <p>Put the rest of your app here</p>
        </WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
};

export default Home;

```

`WalletModalProvider` додає функціональність для відображення модального вікна, у якому користувач може вибрати гаманець для підключення. `WalletMultiButton` змінює свою поведінку залежно від статусу підключення:

![multi button select wallet option](../assets/multi-button-select-wallet.png)

![connect wallet modal](../assets/connect-wallet-modal.png)

![multi button connect options](../assets/multi-button-connect.png)

![multi button connected state](../assets/multi-button-connected.png)

Ви також можете використовувати більш дрібні компоненти, якщо вам потрібна більш специфічна функціональність:

- `WalletConnectButton`
- `WalletModal`
- `WalletModalButton`
- `WalletDisconnectButton`
- `WalletIcon`

### Отримання інформації про акаунт  

Після підключення сайту до гаманця хук `useConnection` поверне об’єкт `Connection`, а `useWallet` — об’єкт `WalletContextState`. Властивість `publicKey` у `WalletContextState` має значення `null`, якщо гаманець не підключений, і отримує публічний ключ акаунта користувача після підключення. Маючи публічний ключ і з’єднання, ви можете отримувати інформацію про акаунт та інші дані.

```tsx
import { useConnection, useWallet } from "@solana/wallet-adapter-react";
import { LAMPORTS_PER_SOL } from "@solana/web3.js";
import { FC, useEffect, useState } from "react";

export const BalanceDisplay: FC = () => {
  const [balance, setBalance] = useState(0);
  const { connection } = useConnection();
  const { publicKey } = useWallet();

  useEffect(() => {
    if (!connection || !publicKey) {
      return;
    }

    connection.onAccountChange(
      publicKey,
      (updatedAccountInfo) => {
        setBalance(updatedAccountInfo.lamports / LAMPORTS_PER_SOL);
      },
      "confirmed",
    );

    connection.getAccountInfo(publicKey).then((info) => {
      setBalance(info.lamports);
    });
  }, [connection, publicKey]);

  return (
    <div>
      <p>{publicKey ? `Balance: ${balance / LAMPORTS_PER_SOL} SOL` : ""}</p>
    </div>
  );
};
```

Зверніть увагу на виклик `connection.onAccountChange()`, який оновлює баланс акаунта після підтвердження транзакції мережею.

### Надсилання транзакцій  

`WalletContextState` також надає функцію `sendTransaction`, яку можна використовувати для надсилання транзакцій на підтвердження.

```tsx
const { publicKey, sendTransaction } = useWallet();
const { connection } = useConnection();

const sendSol = (event) => {
  event.preventDefault();

  const transaction = new web3.Transaction();
  const recipientPubKey = new web3.PublicKey(event.target.recipient.value);

  const sendSolInstruction = web3.SystemProgram.transfer({
    fromPubkey: publicKey,
    toPubkey: recipientPubKey,
    lamports: LAMPORTS_PER_SOL * 0.1,
  });

  transaction.add(sendSolInstruction);
  sendTransaction(transaction, connection).then((sig) => {
    console.log(sig);
  });
};

```

Коли ця функція викликана, підключений гаманець відобразить транзакцію для підтвердження користувачем. Якщо користувач схвалить її, транзакція буде надіслана.

![wallet transaction approval prompt](../assets/wallet-transaction-approval-prompt.png)

# Лабораторна робота 

Давайте візьмемо Ping-програму з попереднього уроку та створимо фронтенд, який дозволить користувачам підтверджувати транзакцію для її виклику. Нагадаємо, що публічний ключ програми — `ChT1B39WKLS8qUrkLvFDXMhEJ4F1XZzwUNHUt4AU9aVa`, а публічний ключ акаунта з даними — `Ah9K7dQ8EHaZqcAsgBW8w37yN2eAy3koFmUn4x3CJtod`.

![Solana Ping App](../assets/solana-ping-app.png)

### 1. Завантажте розширення Phantom для браузера та переключіть його в режим Devnet  

Якщо у вас його ще немає, завантажте [розширення Phantom](https://phantom.app/download) для браузера. На момент написання цього уроку його підтримує Chrome, Brave, Firefox і Edge, тож вам знадобиться один із цих браузерів. Дотримуйтесь інструкцій Phantom для створення нового акаунта та гаманця.

Коли гаманець створено, натисніть значок налаштувань у нижньому правому куті інтерфейсу Phantom. Прокрутіть вниз, знайдіть пункт “Change Network” і виберіть “Devnet”. Це забезпечить підключення Phantom до тієї ж мережі, яку ми використовуватимемо в цій лабораторній роботі .

### 2. Завантажте початковий код  

Завантажте [початковий код для цього проєкту](https://github.com/Unboxed-Software/solana-ping-frontend/tree/starter). Це простий Next.js-застосунок, який наразі майже порожній, окрім компонента `AppBar`. Ми будемо поступово доповнювати його впродовж цієї лабораторної роботи.

Ви можете переглянути поточний стан проекту, виконавши в консолі команду `npm run dev`.

### 3. Обгорніть застосунок у контекстні провайдери  

Спочатку ми створимо новий компонент, який міститиме різні провайдери Wallet-Adapter, що нам знадобляться. Створіть новий файл у папці `components` з назвою `WalletContextProvider.tsx`.  

Почнемо з базової структури функціонального компонента:

```tsx
import { FC, ReactNode } from "react";

const WalletContextProvider: FC<{ children: ReactNode }> = ({ children }) => {
  return (

  ));
};

export default WalletContextProvider;
```

Щоб правильно підключитися до гаманця користувача, нам потрібні `ConnectionProvider`, `WalletProvider` і `WalletModalProvider`. Почніть з імпорту цих компонентів із `@solana/wallet-adapter-react` і `@solana/wallet-adapter-react-ui`. Потім додайте їх до компонента `WalletContextProvider`. Зверніть увагу, що `ConnectionProvider` потребує параметра `endpoint`, а `WalletProvider` — масиву `wallets`. Поки що просто використовуйте порожній рядок і порожній масив відповідно.

```tsx
import { FC, ReactNode } from "react";
import {
  ConnectionProvider,
  WalletProvider,
} from "@solana/wallet-adapter-react";
import { WalletModalProvider } from "@solana/wallet-adapter-react-ui";

const WalletContextProvider: FC<{ children: ReactNode }> = ({ children }) => {
  return (
    <ConnectionProvider endpoint={""}>
      <WalletProvider wallets={[]}>
        <WalletModalProvider>{children}</WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
};

export default WalletContextProvider;

```

Останнє, що нам потрібно, це фактична кінцева точка для `ConnectionProvider` та підтримувані гаманці для `WalletProvider`.

Для кінцевої точки ми використаємо ту ж функцію `clusterApiUrl` з бібліотеки `@solana/web3.js`, яку ми використовували раніше, тому її потрібно імпортувати. Для масиву гаманців також потрібно імпортувати бібліотеку `@solana/wallet-adapter-wallets`.

Після імпорту цих бібліотек створіть константу `endpoint`, яка використовує функцію `clusterApiUrl` для отримання URL для Devnet. Потім створіть константу з назвою `wallets` і встановіть її як порожній масив — оскільки всі гаманці підтримують Wallet Standard, більше не потрібно використовувати власні адаптери для гаманців. Нарешті, замініть порожній рядок та порожній масив у `ConnectionProvider` та `WalletProvider` відповідно.

Щоб завершити цей компонент, додайте `require('@solana/wallet-adapter-react-ui/styles.css');` нижче імпортів, щоб забезпечити правильне стилізування та функціональність компонентів бібліотеки Wallet Adapter.

```tsx
import { FC, ReactNode } from "react";
import {
  ConnectionProvider,
  WalletProvider,
} from "@solana/wallet-adapter-react";
import { WalletModalProvider } from "@solana/wallet-adapter-react-ui";
import * as web3 from "@solana/web3.js";
import * as walletAdapterWallets from "@solana/wallet-adapter-wallets";
require("@solana/wallet-adapter-react-ui/styles.css");

const WalletContextProvider: FC<{ children: ReactNode }> = ({ children }) => {
  const endpoint = web3.clusterApiUrl("devnet");
  const wallets = useMemo(() => [], []);

  return (
    <ConnectionProvider endpoint={endpoint}>
      <WalletProvider wallets={wallets}>
        <WalletModalProvider>{children}</WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
};

export default WalletContextProvider;

```

### 4. Додати багатофункціональну кнопку для гаманця

Наступним кроком налаштуємо кнопку підключення. Поточна кнопка не виконує жодних функцій, оскільки замість використання стандартної кнопки або створення власного компонента, ми будемо використовувати багатофункціональну кнопку Wallet-Adapter. Ця кнопка взаємодіє з провайдерами, які ми налаштували в `WalletContextProvider`, і дозволяє користувачам вибирати гаманець, підключатися до нього та відключатися від нього. Якщо вам потрібна додаткова функціональність, ви можете створити власний компонент для цього.

Перед тим, як додати “багатофункціональну кнопку,” нам потрібно обгорнути застосунок у `WalletContextProvider`. Зробіть це, імпортувавши його в `index.tsx` і додавши після закриваючого тега `</Head>`:

```tsx
import { NextPage } from "next";
import styles from "../styles/Home.module.css";
import WalletContextProvider from "../components/WalletContextProvider";
import { AppBar } from "../components/AppBar";
import Head from "next/head";
import { PingButton } from "../components/PingButton";

const Home: NextPage = (props) => {
  return (
    <div className={styles.App}>
      <Head>
        <title>Wallet-Adapter Example</title>
        <meta name="description" content="Wallet-Adapter Example" />
      </Head>
      <WalletContextProvider>
        <AppBar />
        <div className={styles.AppBody}>
          <PingButton />
        </div>
      </WalletContextProvider>
    </div>
  );
};

export default Home;
```

Якщо ви запустите застосунок, все має виглядати так само, оскільки поточна кнопка у верхньому правому куті все ще не виконує жодних функцій. Щоб це виправити, відкрийте `AppBar.tsx` і замініть `<button>Connect</button>` на `<WalletMultiButton/>`. Не забудьте імпортувати `WalletMultiButton` з `@solana/wallet-adapter-react-ui`.

```tsx
import { FC } from "react";
import styles from "../styles/Home.module.css";
import Image from "next/image";
import { WalletMultiButton } from "@solana/wallet-adapter-react-ui";

export const AppBar: FC = () => {
  return (
    <div className={styles.AppHeader}>
      <Image src="/solanaLogo.png" height={30} width={200} />
      <span>Wallet-Adapter Example</span>
      <WalletMultiButton />
    </div>
  );
};
```

На цьому етапі ви повинні мати можливість запустити застосунок і взаємодіяти з багатофункціональною кнопкою у верхньому правому куті екрану. Тепер вона повинна показувати текст "Select Wallet." Якщо у вас є розширення Phantom і ви увійшли в акаунт, ви повинні мати можливість підключити ваш Phantom гаманець до сайту, використовуючи цю нову кнопку.

### 5. Створити кнопку для пінгу програми

Тепер, коли наш застосунок може підключатися до гаманця Phantom, давайте зробимо так, щоб кнопка "Ping!" насправді виконувала дію.

Для початку відкрийте файл `PingButton.tsx`. Ми замінимо `console.log` всередині функції `onClick` на код, який створюватиме транзакцію та надсилатиме її на затвердження через розширення Phantom для користувача.

Спочатку нам потрібне з’єднання, публічний ключ гаманця та функція `sendTransaction` з Wallet-Adapter. Для цього нам потрібно імпортувати `useConnection` та `useWallet` з `@solana/wallet-adapter-react`. Також давайте імпортуємо `@solana/web3.js`, оскільки він знадобиться для створення нашої транзакції.

```tsx
import { useConnection, useWallet } from '@solana/wallet-adapter-react'
import * as web3 from '@solana/web3.js'
import { FC, useState } from 'react'
import styles from '../styles/PingButton.module.css'

export const PingButton: FC = () => {

  const onClick = () => {
    console.log('Ping!')
  }

  return (
    <div className={styles.buttonContainer} onClick={onClick}>
      <button className={styles.button}>Ping!</button>
    </div>
  )
}
```

Тепер використайте хук `useConnection`, щоб створити константу `connection`, і хук `useWallet`, щоб створити константи `publicKey` та `sendTransaction`.

```tsx
import { useConnection, useWallet } from "@solana/wallet-adapter-react";
import * as web3 from "@solana/web3.js";
import { FC, useState } from "react";
import styles from "../styles/PingButton.module.css";

export const PingButton: FC = () => {
  const { connection } = useConnection();
  const { publicKey, sendTransaction } = useWallet();

  const onClick = () => {
    console.log("Ping!");
  };

  return (
    <div className={styles.buttonContainer} onClick={onClick}>
      <button className={styles.button}>Ping!</button>
    </div>
  );
};
```

Після цього можна заповнити тіло `onClick`.  

Спочатку перевірте, чи існують `connection` і `publicKey` (якщо будь-який із них відсутній, це означає, що гаманець користувача ще не підключений).  

Далі створіть два екземпляри `PublicKey`: один для ідентифікатора програми `ChT1B39WKLS8qUrkLvFDXMhEJ4F1XZzwUNHUt4AU9aVa` і один для акаунта даних `Ah9K7dQ8EHaZqcAsgBW8w37yN2eAy3koFmUn4x3CJtod`.

Далі створіть об'єкт `Transaction`, а потім новий `TransactionInstruction`, що містить акаунт даних як записуваний ключ.  

Після цього додайте цю інструкцію до транзакції.  

Нарешті, викличте `sendTransaction`.

```tsx
const onClick = () => {
  if (!connection || !publicKey) {
    return;
  }

  const programId = new web3.PublicKey(PROGRAM_ID);
  const programDataAccount = new web3.PublicKey(DATA_ACCOUNT_PUBKEY);
  const transaction = new web3.Transaction();

  const instruction = new web3.TransactionInstruction({
    keys: [
      {
        pubkey: programDataAccount,
        isSigner: false,
        isWritable: true,
      },
    ],
    programId,
  });

  transaction.add(instruction);
  sendTransaction(transaction, connection).then((sig) => {
    console.log(sig);
  });
};
```

І це все! Якщо ви оновите сторінку, підключите гаманець і натиснете кнопку "Ping!", Phantom повинен відобразити спливаюче вікно для підтвердження транзакції.

### 6. Додайте фінальні штрихи  

Є багато способів покращити користувацький досвід. Наприклад, можна змінити UI так, щоб кнопка "Ping!" відображалася лише після підключення гаманця, а в іншому випадку показувалася відповідна підказка. Ви також можете додати посилання на транзакцію в Solana Explorer після її підтвердження, щоб користувач міг легко переглянути деталі. Чим більше ви експериментуватимете, тим краще освоїте цей процес, тож не бійтеся проявляти креативність!

Ви також можете завантажити [повний вихідний код цього лабораторного завдання](https://github.com/Unboxed-Software/solana-ping-frontend), щоб розглянути все в контексті.

# Виклик  

Тепер ваша черга самостійно створити застосунок. Розробіть застосунок, який дозволяє користувачеві підключити свій Phantom-гаманець і надіслати SOL на інший акаунт.

![Send SOL App](../assets/solana-send-sol-app.png)

1. Ви можете створити застосунок з нуля або [завантажити стартовий код](https://github.com/Unboxed-Software/solana-send-sol-frontend/tree/starter).  
2. Обгорніть стартовий застосунок у відповідні контекстні провайдери.  
3. У компоненті форми налаштуйте транзакцію та надішліть її в гаманець користувача для підтвердження.  
4. Проявіть креативність у покращенні користувацького досвіду. Додайте посилання для перегляду транзакції у Solana Explorer або інші цікаві функції!

Якщо ви зовсім зайшли в глухий кут, можете [переглянути код розв’язку](https://github.com/Unboxed-Software/solana-send-sol-frontend/tree/main).  

## Завершили лабораторну роботу?  

Завантажте свій код на GitHub і [поділіться своїми враженнями від цього уроку](https://form.typeform.com/to/IPH0UGz7#answers-lesson=69c5aac6-8a9f-4e23-a7f5-28ae2845dfe1)!
