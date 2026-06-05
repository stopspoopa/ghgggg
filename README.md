

import sys
import os
import shutil
import pymysql
from PyQt6.QtWidgets import *
from PyQt6.QtCore import *
from PyQt6.QtGui import *


# --- Оформление интерфейса ---------------------------------------------------

def apply_light_theme(app):
    """
    Применяет светлую цветовую схему ко всему приложению.
    Используется стиль Fusion и пользовательская палитра Qt,
    чтобы интерфейс выглядел единообразно на разных системах.
    """
    app.setStyle("Fusion")
    p = QPalette()
    # Основные цвета фона и текста
    p.setColor(QPalette.ColorRole.Window, QColor(245, 245, 245))
    p.setColor(QPalette.ColorRole.WindowText, QColor(33, 33, 33))
    p.setColor(QPalette.ColorRole.Base, QColor(255, 255, 255))
    p.setColor(QPalette.ColorRole.AlternateBase, QColor(240, 240, 240))
    p.setColor(QPalette.ColorRole.ToolTipBase, QColor(255, 255, 255))
    p.setColor(QPalette.ColorRole.ToolTipText, QColor(33, 33, 33))
    p.setColor(QPalette.ColorRole.Text, QColor(33, 33, 33))
    p.setColor(QPalette.ColorRole.Button, QColor(240, 240, 240))
    p.setColor(QPalette.ColorRole.ButtonText, QColor(33, 33, 33))
    p.setColor(QPalette.ColorRole.BrightText, QColor(200, 0, 0))
    p.setColor(QPalette.ColorRole.Link, QColor(0, 102, 204))
    p.setColor(QPalette.ColorRole.Highlight, QColor(0, 120, 215))
    p.setColor(QPalette.ColorRole.HighlightedText, QColor(255, 255, 255))
    app.setPalette(p)
    # Дополнительные стили для отдельных виджетов
    app.setStyleSheet("""
        QMainWindow, QWidget, QDialog { background-color: #f5f5f5; color: #212121; }
        QLineEdit, QTextEdit, QComboBox {
            background-color: #ffffff; color: #212121;
            border: 1px solid #cccccc; padding: 4px;
        }
        QPushButton {
            background-color: #e8e8e8; color: #212121;
            border: 1px solid #bbbbbb; padding: 6px 12px;
        }
        QPushButton:hover { background-color: #d0d0d0; }
        QFrame { background-color: #ffffff; color: #212121; }
        QScrollArea { background-color: #f5f5f5; border: none; }
        QToolBar { background-color: #eeeeee; border-bottom: 1px solid #cccccc; }
        QLabel { color: #212121; }
        QCheckBox { color: #212121; }
    """)


def parse_price(text):
    """
    Преобразует введённую пользователем строку в числовое значение цены.
    Допускается ввод через точку или запятую (450.50 и 450,50).
    """
    text = text.strip().replace(",", ".")
    if not text:
        raise ValueError("empty price")
    return float(text)


# --- Подключение к базе данных -----------------------------------------------

def db():
    """
    Создаёт и возвращает соединение с сервером MySQL.
    DictCursor позволяет обращаться к полям результата запроса по имени столбца.
    """
    return pymysql.connect(
        host="localhost",
        port=3306,
        user="root",
        password="root",
        database="restaurant_db",
        cursorclass=pymysql.cursors.DictCursor
    )


class DB:
    """
    Класс для работы с базой данных.
    Содержит методы выполнения запросов и сохранения заказов клиента.
    При инициализации проверяет наличие необходимых таблиц и столбцов.
    """

    def __init__(self):
        self.conn = db()
        self.ensure_schema()

    def ensure_schema(self):
        """
        Проверяет структуру базы данных и при необходимости дополняет её.
        Выполняется при каждом запуске приложения для совместимости
        со старыми версиями схемы без столбца photo и таблиц заказов.
        """
        with self.conn.cursor() as c:
            # Столбец для пути к изображению блюда
            c.execute("SHOW COLUMNS FROM MenuItems LIKE 'photo'")
            if not c.fetchone():
                c.execute("ALTER TABLE MenuItems ADD COLUMN photo VARCHAR(255)")
            # Таблица заказов (шапка заказа)
            c.execute("""
                CREATE TABLE IF NOT EXISTS Orders (
                    order_id INT PRIMARY KEY NOT NULL AUTO_INCREMENT,
                    user_id INT NOT NULL,
                    total DECIMAL(10, 2) NOT NULL,
                    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
                )
            """)
            # Таблица позиций заказа (состав корзины)
            c.execute("""
                CREATE TABLE IF NOT EXISTS OrderItems (
                    order_item_id INT PRIMARY KEY NOT NULL AUTO_INCREMENT,
                    order_id INT NOT NULL,
                    item_id INT NOT NULL,
                    quantity INT NOT NULL,
                    price DECIMAL(10, 2) NOT NULL
                )
            """)
        self.conn.commit()

    def get(self, q, p=None):
        """Выполняет SELECT-запрос и возвращает все найденные строки."""
        with self.conn.cursor() as c:
            c.execute(q, p or ())
            return c.fetchall()

    def one(self, q, p=None):
        """Выполняет SELECT-запрос и возвращает одну строку (или None)."""
        with self.conn.cursor() as c:
            c.execute(q, p or ())
            return c.fetchone()

    def run(self, q, p=None):
        """Выполняет INSERT, UPDATE или DELETE с фиксацией изменений (commit)."""
        with self.conn.cursor() as c:
            c.execute(q, p or ())
            self.conn.commit()

    def save_order(self, user_id, cart_items):
        """
        Сохраняет оформленный заказ клиента в базу данных.
        Сначала создаётся запись в Orders, затем — позиции в OrderItems.
        При ошибке выполняется откат транзакции (rollback).
        """
        total = sum(i['price'] * i['qty'] for i in cart_items)
        try:
            with self.conn.cursor() as c:
                c.execute(
                    "INSERT INTO Orders (user_id, total) VALUES (%s, %s)",
                    (user_id, total)
                )
                c.execute("SELECT LAST_INSERT_ID() AS order_id")
                order_id = c.fetchone()['order_id']
                for item in cart_items:
                    c.execute(
                        "INSERT INTO OrderItems (order_id, item_id, quantity, price) VALUES (%s, %s, %s, %s)",
                        (order_id, item['item_id'], item['qty'], item['price'])
                    )
                self.conn.commit()
            return order_id, total
        except Exception:
            self.conn.rollback()
            return None, None


# --- Вспомогательные функции интерфейса --------------------------------------

def photo_label(path, w=260, h=180):
    """
    Формирует виджет QLabel для отображения фотографии блюда.
    Если файл отсутствует, выводится текстовая заглушка.
    """
    lab = QLabel()
    lab.setFixedSize(w, h)
    lab.setAlignment(Qt.AlignmentFlag.AlignCenter)
    lab.setScaledContents(True)
    lab.setStyleSheet("border:1px solid gray; background:#f0f0f0")
    if path and os.path.exists(path):
        pix = QPixmap(path)
        pix = pix.scaled(w, h, Qt.AspectRatioMode.KeepAspectRatio,
                         Qt.TransformationMode.SmoothTransformation)
        lab.setPixmap(pix)
    else:
        lab.setText("Нет фото")
    return lab


# --- Окно авторизации --------------------------------------------------------

class Login(QWidget):
    """
    Форма входа в систему.
    Пользователь вводит логин и пароль; при успешной проверке
    открывается окно администратора или клиента в зависимости от role_id.
    """

    def __init__(self, db):
        super().__init__()
        self.db = db
        self.setWindowTitle("Вход")
        self.resize(300, 200)

        l = QVBoxLayout()
        l.addWidget(QLabel("Логин:"))
        self.u = QLineEdit()
        l.addWidget(self.u)
        l.addWidget(QLabel("Пароль:"))
        self.p = QLineEdit()
        self.p.setEchoMode(QLineEdit.EchoMode.Password)  # скрытый ввод пароля
        l.addWidget(self.p)

        b = QPushButton("Войти")
        b.clicked.connect(self.login)
        l.addWidget(b)

        exit_b = QPushButton("Выход")
        exit_b.clicked.connect(self.exit_app)
        l.addWidget(exit_b)

        self.setLayout(l)

    def exit_app(self):
        """Завершение работы приложения с экрана авторизации."""
        QApplication.instance().quit()

    def login(self):
        """
        Проверка учётных данных в таблице Users.
        При успехе открывается соответствующее окно, форма входа скрывается
        (не уничтожается), чтобы пользователь мог вернуться после выхода.
        """
        u = self.db.one(
            "SELECT * FROM Users WHERE username=%s AND password_hash=%s",
            (self.u.text(), self.p.text())
        )
        if u:
            if u['role_id'] == 1:
                # Администратор — полный доступ к управлению меню
                self.admin = Admin(self.db, u, self)
                self.admin.show()
            elif u['role_id'] == 2:
                # Клиент — просмотр меню и оформление заказов
                self.client = Client(self.db, u, self)
                self.client.show()
            else:
                QMessageBox.critical(self, "", "Неизвестная роль")
                return
            self.hide()
        else:
            QMessageBox.critical(self, "", "Неверно")


# --- Окно администратора -----------------------------------------------------

class Admin(QMainWindow):
    """
    Интерфейс администратора ресторана.
    Предоставляет возможность добавления, редактирования и удаления блюд,
    загрузки фотографий, сортировки по цене.
    """

    def __init__(self, db, user, login_win):
        super().__init__()
        self.db = db
        self.login_win = login_win       # ссылка на форму входа для возврата
        self._logging_out = False        # предотвращает двойной выход в closeEvent
        self.sort = None                 # текущий порядок сортировки: ASC / DESC
        self.setWindowTitle(f"Админ: {user['username']}")
        self.resize(900, 550)

        # Каталог для хранения загруженных изображений блюд
        if not os.path.exists('photos'):
            os.makedirs('photos')

        # Панель инструментов с основными действиями
        tb = self.addToolBar("")
        for t, f in [
            ("Добавить", self.add),
            ("Цена ↑", lambda: self.set_sort("ASC")),
            ("Цена ↓", lambda: self.set_sort("DESC")),
            ("Выход", self.logout),
        ]:
            b = QPushButton(t)
            b.clicked.connect(f)
            tb.addWidget(b)

        # Прокручиваемая область с карточками блюд
        self.scroll = QScrollArea()
        self.scroll.setWidgetResizable(True)
        self.w = QWidget()
        self.layout = QGridLayout(self.w)
        self.scroll.setWidget(self.w)
        self.setCentralWidget(self.scroll)
        self.load()

    def set_sort(self, order):
        """Устанавливает направление сортировки и обновляет список блюд."""
        self.sort = order
        self.load()

    def load(self):
        """
        Загружает блюда из таблицы MenuItems и отображает их в виде карточек.
        Карточки располагаются в сетке по три в ряд.
        """
        # Очистка предыдущих виджетов перед перерисовкой
        for i in reversed(range(self.layout.count())):
            self.layout.itemAt(i).widget().deleteLater()

        q = "SELECT * FROM MenuItems"
        if self.sort:
            q += f" ORDER BY price {self.sort}"
        items = self.db.get(q)

        r, c = 0, 0
        for x in items:
            card = QFrame()
            card.setFrameStyle(QFrame.Shape.Box)
            card.setFixedSize(280, 380)
            l = QVBoxLayout(card)

            l.addWidget(photo_label(x.get('photo')))
            l.addWidget(QLabel(f"<b>{x['name']}</b>"))
            l.addWidget(QLabel(f"Цена: {x['price']} руб"))
            status = "Доступен" if x['is_available'] else "Недоступен"
            l.addWidget(QLabel(status))

            b = QHBoxLayout()
            e = QPushButton("Ред.")
            e.clicked.connect(lambda ch, iid=x['item_id']: self.edit(iid))
            d = QPushButton("Удал.")
            d.clicked.connect(lambda ch, iid=x['item_id']: self.delete(iid))
            b.addWidget(e)
            b.addWidget(d)
            l.addLayout(b)

            self.layout.addWidget(card, r, c)
            c += 1
            if c >= 3:
                c = 0
                r += 1

    def add(self):
        """Открывает диалог добавления нового блюда."""
        self.dialog()

    def edit(self, item_id):
        """Открывает диалог редактирования существующего блюда."""
        self.dialog(item_id)

    def delete(self, item_id):
        """
        Удаляет блюдо из базы данных.
        При наличии связанного файла изображения он также удаляется с диска.
        """
        if QMessageBox.question(self, "", "Удалить?") == QMessageBox.StandardButton.Yes:
            row = self.db.one("SELECT photo FROM MenuItems WHERE item_id=%s", (item_id,))
            if row and row.get('photo') and os.path.exists(row['photo']):
                os.remove(row['photo'])
            self.db.run("DELETE FROM MenuItems WHERE item_id=%s", (item_id,))
            self.load()

    def dialog(self, item_id=None):
        """
        Модальное окно для создания или изменения блюда.
        Параметр item_id=None означает режим добавления новой записи.
        """
        d = QDialog(self)
        d.setWindowTitle("Блюдо")
        d.resize(400, 550)
        l = QVBoxLayout()

        n = QLineEdit()       # название блюда
        p = QLineEdit()       # цена
        a = QCheckBox("Доступен")
        a.setChecked(True)

        # Состояние фото: None — без изменений, "" — удалено, str — путь к файлу
        photo_path = None
        photo_preview = photo_label(None, 380, 250)
        photo_btn = QPushButton("Выбрать фото")
        del_photo_btn = QPushButton("Удалить фото")

        def choose():
            """Выбор файла изображения через стандартный диалог ОС."""
            nonlocal photo_path
            f, _ = QFileDialog.getOpenFileName(
                d, "Выбрать фото", "", "Images (*.png *.jpg *.jpeg *.bmp *.gif)"
            )
            if f:
                photo_path = f
                pix = QPixmap(f)
                pix = pix.scaled(380, 250, Qt.AspectRatioMode.KeepAspectRatio,
                                 Qt.TransformationMode.SmoothTransformation)
                photo_preview.setPixmap(pix)

        def remove_photo():
            """Помечает фото для удаления при сохранении."""
            nonlocal photo_path
            photo_path = ""
            photo_preview.clear()
            photo_preview.setText("Нет фото")

        photo_btn.clicked.connect(choose)
        del_photo_btn.clicked.connect(remove_photo)

        l.addWidget(QLabel("Название:"))
        l.addWidget(n)
        l.addWidget(QLabel("Цена:"))
        l.addWidget(p)
        l.addWidget(QLabel("Фото:"))
        l.addWidget(photo_preview)
        l.addWidget(photo_btn)
        l.addWidget(del_photo_btn)
        l.addWidget(a)

        # Заполнение полей при редактировании существующей записи
        if item_id:
            r = self.db.one("SELECT * FROM MenuItems WHERE item_id=%s", (item_id,))
            if r:
                n.setText(r['name'])
                p.setText(str(r['price']))
                a.setChecked(r['is_available'])
                if r.get('photo') and os.path.exists(r['photo']):
                    photo_path = r['photo']
                    pix = QPixmap(r['photo'])
                    pix = pix.scaled(380, 250, Qt.AspectRatioMode.KeepAspectRatio,
                                     Qt.TransformationMode.SmoothTransformation)
                    photo_preview.setPixmap(pix)

        def save_photo(target_id):
            """Копирует выбранный файл в локальную папку photos/."""
            if not photo_path or photo_path == "":
                return None
            name_clean = n.text().strip().replace(' ', '_').replace('/', '_') or 'item'
            ext = os.path.splitext(photo_path)[1] or '.jpg'
            dest = f"photos/{name_clean}_{target_id}{ext}"
            shutil.copy2(photo_path, dest)
            return dest

        def save():
            """Сохранение данных блюда в таблицу MenuItems."""
            try:
                name = n.text().strip()
                if not name:
                    QMessageBox.critical(d, "Ошибка", "Введите название")
                    return
                price = parse_price(p.text())

                if item_id:
                    # --- Режим редактирования ---
                    if photo_path == "":
                        old = self.db.one("SELECT photo FROM MenuItems WHERE item_id=%s", (item_id,))
                        if old and old.get('photo') and os.path.exists(old['photo']):
                            os.remove(old['photo'])
                        new_photo = None
                    elif photo_path:
                        old = self.db.one("SELECT photo FROM MenuItems WHERE item_id=%s", (item_id,))
                        if old and old.get('photo') and old['photo'] != photo_path and os.path.exists(old['photo']):
                            os.remove(old['photo'])
                        new_photo = save_photo(item_id)
                    else:
                        old = self.db.one("SELECT photo FROM MenuItems WHERE item_id=%s", (item_id,))
                        new_photo = old.get('photo') if old else None

                    self.db.run(
                        "UPDATE MenuItems SET name=%s, price=%s, is_available=%s, photo=%s WHERE item_id=%s",
                        (name, price, a.isChecked(), new_photo, item_id)
                    )
                else:
                    # --- Режим добавления ---
                    # Сначала создаём запись, затем получаем item_id для имени файла фото
                    self.db.run(
                        "INSERT INTO MenuItems (name, price, is_available, photo) VALUES (%s, %s, %s, %s)",
                        (name, price, a.isChecked(), None)
                    )
                    if photo_path and photo_path != "":
                        row = self.db.one("SELECT LAST_INSERT_ID() AS item_id")
                        new_photo = save_photo(row['item_id'])
                        self.db.run(
                            "UPDATE MenuItems SET photo=%s WHERE item_id=%s",
                            (new_photo, row['item_id'])
                        )

                d.accept()
                self.load()
            except ValueError:
                QMessageBox.critical(d, "Ошибка", "Проверьте цену (например: 450 или 450.50)")
            except Exception as e:
                QMessageBox.critical(d, "Ошибка", str(e))

        l.addWidget(QPushButton("Сохранить", clicked=save))
        d.setLayout(l)
        d.exec()

    def logout(self):
        """Выход из учётной записи администратора, возврат на форму входа."""
        self.login_win.u.clear()
        self.login_win.p.clear()
        self.login_win.show()
        self.login_win.raise_()
        self._logging_out = True
        self.close()

    def closeEvent(self, event):
        """
        Обработка закрытия окна кнопкой «крестик».
        Если пользователь не нажимал «Выход», выполняется возврат к авторизации.
        """
        if self.login_win.isHidden() and not self._logging_out:
            self.login_win.u.clear()
            self.login_win.p.clear()
            self.login_win.show()
            self.login_win.raise_()
        event.accept()


# --- Окно клиента ------------------------------------------------------------

class Client(QMainWindow):
    """
    Интерфейс клиента ресторана.
    Клиент может просматривать меню, искать блюда, сортировать по цене,
    добавлять позиции в корзину и оформлять заказ.
    Корзина хранится в оперативной памяти до момента оформления.
    """

    def __init__(self, db, user, login_win):
        super().__init__()
        self.db = db
        self.login_win = login_win
        self._logging_out = False
        self.user = user
        self.sort = None
        self.cart = {}  # словарь: item_id -> данные позиции корзины
        self.setWindowTitle(f"Клиент: {user['username']}")
        self.resize(900, 550)

        w = QWidget()
        self.setCentralWidget(w)
        l = QVBoxLayout(w)

        # Верхняя панель: поиск, сортировка, корзина, выход
        h = QHBoxLayout()
        self.s = QLineEdit()
        self.s.returnPressed.connect(self.search)
        h.addWidget(QLabel("Поиск:"))
        h.addWidget(self.s)
        h.addWidget(QPushButton("Найти", clicked=self.search))
        h.addWidget(QPushButton("Все", clicked=lambda: self.load()))
        h.addWidget(QPushButton("Цена ↑", clicked=lambda: self.set_sort("ASC")))
        h.addWidget(QPushButton("Цена ↓", clicked=lambda: self.set_sort("DESC")))
        self.cart_btn = QPushButton("Корзина (0)")
        self.cart_btn.clicked.connect(self.show_cart)
        h.addWidget(self.cart_btn)
        b = QPushButton("Выход")
        b.clicked.connect(self.logout)
        h.addWidget(b)
        l.addLayout(h)

        self.scroll = QScrollArea()
        self.scroll.setWidgetResizable(True)
        self.w2 = QWidget()
        self.layout2 = QGridLayout(self.w2)
        self.scroll.setWidget(self.w2)
        l.addWidget(self.scroll)
        self.load()

    def cart_count(self):
        """Общее количество единиц товара в корзине."""
        return sum(v['qty'] for v in self.cart.values())

    def cart_total(self):
        """Итоговая сумма заказа по текущему содержимому корзины."""
        return sum(v['price'] * v['qty'] for v in self.cart.values())

    def update_cart_btn(self):
        """Обновляет надпись на кнопке корзины с актуальным количеством."""
        self.cart_btn.setText(f"Корзина ({self.cart_count()})")

    def cart_inc(self, iid, qty_lbl, sum_lbl, total_lbl):
        """Увеличение количества выбранной позиции на единицу."""
        if iid not in self.cart:
            return
        self.cart[iid]['qty'] += 1
        qty_lbl.setText(str(self.cart[iid]['qty']))
        sum_lbl.setText(f"= {self.cart[iid]['price'] * self.cart[iid]['qty']:.0f} руб")
        total_lbl.setText(f"Итого: {self.cart_total():.0f} руб")
        self.update_cart_btn()

    def cart_dec(self, iid, qty_lbl, sum_lbl, total_lbl):
        """Уменьшение количества; минимально допустимое значение — 1."""
        if iid not in self.cart or self.cart[iid]['qty'] <= 1:
            return
        self.cart[iid]['qty'] -= 1
        qty_lbl.setText(str(self.cart[iid]['qty']))
        sum_lbl.setText(f"= {self.cart[iid]['price'] * self.cart[iid]['qty']:.0f} руб")
        total_lbl.setText(f"Итого: {self.cart_total():.0f} руб")
        self.update_cart_btn()

    def cart_remove(self, iid, row, total_lbl, empty_lbl):
        """Полное удаление позиции из корзины."""
        if iid in self.cart:
            del self.cart[iid]
        row.deleteLater()
        total_lbl.setText(f"Итого: {self.cart_total():.0f} руб")
        self.update_cart_btn()
        if not self.cart:
            empty_lbl.show()

    def checkout(self, dialog):
        """
        Оформление заказа: запись в таблицы Orders и OrderItems.
        После успешного сохранения корзина очищается.
        """
        order_id, total = self.db.save_order(self.user['user_id'], list(self.cart.values()))
        if order_id:
            self.cart.clear()
            self.update_cart_btn()
            dialog.accept()
            QMessageBox.information(self, "", f"Заказ №{order_id} на {total:.0f} руб оформлен!")
        else:
            QMessageBox.critical(self, "", "Не удалось сохранить заказ в базу данных")

    def logout(self):
        """Выход из учётной записи клиента, возврат на форму входа."""
        self.login_win.u.clear()
        self.login_win.p.clear()
        self.login_win.show()
        self.login_win.raise_()
        self._logging_out = True
        self.close()

    def closeEvent(self, event):
        """При закрытии окна крестиком — возврат к экрану авторизации."""
        if self.login_win.isHidden() and not self._logging_out:
            self.login_win.u.clear()
            self.login_win.p.clear()
            self.login_win.show()
            self.login_win.raise_()
        event.accept()

    def add_to_cart(self, item):
        """
        Добавление блюда в корзину.
        Если блюдо уже присутствует, увеличивается количество (qty).
        """
        iid = item['item_id']
        if iid in self.cart:
            self.cart[iid]['qty'] += 1
        else:
            self.cart[iid] = {
                'item_id': iid,
                'name': item['name'],
                'price': float(item['price']),
                'qty': 1
            }
        self.update_cart_btn()
        QMessageBox.information(self, "", f"«{item['name']}» добавлено в корзину")

    def set_sort(self, order):
        """Сортировка каталога по цене с учётом текущего поискового запроса."""
        self.sort = order
        self.load(self.s.text() or None)

    def load(self, s=None):
        """
        Отображение каталога доступных блюд (is_available = 1).
        Параметр s — необязательная подстрока для фильтрации по названию.
        """
        for i in reversed(range(self.layout2.count())):
            self.layout2.itemAt(i).widget().deleteLater()

        order = f" ORDER BY price {self.sort}" if self.sort else ""
        if s:
            items = self.db.get(
                f"SELECT * FROM MenuItems WHERE is_available=1 AND name LIKE %s{order}",
                (f"%{s}%",)
            )
        else:
            items = self.db.get(f"SELECT * FROM MenuItems WHERE is_available=1{order}")

        r, c = 0, 0
        for x in items:
            card = QFrame()
            card.setFrameStyle(QFrame.Shape.Box)
            card.setFixedSize(280, 390)
            l2 = QVBoxLayout(card)

            l2.addWidget(photo_label(x.get('photo')))
            l2.addWidget(QLabel(f"<b>{x['name']}</b>"))
            l2.addWidget(QLabel(f"Цена: {x['price']} руб"))

            btns = QHBoxLayout()
            btns.addWidget(QPushButton("Подробнее", clicked=lambda ch, iid=x['item_id']: self.detail(iid)))
            btns.addWidget(QPushButton("В корзину", clicked=lambda ch, item=x: self.add_to_cart(item)))
            l2.addLayout(btns)

            self.layout2.addWidget(card, r, c)
            c += 1
            if c >= 3:
                c = 0
                r += 1

    def search(self):
        """Поиск блюд по введённому названию."""
        self.load(self.s.text())

    def detail(self, item_id):
        """Отображение подробной информации о выбранном блюде."""
        x = self.db.one("SELECT * FROM MenuItems WHERE item_id=%s", (item_id,))
        if x:
            d = QDialog(self)
            d.setWindowTitle(x['name'])
            d.resize(450, 520)
            l = QVBoxLayout()
            if x.get('photo') and os.path.exists(x['photo']):
                lab = QLabel()
                lab.setAlignment(Qt.AlignmentFlag.AlignCenter)
                lab.setScaledContents(True)
                pix = QPixmap(x['photo'])
                pix = pix.scaled(400, 300, Qt.AspectRatioMode.KeepAspectRatio,
                                 Qt.TransformationMode.SmoothTransformation)
                lab.setPixmap(pix)
                l.addWidget(lab)
            l.addWidget(QLabel(f"<b>{x['name']}</b>"))
            l.addWidget(QLabel(f"Цена: {x['price']} руб"))
            btns = QHBoxLayout()
            btns.addWidget(QPushButton("В корзину", clicked=lambda: (self.add_to_cart(x), d.accept())))
            btns.addWidget(QPushButton("Закрыть", clicked=d.accept))
            l.addLayout(btns)
            d.setLayout(l)
            d.exec()

    def show_cart(self):
        """
        Диалоговое окно корзины.
        Позволяет изменить количество, удалить позиции и оформить заказ.
        """
        d = QDialog(self)
        d.setWindowTitle("Корзина")
        d.resize(520, 480)
        l = QVBoxLayout()

        if not self.cart:
            l.addWidget(QLabel("Корзина пуста"))
            l.addWidget(QPushButton("Закрыть", clicked=d.accept))
            d.setLayout(l)
            d.exec()
            return

        scroll = QScrollArea()
        scroll.setWidgetResizable(True)
        box = QWidget()
        box_l = QVBoxLayout(box)

        total_lbl = QLabel(f"Итого: {self.cart_total():.0f} руб")
        total_lbl.setStyleSheet("font-size: 14px; font-weight: bold;")
        empty_lbl = QLabel("Корзина пуста")
        empty_lbl.hide()

        for iid, item in list(self.cart.items()):
            row = QFrame()
            row.setFrameStyle(QFrame.Shape.Box)
            row_l = QHBoxLayout(row)

            row_l.addWidget(QLabel(f"<b>{item['name']}</b>"))
            row_l.addWidget(QLabel(f"{item['price']:.0f} руб"))

            qty_lbl = QLabel(str(item['qty']))
            qty_lbl.setFixedWidth(24)
            qty_lbl.setAlignment(Qt.AlignmentFlag.AlignCenter)
            sum_lbl = QLabel(f"= {item['price'] * item['qty']:.0f} руб")

            qty_l = QHBoxLayout()
            # Символ _ принимает служебный аргумент checked от сигнала QPushButton.clicked
            qty_l.addWidget(QPushButton("-", clicked=lambda _, i=iid, q=qty_lbl, s=sum_lbl, t=total_lbl: self.cart_dec(i, q, s, t)))
            qty_l.addWidget(qty_lbl)
            qty_l.addWidget(QPushButton("+", clicked=lambda _, i=iid, q=qty_lbl, s=sum_lbl, t=total_lbl: self.cart_inc(i, q, s, t)))
            row_l.addLayout(qty_l)
            row_l.addWidget(sum_lbl)
            row_l.addWidget(QPushButton("Убрать", clicked=lambda _, i=iid, r=row, t=total_lbl, e=empty_lbl: self.cart_remove(i, r, t, e)))
            box_l.addWidget(row)

        scroll.setWidget(box)
        l.addWidget(scroll)
        l.addWidget(total_lbl)
        l.addWidget(empty_lbl)

        btns = QHBoxLayout()
        btns.addWidget(QPushButton("Очистить", clicked=lambda: (self.cart.clear(), self.update_cart_btn(), d.accept(), self.show_cart())))
        btns.addWidget(QPushButton("Оформить", clicked=lambda: self.checkout(d)))
        btns.addWidget(QPushButton("Закрыть", clicked=d.accept))
        l.addLayout(btns)

        d.setLayout(l)
        d.exec()


# --- Точка входа в программу -------------------------------------------------

def main():
    """Запуск приложения (используется при установке через pip: qtds-ui)."""
    app = QApplication(sys.argv)
    app.setQuitOnLastWindowClosed(False)
    apply_light_theme(app)
    login = Login(DB())
    login.show()
    login.raise_()
    sys.exit(app.exec())


if __name__ == "__main__":
    main()


"""
-- SQL для MySQL (выполнить в Workbench перед первым запуском):
--
-- Таблицы: Users (пользователи), MenuItems (меню), Orders (заказы),
--          OrderItems (состав заказа)
-- Учётные записи: admin/admin123 (role_id=1), client/client123 (role_id=2)

CREATE DATABASE IF NOT EXISTS restaurant_db;
USE restaurant_db;

CREATE TABLE IF NOT EXISTS Users (
    user_id INT PRIMARY KEY NOT NULL AUTO_INCREMENT,
    username VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role_id INT NOT NULL
);

CREATE TABLE IF NOT EXISTS MenuItems (
    item_id INT PRIMARY KEY NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    is_available BOOLEAN DEFAULT TRUE,
    photo VARCHAR(255)
);

CREATE TABLE IF NOT EXISTS Orders (
    order_id INT PRIMARY KEY NOT NULL AUTO_INCREMENT,
    user_id INT NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS OrderItems (
    order_item_id INT PRIMARY KEY NOT NULL AUTO_INCREMENT,
    order_id INT NOT NULL,
    item_id INT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);

INSERT INTO Users (username, password_hash, role_id) VALUES
('admin', 'admin123', 1),
('client', 'client123', 2);

INSERT INTO MenuItems (name, price, is_available, photo) VALUES
('Маргарита', 450.00, TRUE, NULL),
('Пепперони', 550.00, TRUE, NULL),
('Гавайская', 520.00, TRUE, NULL),
('Четыре сыра', 600.00, TRUE, NULL),
('Диабло', 580.00, TRUE, NULL),
('Мясная', 650.00, TRUE, NULL),
('Вегетарианская', 480.00, TRUE, NULL),
('Морская', 700.00, TRUE, NULL),
('Барбекю', 590.00, TRUE, NULL),
('Сладкая', 350.00, TRUE, NULL);
"""






Инструкция для пользователей приложения управления рестораном

Добро пожаловать в приложение для управления рестораном. Эта программа предназначена для двух типов пользователей администраторов и клиентов. Чтобы начать работу запустите файл main.py. При запуске откроется окно входа в систему где вам нужно ввести логин и пароль. Если вы администратор используйте логин admin и пароль admin123. Если вы клиент используйте логин client и пароль client123.

После успешного входа программа определит вашу роль и откроет соответствующий интерфейс.

Для администратора

Интерфейс администратора позволяет управлять меню ресторана. Вы увидите панель инструментов с кнопками и область где карточками отображаются все блюда. Каждая карточка содержит фотографию блюда если она загружена название цену и статус доступности. Под карточкой есть две кнопки Ред и Удал для редактирования или удаления блюда.

Кнопка Добавить на панели инструментов открывает окно для создания нового блюда. В этом окне нужно ввести название блюда указать цену цифрами можно использовать точку или запятую выбрать доступно или недоступно и при желании загрузить фотографию. После заполнения всех полей нажмите Сохранить и блюдо появится в общем списке.

Чтобы отредактировать существующее блюдо нажмите кнопку Ред на его карточке. Откроется такое же окно но уже с заполненными данными. Вы можете изменить название цену статус доступности или заменить фотографию. Если вы хотите удалить фотографию нажмите кнопку Удалить фото. После внесения изменений нажмите Сохранить.

Кнопка Удал на карточке блюда полностью удаляет это блюдо из базы данных вместе с его фотографией если она есть. Перед удалением программа запросит подтверждение.

Кнопки Цена и Цена на панели инструментов сортируют все блюда по возрастанию или убыванию цены. Это помогает быстро найти самые дорогие или самые дешёвые позиции.

Кнопка Выход закрывает окно администратора и возвращает вас на экран входа в систему.

Для клиента

Интерфейс клиента показывает каталог доступных блюд. Отображаются только те блюда которые администратор отметил как доступные. Каждая карточка блюда содержит фотографию название цену и две кнопки Подробнее и В корзину.

В верхней части окна есть поле для поиска. Введите часть названия блюда и нажмите кнопку Найти или просто нажмите клавишу Enter чтобы отфильтровать меню. Кнопка Все отменяет поиск и показывает все доступные блюда. Кнопки Цена и Цена сортируют блюда по цене как и у администратора.

Чтобы посмотреть подробную информацию о блюде нажмите кнопку Подробнее. Откроется отдельное окно с увеличенной фотографией названием и ценой. Там же есть кнопка В корзину для добавления.

Чтобы добавить блюдо в заказ нажмите кнопку В корзину на карточке блюда или в окне подробной информации. Программа покажет сообщение что блюдо добавлено. Количество добавленных блюд отображается на кнопке Корзина в верхней панели.

Чтобы просмотреть текущий заказ нажмите кнопку Корзина. Откроется окно где перечислены все выбранные блюда их цены количество и итоговая сумма. Для каждого блюда вы можете увеличить или уменьшить количество с помощью кнопок плюс и минус. Кнопка Убрать полностью удаляет блюдо из корзины. Кнопка Очистить удаляет все блюда сразу. Если вы передумали делать заказ просто закройте окно корзины.

Когда вы готовы оформить заказ нажмите кнопку Оформить в окне корзины. Программа сохранит заказ в базу данных присвоит ему уникальный номер и покажет сообщение с номером заказа и итоговой суммой. После этого корзина автоматически очищается. Вы можете продолжать выбирать блюда и оформлять новые заказы.

Кнопка Выход закрывает окно клиента и возвращает вас на экран входа в систему.

Общие правила работы

Фотографии блюд сохраняются в папку photos которая создаётся автоматически в той же папке где находится программа. При переносе программы на другой компьютер обязательно копируйте эту папку вместе с файлом программы иначе фотографии не отобразятся.

Цену блюда можно вводить как с точкой 450.50 так и с запятой 450 50. Пустые поля или буквы не допускаются.

Если при запуске программы возникает ошибка подключения к базе данных убедитесь что установлен и запущен MySQL сервер. На Windows это можно проверить через службы в панели управления. На компьютере с XAMPP нужно запустить модуль MySQL через панель управления XAMPP. На компьютере с Linux используйте команду sudo systemctl start mysql.

Для корректной работы программы необходимо предварительно создать базу данных restaurant_db и выполнить SQL скрипт который создаёт все необходимые таблицы и заполняет их тестовыми данными. Этот скрипт приведён в комментариях в конце файла main.py.

При возникновении любых ошибок программа покажет сообщение с пояснением. Внимательно читайте эти сообщения они помогут понять причину проблемы. Если вы не можете решить проблему обратитесь к администратору системы.
