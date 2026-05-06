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
  - [storage](#codeforgestorage)
  - [clipboard](#codeforgestorage)
  - [file](#codeforgestorage)
  - [share](#codeforgestorage)
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

> **Pengecualian:** `getLanguage()` dan `getCursorPosition()` tidak membutuhkan permission `editor`. Begitu juga `file.getLanguage()` — keduanya bisa dipanggil tanpa permission apapun.

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

// Ambil jumlah baris di editor
var count = CodeForge.editor.getLineCount();
// → number

// Ambil isi baris tertentu (1-indexed)
var line = CodeForge.editor.getLine(5);
// → string

// Ambil konten dalam range tertentu
var text = CodeForge.editor.getContentInRange({ startLine: 1, startCol: 1, endLine: 3, endCol: 10 });
// → string

// Set selection ke range tertentu
CodeForge.editor.setSelection(startLine, startCol, endLine, endCol);

// Scroll editor ke baris tertentu
CodeForge.editor.scrollToLine(10);

// Replace semua kemunculan string (literal, bukan regex)
CodeForge.editor.replaceAll('foo', 'bar');

// Ambil kata di posisi kursor saat ini
var word = CodeForge.editor.getWordAtCursor();
// → string

// Set editor menjadi read-only atau writable
CodeForge.editor.setReadOnly(true);
CodeForge.editor.setReadOnly(false);

// Tambah dekorasi (highlight baris)
var decorId = CodeForge.editor.addDecoration(startLine, endLine, 'my-css-class');
// → string | null (id dekorasi)

// Hapus dekorasi berdasarkan id
CodeForge.editor.removeDecoration(decorId);
```

#### Simpan dan pakai selection untuk operasi async

`replaceSelection` selalu menarget selection yang aktif *saat dipanggil*. Jika kamu menunggu HTTP response, user bisa memindahkan kursor sebelum response balik. Gunakan `saveSelection()` + `replaceRange()` untuk menghindari ini:

```js
CodeForge.commands.register('myCommand', function() {
    var savedRange = CodeForge.editor.saveSelection();
    // → { startLine, startCol, endLine, endCol } | null

    CodeForge.http.post('https://api.example.com', { text: CodeForge.editor.getSelection() }, {})
        .then(function(body) {
            // Pakai range yang sudah disimpan — tidak terpengaruh pergerakan kursor
            CodeForge.editor.replaceRange(savedRange, body);
        });
});
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

// Tampilkan dialog konfirmasi (OK / Cancel)
// callback dipanggil dengan boolean: true jika user tekan OK
CodeForge.ui.showConfirm('Judul', 'Yakin mau hapus?', function(confirmed) {
    if (confirmed) CodeForge.ui.showToast('Dihapus!');
});

// Tampilkan dialog pilihan (dropdown / list)
// options: array of string
// callback dipanggil dengan string yang dipilih, atau null jika cancel
CodeForge.ui.showSelect('Pilih Format', ['JSON', 'YAML', 'TOML'], function(value) {
    if (value) CodeForge.ui.showToast('Dipilih: ' + value);
});

// Tampilkan indikator loading
CodeForge.ui.showProgress('Memproses...');

// Sembunyikan indikator loading
CodeForge.ui.hideProgress();
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
| `fileOpen` | `{ fileName, language, filePath }` | File baru dibuka di editor |
| `contentChange` | `{}` | Isi editor berubah — di-debounce 300ms |
| `cursorMove` | `{ line, col }` | Posisi kursor berubah |
| `editorFocus` | `{}` | Editor mendapat fokus |
| `fileSave` | `{ fileName }` | File berhasil disimpan |

> ⚠️ **Peringatan `contentChange`:** Event ini di-debounce 300ms, bukan fire setiap keystroke mentah. Tetap hindari memanggil `showToast` langsung di dalamnya — gunakan tombol untuk aksi yang butuh feedback ke user.
>
> ```js
> // ❌ Jangan lakukan ini
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
}, {}).then(function(responseBody) {
    CodeForge.ui.showToast('Terkirim!');
}).catch(function(err) {
    CodeForge.ui.showToast('Gagal: ' + err.message);
});
```

> **Penting:** HTTP (non-HTTPS) tidak diizinkan dan akan langsung ditolak.

> **Jangan set `Content-Type` manual di header POST.** Platform sudah otomatis set `Content-Type: application/json; charset=utf-8`. Kalau kamu tambahkan lagi via header, OkHttp akan kirim dua `Content-Type` sekaligus.
>
> ```js
> // ❌ Redundant
> CodeForge.http.post(url, { key: 'value' }, { 'Content-Type': 'application/json' });
>
> // ✅ Cukup begini
> CodeForge.http.post(url, { key: 'value' }, {});
> ```

> ⚠️ **Plugin HTTP async — wajib pakai loading guard.** Tanpa guard, klik tombol berkali-kali sebelum response balik akan mengirim banyak request paralel.
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
>     CodeForge.ui.showProgress('Memproses...');
>     CodeForge.http.post('https://api.example.com', { text: sel }, {})
>         .then(function(body) {
>             // proses response
>         })
>         .catch(function() {
>             CodeForge.ui.showToast('Error');
>         })
>         .then(function() {
>             isProcessing = false;
>             CodeForge.ui.hideProgress();
>         });
> });
> ```

---

### `CodeForge.storage`

Menyimpan data persisten per-plugin. Data disimpan di SharedPreferences dan tetap ada setelah app ditutup. **Tidak membutuhkan permission.**

```js
// Simpan nilai (key dan value harus string)
CodeForge.storage.set('myKey', 'myValue');

// Ambil nilai — async via callback
// callback dipanggil dengan string jika key ada, atau null jika tidak ada
CodeForge.storage.get('myKey', function(value) {
    if (value !== null) {
        CodeForge.ui.showToast('Nilai: ' + value);
    }
});

// Hapus key
CodeForge.storage.remove('myKey');
```

> **Catatan:** `storage.set` dan `storage.remove` sinkron (fire-and-forget). `storage.get` async via callback.

---

### `CodeForge.clipboard`

Membaca dan menulis ke clipboard sistem. **Tidak membutuhkan permission.**

```js
// Tulis teks ke clipboard
CodeForge.clipboard.write('teks yang disalin');

// Baca teks dari clipboard — async via callback
CodeForge.clipboard.read(function(text) {
    CodeForge.ui.showToast('Clipboard: ' + text);
});
```

---

### `CodeForge.file`

Mengakses informasi file yang sedang dibuka dan file-file di project. **Tidak membutuhkan permission.**

```js
// Ambil nama file yang sedang dibuka
var name = CodeForge.file.getName();
// → contoh: 'index.js'

// Ambil ekstensi file (tanpa titik)
var ext = CodeForge.file.getExtension();
// → contoh: 'js'

// Ambil path absolut file yang sedang dibuka
var path = CodeForge.file.getPath();
// → contoh: '/storage/emulated/0/MyProject/index.js'

// Ambil language ID file yang sedang dibuka
var lang = CodeForge.file.getLanguage();
// → contoh: 'javascript'

// Ambil daftar semua file di project yang sedang aktif
// callback dipanggil dengan array of { name, path, language }
CodeForge.file.getProjectFiles(function(files) {
    files.forEach(function(f) {
        CodeForge.ui.showToast(f.name + ' — ' + f.language);
    });
});

// Baca isi file dari path tertentu di project
// callback dipanggil dengan (success: boolean, content: string)
CodeForge.file.readFile('/storage/emulated/0/MyProject/utils.js', function(success, content) {
    if (success) {
        CodeForge.ui.showToast('File dibaca: ' + content.length + ' karakter');
    } else {
        CodeForge.ui.showToast('Gagal baca: ' + content); // content berisi pesan error
    }
});
```

> **Catatan:** `file.save()` tersedia di API tapi belum berfungsi di versi ini.

---

### `CodeForge.share`

Membagikan teks ke aplikasi lain via Android share sheet. **Tidak membutuhkan permission.**

```js
// Bagikan teks ke aplikasi lain
CodeForge.share.text('Isi yang ingin dibagikan', 'Judul Share (opsional)');
```

---

## Permissions

Deklarasikan permission di `manifest.json` sesuai API yang kamu butuhkan.

| Permission | Untuk apa |
|---|---|
| `editor` | Menggunakan `CodeForge.editor.*` (kecuali `getLanguage` dan `getCursorPosition`) |
| `http` | Menggunakan `CodeForge.http.get` / `post` |

> **`storage`, `clipboard`, `file`, dan `share` tidak membutuhkan permission** — langsung bisa dipakai tanpa deklarasi.

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

### HTTP Plugin dengan Loading Guard dan saveSelection

**manifest.json**
```json
{
  "id": "com.example.ai-helper",
  "name": "AI Helper",
  "version": "1.0.0",
  "description": "Kirim seleksi ke API dan replace hasilnya",
  "author": "Nama Kamu",
  "entry": "main.js",
  "permissions": ["editor", "http"]
}
```

**main.js**
```js
var isProcessing = false;

CodeForge.commands.register('runAi', function() {
    if (isProcessing) { CodeForge.ui.showToast('Tunggu sebentar...'); return; }
    var sel = CodeForge.editor.getSelection();
    if (!sel) { CodeForge.ui.showToast('Pilih teks dulu'); return; }

    // Simpan selection sebelum request async
    var savedRange = CodeForge.editor.saveSelection();

    isProcessing = true;
    CodeForge.ui.showProgress('Memproses...');

    CodeForge.http.post('https://api.example.com/fix', { code: sel }, {})
        .then(function(body) {
            // Pakai savedRange agar tidak salah posisi walaupun kursor sudah pindah
            CodeForge.editor.replaceRange(savedRange, body);
        })
        .catch(function(err) {
            CodeForge.ui.showToast('Error: ' + err.message);
        })
        .then(function() {
            isProcessing = false;
            CodeForge.ui.hideProgress();
        });
});

CodeForge.events.on('fileOpen', function() {
    CodeForge.ui.addToolbarButton('runAi', '✨ Fix');
});
```
