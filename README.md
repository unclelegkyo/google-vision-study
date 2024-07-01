import sys
import os
import io
import pandas as pd
from PyQt5.QtWidgets import QApplication, QWidget, QVBoxLayout, QPushButton, QLabel, QFileDialog, QProgressBar, QTextEdit, QCheckBox, QTableWidget, QTableWidgetItem, QHeaderView
from PyQt5.QtGui import QPixmap
from PyQt5.QtCore import Qt, QThread, pyqtSignal
from google.cloud import vision

class UploadThread(QThread):
    progress = pyqtSignal(int)
    finished = pyqtSignal(dict)
    
    def __init__(self, file_path):
        super().__init__()
        self.file_path = file_path

    def run(self):
        self.progress.emit(10)
        try:
            client = vision.ImageAnnotatorClient()

            self.progress.emit(30)
            with io.open(self.file_path, 'rb') as image_file:
                content = image_file.read()

            image = vision.Image(content=content)

            self.progress.emit(70)
            response = client.label_detection(image=image)
            labels = response.label_annotations

            self.progress.emit(90)
            labels_dict = {label.description: label.score for label in labels}
            self.progress.emit(100)
            
            self.finished.emit(labels_dict)

        except Exception as e:
            self.finished.emit({'Error': str(e)})


class ImageUploader(QWidget):
    def __init__(self):
        super().__init__()
        self.initUI()

    def initUI(self):
        self.setWindowTitle('Image Uploader')
        self.setGeometry(100, 100, 400, 400)

        self.layout = QVBoxLayout()

        self.label = QLabel('No image selected')
        self.label.setAlignment(Qt.AlignCenter)
        self.layout.addWidget(self.label)

        self.browse_button = QPushButton('Browse')
        self.browse_button.clicked.connect(self.browse_image)
        self.layout.addWidget(self.browse_button)

        self.upload_button = QPushButton('Upload')
        self.upload_button.clicked.connect(self.upload_image)
        self.upload_button.setEnabled(False)
        self.layout.addWidget(self.upload_button)

        self.progress_bar = QProgressBar()
        self.layout.addWidget(self.progress_bar)

        self.result_area = QTextEdit()
        self.result_area.setReadOnly(True)
        self.layout.addWidget(self.result_area)

        self.show_dict_button = QPushButton('Show Dictionary')
        self.show_dict_button.clicked.connect(self.show_dictionary)
        self.show_dict_button.setEnabled(False)
        self.layout.addWidget(self.show_dict_button)

        self.setLayout(self.layout)
        self.file_path = None
        self.result_dict = {}

    def browse_image(self):
        options = QFileDialog.Options()
        options |= QFileDialog.ReadOnly
        file_path, _ = QFileDialog.getOpenFileName(self, "Select JPEG Image", "", "JPEG Files (*.jpeg *.jpg)", options=options)
        if file_path:
            file_size = os.path.getsize(file_path)
            if file_size > 5 * 1024 * 1024:
                self.label.setText('File is too large. Maximum size is 5MB.')
                self.upload_button.setEnabled(False)
            else:
                self.file_path = file_path
                pixmap = QPixmap(file_path)
                self.label.setPixmap(pixmap.scaled(300, 300, Qt.KeepAspectRatio))
                self.upload_button.setEnabled(True)

    def upload_image(self):
        if self.file_path:
            self.upload_button.setEnabled(False)
            self.result_area.clear()
            self.upload_thread = UploadThread(self.file_path)
            self.upload_thread.progress.connect(self.progress_bar.setValue)
            self.upload_thread.finished.connect(self.upload_finished)
            self.upload_thread.start()
    
    def upload_finished(self, result):
        if 'Error' in result:
            self.result_area.setPlainText(result['Error'])
        else:
            self.result_dict = result
            result_str = '\n'.join([f"{key}: {value:.2f}" for key, value in result.items()])
            self.result_area.setPlainText(result_str)
        self.upload_button.setEnabled(True)
        self.show_dict_button.setEnabled(True)

    def show_dictionary(self):
        self.dict_viewer = DictionaryViewer(self.result_dict)
        self.dict_viewer.show()

class DictionaryViewer(QWidget):
    def __init__(self, data_dict):
        super().__init__()
        self.data_dict = data_dict
        self.initUI()

    def initUI(self):
        self.setWindowTitle('Dictionary Viewer')
        self.setGeometry(100, 100, 600, 400)

        self.layout = QVBoxLayout()

        self.table_widget = QTableWidget()
        self.table_widget.setRowCount(len(self.data_dict))
        self.table_widget.setColumnCount(3)
        self.table_widget.setHorizontalHeaderLabels(['Select', 'Key', 'Value'])
        self.table_widget.horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)
        
        for row, (key, value) in enumerate(self.data_dict.items()):
            select_item = QTableWidgetItem()
            select_item.setCheckState(Qt.Unchecked)
            self.table_widget.setItem(row, 0, select_item)
            self.table_widget.setItem(row, 1, QTableWidgetItem(key))
            self.table_widget.setItem(row, 2, QTableWidgetItem(f"{value:.2f}"))
        
        self.layout.addWidget(self.table_widget)

        self.export_button = QPushButton('Export to Excel')
        self.export_button.clicked.connect(self.export_to_excel)
        self.layout.addWidget(self.export_button)

        self.setLayout(self.layout)

    def export_to_excel(self):
        selected_data = []
        for row in range(self.table_widget.rowCount()):
            if self.table_widget.item(row, 0).checkState() == Qt.Checked:
                key = self.table_widget.item(row, 1).text()
                value = self.table_widget.item(row, 2).text()
                selected_data.append((key, value))

        if selected_data:
            df = pd.DataFrame(selected_data, columns=['Key', 'Value'])
            file_path, _ = QFileDialog.getSaveFileName(self, "Save Excel File", "", "Excel Files (*.xlsx)")
            if file_path:
                df.to_excel(file_path, index=False)

if __name__ == '__main__':
    app = QApplication(sys.argv)
    uploader = ImageUploader()
    uploader.show()
    sys.exit(app.exec_())
