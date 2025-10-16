# ferminmurillo.github.io
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Transferencia Móvil de Fotos (Firebase)</title>
    <!-- Carga de Tailwind CSS -->
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
    
    <!-- Contenedor Principal -->
    <div class="w-full max-w-xl bg-white shadow-2xl rounded-xl p-6 sm:p-10 mt-8">
        
        <h1 class="text-3xl font-bold text-center text-blue-700 mb-2">
            📸 Photo Bridge (Cloud)
        </h1>
        <p class="text-center text-gray-500 mb-8">
            Transfiere fotos en tiempo real usando Firebase Storage.
        </p>

        <!-- Sección de Subida -->
        <div id="upload-section" class="mb-8 p-6 bg-blue-50 rounded-lg border border-blue-200">
            <h2 class="text-xl font-semibold text-blue-600 mb-3">
                Subir Fotos
            </h2>
            <p class="text-gray-600 mb-4 text-sm">
                Selecciona las imágenes. Se subirán a la nube y estarán disponibles para el otro dispositivo al instante.
            </p>

            <!-- Input de Archivo (oculto) -->
            <input type="file" id="photo-upload" accept="image/*" multiple>

            <!-- Botón de Subida (etiqueta personalizada) -->
            <label for="photo-upload" id="upload-label" class="cursor-pointer inline-flex items-center justify-center w-full px-6 py-3 border border-transparent text-base font-medium rounded-lg shadow-sm text-white bg-green-500 hover:bg-green-600 transition duration-150 ease-in-out">
                Seleccionar y Subir Fotos
            </label>
        </div>

        <!-- Sección de Estado y Descarga -->
        <div class="p-6 bg-white rounded-lg border border-gray-200">
            <h2 class="text-xl font-semibold text-gray-700 mb-3">
                Fotos para Descarga (Tiempo Real)
            </h2>
            <div id="auth-status" class="text-xs text-gray-500 mb-4 truncate">
                <!-- El estado de autenticación (ID de Usuario) se mostrará aquí -->
            </div>

            <!-- Indicador de carga / Mensajes -->
            <div id="status-message" class="hidden flex items-center justify-center p-4 bg-yellow-100 text-yellow-700 rounded-lg">
                <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-yellow-700" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
                <span>Conectando...</span>
            </div>

            <!-- Lista de Archivos para Descarga -->
            <ul id="file-list" class="mt-4 space-y-3">
                <li id="initial-message" class="text-gray-400 text-center py-4">
                    Conectando a la base de datos...
                </li>
            </ul>
        </div>
    </div>

    <!-- Script de Firebase y Lógica de la Aplicación -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, query, onSnapshot, addDoc, serverTimestamp, orderBy } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { getStorage, ref, uploadBytesResumable, getDownloadURL } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-storage.js";

        // **************** CONFIGURACIÓN INICIAL (¡REEMPLAZA ESTO!) ****************
        // 1. Reemplaza el objeto de configuración con el tuyo de la Consola de Firebase.
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        // ************************************************************************

        // Inicializar Firebase
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);
        const storage = getStorage(app);

        // Referencias del DOM
        const fileInput = document.getElementById('photo-upload');
        const fileList = document.getElementById('file-list');
        const initialMessage = document.getElementById('initial-message');
        const statusMessage = document.getElementById('status-message');
        const authStatus = document.getElementById('auth-status');

        let userId = null;
        const COLLECTION_PATH = 'public_photos';

        /**
         * Muestra mensajes de estado en la caja de carga.
         * @param {string} message - El mensaje a mostrar.
         * @param {boolean} isLoading - Si es un estado de carga (mostrar spinner).
         */
        function updateStatus(message, isLoading = false) {
            statusMessage.classList.remove('hidden', 'bg-red-100', 'text-red-700', 'bg-yellow-100', 'text-yellow-700', 'bg-green-100', 'text-green-700');
            statusMessage.classList.add('flex');
            statusMessage.innerHTML = isLoading 
                ? `<svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-yellow-700" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg><span>${message}</span>`
                : `<span>${message}</span>`;
            
            if (!isLoading) {
                 statusMessage.classList.add('bg-green-100', 'text-green-700');
                 setTimeout(() => statusMessage.classList.add('hidden'), 3000); // Ocultar después de 3s
            }
        }

        /**
         * Inicializa la autenticación del usuario.
         */
        async function initializeAuth() {
            try {
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Error en la autenticación:", error);
                updateStatus(`Error de autenticación: ${error.message}`, false);
            }
        }

        /**
         * Configura el listener de Firestore para la lista de archivos.
         */
        function setupFirestoreListener() {
            const path = `/artifacts/${appId}/public/data/${COLLECTION_PATH}`;
            const filesQuery = collection(db, path);
            
            // Usar onSnapshot para obtener actualizaciones en tiempo real
            onSnapshot(filesQuery, (snapshot) => {
                const filesData = [];
                snapshot.forEach(doc => {
                    const data = doc.data();
                    filesData.push({ 
                        id: doc.id,
                        ...data,
                        timestamp: data.timestamp ? data.timestamp.toDate() : new Date(0) // Manejar el timestamp
                    });
                });
                
                // Ordenar en memoria por fecha descendente
                filesData.sort((a, b) => b.timestamp - a.timestamp);

                renderFileList(filesData);
            }, (error) => {
                console.error("Error al escuchar Firestore:", error);
                updateStatus(`Error al cargar la lista: ${error.message}`, false);
            });
        }

        /**
         * Renderiza la lista de archivos.
         * @param {Array<Object>} files - Array de objetos de archivo con nombre y url.
         */
        function renderFileList(files) {
            fileList.innerHTML = '';
            if (files.length === 0) {
                initialMessage.classList.remove('hidden');
                initialMessage.textContent = 'Ninguna foto cargada aún. ¡Sube una para empezar!';
            } else {
                initialMessage.classList.add('hidden');
                files.forEach(file => appendToFileList(file.name, file.url, file.id));
            }
        }

        /**
         * Añade un elemento visual a la lista de descargas.
         */
        function appendToFileList(fileName, fileURL, docId) {
            const listItem = document.createElement('li');
            listItem.id = `file-${docId}`;
            listItem.className = 'flex items-center justify-between p-3 bg-gray-50 rounded-md hover:bg-gray-100 transition';
            
            const fileExtension = fileName.split('.').pop().toUpperCase();

            listItem.innerHTML = `
                <div class="flex items-center flex-grow min-w-0">
                    <span class="text-xs font-semibold px-2 py-1 mr-3 rounded-full bg-indigo-200 text-indigo-800">${fileExtension}</span>
                    <span class="text-sm font-medium text-gray-700 truncate mr-4">
                        ${fileName}
                    </span>
                </div>
                <a href="${fileURL}" download="${fileName}" target="_blank" class="flex-shrink-0 px-3 py-1 text-xs font-semibold rounded-full text-white bg-blue-500 hover:bg-blue-600 transition duration-150 download-link">
                    Descargar
                </a>
            `;
            fileList.appendChild(listItem);
        }

        /**
         * Sube el archivo a Firebase Storage y guarda la referencia en Firestore.
         */
        async function uploadFile(file) {
            const uniqueFileName = `${Date.now()}_${file.name}`;
            const storageRef = ref(storage, `${userId}/${uniqueFileName}`);
            
            // Subir archivo
            const uploadTask = uploadBytesResumable(storageRef, file);

            uploadTask.on('state_changed', 
                (snapshot) => {
                    const progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
                    updateStatus(`Subiendo ${file.name}: ${progress.toFixed(0)}%`, true);
                }, 
                (error) => {
                    console.error("Error al subir archivo:", error);
                    updateStatus(`Error al subir ${file.name}: ${error.message}`, false);
                }, 
                async () => {
                    // Subida completada: Obtener URL de descarga y guardar en Firestore
                    const downloadURL = await getDownloadURL(uploadTask.snapshot.ref);
                    
                    const docPath = `/artifacts/${appId}/public/data/${COLLECTION_PATH}`;
                    await addDoc(collection(db, docPath), {
                        name: file.name,
                        url: downloadURL,
                        uploadedBy: userId,
                        timestamp: serverTimestamp()
                    });

                    updateStatus(`¡${file.name} subido y listo para descargar!`, false);
                }
            );
        }

        // ----------------------------------------------------
        // LÓGICA PRINCIPAL
        // ----------------------------------------------------

        // 1. Esperar el estado de autenticación
        onAuthStateChanged(auth, (user) => {
            if (user) {
                userId = user.uid;
                authStatus.textContent = `Usuario autenticado (ID): ${userId}. ¡Comparte esta página!`;
                
                // 2. Ocultar el mensaje de "Conectando..."
                statusMessage.classList.add('hidden');

                // 3. Iniciar el listener de Firestore para la lista de fotos
                setupFirestoreListener();

            } else {
                // Si no hay usuario, iniciar sesión anónima
                initializeAuth();
            }
        });


        // 4. Manejar la selección de archivos
        fileInput.addEventListener('change', async (event) => {
            const files = event.target.files;
            if (files.length === 0 || !userId) return;

            // Mostrar estado de procesamiento
            updateStatus(`Procesando ${files.length} archivo(s)...`, true);

            // Subir cada archivo secuencialmente (para una mejor retroalimentación)
            for (const file of Array.from(files)) {
                await uploadFile(file);
            }
            // Limpiar el input para permitir volver a subir el mismo archivo
            fileInput.value = '';
        });

    </script>

</body>
</html>
