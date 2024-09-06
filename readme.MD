`Instalacion y Creacion de Web en django`

[Python](https://www.python.org/downloads/)

----

- En Poweshell (SH)
```sh
pip install django
```


```sh
django-admin startproject Reconocimiento_Facial

cd Reconocimiento_Facial

python manage.py startapp usuarios
```

- Instalar dependencias necesarias para el reconocimiento
```sh
pip install django opencv-python-headless tensorflow Pillow

```


----
- # Configuracion del Proyecto en VsCode
----

- #### Configuracion Settings.py
se agrego `"usuarios"` a `INSTALLED_APPS` *(reconocimiento_Facial/settings.py)*

```python
INSTALLED_APPS = [
    'usuarios',
]
```

----

- Creacion de modelos: Dentro de usuarios/models.py:

```python 
from django.db import models

class Usuario(models.Model):
    nombre = models.CharField(max_length=100)
    rut = models.CharField(max_length=10, unique=True)
    embedding_facial = models.BinaryField()

    def __str__(self):
        return self.nombre
```
----
- Configuracion Templates
----

- #### Crear Carpeta para los templates
En la carpeta del proyecto (`"Reconocimiento_facial"`), creamos una carpeta llamada `"templates" `
luego otra carpeta llamada igual que la aplicacion (`"usuarios"`)
y dentro de esa carpeta agregamos los .html

----

- #### Agregar los Templates a settings.py `(reconocimiento_Facial/settings.py)`

En el settings.py importamos OS
```python
import os
```
Y agregamos la carpeta en la parte de `'DIRS'` tal cual esta en el codigo 
(solo esa parte se modifica)

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')] ## Esto se modifica,
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                ...
            ],
        },
    },
]
```

----
- Creacion de Templates
----

- ###  \registro.html

```HTML

{% load static %}
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Registro de Usuario</title>
    <link rel="stylesheet" href="{% static 'css/bootstrap.min.css' %}">
</head>
<body>
    <div class="container">
        <h2>Registro de Usuario</h2>
        <form method="post" enctype="multipart/form-data" id="registroForm">
            {% csrf_token %}
            {{ form.as_p }}
            <div class="mb-3">
                <label for="videoInput" class="form-label">Captura de Video</label>
                <video id="videoElement" width="480" height="320" autoplay></video>
                <button type="button" class="btn btn-primary mt-3" onclick="startRecording()">Iniciar Grabación</button>
                <button type="button" class="btn btn-danger mt-3" onclick="stopRecording()">Detener Grabación</button>
                <input type="hidden" id="videoData" name="video_data">
            </div>
            <button type="submit" class="btn btn-primary">Registrar</button>
        </form>
    </div>

    <script>
        let videoElement = document.getElementById('videoElement');
        let mediaRecorder;
        let chunks = [];

        navigator.mediaDevices.getUserMedia({ video: true })
            .then(function(stream) {
                videoElement.srcObject = stream;
                mediaRecorder = new MediaRecorder(stream);

                mediaRecorder.ondataavailable = function(event) {
                    if (event.data.size > 0) {
                        chunks.push(event.data);
                    }
                };

                mediaRecorder.onstop = function() {
                    let blob = new Blob(chunks, { type: 'video/webm' });
                    chunks = [];
                    let reader = new FileReader();
                    reader.readAsDataURL(blob);
                    reader.onloadend = function() {
                        document.getElementById('videoData').value = reader.result;
                    }
                };
            })
            .catch(function(error) {
                console.log("Error al acceder a la cámara: " + error);
            });

        function startRecording() {
            chunks = [];
            mediaRecorder.start();
        }

        function stopRecording() {
            mediaRecorder.stop();
        }
    </script>

    <script src="{% static 'js/bootstrap.bundle.min.js' %}"></script>
</body>
</html>
```

- ### \LOGIN.HTML

```HTML

{% load static %}
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Login de Usuario</title>
    <link rel="stylesheet" href="{% static 'css/bootstrap.min.css' %}">
</head>
<body>
    <div class="container">
        <h2>Login de Usuario</h2>
        <video id="videoElement" width="480" height="320" autoplay></video>
        <button type="button" class="btn btn-primary mt-3" onclick="captureImage()">Iniciar Sesión</button>
        <form method="post" enctype="multipart/form-data" id="loginForm">
            {% csrf_token %}
            <input type="hidden" id="imageData" name="image_data">
        </form>
    </div>

    <script>
        let videoElement = document.getElementById('videoElement');

        navigator.mediaDevices.getUserMedia({ video: true })
            .then(function(stream) {
                videoElement.srcObject = stream;
            })
            .catch(function(error) {
                console.log("Error al acceder a la cámara: " + error);
            });

        function captureImage() {
            let canvas = document.createElement('canvas');
            canvas.width = videoElement.videoWidth;
            canvas.height = videoElement.videoHeight;
            canvas.getContext('2d').drawImage(videoElement, 0, 0);
            let imageData = canvas.toDataURL('image/png');
            document.getElementById('imageData').value = imageData;
            document.getElementById('loginForm').submit();
        }
    </script>

    <script src="{% static 'js/bootstrap.bundle.min.js' %}"></script>
</body>
</html>
```

----
- Creacion de Vistas
----

- ### Vistas Vista de Registro Dentro de usuarios/views.py:

```python 

import cv2
import numpy as np
import base64
from django.shortcuts import render, redirect
from .models import Usuario
from .forms import RegistroForm
from tensorflow.keras.models import load_model
from io import BytesIO
from PIL import Image

# Cargar modelo preentrenado de FaceNet
facenet_model = load_model('ruta_al_modelo_facenet.h5')

def generar_embedding(imagen):
    imagen = cv2.resize(imagen, (160, 160))
    imagen = np.expand_dims(imagen, axis=0)
    embedding = facenet_model.predict(imagen)
    return embedding

def registrar_usuario(request):
    if request.method == 'POST':
        form = RegistroForm(request.POST, request.FILES)
        if form.is_valid():
            nombre = form.cleaned_data['nombre']
            rut = form.cleaned_data['rut']
            video_data = form.cleaned_data['video_data']

            # Decodificar el video base64
            video_decoded = base64.b64decode(video_data.split(',')[1])
            video_path = '/tmp/temp_video.webm'  # Ruta temporal para guardar el video
            with open(video_path, 'wb') as f:
                f.write(video_decoded)

            # Captura y procesamiento de frames desde el video
            video_capture = cv2.VideoCapture(video_path)
            embeddings = []
            while True:
                ret, frame = video_capture.read()
                if not ret:
                    break
                embedding = generar_embedding(frame)
                embeddings.append(embedding)
            video_capture.release()

            # Promedio de embeddings para mayor precisión
            if embeddings:
                embedding_promedio = np.mean(embeddings, axis=0).tobytes()

                # Guardar usuario en la base de datos
                usuario = Usuario(nombre=nombre, rut=rut, embedding_facial=embedding_promedio)
                usuario.save()

            return redirect('login')
    else:
        form = RegistroForm()
    return render(request, 'usuarios/registro.html', {'form': form})

```
- ### Vista de Login Dentro de usuarios/views.py:

```python

def login_usuario(request):
    if request.method == 'POST':
        image_data = request.POST.get('image_data')

        # Decodificar la imagen base64
        image_decoded = base64.b64decode(image_data.split(',')[1])
        image = Image.open(BytesIO(image_decoded))
        frame = np.array(image)

        # Generar embedding de la imagen
        embedding_actual = generar_embedding(frame)

        # Comparar con embeddings en la base de datos
        for usuario in Usuario.objects.all():
            embedding_db = np.frombuffer(usuario.embedding_facial, dtype=np.float32)
            if np.linalg.norm(embedding_actual - embedding_db) < 0.6:  # Ajusta el umbral según sea necesario
                return redirect('menu_principal', usuario_id=usuario.id)
        return render(request, 'usuarios/login.html', {'error': 'Autenticación fallida'})
    return render(request, 'usuarios/login.html')
    
```

----
----


- #### Migrar el modelo a sqlite
```sh
python manage.py makemigrations
```

```sh
python manage.py migrate
```


- #### Configurar los URLS.PY
(FALTA)
