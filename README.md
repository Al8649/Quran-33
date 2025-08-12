<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>قارئ السور - اختيار/قراءة/استماع</title>

<!-- Bootstrap CSS -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">

<style>
  body { background: #fff; font-family: "Segoe UI", Tahoma, Arial; padding-top: 24px; }
  .controls button { min-width: 120px; }
  #ayahsBox { max-height: 55vh; overflow:auto; text-align: right; direction: rtl; }
  .ayah { padding:8px 12px; border-bottom:1px solid #eee; font-size: 20px; }
  .surah-title { font-weight:700; font-size:22px; }
  .history-list li { display:flex; justify-content:space-between; align-items:center; padding:6px 0; }
</style>
</head>
<body>

<div class="container">
  <div class="row justify-content-center mb-3">
    <div class="col-md-8 text-center">
      <h1 class="mb-2">قارئ السور</h1>
      <p class="text-muted mb-3">اختَر سورة عشوائية، اقرأها (تُسجَّل فقط عند الضغط على "قراءة") أو شغّل صوت البداية بصوت مشاري العفاسي.</p>

      <div class="controls d-flex justify-content-center gap-2 mb-3">
        <button id="btnChoose" class="btn btn-success">اختيار سورة</button>
        <button id="btnRead" class="btn btn-primary" disabled>قراءة</button>
        <button id="btnPlay" class="btn btn-warning" disabled>تشغيل الصوت</button>
      </div>

      <div id="currentCard" class="card shadow-sm">
        <div class="card-body text-end">
          <div id="currentInfo">
            <div class="surah-title" id="surahName">لم يتم اختيار سورة بعد</div>
            <div id="surahMeta" class="text-muted small"></div>
          </div>
          <audio id="audioPlayer" controls style="width:100%; display:none; margin-top:10px;"></audio>
        </div>
      </div>

    </div>
  </div>

  <div class="row">
    <!-- عرض الآيات -->
    <div class="col-md-8">
      <div class="card mb-3">
        <div class="card-header">نص السورة</div>
        <div class="card-body" id="ayahsBox">
          <div id="placeholder" class="text-muted">اضغط "قراءة" لعرض نص السورة هنا (وبذلك تُسجّل السورة في السجل).</div>
          <div id="ayahs" style="display:none;"></div>
        </div>
      </div>
    </div>

    <!-- السجل -->
    <div class="col-md-4">
      <div class="card mb-3">
        <div class="card-header d-flex justify-content-between align-items-center">
          <span>📜 السجل (السور المقروءة)</span>
          <button id="clearAll" class="btn btn-sm btn-outline-danger" title="حذف كل السجل">حذف الكل</button>
        </div>
        <div class="card-body">
          <ul id="historyList" class="list-unstyled history-list mb-0"></ul>
        </div>
      </div>

      <div class="card">
        <div class="card-header">تعليمات سريعة</div>
        <div class="card-body small text-muted text-end">
          • اختر سورة عشوائية بالضغط على "اختيار سورة".<br>
          • اضغط "قراءة" لعرض النص — عندها تُسجَّل السورة في السجل.<br>
          • اضغط "تشغيل الصوت" لتشغيل بداية السورة (لا يعمل تلقائياً).<br>
          • احذف أي سورة من السجل بالزر بجانبها.
        </div>
      </div>
    </div>
  </div>
</div>

<!-- Bootstrap JS (bundle) -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js"></script>

<script>
/*
  السلوك:
  - نستخدم api.alquran.cloud لجلب بيانات السور (النص + روابط الصوت لآيات مشاري العفاسي).
  - عند اختيار سورة: نجلب بيانات السورة (لكن لا نضيفها للسجل).
  - عند الضغط "قراءة": نظهر الآيات في الواجهة ونضيف السورة للسجل في localStorage.
  - عند الضغط "تشغيل الصوت": نشغّل رابط الصوت (نستخدم رابط أول آية كبداية).
  - السجل مخزن في localStorage كمصفوفة أسماء السور.
*/

const apiBase = "https://api.alquran.cloud/v1";
let allSurahs = [];            // قائمة أسماء السور (index => name)
let currentSurah = null;       // يحتوي على { number, englishName, name, ayahs: [...] }
const historyKey = "readSurahs_v1";

// عناصر الـ DOM
const btnChoose = document.getElementById("btnChoose");
const btnRead   = document.getElementById("btnRead");
const btnPlay   = document.getElementById("btnPlay");
const surahNameEl = document.getElementById("surahName");
const surahMetaEl = document.getElementById("surahMeta");
const ayahsEl = document.getElementById("ayahs");
const placeholder = document.getElementById("placeholder");
const audioPlayer = document.getElementById("audioPlayer");
const historyList = document.getElementById("historyList");
const clearAllBtn = document.getElementById("clearAll");

// تحميل قائمة السور عند البداية
async function loadSurahList() {
  try {
    const res = await fetch(`${apiBase}/surah`);
    const json = await res.json();
    if (json.code === 200 && Array.isArray(json.data)) {
      allSurahs = json.data; // كل عنصر يحتوي name, englishName, number, ayahs? (maybe not)
    } else {
      console.error("خطأ في جلب قائمة السور", json);
    }
  } catch (e) {
    console.error("فشل الاتصال بالـ API:", e);
  }
}

// عند اختيار سورة عشوائية: نجلب بيانات السورة (النسخة ar.alafasy) لكن لا نعرض النص حتى تضغط قراءة
async function chooseRandomSurah() {
  if (!allSurahs || allSurahs.length === 0) {
    await loadSurahList();
    if (!allSurahs || allSurahs.length === 0) {
      alert("تعذر الحصول على قائمة السور. تأكد من اتصال الإنترنت وحاول مرة أخرى.");
      return;
    }
  }

  // نختار عشوائيًا سورة
  const idx = Math.floor(Math.random() * allSurahs.length);
  const surahNum = allSurahs[idx].number;

  // جلب بيانات السورة (نسخة recitation/text مع صوت alafasy)
  // نستخدم /surah/{number}/ar.alafasy لكي نحصل على نص الآيات وروابط الصوت لكل آية
  try {
    btnChoose.disabled = true;
    btnChoose.textContent = "جاري التحضير...";
    const res = await fetch(`${apiBase}/surah/${surahNum}/ar.alafasy`);
    const json = await res.json();
    if (json.code === 200 && json.data) {
      currentSurah = json.data; // يحتوي name, englishName, number, ayahs (كل آية فيها text + audio)
      showCurrentBasic();
      btnRead.disabled = false;
      btnPlay.disabled = false;
      audioPlayer.style.display = "none";
      audioPlayer.src = "";
      audioPlayer.pause();
    } else {
      console.error("خطأ في جلب بيانات السورة:", json);
      alert("تعذر جلب بيانات السورة، حاول مرة أخرى.");
    }
  } catch (err) {
    console.error(err);
    alert("حصل خطأ أثناء الاتصال بالإنترنت.");
  } finally {
    btnChoose.disabled = false;
    btnChoose.textContent = "اختيار سورة";
  }
}

// عرض اسم السورة ومعلومات بسيطة
function showCurrentBasic() {
  if (!currentSurah) return;
  surahNameEl.textContent = `${currentSurah.number} - ${currentSurah.name}`;
  surahMetaEl.textContent = `الآيات: ${currentSurah.numberOfAyahs ?? currentSurah.ayahs.length || currentSurah.ayahs.length}`;
  // لا نعرض الآيات هنا; تُعرض عند الضغط على "قراءة"
  placeholder.style.display = "block";
  ayahsEl.style.display = "none";
  ayahsEl.innerHTML = "";
}

// عند الضغط على قراءة: نعرض كل الآيات ونضيف اسم السورة للسجل (إذا لم يكن موجودًا)
function readCurrentSurah() {
  if (!currentSurah) {
    alert("لم تختَر سورة بعد.");
    return;
  }

  // عرض الآيات
  placeholder.style.display = "none";
  ayahsEl.style.display = "block";
  ayahsEl.innerHTML = "";
  currentSurah.ayahs.forEach(a => {
    const div = document.createElement("div");
    div.className = "ayah";
    // نعرض رقم الآية ثم النص (الآية بالخط العربي المتوفر من API)
    div.innerHTML = `<span class="text-muted small">(${a.numberInSurah})</span> <span>${a.text}</span>`;
    ayahsEl.appendChild(div);
  });

  // تسجيل السورة في السجل (LocalStorage) إن لم تكن مسجلة
  let saved = JSON.parse(localStorage.getItem(historyKey) || "[]");
  const nameKey = `${currentSurah.number} - ${currentSurah.name}`;
  if (!saved.includes(nameKey)) {
    saved.push(nameKey);
    localStorage.setItem(historyKey, JSON.stringify(saved));
    renderHistory();
  }
}

// عند الضغط تشغيل الصوت: نشغّل رابط الصوت لأول آية (ممكن تطوير لتشغيل كامل السورة)
function playAudio() {
  if (!currentSurah) { alert("لم تختَر سورة بعد."); return; }
  // نأخذ رابط أول آية (مفيد كبداية السورة). حقل audio موجود في كل ayah من الـ API.
  const firstAudio = currentSurah.ayahs && currentSurah.ayahs.length ? currentSurah.ayahs[0].audio : null;
  if (!firstAudio) {
    alert("لا يوجد رابط صوتي متاح للسورة.");
    return;
  }
  audioPlayer.src = firstAudio;
  audioPlayer.style.display = "block";
  audioPlayer.play().catch(err => {
    // بعض المتصفحات تمنع التشغيل التلقائي؛ المستخدم يجب أن يتفاعل.
    console.warn("خطأ تشغيل الصوت:", err);
  });
}

// عرض سجل السور المقروءة مع أزرار حذف لكل واحدة
function renderHistory() {
  const saved = JSON.parse(localStorage.getItem(historyKey) || "[]");
  historyList.innerHTML = "";
  if (saved.length === 0) {
    historyList.innerHTML = `<li class="text-muted">لا توجد سور مقروءة بعد.</li>`;
    return;
  }
  saved.forEach((item, idx) => {
    const li = document.createElement("li");
    const span = document.createElement("span");
    span.textContent = item;
    const delBtn = document.createElement("button");
    delBtn.className = "btn btn-sm btn-outline-danger ms-2";
    delBtn.textContent = "حذف";
    delBtn.onclick = () => {
      deleteFromHistory(idx);
    };
    li.appendChild(span);
    li.appendChild(delBtn);
    historyList.appendChild(li);
  });
}

function deleteFromHistory(index) {
  let saved = JSON.parse(localStorage.getItem(historyKey) || "[]");
  if (index >= 0 && index < saved.length) {
    saved.splice(index, 1);
    localStorage.setItem(historyKey, JSON.stringify(saved));
    renderHistory();
  }
}

function clearAllHistory() {
  if (!confirm("هل تريد حذف كل السجل؟")) return;
  localStorage.removeItem(historyKey);
  renderHistory();
}

// ربط الأزرار
btnChoose.addEventListener("click", chooseRandomSurah);
btnRead.addEventListener("click", readCurrentSurah);
btnPlay.addEventListener("click", playAudio);
clearAllBtn.addEventListener("click", clearAllHistory);

// تشغيل أولي: تحميل قائمة السور وعرض السجل
(async function init() {
  await loadSurahList();
  renderHistory();
})();
</script>
</body>
</html>
