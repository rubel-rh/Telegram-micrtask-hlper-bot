
import sqlite3
from datetime import datetime

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ContextTypes,
    filters
)

# =====================================
# 1. BOT SETTINGS
# =====================================

TOKEN = "9209702938:AAGGh3AEtfWhDvJxf1YGDU_SQmsmA4nDFls"
ADMIN_ID = 5881349343

DB_NAME = "microtask360.db"
DEFAULT_PRICE = 50

PAYMENT_METHODS = {
    "bkash": "bKash",
    "nagad": "Nagad",
    "rocket": "Rocket",
    "binance": "Binance"
}


# =====================================
# 2. DATABASE
# =====================================

def connect_db():
    return sqlite3.connect(DB_NAME)


def setup_database():
    con = connect_db()
    cur = con.cursor()

    cur.execute("""
        CREATE TABLE IF NOT EXISTS customers (
            user_id INTEGER PRIMARY KEY,
            name TEXT,
            username TEXT,
            joined TEXT
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            service TEXT,
            price INTEGER,
            method TEXT,
            trx_id TEXT,
            status TEXT,
            created TEXT
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS settings (
            name TEXT PRIMARY KEY,
            value TEXT
        )
    """)

    cur.execute("""
        INSERT OR IGNORE INTO settings VALUES
        ('gmail_price', '50')
    """)

    for method in PAYMENT_METHODS:
        cur.execute(
            "INSERT OR IGNORE INTO settings VALUES (?, '')",
            (f"payment_{method}",)
        )

    con.commit()
    con.close()


def get_setting(name, default=""):
    con = connect_db()
    cur = con.cursor()
    cur.execute(
        "SELECT value FROM settings WHERE name=?",
        (name,)
    )
    result = cur.fetchone()
    con.close()
    return result[0] if result else default


def set_setting(name, value):
    con = connect_db()
    cur = con.cursor()
    cur.execute("""
        INSERT OR REPLACE INTO settings (name, value)
        VALUES (?, ?)
    """, (name, str(value)))
    con.commit()
    con.close()


def get_price():
    return int(get_setting("gmail_price", str(DEFAULT_PRICE)))


def save_customer(user):
    con = connect_db()
    cur = con.cursor()

    cur.execute("""
        INSERT OR IGNORE INTO customers
        (user_id, name, username, joined)
        VALUES (?, ?, ?, ?)
    """, (
        user.id,
        user.full_name,
        user.username or "",
        datetime.now().strftime("%Y-%m-%d %H:%M")
    ))

    cur.execute("""
        UPDATE customers
        SET name=?, username=?
        WHERE user_id=?
    """, (user.full_name, user.username or "", user.id))

    con.commit()
    con.close()


# =====================================
# 3. CUSTOMER DASHBOARD
# =====================================

def customer_menu():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton(
            "📧 Gmail Services", callback_data="service"
        )],
        [InlineKeyboardButton(
            "📦 My Orders", callback_data="my_orders"
        )],
        [InlineKeyboardButton(
            "💬 Customer Support", callback_data="support"
        )]
    ])


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    save_customer(update.effective_user)

    context.user_data.clear()

    await update.message.reply_text(
        "🌟 স্বাগতম MicroTask Helper Bot-এ!\n\n"
        "আপনার ডিজিটাল সার্ভিস পেতে নিচের মেনু ব্যবহার করুন।",
        reply_markup=customer_menu()
    )


# =====================================
# 4. ADMIN DASHBOARD
# =====================================

def admin_menu():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton(
            "📧 Gmail Services", callback_data="admin_gmail"
        )],
        [InlineKeyboardButton(
            "📦 New Orders", callback_data="admin_orders"
        )],
        [InlineKeyboardButton(
            "💰 Payment Management", callback_data="admin_payments"
        )],
        [InlineKeyboardButton(
            "⚙️ Set Service Price", callback_data="admin_price"
        )],
        [InlineKeyboardButton(
            "🏦 Payment Account Setup", callback_data="admin_payment_setup"
        )],
        [InlineKeyboardButton(
            "👥 Customer Management", callback_data="admin_customers"
        )]
    ])


async def admin(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text(
            "⛔ শুধু Admin এই প্যানেল ব্যবহার করতে পারবেন।"
        )
        return

    context.user_data.clear()

    await update.message.reply_text(
        "👑 MicroTask Helper\n"
        "ADMIN CONTROL PANEL\n\n"
        "আপনার পছন্দের অপশন নির্বাচন করুন:",
        reply_markup=admin_menu()
    )


# =====================================
# 5. GMAIL SERVICE
# =====================================

async def show_service(query):
    price = get_price()

    keyboard = [
        [InlineKeyboardButton(
            "🛒 Order Now", callback_data="new_order"
        )],
        [InlineKeyboardButton(
            "🔙 Main Menu", callback_data="customer_home"
        )]
    ]

    await query.edit_message_text(
        "📧 GMAIL SERVICES\n\n"
        "নিজের Gmail অ্যাকাউন্ট তৈরি ও সেটআপে সহায়তা।\n\n"
        f"সার্ভিস মূল্য: {price} BDT\n\n"
        "আপনার Gmail অ্যাকাউন্ট ও পাসওয়ার্ডের "
        "নিয়ন্ত্রণ আপনার কাছেই থাকবে।",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )


# =====================================
# 6. CREATE ORDER
# =====================================

async def create_order(query, context):
    user = query.from_user
    price = get_price()

    save_customer(user)

    con = connect_db()
    cur = con.cursor()

    cur.execute("""
        INSERT INTO orders
        (user_id, service, price, method, trx_id, status, created)
        VALUES (?, ?, ?, '', '', 'awaiting_payment', ?)
    """, (
        user.id,
        "Gmail Setup Assistance",
        price,
        datetime.now().strftime("%Y-%m-%d %H:%M")
    ))

    order_id = cur.lastrowid

    con.commit()
    con.close()

    keyboard = [
        [InlineKeyboardButton(
            "bKash", callback_data=f"pay:bkash:{order_id}"
        )],
        [InlineKeyboardButton(
            "Nagad", callback_data=f"pay:nagad:{order_id}"
        )],
        [InlineKeyboardButton(
            "Rocket", callback_data=f"pay:rocket:{order_id}"
        )],
        [InlineKeyboardButton(
            "Binance", callback_data=f"pay:binance:{order_id}"
        )]
    ]

    await query.edit_message_text(
        f"✅ অর্ডার তৈরি হয়েছে!\n\n"
        f"Order ID: #{order_id}\n"
        f"সার্ভিস মূল্য: {price} BDT\n\n"
        "পেমেন্ট মাধ্যম নির্বাচন করুন:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

    await context.bot.send_message(
        ADMIN_ID,
        f"🔔 নতুন অর্ডার!\n\n"
        f"Order ID: #{order_id}\n"
        f"গ্রাহক: {user.full_name}\n"
        f"Customer ID: {user.id}\n"
        f"মূল্য: {price} BDT"
    )


# =====================================
# 7. PAYMENT SELECTION
# =====================================

async def choose_payment(query, context):
    _, method, order_id_text = query.data.split(":")
    order_id = int(order_id_text)

    con = connect_db()
    cur = con.cursor()

    cur.execute("""
        SELECT user_id, status, price
        FROM orders WHERE id=?
    """, (order_id,))

    order = cur.fetchone()

    if not order or order[0] != query.from_user.id:
        con.close()
        await query.edit_message_text("অর্ডার পাওয়া যায়নি।")
        return

    if order[1] != "awaiting_payment":
        con.close()
        await query.edit_message_text(
            "এই অর্ডারের পেমেন্ট আর গ্রহণ করা হচ্ছে না।"
        )
        return

    cur.execute(
        "UPDATE orders SET method=? WHERE id=?",
        (method, order_id)
    )

    con.commit()
    con.close()

    context.user_data["payment_order"] = order_id

    details = get_setting(f"payment_{method}")

    if not details:
        details = (
            "এই পেমেন্ট মাধ্যমের তথ্য এখনো Admin সেট করেননি। "
            "অনুগ্রহ করে Admin-এর সঙ্গে যোগাযোগ করুন।"
        )

    await query.edit_message_text(
        f"💳 Payment: {PAYMENT_METHODS[method]}\n\n"
        f"{details}\n\n"
        f"Order: #{order_id}\n"
        f"Amount: {order[2]} BDT\n\n"
        "পেমেন্ট করার পর Transaction ID লিখে পাঠান।\n"
        "পেমেন্ট না করে Transaction ID পাঠাবেন না।"
    )


# =====================================
# 8. ADMIN ORDER REVIEW
# =====================================

async def review_order(query, context, order_id, decision):
    con = connect_db()
    cur = con.cursor()

    cur.execute("""
        SELECT user_id, status
        FROM orders WHERE id=?
    """, (order_id,))

    order = cur.fetchone()

    if not order or order[1] != "payment_submitted":
        con.close()
        await query.answer(
            "এই অর্ডার যাচাইয়ের অপেক্ষায় নেই।",
            show_alert=True
        )
        return

    new_status = (
        "approved" if decision == "approve" else "rejected"
    )

    cur.execute(
        "UPDATE orders SET status=? WHERE id=?",
        (new_status, order_id)
    )

    con.commit()
    con.close()

    if decision == "approve":
        message = (
            f"✅ আপনার অর্ডার #{order_id} অনুমোদিত হয়েছে।\n"
            "Admin পরবর্তী সার্ভিসের ধাপ জানাবেন।"
        )
    else:
        message = (
            f"❌ আপনার অর্ডার #{order_id} বাতিল করা হয়েছে।\n"
            "প্রয়োজনে Admin-এর সঙ্গে যোগাযোগ করুন।"
        )

    try:
        await context.bot.send_message(order[0], message)
    except Exception:
        pass

    await query.edit_message_text(
        f"Order #{order_id}: {new_status.upper()}"
    )


# =====================================
# 9. BUTTON HANDLER
# =====================================

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data
    user_id = query.from_user.id

    # Protect every admin action
    admin_actions = (
    data.startswith("admin_")
    or data.startswith("approve:")
    or data.startswith("reject:")
    or data.startswith("setup:")
)

    if admin_actions and user_id != ADMIN_ID:
        await query.answer(
            "শুধু Admin ব্যবহার করতে পারবেন!",
            show_alert=True
        )
        return

    await query.answer()

    # CUSTOMER HOME
    if data == "customer_home":
        context.user_data.clear()
        await query.edit_message_text(
            "🏠 Customer Dashboard\n\nসেবা নির্বাচন করুন:",
            reply_markup=customer_menu()
        )

    # SHOW SERVICE
    elif data == "service":
        await show_service(query)

    # CREATE ORDER
    elif data == "new_order":
        await create_order(query, context)

    # SELECT PAYMENT
    elif data.startswith("pay:"):
        await choose_payment(query, context)

    # MY ORDERS
    elif data == "my_orders":
        con = connect_db()
        cur = con.cursor()

        cur.execute("""
            SELECT id, service, price, status, created
            FROM orders WHERE user_id=?
            ORDER BY id DESC LIMIT 10
        """, (user_id,))

        orders = cur.fetchall()
        con.close()

        if orders:
            message = "📦 YOUR ORDERS\n\n"

            for item in orders:
                message += (
                    f"Order: #{item[0]}\n"
                    f"Service: {item[1]}\n"
                    f"Price: {item[2]} BDT\n"
                    f"Status: {item[3]}\n"
                    f"Date: {item[4]}\n\n"
                )
        else:
            message = "এখনো কোনো অর্ডার করেননি।"

        await query.edit_message_text(
            message,
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton(
                    "🔙 Back", callback_data="customer_home"
                )]
            ])
        )

    # SUPPORT
    elif data == "support":
        context.user_data.clear()
        context.user_data["support_mode"] = True

        await query.edit_message_text(
            "💬 আপনার সমস্যা বা প্রশ্ন লিখে পাঠান।\n"
            "আপনার মেসেজ সরাসরি Admin-এর কাছে যাবে।"
        )

    # ADMIN HOME
    elif data == "admin_home":
        await query.edit_message_text(
            "👑 MicroTask Helper\nADMIN CONTROL PANEL",
            reply_markup=admin_menu()
        )

    # ADMIN GMAIL
    elif data == "admin_gmail":
        await query.edit_message_text(
            f"📧 Gmail Services\n\n"
            f"বর্তমান মূল্য: {get_price()} BDT",
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton(
                    "⚙️ Change Price", callback_data="admin_price"
                )],
                [InlineKeyboardButton(
                    "🔙 Admin Menu", callback_data="admin_home"
                )]
            ])
        )

    # ADMIN PRICE
    elif data == "admin_price":
        context.user_data.clear()
        context.user_data["change_price"] = True

        await query.edit_message_text(
            "⚙️ নতুন সার্ভিস মূল্য BDT-তে লিখুন।\n\n"
            "উদাহরণ: 100"
        )

    # PAYMENT ACCOUNT SETUP
    elif data == "admin_payment_setup":
        keyboard = [
            [InlineKeyboardButton(
                "bKash", callback_data="setup:bkash"
            )],
            [InlineKeyboardButton(
                "Nagad", callback_data="setup:nagad"
            )],
            [InlineKeyboardButton(
                "Rocket", callback_data="setup:rocket"
            )],
            [InlineKeyboardButton(
                "Binance", callback_data="setup:binance"
            )],
            [InlineKeyboardButton(
                "🔙 Admin Menu", callback_data="admin_home"
            )]
        ]

        await query.edit_message_text(
            "🏦 Payment Account Setup\n\n"
            "যে মাধ্যমের পেমেন্ট তথ্য পরিবর্তন করতে চান সেটি নির্বাচন করুন:",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

    elif data.startswith("setup:"):
        method = data.split(":")[1]

        context.user_data.clear()
        context.user_data["setup_payment"] = method

        await query.edit_message_text(
            f"🏦 {PAYMENT_METHODS[method]} Payment Setup\n\n"
            "আপনার পেমেন্ট নম্বর বা নির্দেশনা লিখে পাঠান।\n"
            "উদাহরণ: 01XXXXXXXXX (Personal)"
        )

    # ADMIN ORDERS
    elif data == "admin_orders":
        con = connect_db()
        cur = con.cursor()

        cur.execute("""
            SELECT id, user_id, price, status
            FROM orders ORDER BY id DESC LIMIT 15
        """)

        orders = cur.fetchall()
        con.close()

        if not orders:
            message = "এখনো কোনো অর্ডার নেই।"
        else:
            message = "📦 RECENT ORDERS\n\n"

            for item in orders:
                message += (
                    f"Order: #{item[0]}\n"
                    f"Customer ID: {item[1]}\n"
                    f"Price: {item[2]} BDT\n"
                    f"Status: {item[3]}\n\n"
                )

        await query.edit_message_text(
            message,
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton(
                    "🔙 Admin Menu", callback_data="admin_home"
                )]
            ])
        )

    # PAYMENT MANAGEMENT
    elif data == "admin_payments":
        con = connect_db()
        cur = con.cursor()

        cur.execute("""
            SELECT id, user_id, method, trx_id, price
            FROM orders
            WHERE status='payment_submitted'
            ORDER BY id DESC LIMIT 10
        """)

        payments = cur.fetchall()
        con.close()

        if not payments:
            await query.edit_message_text(
                "💰 যাচাইয়ের অপেক্ষায় কোনো পেমেন্ট নেই।",
                reply_markup=InlineKeyboardMarkup([
                    [InlineKeyboardButton(
                        "🔙 Admin Menu", callback_data="admin_home"
                    )]
                ])
            )
            return

        for item in payments:
            keyboard = [[
                InlineKeyboardButton(
                    "✅ Approve",
                    callback_data=f"approve:{item[0]}"
                ),
                InlineKeyboardButton(
                    "❌ Reject",
                    callback_data=f"reject:{item[0]}"
                )
            ]]

            await context.bot.send_message(
                ADMIN_ID,
                f"💰 PAYMENT REVIEW\n\n"
                f"Order: #{item[0]}\n"
                f"Customer ID: {item[1]}\n"
                f"Method: {PAYMENT_METHODS.get(item[2], item[2])}\n"
                f"Transaction ID: {item[3]}\n"
                f"Amount: {item[4]} BDT",
                reply_markup=InlineKeyboardMarkup(keyboard)
            )

        await query.edit_message_text(
            "পেমেন্ট যাচাইয়ের তালিকা Admin-এ পাঠানো হয়েছে।",
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton(
                    "🔙 Admin Menu", callback_data="admin_home"
                )]
            ])
        )

    # APPROVE / REJECT
    elif data.startswith("approve:"):
        await review_order(
            query, context, int(data.split(":")[1]), "approve"
        )

    elif data.startswith("reject:"):
        await review_order(
            query, context, int(data.split(":")[1]), "reject"
        )

    # CUSTOMERS
    elif data == "admin_customers":
        con = connect_db()
        cur = con.cursor()

        cur.execute("SELECT COUNT(*) FROM customers")
        total = cur.fetchone()[0]

        cur.execute("""
            SELECT name, username, user_id
            FROM customers ORDER BY joined DESC LIMIT 15
        """)

        customers = cur.fetchall()
        con.close()

        message = f"👥 Total Customers: {total}\n\n"

        for item in customers:
            message += (
                f"Name: {item[0]}\n"
                f"Username: @{item[1] if item[1] else 'None'}\n"
                f"ID: {item[2]}\n\n"
            )

        await query.edit_message_text(
            message,
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton(
                    "🔙 Admin Menu", callback_data="admin_home"
                )]
            ])
        )


# =====================================
# 10. TEXT MESSAGE HANDLER
# =====================================

async def text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    message = update.message.text.strip()

    # CHANGE SERVICE PRICE
    if user.id == ADMIN_ID and context.user_data.get("change_price"):

        if not message.isdigit() or int(message) < 1:
            await update.message.reply_text(
                "সঠিক মূল্য লিখুন। উদাহরণ: 100"
            )
            return

        set_setting("gmail_price", message)
        context.user_data.pop("change_price", None)

        await update.message.reply_text(
            f"✅ নতুন মূল্য সেট হয়েছে: {message} BDT",
            reply_markup=admin_menu()
        )
        return

    # SET PAYMENT DETAILS
    if user.id == ADMIN_ID and context.user_data.get("setup_payment"):

        method = context.user_data.pop("setup_payment")

        set_setting(f"payment_{method}", message)

        await update.message.reply_text(
            f"✅ {PAYMENT_METHODS[method]} পেমেন্ট তথ্য সংরক্ষণ হয়েছে।",
            reply_markup=admin_menu()
        )
        return

    # CUSTOMER TRANSACTION ID
    order_id = context.user_data.get("payment_order")

    if order_id:
        con = connect_db()
        cur = con.cursor()

        cur.execute("""
            SELECT user_id, status
            FROM orders WHERE id=?
        """, (order_id,))

        order = cur.fetchone()

        if (
            not order
            or order[0] != user.id
            or order[1] != "awaiting_payment"
        ):
            con.close()
            context.user_data.pop("payment_order", None)

            await update.message.reply_text(
                "এই অর্ডারটি আর পেমেন্ট গ্রহণ করছে না।"
            )
            return

        cur.execute("""
            UPDATE orders
            SET trx_id=?, status='payment_submitted'
            WHERE id=?
        """, (message, order_id))

        con.commit()
        con.close()

        context.user_data.pop("payment_order", None)

        await update.message.reply_text(
            "✅ Transaction ID জমা হয়েছে।\n"
            "Admin পেমেন্ট যাচাই করবেন।"
        )

        await context.bot.send_message(
            ADMIN_ID,
            f"🔔 নতুন Payment Submission!\n\n"
            f"Order: #{order_id}\n"
            f"Customer ID: {user.id}\n"
            f"Transaction ID: {message}\n\n"
            "Admin Panel থেকে Payment Management খুলে যাচাই করুন।"
        )
        return

    # CUSTOMER SUPPORT MESSAGE
    if context.user_data.get("support_mode"):

        context.user_data.pop("support_mode", None)

        await context.bot.send_message(
            ADMIN_ID,
            f"💬 CUSTOMER SUPPORT\n\n"
            f"Name: {user.full_name}\n"
            f"User ID: {user.id}\n"
            f"Message: {message}"
        )

        await update.message.reply_text(
            "✅ আপনার মেসেজ Admin-এর কাছে পাঠানো হয়েছে।"
        )
        return

    await update.message.reply_text(
        "মেনু ব্যবহার করতে /start লিখুন।"
    )


# =====================================
# 11. START APPLICATION
# =====================================

def main():

    setup_database()

    application = (
        Application.builder()
        .token(TOKEN)
        .build()
    )

    application.add_handler(
        CommandHandler("start", start)
    )

    application.add_handler(
        CommandHandler("admin", admin)
    )

    application.add_handler(
        CallbackQueryHandler(button_handler)
    )

    application.add_handler(
        MessageHandler(
            filters.TEXT & ~filters.COMMAND,
            text_handler
        )
    )

    print("MicroTask Helper Bot চালু হয়েছে!")

    application.run_polling()


if __name__ == "__main__":
    main()
