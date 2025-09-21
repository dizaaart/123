# 123


<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Визуализатор изображений</title>
  <style>
    body {
      font-family: 'Inter', sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      margin-top: 20px;
      padding: 20px;
      background-color: #1a1a1a;
      color: #ffffff;
      min-height: 100vh;
      box-sizing: border-box;
    }

    .main-wrapper {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      width: 100%;
      max-width: 1400px;
    }

    .image-container {
      display: flex;
      flex-wrap: wrap;
      gap: 20px;
      margin-top: 5px;
      justify-content: center;
    }

    .image-wrapper {
      position: relative;
      text-align: center;
      border: 1px solid #595959;
      padding: 10px;
      border-radius: 8px;
      transition: transform 0.2s ease-in-out;
    }

    .image-wrapper.drag-over {
      box-shadow: 0 0 10px 3px #fff;
    }

    .uploaded-image {
      max-width: 200px;
      max-height: 200px;
      border-radius: 4px;
    }

    .image-info {
      font-size: 12px;
      color: #ccc;
      margin-top: 5px;
    }

    .delete-btn {
      position: absolute;
      top: -10px;
      right: -10px;
      background-color: #dc3545;
      color: white;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      width: 24px;
      height: 24px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 16px;
      line-height: 1;
      padding: 0;
    }

    #drop-zone {
      width: 100%;
      height: 100px;
      border: 2px dashed #888;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      margin-top: 20px;
      color: #aaa;
      text-align: center;
      cursor: pointer;
      border-radius: 8px;
    }

    #drop-zone.dragover {
      border-color: #fff;
      color: #fff;
    }

    button {
      margin-top: 15px;
      padding: 10px 10px;
      background-color: #384354;
      color: #ffffff;
      border: none;
      cursor: pointer;
      border-radius: 6px;
    }

    button:hover {
      background-color: #475467;
    }
  </style>
</head>
<body>

  <div class="main-wrapper">
    <div class="image-container" id="image-container"></div>
    <div id="drop-zone">Перетащите сюда изображение для своей коллекции или нажмите, чтобы выбрать</div>
    <button id="reset-button">Сбросить изображения</button>
  </div>

  <script>
    // Настройки для IndexedDB
    const DB_NAME = 'image-visualizer-db';
    const STORE_NAME = 'images';
    const DB_VERSION = 1;

    // Глобальные переменные для перетаскивания
    let draggedItem = null;
    let draggedItemId = null;
    
    // Вспомогательная функция для открытия базы данных
    function openDb() {
      return new Promise((resolve, reject) => {
        const req = indexedDB.open(DB_NAME, DB_VERSION);
        req.onupgradeneeded = (e) => {
          const db = e.target.result;
          if (!db.objectStoreNames.contains(STORE_NAME)) {
            // Создаем хранилище с автоинкрементным ключом 'id'
            db.createObjectStore(STORE_NAME, { keyPath: 'id', autoIncrement: true });
          }
        };
        req.onsuccess = () => resolve(req.result);
        req.onerror = () => reject(req.error);
      });
    }

    // Функция для добавления изображения в базу данных
    async function addImage(file) {
      const db = await openDb();
      return new Promise((resolve, reject) => {
        const tx = db.transaction(STORE_NAME, 'readwrite');
        const store = tx.objectStore(STORE_NAME);
        const item = {
          file: file,
          fileName: file.name,
          size: file.size,
          type: file.type,
          timestamp: Date.now(),
          sortOrder: Date.now()
        };
        const req = store.add(item);
        req.onsuccess = () => resolve({ ...item, id: req.result });
        req.onerror = () => reject(req.error);
      });
    }

    // Функция для получения всех изображений из базы данных
    async function getAllImages() {
      const db = await openDb();
      return new Promise((resolve, reject) => {
        const tx = db.transaction(STORE_NAME, 'readonly');
        const store = tx.objectStore(STORE_NAME);
        const req = store.getAll();
        req.onsuccess = () => resolve(req.result);
        req.onerror = () => reject(req.error);
      });
    }
    
    // Функция для обновления изображения в базе данных
    async function updateImage(item) {
      const db = await openDb();
      return new Promise((resolve, reject) => {
        const tx = db.transaction(STORE_NAME, 'readwrite');
        const store = tx.objectStore(STORE_NAME);
        const req = store.put(item);
        req.onsuccess = () => resolve();
        req.onerror = () => reject(req.error);
      });
    }

    // Функция для удаления изображения из базы данных
    async function deleteImage(id) {
      const db = await openDb();
      return new Promise((resolve, reject) => {
        const tx = db.transaction(STORE_NAME, 'readwrite');
        const store = tx.objectStore(STORE_NAME);
        const req = store.delete(id);
        req.onsuccess = () => resolve();
        req.onerror = () => reject(req.error);
      });
    }

    // Функция для очистки всей базы данных
    async function clearAll() {
      const db = await openDb();
      return new Promise((resolve, reject) => {
        const tx = db.transaction(STORE_NAME, 'readwrite');
        const store = tx.objectStore(STORE_NAME);
        const req = store.clear();
        req.onsuccess = () => resolve();
        req.onerror = () => reject(req.error);
      });
    }

    const dropZone = document.getElementById('drop-zone');
    const imageContainer = document.getElementById('image-container');
    const resetButton = document.getElementById('reset-button');

    // Функция для создания и добавления одного элемента изображения в DOM
    function addImageElementToDOM(item) {
      const imageWrapper = document.createElement('div');
      imageWrapper.classList.add('image-wrapper');
      imageWrapper.draggable = true;
      imageWrapper.dataset.id = item.id;
      
      const img = document.createElement('img');
      const url = URL.createObjectURL(item.file);
      img.src = url;
      img.classList.add('uploaded-image');
      
      const title = document.createElement('div');
      title.classList.add('image-info');
      title.textContent = item.fileName;

      const resInfo = document.createElement('div');
      resInfo.classList.add('image-info');

      // После загрузки изображения получаем его размеры
      img.onload = () => {
        const width = img.naturalWidth;
        const height = img.naturalHeight;
        resInfo.textContent = `${width}×${height}px`;
        URL.revokeObjectURL(url); // Очищаем URL, чтобы освободить память
      };
      
      const sizeInfo = document.createElement('div');
      sizeInfo.classList.add('image-info');
      let sizeText;
      if (item.size / 1024 > 1020) {
        sizeText = (item.size / (1024 * 1024)).toFixed(2) + ' MB';
      } else {
        sizeText = (item.size / 1024).toFixed(2) + ' KB';
      }
      sizeInfo.textContent = sizeText;

      const deleteBtn = document.createElement('button');
      deleteBtn.classList.add('delete-btn');
      deleteBtn.innerHTML = '&times;';
      deleteBtn.addEventListener('click', async () => {
        await deleteImage(item.id);
        imageWrapper.remove(); // Удаляем элемент из DOM, не перерисовывая всю галерею
      });

      imageWrapper.appendChild(img);
      imageWrapper.appendChild(title);
      imageWrapper.appendChild(resInfo);
      imageWrapper.appendChild(sizeInfo);
      imageWrapper.appendChild(deleteBtn);
      imageContainer.appendChild(imageWrapper);
    }

    // Функция для первоначальной загрузки и рендеринга галереи
    async function renderInitialGallery() {
      const items = await getAllImages();
      // Сортируем по sortOrder, затем по timestamp для новых элементов
      items.sort((a, b) => (a.sortOrder || a.timestamp) - (b.sortOrder || b.timestamp)); 
      for (const item of items) {
        addImageElementToDOM(item);
      }
    }

    // Обработчик перетаскивания
    imageContainer.addEventListener('dragstart', (e) => {
      // Разрешаем перетаскивание только с зажатой клавишей Ctrl
      if (!e.ctrlKey) {
        e.preventDefault();
        return;
      }
      draggedItem = e.target.closest('.image-wrapper');
      draggedItemId = parseInt(draggedItem.dataset.id);
      e.dataTransfer.effectAllowed = 'move';
      e.dataTransfer.setData('text/plain', '');
      setTimeout(() => {
        draggedItem.style.opacity = '0.5';
      }, 0);
    });

    imageContainer.addEventListener('dragover', (e) => {
      e.preventDefault();
      const target = e.target.closest('.image-wrapper');
      if (e.ctrlKey && target && target !== draggedItem) {
        target.classList.add('drag-over');
      }
    });
    
    imageContainer.addEventListener('dragleave', (e) => {
      const target = e.target.closest('.image-wrapper');
      if (target) {
        target.classList.remove('drag-over');
      }
    });

    imageContainer.addEventListener('drop', async (e) => {
      e.preventDefault();
      const droppedOnItem = e.target.closest('.image-wrapper');
      if (!e.ctrlKey || !draggedItem || !droppedOnItem || draggedItem === droppedOnItem) {
        if (droppedOnItem) {
          droppedOnItem.classList.remove('drag-over');
        }
        return;
      }

      droppedOnItem.classList.remove('drag-over');
      
      const items = Array.from(imageContainer.children);
      const draggedIndex = items.indexOf(draggedItem);
      const droppedIndex = items.indexOf(droppedOnItem);

      if (draggedIndex < droppedIndex) {
        imageContainer.insertBefore(draggedItem, droppedOnItem.nextSibling);
      } else {
        imageContainer.insertBefore(draggedItem, droppedOnItem);
      }
      
      // Обновляем порядок сортировки в IndexedDB
      const db = await openDb();
      const tx = db.transaction(STORE_NAME, 'readwrite');
      const store = tx.objectStore(STORE_NAME);

      const newOrder = Array.from(imageContainer.children).map(el => parseInt(el.dataset.id));
      const oldItems = await getAllImages();

      for (const id of newOrder) {
        const oldItem = oldItems.find(item => item.id === id);
        if (oldItem) {
          oldItem.sortOrder = newOrder.indexOf(id);
          store.put(oldItem);
        }
      }

      tx.oncomplete = () => {
        console.log('Порядок успешно обновлен в базе данных.');
      };
      
      tx.onerror = (event) => {
        console.error('Ошибка при обновлении порядка:', event.target.error);
      };
      
      // Не вызываем renderGallery(), так как DOM уже обновлен
    });

    imageContainer.addEventListener('dragend', () => {
      if (draggedItem) {
        draggedItem.style.opacity = '1';
        draggedItem = null;
        draggedItemId = null;
      }
    });

    // Функция для обработки файлов, добавленных с помощью drag-and-drop или выбора
    function handleFiles(files) {
      [...files].forEach(async (file) => {
        if (file.type.startsWith('image/')) {
          const addedItem = await addImage(file);
          addImageElementToDOM(addedItem);
        }
      });
    }

    dropZone.addEventListener('dragover', (e) => {
      e.preventDefault();
      dropZone.classList.add('dragover');
    });

    dropZone.addEventListener('dragleave', () => {
      dropZone.classList.remove('dragover');
    });

    dropZone.addEventListener('drop', (e) => {
      e.preventDefault();
      dropZone.classList.remove('dragover');
      const files = e.dataTransfer.files;
      handleFiles(files);
    });

    dropZone.addEventListener('click', () => {
      const input = document.createElement('input');
      input.type = 'file';
      input.accept = 'image/*';
      input.multiple = true;
      input.addEventListener('change', (e) => handleFiles(e.target.files));
      input.click();
    });

    document.addEventListener('paste', (e) => {
      const clipboardItems = e.clipboardData.items;
      for (let i = 0; i < clipboardItems.length; i++) {
        const item = clipboardItems[i];
        if (item.type.startsWith('image/')) {
          const blob = item.getAsFile();
          handleFiles([blob]);
          break;
        }
      }
    });

    resetButton.addEventListener('click', async () => {
      await clearAll();
      imageContainer.innerHTML = ''; // Для сброса можно очистить все
    });

    // Инициализация при загрузке страницы
    window.addEventListener('DOMContentLoaded', async () => {
      try {
        await openDb();
        renderInitialGallery();
      } catch (err) {
        console.error('Ошибка IndexedDB:', err);
        dropZone.textContent = 'Ошибка: IndexedDB недоступен в этом браузере.';
        dropZone.style.color = 'crimson';
        resetButton.style.display = 'none';
      }
    });
  </script>

</body>
</html>

___



<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Инструмент Сравнения и Сжатия Изображений</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: Arial, sans-serif; background-color: #1c1c1c; color: white; padding: 0; overflow-y: auto; }
    .container { width: 100%; padding: 20px; background-color: #2a2a2a; border-radius: 10px; box-shadow: 0 0 20px rgba(0,0,0,0.5); }
    h1 { font-size: 20px; text-align: center; margin-bottom: 20px; }

    .upload-section { display: flex; gap: 10px; flex-wrap: wrap; }
    .upload-box { flex: 1; min-width: 200px; background-color: #3a3a3a; border: 2px dashed #4a4a4a; border-radius: 10px; padding: 10px; position: relative; transition: opacity 0.2s, border-style 0.2s; }
    .upload-box.can-drag { cursor: grab; }
    .upload-box.dragging { opacity: 0.7; border-style: solid; }
    .upload-box.over { border-style: solid; border-color: #fff; }
    .upload-box input[type=file] { display: none; }
    .upload-box img { max-width: 100%; max-height: 200px; border-radius: 5px; display: block; margin: 0 auto; }
    .upload-box p { margin-top: 10px; color: #a6a6a6; font-size: 14px; text-align: center; }
    .file-info { font-size: 12px; color: #888; margin-top: 4px; }

    .quality-control, .size-control, .format-control {
      margin-top: 10px;
      display: flex;
      align-items: center;
      gap: 8px;
      justify-content: center;
      color: #a6a6a6;
    }
    .quality-control input[type=range],
    .size-control input[type=range],
    .format-control select {
      width: 70%;
      background-color: #2a2a2a;
      color: white;
      border: 1px solid #777;
      border-radius: 4px;
      padding: 2px 4px;
    }

    .compare-section { margin-top: 20px; background-color: #4a4a4a; aspect-ratio: 16/9; border-radius: 10px; display: none; position: relative; overflow: hidden; }
    .zoom-container { width: 100%; height: 100%; position: relative; transform-origin: center center; transition: transform 0.1s; }
    .zoom-container img { width: 100%; height: 100%; object-fit: contain; position: absolute; top: 0; left: 0; pointer-events: none; }
    #compareImg1 { z-index: 3; clip-path: inset(0 67% 0 0); }
    #compareImg2 { z-index: 2; clip-path: inset(0 33% 0 33%); }
    #compareImg3 { z-index: 1; clip-path: inset(0 0 0 67%); }
    .slider { position: absolute; top: 0; bottom: 0; width: 1px; background-color: rgba(255,255,255,0.1); cursor: ew-resize; z-index: 10; transform: translateX(-50%); }

    .bg-color-picker { margin-top: 15px; text-align: center; display: flex; justify-content: center; align-items: center; gap: 10px; }
    .bg-color-picker input[type=color], .bg-color-picker input[type=text] { padding: 3px; border: 1px solid #777; border-radius: 4px; background-color: #2a2a2a; color: white; }
    .checker-button { width: 24px; height: 24px; border: 1px solid #888; cursor: pointer; background-image: linear-gradient(45deg,#ccc 25%,transparent 25%), linear-gradient(-45deg,#ccc 25%,transparent 25%), linear-gradient(45deg,transparent 75%,#ccc 75%), linear-gradient(-45deg,transparent 75%,#ccc 75%); background-size:12px 12px; background-position:0 0,0 6px,6px -6px,-6px 0; }
    .checker-button.active { border-color: white; }
    .checkerboard-bg { background-image: linear-gradient(45deg,#ccc 25%,transparent 25%), linear-gradient(-45deg,#ccc 25%,transparent 25%), linear-gradient(45deg,transparent 75%,#ccc 75%), linear-gradient(-45deg,transparent 75%,#ccc 75%); background-size:20px 20px; background-position:0 0,0 10px,10px -10px,-10px 0; }

    .clear-button { width: 100%; margin-top: 20px; padding: 10px; background-color: #5a5a5a; color: white; font-size: 14px; border: none; border-radius: 5px; cursor: pointer; }
    .clear-button:hover { background-color: #777; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Инструмент Сравнения и Сжатия Изображений</h1>
    <div class="upload-section">
      <label class="upload-box" id="uploadBox1" draggable="true" data-index="1">
        <input type="file" id="image1" accept="image/*">
        <img id="previewImg1" style="display:none;">
        <p id="fileName1">Выберите или перетащите изображение 1</p>
        <p id="fileInfo1" class="file-info"></p>
        <div class="quality-control">
          <label for="qualitySlider1">Качество:</label>
          <input type="range" id="qualitySlider1" min="0.1" max="1" step="0.05" value="1" disabled>
          <span id="qualityValue1">100%</span>
        </div>
        <div class="size-control">
          <label for="sizeSlider1">Размер (px):</label>
          <input type="range" id="sizeSlider1" min="0.1" max="1" step="0.05" value="1" disabled>
          <span id="sizeValue1"></span>
        </div>
        <div class="format-control">
          <label for="formatSelect1">Формат:</label>
          <select id="formatSelect1" disabled>
            <option value="auto">Авто</option>
            <option value="jpeg">JPEG</option>
            <option value="png">PNG</option>
            <option value="webp">WEBP</option>
          </select>
        </div>
      </label>
      <label class="upload-box" id="uploadBox2" draggable="true" data-index="2">
        <input type="file" id="image2" accept="image/*">
        <img id="previewImg2" style="display:none;">
        <p id="fileName2">Выберите или перетащите изображение 2</p>
        <p id="fileInfo2" class="file-info"></p>
        <div class="quality-control">
          <label for="qualitySlider2">Качество:</label>
          <input type="range" id="qualitySlider2" min="0.1" max="1" step="0.05" value="1" disabled>
          <span id="qualityValue2">100%</span>
        </div>
        <div class="size-control">
          <label for="sizeSlider2">Размер (px):</label>
          <input type="range" id="sizeSlider2" min="0.1" max="1" step="0.05" value="1" disabled>
          <span id="sizeValue2"></span>
        </div>
        <div class="format-control">
          <label for="formatSelect2">Формат:</label>
          <select id="formatSelect2" disabled>
            <option value="auto">Авто</option>
            <option value="jpeg">JPEG</option>
            <option value="png">PNG</option>
            <option value="webp">WEBP</option>
          </select>
        </div>
      </label>
      <label class="upload-box" id="uploadBox3" draggable="true" data-index="3">
        <input type="file" id="image3" accept="image/*">
        <img id="previewImg3" style="display:none;">
        <p id="fileName3">Выберите или перетащите изображение 3</p>
        <p id="fileInfo3" class="file-info"></p>
        <div class="quality-control">
          <label for="qualitySlider3">Качество:</label>
          <input type="range" id="qualitySlider3" min="0.1" max="1" step="0.05" value="1" disabled>
          <span id="qualityValue3">100%</span>
        </div>
        <div class="size-control">
          <label for="sizeSlider3">Размер (px):</label>
          <input type="range" id="sizeSlider3" min="0.1" max="1" step="0.05" value="1" disabled>
          <span id="sizeValue3"></span>
        </div>
        <div class="format-control">
          <label for="formatSelect3">Формат:</label>
          <select id="formatSelect3" disabled>
            <option value="auto">Авто</option>
            <option value="jpeg">JPEG</option>
            <option value="png">PNG</option>
            <option value="webp">WEBP</option>
          </select>
        </div>
      </label>
    </div>
    <div class="compare-section" id="compareSection">
      <div class="zoom-container" id="zoomContainer">
        <img id="compareImg1">
        <img id="compareImg2">
        <img id="compareImg3">
        <div class="slider" id="slider1"></div>
        <div class="slider" id="slider2"></div>
      </div>
    </div>

    <div class="bg-color-picker">
      <label for="bgColorPicker">Цвет фона:</label>
      <input type="color" id="bgColorPicker" value="#b3b3b3">
      <input type="text" id="hexInput" placeholder="Hex" maxlength="7">
      <span id="checkerToggle" class="checker-button" title="Шахматный фон"></span>
    </div>
    <button class="clear-button" onclick="clearImages()">Очистить</button>
  </div>

  <script>
    let imgData = {1: null, 2: null, 3: null};
    let imgMeta = {1: {}, 2: {}, 3: {}};
    let originalFileData = {1: null, 2: null, 3: null};

    const qualitySliders = {
      1: document.getElementById('qualitySlider1'),
      2: document.getElementById('qualitySlider2'),
      3: document.getElementById('qualitySlider3')
    };
    const qualityValues = {
      1: document.getElementById('qualityValue1'),
      2: document.getElementById('qualityValue2'),
      3: document.getElementById('qualityValue3')
    };
    const sizeSliders = {
      1: document.getElementById('sizeSlider1'),
      2: document.getElementById('sizeSlider2'),
      3: document.getElementById('sizeSlider3')
    };
    const sizeValues = {
      1: document.getElementById('sizeValue1'),
      2: document.getElementById('sizeValue2'),
      3: document.getElementById('sizeValue3')
    };
    const formatSelects = {
      1: document.getElementById('formatSelect1'),
      2: document.getElementById('formatSelect2'),
      3: document.getElementById('formatSelect3')
    };

    function updateSizeDisplay(idx) {
      const meta = imgMeta[idx];
      if (!meta.width) return;
      const fmt = formatSelects[idx].value;
      if (fmt === 'auto' && originalFileData[idx]) {
        sizeValues[idx].textContent = `${originalFileData[idx].width}×${originalFileData[idx].height}`;
        qualitySliders[idx].value = 1;
        sizeSliders[idx].value = 1;
        qualityValues[idx].textContent = '100%';
        qualitySliders[idx].disabled = true;
        sizeSliders[idx].disabled = true;
      } else {
        const ratio = parseFloat(sizeSliders[idx].value);
        sizeValues[idx].textContent = `${Math.round(meta.width * ratio)}×${Math.round(meta.height * ratio)}`;
        qualitySliders[idx].disabled = false;
        sizeSliders[idx].disabled = false;
      }
    }

    function compressAndUpdate(idx) {
      const quality = parseFloat(qualitySliders[idx].value);
      const ratio   = parseFloat(sizeSliders[idx].value);
      const fmt     = formatSelects[idx].value;
      qualityValues[idx].textContent = Math.round(quality * 100) + '%';
      updateSizeDisplay(idx);

      const src = imgData[idx];
      if (!src) return;

      if (fmt === 'auto' && originalFileData[idx]) {
        document.getElementById(`previewImg${idx}`).src = originalFileData[idx].dataUrl;
        document.getElementById(`previewImg${idx}`).style.display = 'block';
        document.getElementById(`compareImg${idx}`).src = originalFileData[idx].dataUrl;
        document.getElementById(`fileName${idx}`).textContent = originalFileData[idx].name;
        document.getElementById(`fileInfo${idx}`).textContent = `${originalFileData[idx].width}×${originalFileData[idx].height}, ${Math.round(originalFileData[idx].size/1024)} KB`;
      } else {
        const img = new Image();
        img.onload = () => {
          const canvas = document.createElement('canvas');
          canvas.width  = Math.round(img.width * ratio);
          canvas.height = Math.round(img.height * ratio);
          canvas.getContext('2d').drawImage(img, 0, 0, canvas.width, canvas.height);

          let mime = fmt === 'png' ? 'image/png' : fmt === 'webp' ? 'image/webp' : 'image/jpeg';
          if (fmt === 'auto') {
            const t = imgMeta[idx].type || '';
            mime = /(png|webp|gif)$/.test(t) ? 'image/webp' : 'image/jpeg';
          }

          const dataUrl = canvas.toDataURL(mime, quality);
          document.getElementById(`previewImg${idx}`).src = dataUrl;
          document.getElementById(`previewImg${idx}`).style.display = 'block';
          document.getElementById(`compareImg${idx}`).src = dataUrl;

          const kb = Math.round((dataUrl.length * 3/4) / 1024);
          let displayName = imgMeta[idx].name;
          if (fmt !== 'auto') {
            const nameWithoutExt = imgMeta[idx].name.replace(/\.[^/.]+$/, '');
            const ext = mime.split('/')[1];
            displayName = `${nameWithoutExt}.${ext}`;
          }
          document.getElementById(`fileName${idx}`).textContent = displayName;
          document.getElementById(`fileInfo${idx}`).textContent = `${Math.round(img.width*ratio)}×${Math.round(img.height*ratio)}, ${kb} KB`;
        };
        img.src = src;
      }

      const hasImages = imgData[1] || imgData[2] || imgData[3];
      if (hasImages) {
        document.getElementById('compareSection').style.display = 'flex';
        initializeDoubleSlider();
      } else {
        document.getElementById('compareSection').style.display = 'none';
      }
    }

    [1, 2, 3].forEach(i => {
      const input = document.getElementById(`image${i}`);
      const box = document.getElementById(`uploadBox${i}`);

      input.addEventListener('change', e => {
        const file = e.target.files[0];
        if (!file) return;
        readFile(file, i);
      });
      box.addEventListener('dragover', e => e.preventDefault());
      box.addEventListener('drop', e => {
        e.preventDefault();
        const file = e.dataTransfer.files[0];
        if (!file) return;
        readFile(file, i);
      });
      qualitySliders[i].addEventListener('input', () => compressAndUpdate(i));
      sizeSliders[i].addEventListener('input', () => compressAndUpdate(i));
      formatSelects[i].addEventListener('change', () => compressAndUpdate(i));
    });

    function readFile(file, idx) {
      const reader = new FileReader();
      reader.onload = ev => {
        imgData[idx] = ev.target.result;
        const tempImg = new Image();
        tempImg.onload = () => {
          imgMeta[idx] = { name: file.name, width: tempImg.width, height: tempImg.height, type: file.type };
          originalFileData[idx] = {
            dataUrl: ev.target.result,
            name: file.name,
            width: tempImg.width,
            height: tempImg.height,
            size: file.size
          };
          formatSelects[idx].value = 'auto';
          formatSelects[idx].disabled = false;
          compressAndUpdate(idx);
        };
        tempImg.src = ev.target.result;
      };
      reader.readAsDataURL(file);
    }
    
    // Drag-and-drop functionality for reordering
    let draggedIndex = null;
    const uploadBoxes = document.querySelectorAll('.upload-box');

    // Add and remove 'can-drag' class based on Ctrl key
    document.addEventListener('keydown', e => {
        if (e.key === 'Control') {
            uploadBoxes.forEach(box => box.classList.add('can-drag'));
        }
    });

    document.addEventListener('keyup', e => {
        if (e.key === 'Control') {
            uploadBoxes.forEach(box => box.classList.remove('can-drag'));
        }
    });

    uploadBoxes.forEach(box => {
      box.addEventListener('dragstart', e => {
        // Проверка на зажатую клавишу Ctrl для разрешения перетаскивания
        if (!e.ctrlKey) {
            e.preventDefault();
            return;
        }
        draggedIndex = e.target.closest('.upload-box').dataset.index;
        e.target.closest('.upload-box').classList.add('dragging');
      });
      
      box.addEventListener('dragover', e => {
        e.preventDefault();
        const dropTarget = e.target.closest('.upload-box');
        if (dropTarget && dropTarget.dataset.index !== draggedIndex) {
          dropTarget.classList.add('over');
        }
      });
      
      box.addEventListener('dragleave', e => {
        const dropTarget = e.target.closest('.upload-box');
        if (dropTarget) {
          dropTarget.classList.remove('over');
        }
      });
      
      box.addEventListener('drop', e => {
        e.preventDefault();
        const dropTarget = e.target.closest('.upload-box');
        if (draggedIndex && dropTarget && draggedIndex !== dropTarget.dataset.index) {
          const dropIndex = dropTarget.dataset.index;
          
          const tempImgData = imgData[draggedIndex];
          const tempImgMeta = imgMeta[draggedIndex];
          const tempOriginalData = originalFileData[draggedIndex];
          
          imgData[draggedIndex] = imgData[dropIndex];
          imgMeta[draggedIndex] = imgMeta[dropIndex];
          originalFileData[draggedIndex] = originalFileData[dropIndex];
          
          imgData[dropIndex] = tempImgData;
          imgMeta[dropIndex] = tempImgMeta;
          originalFileData[dropIndex] = tempOriginalData;
          
          compressAndUpdate(draggedIndex);
          compressAndUpdate(dropIndex);
          
          document.getElementById('compareImg1').src = imgData[1] || '';
          document.getElementById('compareImg2').src = imgData[2] || '';
          document.getElementById('compareImg3').src = imgData[3] || '';
        }
        draggedIndex = null;
        if (dropTarget) dropTarget.classList.remove('over');
        document.querySelectorAll('.upload-box').forEach(el => el.classList.remove('dragging'));
      });
      
      box.addEventListener('dragend', e => {
        e.target.classList.remove('dragging');
        uploadBoxes.forEach(box => box.classList.remove('can-drag'));
      });
    });
    
    // Global variables for slider positions
    let percent1 = 33;
    let percent2 = 66;
    const minSeparation = 0;

    // Updates the clip paths based on slider percentages
    function updateClips() {
      const slider1 = document.getElementById('slider1');
      const slider2 = document.getElementById('slider2');
      const img1 = document.getElementById('compareImg1');
      const img2 = document.getElementById('compareImg2');
      const img3 = document.getElementById('compareImg3');
      
      slider1.style.left = percent1 + '%';
      slider2.style.left = percent2 + '%';
      
      img1.style.clipPath = `inset(0 ${100 - percent1}% 0 0)`;
      img2.style.clipPath = `inset(0 ${100 - percent2}% 0 ${percent1}%)`;
      img3.style.clipPath = `inset(0 0 0 ${percent2}%)`;
    }

    function initializeDoubleSlider() {
      updateClips();
    }
    
    // Panning and zoom variables
    const compareSection = document.getElementById('compareSection');
    let scale = 1, panX = 0, panY = 0, isPanning = false, isDraggingSlider = false, startX, startY, sliderToMove;
    
    compareSection.addEventListener('mousedown', e => {
      e.preventDefault();
      
      if (e.button === 0) {
        const container = document.getElementById('zoomContainer');
        const rect = container.getBoundingClientRect();
        const clickX = e.clientX - rect.left;
        const clickPercent = (clickX / rect.width) * 100;
        
        const slider1Pos = parseFloat(document.getElementById('slider1').style.left);
        const slider2Pos = parseFloat(document.getElementById('slider2').style.left);
        
        const dist1 = Math.abs(clickPercent - slider1Pos);
        const dist2 = Math.abs(clickPercent - slider2Pos);
        
        if (dist1 <= dist2) {
          sliderToMove = 'slider1';
        } else {
          sliderToMove = 'slider2';
        }
        
        isDraggingSlider = true;
      }
      
      else if (e.button === 1) {
        isPanning = true;
        startX = e.clientX;
        startY = e.clientY;
        document.body.style.userSelect = 'none';
      }
    });

    document.addEventListener('mouseup', e => {
      if (e.button === 0 && isDraggingSlider) {
        isDraggingSlider = false;
        sliderToMove = null;
      }
      if (e.button === 1) {
        isPanning = false;
        document.body.style.userSelect = '';
      }
    });

    document.addEventListener('mousemove', e => {
      if (isDraggingSlider) {
        e.preventDefault();
        const container = document.getElementById('zoomContainer');
        const rect = container.getBoundingClientRect();
        const newClickX = e.clientX - rect.left;
        let newClickPercent = (newClickX / rect.width) * 100;

        if (sliderToMove === 'slider1') {
          // Check for collision with slider2
          if (newClickPercent > percent2 - minSeparation) {
            percent1 = Math.max(0, percent2 - minSeparation);
            percent2 = Math.min(100, newClickPercent + minSeparation);
          } else {
            percent1 = Math.max(0, Math.min(100, newClickPercent));
          }
        } else if (sliderToMove === 'slider2') {
          // Check for collision with slider1
          if (newClickPercent < percent1 + minSeparation) {
            percent2 = Math.min(100, percent1 + minSeparation);
            percent1 = Math.max(0, newClickPercent - minSeparation);
          } else {
            percent2 = Math.max(0, Math.min(100, newClickPercent));
          }
        }
        
        updateClips();
      } else if (isPanning) {
        e.preventDefault();
        panX += e.clientX - startX;
        panY += e.clientY - startY;
        startX = e.clientX;
        startY = e.clientY;
        document.getElementById('zoomContainer').style.transform = `translate(${panX}px, ${panY}px) scale(${scale})`;
      }
    });
    
    document.addEventListener('keydown', e => {
      if (e.key === 'Home') {
        scale = 1; panX = 0; panY = 0;
        document.getElementById('zoomContainer').style.transform = `translate(0,0) scale(1)`;
        updateClips();
      }
    });

    function clearImages() {
      [1, 2, 3].forEach(i => {
        imgData[i] = null;
        imgMeta[i] = {};
        originalFileData[i] = null;
        document.getElementById(`previewImg${i}`).style.display = 'none';
        document.getElementById(`fileName${i}`).textContent = `Выберите или перетащите изображение ${i}`;
        document.getElementById(`fileInfo${i}`).textContent = '';
        qualitySliders[i].value = 1;
        sizeSliders[i].value = 1;
        formatSelects[i].value = 'auto';
        qualityValues[i].textContent = '100%';
        sizeValues[i].textContent = '';
        qualitySliders[i].disabled = true;
        sizeSliders[i].disabled = true;
        formatSelects[i].disabled = true;
      });
      document.getElementById('compareSection').style.display = 'none';
      document.getElementById('zoomContainer').style.transform = 'translate(0,0) scale(1)';
      percent1 = 33;
      percent2 = 66;
      updateClips();
    }

    const bgPicker = document.getElementById('bgColorPicker');
    const hexInput = document.getElementById('hexInput');
    const checker = document.getElementById('checkerToggle');
    let is_zooming = false;

    bgPicker.addEventListener('input', e => {
      const c = e.target.value;
      hexInput.value = c.substring(1);
      compareSection.style.backgroundColor = c;
      compareSection.classList.remove('checkerboard-bg');
      checker.classList.remove('active');
    });

    hexInput.addEventListener('input', () => {
      const v = hexInput.value.trim();
      if (/^[0-9A-Fa-f]{6}$/.test(v)) {
        const c = '#' + v;
        bgPicker.value = c;
        compareSection.style.backgroundColor = c;
        compareSection.classList.remove('checkerboard-bg');
        checker.classList.remove('active');
      }
    });

    checker.addEventListener('click', () => {
      checker.classList.toggle('active');
      compareSection.classList.toggle('checkerboard-bg');
      if (compareSection.classList.contains('checkerboard-bg')) {
        compareSection.style.backgroundColor = '';
      } else {
        compareSection.style.backgroundColor = bgPicker.value;
      }
    });

    document.addEventListener('wheel', e => {
      if (e.shiftKey) {
        e.preventDefault();
        scale += e.deltaY < 0 ? 0.1 : -0.1;
        scale = Math.max(0.1, scale);
        document.getElementById('zoomContainer').style.transform = `translate(${panX}px, ${panY}px) scale(${scale})`;
      }
    });
  </script>
</body>
</html>
