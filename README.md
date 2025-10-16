# ferminmurillo.github.io
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Transferencia Móvil de Fotos</title>
    <!-- Carga de Tailwind CSS para un diseño limpio y responsivo -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f9fb;
        }
        /* Ocultar el input de archivo y usar el botón personalizado */
        #photo-upload {
            display: none;
        }
    </style>
</head>
<body class="min-h-screen flex items-start justify-center p-4 sm:p-8">
    
    <div class="w-full max-w-xl bg-white shadow-2xl rounded-xl p-6 sm:p-10 mt-8">
        
        <h1 class="text-3xl font-bold text-center text-blue-700 mb-2">
            📸 Photo Bridge
        </h1>
        <p class="text-center text-gray-500 mb-8">
            Transfiere fotos entre tu iPhone/iPad y tu dispositivo Android (y viceversa).
        </p>

        <!-- Sección de Subida -->
        <div class="mb-8 p-6 bg-blue-50 rounded-lg border border-blue-200">
            <h2 class="text-xl font-semibold text-blue-600 mb-3">
                Paso 1: Dispositivo de Origen (Subir)
            </h2>
            <p class="text-gray-600 mb-4 text-sm">
                Selecciona las imágenes que quieres transferir. Recuerda: los archivos solo se almacenan temporalmente en la memoria de este navegador.
            </p>

            <!-- Input de Archivo (oculto) -->
            <input type="file" id="photo-upload" accept="image/*" multiple>

            <!-- Botón de Subida (etiqueta personalizada) -->
            <label for="photo-upload" id="upload-label" class="cursor-pointer inline-flex items-center justify-center w-full px-6 py-3 border border-transparent text-base font-medium rounded-lg shadow-sm text-white bg-green-500 hover:bg-green-600 transition duration-150 ease-in-out">
                Seleccionar y Cargar Fotos
            </label>
        </div>

        <!-- Sección de Estado y Descarga -->
        <div class="p-6 bg-white rounded-lg border border-gray-200">
            <h2 class="text-xl font-semibold text-gray-700 mb-3">
                Paso 2: Dispositivo de Destino (Descargar)
            </h2>
            <p class="text-gray-600 mb-4 text-sm">
                Abre esta misma URL en el otro dispositivo para ver y descargar los archivos cargados.
            </p>

            <!-- Indicador de carga -->
            <div id="loading-indicator" class="hidden flex items-center justify-center p-4 bg-yellow-100 text-yellow-700 rounded-lg">
                <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-yellow-700" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
                <span>Cargando archivos...</span>
            </div>

            <!-- Lista de Archivos para Descarga -->
            <ul id="file-list" class="mt-4 space-y-3">
                <li id="initial-message" class="text-gray-400 text-center py-4">
                    Ninguna foto cargada aún.
                </li>
            </ul>
        </div>

    </div>

    <script>
        // Configuración de la aplicación
        const fileInput = document.getElementById('photo-upload');
        const fileList = document.getElementById('file-list');
        const initialMessage = document.getElementById('initial-message');
        const loadingIndicator = document.getElementById('loading-indicator');
        const uploadLabel = document.getElementById('upload-label');

        // Array para almacenar temporalmente los objetos de archivo cargados
        let loadedFiles = []; 

        /**
         * Maneja el evento de cambio cuando se seleccionan archivos.
         */
        fileInput.addEventListener('change', async (event) => {
            const files = event.target.files;
            if (files.length === 0) return;

            // Mostrar indicador de carga
            initialMessage.classList.add('hidden');
            fileList.innerHTML = ''; // Limpiar la lista anterior
            loadingIndicator.classList.remove('hidden');

            // Revocar URL anteriores para liberar memoria (Buena práctica)
            loadedFiles.forEach(file => URL.revokeObjectURL(file.url));
            loadedFiles = []; 
            
            const processingPromises = [];

            // Procesar cada archivo seleccionado
            Array.from(files).forEach(file => {
                const promise = new Promise(resolve => {
                    // Crea una URL temporal para el archivo
                    const fileURL = URL.createObjectURL(file);

                    // Almacenar el objeto de archivo para la lista de descargas
                    loadedFiles.push({
                        name: file.name,
                        url: fileURL
                    });
                    
                    // Añadir visualmente el archivo a la lista
                    appendToFileList(file.name, fileURL);
                    resolve();
                });
                processingPromises.push(promise);
            });

            // Esperar a que todos los archivos sean procesados (muy rápido)
            await Promise.all(processingPromises);

            // Ocultar indicador de carga
            loadingIndicator.classList.add('hidden');

            // Mensaje de éxito
            const count = loadedFiles.length;
            const message = `${count} foto(s) cargada(s) temporalmente. ¡Abre esta misma URL en el otro dispositivo para descargar!`;
            
            // Usar un modal simple en lugar de alert()
            showTemporaryMessage(message, 'success');
        });

        /**
         * Muestra un archivo en la lista de descargas.
         * @param {string} fileName - Nombre del archivo.
         * @param {string} fileURL - URL temporal del archivo (blob URL).
         */
        function appendToFileList(fileName, fileURL) {
            const listItem = document.createElement('li');
            listItem.className = 'flex items-center justify-between p-3 bg-gray-50 rounded-md hover:bg-gray-100 transition';
            
            listItem.innerHTML = `
                <span class="text-sm font-medium text-gray-700 truncate mr-4">
                    ${fileName}
                </span>
                <a href="${fileURL}" download="${fileName}" class="flex-shrink-0 px-3 py-1 text-xs font-semibold rounded-full text-white bg-blue-500 hover:bg-blue-600 transition duration-150 download-link">
                    Descargar
                </a>
            `;
            fileList.appendChild(listItem);
        }

        /**
         * Implementación simple de un modal de mensaje temporal (sustituto de alert()).
         * @param {string} message - El mensaje a mostrar.
         * @param {string} type - El tipo de mensaje ('success', 'error', etc.) para el estilo.
         */
        function showTemporaryMessage(message, type) {
            const container = document.body;
            let messageBox = document.getElementById('temp-message-box');

            if (!messageBox) {
                messageBox = document.createElement('div');
                messageBox.id = 'temp-message-box';
                messageBox.className = 'fixed top-4 left-1/2 transform -translate-x-1/2 z-50 p-4 rounded-lg shadow-xl text-white transition-opacity duration-300 opacity-0';
                container.appendChild(messageBox);
            }

            // Aplicar estilo basado en el tipo
            messageBox.className = 'fixed top-4 left-1/2 transform -translate-x-1/2 z-50 p-4 rounded-lg shadow-xl text-white transition-opacity duration-300 opacity-0';
            if (type === 'success') {
                messageBox.classList.add('bg-green-500');
            } else if (type === 'error') {
                messageBox.classList.add('bg-red-500');
            } else {
                messageBox.classList.add('bg-gray-700');
            }

            messageBox.innerHTML = message;

            // Mostrar el mensaje
            setTimeout(() => {
                messageBox.classList.add('opacity-100');
            }, 50);

            // Ocultar el mensaje después de 4 segundos
            setTimeout(() => {
                messageBox.classList.remove('opacity-100');
                // Eliminar del DOM después de la transición
                setTimeout(() => {
                    if (messageBox.parentElement) {
                        messageBox.parentElement.removeChild(messageBox);
                    }
                }, 300);
            }, 4000);
        }

        // ----------------------------------------------------
        // Lógica de Soporte para la URL temporal (Advertencia)
        // ----------------------------------------------------
        
        // Simplemente un mensaje de consola para recordar la limitación
        console.warn("ADVERTENCIA: Esta solución depende de 'URL.createObjectURL', lo que significa que las fotos solo existen en el navegador donde fueron subidas. Para una transferencia real entre dos dispositivos distintos, se requiere un servidor backend para el almacenamiento.");

    </script>

</body>
</html>
