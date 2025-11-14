"""
dam_pyqt5.py
A simple desktop DAM prototype with:
1) Authentication (Admin/Editor/Viewer)
3) Metadata (custom fields, tags)
4) Search (keyword, tag filters, date range)

Dependencies:
- PyQt5
- SQLAlchemy

Run:
pip install PyQt5 SQLAlchemy
python dam_pyqt5.py
"""

import sys
import os
import hashlib, binascii, datetime
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QLineEdit, QTableWidget, QTableWidgetItem,
    QDialog, QMessageBox, QFormLayout, QComboBox, QTextEdit,
    QListWidget, QListWidgetItem, QDateEdit, QGroupBox, QInputDialog
)
from PyQt5.QtCore import Qt, QDate
from sqlalchemy import (
    create_engine, Column, Integer, String, DateTime, ForeignKey, Table, Text
)
from sqlalchemy.orm import declarative_base, relationship, sessionmaker

# ---------- Database models (SQLAlchemy) ----------
Base = declarative_base()

# association table for many-to-many Asset <-> Tag
asset_tag_table = Table(
    'asset_tag', Base.metadata,   # ✅ must remain singular
    Column('asset_id', Integer, ForeignKey('assets.id'), primary_key=True),
    Column('tag_id', Integer, ForeignKey('tags.id'), primary_key=True)
)

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    username = Column(String, unique=True, nullable=False)
    pw_hash = Column(String, nullable=False)
    role = Column(String, nullable=False)  # Admin, Editor, Viewer

class Asset(Base):
    __tablename__ = 'assets'
    id = Column(Integer, primary_key=True)
    title = Column(String, nullable=False)
    description = Column(Text, default="")
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    metadatas = relationship("Metadata", back_populates="asset", cascade="all, delete-orphan")
    tags = relationship("Tag", secondary=asset_tag_table, back_populates="assets")

class Metadata(Base):
    __tablename__ = 'metadata'
    id = Column(Integer, primary_key=True)
    asset_id = Column(Integer, ForeignKey('assets.id'), nullable=False)
    key = Column(String, nullable=False)
    value = Column(String, nullable=True)
    asset = relationship("Asset", back_populates="metadatas")

class Tag(Base):
    __tablename__ = 'tags'
    id = Column(Integer, primary_key=True)
    name = Column(String, unique=True, nullable=False)
    assets = relationship("Asset", secondary=asset_tag_table, back_populates="tags")

# ---------- Utilities: password hashing ----------
def hash_password(password: str, salt: bytes = None):
    """Return salt + hash hex string."""
    if salt is None:
        salt = os.urandom(16)
    dk = hashlib.pbkdf2_hmac('sha256', password.encode(), salt, 100_000)
    return binascii.hexlify(salt + dk).decode()

def verify_password(stored: str, provided: str) -> bool:
    data = binascii.unhexlify(stored.encode())
    salt = data[:16]
    dk = data[16:]
    provided_dk = hashlib.pbkdf2_hmac('sha256', provided.encode(), salt, 100_000)
    return provided_dk == dk

# ---------- DB init ----------
DB_FILE = "dam_local.db"
engine = create_engine(f"sqlite:///{DB_FILE}", echo=False, connect_args={"check_same_thread": False})
Session = sessionmaker(bind=engine)
Base.metadata.create_all(engine)

# ---------- Seed sample data ----------
def seed_if_needed():
    s = Session()
    if s.query(User).count() == 0:
        admin = User(username="admin", pw_hash=hash_password("admin123"), role="Admin")
        editor = User(username="editor", pw_hash=hash_password("editor123"), role="Editor")
        viewer = User(username="viewer", pw_hash=hash_password("viewer123"), role="Viewer")

        # sample tags and assets
        t1 = Tag(name="invoice")
        t2 = Tag(name="marketing")

        a1 = Asset(title="Invoice Jan", description="Client invoice for January")
        a1.tags = [t1]
        a1.metadatas = [
            Metadata(key="client", value="ACME Corp"),
            Metadata(key="amount", value="1500")
        ]

        a2 = Asset(title="Campaign Image", description="Hero image for landing page")
        a2.tags = [t2]
        a2.metadatas = [
            Metadata(key="resolution", value="1920x1080"),
            Metadata(key="color", value="vibrant")
        ]

        s.add_all([admin, editor, viewer, t1, t2, a1, a2])
        s.commit()
    s.close()

seed_if_needed()

# ---------- GUI components ----------
class LoginDialog(QDialog):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setWindowTitle("Login")
        self.resize(300, 120)
        layout = QFormLayout()
        self.username = QLineEdit()
        self.password = QLineEdit()
        self.password.setEchoMode(QLineEdit.Password)
        layout.addRow("Username:", self.username)
        layout.addRow("Password:", self.password)
        btn_login = QPushButton("Login")
        btn_login.clicked.connect(self.try_login)
        layout.addRow(btn_login)
        self.setLayout(layout)
        self.user = None

    def try_login(self):
        s = Session()
        username = self.username.text().strip()
        pw = self.password.text()
        user = s.query(User).filter_by(username=username).first()
        s.close()
        if not user or not verify_password(user.pw_hash, pw):
            QMessageBox.warning(self, "Login failed", "Incorrect username or password")
            return
        self.user = user
        self.accept()

class MetadataEditorDialog(QDialog):
    def __init__(self, session, asset, parent=None, role="Editor"):
        super().__init__(parent)
        self.setWindowTitle(f"Metadata for: {asset.title}")
        self.session = session
        self.asset = asset
        self.role = role
        self.resize(400, 300)
        layout = QVBoxLayout()
        self.meta_list = QListWidget()
        self.meta_items = []
        for m in asset.metadatas:
            it = QListWidgetItem(f"{m.key} = {m.value}")
            it.setData(Qt.UserRole, m)
            self.meta_list.addItem(it)
        layout.addWidget(self.meta_list)

        btn_box = QHBoxLayout()
        self.add_btn = QPushButton("Add")
        self.edit_btn = QPushButton("Edit")
        self.del_btn = QPushButton("Delete")
        self.close_btn = QPushButton("Close")
        btn_box.addWidget(self.add_btn)
        btn_box.addWidget(self.edit_btn)
        btn_box.addWidget(self.del_btn)
        btn_box.addWidget(self.close_btn)
        layout.addLayout(btn_box)
        self.setLayout(layout)

        self.add_btn.clicked.connect(self.add_meta)
        self.edit_btn.clicked.connect(self.edit_meta)
        self.del_btn.clicked.connect(self.del_meta)
        self.close_btn.clicked.connect(self.on_close)

        if self.role == "Viewer":
            self.add_btn.setEnabled(False)
            self.edit_btn.setEnabled(False)
            self.del_btn.setEnabled(False)

    def add_meta(self):
        k, ok = QInputDialog.getText(self, "Key", "Metadata key:")
        if not ok or not k.strip():
            return
        v, ok = QInputDialog.getText(self, "Value", "Metadata value:")
        if not ok:
            return
        meta = Metadata(asset=self.asset, key=k.strip(), value=v)
        self.session.add(meta)
        self.session.commit()
        self.meta_list.addItem(QListWidgetItem(f"{meta.key} = {meta.value}"))

    def edit_meta(self):
        it = self.meta_list.currentItem()
        if not it:
            return
        meta = it.data(Qt.UserRole)
        new_k, ok = QInputDialog.getText(self, "Key", "Metadata key:", text=meta.key)
        if not ok or not new_k.strip():
            return
        new_v, ok = QInputDialog.getText(self, "Value", "Metadata value:", text=meta.value)
        if not ok:
            return
        meta.key = new_k.strip()
        meta.value = new_v
        self.session.commit()
        it.setText(f"{meta.key} = {meta.value}")

    def del_meta(self):
        it = self.meta_list.currentItem()
        if not it:
            return
        meta = it.data(Qt.UserRole)
        self.session.delete(meta)
        self.session.commit()
        self.meta_list.takeItem(self.meta_list.row(it))

    def on_close(self):
        self.accept()

class MainWindow(QMainWindow):
    def __init__(self, user):
        super().__init__()
        self.user = user
        self.setWindowTitle(f"DAM Desktop - Logged in as {user.username} ({user.role})")
        self.resize(900, 600)
        self.session = Session()

        main = QWidget()
        vbox = QVBoxLayout()

        # --- Search / filters ---
        search_box = QHBoxLayout()
        self.search_input = QLineEdit()
        self.search_input.setPlaceholderText("Search title, description, metadata...")
        self.search_btn = QPushButton("Search")
        self.search_btn.clicked.connect(self.perform_search)
        search_box.addWidget(QLabel("Keyword:"))
        search_box.addWidget(self.search_input)
        search_box.addWidget(self.search_btn)

        # tag filter list
        tag_box = QVBoxLayout()
        tag_box.addWidget(QLabel("Tags: (check to filter)"))
        self.tag_list = QListWidget()
        self.tag_list.setSelectionMode(QListWidget.MultiSelection)
        tag_box.addWidget(self.tag_list)
        # date range
        date_box = QHBoxLayout()
        date_box.addWidget(QLabel("From:"))
        self.date_from = QDateEdit()
        self.date_from.setCalendarPopup(True)
        self.date_from.setDate(QDate.currentDate().addYears(-1))
        date_box.addWidget(self.date_from)
        date_box.addWidget(QLabel("To:"))
        self.date_to = QDateEdit()
        self.date_to.setCalendarPopup(True)
        self.date_to.setDate(QDate.currentDate())
        date_box.addWidget(self.date_to)

        filters = QHBoxLayout()
        filters.addLayout(tag_box, 1)
        filters.addLayout(date_box, 2)

        vbox.addLayout(search_box)
        vbox.addLayout(filters)

        # --- Asset table ---
        self.table = QTableWidget(0, 4)
        self.table.setHorizontalHeaderLabels(["ID", "Title", "Description", "Created At / Tags"])
        self.table.cellDoubleClicked.connect(self.on_asset_double)
        vbox.addWidget(self.table)

        # --- Buttons based on roles ---
        btns = QHBoxLayout()
        self.btn_add = QPushButton("Add Asset")
        self.btn_edit = QPushButton("Edit Asset")
        self.btn_delete = QPushButton("Delete Asset")
        self.btn_metadata = QPushButton("Edit Metadata")
        self.btn_manage_users = QPushButton("Manage Users")
        btns.addWidget(self.btn_add)
        btns.addWidget(self.btn_edit)
        btns.addWidget(self.btn_delete)
        btns.addWidget(self.btn_metadata)
        btns.addWidget(self.btn_manage_users)
        vbox.addLayout(btns)

        main.setLayout(vbox)
        self.setCentralWidget(main)

        # wire buttons
        self.btn_add.clicked.connect(self.add_asset)
        self.btn_edit.clicked.connect(self.edit_asset)
        self.btn_delete.clicked.connect(self.delete_asset)
        self.btn_metadata.clicked.connect(self.open_metadata)
        self.btn_manage_users.clicked.connect(self.manage_users)

        # role-based enabling
        if self.user.role == "Viewer":
            self.btn_add.setEnabled(False)
            self.btn_edit.setEnabled(False)
            self.btn_delete.setEnabled(False)
            self.btn_manage_users.setEnabled(False)
        elif self.user.role == "Editor":
            self.btn_manage_users.setEnabled(False)

        self.refresh_tags()
        self.perform_search()  # load initial table

    def refresh_tags(self):
        self.tag_list.clear()
        tags = self.session.query(Tag).order_by(Tag.name).all()
        for t in tags:
            item = QListWidgetItem(t.name)
            item.setData(Qt.UserRole, t)
            item.setCheckState(Qt.Unchecked)
            self.tag_list.addItem(item)

    def perform_search(self):
        kw = self.search_input.text().strip().lower()
        # gather chosen tags
        chosen_tags = []
        for i in range(self.tag_list.count()):
            it = self.tag_list.item(i)
            if it.checkState() == Qt.Checked:
                chosen_tags.append(it.data(Qt.UserRole).name)
        date_from_py = self.date_from.date().toPyDate()
        date_to_py = self.date_to.date().toPyDate()
        # build query: start broad, then filter in python for metadata searching convenience
        assets = self.session.query(Asset).order_by(Asset.created_at.desc()).all()
        results = []
        for a in assets:
            # date filter
            created_date = a.created_at.date() if a.created_at else None
            if created_date is None:
                continue
            if created_date < date_from_py or created_date > date_to_py:
                continue
            # tag filter
            asset_tag_names = [t.name for t in a.tags]
            if chosen_tags and not set(chosen_tags).issubset(set(asset_tag_names)):
                continue
            # keyword filter
            if kw:
                in_title = kw in (a.title or "").lower()
                in_desc = kw in (a.description or "").lower()
                in_meta = any(kw in (m.value or "").lower() or kw in (m.key or "").lower() for m in a.metadatas)
                if not (in_title or in_desc or in_meta):
                    continue
            results.append(a)
        self.populate_table(results)

    def populate_table(self, assets):
        self.table.setRowCount(0)
        for a in assets:
            r = self.table.rowCount()
            self.table.insertRow(r)
            self.table.setItem(r, 0, QTableWidgetItem(str(a.id)))
            self.table.setItem(r, 1, QTableWidgetItem(a.title))
            self.table.setItem(r, 2, QTableWidgetItem(a.description))
            tag_str = ", ".join([t.name for t in a.tags])
            created = a.created_at.strftime("%Y-%m-%d %H:%M:%S") if a.created_at else ""
            self.table.setItem(r, 3, QTableWidgetItem(f"{created}\nTags: {tag_str}"))

    def get_selected_asset(self):
        row = self.table.currentRow()
        if row < 0:
            QMessageBox.information(self, "Select asset", "Please select an asset row first.")
            return None
        aid = int(self.table.item(row, 0).text())
        return self.session.query(Asset).get(aid)

    def on_asset_double(self, row, col):
        aid = int(self.table.item(row, 0).text())
        asset = self.session.query(Asset).get(aid)
        self.show_asset_details(asset)

    def show_asset_details(self, asset):
        txt = f"Title: {asset.title}\n\nDescription:\n{asset.description}\n\nCreated: {asset.created_at}\n\nTags: {', '.join([t.name for t in asset.tags])}\n\nMetadata:\n"
        for m in asset.metadatas:
            txt += f"- {m.key}: {m.value}\n"
        dlg = QMessageBox(self)
        dlg.setWindowTitle("Asset details")
        dlg.setText(txt)
        dlg.exec_()

    def add_asset(self):
        title, ok = QInputDialog.getText(self, "New asset", "Title:")
        if not ok or not title.strip():
            return
        desc, ok = QInputDialog.getMultiLineText(self, "New asset", "Description:")
        if not ok:
            desc = ""
        a = Asset(title=title.strip(), description=desc, created_at=datetime.datetime.utcnow())
        # pick tags (simple multi-select from existing tags)
        tags = [self.tag_list.item(i).data(Qt.UserRole) for i in range(self.tag_list.count()) if self.tag_list.item(i).checkState() == Qt.Checked]
        a.tags = tags
        self.session.add(a)
        self.session.commit()
        self.perform_search()

    def edit_asset(self):
        asset = self.get_selected_asset()
        if not asset: return
        title, ok = QInputDialog.getText(self, "Edit asset", "Title:", text=asset.title)
        if not ok or not title.strip():
            return
        desc, ok = QInputDialog.getMultiLineText(self, "Edit asset", "Description:", text=asset.description)
        if not ok:
            desc = asset.description
        asset.title = title.strip()
        asset.description = desc
        self.session.commit()
        self.perform_search()

    def delete_asset(self):
        if self.user.role == "Viewer":
            QMessageBox.warning(self, "Permission denied", "Viewers cannot delete assets.")
            return
        asset = self.get_selected_asset()
        if not asset: return
        confirm = QMessageBox.question(self, "Delete", f"Delete asset '{asset.title}'?", QMessageBox.Yes | QMessageBox.No)
        if confirm == QMessageBox.Yes:
            self.session.delete(asset)
            self.session.commit()
            self.perform_search()

    def open_metadata(self):
        asset = self.get_selected_asset()
        if not asset: return
        dlg = MetadataEditorDialog(self.session, asset, parent=self, role=self.user.role)
        dlg.exec_()
        # refresh table after possible metadata edits
        self.perform_search()

    def manage_users(self):
        if self.user.role != "Admin":
            QMessageBox.warning(self, "Permission denied", "Only Admins can manage users.")
            return
        dlg = UserManagerDialog(self.session, parent=self)
        dlg.exec_()

class UserManagerDialog(QDialog):
    def __init__(self, session, parent=None):
        super().__init__(parent)
        self.session = session
        self.setWindowTitle("User Manager")
        self.resize(400, 300)
        layout = QVBoxLayout()
        self.user_list = QListWidget()
        layout.addWidget(self.user_list)
        btns = QHBoxLayout()
        self.add_btn = QPushButton("Add User")
        self.del_btn = QPushButton("Delete User")
        self.close_btn = QPushButton("Close")
        btns.addWidget(self.add_btn)
        btns.addWidget(self.del_btn)
        btns.addWidget(self.close_btn)
        layout.addLayout(btns)
        self.setLayout(layout)
        self.add_btn.clicked.connect(self.add_user)
        self.del_btn.clicked.connect(self.del_user)
        self.close_btn.clicked.connect(self.accept)
        self.refresh()

    def refresh(self):
        self.user_list.clear()
        for u in self.session.query(User).order_by(User.username).all():
            self.user_list.addItem(f"{u.username} ({u.role})")

    def add_user(self):
        username, ok = QInputDialog.getText(self, "New user", "Username:")
        if not ok or not username.strip():
            return
        pw, ok = QInputDialog.getText(self, "New user", "Password:", QLineEdit.Password)
        if not ok or not pw:
            return
        role, ok = QInputDialog.getItem(self, "Role", "Select role:", ["Admin", "Editor", "Viewer"], 2, False)
        if not ok:
            return
        if self.session.query(User).filter_by(username=username).first():
            QMessageBox.warning(self, "Exists", "Username already exists")
            return
        user = User(username=username.strip(), pw_hash=hash_password(pw), role=role)
        self.session.add(user)
        self.session.commit()
        self.refresh()

    def del_user(self):
        it = self.user_list.currentItem()
        if not it:
            return
        username = it.text().split(" ")[0]
        u = self.session.query(User).filter_by(username=username).first()
        if u:
            confirm = QMessageBox.question(self, "Delete user", f"Delete user {u.username}?", QMessageBox.Yes | QMessageBox.No)
            if confirm == QMessageBox.Yes:
                self.session.delete(u)
                self.session.commit()
                self.refresh()

# ---------- Program entry ----------
def main():
    app = QApplication(sys.argv)
    login = LoginDialog()
    if login.exec_() != QDialog.Accepted:
        return
    user = login.user
    window = MainWindow(user)
    window.show()
    sys.exit(app.exec_())

if __name__ == "__main__":
    main()
