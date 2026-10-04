# Arsitektur Monolith vs Microservices dengan Flask Python

Praktikum mata kuliah **Paradigma Sistem untuk IT** yang membandingkan arsitektur **Monolith** dan **Microservices** melalui studi kasus sederhana: aplikasi toko buku dengan dua fitur, yaitu **katalog buku** dan **pemesanan**.

| | |
|---|---|
| **Nama** | Gibran Surya Gunanda |
| **NIM** | 2024903430084 |
| **Kelas** | TRKJ 3C |
| **Program Studi** | Teknologi Rekayasa Komputer Jaringan |
| **Institusi** | Politeknik Negeri Lhokseumawe |
| **Dosen Pengampu** | Muhammad Reza Zulma, S.ST., M.Sc |
| **Bahasa / Framework** | Python 3 / Flask |
| **Lingkungan** | Windows, PowerShell, Virtual Environment (venv) |

---

## Tujuan Praktikum

- Memahami konsep dasar aplikasi Monolith dan Microservices.
- Mampu membangun backend sederhana menggunakan Flask Python.
- Menganalisis kelebihan dan kekurangan masing-masing arsitektur melalui praktik langsung.

---

## Struktur Proyek

```
paradigma_sistem/
├── monolith_app.py     # Aplikasi Monolith (port 5000)
├── book_service.py     # Microservice Buku (port 5001)
├── order_service.py    # Microservice Pesanan (port 5002)
└── README.md
```

---

## Langkah 1 — Persiapan Lingkungan

Pastikan Python 3 sudah terpasang:

```powershell
python --version
```

(Opsional, disarankan) Buat dan aktifkan virtual environment:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

Pasang library yang dibutuhkan:

```powershell
pip install Flask requests
```

---

## Langkah 2 — Aplikasi Monolith

Seluruh fitur (Buku dan Pesanan) berada dalam **satu file**, satu proses, dan berjalan di **port 5000**. Fitur Pesanan mengakses data buku secara langsung lewat variabel global.

**`monolith_app.py`**

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

# Database bohongan (In-memory)
books = [{"id": 1, "title": "Belajar Flask", "stock": 5}]
orders = []

# --- FITUR BUKU ---
@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)

# --- FITUR PESANAN ---
@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')
    # Logika bisnis: Cek stok buku langsung dari variabel global
    for b in books:
        if b['id'] == book_id and b['stock'] > 0:
            b['stock'] -= 1
            order = {"id": len(orders)+1, "book_id": book_id, "status": "berhasil"}
            orders.append(order)
            return jsonify(order), 201
    return jsonify({"error": "Buku tidak ditemukan atau stok habis"}), 400

if __name__ == '__main__':
    # Aplikasi berjalan di port 5000
    app.run(port=5000, debug=True)
```

**Menjalankan server:**

```powershell
python monolith_app.py
```

**Pengujian** (buka terminal baru):

```powershell
# Ambil daftar buku
curl http://localhost:5000/books

# Buat pesanan baru
curl.exe -X POST -H "Content-Type: application/json" -d '{\"book_id\": 1}' http://localhost:5000/orders

# Cek ulang stok (seharusnya berkurang dari 5 menjadi 4)
curl http://localhost:5000/books
```

**Hasil yang diharapkan:** stok buku langsung berkurang menjadi `4` karena fitur Pesanan dan fitur Buku berbagi memori dan proses yang sama.

---

## Langkah 3 — Aplikasi Microservices

Sistem yang sama dipecah menjadi **dua layanan terpisah**. Order Service tidak bisa lagi mengakses data buku secara langsung, sehingga harus melakukan **HTTP request** ke Book Service.

### 3.1 Book Service (port 5001)

**`book_service.py`**

```python
from flask import Flask, jsonify

app = Flask(__name__)

# Database khusus Book Service
books = [{"id": 1, "title": "Belajar Flask", "stock": 5}]

@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)

@app.route('/books/<int:book_id>', methods=['GET'])
def get_book(book_id):
    for b in books:
        if b['id'] == book_id:
            return jsonify(b)
    return jsonify({"error": "Not found"}), 404

if __name__ == '__main__':
    # Book service berjalan di port 5001
    app.run(port=5001, debug=True)
```

### 3.2 Order Service (port 5002)

**`order_service.py`**

```python
from flask import Flask, jsonify, request
import requests

app = Flask(__name__)
orders = []
BOOK_SERVICE_URL = "http://localhost:5001"  # Alamat Book Service

@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')

    # Komunikasi antar service: Tanya Book Service apakah buku ada
    try:
        response = requests.get(f"{BOOK_SERVICE_URL}/books/{book_id}")
        if response.status_code == 200:
            book_data = response.json()
            if book_data['stock'] > 0:
                order = {"id": len(orders)+1, "book_id": book_id, "status": "berhasil"}
                orders.append(order)
                return jsonify(order), 201

        return jsonify({"error": "Buku tidak tersedia"}), 400
    except requests.exceptions.ConnectionError:
        return jsonify({"error": "Book Service sedang down!"}), 500

if __name__ == '__main__':
    # Order service berjalan di port 5002
    app.run(port=5002, debug=True)
```

### 3.3 Menjalankan kedua layanan

Gunakan **tiga terminal** berbeda:

```powershell
# Terminal 1
python book_service.py

# Terminal 2
python order_service.py

# Terminal 3 - uji pesanan
Invoke-RestMethod -Uri "http://localhost:5002/orders" -Method Post -ContentType "application/json" -Body '{"book_id": 1}'
```

**Hasil yang diharapkan:**

```
book_id id status
------- -- ------
      1  1 berhasil
```

Log pada terminal Book Service akan menampilkan permintaan masuk `GET /books/1`, membuktikan bahwa kedua layanan berkomunikasi melalui jaringan, bukan pemanggilan fungsi langsung.

---

## Langkah 4 — Eksperimen Fault Isolation

Tujuan: menguji ketahanan sistem ketika salah satu layanan mati.

1. Matikan Book Service dengan menekan `Ctrl+C` di **Terminal 1**.
2. Kirim kembali permintaan pesanan ke Order Service:

   ```powershell
   curl.exe -X POST -H "Content-Type: application/json" -d '{\"book_id\": 1}' http://localhost:5002/orders
   ```

3. Amati hasilnya.

**Hasil yang diharapkan:**

```json
{
  "error": "Book Service sedang down!"
}
```

Order Service **tidak ikut crash**. Ia menangkap `ConnectionError` dan mengembalikan respons error yang terkontrol. Inilah konsep **fault isolation**.

---

## Perbandingan Arsitektur

| Aspek | Monolith | Microservices |
|---|---|---|
| Struktur kode | Satu codebase, satu proses | Terpisah per layanan (Book Service, Order Service) |
| Komunikasi antar fitur | Pemanggilan fungsi/variabel langsung | HTTP Request antar service |
| Deployment | Satu kali deploy untuk seluruh aplikasi | Deploy independen tiap service |
| Skalabilitas | Scale seluruh aplikasi sekaligus | Scale selektif per service |
| Fault isolation | Satu bug fatal bisa menjatuhkan seluruh sistem | Kegagalan satu service tidak mematikan service lain |
| Latensi | Rendah (tanpa jaringan) | Ada tambahan latensi jaringan (HTTP call) |
| Kompleksitas operasional | Rendah | Lebih tinggi (banyak service, banyak port) |

---

## Catatan Penting

**Konsistensi data.** Pada versi Microservices, Order Service hanya *memeriksa* stok ke Book Service, tetapi **tidak mengurangi stok** setelah pesanan berhasil. Berbeda dengan versi Monolith yang langsung mengurangi `stock`. Ini menggambarkan tantangan *data consistency* pada Microservices, yang bisa diatasi dengan mekanisme event-driven atau *saga pattern*.

**Solusi latensi jaringan.** Caching (Redis), gRPC, message broker (Kafka/RabbitMQ), API Gateway dan load balancer, circuit breaker, serta connection pooling.

**Strategi scaling.** Jika traffic pencarian buku melonjak, cukup scale **Book Service** (port 5001). Order Service tidak perlu ditambah.

---

## Kesimpulan

- **Monolith** sederhana dikembangkan dan diuji, tetapi satu crash fatal dapat menjatuhkan seluruh aplikasi.
- **Microservices** memberikan fault isolation dan scaling selektif, dengan konsekuensi latensi jaringan dan kompleksitas operasional yang lebih tinggi.

## Saran Pengembangan

- Tambahkan sinkronisasi stok antar layanan pada versi Microservices.
- Eksplorasi API Gateway dan service discovery.
- Tambahkan pengujian dengan Postman atau automated testing.

---

## Referensi

- Flask Documentation. (2024). *Welcome to Flask*. https://flask.palletsprojects.com/
- Requests Documentation. (2024). *Requests: HTTP for Humans*. https://requests.readthedocs.io/
- Newman, S. (2021). *Building Microservices: Designing Fine-Grained Systems* (2nd ed.). O'Reilly Media.
