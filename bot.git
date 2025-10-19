import logging
import random
import string
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, filters, ContextTypes
import sqlite3

# Настройки бота
BOT_TOKEN = "7567556638:AAGG2kmz0AGKVSorsVETUME5USI1Q8J6SEE"
ADMIN_IDS = [6754891703, 1684583517]
GROUP_ID = -1003004800348
ORDER_STATUSES = ["Выкуплен", "Прибыл на склад в Гуанчжоу", "Отправлен", "Прибыл в Москву", "Завершён"]
ORDER_STATUS_MAPPING = {
    "Выкуплен": "bought",
    "Прибыл на склад в Гуанчжоу": "warehouse_guangzhou", 
    "Отправлен": "shipped",
    "Прибыл в Москву": "moscow",
    "Завершён": "completed"
}

# Обратный маппинг для получения статуса по callback_data
CALLBACK_TO_STATUS = {v: k for k, v in ORDER_STATUS_MAPPING.items()}
TRACK_NUMBER_LENGTH = 10
TRACK_NUMBER_CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
DATABASE_NAME = "orders.db"

# Настройки меню
BOT_INFO_LINK = "https://t.me/resalevlg"    # Замените на ссылку с информацией о боте

# Настройка логирования
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

class OrderTrackerBot:
    def __init__(self, token: str):
        self.token = token
        self.init_database()
    
    def init_database(self):
        """Инициализация базы данных"""
        conn = None
        try:
            conn = sqlite3.connect(DATABASE_NAME, timeout=10.0)
            cursor = conn.cursor()
            
            # Создание таблицы заказов
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS orders (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    track_number TEXT UNIQUE NOT NULL,
                    status TEXT NOT NULL DEFAULT 'В обработке',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            ''')
            
            # Создание таблицы пользователей
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER UNIQUE NOT NULL,
                    first_name TEXT,
                    last_name TEXT,
                    username TEXT,
                    first_seen TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    last_seen TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            ''')
            
            conn.commit()
        except Exception as e:
            logger.error(f"Database initialization error: {e}")
        finally:
            if conn:
                conn.close()
    
    def generate_track_number(self) -> str:
        """Генерация случайного трек номера длиной 10 символов"""
        return ''.join(random.choice(TRACK_NUMBER_CHARS) for _ in range(TRACK_NUMBER_LENGTH))
    
    def is_admin(self, user_id: int) -> bool:
        """Проверка, является ли пользователь админом"""
        return user_id in ADMIN_IDS
    
    def get_order_status(self, track_number: str) -> str:
        """Получение статуса заказа по трек номеру"""
        conn = None
        try:
            conn = sqlite3.connect(DATABASE_NAME, timeout=10.0)
            cursor = conn.cursor()
            
            cursor.execute('SELECT status FROM orders WHERE track_number = ?', (track_number,))
            result = cursor.fetchone()
            
            if result:
                return result[0]
            return None
        except Exception as e:
            logger.error(f"Database error in get_order_status: {e}")
            return None
        finally:
            if conn:
                conn.close()
    
    def create_order(self, track_number: str, status: str = 'Выкуплен') -> bool:
        """Создание нового заказа"""
        conn = None
        try:
            conn = sqlite3.connect(DATABASE_NAME, timeout=10.0)
            cursor = conn.cursor()
            
            cursor.execute('''
                INSERT INTO orders (track_number, status) 
                VALUES (?, ?)
            ''', (track_number, status))
            
            conn.commit()
            return True
        except sqlite3.IntegrityError:
            return False  # Трек номер уже существует
        except Exception as e:
            logger.error(f"Database error in create_order: {e}")
            return False
        finally:
            if conn:
                conn.close()
    
    def update_order_status(self, track_number: str, new_status: str) -> bool:
        """Обновление статуса заказа"""
        conn = None
        try:
            conn = sqlite3.connect(DATABASE_NAME, timeout=10.0)
            cursor = conn.cursor()
            
            cursor.execute('''
                UPDATE orders 
                SET status = ?, updated_at = CURRENT_TIMESTAMP 
                WHERE track_number = ?
            ''', (new_status, track_number))
            
            affected_rows = cursor.rowcount
            conn.commit()
            
            return affected_rows > 0
        except Exception as e:
            logger.error(f"Database error in update_order_status: {e}")
            return False
        finally:
            if conn:
                conn.close()
    
    def get_all_orders(self) -> list:
        """Получение всех заказов"""
        conn = None
        try:
            conn = sqlite3.connect(DATABASE_NAME, timeout=10.0)
            cursor = conn.cursor()
            
            cursor.execute('SELECT track_number, status FROM orders ORDER BY created_at DESC')
            orders = cursor.fetchall()
            
            return orders
        except Exception as e:
            logger.error(f"Database error in get_all_orders: {e}")
            return []
        finally:
            if conn:
                conn.close()
    
    def is_user_registered(self, user_id: int) -> bool:
        """Проверка, зарегистрирован ли пользователь"""
        conn = None
        try:
            conn = sqlite3.connect(DATABASE_NAME, timeout=10.0)
            cursor = conn.cursor()
            
            cursor.execute('SELECT user_id FROM users WHERE user_id = ?', (user_id,))
            result = cursor.fetchone()
            
            return result is not None
        except Exception as e:
            logger.error(f"Database error in is_user_registered: {e}")
            return False
        finally:
            if conn:
                conn.close()
    
    def register_user(self, user_id: int, first_name: str = None, last_name: str = None, username: str = None) -> bool:
        """Регистрация нового пользователя"""
        conn = None
        try:
            conn = sqlite3.connect(DATABASE_NAME, timeout=10.0)
            cursor = conn.cursor()
            
            # Сначала проверяем, существует ли пользователь
            cursor.execute('SELECT user_id FROM users WHERE user_id = ?', (user_id,))
            existing_user = cursor.fetchone()
            
            if existing_user:
                # Пользователь уже существует, обновляем last_seen
                cursor.execute('''
                    UPDATE users 
                    SET last_seen = CURRENT_TIMESTAMP, 
                        first_name = COALESCE(?, first_name),
                        last_name = COALESCE(?, last_name),
                        username = COALESCE(?, username)
                    WHERE user_id = ?
                ''', (first_name, last_name, username, user_id))
                conn.commit()
                return False
            else:
                # Новый пользователь
                cursor.execute('''
                    INSERT INTO users (user_id, first_name, last_name, username) 
                    VALUES (?, ?, ?, ?)
                ''', (user_id, first_name, last_name, username))
                conn.commit()
                return True
                
        except sqlite3.OperationalError as e:
            logger.error(f"Database error in register_user: {e}")
            return False
        except Exception as e:
            logger.error(f"Unexpected error in register_user: {e}")
            return False
        finally:
            if conn:
                conn.close()

# Создание экземпляра бота
bot_instance = None

# Словарь для хранения состояний пользователей
user_states = {}

async def notify_group(message: str, context: ContextTypes.DEFAULT_TYPE):
    """Отправка уведомления в группу"""
    try:
        await context.bot.send_message(
            chat_id=GROUP_ID,
            text=message,
            parse_mode='HTML'
        )
    except Exception as e:
        logger.error(f"Ошибка отправки уведомления в группу: {e}")

async def notify_new_user(user_info: str, context: ContextTypes.DEFAULT_TYPE):
    """Уведомление о новом пользователе"""
    message = f"🆕 <b>Новый пользователь!</b>\n\n{user_info}"
    await notify_group(message, context)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обработчик команды /start"""
    user_id = update.effective_user.id
    user = update.effective_user
    
    # Регистрация пользователя и уведомление (только если это не админ)
    if not bot_instance.is_admin(user_id):
        # Регистрируем пользователя
        is_new_user = bot_instance.register_user(
            user_id=user_id,
            first_name=user.first_name,
            last_name=user.last_name,
            username=user.username
        )
        
        # Уведомляем только о новых пользователях
        if is_new_user:
            user_info = f"👤 <b>Имя:</b> {user.first_name or 'Не указано'}"
            if user.last_name:
                user_info += f" {user.last_name}"
            user_info += f"\n🆔 <b>ID:</b> <code>{user_id}</code>"
            if user.username:
                user_info += f"\n📱 <b>Username:</b> @{user.username}"
            user_info += f"\n📅 <b>Время:</b> {update.message.date.strftime('%d.%m.%Y %H:%M')}"
            user_info += f"\n🔗 <b>Ссылка:</b> <a href='tg://user?id={user_id}'>Написать пользователю</a>"
            
            await notify_new_user(user_info, context)
    
    # Единое приветственное сообщение для всех пользователей
    keyboard = [
        [InlineKeyboardButton("📦 Отследить заказ", callback_data="track_order")],
        [InlineKeyboardButton("🏠 Главное меню", callback_data="main_menu")],
        [InlineKeyboardButton("❓ Частые вопросы", callback_data="faq")]
    ]
    
    # Добавляем кнопку "Админ" только для админов
    if bot_instance.is_admin(user_id):
        keyboard.append([InlineKeyboardButton("🔧 Админ", callback_data="admin_panel")])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await update.message.reply_text(
        "👋 Добро пожаловать в бот отслеживания заказов!\n\n"
        "Выберите нужное действие:",
        reply_markup=reply_markup
    )

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обработчик текстовых сообщений"""
    # Проверяем, что сообщение существует и содержит текст
    if not update.message or not update.message.text:
        return
        
    user_id = update.effective_user.id
    text = update.message.text.strip()
    
    # Проверяем, находится ли пользователь в режиме отслеживания
    if user_states.get(user_id) != 'tracking':
        # Если пользователь не в режиме отслеживания, НЕ РЕАГИРУЕМ на сообщения
        # Только команды /start и /menu обрабатываются
        return
    
    # Проверка трек номера (только если пользователь в режиме отслеживания)
    if len(text) == 10 and text.isalnum():
        status = bot_instance.get_order_status(text.upper())
        if status:
            # Определяем правильную кнопку "Назад" в зависимости от роли пользователя
            if bot_instance.is_admin(user_id):
                back_callback = "back_from_admin"
            else:
                back_callback = "main_menu"
                
            keyboard = [
                [InlineKeyboardButton("🔙 Назад", callback_data=back_callback)],
                [InlineKeyboardButton("❓ Частые вопросы", callback_data="faq")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await update.message.reply_text(
                f"📦 Трек номер: <code>{text.upper()}</code>\n"
                f"📊 Статус: {status}",
                reply_markup=reply_markup,
                parse_mode='HTML'
            )
        else:
            # Определяем правильную кнопку "Назад" в зависимости от роли пользователя
            if bot_instance.is_admin(user_id):
                back_callback = "back_from_admin"
            else:
                back_callback = "main_menu"
                
            keyboard = [
                [InlineKeyboardButton("🔙 Назад", callback_data=back_callback)],
                [InlineKeyboardButton("❓ Частые вопросы", callback_data="faq")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await update.message.reply_text(
                "❌ Заказ с таким трек номером не найден.\n"
                "Проверьте правильность введенного номера.",
                reply_markup=reply_markup
            )
        
        # Сбрасываем состояние после обработки трек номера
        user_states[user_id] = None
    else:
        keyboard = [
            [InlineKeyboardButton("🏠 Главное меню", callback_data="main_menu")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await update.message.reply_text(
            "❌ Неверный формат трек номера.\n"
            "Трек номер должен содержать 10 символов (буквы и цифры).\n\n"
            "Попробуйте еще раз:",
            reply_markup=reply_markup
        )

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обработчик нажатий на кнопки"""
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    
    if query.data == "all_orders":
        if not bot_instance.is_admin(user_id):
            await query.edit_message_text("❌ У вас нет прав доступа к админской панели.")
            return
            
        orders = bot_instance.get_all_orders()
        if orders:
            message = "📦 Все заказы:\n\n"
            for track_number, status in orders:
                message += f"🔸 <code>{track_number}</code>: {status}\n"
        else:
            message = "📦 Заказы не найдены."
        
        keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data="back_to_admin")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(message, reply_markup=reply_markup, parse_mode='HTML')
    
    elif query.data == "create_order":
        if not bot_instance.is_admin(user_id):
            await query.edit_message_text("❌ У вас нет прав доступа к админской панели.")
            return
            
        track_number = bot_instance.generate_track_number()
        success = bot_instance.create_order(track_number)
        
        if success:
            message = f"✅ Новый заказ создан!\n\n📦 Трек номер: <code>{track_number}</code>\n📊 Статус: Выкуплен"
            
            # Уведомление в группу о создании заказа
            admin_notification = f"📦 <b>Новый заказ создан!</b>\n\n📦 <b>Трек номер:</b> <code>{track_number}</code>\n📊 <b>Статус:</b> Выкуплен\n👤 <b>Создал:</b> {query.from_user.first_name or 'Админ'}"
            await notify_group(admin_notification, context)
        else:
            message = "❌ Ошибка при создании заказа."
        
        keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data="back_to_admin")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(message, reply_markup=reply_markup, parse_mode='HTML')
    
    elif query.data == "change_status":
        if not bot_instance.is_admin(user_id):
            await query.edit_message_text("❌ У вас нет прав доступа к админской панели.")
            return
            
        orders = bot_instance.get_all_orders()
        if orders:
            keyboard = []
            for track_number, status in orders[:10]:  # Показываем только первые 10
                keyboard.append([InlineKeyboardButton(
                    f"{track_number} - {status}", 
                    callback_data=f"edit_{track_number}"
                )])
            
            keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="back_to_admin")])
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await query.edit_message_text(
                "✏️ Выберите заказ для изменения статуса:",
                reply_markup=reply_markup
            )
        else:
            keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data="back_to_admin")]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text("📦 Заказы не найдены.", reply_markup=reply_markup)
    
    elif query.data == "back_to_admin":
        keyboard = [
            [InlineKeyboardButton("📦 Все заказы", callback_data="all_orders")],
            [InlineKeyboardButton("➕ Создать заказ", callback_data="create_order")],
            [InlineKeyboardButton("✏️ Изменить статус", callback_data="change_status")],
            [InlineKeyboardButton("🔙 Назад", callback_data="back_from_admin")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        # Проверяем, нужно ли менять сообщение
        current_text = query.message.text
        new_text = "🔧 Админская панель\n\nВыберите действие:"
        
        if current_text != new_text:
            await query.edit_message_text(new_text, reply_markup=reply_markup)
        else:
            await query.answer("Вы уже в админской панели")
    
    elif query.data.startswith("edit_"):
        if not bot_instance.is_admin(user_id):
            await query.edit_message_text("❌ У вас нет прав доступа к админской панели.")
            return
            
        track_number = query.data[5:]  # Убираем префикс "edit_"
        
        keyboard = []
        for status in ORDER_STATUSES:
            callback_data = ORDER_STATUS_MAPPING[status]
            keyboard.append([InlineKeyboardButton(
                f"📦 {status}", 
                callback_data=f"status_{track_number}_{callback_data}"
            )])
        
        keyboard.append([InlineKeyboardButton("🔙 Назад", callback_data="change_status")])
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            f"✏️ Изменение статуса для заказа <code>{track_number}</code>:\n\nВыберите новый статус:",
            reply_markup=reply_markup,
            parse_mode='HTML'
        )
    
    elif query.data.startswith("status_"):
        if not bot_instance.is_admin(user_id):
            await query.edit_message_text("❌ У вас нет прав доступа к админской панели.")
            return
            
        parts = query.data.split("_")
        track_number = parts[1]
        callback_status = parts[2]
        new_status = CALLBACK_TO_STATUS[callback_status]
        
        success = bot_instance.update_order_status(track_number, new_status)
        
        if success:
            message = f"✅ Статус заказа <code>{track_number}</code> изменен на: <code>{new_status}</code>"
            
            # Уведомление в группу об изменении статуса
            admin_notification = f"✏️ <b>Статус заказа изменен!</b>\n\n📦 <b>Трек номер:</b> <code>{track_number}</code>\n📊 <b>Новый статус:</b> {new_status}\n👤 <b>Изменил:</b> {query.from_user.first_name or 'Админ'}"
            await notify_group(admin_notification, context)
        else:
            message = f"❌ Ошибка при изменении статуса заказа <code>{track_number}</code>"
        
        keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data="back_to_admin")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(message, reply_markup=reply_markup, parse_mode='HTML')
    
    elif query.data == "main_menu":
        # Сбрасываем состояние пользователя при возврате в главное меню
        user_states[user_id] = None
        
        if bot_instance.is_admin(user_id):
            keyboard = [
                [InlineKeyboardButton("📦 Все заказы", callback_data="all_orders")],
                [InlineKeyboardButton("➕ Создать заказ", callback_data="create_order")],
                [InlineKeyboardButton("✏️ Изменить статус", callback_data="change_status")],
                [InlineKeyboardButton("🔙 Назад", callback_data="back_from_admin")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            # Проверяем, нужно ли менять сообщение
            current_text = query.message.text
            new_text = "🔧 Админская панель\n\nВыберите действие:"
            
            if current_text != new_text:
                await query.edit_message_text(new_text, reply_markup=reply_markup)
            else:
                await query.answer("Вы уже в главном меню админа")
        else:
            keyboard = [
                [InlineKeyboardButton("📦 Отследить заказ", callback_data="track_order")],
                [InlineKeyboardButton("🏠 Главное меню", callback_data="main_menu")],
                [InlineKeyboardButton("❓ Частые вопросы", callback_data="faq")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            # Проверяем, нужно ли менять сообщение
            current_text = query.message.text
            new_text = "👋 Добро пожаловать в бот отслеживания заказов!\n\nВыберите нужное действие:"
            
            if current_text != new_text:
                await query.edit_message_text(new_text, reply_markup=reply_markup)
            else:
                await query.answer("Вы уже в главном меню")
    
    elif query.data == "track_order":
        user_id = query.from_user.id
        # Устанавливаем режим отслеживания для пользователя
        user_states[user_id] = 'tracking'
        
        # Определяем правильную кнопку "Назад" в зависимости от роли пользователя
        if bot_instance.is_admin(user_id):
            back_callback = "back_from_admin"
        else:
            back_callback = "main_menu"
        
        keyboard = [
            [InlineKeyboardButton("🔙 Назад", callback_data=back_callback)]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "📦 Отслеживание заказа\n\n"
            "Введите свой трек номер (10 символов, буквы и цифры):",
            reply_markup=reply_markup
        )
    
    elif query.data == "admin_panel":
        if not bot_instance.is_admin(user_id):
            await query.edit_message_text("❌ У вас нет прав доступа к админской панели.")
            return
            
        keyboard = [
            [InlineKeyboardButton("📦 Все заказы", callback_data="all_orders")],
            [InlineKeyboardButton("➕ Создать заказ", callback_data="create_order")],
            [InlineKeyboardButton("✏️ Изменить статус", callback_data="change_status")],
            [InlineKeyboardButton("🔙 Назад", callback_data="back_from_admin")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "🔧 Админская панель\n\n"
            "Выберите действие:",
            reply_markup=reply_markup
        )
    
    elif query.data == "back_from_admin":
        # Возврат из админской панели в главное меню
        keyboard = [
            [InlineKeyboardButton("📦 Отследить заказ", callback_data="track_order")],
            [InlineKeyboardButton("🏠 Главное меню", callback_data="main_menu")],
            [InlineKeyboardButton("❓ Частые вопросы", callback_data="faq")]
        ]
        
        # Добавляем кнопку "Админ" только для админов
        if bot_instance.is_admin(user_id):
            keyboard.append([InlineKeyboardButton("🔧 Админ", callback_data="admin_panel")])
        
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "👋 Добро пожаловать в бот отслеживания заказов!\n\n"
            "Выберите нужное действие:",
            reply_markup=reply_markup
        )
    
    elif query.data == "faq":
        # Сбрасываем состояние пользователя при переходе в FAQ
        user_states[user_id] = None
        
        faq_text = """
❓ Часто задаваемые вопросы:

🔸 <b>Как отследить заказ?</b>
Нажмите кнопку "📦 Отследить заказ", затем введите трек номер в формате: ABC123DEF4 (10 символов)

🔸 <b>Что означают статусы?</b>
• Выкуплен - ожидает прибытия на склад в Китае
• Прибыл на склад в Гуанчжоу - заказ скоро будет готов к отправке
• Отправлен - едет в Китае, будет на границе РФ через 7 дней
• Прибыл в Москву - скоро отправится в Волгоград
• Завершён

🔸 <b>Как долго обрабатывается заказ?</b>
Обычно 1-3 рабочих дня
        """
        
        # Определяем правильную кнопку "Назад" в зависимости от роли пользователя
        if bot_instance.is_admin(user_id):
            back_callback = "back_from_admin"
        else:
            back_callback = "main_menu"
            
        keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data=back_callback)]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(faq_text, reply_markup=reply_markup, parse_mode='HTML')


async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик ошибок"""
    error_message = str(context.error)
    
    # Игнорируем ошибки, которые не критичны
    if any(phrase in error_message for phrase in [
        "Message is not modified",
        "terminated by other getUpdates request",
        "Conflict:",
        "BadRequest: Message is not modified",
        "400 Bad Request",
        "'NoneType' object has no attribute 'text'",
        "database is locked",
        "UNIQUE constraint failed",
        "Button_data_invalid"
    ]):
        return
    
    logger.error("Exception while handling an update:", exc_info=context.error)

def main():
    """Основная функция запуска бота"""
    global bot_instance
    bot_instance = OrderTrackerBot(BOT_TOKEN)
    
    # Создание приложения
    application = Application.builder().token(BOT_TOKEN).build()
    
    # Добавление обработчика ошибок
    application.add_error_handler(error_handler)
    
    # Добавление обработчиков
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("menu", start))  # Команда /menu ведет к главному меню
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))  # Все текстовые сообщения
    application.add_handler(CallbackQueryHandler(button_callback))
    
    # Запуск бота
    print("🤖 Бот запущен...")
    print("🛑 Для остановки нажмите Ctrl+C")
    print("⚠️  Если возникают конфликты, используйте файл stop_and_start.bat")
    
    try:
        application.run_polling()
    except KeyboardInterrupt:
        print("\n🛑 Бот остановлен пользователем")
    except Exception as e:
        error_msg = str(e)
        if "Conflict" in error_msg or "getUpdates" in error_msg:
            print(f"\n⚠️  Обнаружен конфликт: {error_msg}")
            print("🔧 Решение:")
            print("1. Запустите файл stop_and_start.bat")
            print("2. Или выполните: taskkill /f /im python.exe")
            print("3. Затем запустите бота заново")
        else:
            print(f"\n❌ Ошибка при запуске бота: {e}")

if __name__ == '__main__':
    main()
