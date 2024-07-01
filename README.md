import sys
import os
from PyQt5.QtWidgets import QApplication, QWidget, QVBoxLayout, QPushButton, QLabel, QFileDialog, QProgressBar, QTextEdit
from PyQt5.QtGui import QPixmap
from PyQt5.QtCore import Qt, QThread, pyqtSignal
from google.cloud import vision
import io

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

        self.setLayout(self.layout)
        self.file_path = None

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
            result_str = '\n'.join([f"{key}: {value:.2f}" for key, value in result.items()])
            self.result_area.setPlainText(result_str)
        self.upload_button.setEnabled(True)

if __name__ == '__main__':
    app = QApplication(sys.argv)
    uploader = ImageUploader()
    uploader.show()
    sys.exit(app.exec_())
