# -9-
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Шифрование: Перестановка, усложненная по таблице</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #f4f4f4;
            padding: 20px;
            color: #333;
        }
        .container {
            max-width: 900px;
            margin: 0 auto;
            background: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.1);
        }
        h2 {
            margin-top: 0;
            font-size: 18px;
            color: #444;
        }
        textarea {
            width: 100%;
            height: 120px;
            margin-bottom: 15px;
            padding: 8px;
            border: 1px solid #ccc;
            border-radius: 4px;
            box-sizing: border-box;
            font-family: monospace;
            font-size: 14px;
            resize: vertical;
        }
        .controls {
            display: flex;
            gap: 15px;
            align-items: center;
            margin-bottom: 15px;
            flex-wrap: wrap;
        }
        .controls label {
            font-size: 14px;
            font-weight: bold;
        }
        .controls input {
            width: 60px;
            padding: 4px;
            border: 1px solid #ccc;
            border-radius: 4px;
            font-size: 14px;
        }
        button {
            padding: 8px 15px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 14px;
            color: #fff;
            background-color: #007bff;
            transition: background 0.2s;
        }
        button:hover {
            background-color: #0056b3;
        }
        button:disabled {
            background-color: #cccccc;
            cursor: not-allowed;
        }
        button.danger {
            background-color: #dc3545;
            padding: 5px 10px;
            font-size: 16px;
        }
        button.danger:hover {
            background-color: #a71d2a;
        }
        table {
            border-collapse: collapse;
            margin-bottom: 20px;
            width: 100%;
            table-layout: fixed;
        }
        th, td {
            border: 1px solid #999;
            text-align: center;
            padding: 8px;
            font-family: monospace;
            height: 30px;
            overflow: hidden;
        }
        th {
            background-color: #e9ecef;
            font-weight: bold;
        }
        .empty-cell {
            background-color: #ffebee;
            color: #dc3545;
            font-weight: bold;
        }
        .hint {
            font-size: 12px;
            color: #666;
            margin-bottom: 10px;
        }
        #statusMessage {
            display: none;
            margin-top: 15px;
            padding: 10px;
            background-color: #d4edda;
            color: #155724;
            border: 1px solid #c3e6cb;
            border-radius: 4px;
            text-align: center;
            font-weight: bold;
        }
    </style>
</head>
<body>

<div class="container">
    <h2>Исходное сообщение:</h2>
    <textarea id="sourceText" placeholder="Введите текст для шифрования (можно в несколько строк)..."></textarea>

    <div class="controls">
        <label>Размер блока: N= <input type="number" id="nInput" value="" min="3"></label>
        <label>M= <input type="number" id="mInput" value="" min="3"></label>
        <label>Кол-во неисп. ячеек: <input type="number" id="zInput" value="" min="1"></label>
        <button id="btnGenerateKey" class="danger" title="Изменить расположение неиспользуемых ячеек">↻</button>
    </div>
    <div class="hint">Таблица генерируется автоматически при изменении N, M и Z.</div>

    <div id="gridContainer" style="overflow-x: auto; min-height: 50px; margin-bottom: 20px;">
        <!-- Таблица будет сгенерирована здесь -->
    </div>

    <button id="btnEncrypt" style="width: 100%; padding: 12px; font-size: 16px;">Зашифровать и скачать файлы</button>

    <!-- Блок для уведомления (теперь он есть в HTML) -->
    <div id="statusMessage"></div>
</div>

<script>
    let n = 0, m = 0, z = 0;
    let grid = [];
    let keyRow0 = [];
    let lastN = 0;

    const sourceText = document.getElementById('sourceText');
    const gridContainer = document.getElementById('gridContainer');
    const btnEncrypt = document.getElementById('btnEncrypt');
    const btnGenerateKey = document.getElementById('btnGenerateKey');
    const statusMessage = document.getElementById('statusMessage');

    function downloadFile(filename, content) {
        const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    }

    
    function checkInputs() {
        if (sourceText.value !== '' && n >= 3 && m >= 3 && z > 0 && grid.length > 0) {
            btnEncrypt.disabled = false;
        } else {
            btnEncrypt.disabled = true;
        }
    }


    function autoGenerateGrid() {
        const nVal = parseInt(document.getElementById('nInput').value) || 0;
        const mVal = parseInt(document.getElementById('mInput').value) || 0;
        const zVal = parseInt(document.getElementById('zInput').value) || 0;

        if (nVal < 3 || mVal < 3 || zVal <= 0 || zVal >= nVal * mVal - nVal) {
            
            grid = [];
            checkInputs();
            return;
        }

        n = nVal;
        m = mVal;
        z = zVal;
        if (n !== lastN || keyRow0.length !== n) {
            keyRow0 = Array.from({length: n}, (_, i) => i + 1);
            for (let i = keyRow0.length - 1; i > 0; i--) {
                const j = Math.floor(Math.random() * (i + 1));
                [keyRow0[i], keyRow0[j]] = [keyRow0[j], keyRow0[i]];
            }
            lastN = n;
        }
        grid = Array.from({ length: m }, () => Array(n).fill(''));
        grid[0] = [...keyRow0];

        let placed = 0;
        while (placed < z) {
            let r = Math.floor(Math.random() * (m - 1)) + 1;
            let c = Math.floor(Math.random() * n);
            if (grid[r][c] !== '-') {
                grid[r][c] = '-';
                placed++;
            }
        }

        renderGrid();
        checkInputs();
    }
    function renderGrid() {
        if (grid.length === 0) return;

        let html = '<table>';
        for (let r = 0; r < m; r++) {
            html += '<tr>';
            for (let c = 0; c < n; c++) {
                let val = grid[r][c];
                let cellClass = (r > 0 && val === '-') ? 'empty-cell' : '';
                html += `<td class="${cellClass}">${val}</td>`;
            }
            html += '</tr>';
        }
        html += '</table>';
        gridContainer.innerHTML = html;
    }
    document.getElementById('nInput').addEventListener('input', autoGenerateGrid);
    document.getElementById('mInput').addEventListener('input', autoGenerateGrid);
    document.getElementById('zInput').addEventListener('input', autoGenerateGrid);
    sourceText.addEventListener('input', checkInputs);
    btnGenerateKey.addEventListener('click', () => {
        if (grid.length === 0) return;
        
        for (let r = 1; r < m; r++) {
            for (let c = 0; c < n; c++) {
                grid[r][c] = '';
            }
        }

        let placed = 0;
        while (placed < z) {
            let r = Math.floor(Math.random() * (m - 1)) + 1;
            let c = Math.floor(Math.random() * n);
            if (grid[r][c] !== '-') {
                grid[r][c] = '-';
                placed++;
            }
        }
        renderGrid();
    });
    btnEncrypt.addEventListener('click', () => {
        statusMessage.style.display = 'none';

        let text = sourceText.value.replace(/\r/g, ''); 
        const blockSize = n * m - z;
        let ciphertext = '';
        let keyString = keyRow0.join('#') + '#\n';
        keyString += `${n}#${m}#\n`;
        
        let emptyCellsIndices = [];
        for (let r = 1; r < m; r++) {
            for (let c = 0; c < n; c++) {
                if (grid[r][c] === '-') {
                    emptyCellsIndices.push((r - 1) * n + c + 1);
                }
            }
        }
        keyString += emptyCellsIndices.join('#') + '#';

        let pos = 0;
        while (pos < text.length) {
            let block = text.substring(pos, pos + blockSize);
            pos += blockSize;

            for (let r = 1; r < m; r++) {
                for (let c = 0; c < n; c++) {
                    if (grid[r][c] !== '-') grid[r][c] = '';
                }
            }
            let charIdx = 0;
            for (let r = 1; r < m; r++) {
                for (let c = 0; c < n; c++) {
                    if (grid[r][c] !== '-' && charIdx < block.length) {
                        grid[r][c] = block[charIdx++];
                    }
                }
            }

            for (let k = 1; k <= n; k++) {
                let colIndex = keyRow0.indexOf(k);
                for (let r = 1; r < m; r++) {
                    if (grid[r][colIndex] !== '-' && grid[r][colIndex] !== '') {
                        ciphertext += grid[r][colIndex];
                    }
                }
            }
        }

        renderGrid(); 
        downloadFile('Output.txt', ciphertext);
        
        setTimeout(() => {
            downloadFile('Input.txt', keyString);
            
            statusMessage.style.display = 'block';
            
            setTimeout(() => {
                statusMessage.style.display = 'none';
            }, 5000);

        }, 500);
    });

    window.onload = function() {
        autoGenerateGrid();
    };
</script>

</body>
</html>
