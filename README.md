<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>線上 PDF 合併與重排工具</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/pdf-lib/dist/pdf-lib.min.js"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700&display=swap" rel="stylesheet">
    <!-- Chosen Palette: Calm Neutrals with Tech Accents -->
    <!-- Application Structure Plan: 採用簡潔的單頁工具佈局。頂部為標題和簡要說明。核心區域分為兩部分：左側為檔案上傳/拖曳區，右側為已上傳 PDF 檔案的列表，支援拖曳排序。底部為操作按鈕（合併與下載）和狀態訊息區。新增模態視窗用於單一 PDF 內部頁面排序，提供更精細的控制。 -->
    <!-- Visualization & Content Choices: 使用 HTML 元素（如 div, input, ul, li）來構建檔案上傳區和列表，確保響應式佈局。透過 JavaScript 處理檔案讀取、列表的動態更新和拖曳排序功能。核心的 PDF 處理（合併與頁面重排）將使用 pdf-lib 函式庫在客戶端完成，確保數據安全與效率。狀態訊息將以文本形式顯示，提供操作反饋。新增頁面排序模態視窗，以文字標籤而非縮圖實現頁面拖曳排序，平衡功能與效能。 -->
    <!-- CONFIRMATION: NO SVG graphics used. NO Mermaid JS used. -->
    <style>
        body {
            font-family: 'Noto Sans TC', sans-serif;
            background-color: #FDFBF8;
            color: #4A4A4A;
        }
        .drop-zone {
            border: 2px dashed #B0B0B0;
            background-color: #FAFAFA;
            min-height: 150px;
            display: flex;
            align-items: center;
            justify-content: center;
            text-align: center;
            cursor: pointer;
            transition: all 0.2s ease;
        }
        .drop-zone.hover {
            background-color: #E0F2F7; /* Light blue on hover */
            border-color: #2196F3; /* Blue border on hover */
        }
        .file-item {
            background-color: #FFFFFF;
            border: 1px solid #E5E7EB;
            cursor: grab;
            transition: all 0.2s ease;
        }
        .file-item.dragging {
            opacity: 0.5;
            transform: scale(1.02);
        }
        .file-item:hover {
            box-shadow: 0 2px 8px rgba(0,0,0,0.08);
        }
        .button-primary {
            background-color: #A7727D;
            color: white;
            transition: background-color 0.2s ease;
        }
        .button-primary:hover {
            background-color: #8C5E6A;
        }
        .button-primary:disabled {
            background-color: #D1D5DB;
            cursor: not-allowed;
        }
        .loading-spinner {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #A7727D;
            border-radius: 50%;
            width: 24px;
            height: 24px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        /* Modal styles */
        .modal {
            background-color: rgba(0, 0, 0, 0.5);
            z-index: 1000;
        }
        .modal-content {
            max-height: 90vh;
            overflow-y: auto;
        }
        .page-item {
            background-color: #F8F8F8;
            border: 1px solid #E5E7EB;
            cursor: grab;
            transition: all 0.2s ease;
        }
        .page-item:hover {
            background-color: #F0F0F0;
        }
        .page-item.dragging {
            opacity: 0.5;
        }
    </style>
</head>
<body class="antialiased">
    <div class="container mx-auto p-4 sm:p-6 md:p-8 max-w-4xl">
        <header class="text-center mb-10">
            <h1 class="text-3xl sm:text-4xl md:text-5xl font-bold text-[#A7727D] mb-2">線上 PDF 合併與重排工具</h1>
            <p class="text-lg sm:text-xl text-gray-500">輕鬆合併多個 PDF 檔案並調整頁面順序</p>
        </header>

        <main class="bg-white rounded-xl shadow-lg p-6 sm:p-8">
            <section class="mb-8">
                <h2 class="text-2xl font-semibold mb-4 text-[#4A4A4A]">上傳或拖曳 PDF 檔案</h2>
                <div id="dropZone" class="drop-zone rounded-lg p-6 mb-4 text-gray-600">
                    <p>將 PDF 檔案拖曳到此處，或點擊選擇檔案</p>
                    <input type="file" id="pdfFileInput" accept="application/pdf" multiple class="hidden">
                </div>
                <p class="text-sm text-gray-500 text-center">請確保上傳的檔案是 PDF 格式 (.pdf)</p>
            </section>

            <section class="mb-8">
                <h2 class="text-2xl font-semibold mb-4 text-[#4A4A4A]">已上傳檔案與頁面順序</h2>
                <ul id="fileList" class="space-y-3">
                    <li class="text-gray-500 text-center py-4">目前沒有檔案。請上傳 PDF。</li>
                </ul>
                <p class="text-sm text-gray-500 mt-4 text-center">拖曳列表中的檔案可調整 PDF 檔案順序</p>
            </section>

            <section class="text-center">
                <button id="combineAndDownloadBtn" class="button-primary px-8 py-3 rounded-full font-bold text-lg shadow-md hover:shadow-lg transition-all flex items-center justify-center mx-auto" disabled>
                    <span id="combineButtonText">合併並下載 PDF</span>
                    <div id="combineLoadingSpinner" class="loading-spinner ml-3 hidden"></div>
                </button>
                <p id="statusMessage" class="text-sm text-gray-600 mt-4"></p>
            </section>
        </main>
    </div>

    <!-- Page Reorder Modal -->
    <div id="pageReorderModal" class="modal fixed inset-0 flex items-center justify-center hidden">
        <div class="modal-content bg-white rounded-lg p-8 shadow-xl w-full max-w-lg mx-4">
            <h3 class="text-xl font-bold mb-6 text-[#A7727D]" id="modalTitle">調整頁面順序</h3>
            <ul id="modalPageList" class="space-y-2 border border-gray-200 rounded-md p-2 mb-6">
                <!-- Page items will be rendered here -->
            </ul>
            <div class="flex justify-end gap-3">
                <button id="cancelPageReorder" class="px-6 py-2 rounded-full text-gray-700 bg-gray-200 hover:bg-gray-300 transition-all">取消</button>
                <button id="confirmPageReorder" class="button-primary px-6 py-2 rounded-full">確認</button>
            </div>
        </div>
    </div>

    <script>
        const { PDFDocument } = PDFLib;
        const dropZone = document.getElementById('dropZone');
        const pdfFileInput = document.getElementById('pdfFileInput');
        const fileList = document.getElementById('fileList');
        const combineAndDownloadBtn = document.getElementById('combineAndDownloadBtn');
        const combineButtonText = document.getElementById('combineButtonText');
        const combineLoadingSpinner = document.getElementById('combineLoadingSpinner');
        const statusMessage = document.getElementById('statusMessage');

        // Modal elements
        const pageReorderModal = document.getElementById('pageReorderModal');
        const modalTitle = document.getElementById('modalTitle');
        const modalPageList = document.getElementById('modalPageList');
        const cancelPageReorderBtn = document.getElementById('cancelPageReorder');
        const confirmPageReorderBtn = document.getElementById('confirmPageReorder');

        let pdfFiles = []; // Stores { id, name, file, pdfDoc, pageOrder, isLoading, error }
        let currentFileDragging = null; // For file reordering
        let currentPageDragging = null; // For page reordering within modal
        let currentPdfIndexInModal = -1; // Index of the PDF being edited in the modal

        // --- File Drag and Drop Handlers ---
        dropZone.addEventListener('dragover', (e) => {
            e.preventDefault();
            dropZone.classList.add('hover');
        });

        dropZone.addEventListener('dragleave', () => {
            dropZone.classList.remove('hover');
        });

        dropZone.addEventListener('drop', (e) => {
            e.preventDefault();
            dropZone.classList.remove('hover');
            handleFiles(e.dataTransfer.files);
        });

        dropZone.addEventListener('click', () => {
            pdfFileInput.click();
        });

        pdfFileInput.addEventListener('change', (e) => {
            handleFiles(e.target.files);
        });

        async function handleFiles(files) {
            statusMessage.textContent = '';
            statusMessage.style.color = 'inherit';

            for (const file of files) {
                if (file.type === 'application/pdf') {
                    const newPdfEntry = {
                        id: Date.now() + Math.random(), // Unique ID
                        name: file.name,
                        file: file,
                        pdfDoc: null,
                        pageOrder: [],
                        isLoading: true,
                        error: null
                    };
                    pdfFiles.push(newPdfEntry);
                    renderFileList(); // Render immediately with loading state

                    try {
                        const arrayBuffer = await file.arrayBuffer();
                        const pdfDoc = await PDFDocument.load(arrayBuffer);
                        newPdfEntry.pdfDoc = pdfDoc;
                        newPdfEntry.pageOrder = Array.from({ length: pdfDoc.getPageCount() }, (_, i) => i);
                        newPdfEntry.isLoading = false;
                    } catch (error) {
                        console.error(`載入 PDF "${file.name}" 失敗:`, error);
                        newPdfEntry.error = `載入失敗: ${error.message}`;
                        newPdfEntry.isLoading = false;
                        statusMessage.textContent = `檔案 "${file.name}" 載入失敗。`;
                        statusMessage.style.color = 'red';
                    }
                    renderFileList(); // Re-render after loading/error
                } else {
                    statusMessage.textContent = `檔案 "${file.name}" 不是 PDF 格式，已忽略。`;
                    statusMessage.style.color = 'red';
                }
            }
            updateButtonState();
        }

        function renderFileList() {
            fileList.innerHTML = ''; // Clear existing list
            if (pdfFiles.length === 0) {
                fileList.innerHTML = '<li class="text-gray-500 text-center py-4">目前沒有檔案。請上傳 PDF。</li>';
                return;
            }

            pdfFiles.forEach((entry, index) => {
                const li = document.createElement('li');
                li.className = 'file-item flex items-center justify-between p-4 rounded-lg shadow-sm mb-2 text-gray-700';
                li.draggable = true;
                li.dataset.id = entry.id; // Use unique ID for dragging

                let statusHtml = '';
                if (entry.isLoading) {
                    statusHtml = `<div class="loading-spinner ml-3"></div> <span class="ml-2 text-sm text-gray-500">載入中...</span>`;
                } else if (entry.error) {
                    statusHtml = `<span class="ml-2 text-sm text-red-500">${entry.error}</span>`;
                } else {
                    statusHtml = `<span class="ml-2 text-sm text-gray-500">${entry.pdfDoc.getPageCount()} 頁</span>`;
                }

                li.innerHTML = `
                    <span class="flex-grow truncate">${entry.name} ${statusHtml}</span>
                    <div class="flex items-center space-x-2 ml-4">
                        <button class="px-3 py-1 rounded-full bg-blue-500 text-white text-sm hover:bg-blue-600 focus:outline-none adjust-pages-btn" data-id="${entry.id}" ${entry.isLoading || entry.error ? 'disabled' : ''}>調整頁面</button>
                        <button class="text-red-500 hover:text-red-700 focus:outline-none remove-file-btn" data-id="${entry.id}">移除</button>
                    </div>
                `;
                
                // Add remove functionality
                li.querySelector('.remove-file-btn').addEventListener('click', (e) => {
                    const removeId = parseFloat(e.target.dataset.id);
                    pdfFiles = pdfFiles.filter(f => f.id !== removeId);
                    renderFileList();
                    updateButtonState();
                });

                // Add adjust pages functionality
                li.querySelector('.adjust-pages-btn').addEventListener('click', (e) => {
                    const adjustId = parseFloat(e.target.dataset.id);
                    const index = pdfFiles.findIndex(f => f.id === adjustId);
                    if (index !== -1 && pdfFiles[index].pdfDoc) {
                        openPageReorderModal(index);
                    }
                });

                // File Drag events
                li.addEventListener('dragstart', (e) => {
                    currentFileDragging = li;
                    e.dataTransfer.effectAllowed = 'move';
                    e.dataTransfer.setData('text/plain', entry.id); // Set data for drop
                    li.classList.add('dragging');
                });

                li.addEventListener('dragend', () => {
                    currentFileDragging.classList.remove('dragging');
                    currentFileDragging = null;
                });

                li.addEventListener('dragover', (e) => {
                    e.preventDefault();
                    const boundingBox = li.getBoundingClientRect();
                    const offset = e.clientY - boundingBox.top;
                    if (currentFileDragging !== li) {
                        if (offset > boundingBox.height / 2) {
                            li.style.borderBottom = '2px solid #A7727D';
                            li.style.borderTop = 'none';
                        } else {
                            li.style.borderTop = '2px solid #A7727D';
                            li.style.borderBottom = 'none';
                        }
                    }
                });

                li.addEventListener('dragleave', () => {
                    li.style.borderTop = 'none';
                    li.style.borderBottom = 'none';
                });

                li.addEventListener('drop', (e) => {
                    e.preventDefault();
                    li.style.borderTop = 'none';
                    li.style.borderBottom = 'none';

                    if (currentFileDragging && currentFileDragging !== li) {
                        const fromId = parseFloat(currentFileDragging.dataset.id);
                        const toId = parseFloat(li.dataset.id);

                        const fromIndex = pdfFiles.findIndex(f => f.id === fromId);
                        const toIndex = pdfFiles.findIndex(f => f.id === toId);

                        if (fromIndex !== -1 && toIndex !== -1) {
                            const [movedFileEntry] = pdfFiles.splice(fromIndex, 1);
                            pdfFiles.splice(toIndex, 0, movedFileEntry);
                            renderFileList(); // Re-render to reflect new order
                        }
                    }
                });
                fileList.appendChild(li);
            });
        }

        function updateButtonState() {
            // Button is enabled if there's at least one file and no file is loading or has an error
            const canCombine = pdfFiles.length > 0 && pdfFiles.every(f => !f.isLoading && !f.error);
            combineAndDownloadBtn.disabled = !canCombine;
            statusMessage.textContent = ''; // Clear status message when files change
            statusMessage.style.color = 'inherit';
        }

        // --- Page Reorder Modal Logic ---
        function openPageReorderModal(pdfIndex) {
            currentPdfIndexInModal = pdfIndex;
            const pdfEntry = pdfFiles[pdfIndex];
            modalTitle.textContent = `調整 "${pdfEntry.name}" 的頁面順序`;
            renderModalPageList(pdfEntry.pageOrder);
            pageReorderModal.classList.remove('hidden');
        }

        function renderModalPageList(pageOrder) {
            modalPageList.innerHTML = '';
            if (pageOrder.length === 0) {
                modalPageList.innerHTML = '<li class="text-gray-500 text-center py-4">此 PDF 沒有頁面。</li>';
                return;
            }

            pageOrder.forEach((originalPageIndex, displayIndex) => {
                const li = document.createElement('li');
                li.className = 'page-item flex items-center justify-between p-3 rounded-md shadow-sm mb-1 text-gray-700';
                li.draggable = true;
                li.dataset.originalIndex = originalPageIndex; // Store original page index
                li.dataset.displayIndex = displayIndex; // Store current display index

                li.innerHTML = `
                    <span class="flex-grow">頁面 ${originalPageIndex + 1}</span>
                `;

                // Page Drag events
                li.addEventListener('dragstart', (e) => {
                    currentPageDragging = li;
                    e.dataTransfer.effectAllowed = 'move';
                    e.dataTransfer.setData('text/plain', displayIndex); // Dragging display index
                    li.classList.add('dragging');
                });

                li.addEventListener('dragend', () => {
                    currentPageDragging.classList.remove('dragging');
                    currentPageDragging = null;
                });

                li.addEventListener('dragover', (e) => {
                    e.preventDefault();
                    const boundingBox = li.getBoundingClientRect();
                    const offset = e.clientY - boundingBox.top;
                    if (currentPageDragging !== li) {
                        if (offset > boundingBox.height / 2) {
                            li.style.borderBottom = '2px solid #A7727D';
                            li.style.borderTop = 'none';
                        } else {
                            li.style.borderTop = '2px solid #A7727D';
                            li.style.borderBottom = 'none';
                        }
                    }
                });

                li.addEventListener('dragleave', () => {
                    li.style.borderTop = 'none';
                    li.style.borderBottom = 'none';
                });

                li.addEventListener('drop', (e) => {
                    e.preventDefault();
                    li.style.borderTop = 'none';
                    li.style.borderBottom = 'none';

                    if (currentPageDragging && currentPageDragging !== li) {
                        const fromDisplayIndex = parseInt(currentPageDragging.dataset.displayIndex);
                        const toDisplayIndex = parseInt(li.dataset.displayIndex);

                        const currentPdfEntry = pdfFiles[currentPdfIndexInModal];
                        const newPageOrder = [...currentPdfEntry.pageOrder]; // Create a copy
                        
                        const [movedPageOriginalIndex] = newPageOrder.splice(fromDisplayIndex, 1);
                        newPageOrder.splice(toDisplayIndex, 0, movedPageOriginalIndex);
                        
                        currentPdfEntry.pageOrder = newPageOrder; // Update the actual pageOrder
                        renderModalPageList(newPageOrder); // Re-render modal list
                    }
                });
                modalPageList.appendChild(li);
            });
        }

        cancelPageReorderBtn.addEventListener('click', () => {
            pageReorderModal.classList.add('hidden');
            // Optionally, revert changes if needed, but for simplicity, we'll keep the changes
        });

        confirmPageReorderBtn.addEventListener('click', () => {
            pageReorderModal.classList.add('hidden');
            // Changes are already applied to pdfFiles[currentPdfIndexInModal].pageOrder
        });


        // --- PDF Processing ---
        combineAndDownloadBtn.addEventListener('click', async () => {
            if (pdfFiles.length === 0 || pdfFiles.some(f => f.isLoading || f.error)) {
                statusMessage.textContent = '請先上傳有效的 PDF 檔案。';
                statusMessage.style.color = 'red';
                return;
            }

            combineAndDownloadBtn.disabled = true;
            combineButtonText.textContent = '處理中...';
            combineLoadingSpinner.classList.remove('hidden');
            statusMessage.textContent = '正在合併和重排 PDF，請稍候...';
            statusMessage.style.color = '#A7727D';

            try {
                const mergedPdf = await PDFDocument.create();

                for (const fileEntry of pdfFiles) {
                    if (fileEntry.pdfDoc) { // Ensure PDF is loaded
                        const originalPdf = fileEntry.pdfDoc;
                        const pageIndicesToCopy = fileEntry.pageOrder; // Use the reordered page indices

                        for (const originalPageIndex of pageIndicesToCopy) {
                            const [copiedPage] = await mergedPdf.copyPages(originalPdf, [originalPageIndex]);
                            mergedPdf.addPage(copiedPage);
                        }
                    }
                }

                const pdfBytes = await mergedPdf.save();
                const blob = new Blob([pdfBytes], { type: 'application/pdf' });
                const url = URL.createObjectURL(blob);

                const a = document.createElement('a');
                a.href = url;
                a.download = '合併後的PDF.pdf';
                document.body.appendChild(a);
                a.click();
                document.body.removeChild(a);
                URL.revokeObjectURL(url);

                statusMessage.textContent = 'PDF 合併成功！已開始下載。';
                statusMessage.style.color = 'green';

            } catch (error) {
                console.error('PDF 合併失敗:', error);
                statusMessage.textContent = `PDF 合併失敗：${error.message || '未知錯誤'}`;
                statusMessage.style.color = 'red';
            } finally {
                combineAndDownloadBtn.disabled = false;
                combineButtonText.textContent = '合併並下載 PDF';
                combineLoadingSpinner.classList.add('hidden');
            }
        });

        // Initial render
        renderFileList();
        updateButtonState();
    </script>
</body>
</html>
