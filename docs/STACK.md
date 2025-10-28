1. Frontend (Клиентская Часть) — Flutter

Структура Проекта:

lib/main.dart: Точка входа, инициализация Firebase, роутинг (GoRouter или Navigator).
lib/screens/: Экраны (LoginScreen, MainMenuScreen, ChatScreen, ProfileScreen, RulesScreen).
lib/widgets/: Переиспользуемые компоненты (ChatBubble, MediaPreview с маской, RatingDialog).
lib/providers/: State management (AuthProvider, ChatProvider, WalletProvider).
lib/services/: Сервисы для интеграций (AuthService, ChatService, PaymentService, MediaEncryptionService).
lib/models/: Data models (User, Order, Message, Media).


Ключевые Функции:

Авторизация: Phone auth via Firebase Auth. Экран ввода номера, SMS-код, переход в главное меню.
Главное Меню: Два кнопки (Создать/Исполнить). Для создания: ввод суммы, оплата (интеграция Stripe), генерация уникальной ссылки (Firestore генерирует ID чата). Для исполнения: ввод ссылки, валидация и переход в чат.
Профиль: Fetch данных из Firestore (рейтинг как средняя оценка, уровень на основе completed deals, кошелек — баланс в Stripe/Firestore, история — список заказов, правила — статичный текст или Markdown, жалобы — форма отправки в Firestore, выход — signOut).
Чат: Realtime чат на Firestore (stream сообщений). Отправка текста/медиа: шифрование на клиенте (используйте пакет pointycastle или encrypt), upload в Storage, preview с маской (blur/watermark via image пакет). Скачивание/полный доступ только после mutual confirmation (update флага в БД).
Завершение Сделки: Кнопки подтверждения от обеих сторон. При success: перевод денег (Cloud Function вызывает Stripe), разблокировка медиа, модал для оценки (update рейтинга).
Медиа Обработка: На клиенте — шифрование/маска перед upload. Preview: отображать blurred версию до завершения.
Оценка и Блокировки: После сделки — обязательная оценка. Track refused orders в профиле; если >3 подряд — блокировка (update user status в БД).


Кроссплатформенные Адаптации:

Android: Полный native опыт, push-уведомления via Firebase Messaging.
Браузер: Web-версия с responsive UI (MediaQuery для адаптации). Ограничения: SMS может требовать fallback (email), медиа upload via FilePicker.
Build: flutter build apk для Android, flutter build web для браузера (хостинг на Firebase Hosting).



2. Backend (Серверная Часть) — Firebase

Компоненты:

Firebase Auth: Phone verification. Правила безопасности: только авторизованные пользователи.
Firestore (База Данных): NoSQL для realtime данных.

Collections:

users: {id, phone, rating (float), level (int, based on deals), wallet_balance (via Stripe link), history (array of order_ids), blocked (bool)}.
orders: {id, creator_id, executor_id (null initially), amount, status (pending/active/completed/disputed/refused), link (unique URL), completion_flags (creator: bool, executor: bool)}.
chats: Subcollection под orders: {messages: [{sender_id, text/media_url, timestamp, encrypted: bool}]}.
disputes: Для арбитража {order_id, proofs (array media_urls), status}.
ratings: {order_id, creator_rating, executor_rating}.


Security Rules: Читать/писать только для участников чата. Медиа URLs — private до completion.


Cloud Storage: Хранение медиа. Rules: Доступ только после completion (signed URLs via Functions).
Cloud Functions: Серверная логика (Node.js или Python).

Генерация ссылок: При создании order — create unique ID, return URL like app://chat/{order_id}.
Escrow: Интеграция Stripe (hold payment on create, release on complete). Function: onOrderComplete — transfer funds, update wallet.
Арбитраж: Manual (via admin console) или auto (проверка proofs).
Уровни: Trigger on deal complete — increment level if criteria met (e.g., >5 deals → level 2).
Блокировки: Function checks refused count, sets blocked.
Шифрование: Клиент шифрует, сервер хранит keys в Firestore (private to users).


Firebase Messaging: Push-уведомления для новых сообщений/завершений.


Платежи (Escrow): Интегрируйте Stripe via Firebase Extensions. Деньги блокируются на Stripe Connect аккаунте, переводятся исполнителю минус комиссия. Для соло — используйте test mode сначала.
Арбитраж и Модерация: Admin SDK для ручного разбора. Авто-детект: ML для NSFW (via Cloud Vision, но опционально).

3. Интеграции и Безопасность

Шифрование Медиа: На клиенте — AES encryption (пакет encrypt). Key генерируется per chat, делится только после completion.
Маска на Изображения: Используйте flutter_blurhash или canvas для blur/watermark на preview.
Анонимность и Уровни: Уровень в user doc, unlocks higher amounts (e.g., level 1: max $100). Анонимность: Имена hidden до level up.
Правила: Статичный экран с Markdown текстом (из вашего описания). Динамично — fetch из Firestore для обновлений.
Жалобы: Форма отправляет в Firestore collection reports, trigger function for review.
Ограничения Новичков: В rules — check user.level before creating high-amount orders.

4. Разработка и Деплой

Соло-Workflow: Начните с MVP (auth + menu + basic chat). Используйте Firebase Emulator локально. Тестируйте на Android emulator и Chrome.
Инструменты: VS Code + Flutter SDK. Git для версионного контроля.
Деплой:

Android: Google Play Console.
Web: Firebase Hosting (flutter build web → firebase deploy).


Стоимость: Firebase free tier хватит для старта (pay-as-you-go после). Stripe — 2-3% комиссия.
Потенциальные Риски: Обработка споров (добавьте admin panel на Flutter web). Compliance: Убедитесь в легальности escrow в вашей юрисдикции.