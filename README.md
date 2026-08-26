<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>DeepL 汉字笔顺描红字帖</title>
  <script src="https://cdn.jsdelivr.net/npm/hanzi-writer@3.5/dist/hanzi-writer.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/pinyin-pro@3.18.3/dist/index.js"></script>
  
  <style>
    * { box-sizing: border-box; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "PingFang SC", "Microsoft YaHei", sans-serif;
      margin: 0;
      padding: 20px;
      background: #f0f2f5;
    }

    /* 顶部控制栏 */
    .controls {
      background: #fff;
      padding: 16px 20px;
      border-radius: 8px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
      max-width: 900px;
      margin: 0 auto 20px;
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      align-items: center;
    }
    .controls input[type="text"], .controls input[type="password"] {
      padding: 8px 12px;
      font-size: 14px;
      border: 1px solid #d9d9d9;
      border-radius: 4px;
    }
    #textInput { flex: 2; min-width: 180px; }
    #apiKeyInput { flex: 1.5; min-width: 180px; }
    
    .controls select {
      padding: 8px 10px;
      font-size: 14px;
      border: 1px solid #d9d9d9;
      border-radius: 4px;
      background: #fff;
    }
    .btn {
      padding: 8px 16px;
      font-size: 14px;
      background: #1890ff;
      color: #fff;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    .btn-print { background: #52c41a; }
    .btn:hover { opacity: 0.9; }

    /* 字帖 A4 预览容器 */
    .sheet-page {
      background: #fff;
      width: 210mm;
      min-height: 297mm;
      margin: 0 auto;
      padding: 15mm;
      box-shadow: 0 4px 16px rgba(0,0,0,0.1);
      box-sizing: border-box;
    }

    .char-row {
      display: flex;
      align-items: center;
      margin-bottom: 12px;
      border-bottom: 1px dashed #e8e8e8;
      padding-bottom: 8px;
    }

    .char-meta {
      width: 100px;
      text-align: center;
      margin-right: 12px;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    .char-pinyin {
      font-size: 15px;
      color: #1890ff;
      font-weight: bold;
    }
    .char-trans {
      font-size: 12px;
      color: #4a5568;
      margin: 2px 0 4px;
      max-width: 95px;
      word-break: break-word;
      line-height: 1.2;
    }
    .btn-animate {
      padding: 2px 6px;
      font-size: 11px;
      background: #fa8c16;
      color: #fff;
      border: none;
      border-radius: 3px;
      cursor: pointer;
    }

    .grid-list {
      display: flex;
      flex-wrap: wrap;
      gap: 6px;
    }
    .mi-grid {
      width: 58px;
      height: 58px;
      border: 1px solid #ff4d4f;
      position: relative;
      background:
        linear-gradient(to top right, transparent calc(50% - 0.5px), #ffa39e 50%, transparent calc(50% + 0.5px)),
        linear-gradient(to bottom right, transparent calc(50% - 0.5px), #ffa39e 50%, transparent calc(50% + 0.5px)),
        linear-gradient(to right, transparent calc(50% - 0.5px), #ffa39e 50%, transparent calc(50% + 0.5px)),
        linear-gradient(to bottom, transparent calc(50% - 0.5px), #ffa39e 50%, transparent calc(50% + 0.5px));
      background-size: 100% 100%;
    }

    @media print {
      body { background: transparent; padding: 0; }
      .controls { display: none !important; }
      .sheet-page { box-shadow: none; width: 100%; margin: 0; padding: 0; }
      .btn-animate { display: none; }
    }
  </style>
</head>
<body>

  <div class="controls">
    <input type="text" id="textInput" value="山水木林" placeholder="输入生字..." />
    
    <!-- 语言选项（匹配 DeepL 语言代码） -->
    <select id="langSelect">
      <option value="DE">🇩🇪 德语 (DE)</option>
      <option value="EN-GB">🇬🇧 英语 (EN)</option>
      <option value="FR">🇫🇷 法语 (FR)</option>
      <option value="ES">🇪🇸 西班牙语 (ES)</option>
      <option value="none">🚫 无翻译</option>
    </select>

    <!-- DeepL API Key 输入框（会自动保存在浏览器本地，无需重复输入） -->
    <input type="password" id="apiKeyInput" placeholder="DeepL API Key (xxxx:fx)" />

    <button class="btn" onclick="generateSheet()">生成字帖</button>
    <button class="btn btn-print" onclick="window.print()">🖨️ 打印</button>
  </div>

  <div class="sheet-page" id="sheetContainer"></div>

  <script>
    const { pinyin } = pinyinPro;

    // 页面载入时自动恢复已保存的 API Key
    window.onload = () => {
      const savedKey = localStorage.getItem('deepl_api_key');
      if (savedKey) document.getElementById('apiKeyInput').value = savedKey;
      generateSheet();
    };

    // DeepL 翻译函数
    async function translateWithDeepL(text, targetLang, apiKey) {
      if (targetLang === 'none' || !apiKey) return '';
      
      // 判断是免费版端点还是 Pro 版端点
      const endpoint = apiKey.endsWith(':fx') 
        ? 'https://api-free.deepl.com/v2/translate' 
        : 'https://api.deepl.com/v2/translate';

      try {
        const response = await fetch(endpoint, {
          method: 'POST',
          headers: {
            'Authorization': `DeepL-Auth-Key ${apiKey}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            text: [text],
            target_lang: targetLang,
            source_lang: 'ZH'
          })
        });

        if (!response.ok) {
          console.error('DeepL API 请求错误:', response.status);
          return '';
        }

        const data = await response.json();
        return data.translations[0]?.text?.toLowerCase() || '';
      } catch (err) {
        console.error('翻译失败:', err);
        return '';
      }
    }

    async function generateSheet() {
      const container = document.getElementById('sheetContainer');
      const text = document.getElementById('textInput').value.trim();
      const targetLang = document.getElementById('langSelect').value;
      const apiKey = document.getElementById('apiKeyInput').value.trim();

      // 保存 API Key 到本地缓存
      if (apiKey) localStorage.setItem('deepl_api_key', apiKey);

      container.innerHTML = '';
      const chars = text.match(/[\u4e00-\u9fa5]/g) || [];
      if (chars.length === 0) {
        alert('请输入汉字！');
        return;
      }

      for (let charIdx = 0; charIdx < chars.length; charIdx++) {
        const char = chars[charIdx];
        const row = document.createElement('div');
        row.className = 'char-row';

        const py = pinyin(char, { toneType: 'symbol' });
        const transBoxId = `trans-${charIdx}`;

        const metaBox = document.createElement('div');
        metaBox.className = 'char-meta';
        metaBox.innerHTML = `
          <div class="char-pinyin">${py}</div>
          <div class="char-trans" id="${transBoxId}">...</div>
          <button class="btn-animate" onclick="playStroke('${charIdx}', '${char}')">▶ 演示</button>
        `;
        row.appendChild(metaBox);

        const gridList = document.createElement('div');
        gridList.className = 'grid-list';
        for (let i = 0; i < 7; i++) {
          const grid = document.createElement('div');
          grid.className = 'mi-grid';
          grid.id = `char-${charIdx}-grid-${i}`;
          gridList.appendChild(grid);
        }
        row.appendChild(gridList);
        container.appendChild(row);

        // 渲染示范与描红格
        HanziWriter.create(`char-${charIdx}-grid-0`, char, {
          width: 58, height: 58, padding: 5, strokeColor: '#000000', showOutline: false
        });
        for (let i = 1; i <= 2; i++) {
          HanziWriter.create(`char-${charIdx}-grid-${i}`, char, {
            width: 58, height: 58, padding: 5, strokeColor: '#d9d9d9', showOutline: false
          });
        }

        // 调用 DeepL 翻译填充
        translateWithDeepL(char, targetLang, apiKey).then(res => {
          const el = document.getElementById(transBoxId);
          if (el) el.innerText = res;
        });
      }
    }

    function playStroke(charIdx, char) {
      const targetElem = document.getElementById(`char-${charIdx}-grid-0`);
      targetElem.innerHTML = '';
      const writer = HanziWriter.create(targetElem.id, char, {
        width: 58, height: 58, padding: 5,
        strokeColor: '#1890ff',
        strokeAnimationSpeed: 1.2,
        delayBetweenStrokes: 150
      });
      writer.animateCharacter();
    }
  </script>
</body>
</html>
