# CodeForge Plugin Development Guide

Panduan lengkap untuk membuat plugin CodeForge (`.cfplugin`).

---

## Daftar Isi

- [Struktur Plugin](#struktur-plugin)
- [manifest.json](#manifestjson)
- [main.js — Plugin API](#mainjs--plugin-api)
  - [editor](#codeforgeditor)
  - [ui](#codeforgeui)
  - [commands](#codeforgecommands)
  - [events](#codeforgeevents)
  - [http](#codeforgehttp)
- [Permissions](#permissions)
- [Cara Paket & Distribusi](#cara-paket--distribusi)
- [Contoh Plugin Lengkap](#contoh-plugin-lengkap)

---

## Struktur Plugin

Plugin CodeForge adalah file ZIP biasa dengan ekstensi `.cfplugin`, berisi minimal 2 file:

```
my-plugin.cfplugin   ← rename dari .zip ke .cfplugin
├── manifest.json    ← wajib ada
└── main.js          ← entry point (sesuai field "entry" di manifest)
```

---

## manifest.json

```json
{
  "id": "com.namaKamu.namaPlugin",
  "name": "Nama Plugin",
  "version": "1.0.0",
  "description": "Deskripsi singkat plugin kamu",
  "author": "Nama Kamu",
  "entry": "main.js",
  "permissions": ["editor"]
}
```

| Field | Wajib | Keterangan |
|---|---|---|
| `id` | ✅ | ID unik, hanya boleh huruf, angka, `.`, `-`, `_` |
| `name` | ✅ | Nama yang tampil di Plugin Manager |
| `version` | ✅ | Format bebas, contoh: `"1.0.0"` |
| `description` | - | Deskripsi singkat |
| `author` | - | Nama pembuat |
| `entry` | - | Nama file JS utama (default: `"main.js"`) |
| `permissions` | - | Array permission yang dibutuhkan (lihat [Permissions](#permissions)) |

---

## main.js — Plugin API

Di dalam `main.js`, kamu punya akses ke objek `CodeForge` yang berisi semua API plugin.

> **Catatan:** Jangan bungkus kode kamu dalam IIFE sendiri — CodeForge sudah otomatis membungkusnya dalam sandbox yang aman.

```js
// ✅ Benar
CodeForge.ui.showToast('Hello!');

// ❌ Jangan lakukan ini
(function() {
    CodeForge.ui.showToast('Hello!');
})();
```

---

### `CodeForge.editor`

Membaca dan memanipulasi konten editor. **Membutuhkan permission `editor`.**

> **Pengecualian:** `getLanguage()` dan `getCursorPosition()` tidak membutuhkan permission `editor`.

```js
// Ambil seluruh isi file yang sedang dibuka
var content = CodeForge.editor.getContent();
// → string

// Ganti seluruh isi editor dengan teks baru
CodeForge.editor.setContent('teks baru');

// Sisipkan teks di posisi kursor saat ini
CodeForge.editor.insertAtCursor('teks yang disisipkan');

// Ambil teks yang sedang di-select user
var selected = CodeForge.editor.getSelection();
// → string (kosong jika tidak ada seleksi)

// Ganti teks yang sedang di-select dengan teks baru
CodeForge.editor.replaceSelection('teks pengganti');

// Ambil bahasa file yang sedang dibuka
var lang = CodeForge.editor.getLanguage();
// → contoh: 'json', 'javascript', 'python', 'html', 'plaintext', dll.
// Tidak butuh permission 'editor'

// Ambil posisi kursor saat ini
var pos = CodeForge.editor.getCursorPosition();
// → { line: number, col: number }
// Tidak butuh permission 'editor'
```

**Language ID yang tersedia:**

| Ekstensi | Language ID |
|---|---|
| `.html`, `.htm` | `html` |
| `.css` | `css` |
| `.js`, `.mjs` | `javascript` |
| `.ts` | `typescript` |
| `.json` | `json` |
| `.md` | `markdown` |
| `.py` | `python` |
| `.kt`, `.kts` | `kotlin` |
| `.java` | `java` |
| `.sh`, `.bash` | `shell` |
| `.yaml`, `.yml` | `yaml` |
| `.xml`, `.svg` | `xml` |
| `.sql` | `sql` |
| `.php` | `php` |
| `.rs` | `rust` |
| `.go` | `go` |
| `.scss`, `.sass` | `scss` |
| lainnya | `plaintext` |

---

### `CodeForge.ui`

Menampilkan UI ke user.

```js
// Tampilkan pesan singkat (snackbar) di bagian bawah layar
CodeForge.ui.showToast('Pesan kamu di sini');

// Tambahkan tombol ke toolbar editor
// commandId harus sama dengan yang didaftarkan di CodeForge.commands.register()
CodeForge.ui.addToolbarButton('commandId', 'Label Tombol');

// Tampilkan dialog input teks ke user
// callback dipanggil dengan string hasil input (kosong jika user cancel)
CodeForge.ui.showInputDialog('Judul Dialog', 'Placeholder hint', function(value) {
    if (value) CodeForge.ui.showToast('Input: ' + value);
});
```

**Kapan harus panggil `addToolbarButton`?**

Panggil di dalam listener `fileOpen` — bukan langsung di root `main.js`. Ini karena toolbar direset setiap kali file baru dibuka.

```js
// ✅ Benar — button muncul setelah file terbuka
CodeForge.events.on('fileOpen', function(data) {
    CodeForge.ui.addToolbarButton('myCommand', 'Klik Aku');
});

// ❌ Kurang tepat — button bisa hilang saat ganti file
CodeForge.ui.addToolbarButton('myCommand', 'Klik Aku');
```

Kalau mau button hanya muncul untuk file tertentu, filter berdasarkan `data.language`:

```js
CodeForge.events.on('fileOpen', function(data) {
    if (data.language === 'json') {
        CodeForge.ui.addToolbarButton('format', '{ }');
    }
});
```

---

### `CodeForge.commands`

Mendaftarkan fungsi yang dipanggil saat tombol toolbar diklik.

```js
CodeForge.commands.register('commandId', function() {
    // kode yang dijalankan saat tombol diklik
    CodeForge.ui.showToast('Tombol diklik!');
});
```

`commandId` di `commands.register` harus sama persis dengan `commandId` di `addToolbarButton`.

---

### `CodeForge.events`

Mendengarkan event yang terjadi di editor.

```js
// Mulai mendengarkan event
CodeForge.events.on('namaEvent', function(data) {
    // handler
});

// Berhenti mendengarkan event
CodeForge.events.off('namaEvent', handler);
```

**Event yang tersedia:**

| Event | Data | Keterangan |
|---|---|---|
| `fileOpen` | `{ fileName, language }` | File baru dibuka di editor |
| `contentChange` | `{}` | Isi editor berubah — fire setiap keystroke |
| `cursorMove` | `{ line, col }` | Posisi kursor berubah |
| `editorFocus` | `{}` | Editor mendapat fokus |
| `fileSave` | `{ fileName }` | File berhasil disimpan |

> ⚠️ **Peringatan `contentChange`:** Event ini fire **setiap ketukan keyboard**, bukan debounced. Jangan panggil `showToast` langsung di dalamnya — snackbar akan numpuk di queue dan terus muncul lama setelah user berhenti ngetik, membuat editor terasa broken.
>
> ```js
> // ❌ Jangan lakukan ini — spam snackbar tiap keystroke
> CodeForge.events.on('contentChange', function() {
>     CodeForge.ui.showToast('Words: ' + countWords());
> });
>
> // ✅ Gunakan tombol untuk aksi yang butuh feedback ke user
> CodeForge.commands.register('countNow', function() {
>     CodeForge.ui.showToast('Words: ' + countWords());
> });
>
> CodeForge.events.on('fileOpen', function() {
>     CodeForge.ui.addToolbarButton('countNow', 'Count');
> });
> ```
>
> `contentChange` cocok untuk operasi ringan di background yang tidak perlu feedback langsung ke user (misalnya update state internal plugin).

> ⚠️ **Peringatan timer di `contentChange`:** Kalau kamu pakai `setTimeout` di dalam handler `contentChange`, **selalu cancel timer lama di handler `fileOpen`**. Setiap kali file baru dibuka, plugin di-reinject ulang dari awal — variabel `idleTimer`-mu reset ke nilai awal, sehingga timer dari sesi file sebelumnya tidak bisa di-cancel dan tetap jalan di background.
>
> ```js
> // ❌ Timer lama tidak di-cancel saat ganti file
> var idleTimer = null;
>
> CodeForge.events.on('contentChange', function() {
>     clearTimeout(idleTimer);
>     idleTimer = setTimeout(function() {
>         CodeForge.ui.showToast('reminder');
>     }, 3000);
> });
>
> // ✅ Cancel timer di fileOpen sebelum state di-reset
> CodeForge.events.on('fileOpen', function() {
>     clearTimeout(idleTimer); // ← wajib ada
>     idleTimer = null;
>     // reset state lainnya di sini...
> });
> ```

---

### `CodeForge.http`

Melakukan HTTP request ke server eksternal. **Membutuhkan permission `http`. Hanya mendukung HTTPS.**

```js
// GET request
CodeForge.http.get('https://api.example.com/data', {
    'Authorization': 'Bearer token123'
}).then(function(responseBody) {
    // responseBody adalah string
    var data = JSON.parse(responseBody);
    CodeForge.ui.showToast('Berhasil: ' + data.message);
}).catch(function(err) {
    CodeForge.ui.showToast('Error: ' + err.message);
});

// POST request
CodeForge.http.post('https://api.example.com/submit', {
    key: 'value'
}, {
    'Content-Type': 'application/json'
}).then(function(responseBody) {
    CodeForge.ui.showToast('Terkirim!');
}).catch(function(err) {
    CodeForge.ui.showToast('Gagal: ' + err.message);
});
```

> **Penting:** HTTP (non-HTTPS) tidak diizinkan dan akan langsung ditolak.

> **Jangan set `Content-Type` manual di header POST.** Platform sudah otomatis set `Content-Type: application/json; charset=utf-8` dari sisi Kotlin. Kalau kamu tambahkan lagi via header, OkHttp akan kirim dua `Content-Type` sekaligus.
>
> ```js
> // ❌ Redundant — Content-Type sudah di-set otomatis
> CodeForge.http.post(url, { key: 'value' }, { 'Content-Type': 'application/json' });
>
> // ✅ Cukup begini
> CodeForge.http.post(url, { key: 'value' }, {});
> ```

> ⚠️ **Plugin HTTP async — wajib pakai loading guard.** Tanpa guard, klik tombol berkali-kali sebelum response balik akan mengirim banyak request paralel dan hasilnya bisa tabrakan di editor.
>
> ```js
> var isProcessing = false;
>
> CodeForge.commands.register('myCommand', function() {
>     if (isProcessing) { CodeForge.ui.showToast('Tunggu sebentar...'); return; }
>     var sel = CodeForge.editor.getSelection();
>     if (!sel) { CodeForge.ui.showToast('Pilih teks dulu'); return; }
>
>     isProcessing = true;
>     CodeForge.http.post('https://api.example.com', { text: sel }, {})
>         .then(function(body) {
>             // proses response
>         })
>         .catch(function() {
>             CodeForge.ui.showToast('Error');
>         })
>         .then(function() {
>             isProcessing = false; // selalu reset di akhir
>         });
> });
> ```

> ⚠️ **`replaceSelection` setelah async tidak menjamin posisi yang benar.** `replaceSelection` selalu menarget selection yang aktif *saat dipanggil*, bukan saat tombol diklik. Kalau user memindahkan kursor selagi menunggu response, teks hasil akan masuk di posisi yang salah. Ini adalah keterbatasan platform saat ini — belum ada API `replaceRange(range, text)` untuk menarget range yang sudah disimpan sebelumnya.

---

## Permissions

Deklarasikan permission di `manifest.json` sesuai API yang kamu butuhkan.

| Permission | Untuk apa |
|---|---|
| `editor` | Menggunakan `CodeForge.editor.getContent/setContent/getSelection/replaceSelection/insertAtCursor` |
| `http` | Menggunakan `CodeForge.http.get/post` |

Jika plugin memanggil API yang butuh permission tapi tidak dideklarasikan, plugin akan **crash dan otomatis dinonaktifkan** oleh CodeForge.

```json
{
  "permissions": ["editor", "http"]
}
```

---

## Cara Paket & Distribusi

**1. Buat file ZIP berisi `manifest.json` dan `main.js`**

```bash
zip my-plugin.zip manifest.json main.js
mv my-plugin.zip my-plugin.cfplugin
```

Di Windows: select kedua file → klik kanan → Send to → Compressed folder → rename ke `.cfplugin`.

**2. Upload ke GitHub Releases**

Buat Release baru di repo kamu, upload file `.cfplugin` sebagai asset.

**3. Daftarkan ke Plugin Store CodeForge**

Buka pull request ke repo [Codeforge-plugin](https://github.com/az925-crypto/Codeforge-plugin) dengan menambahkan entry ke `plugins.json`:

```json
{
  "id": "com.namaKamu.namaPlugin",
  "name": "Nama Plugin",
  "version": "1.0.0",
  "description": "Deskripsi singkat",
  "author": "Nama Kamu",
  "category": "utility",
  "downloadUrl": "https://github.com/namaKamu/nama-repo/releases/latest/download/nama-plugin.cfplugin"
}
```

**Category yang tersedia:** `formatter`, `utility`, `theme`, `git`, `language`

---

## Contoh Plugin Lengkap

### Case Converter — Konversi teks yang di-select

**manifest.json**
```json
{
  "id": "com.example.case-converter",
  "name": "Case Converter",
  "version": "1.0.0",
  "description": "Konversi teks ke UPPER, lower, atau Title Case",
  "author": "Nama Kamu",
  "entry": "main.js",
  "permissions": ["editor"]
}
```

**main.js**
```js
CodeForge.commands.register('toUpper', function() {
    var sel = CodeForge.editor.getSelection();
    if (!sel) { CodeForge.ui.showToast('Pilih teks dulu'); return; }
    CodeForge.editor.replaceSelection(sel.toUpperCase());
});

CodeForge.commands.register('toLower', function() {
    var sel = CodeForge.editor.getSelection();
    if (!sel) { CodeForge.ui.showToast('Pilih teks dulu'); return; }
    CodeForge.editor.replaceSelection(sel.toLowerCase());
});

CodeForge.events.on('fileOpen', function() {
    CodeForge.ui.addToolbarButton('toUpper', 'AA');
    CodeForge.ui.addToolbarButton('toLower', 'aa');
});
```

---

## Catatan Pengembangan Platform

> Bagian ini ditujukan untuk developer yang mengembangkan CodeForge itu sendiri, bukan untuk pembuat plugin.

### Listener accumulation saat ganti file

Setiap kali file dibuka, seluruh plugin di-reinject ulang via `injectPlugins()`. Ini menyebabkan handler baru didaftarkan ke `__cfListeners` tanpa menghapus handler lama. Setelah beberapa kali ganti file, satu event bisa men-trigger banyak handler duplikat dari sesi sebelumnya.

**Solusi:** Reset `__cfListeners` sebelum `emitFileOpen` dipanggil, misalnya dengan menambahkan fungsi `__cfResetListeners()` yang dipanggil dari `MonacoWebView.emitFileOpen()` sebelum emit event-nya.

```js
// Di index.html — tambahkan fungsi ini
function __cfResetListeners() {
    __cfListeners = {};
}
```

```kotlin
// Di MonacoWebView.kt — panggil reset sebelum emit
fun emitFileOpen(fileName: String, language: String) {
    eval("__cfResetListeners()")
    val safeFile = fileName.replace("'", "\\'")
    val safeLang = language.replace("'", "\\'")
    eval("if(typeof __cfEmitFileOpen==='function') __cfEmitFileOpen('$safeFile','$safeLang')")
}
```

### HTTP response body — UTF-8 garbled untuk non-ASCII

`resolveHttpCallback` di `MonacoWebView.kt` mengirim body ke JS via `atob()` saja, tanpa `decodeURIComponent(escape(...))`. Akibatnya karakter non-ASCII di response body (huruf aksen, karakter Asia, emoji) akan muncul garbled di sisi plugin. Berbeda dengan `setContentBase64` dan `insertAtCursorBase64` yang sudah pakai pattern yang benar.

**Solusi di `MonacoWebView.kt`:**

```kotlin
// Sebelum
eval("if(typeof __cfHttpResult==='function') __cfHttpResult('$safeId',$success,atob('$safeBody'))")

// Sesudah
eval("if(typeof __cfHttpResult==='function') __cfHttpResult('$safeId',$success,decodeURIComponent(escape(atob('$safeBody'))))")
```

### `replaceSelection` tidak mendukung saved range

`replaceSelection` selalu menarget `editor.getSelection()` saat dipanggil. Tidak ada cara untuk menyimpan range dari sisi plugin dan menggunakannya setelah operasi async selesai. Ini menjadi masalah nyata untuk plugin berbasis HTTP — user bisa memindahkan kursor selagi menunggu response.

**Solusi jangka panjang:** Tambahkan API `replaceRange(line, col, endLine, endCol, text)` di plugin context, atau expose `savedSelection` yang bisa di-pass kembali ke `replaceAtRange`.
