# Interfaz-grafica-de-un-estacionamiento
Interfaz grafica de un control de entrada de un esatcionamiento, basado en un sensor inductivo, un servomotor y una pantallla LCD
import sys
import serial
from PyQt5 import QtWidgets, QtCore, QtGui
from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
from matplotlib.figure import Figure
import matplotlib.pyplot as plt
from collections import deque
import time
import threading
import serial.tools.list_ports

# Configuración de estilos para Matplotlib
plt.style.use('ggplot')
plt.rcParams.update({
    'axes.facecolor': '#2A2E33',
    'axes.edgecolor': '#FFDE59',
    'axes.labelcolor': '#FFFFFF',
    'axes.grid': True,
    'grid.color': '#3D434A',
    'grid.linestyle': '--',
    'grid.alpha': 0.7,
    'xtick.color': '#FFDE59',
    'ytick.color': '#FFDE59',
    'text.color': '#FFFFFF',
    'font.family': 'Arial',
    'figure.facecolor': '#1D2024'
})

class Parkea2GUIMinimalist(QtWidgets.QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("PARKΞA2")
        self.resize(1200, 800)
        
        # Paleta de colores actualizada
        self.bg_color = QtGui.QColor("#1D2024")  # Fondo gris oscuro
        self.text_color = QtGui.QColor("#FFFFFF")  # Texto blanco
        self.accent_color = QtGui.QColor("#FFDE59")  # Amarillo como acento
        self.secondary_color = QtGui.QColor("#3D434A")  # Gris medio para detalles
        
        # Configurar paleta
        palette = self.palette()
        palette.setColor(QtGui.QPalette.Window, self.bg_color)
        palette.setColor(QtGui.QPalette.WindowText, self.text_color)
        self.setPalette(palette)
        
        # Widget central
        central_widget = QtWidgets.QWidget()
        self.setCentralWidget(central_widget)
        main_layout = QtWidgets.QVBoxLayout(central_widget)
        main_layout.setSpacing(20)
        main_layout.setContentsMargins(20, 20, 20, 20)
        
        # --- HEADER REDISEÑADO ---
        header = QtWidgets.QWidget()
        header_layout = QtWidgets.QHBoxLayout(header)
        header_layout.setContentsMargins(0, 0, 0, 0)
        
        # Logo en lugar de texto
        self.logo_label = QtWidgets.QLabel()
        try:
            logo_pixmap = QtGui.QPixmap("logo.png")
            logo_pixmap = logo_pixmap.scaled(180, 60, QtCore.Qt.KeepAspectRatio, QtCore.Qt.SmoothTransformation)
            self.logo_label.setPixmap(logo_pixmap)
        except:
            # Fallback si no se encuentra la imagen
            self.logo_label = QtWidgets.QLabel("PARKΞA2")
            self.logo_label.setFont(QtGui.QFont("Arial", 24, QtGui.QFont.Bold))
            self.logo_label.setStyleSheet(f"color: {self.accent_color.name()};")
        
        header_layout.addWidget(self.logo_label)
        
        # Espacio elástico
        header_layout.addStretch()
        
        # Hora actual con nuevo estilo
        time_layout = QtWidgets.QVBoxLayout()
        time_layout.setAlignment(QtCore.Qt.AlignRight)
        
        self.current_time_label = QtWidgets.QLabel("00:00:00")
        self.current_time_label.setFont(QtGui.QFont("Arial", 20, QtGui.QFont.Light))
        self.current_time_label.setStyleSheet(f"color: {self.accent_color.name()};")
        
        self.date_label = QtWidgets.QLabel("01 Enero 2023")
        self.date_label.setFont(QtGui.QFont("Arial", 12))
        self.date_label.setStyleSheet(f"color: {self.text_color.name()};")
        
        time_layout.addWidget(self.current_time_label)
        time_layout.addWidget(self.date_label)
        header_layout.addLayout(time_layout)
        
        main_layout.addWidget(header)
        
        # Separador elegante con nuevo estilo
        separator = QtWidgets.QFrame()
        separator.setFrameShape(QtWidgets.QFrame.HLine)
        separator.setStyleSheet(f"background-color: {self.accent_color.name()}; height: 1px; margin: 5px 0;")
        separator.setFixedHeight(1)
        main_layout.addWidget(separator)
        
        # --- TIEMPO EJECUCIÓN REDISEÑADO ---
        runtime_widget = QtWidgets.QWidget()
        runtime_layout = QtWidgets.QHBoxLayout(runtime_widget)
        runtime_layout.setContentsMargins(0, 0, 0, 0)
        
        # Tiempo de operación con nuevo estilo
        self.runtime_label = QtWidgets.QLabel("Operación: 00:00:00")
        self.runtime_label.setFont(QtGui.QFont("Arial", 14))
        self.runtime_label.setStyleSheet(f"""
            color: {self.text_color.name()};
            padding: 8px 15px;
            border-radius: 8px;
            background-color: {self.secondary_color.name()};
        """)
        self.runtime_label.setAlignment(QtCore.Qt.AlignCenter)
        runtime_layout.addWidget(self.runtime_label)
        
        # Espacio entre elementos
        runtime_layout.addSpacing(20)
        
        # Indicador del valor serial con nuevo estilo
        serial_value_frame = QtWidgets.QFrame()
        serial_value_frame.setStyleSheet(f"""
            background-color: {self.secondary_color.name()};
            border: 1px solid {self.accent_color.name()};
            border-radius: 8px;
            padding: 5px 15px;
        """)
        serial_value_layout = QtWidgets.QHBoxLayout(serial_value_frame)
        
        serial_title = QtWidgets.QLabel("Valor Serial:")
        serial_title.setFont(QtGui.QFont("Arial", 14))
        serial_title.setStyleSheet(f"color: {self.text_color.name()};")
        serial_value_layout.addWidget(serial_title)
        
        self.serial_value_label = QtWidgets.QLabel("0")
        self.serial_value_label.setFont(QtGui.QFont("Arial", 24, QtGui.QFont.Bold))
        self.serial_value_label.setStyleSheet(f"""
            color: {self.accent_color.name()};
            min-width: 40px;
            text-align: center;
        """)
        serial_value_layout.addWidget(self.serial_value_label)
        
        runtime_layout.addWidget(serial_value_frame)
        main_layout.addWidget(runtime_widget)
        
        # --- CUERPO PRINCIPAL REDISEÑADO ---
        body_widget = QtWidgets.QWidget()
        body_layout = QtWidgets.QHBoxLayout(body_widget)
        body_layout.setSpacing(20)
        body_layout.setContentsMargins(0, 0, 0, 0)
        
        # --- GRÁFICAS CON ESTILO ACTUALIZADO ---
        graphs_widget = QtWidgets.QWidget()
        graphs_layout = QtWidgets.QVBoxLayout(graphs_widget)
        graphs_layout.setSpacing(20)
        graphs_layout.setContentsMargins(0, 0, 0, 0)
        
        # Configuración de datos para gráficas
        self.max_samples = 250
        
        # Gráfica 1: Presencia de coche
        self.car_figure = Figure(figsize=(8, 3), dpi=100)
        self.car_canvas = FigureCanvas(self.car_figure)
        self.car_ax = self.car_figure.add_subplot(111)
        
        # Configurar estilo de la gráfica
        self.car_ax.set_title("Presencia de Vehículo", fontsize=14, color=self.text_color.name())
        self.car_ax.set_ylim(-0.1, 1.1)
        self.car_ax.set_xlim(0, self.max_samples)
        self.car_ax.set_yticks([0, 1])
        self.car_ax.set_yticklabels(['Ausente', 'Presente'], color=self.text_color.name())
        self.car_ax.grid(True, linestyle='--', alpha=0.7)
        self.car_ax.set_facecolor(self.secondary_color.name())
        self.car_figure.patch.set_facecolor(self.bg_color.name())
        
        # Color de los ejes
        self.car_ax.spines['bottom'].set_color(self.accent_color.name())
        self.car_ax.spines['top'].set_color(self.accent_color.name()) 
        self.car_ax.spines['right'].set_color(self.accent_color.name())
        self.car_ax.spines['left'].set_color(self.accent_color.name())
        
        # Crear línea inicial
        self.car_line, = self.car_ax.step([], [], where='post', 
                                         color=self.accent_color.name(), 
                                         linewidth=2)
        
        graphs_layout.addWidget(self.car_canvas)
        
        # Gráfica 2: Estado de la pluma
        self.gate_figure = Figure(figsize=(8, 3), dpi=100)
        self.gate_canvas = FigureCanvas(self.gate_figure)
        self.gate_ax = self.gate_figure.add_subplot(111)
        
        # Configurar estilo de la gráfica
        self.gate_ax.set_title("Estado de la Pluma", fontsize=14, color=self.text_color.name())
        self.gate_ax.set_ylim(-0.1, 1.1)
        self.gate_ax.set_xlim(0, self.max_samples)
        self.gate_ax.set_yticks([0, 1])
        self.gate_ax.set_yticklabels(['Abajo', 'Arriba'], color=self.text_color.name())
        self.gate_ax.grid(True, linestyle='--', alpha=0.7)
        self.gate_ax.set_facecolor(self.secondary_color.name())
        self.gate_figure.patch.set_facecolor(self.bg_color.name())
        
        # Color de los ejes
        self.gate_ax.spines['bottom'].set_color(self.accent_color.name())
        self.gate_ax.spines['top'].set_color(self.accent_color.name()) 
        self.gate_ax.spines['right'].set_color(self.accent_color.name())
        self.gate_ax.spines['left'].set_color(self.accent_color.name())
        
        # Crear línea inicial
        self.gate_line, = self.gate_ax.step([], [], where='post', 
                                           color=self.accent_color.name(), 
                                           linewidth=2)
        
        graphs_layout.addWidget(self.gate_canvas)
        
        body_layout.addWidget(graphs_widget, 70)  # 70% del espacio
        
        # --- PANEL DERECHO REDISEÑADO (CONTADOR) ---
        counter_widget = QtWidgets.QWidget()
        counter_layout = QtWidgets.QVBoxLayout(counter_widget)
        counter_layout.setAlignment(QtCore.Qt.AlignCenter)
        counter_layout.setContentsMargins(0, 0, 0, 0)
        
        counter_frame = QtWidgets.QFrame()
        counter_frame.setStyleSheet(f"""
            background-color: {self.secondary_color.name()};
            border: 1px solid {self.accent_color.name()};
            border-radius: 12px;
        """)
        counter_frame_layout = QtWidgets.QVBoxLayout(counter_frame)
        counter_frame_layout.setAlignment(QtCore.Qt.AlignCenter)
        counter_frame_layout.setContentsMargins(20, 20, 20, 20)
        counter_frame_layout.setSpacing(15)
        
        counter_title = QtWidgets.QLabel("Vehículos Ingresados")
        counter_title.setFont(QtGui.QFont("Arial", 16, QtGui.QFont.Bold))
        counter_title.setStyleSheet(f"color: {self.accent_color.name()};")
        counter_title.setAlignment(QtCore.Qt.AlignCenter)
        counter_frame_layout.addWidget(counter_title)
        
        self.counter_value = QtWidgets.QLabel("0")
        self.counter_value.setFont(QtGui.QFont("Arial", 48, QtGui.QFont.Light))
        self.counter_value.setStyleSheet(f"color: {self.accent_color.name()};")
        self.counter_value.setAlignment(QtCore.Qt.AlignCenter)
        counter_frame_layout.addWidget(self.counter_value)
        
        # Imagen del carro en lugar del icono dibujado
        car_image_label = QtWidgets.QLabel()
        try:
            car_pixmap = QtGui.QPixmap("carro.png")
            car_pixmap = car_pixmap.scaled(120, 120, QtCore.Qt.KeepAspectRatio, QtCore.Qt.SmoothTransformation)
            car_image_label.setPixmap(car_pixmap)
        except:
            # Fallback si no se encuentra la imagen
            car_image_label = QtWidgets.QLabel("🚗")
            car_image_label.setFont(QtGui.QFont("Arial", 48))
        
        car_image_label.setAlignment(QtCore.Qt.AlignCenter)
        counter_frame_layout.addWidget(car_image_label)
        
        counter_layout.addWidget(counter_frame)
        body_layout.addWidget(counter_widget, 30)  # 30% del espacio
        
        main_layout.addWidget(body_widget)
        
        # --- STATUS BAR REDISEÑADO ---
        self.status_bar = self.statusBar()
        self.status_bar.setStyleSheet(f"""
            QStatusBar {{
                background-color: {self.secondary_color.name()};
                color: {self.text_color.name()};
                border-top: 1px solid {self.accent_color.name()};
                font-size: 10pt;
                padding: 5px;
            }}
        """)
        self.status_bar.showMessage("Inicializando sistema...")
        
        # Añadir indicador de conexión serial en barra de estado
        self.serial_status = QtWidgets.QLabel("Serial: Conectando...")
        self.serial_status.setStyleSheet("color: #FFDE59; font-weight: bold;")
        self.status_bar.addPermanentWidget(self.serial_status)
        
        # Configuración de datos (resto de variables)
        self.car_data = deque([0] * self.max_samples, maxlen=self.max_samples)
        self.gate_data = deque([0] * self.max_samples, maxlen=self.max_samples)
        self.x_data = list(range(self.max_samples))
        self.start_time = time.time()
        self.car_count = 0
        self.gate_status = 0  # 0: abajo, 1: arriba
        self.gate_timer = None  # Temporizador para la pluma
        self.last_car_value = 0  # Guarda el último valor del vehículo
        self.serial_port = None
        self.serial_port_name = None
        self.last_reconnect_time = 0
        
        # Inicializar gráficos con datos vacíos
        self.update_plots()
        
        # Configurar puerto serial a 115200 baudios
        self.setup_serial()
        
        # Temporizadores
        self.clock_timer = QtCore.QTimer()
        self.clock_timer.timeout.connect(self.update_clock)
        self.clock_timer.start(1000)  # Actualizar cada segundo
        
        self.runtime_timer = QtCore.QTimer()
        self.runtime_timer.timeout.connect(self.update_runtime)
        self.runtime_timer.start(1000)
        
        self.data_timer = QtCore.QTimer()
        self.data_timer.timeout.connect(self.update_data)
        self.data_timer.start(50)  # Actualizar datos cada 50ms
        
        self.update_clock()  # Inicializar reloj

    def update_plots(self):
        """Actualiza las gráficas con los datos actuales"""
        # Gráfica de presencia de vehículo
        self.car_line.set_data(self.x_data, list(self.car_data))
        self.car_ax.set_xlim(0, len(self.car_data))
        self.car_canvas.draw_idle()
        
        # Gráfica de estado de la pluma
        self.gate_line.set_data(self.x_data, list(self.gate_data))
        self.gate_ax.set_xlim(0, len(self.gate_data))
        self.gate_canvas.draw_idle()

    def setup_serial(self):
        """Buscar y conectar al puerto serial"""
        self.status_bar.showMessage("Buscando puertos seriales...")
        
        # Obtener lista de puertos disponibles
        ports = serial.tools.list_ports.comports()
        if not ports:
            self.serial_status.setText("Serial: No se encontraron puertos")
            self.serial_status.setStyleSheet("color: #ff5555; font-weight: bold;")
            self.status_bar.showMessage("Error: No se encontraron puertos seriales disponibles")
            return
        
        # Intentar conectar a cada puerto hasta encontrar uno funcional
        for port in ports:
            port_name = port.device
            try:
                self.serial_port = serial.Serial(
                    port=port_name,
                    baudrate=115200,
                    bytesize=serial.EIGHTBITS,
                    parity=serial.PARITY_NONE,
                    stopbits=serial.STOPBITS_ONE,
                    timeout=0.1,
                    write_timeout=0.1
                )
                self.serial_port_name = port_name
                self.serial_status.setText(f"Serial: Conectado a {port_name} @ 115200 bps")
                self.serial_status.setStyleSheet("color: #55aa55; font-weight: bold;")
                self.status_bar.showMessage(f"Conectado a {port_name} - Sistema listo")
                
                # Limpiar buffer de entrada
                self.serial_port.reset_input_buffer()
                
                return  # Salir si la conexión fue exitosa
            except Exception as e:
                error_msg = str(e)
                if "Access is denied" in error_msg:
                    self.serial_status.setText(f"Serial: Acceso denegado a {port_name}")
                else:
                    self.serial_status.setText(f"Serial: Error en {port_name}")
                self.serial_status.setStyleSheet("color: #ff5555; font-weight: bold;")
                continue
        
        # Si llegamos aquí, no se pudo conectar a ningún puerto
        self.serial_status.setText("Serial: No se pudo conectar")
        self.serial_status.setStyleSheet("color: #ff5555; font-weight: bold;")
        self.status_bar.showMessage("Error: No se pudo conectar a ningún puerto serial")

    def update_clock(self):
        now = QtCore.QDateTime.currentDateTime()
        self.current_time_label.setText(now.toString("hh:mm:ss"))
        self.date_label.setText(now.toString("dd MMMM yyyy"))

    def update_runtime(self):
        runtime = int(time.time() - self.start_time)
        hours, remainder = divmod(runtime, 3600)
        minutes, seconds = divmod(remainder, 60)
        self.runtime_label.setText(f"Operación: {hours:02d}:{minutes:02d}:{seconds:02d}")

    def lower_gate(self):
        """Baja la pluma después de 6 segundos"""
        self.gate_status = 0
        self.gate_timer = None

    def update_data(self):
        # Si no hay conexión serial, intentar reconectar cada 5 segundos
        current_time = time.time()
        if not self.serial_port or not self.serial_port.is_open:
            if current_time - self.last_reconnect_time > 5:
                self.last_reconnect_time = current_time
                self.setup_serial()
            return
        
        car_value = 0
        try:
            if self.serial_port.in_waiting:
                # Leer todos los bytes disponibles
                raw_bytes = self.serial_port.read(self.serial_port.in_waiting)
                
                # Convertir a cadena, ignorando caracteres no válidos
                try:
                    raw_data = raw_bytes.decode('ascii', errors='ignore').strip()
                except UnicodeDecodeError:
                    raw_data = raw_bytes.decode('utf-8', errors='ignore').strip()
                
                # Procesar cada línea de datos
                for line in raw_data.splitlines():
                    line = line.strip()
                    if line:
                        # Mostrar datos crudos en la barra de estado para depuración
                        self.status_bar.showMessage(f"Datos recibidos: {line}")
                        
                        # Intentar convertir a entero
                        try:
                            car_value = int(line)
                            break  # Usar el primer valor válido
                        except ValueError:
                            # Si no es un número, buscar valores booleanos
                            line_lower = line.lower()
                            if line_lower in ["1", "true", "on", "high"]:
                                car_value = 1
                            elif line_lower in ["0", "false", "off", "low"]:
                                car_value = 0
                
                # Actualizar indicador de valor serial
                self.serial_value_label.setText(str(car_value))
        except Exception as e:
            # En caso de error, cerrar el puerto e intentar reconectar
            if self.serial_port:
                try:
                    self.serial_port.close()
                except:
                    pass
            self.serial_port = None
            self.serial_status.setText(f"Serial: Error - {str(e)}")
            self.serial_status.setStyleSheet("color: #ff5555; font-weight: bold;")
            self.status_bar.showMessage(f"Error de lectura: {str(e)} - Intentando reconectar")
            return
        
        # Guardar el último valor para detección de transiciones
        current_car_value = car_value
        self.last_car_value = current_car_value
        
        # Actualizar datos del vehículo
        self.car_data.append(current_car_value)
        
        # Si detectamos un vehículo (1)
        if current_car_value == 1:
            # Contar solo en transición de 0 a 1
            if len(self.car_data) < 2 or self.car_data[-2] == 0:
                self.car_count += 1
                self.counter_value.setText(str(self.car_count))
            
            # Subir la pluma
            self.gate_status = 1
            
            # Cancelar temporizador anterior si existe
            if self.gate_timer:
                self.gate_timer.cancel()
            
            # Crear nuevo temporizador para bajar la pluma en 6 segundos
            self.gate_timer = threading.Timer(6.0, self.lower_gate)
            self.gate_timer.daemon = True  # Para que no impida el cierre de la aplicación
            self.gate_timer.start()
        
        # Actualizar datos de la pluma
        self.gate_data.append(self.gate_status)
        
        # Actualizar gráficas
        self.update_plots()

    def closeEvent(self, event):
        if hasattr(self, 'serial_port') and self.serial_port and self.serial_port.is_open:
            try:
                self.serial_port.close()
            except:
                pass
        if self.gate_timer:
            self.gate_timer.cancel()
        event.accept()


if __name__ == "__main__":
    app = QtWidgets.QApplication(sys.argv)
    app.setStyle("Fusion")  # Para un aspecto más moderno
    
    # Aplicar estilo global actualizado
    app.setStyleSheet("""
        * {
            font-family: 'Arial';
            background-color: #1D2024;
            color: #FFFFFF;
        }
        QMainWindow {
            background-color: #1D2024;
        }
        QLabel {
            color: #FFFFFF;
        }
        QFrame {
            border: none;
        }
    """)
    
    window = Parkea2GUIMinimalist()
    
    # Añadir sombra sutil para profundidad
    shadow = QtWidgets.QGraphicsDropShadowEffect()
    shadow.setBlurRadius(25)
    shadow.setColor(QtGui.QColor(0, 0, 0, 80))
    shadow.setOffset(5, 5)
    window.setGraphicsEffect(shadow)
    
    window.show()
    sys.exit(app.exec_())
