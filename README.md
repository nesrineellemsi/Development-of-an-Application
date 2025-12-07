# Development-of-an-Application
#Desktop application development (Python, PyQt5) for tester data management and archiving. Automated data collection and traceability using FileZilla and JSON
import xml.etree.ElementTree as ET
import os
from xml.dom import minidom

class FileZillaUserManager:
    def __init__(self, config_path=r"D:\FileZilla Server\FileZilla Server.xml"):  # ⬅️ Change le chemin si besoin
        self.config_path = config_path
        self.tree = ET.parse(config_path)
        self.root = self.tree.getroot()

    def add_user(self, username, password, folder_base="C:\\FTP\\Users\\"):
        # Vérifie si l'utilisateur existe déjà
        for user_elem in self.root.findall(".//User"):
            if user_elem.attrib["Name"] == username:
                return False  # Existe déjà

        # Crée le dossier utilisateur
        user_folder = os.path.join(folder_base, username)
        os.makedirs(user_folder, exist_ok=True)

        # Crée le nouvel utilisateur
        user_elem = ET.SubElement(self.root.find("Users"), "User", Name=username)

        # Mot de passe (en clair)
        pass_elem = ET.SubElement(user_elem, "Pass")
        pass_elem.text = password
        pass_elem.set("Type", "normal")

        # Paramètres
        settings = ET.SubElement(user_elem, "Settings")

        # Dossier partagé
        folder_setting = ET.SubElement(settings, "Setting", Name="Shared Folder")
        folder_setting.text = user_folder

        # Droits
        read_setting = ET.SubElement(settings, "Setting", Name="Read")
        read_setting.text = "1"
        write_setting = ET.SubElement(settings, "Setting", Name="Write")
        write_setting.text = "1"

        # Sauvegarde
        self.save()
        return True

    def save(self):
        xml_str = ET.tostring(self.root, encoding='unicode')
        dom = minidom.parseString(xml_str)
        pretty_xml = dom.toprettyxml(indent="  ")
        with open(self.config_path, "w", encoding="utf-8") as f:
            f.write(pretty_xml)




import os
import sys
import json
from PyQt5.QtWidgets import (
    QApplication, QDialog, QWidget, QLabel,
    QPushButton, QLineEdit, QListWidget,
    QVBoxLayout, QHBoxLayout, QMessageBox, QTextEdit
)
from PyQt5.QtCore import Qt
from ftplib import FTP
import datetime
from PyQt5.QtGui import QPixmap


# === Fenêtre principale ===
class MainWindow(QWidget):
    def __init__(self):

        super().__init__()
        self.setWindowTitle("Archiver - Sélection Testeur")
        self.resize(400, 300)
        self.testeurs = {}
        self.setup_ui()
        self.apply_stylesheet()
        self.setStyleSheet("QWidget#MainWindow, QWidget#TesteurFormWindow, QWidget#EditTesteurWindow, QWidget#ArchivageWindow { background-color: #f8f9fa; }")
    
    def apply_stylesheet(self):
      self.setStyleSheet("""
        QWidget {
            background-color: #f8f9fa;
            font-family: 'Segoe UI', Arial, sans-serif;
            font-size: 13px;
            color: #212529;
        }
        QLabel {
            color: #495057;
        }
        QLineEdit, QListWidget, QTextEdit {
            border: 1px solid #ced4da;
            border-radius: 6px;
            padding: 8px;
            background-color: #ffffff;
        }
        QListWidget::item:selected {
            background-color: #d0e1f9;
            color: #1a3b69;
            border-left: 3px solid #0056b3;
        }
        QPushButton {
            padding: 8px 12px;
            border: 1px solid #adb5bd;
            border-radius: 6px;
            background-color: #f0f0f0;
            color: #212529;
        }
        QPushButton:hover {
            background-color: #e9ecef;
        }
        QPushButton:pressed {
            background-color: #dcdfe2;
        }
        QPushButton#primary {
            background-color: #007BFF;
            color: white;
            border-color: #0069d9;
        }
        QPushButton#primary:hover {
            background-color: #0069d9;
        }
        QPushButton#success {
            background-color: #28a745;
            color: white;
            border-color: #1e7e34;
        }
        QPushButton#success:hover {
            background-color: #1e7e34;
        }
        QPushButton#danger {
            background-color: #dc3545;
            color: white;
            border-color: #c82333;
        }
        QPushButton#danger:hover {
            background-color: #c82333;
        }
        QTextEdit {
            background-color: #f1f3f5;
            font-family: 'Consolas', monospace;
            font-size: 12px;
        }
        QScrollBar:vertical {
            width: 12px;
            background: #f8f9fa;
            border-radius: 6px;
            border: 1px solid #e9ecef;
        }
        QScrollBar::handle:vertical {
            background: #adb5bd;
            border-radius: 6px;
            min-height: 30px;
            margin: 2px;
        }
        QScrollBar::handle:vertical:hover {
            background: #6c757d;
        }
    """)

    def setup_ui(self):
        layout = QVBoxLayout()
        title = QLabel("ARCHIVER")
        title.setProperty("title", True) 
        title.setStyleSheet("font-size: 20px; font-weight: bold; color: #2c3e50;")
        layout.addWidget(title)

 
        layout.addSpacing(20)  # Espace visuel

        self.testeur_list = QListWidget()
        self.testeur_list.itemDoubleClicked.connect(self.open_archivage_window)
        layout.addWidget(QLabel("Liste des testeurs :"))
        layout.addWidget(self.testeur_list)
        
        add_btn = QPushButton("➕ Ajouter un testeur")
        edit_btn = QPushButton("🔧 Modifier/Supprimer un testeur")
        add_btn.clicked.connect(self.open_add_testeur)
        edit_btn.clicked.connect(self.open_edit_testeur)
        add_btn.setObjectName("primary")
        edit_btn.setObjectName("success")   
        btn_layout = QHBoxLayout()
        btn_layout.addWidget(add_btn)
        btn_layout.addWidget(edit_btn)
        layout.addLayout(btn_layout)
        self.setLayout(layout)
        self.refresh_testeurs()
        self.testeur_list.itemDoubleClicked.connect(self.open_archivage_window)

        subtitle = QLabel("Sélectionnez un testeur pour commencer")
        subtitle.setProperty("subtitle", True) 
        subtitle.setStyleSheet("font-size: 13px; color: #7f8c8d;")
        layout.addWidget(title)
        layout.addWidget(subtitle)

    def open_add_testeur(self):
        self.form_window = TesteurFormWindow()
        self.form_window.show()
        self.close()

    def open_edit_testeur(self):
        self.edit_window = EditTesteurWindow()
        self.edit_window.show()
        self.close()

    def load_testeurs(self):
        if os.path.exists("testeurs.json"):
            with open("testeurs.json", "r") as f:
                self.testeurs = json.load(f)
        else:
            self.testeurs = {}


    def refresh_testeurs(self):
        self.testeur_list.clear()
        if os.path.exists("testeurs.json"):
            with open("testeurs.json", "r") as f:
                testeurs = json.load(f)

                
            for key in testeurs:
                data = testeurs[key]
                display_name = data.get("name", key)  # Affiche "name" si disponible
                self.testeur_list.addItem(display_name)

    def open_archivage_window(self, item):
            displayed_name = item.text().strip()
            found = False

            if os.path.exists("testeurs.json"):
                with open("testeurs.json", "r") as f:
                    testeurs = json.load(f)

                for key in testeurs:
                    data = testeurs[key]
                    if data.get("name") == displayed_name:
                        host = data["host"]
                        port = data["port"]
                        user = data["user"]
                        password = data["password"]

                        # ⬇️ Empêche l'ouverture de deux fenêtres
                        if hasattr(self, 'archivage_window') and self.archivage_window is not None:
                            try:
                                self.archivage_window.close()
                            except:
                                pass

                        self.archivage_window = ArchivageWindow(displayed_name, host, port, user, password)
                        self.archivage_window.show()
                        self.close()
                        found = True
                        break

            if not found:
                QMessageBox.warning(self, "Erreur", "❌ Testeur non trouvé")
    def user_exists_on_ftp(self, host, port, user, password):
        try:
            ftp = FTP()
            ftp.connect(host, int(port))
            ftp.login(user, password)
            ftp.quit()
            return True
        except Exception as e:
            return False

# === Fenêtre pour ajouter un testeur ===
class TesteurFormWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Ajouter un testeur")
        self.resize(400, 300)
        self.setup_ui()
        self.apply_stylesheet()
        self.setStyleSheet("QWidget#MainWindow, QWidget#TesteurFormWindow, QWidget#EditTesteurWindow, QWidget#ArchivageWindow { background-color: #f8f9fa; }")

    def apply_stylesheet(self):
        self.setStyleSheet("""
            QWidget {
                background-color: #f8f9fa;
                font-family: 'Segoe UI', Arial, sans-serif;
                font-size: 13px;
                color: #212529;
            }
            QLabel {
                color: #495057;
            }
            QLineEdit, QListWidget, QTextEdit {
                border: 1px solid #ced4da;
                border-radius: 6px;
                padding: 8px;
                background-color: #ffffff;
            }
            QListWidget::item:selected {
                background-color: #d0e1f9;
                color: #1a3b69;
                border-left: 3px solid #0056b3;
            }
            QPushButton {
                padding: 8px 12px;
                border: 1px solid #adb5bd;
                border-radius: 6px;
                background-color: #f0f0f0;
                color: #212529;
            }
            QPushButton:hover {
                background-color: #e9ecef;
            }
            QPushButton:pressed {
                background-color: #dcdfe2;
            }
            QPushButton#primary {
                background-color: #007BFF;
                color: white;
                border-color: #0069d9;
            }
            QPushButton#primary:hover {
                background-color: #0069d9;
            }
            QPushButton#success {
                background-color: #28a745;
                color: white;
                border-color: #1e7e34;
            }
            QPushButton#success:hover {
                background-color: #1e7e34;
            }
            QPushButton#danger {
                background-color: #dc3545;
                color: white;
                border-color: #c82333;
            }
            QPushButton#danger:hover {
                background-color: #c82333;
            }
            QTextEdit {
                background-color: #f1f3f5;
                font-family: 'Consolas', monospace;
                font-size: 12px;
            }
            QScrollBar:vertical {
                width: 12px;
                background: #f8f9fa;
                border-radius: 6px;
                border: 1px solid #e9ecef;
            }
            QScrollBar::handle:vertical {
                background: #adb5bd;
                border-radius: 6px;
                min-height: 30px;
                margin: 2px;
            }
            QScrollBar::handle:vertical:hover {
                background: #6c757d;
            }
        """)
    def setup_ui(self):
        layout = QVBoxLayout()
        self.apply_stylesheet() 
        self.name_input = QLineEdit()
        self.host_input = QLineEdit()
        self.port_input = QLineEdit("21")
        self.user_input = QLineEdit()
        self.pass_input = QLineEdit()
        self.pass_input.setEchoMode(QLineEdit.Password)

        layout.addWidget(QLabel("Name:"))
        layout.addWidget(self.name_input)
        layout.addWidget(QLabel("Serveur (Host) :"))
        layout.addWidget(self.host_input)
        layout.addWidget(QLabel("Port :"))
        layout.addWidget(self.port_input)
        layout.addWidget(QLabel("Utilisateur :"))
        layout.addWidget(self.user_input)
        layout.addWidget(QLabel("Mot de passe :"))
        layout.addWidget(self.pass_input)

        save_btn = QPushButton("💾Sauvegarder et continuer")
        save_btn.clicked.connect(self.save_testeur)
        layout.addWidget(save_btn)
        #save_btn.setStyleSheet("background-color: #28a745;")
       # back_btn.setStyleSheet("background-color: #6c757d;")
        save_btn.setProperty("success", True)
        back_btn = QPushButton("⬅️ Retour")
        back_btn.clicked.connect(self.go_back)
        layout.addWidget(back_btn)

        self.setLayout(layout)

    def save_testeur(self):
        name = self.name_input.text().strip()
        host = self.host_input.text().strip()
        port = self.port_input.text().strip()
        user = self.user_input.text().strip()
        password = self.pass_input.text().strip()

        if not all([name,host, port, user, password]):
            QMessageBox.warning(self, "Erreur", "Tous les champs doivent être remplis.")
            return

        try:
            port_int = int(port)
        except ValueError:
            QMessageBox.warning(self, "Erreur", "Le port doit être un nombre valide.")
            return

  # Vérifie la connexion avant d'ajouter
        try:
            ftp = FTP()
            ftp.connect(host, port_int)
            ftp.login(user, password)
            ftp.quit()
        except Exception as e:
            QMessageBox.critical(self, "Erreur", f"❌ Connexion impossible au serveur.\n{str(e)}")
            return
        
        testeurs = {}
        if os.path.exists("testeurs.json"):
                with open("testeurs.json", "r") as f:
                    testeurs = json.load(f)
        data = {
            "name": name,
            "host": host,
            "port": port_int,
            "user": user,
            "password": password
        }

        new_name = name
        testeurs[name] = data  # ✅ Maintenant, testeurs est bien définie
        with open("testeurs.json", "w") as f:
            json.dump(testeurs, f, indent=4)

        QMessageBox.information(self, "Succès", "Testeur ajouté avec succès.")

        self.parent_window = MainWindow()
        self.parent_window.show()
        self.close()

        # ✅ Ajoute l'utilisateur dans FileZilla Server
        try:
            filezilla = FileZillaUserManager()
            if filezilla.add_user(user, password):
                QMessageBox.information(self, "Succès", "✅ Utilisateur ajouté dans FileZilla Server")
            else:
                QMessageBox.warning(self, "Info", "⚠ Utilisateur déjà existant dans FileZilla")
        except Exception as e:
            QMessageBox.critical(self, "Erreur", f"❌ Impossible d'ajouter dans FileZilla Server :\n{str(e)}")

    def go_back(self):
        self.back_window = MainWindow()
        self.back_window.show()
        self.close()


# === Fenêtre pour modifier/supprimer un testeur ===
class EditTesteurWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("EDITER un testeur")
        self.resize(500, 400)
        self.testeurs = self.load_testeurs()
        self.selected_name = None
        self.setup_ui()
        self.apply_stylesheet()
        self.setStyleSheet("QWidget#MainWindow, QWidget#TesteurFormWindow, QWidget#EditTesteurWindow, QWidget#ArchivageWindow { background-color: #f8f9fa; }")
      
    def apply_stylesheet(self):
        self.setStyleSheet("""
            QWidget {
                background-color: #f8f9fa;
                font-family: 'Segoe UI', Arial, sans-serif;
                font-size: 13px;
                color: #212529;
            }
            QLabel {
                color: #495057;
            }
            QLineEdit, QListWidget, QTextEdit {
                border: 1px solid #ced4da;
                border-radius: 6px;
                padding: 8px;
                background-color: #ffffff;
            }
            QListWidget::item:selected {
                background-color: #d0e1f9;
                color: #1a3b69;
                border-left: 3px solid #0056b3;
            }
            QPushButton {
                padding: 8px 12px;
                border: 1px solid #adb5bd;
                border-radius: 6px;
                background-color: #f0f0f0;
                color: #212529;
            }
            QPushButton:hover {
                background-color: #e9ecef;
            }
            QPushButton:pressed {
                background-color: #dcdfe2;
            }
            QPushButton#primary {
                background-color: #007BFF;
                color: white;
                border-color: #0069d9;
            }
            QPushButton#primary:hover {
                background-color: #0069d9;
            }
            QPushButton#success {
                background-color: #28a745;
                color: white;
                border-color: #1e7e34;
            }
            QPushButton#success:hover {
                background-color: #1e7e34;
            }
            QPushButton#danger {
                background-color: #dc3545;
                color: white;
                border-color: #c82333;
            }
            QPushButton#danger:hover {
                background-color: #c82333;
            }
            QTextEdit {
                background-color: #f1f3f5;
                font-family: 'Consolas', monospace;
                font-size: 12px;
            }
            QScrollBar:vertical {
                width: 12px;
                background: #f8f9fa;
                border-radius: 6px;
                border: 1px solid #e9ecef;
            }
            QScrollBar::handle:vertical {
                background: #adb5bd;
                border-radius: 6px;
                min-height: 30px;
                margin: 2px;
            }
            QScrollBar::handle:vertical:hover {
                background: #6c757d;
            }
        """)

    def load_testeurs(self):
        if os.path.exists("testeurs.json"):
            with open("testeurs.json", "r") as f:
                return json.load(f)
        return {}

    def setup_ui(self):
        layout = QVBoxLayout()
        self.apply_stylesheet()
        self.testeur_list = QListWidget()
        self.testeur_list.itemClicked.connect(self.show_details)

        self.name_input = QLineEdit()
        self.host_input = QLineEdit()
        self.port_input = QLineEdit()
        self.user_input = QLineEdit()
        self.pass_input = QLineEdit()
        #self.pass_input = QLineEdit()

        edit_btn = QPushButton("✏️ Modifier")
        delete_btn = QPushButton("🗑️ Supprimer")
        back_btn = QPushButton("⬅️ Retour")
        edit_btn.setObjectName("success")
        delete_btn.setObjectName("danger")

        edit_btn.clicked.connect(self.modify_testeur)
        delete_btn.clicked.connect(self.delete_testeur)
        back_btn.clicked.connect(self.go_back)

        layout.addWidget(QLabel("Liste des testeurs :"))
        layout.addWidget(self.testeur_list)

        layout.addWidget(QLabel("Nom :"))
        layout.addWidget(self.name_input)
        layout.addWidget(QLabel("Host :"))
        layout.addWidget(self.host_input)
        layout.addWidget(QLabel("Port :"))
        layout.addWidget(self.port_input)
        layout.addWidget(QLabel("Utilisateur :"))
        layout.addWidget(self.user_input)
        layout.addWidget(QLabel("Mot de passe :"))
        layout.addWidget(self.pass_input)

        btn_layout = QHBoxLayout()
        btn_layout.addWidget(edit_btn)
        btn_layout.addWidget(delete_btn)

        layout.addLayout(btn_layout)
        layout.addWidget(back_btn)

        self.setLayout(layout)
        self.refresh_list()
        self.apply_stylesheet()     

    def show_details(self, item):
        displayed_name = item.text().strip()  # C’est le "name" affiché dans la liste
        # On cherche dans le dictionnaire la clé qui correspond à ce nom
        for key in self.testeurs:
            data = self.testeurs[key]
            if data.get("name") == displayed_name or key == displayed_name:
                self.selected_name = key
                self.name_input.setText(data.get("name", ""))
                self.host_input.setText(data.get("host", ""))
                self.port_input.setText(str(data.get("port", "")))
                self.user_input.setText(data.get("user", ""))
                self.pass_input.setText(data.get("password", ""))
                return
        QMessageBox.warning(self, "Erreur", "⚠ Testeur non trouvé")

    def modify_testeur(self):
        if not self.selected_name:
            QMessageBox.warning(self, "Erreur", "Veuillez sélectionner un testeur.")
            return

        new_name = self.name_input.text().strip()
        host = self.host_input.text().strip()
        port = self.port_input.text().strip()
        user = self.user_input.text().strip()
        password = self.pass_input.text().strip()

        try:
            port_int = int(port)
        except ValueError:
            QMessageBox.warning(self, "Erreur", "Le port doit être un nombre valide.")
            return

            # ⬇️ Teste la connexion avec les nouveaux paramètres
        try:
            ftp = FTP()
            ftp.connect(host, port_int)
            ftp.login(user, password)
            ftp.quit()
        except Exception as e:
            QMessageBox.critical(self, "Erreur", f"❌ Identifiants invalides pour ce testeur.\n{str(e)}")
            return

        # ⬇️ Si la connexion est réussie, on continue la modification
        old_data = self.testeurs.get(self.selected_name)

        if old_data:
            old_data["name"] = new_name
            old_data["host"] = host
            old_data["port"] = port_int
            old_data["user"] = user
            old_data["password"] = password

        # ⬇️ Optionnel : Changer la clé du dictionnaire si besoin
        if new_name != self.selected_name:
            
            del self.testeurs[self.selected_name]
            self.selected_name = new_name  # Met à jour selected_name pour éviter les bugs

        with open("testeurs.json", "w") as f:
            json.dump(self.testeurs, f, indent=4)

        self.refresh_list()
        self.clear_inputs()
        QMessageBox.information(self, "Succès", "✅ Testeur modifié avec succès.") 
    
    def save_testeurs(self):
        with open("testeurs.json", "w") as f:
            json.dump(self.testeurs, f, indent=4)

    def delete_testeur(self):
        if not self.selected_name:
            QMessageBox.warning(self, "Erreur", "Veuillez sélectionner un testeur.")
            return

        reply = QMessageBox.question(
            self,
            "Supprimer",
            f"Êtes-vous sûr de vouloir supprimer {self.selected_name} ?",
            QMessageBox.Yes | QMessageBox.No,
            QMessageBox.No
        )

        if reply == QMessageBox.Yes:
            if self.selected_name in self.testeurs:
                del self.testeurs[self.selected_name]
                with open("testeurs.json", "w") as f:
                    json.dump(self.testeurs, f, indent=4)
                self.selected_name = None
                self.refresh_list()
                self.clear_inputs()
                QMessageBox.information(self, "Succès", "🗑️ Testeur supprimé.")
            else:
                QMessageBox.critical(self, "Erreur", "❌ Testeur introuvable dans les données.")
    
    def clear_inputs(self):
        self.name_input.clear()
        self.host_input.clear()
        self.port_input.clear()
        self.user_input.clear()
        self.pass_input.clear()

    def refresh_list(self):
        self.testeur_list.clear()
        for key in self.testeurs:
            data = self.testeurs[key]
            display_name = data.get("name", key)
            self.testeur_list.addItem(display_name)
    
    def go_back(self):
        with open("testeurs.json", "w") as f:
         json.dump(self.testeurs, f, indent=4)
        self.parent_window = MainWindow()
        self.parent_window.show()
        self.close()
        
class ArchivageWindow(QWidget):  # QMainWindow aussi possible si tu veux une barre de menus
    def __init__(self, testeur_name, host, port, user, password):
        super().__init__()
        self.testeur_name = testeur_name
        self.host = host
        self.port = port
        self.user = user
        self.password = password
        self.ftp = None
        self.current_path = "/"  # Chemin courant sur le serveur
        self.setWindowTitle(f"Archivage - {self.testeur_name}")
        self.resize(800, 600)
        self.setup_ui() 
        self.apply_stylesheet()
        self.setStyleSheet("QWidget#MainWindow, QWidget#TesteurFormWindow, QWidget#EditTesteurWindow, QWidget#ArchivageWindow { background-color: #f8f9fa; }")
    def apply_stylesheet(self):
        self.setStyleSheet("""
           QWidget {
            background-color: #f8f9fa;
            font-family: 'Segoe UI', 'Helvetica', Arial, sans-serif;
            font-size: 13px;
            color: #2c3e50;
        }
        
        QLabel {
            color: #34495e;
            font-weight: normal;
            padding: 2px;
        }
        
        QLabel[bold="true"], QLabel:header {
            font-weight: 600;
            color: #2c3e50;
            font-size: 15px;
            padding: 4px 0px;
        }
        
        QLineEdit, QListWidget, QTextEdit {
            border: 2px solid #e9ecef;
            border-radius: 6px;
            padding: 8px;
            background-color: #ffffff;
            color: #2c3e50;
            selection-background-color: #dae8fc;
        }
        
        QLineEdit:focus, QListWidget:focus, QTextEdit:focus {
            border-color: #7fb3d3;
            background-color: #ffffff;
            outline: none;
        }
        
        QListWidget {
            border: 2px solid #e9ecef;
            background-color: #ffffff;
            alternate-background-color: #f8f9fa;
            gridline-color: #e9ecef;
        }
        
        QListWidget::item {
            padding: 8px 12px;
            border-bottom: 1px solid #f1f3f4;
        }
        
        QListWidget::item:hover {
            background-color: #f1f3f4;
        }
        
        QListWidget::item:selected {
            background-color: #e3f2fd;
            color: #1565c0;
            border-left: 3px solid #42a5f5;
        }
        
        QPushButton {
            background-color: #6c757d;
            color: white;
            border: none;
            border-radius: 6px;
            padding: 10px 16px;
            font-size: 13px;
            font-weight: 500;
            min-height: 32px;
        }
        
        QPushButton:hover {
            background-color: #5a6268;
            transform: translateY(-1px);
        }
        
        QPushButton:pressed {
            background-color: #545b62;
            transform: translateY(0px);
        }
        
        QPushButton:disabled {
            background-color: #e9ecef;
            color: #6c757d;
        }
        
        /* Primary buttons */
        QPushButton[primary="true"] {
            background-color: #495057;
            color: white;
        }
        
        QPushButton[primary="true"]:hover {
            background-color: #343a40;
        }
        
        /* Success buttons */
        QPushButton[success="true"] {
            background-color: #28a745;
            color: white;
        }
        
        QPushButton[success="true"]:hover {
            background-color: #218838;
        }
        
        /* Warning buttons */
        QPushButton[warning="true"] {
            background-color: #fd7e14;
            color: white;
        }
        
        QPushButton[warning="true"]:hover {
            background-color: #e8630a;
        }
        
        /* Danger buttons */
        QPushButton[danger="true"] {
            background-color: #dc3545;
            color: white;
        }
        
        QPushButton[danger="true"]:hover {
            background-color: #c82333;
        }
        
        QMessageBox {
            background-color: #ffffff;
            color: #2c3e50;
            border: 1px solid #dee2e6;
        }
        
        QMessageBox QLabel {
            color: #495057;
            padding: 10px;
        }
        
        QMessageBox QPushButton {
            background-color: #6c757d;
            color: white;
            padding: 8px 16px;
            min-width: 80px;
            margin: 4px;
        }
        
        QMessageBox QPushButton:hover {
            background-color: #5a6268;
        }
        
        QScrollBar:vertical {
            width: 12px;
            background: #f8f9fa;
            border-radius: 6px;
            border: 1px solid #e9ecef;
        }
        
        QScrollBar::handle:vertical {
            background: #adb5bd;
            border-radius: 6px;
            min-height: 30px;
            margin: 2px;
        }
        
        QScrollBar::handle:vertical:hover {
            background: #6c757d;
        }
        
        QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical {
            height: 0px;
        }
        
        /* Title styling */
        QLabel[title="true"] {
            font-size: 24px;
            font-weight: 300;
            color: #2c3e50;
            padding: 10px 0px;
        }
        
        QLabel[subtitle="true"] {
            font-size: 14px;
            color: #6c757d;
            padding: 5px 0px 15px 0px;
        }
    """)

    def log(self, message):
        self.log_box.append(message)
        self.log_box.ensureCursorVisible()

    def setup_ui(self):
        layout = QVBoxLayout()

          # === Bouton Retour à MainWindow ===
        main_back_btn = QPushButton("🏠 Retour à la liste")
       # main_back_btn.setStyleSheet("background-color: #ffc107; color: black;")
        main_back_btn.setObjectName("warning")

        
        main_back_btn.setFixedHeight(30)
        layout.addWidget(main_back_btn)
        main_back_btn.clicked.connect(self.go_back_to_main)
        

    # ⬇️ Affichage du chemin d'archive
        self.path_label = QLabel(f"📍 Chemin actuel : {self.current_path}")
        self.archive_path_label = QLabel()
        #self.update_archive_path_label()  # Met à jour le texte du label
        self.archive_path_label.setWordWrap(True)
        self.archive_path_label.setStyleSheet("""
            font-size: 13px; 
            background-color: #e9ecef; 
            padding: 12px; 
            border-radius: 6px;
            border: 1px solid #dee2e6;
            color: #495057;
        """)
        layout.addWidget(QLabel("Dossier d'archive :"))
        layout.addWidget(self.archive_path_label)


         # === Bouton pour naviguer dans les dossiers ===
        archive_btn = QPushButton("📥 Archiver les fichiers sélectionnés")
        archive_btn.setObjectName("success")
        archive_btn.setFixedHeight(30)
        archive_btn.clicked.connect(self.archive_selected_files)
        layout.addWidget(archive_btn)

        # Liste des fichiers
        self.file_list = QListWidget()
        self.file_list.setStyleSheet("""
            ListWidget::item:selected {
            background-color: #e3f2fd;
            color: #1565c0;
            border-left: 3px solid #42a5f5;
        }
    """)
        layout.addWidget(QLabel("Fichiers disponibles :"))
        layout.addWidget(self.file_list)

         # === Bouton pour revenir au dossier parent ===
        self.return_btn = QPushButton("📁 dossier précédent")
        self.return_btn.clicked.connect(self.go_back_from_folder)
        self.return_btn.hide()  # Caché par défaut
        layout.addWidget(self.return_btn)

        # Journal
        self.log_box = QTextEdit()
        self.log_box.setReadOnly(True)
        self.log_box.setStyleSheet("""
            border: 2px solid #e9ecef; 
            background-color: #ffffff;
            border-radius: 6px;
            padding: 8px;
        """)
        layout.addWidget(QLabel("📝Journal :"))
        layout.addWidget(self.log_box)

        # === Données du testeur ===
        data_group = QVBoxLayout()
        data_group.addWidget(QLabel(f"📁 Testeur : {self.testeur_name}"))
        data_group.addWidget(QLabel(f"🌐 Host : {self.host}"))
        data_group.addWidget(QLabel(f"🔢 Port : {self.port}"))
        data_group.addWidget(QLabel(f"👤 Utilisateur : {self.user}"))
        layout.addLayout(data_group)
        self.file_list.itemDoubleClicked.connect(self.handle_item_double_click)


        self.setLayout(layout)
        self.try_connect()

    def handle_item_double_click(self, item):
        
        if not self.ftp:
          self.log("⚠ Vous devez être connecté pour accéder aux dossiers")
          return
        line = item.text().strip()
        parts = line.split()
        if len(parts) < 9:
            self.log("⚠ Ligne invalide")
            return

        file_permissions = parts[0]
        filename = ' '.join(parts[8:])

        if file_permissions.startswith('d'):
            self.enter_ftp_folder(filename)
            self.return_btn.show()
        else:
            self.preview_file()

    def enter_ftp_folder(self, folder_name):
        try:
            self.ftp.cwd(folder_name)
            files = []
            self.ftp.retrlines('LIST', files.append)
            self.file_list.clear()
            self.file_list.addItems(files)  

    # ⬇️ Met à jour le chemin actuel
            if self.current_path == "/":
                self.current_path = f"/{folder_name}"
            else:
                self.current_path += f"/{folder_name}"

            self.path_label.setText(f"📍 Chemin actuel : {self.current_path}")

    # ❗ Remplace le bouton retour précédent s'il existe déjà
            if hasattr(self, 'top_layout'):
                self.top_layout.insertWidget(0, self.return_btn)
            else:
                top_layout = QHBoxLayout()
                top_layout.addWidget(self.return_btn)
                top_layout.addStretch()
                main_layout = self.layout()
                main_layout.insertLayout(0, top_layout)
        except Exception as e:
            self.log(f"❌ Impossible d'ouvrir '{folder_name}' : {str(e)}")

            self.log(f"📂 Vous êtes dans le dossier : {folder_name}")
        except Exception as e:
            self.log(f"❌ Impossible d'ouvrir le dossier '{folder_name}' : {str(e)}")

    def go_back_from_folder(self):
        try:
            self.ftp.cwd("..")
            files = []
            self.ftp.retrlines('LIST', files.append)
            self.file_list.clear()
            self.file_list.addItems(files)

            # ⬇️ Mets à jour le chemin
            parts = self.current_path.split("/")
            if len(parts) > 2:
                self.current_path = "/".join(parts[:-1])
            else:
                self.current_path = "/"

            self.path_label.setText(f"📍 Chemin actuel : {self.current_path}")
            self.log("⬅️ Retour au dossier parent ")

        except Exception as e:
            self.log(f"❌ Erreur lors du retour : {str(e)}")

    def update_archive_path_label(self):
        now = datetime.datetime.now()
        year = now.strftime("%Y")
        month = now.strftime("%m_%B")
        day = now.strftime("%d")
        hour = now.strftime("%Hh%M")

        base_path = os.path.join("archives", self.testeur_name, year, month, day, hour)
        self.archive_path_label.setText(base_path)
        os.makedirs(base_path, exist_ok=True)


    def try_connect(self):
        try:
            self.ftp = FTP()
            self.ftp.connect(self.host, self.port)
            response =self.ftp.login(self.user, self.password)
              
            if "230" not in response:  # Code 230 = Login successful
                self.log("❌ Authentification échouée : mauvais utilisateur/mot de passe")
                QMessageBox.critical(self, "Erreur", "🚫 Échec de connexion : mauvais identifiants")
                self.ftp = None
                return
        
            files = []
            self.ftp.retrlines('LIST', files.append)
            self.file_list.addItems(files)
            self.log("✅ Connexion réussie !")
        except Exception as e:
            self.log(f"❌ Échec de connexion : {str(e)}")
            QMessageBox.critical(self, "Erreur", f"Impossible de se connecter au serveur FTP.\n{str(e)}")
            self.ftp = None

    def archive_selected_files(self):
        selected_items = self.file_list.selectedItems()
        if not selected_items:
            self.log("⚠ Aucun fichier sélectionné")
            return

        now = datetime.datetime.now()
        year = now.strftime("%Y")
        month = now.strftime("%m_%B")
        day = now.strftime("%d")
        hour = now.strftime("%Hh%M")

        base_path = os.path.join("archives", self.testeur_name, year, month, day, hour)
        self.archive_path_label.setText(base_path)
        os.makedirs(base_path, exist_ok=True)

        for item in selected_items:
            line = item.text().strip()
            parts = line.split()
            if len(parts) < 9:
                continue
            filename = ' '.join(parts[8:])
            local_path = os.path.join(base_path, filename)

        if self.is_already_archived(filename):
            if os.path.exists(local_path):
                # ✅ Le fichier est déjà présent → on le retire de la liste
                self.log(f"📁 Fichier déjà archivé : {filename}")
                self.file_list.takeItem(self.file_list.row(item))
            else:
                # ❌ Le fichier était marqué comme archivé mais n’existe plus → on le télécharge à nouveau
                self.log(f"🔄 {filename}→ téléchargement lancé")
                try:
                    with open(local_path, 'wb') as f:
                        self.ftp.retrbinary(f'RETR {filename}', f.write)
                    self.log(f"📥 Téléchargement terminé : {filename}")
                    self.file_list.takeItem(self.file_list.row(item))
                    self.mark_as_archived(filename)
                except Exception as e:
                    self.log(f"❌ Échec sur {filename} : {str(e)}")
        else:
            # ⬇️ Nouveau fichier → on le télécharge et on le marque comme archivé
            try:
                with open(local_path, 'wb') as f:
                    self.ftp.retrbinary(f'RETR {filename}', f.write)
                self.log(f"📥 Téléchargé : {filename}")
                self.file_list.takeItem(self.file_list.row(item))
                self.mark_as_archived(filename)
            except Exception as e:
                self.log(f"❌ Échec sur {filename} : {str(e)}")

        self.log("✅ Archivage terminé.")
        
    def is_already_archived(self, filename):
        if os.path.exists("archived_files.txt"):
            with open("archived_files.txt", "r") as f:
                archived = f.read().splitlines()
            return filename in archived
        return False
    
    def mark_as_archived(self, filename):
        with open("archived_files.txt", "a") as f:
            f.write(filename + "\n")

    def clean_archived_log(self):
        if not os.path.exists("archived_files.txt"):
            return

        with open("archived_files.txt", "r") as f:
            archived = f.read().splitlines()

        cleaned = []
        for filename in archived:
            found = False
            for root, _, files in os.walk("archives"):
                if filename in files:
                    found = True
                    break
            if found:
                cleaned.append(filename)

        with open("archived_files.txt", "w") as f:
            f.write("\n".join(cleaned))

    def preview_file(self):
        selected_items = self.file_list.selectedItems()
        if not selected_items:
            self.log("⚠ Aucun fichier sélectionné.")
            return

        for item in selected_items:
            line = item.text().strip()
            parts = line.split()
            if len(parts) < 9:
                self.log(f"⚠ Format invalide : {line}")
                continue

            file_permissions = parts[0]
            filename = ' '.join(parts[8:])
            
            # Ignore les dossiers
            if file_permissions.startswith('d'):
                self.log(f"📁 Ignoré : {filename} (c'est un dossier)")
                continue

            try:
                # Téléchargement temporaire pour aperçu
                temp_path = f".preview_{filename}"
                with open(temp_path, 'wb') as f:
                    self.ftp.retrbinary(f'RETR {filename}', f.write)

                # Affiche l'aperçu selon le type de fichier
                self.show_preview_dialog(temp_path, filename)

                os.remove(temp_path)  # Nettoie après aperçu

            except Exception as e:
                self.log(f"❌ Impossible d'afficher l'aperçu de {filename} : {str(e)}")

    def download_ftp_folder(self, remote_dir, local_dir):
        try:
            os.makedirs(local_dir, exist_ok=True)
            self.ftp.cwd(remote_dir)

            files = []
            self.ftp.retrlines('LIST', files.append)

            for line in files:
                parts = line.split()
                if len(parts) < 9:
                    continue
                fname = ' '.join(parts[8:])
                if fname in ('.', '..'):
                    continue

                if parts[0].startswith('d'):  # C’est un dossier
                    self.download_ftp_folder(fname, os.path.join(local_dir, fname))
                else:  # C’est un fichier
                    with open(os.path.join(local_dir, fname), 'wb') as f:
                        self.ftp.retrbinary(f'RETR {fname}', f.write)
            self.ftp.cwd("..")
        except Exception as e:
            self.log(f"❌ Impossible de télécharger '{remote_dir}' : {str(e)}")

    def is_ftp_directory(self, name):
        """Vérifie si un élément est un dossier FTP"""
        try:
            self.ftp.cwd(name)
            self.ftp.cwd("..")
            return True
        except:
            return False

    def delete_ftp_folder(self, folder_name):
        try:
            self.ftp.cwd(folder_name)
            files = []
            self.ftp.retrlines('LIST', files.append)
            for line in files:
                parts = line.split()
                if len(parts) >= 9:
                    fname = ' '.join(parts[8:])
                    if self.is_ftp_directory(fname):
                        self.delete_ftp_folder(fname)
                    else:
                        self.ftp.delete(fname)
            self.ftp.cwd("..")
            self.ftp.rmd(folder_name)  # Supprime le dossier
        except Exception as e:
            self.log(f"❌ Impossible de supprimer '{folder_name}' : {str(e)}")

    def is_already_archived(self, filename):
        if os.path.exists("archived_files.txt"):
            with open("archived_files.txt", "r") as f:
                archived = f.read().splitlines()
            return filename in archived

    def mark_as_archived(self, filename):
        with open("archived_files.txt", "a") as f:
            f.write(filename + "\n")

    def show_preview_dialog(self, file_path, filename):
        ext = os.path.splitext(filename)[1].lower()

        dialog = QDialog(self)
        dialog.setWindowTitle(f"Aperçu - {filename}")
        layout = QVBoxLayout(dialog)

        if ext in ('.txt', '.csv', '.log', '.json', '.xml', '.py'):
            text_edit = QTextEdit()
            text_edit.setReadOnly(True)
            try:
                with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                    content = f.read(2000)
                    text_edit.setText(content)
            except Exception:
                text_edit.setText("Impossible d'afficher ce fichier")
            layout.addWidget(text_edit)

        elif ext in ('.png', '.jpg', '.jpeg', '.gif', '.bmp'):
            label = QLabel()
            pixmap = QPixmap(file_path)
            if pixmap.isNull():
                label.setText("Impossible de charger l'image.")
            else:
                label.setPixmap(pixmap.scaledToWidth(400))
            layout.addWidget(label)

        else:
            layout.addWidget(QLabel("Aperçu non disponible pour ce type de fichier."))

        dialog.setLayout(layout)
        dialog.exec_()

    def go_back_to_main(self):
        """Revenir à MainWindow sans fermer la connexion FTP"""
        reply = QMessageBox.question(
            self,
            "Retour",
            "Voulez-vous vraiment revenir à la liste des testeurs ?",
            QMessageBox.Yes | QMessageBox.No,
            QMessageBox.No
        )

        if reply == QMessageBox.Yes:
            if self.ftp:
                try:
                    self.ftp.quit()
                except:
                    pass
            self.parent_window = MainWindow()
            self.parent_window.show()
            self.close()

    def go_back_from_folder(self):
        """Revenir au dossier parent sur le serveur FTP"""
        try:
            self.ftp.cwd("..")
            files = []
            self.ftp.retrlines('LIST', files.append)
            self.file_list.clear()
            self.file_list.addItems(files)
            self.log("⬅️ Retour au dossier parent")
        except Exception as e:
            self.log(f"❌ Erreur lors du retour : {str(e)}")

    def go_back(self):
        self.parent_window = MainWindow()
        self.parent_window.show()
        self.close()      

if __name__ == "__main__":
    app = QApplication(sys.argv)
    
    # === Lance la première fenêtre ===
    main_window = MainWindow()
    main_window.show()
    
    sys.exit(app.exec_())
