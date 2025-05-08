<script setup>
import { ref, watch } from 'vue'

const defaultFormValue = {
  length: 12,
  upper: 2,
  lower: 4,
  symbol: 2,
  number: 2
} 
  //如果長度是使用者自行決定的話，此時的length會是''嗎?

// ＃表單資料重置邏輯說明：為什麼使用 {...defaultFormValue} 而不是直接指派 = defaultFormValue？
/*
  在這裡我們使用淺拷貝（spread operator）將 defaultFormValue 展開成新的物件，
  確保 reset 時「不是直接綁定原物件」，而是創造一份新的表單內容。

  若寫成：
    form.value = defaultFormValue
  會造成 form 與 defaultFormValue 指向同一個記憶體位置
  → 未來若修改 form，原始預設值也會被意外改變！

  所以：
    form.value = { ...defaultFormValue }
  可以保留 defaultFormValue 不被污染，同時正確重設表單。

  ▲ 簡單測試：
    let newFormValue = defaultFormValue;
    let copyFormValue = { ...defaultFormValue }
    console.log(newFormValue == defaultFormValue);  //輸出結果true
    console.log(copyFormValue == defaultFormValue); //輸出結果false

  延伸說明：什麼是「深拷貝」？
  假設 defaultFormValue 中有一個物件欄位（例如 meta）：
    const defaultFormValue = {
      length: 12,
      upper: 2,
      lower: 4,
      symbol: 2,
      number: 2,
      meta: { 
        updatedBy: 'Lumi' 
      }
    }

  若使用 {...defaultFormValue}，會複製第一層的欄位內容，但 meta 屬性仍然是「同一個物件參考」：
    let copyFormValue = { ...defaultFormValue }

  ▲ 深拷貝測試：
    console.log(copyFormValue == defaultFormValue); //輸出結果false
    console.log(copyFormValue.meta == defaultFormValue.meta); //輸出結果true

  ● 結論：
    → spread operator 是「淺拷貝」
    → 巢狀物件或陣列內容仍然共享記憶體
    → 若要避免這種共享，需使用「深拷貝」

  常見深拷貝做法：
    - 使用 JSON：JSON.parse(JSON.stringify(obj))
    - 使用 JS 工具庫：如 lodash 的 _.cloneDeep()

  √ 補充：
    本範例中的 defaultFormValue 全為原始型別（number），不含巢狀結構，
    所以使用淺拷貝已經足夠。
*/

// 參考：
// JavaScript spread syntax 官方文件
// https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
// JavaScript deep copy vs shallow copy 官方文件：
// https://developer.mozilla.org/en-US/docs/Glossary/Deep_copy
const form = ref({ ...defaultFormValue });
const result = ref('');
const hint = ref('');

//資源池：
//  小寫  
// ＃字元編碼轉換說明：為什麼可以用 String.fromCharCode(97 + i) 產生小寫英文字母？
/*
  每個字母在電腦中都有對應的數值編碼（ASCII）：
    - 'a' = 97，'b' = 98，...，'z' = 122

  String.fromCharCode(97 + i) 可以將數字轉回字元：
    → 97 + 0 = 'a'、97 + 1 = 'b'、依此類推

  結合 Array.from({ length: 26 }, (_, i) => ...)，就能快速產出 a~z 陣列。

  若想產出大寫，從 65 開始即可：
    → String.fromCharCode(65) = 'A'
*/
// 參考：詳解3
const lowercaseChars = Array.from({ length: 26 }, (_, i) =>
  String.fromCharCode(97 + i)
) 

//  大寫
const uppercaseChars = lowercaseChars.map((char) => char.toUpperCase());
//  數字
const numberChars = Array.from({ length: 10 }, (_, i) => i.toString());
//  符號
const symbolChars = [
  '!', '@', '#', '$', '%', '^', '&', '*', '(', ')',
  '-', '_', '=', '+', '[', ']', '{', '}', ';', ':',
  ',', '.', '<', '>', '?', '/'
];
// 生成混合大小寫、數字、符號資源池
const charPool = [...lowercaseChars, ...uppercaseChars, ...numberChars, ...symbolChars];

// ＃表單同步重置機制說明：為什麼不使用 <form> 原生 reset，而改用 @click 搭配 resetForm()？
/*
  HTML 提供了 <button type="reset"> 的原生重設功能，
  會自動清除該 <form> 範圍內所有 input 的畫面值。

  但！Vue 使用 v-model 雙向綁定資料物件，
  原生 reset 只會清除 DOM 的顯示值（也就是你「眼睛看到的」），
  **不會改變 v-model 綁定的資料（例如 form.value）**

  ▲ 測試範例：
    - 預設 form.value = { length: 12, upper: 2, ... }
    - 點擊 reset 按鈕後，畫面會清空
    - 但 console.log(form.value) 結果仍是原本的資料
      → 沒有同步

  ● 問題：
    → 畫面看起來已清空，但實際資料沒變
    → 若再次觸發 Vue 的渲染或事件，這些「看似被清除」的值可能會重新出現
    → 「產生密碼」的邏輯仍會使用舊資料，導致結果與畫面不一致

  √ 延伸補充：
    若日後使用 UI 框架、元件封裝、或非同步資料初始化等情境，
    自定義 reset 方法會更穩定、可測試、也更易於維護。
*/
const resetForm = () => {
  form.value = { ...defaultFormValue }
  result.value = ''
  hint.value = ''
}

// ＃複製密碼 copyResult()
// 使用 Clipboard API 將產生的密碼複製到剪貼簿
/*
  ▲ 核心概念：
    navigator.clipboard.writeText(...) 是一個非同步操作，會回傳 Promise 物件

  ★ 基本寫法（使用 .then() 處理結果）：
*/
// 參考：詳解4
const copyResult = () => {
  if (result.value) {
    navigator.clipboard.writeText(result.value).then(() => {
      alert('密碼已複製到剪貼簿！')
    }).catch((err) => {
      alert('複製失敗：' + err)
    })
  }
}

watch(form, () => {
  const { length, upper, lower, symbol, number } = form.value;  // 參考：詳解1
  const minRequired = upper + lower + symbol + number;
  if (length < minRequired) {
    hint.value = `X 規則需求總長 ${minRequired} 已超過設定長度 ${length}`;
  } else {
    hint.value = ''
  }
}, { deep: true })
  //這裡的{ deep: true } 是在監聽 form.value嗎? 是因為length會變化嗎?
  //還是因為defaultFormValue代出的是物件，只要更改任何一個值，都算深層的修改

const generatePassword = () => {
  const { length, upper, lower, symbol, number } = form.value;  // 參考：詳解1
  const minRequired = upper + lower + symbol + number;

  if (hint.value) {
    alert(`密碼設定長度 ${length}，但規則需求長度 ${minRequired}。請重新確認。`);
    //這個 alert是什麼時候會出現? 因為預設長度是12時，就算資料加起來只有10，也可以生成密碼
    //如果長度小於資料的值則會出現這句 `X 規則需求總長 ${minRequired} 已超過設定長度 ${length}` 則按鈕失效，無法觸擊
    return;
  }

  let resultArr = [];

  if (upper) resultArr.push(...insertChar(uppercaseChars, upper));
  if (lower) resultArr.push(...insertChar(lowercaseChars, lower));
  if (symbol) resultArr.push(...insertChar(symbolChars, symbol));
  if (number) resultArr.push(...insertChar(numberChars, number));
  if (resultArr.length < length) resultArr.push(...insertChar(charPool, length - minRequired));

  result.value = shuffle(resultArr).join('');
}

// ＃隨機抽取陣列元素 insertChar(arr, len)
// 用來從給定的字元陣列中，隨機抽取指定數量（len）的項目，回傳為新陣列。
/*
  ▲ 實作目標：
  ● arr：來源陣列（如：小寫字母、大寫字母、符號等）
  ● len：要抽出的數量（幾個字元）
  ● 回傳一個長度為 len 的新陣列，每個元素為 arr 中隨機選取的值

  ----------------------------------------
  ▼ 傳統版本（for 迴圈方式）：
  
  const insertChar = (arr, len) => {
    let charArr = []
    for (let i = 0; i < len; i++) {
      const randomChar = arr[Math.floor(Math.random() * arr.length)]
      charArr.push(randomChar)
    }
    return charArr
  }

  ● 優點：語意清楚、容易追蹤邏輯流程
  ● 適合教學初期或剛學習隨機邏輯的讀者

  ----------------------------------------
  ▼ 精簡版本（Array.from + 隨機取值）：

  ● 優點：程式碼簡潔、語法現代
  ● 缺點：不熟悉語法者可能不易理解，適合進階使用

  ----------------------------------------
  ★ 小提醒：
    這兩種寫法在邏輯上完全等價，選擇哪一種取決於情境與團隊風格。
    - 想教概念 → 用 for 迴圈
    - 想練語法 → 用 Array.from()
*/
// 參考：詳解2
const insertChar = (arr, len) => Array.from({ length: len }, () =>
  arr[Math.floor(Math.random() * arr.length)]
)

// ＃Fisher–Yates 洗牌演算法
// 為什麼要洗牌？為了將密碼中所有類別字元的順序打亂，避免出現模式化結果。
/*
  ▲ 為什麼是從「最後一個元素」往前？
  ● 因為這樣能保證：
    - 每一輪隨機的目標索引（j）只會落在尚未洗過的部分
    - 隨機空間會隨著迴圈遞減，逐步完成每一項交換

  ★ 實作技巧：
    const j = Math.floor(Math.random() * (i + 1))
    → i 會從尾端開始遞減，這樣每圈都只會動到還沒處理過的部分

  ▲ 為什麼使用「解構賦值」交換？
  ● 使用 ES6 解構語法：[a, b] = [b, a]
    可以避免使用 temp 暫存變數，提升可讀性與簡潔性：
    → [array[i], array[j]] = [array[j], array[i]]

  ▲ 如果不使用解構，可以怎麼寫？
  ● 可以使用傳統的暫存變數 temp：
      const temp = array[i]
      array[i] = array[j]
      array[j] = temp
    → 這種寫法功能完全正確，但程式碼較冗長，現代開發多優先使用解構。

  ▲ 陣列是否需要複製？
  ● 如果你後續還要保留原始順序，就應該複製一份資料再洗：
    → const shuffled = [...array]

    本範例直接原地洗牌，因為 resultArr 不再需要原順序
    所以可直接修改本體，提升效能。
*/
const shuffle = (arr) => {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];    // 參考：詳解1
  }
  return arr
}



/*
==============================
＃詳解標記區（知識點統整）
==============================

● 詳解1. 解構賦值（Destructuring assignment）
    解構賦值是一種語法糖，讓你可以快速從陣列或物件中提取值並賦給變數。

    例如：
    const arr = [10, 20]
    const [a, b] = arr
    → a = 10, b = 20

    在本專案中，我們使用這段程式碼：
    const { length, upper, lower, symbol, number } = form.value

    表示將 `form.value` 中的每個欄位，快速取出對應值並賦予變數名稱，
    相當於：
    const length = form.value.length
    const upper = form.value.upper
    ...以此類推

    ● 解構也可以應用在陣列交換：
    [arr[i], arr[j]] = [arr[j], arr[i]]
    等同於使用暫存變數交換值的寫法：
    const temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;

    → 這種寫法常見於「Fisher–Yates 洗牌」等邏輯中，
    可簡潔地完成元素交換，是現代 JavaScript 中很推薦的技巧。

    參考文件：
    → https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment

● 詳解2. 隨機抽取陣列元素 insertChar
    這是一個自訂函式，用來從給定的陣列中，隨機抽取指定數量的元素。

    本次使用兩種寫法（邏輯相同）：

    ▼ 傳統版本：
    const insertChar = (arr, len) => {
        let charArr = []
        for (let i = 0; i < len; i++) {
            const randomChar = arr[Math.floor(Math.random() * arr.length)]
            charArr.push(randomChar)
        }
        return charArr
    }

    ▼ 精簡版本（教學使用）：
    const insertChar = (arr, len) => Array.from({ length: len }, () =>
        arr[Math.floor(Math.random() * arr.length)]
    )

    ◎ 核心邏輯說明：
        - `Math.random()`：會回傳一個 0～1 之間的小數
        - `Math.floor(...)`：將其向下取整，變成整數
        - `arr[...]`：用上述隨機索引，從陣列中取出對應字元
        - 重複 `len` 次，產生新陣列

    ★ 實務小提醒：
    這種邏輯常見於「抽卡」、「隨機選字」、「密碼組合」等場景中，
    你也可以用這個邏輯實作簡易的抽籤工具、點名器等。

    參考：Math.random() + Array.from() 文件
    → https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random
    → https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/from

● 詳解3. 字元編碼轉換與 ASCII 表
    ◎ ASCII 是什麼？
        - ASCII（American Standard Code for Information Interchange）是一種文字編碼系統。
    ◆ 每個英文字母、符號、數字都對應一個數字編號（0～127）
        → 'a' = 97
        → 'A' = 65
        → '0' = 48

    ◎ 常用函式：String.fromCharCode()
        將數字轉換成字元：
            String.fromCharCode(97) → 'a'
            String.fromCharCode(65) → 'A'

    ◎ 配合 Array.from 實作英文字母陣列：
        - 建立 a~z：
            Array.from({ length: 26 }, (_, i) => String.fromCharCode(97 + i))
        - 建立 A~Z：
            Array.from({ length: 26 }, (_, i) => String.fromCharCode(65 + i))

    ◎ 額外補充：如何反查字元的 ASCII 編碼？
        使用 'a'.charCodeAt(0) → 回傳 97

    ◎ 實務應用場景：
        - 建立字母表、驗證輸入格式
        - 判斷是否為大寫、小寫或數字
        - 密碼組合、字串處理、資安邏輯等

    參考文件：
    → https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/fromCharCode
    → https://www.ascii-code.com/

● 詳解4. 非同步與 Promise（以 clipboard 為例）
    ◎ 核心觀念：
        ● JavaScript 是單執行緒語言，但支援非同步操作（例如：資料請求、剪貼簿寫入）
        ● 非同步操作通常會「立即返回一個 Promise」，表示「稍後才會完成」

    ◎ Promise 是什麼？
        ● 一種代表「未來完成（或失敗）」的非同步結果容器

    ◎ 兩種常見寫法：
        ▼ 方法一：使用 .then() 搭配 .catch()
            navigator.clipboard.writeText(result.value).then(() => {
                alert('密碼已複製到剪貼簿！');
            }).catch(err => {
                alert('複製失敗：' + err);
            })

        ▼ 方法二：使用 async/await 語法
            const copyResult = async () => {
                try {
                    await navigator.clipboard.writeText(result.value);
                    alert('密碼已複製到剪貼簿！');
                } catch (err) {
                    alert('複製失敗：' + err);
                }
            }

    ★ 延伸說明：
        ● await 本身就是語法糖，讓非同步操作的流程看起來像同步
        ● 兩者功能一致，但 async/await 更適合多段連續邏輯
        ● 非同步 ≠ 多執行緒，實作原理是靠「事件迴圈」與「任務佇列」

    ◆ 實務小提醒：
        - 若需要等待結果再進行後續處理，建議使用 await（或 .then() 回傳 Promise）
        - alert() 不屬於 Promise 機制，但常用於通知完成狀態

    參考文件：
    → Promise：https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
    → Clipboard API：https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/writeText

    ● 詳解5. push(...陣列) vs push(陣列)
    ◎ 核心概念：
        push(...) 是陣列方法，用來將元素加到原陣列尾端。
        但... 推進「什麼形式」的資料，會影響結果！

    ◎ 說明：
        假設：
        let resultArr = []
        let sample = ['A', 'B', 'C']

        ▼ 寫法一：resultArr.push(sample)
            → 推進的是「整個陣列」當作一個元素
            → 結果變成：[['A', 'B', 'C']]

        ▼ 寫法二：resultArr.push(...sample)
            → 使用展開運算子（spread operator）
            → 將 sample 中的每個元素「展開後個別推入」
            → 結果變成：['A', 'B', 'C']

    ◎ 對照：
        - push(陣列)     → 陣列被當成單一項目
        - push(...陣列)  → 陣列中每個元素被逐一推入

    ◎ 應用於本範例：
        我們使用：
        resultArr.push(...insertChar(uppercaseChars, upper))
        因為：
            - insertChar() 回傳的是一整個陣列
            - 我們要把裡面的每個字元加入 resultArr
            - 所以需要展開：...陣列

        若寫成：
        resultArr.push(insertChar(...))
            → 將導致 resultArr 裡面變成「巢狀陣列」

        ※ 常見錯誤：
            resultArr = [1, 2]
            resultArr.push([3, 4])
                → [1, 2, [3, 4]]

            resultArr.push(...[3, 4])
                → [1, 2, 3, 4]

    ◎ 小總結：
        ◆ push() 是推進「幾項」的動作
        ◆ ... 是展開「集合」的語法
            → 兩者搭配時，會決定要推幾個元素進陣列！
            → 當你看到 push(...)，請記得它會「展開陣列內部元素」
            → 若直接 push 陣列，則會把整個陣列視為一個元素加入

    ※ 參考文件：
    → Array.prototype.push(): https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/push
    → Spread syntax: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
*/

</script>

<template>
  <div class="container">
    <h2 class="title">密碼產生器</h2>

    <form class="form">
      <div class="form-group">
        <label for="length">密碼總長度</label>
        <input id="length" type="number" min="0" v-model.number="form.length" class="input" :class="{ 'input-error': hint }"/>
      </div>

      <div class="form-group">
        <label for="upper">大寫字元數</label>
        <input id="upper" type="number" min="0" v-model.number="form.upper" class="input" />
      </div>

      <div class="form-group">
        <label for="lower">小寫字元數</label>
        <input id="lower" type="number" min="0" v-model.number="form.lower" class="input" />
      </div>

      <div class="form-group">
        <label for="symbol">符號數量</label>
        <input id="symbol" type="number" min="0" v-model.number="form.symbol" class="input" />
      </div>

      <div class="form-group">
        <label for="number">數字數量</label>
        <input id="number" type="number" min="0" v-model.number="form.number" class="input" />
      </div>

      <div class="form-actions">
        <button type="button" @click="resetForm" class="reset-button">重設條件</button>
      </div>
    </form>

    <div v-if="hint" class="hint">{{ hint }}</div>

    <div class="generate-action">
      <button type="button" @click="generatePassword" :disabled="!!hint" class="generate-button">產生密碼</button>
    </div>

    <div class="result-block">
      <label class="result-label">產生結果：</label>
      <div class="result-box">
        <span class="result-text">{{ result }}</span>
        <button @click="copyResult" class="copy-button">複製</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 400px;
  margin: 40px auto;
  padding: 24px;
  background-color: #fff;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-family: sans-serif;
}

.title {
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 20px;
  text-align: center;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.form-group {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.input {
  width: 80px;
  padding: 6px;
  text-align: right;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.form-actions {
  text-align: right;
  margin-top: 8px;
}

.reset-button {
  background: none;
  border: none;
  color: #007bff;
  cursor: pointer;
  font-size: 14px;
  text-decoration: underline;
}

.hint {
  margin-top: 16px;
  padding: 8px;
  background-color: #fff8e1;
  color: #8a6d3b;
  border: 1px solid #faebcc;
  border-radius: 4px;
  font-size: 14px;
}

.input-error {
  border: 1px solid #dc3545;
  background-color: #fff5f5;
}

.generate-action {
  margin-top: 12px;
  text-align: center;
}

.generate-button {
  padding: 8px 16px;
  font-size: 14px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
.generate-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.result-block {
  margin-top: 24px;
}

.result-label {
  font-weight: bold;
  display: block;
  margin-bottom: 8px;
}

.result-box {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #f0f0f0;
  padding: 10px;
  border-radius: 6px;
}

.result-text {
  word-break: break-all;
  flex: 1;
}

.copy-button {
  margin-left: 10px;
  background: none;
  border: none;
  color: #007bff;
  cursor: pointer;
  font-size: 14px;
  text-decoration: underline;
}
</style>


<!--
邏輯說明

功能流程：
1. 初始設定：
   - defaultFormValue 定義初始密碼規則（長度、各類字元數量）
   - form 使用 `ref({ ...defaultFormValue })`，透過淺拷貝確保資料隔離

2. 監控變化：
   - 使用 `watch(form, ..., { deep: true })` 深層監聽
   - 每當使用者更改欄位（如 upper、lower、symbol、number），會即時比對是否總需求長度 > 密碼長度
   - 若不符則更新 hint，並禁止按鈕點擊（disabled）

3. 重設表單：
   - 點擊重設按鈕時觸發 `resetForm()`
   - 使用 `{ ...defaultFormValue }` 淺拷貝重建 form，並清空 result 與 hint
   - 避免原生 reset 造成畫面與資料不同步的問題（詳見註解）

4. 產生密碼：
   - 點擊「產生密碼」後，若 hint 存在則跳出提醒終止
   - 各類字元使用 insertChar 隨機抽取
     → 若還有剩餘長度，從 charPool 抽補齊
   - 將結果交給 shuffle() 洗牌，打亂順序後輸出為字串

5. 複製密碼：
   - 點擊「複製」按鈕時使用 `navigator.clipboard.writeText(result)`
   - 採用 Promise then 寫法，確保非同步完成後再 alert 通知

命名與邏輯設計：
- form / defaultFormValue：表單資料與初始狀態
- result：儲存最終密碼
- hint：儲存驗證警告訊息
- insertChar(arr, len)：從來源陣列隨機抽取 len 個字元
- shuffle(arr)：使用 Fisher–Yates 洗牌法打亂密碼順序
- copyResult()：複製剪貼簿

防呆設計：
- 密碼長度驗證：即時比對總規則需求是否超過輸入長度
- 禁用按鈕：hint 存在時禁止觸發密碼產生邏輯
- 重設使用淺拷貝：避免直接覆蓋原始參考值
- 錯誤處理：複製時 catch Promise 錯誤並提醒

附註：
- 這份程式涵蓋了許多基礎邏輯設計概念，包含：
  → v-model 與資料同步問題
  → 深拷貝 vs 淺拷貝
  → 資料與視圖的分離處理
  → 解構賦值、洗牌演算法、字元編碼
  → 非同步操作（Clipboard API、Promise）
- 若搭配註解學習，將能強化資料結構與流程控制的理解，並奠定更進一步的應用能力基礎

＃實務補充與安全建議
    本範例屬於「進階任務設計」，不僅結構清晰、資料處理完善，邏輯也貼近實務應用。
    並未刻意簡化觀念，若你剛完成初階 JS/Vue，請多次閱讀與實作，這是進入中階開發者的重要訓練。

    ◎實務補充：前端產生密碼 vs 後端產生？
    ● 在實際應用中，如果密碼會進入帳號系統、登入驗證或伺服器儲存流程，應由後端統一產生或驗證：
        - 可以控管格式與強度
        - 方便日誌紀錄與資安追蹤
        - 可避免前端資料被竄改後直接傳入資料庫

    ● 但若只是為使用者提供「臨時密碼建議」或「本地端用途」，則前端產生是合理且常見的做法，常見場景包含：
        - 註冊畫面提供建議密碼
        - 密碼更新前的即時強度測試
        - 不與伺服器互動的個人密碼工具

    ★ 進階補充（若考慮高安全性場景）：
        - 可改用 `crypto.getRandomValues()` 或 Web Crypto API 提供更高熵的亂數
        - 若資料需儲存，應經過加密或散列處理（例如 SHA256 + 加鹽）

    ● 總結：
        → 若是 UX 為主、無需儲存密碼 → 前端產生即可
        → 若牽涉帳號安全、驗證流程 → 應交由後端負責
-->
