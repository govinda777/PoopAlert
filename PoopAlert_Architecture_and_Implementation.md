# PoopAlert - Arquitetura e Implementação do MVP

Este documento descreve a arquitetura de ponta a ponta, a estrutura do projeto Android, os principais códigos Kotlin/XML e o pipeline em Python para o treinamento do modelo de Inteligência Artificial do **PoopAlert**.

---

## 1. Estrutura de Pastas do Projeto Android

No Android Studio, crie um novo projeto "Empty Views Activity" (Kotlin). A estrutura de pastas deverá seguir este padrão:

```text
PoopAlert/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── AndroidManifest.xml
│   │   │   ├── java/com/example/poopalert/
│   │   │   │   ├── MainActivity.kt               # Setup, Permissões e Coleta
│   │   │   │   ├── MonitoringActivity.kt         # Modo Sentinela e Alarme
│   │   │   │   └── ObjectDetectorHelper.kt       # Inferência TFLite e Lógica Temporal
│   │   │   ├── res/
│   │   │   │   ├── layout/
│   │   │   │   │   ├── activity_main.xml         # Tela de Câmera e Botões
│   │   │   │   │   └── activity_monitoring.xml   # Tela Preta e Botão de Silenciar
│   │   │   ├── assets/
│   │   │   │   └── model.tflite                  # Modelo placeholder (será substituído)
│   ├── build.gradle (Module: app)
├── build.gradle (Project)
```

---

## 2. Configurações e Dependências Android

### 2.1 `AndroidManifest.xml`
Adicione as permissões essenciais para uso de câmera, armazenamento e execução de alarme em background.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.poopalert">

    <uses-feature android:name="android.hardware.camera" />
    <uses-permission android:name="android.permission.CAMERA" />
    <!-- Necessário para salvar fotos da coleta de dados em Androids mais antigos -->
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="28" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="PoopAlert"
        android:theme="@style/Theme.PoopAlert">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name=".MonitoringActivity" />
    </application>
</manifest>
```

### 2.2 `build.gradle` (Module: app)
Adicione as dependências do CameraX para lidar com fluxo de câmera facilmente e do TensorFlow Lite para a IA.

```gradle
dependencies {
    implementation 'androidx.core:core-ktx:1.9.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.8.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'

    // CameraX
    def camerax_version = "1.2.2"
    implementation "androidx.camera:camera-core:${camerax_version}"
    implementation "androidx.camera:camera-camera2:${camerax_version}"
    implementation "androidx.camera:camera-lifecycle:${camerax_version}"
    implementation "androidx.camera:camera-view:${camerax_version}"

    // TensorFlow Lite
    implementation 'org.tensorflow:tensorflow-lite:2.13.0'
    implementation 'org.tensorflow:tensorflow-lite-support:0.4.4'
    implementation 'org.tensorflow:tensorflow-lite-gpu:2.13.0' // Opcional, para aceleração via Edge AI
}

// Configuração para evitar compressão do arquivo TFLite no build
android {
    aaptOptions {
        noCompress "tflite"
    }
}
```

---

## 3. Interfaces (Layouts XML)

### 3.1 `activity_main.xml` (Setup e Coleta de Dados)
Contém o preview da câmera, botões para modo coleta de dados e iniciar o monitoramento.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.camera.view.PreviewView
        android:id="@+id/viewFinder"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <Button
        android:id="@+id/btnDataCollection"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="📸 MODO COLETA DE DADOS"
        app:layout_constraintBottom_toTopOf="@+id/btnStartMonitoring"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        android:layout_marginBottom="16dp"/>

    <Button
        android:id="@+id/btnStartMonitoring"
        android:layout_width="0dp"
        android:layout_height="80dp"
        android:text="🛡️ INICIAR MONITORAMENTO"
        android:backgroundTint="#4CAF50"
        android:textSize="18sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        android:layout_margin="16dp"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

### 3.2 `activity_monitoring.xml` (Modo Sentinela)
Tela preta para economia de energia. Quando o alarme disparar, a tela ficará vermelha com o botão para silenciar.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/monitoringRoot"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#000000"> <!-- Fundo Preto (Sentinela) -->

    <!-- Câmera invisível usada apenas para processar frames em background -->
    <androidx.camera.view.PreviewView
        android:id="@+id/hiddenViewFinder"
        android:layout_width="1dp"
        android:layout_height="1dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent" />

    <TextView
        android:id="@+id/tvStatus"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Monitoramento Ativo.\nTela bloqueada para economizar bateria."
        android:textColor="#888888"
        android:textAlignment="center"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"/>

    <Button
        android:id="@+id/btnStopAlarm"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="SILENCIAR ALARME"
        android:visibility="gone"
        android:textSize="24sp"
        android:padding="32dp"
        android:backgroundTint="#FF0000"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## 4. Códigos Kotlin (Lógica do App)

### 4.1 `MainActivity.kt` (Permissões, Setup e Time-lapse)
```kotlin
package com.example.poopalert

import android.Manifest
import android.content.Intent
import android.content.pm.PackageManager
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.widget.Button
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.CameraSelector
import androidx.camera.core.ImageCapture
import androidx.camera.core.Preview
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    private lateinit var viewFinder: PreviewView
    private var imageCapture: ImageCapture? = null
    private lateinit var cameraExecutor: ExecutorService

    private var isCollecting = false
    private val handler = Handler(Looper.getMainLooper())

    // Variável configurável de intervalo do time-lapse (em milissegundos)
    // 5000L = 1 foto a cada 5 segundos
    private val TIME_LAPSE_INTERVAL = 5000L

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        viewFinder = findViewById(R.id.viewFinder)
        cameraExecutor = Executors.newSingleThreadExecutor()

        if (allPermissionsGranted()) {
            startCamera()
        } else {
            ActivityCompat.requestPermissions(this, REQUIRED_PERMISSIONS, REQUEST_CODE_PERMISSIONS)
        }

        findViewById<Button>(R.id.btnDataCollection).setOnClickListener {
            toggleDataCollection(it as Button)
        }

        findViewById<Button>(R.id.btnStartMonitoring).setOnClickListener {
            val intent = Intent(this, MonitoringActivity::class.java)
            startActivity(intent)
        }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()
            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(viewFinder.surfaceProvider)
            }
            imageCapture = ImageCapture.Builder().build()
            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(this, cameraSelector, preview, imageCapture)
            } catch (exc: Exception) { }
        }, ContextCompat.getMainExecutor(this))
    }

    private fun toggleDataCollection(btn: Button) {
        isCollecting = !isCollecting
        if (isCollecting) {
            btn.text = "⏹️ PARAR COLETA (Time-lapse ativo)"
            startTimelapse()
        } else {
            btn.text = "📸 MODO COLETA DE DADOS"
            handler.removeCallbacksAndMessages(null)
        }
    }

    private fun startTimelapse() {
        handler.post(object : Runnable {
            override fun run() {
                takePhoto()
                handler.postDelayed(this, TIME_LAPSE_INTERVAL)
            }
        })
    }

    private fun takePhoto() {
        // Lógica para capturar foto usando ImageCapture e salvar na pasta.
        // Implementação detalhada no MVP final dependerá do armazenamento Android (MediaStore).
        Toast.makeText(this, "Foto coletada!", Toast.LENGTH_SHORT).show()
    }

    private fun allPermissionsGranted() = REQUIRED_PERMISSIONS.all {
        ContextCompat.checkSelfPermission(baseContext, it) == PackageManager.PERMISSION_GRANTED
    }

    companion object {
        private const val REQUEST_CODE_PERMISSIONS = 10
        private val REQUIRED_PERMISSIONS = arrayOf(Manifest.permission.CAMERA)
    }
}
```

### 4.2 `MonitoringActivity.kt` (Modo Sentinela e Lógica do Alarme)
```kotlin
package com.example.poopalert

import android.graphics.Color
import android.media.Ringtone
import android.media.RingtoneManager
import android.os.Bundle
import android.view.View
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.CameraSelector
import androidx.camera.core.ImageAnalysis
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.core.content.ContextCompat
import java.util.concurrent.Executors

class MonitoringActivity : AppCompatActivity() {

    private lateinit var objectDetectorHelper: ObjectDetectorHelper
    private var alarmRingtone: Ringtone? = null
    private var isAlarmRinging = false

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_monitoring)

        val defaultAlarmUri = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM)
        alarmRingtone = RingtoneManager.getRingtone(applicationContext, defaultAlarmUri)

        objectDetectorHelper = ObjectDetectorHelper(
            context = this,
            objectDetectorListener = object : ObjectDetectorHelper.DetectorListener {
                override fun onObjectDetected() {
                    runOnUiThread { triggerAlarm() }
                }
            }
        )

        findViewById<Button>(R.id.btnStopAlarm).setOnClickListener {
            stopAlarm()
        }

        startHiddenCamera()
    }

    private fun startHiddenCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val imageAnalyzer = ImageAnalysis.Builder()
                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                .build()
                .also {
                    it.setAnalyzer(Executors.newSingleThreadExecutor()) { imageProxy ->
                        if (!isAlarmRinging) {
                            objectDetectorHelper.detect(imageProxy)
                        } else {
                            imageProxy.close()
                        }
                    }
                }

            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA
            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(this, cameraSelector, imageAnalyzer)
            } catch (exc: Exception) { }
        }, ContextCompat.getMainExecutor(this))
    }

    private fun triggerAlarm() {
        if (isAlarmRinging) return
        isAlarmRinging = true

        findViewById<View>(R.id.monitoringRoot).setBackgroundColor(Color.RED)
        findViewById<TextView>(R.id.tvStatus).visibility = View.GONE
        findViewById<Button>(R.id.btnStopAlarm).visibility = View.VISIBLE

        alarmRingtone?.play()
    }

    private fun stopAlarm() {
        isAlarmRinging = false
        alarmRingtone?.stop()

        findViewById<View>(R.id.monitoringRoot).setBackgroundColor(Color.BLACK)
        findViewById<TextView>(R.id.tvStatus).visibility = View.VISIBLE
        findViewById<Button>(R.id.btnStopAlarm).visibility = View.GONE

        // Zera o contador temporal para nova detecção
        objectDetectorHelper.resetTemporalCounter()
    }

    override fun onDestroy() {
        super.onDestroy()
        alarmRingtone?.stop()
    }
}
```

### 4.3 `ObjectDetectorHelper.kt` (Lógica de TFLite e Filtro de Persistência)
```kotlin
package com.example.poopalert

import android.content.Context
import androidx.camera.core.ImageProxy

class ObjectDetectorHelper(
    val context: Context,
    val objectDetectorListener: DetectorListener
) {
    interface DetectorListener {
        fun onObjectDetected()
    }

    // Configurações do Filtro Temporal e Confiança
    private val CONFIDENCE_THRESHOLD = 0.75f
    private val REQUIRED_CONSECUTIVE_FRAMES = 4
    private var consecutiveFramesDetected = 0

    // TODO: Inicialize seu modelo TFLite aqui utilizando Interpreter do TensorFlow Lite
    // val tflite = Interpreter(loadModelFile("model.tflite"))

    fun detect(imageProxy: ImageProxy) {
        // 1. Converter ImageProxy para Bitmap/TensorImage
        // 2. Passar a imagem para o interpretador TFLite
        // 3. Obter o array de resultados (Bounding boxes, Classes e Confianças)

        // Simulação do resultado da inferência:
        val isDogPoopDetected = mockTfliteInference(imageProxy)

        if (isDogPoopDetected) {
            consecutiveFramesDetected++
            if (consecutiveFramesDetected >= REQUIRED_CONSECUTIVE_FRAMES) {
                objectDetectorListener.onObjectDetected()
            }
        } else {
            // Se o objeto sumiu ou caiu abaixo do limiar, resetar o contador
            consecutiveFramesDetected = 0
        }

        imageProxy.close() // SEMPRE fechar a imagem após processamento
    }

    private fun mockTfliteInference(imageProxy: ImageProxy): Boolean {
        // Lógica real validará se "class == dog_poop" e "confidence >= CONFIDENCE_THRESHOLD"
        return false // Placeholder
    }

    fun resetTemporalCounter() {
        consecutiveFramesDetected = 0
    }
}
```

---

## 5. IA Edge Computing - Treinamento e Exportação

O script abaixo utiliza Python e a biblioteca **Ultralytics** para treinar a sua base de dados rotulada (exportada do Roboflow em formato YOLO) usando o modelo YOLOv11n (ou YOLOv8n) e, em seguida, exportar o modelo para TFLite (TensorFlow Lite), adequado para o Android.

### 5.1 `train_export_yolo.py`

```python
# Instalação das dependências (execute no terminal ou Google Colab):
# pip install ultralytics

from ultralytics import YOLO

def main():
    # 1. Carregar um modelo YOLO base super leve (nano)
    # ideal para dispositivos móveis
    model = YOLO("yolo11n.pt")

    print("Iniciando o Treinamento...")
    # 2. Treinar o modelo com o dataset
    # Substitua "data.yaml" pelo caminho do dataset exportado pelo Roboflow
    results = model.train(
        data="data.yaml",
        epochs=100,         # Número de épocas recomendadas para começar
        imgsz=320,          # Tamanho da imagem reduzido para acelerar inferência mobile
        batch=16,
        name="poop_alert_model"
    )

    print("Treinamento finalizado. Iniciando validação...")
    # 3. Validar o modelo treinado
    metrics = model.val()
    print(f"Precisão (mAP50): {metrics.box.map50}")

    print("Exportando modelo para TensorFlow Lite (TFLite)...")
    # 4. Exportar o modelo para TFLite (formato suportado nativamente pelo Android)
    # A exportação usará INT8 ou FP16 otimizando tamanho e velocidade no celular
    export_path = model.export(format="tflite", int8=True)

    print(f"Exportação concluída com sucesso! Arquivo gerado: {export_path}")
    print("Pegue o arquivo best_saved_model/best_full_integer_quant.tflite")
    print("Renomeie para 'model.tflite' e cole na pasta app/src/main/assets do Android Studio.")

if __name__ == "__main__":
    main()
```

---

## 6. Instruções de Compilação

1. **Abra o Android Studio**, clique em `File > New > New Project` e selecione `Empty Views Activity`. Nomeie como `PoopAlert`.
2. Adicione as dependências listadas no passo 2 no `build.gradle (app)`.
3. Crie a pasta `assets` clicando com o botão direito em `app/src/main` > `New` > `Folder` > `Assets Folder`.
4. Coloque um arquivo `.tflite` de placeholder (vazio) lá para evitar erros de compilação ou integre imediatamente seu modelo treinado.
5. Copie os códigos `.kt` e `.xml` para suas respectivas pastas.
6. Clique em **Sync Project with Gradle Files**.
7. Conecte o celular antigo via USB, certifique-se de que o modo *Desenvolvedor* e *Depuração USB* estão ativos.
8. Clique em **Run 'app'**.
