# Understanding Subscriber and Message Broker

## a. What is AMQP?

AMQP (*Advanced Message Queuing Protocol*) adalah protokol yang digunakan agar aplikasi bisa saling mengirim pesan melalui sebuah message broker seperti RabbitMQ. AMQP itu seperti “aturan komunikasi” supaya program-program yang berbeda (bahkan yang ditulis dalam bahasa pemrograman berbeda) bisa bertukar data dengan aman dan teratur melalui *message broker*. Dengan AMQP, publisher dapat mengirim pesan ke broker, lalu subscriber menerima dan memproses pesan tersebut tanpa harus terhubung langsung satu sama lain.

## b. Apa arti `guest:guest@localhost:5672`?

URL `amqp://guest:guest@localhost:5672` adalah alamat koneksi untuk menghubungkan program ke RabbitMQ.
Penjelasannya:
- guest pertama : username untuk login ke RabbitMQ
- guest kedua : password dari user tersebut
- localhost : RabbitMQ berjalan di komputer yang sama
- 5672 : port default yang digunakan RabbitMQ untuk komunikasi AMQP

Jadi, alamat tersebut berarti program mencoba terhubung ke RabbitMQ lokal menggunakan akun default guest.

## c. Simulation: Slow Subscriber

Pada percobaan ini, subscriber dikonfigurasi untuk memproses pesan secara lambat dengan menambahkan `thread::sleep(time::Duration::from_millis(1000))` pada fungsi `handle`.

![Slow Subscriber](assets/images/slowsubscriber.png)

### Mengapa jumlah antrean (queue) bisa menumpuk?

Berdasarkan hasil observasi pada chart RabbitMQ, jumlah pesan di dalam queue (Ready) mencapai angka **5** sesaat setelah publisher dijalankan. Hal ini terjadi karena:

1.  **Publisher sangat cepat**: Publisher mengirimkan 5 pesan/event sekaligus dalam waktu yang sangat singkat (kurang dari 1 detik).
2.  **Subscriber lambat**: Subscriber diatur untuk tidur selama 1 detik (`sleep(1000ms)`) setiap kali menerima satu pesan.
3.  **Ketimpangan Kecepatan**: Karena publisher mengirim jauh lebih cepat daripada kemampuan subscriber memproses (1 pesan/detik), maka pesan-pesan tersebut harus mengantre di dalam message broker (RabbitMQ) sampai subscriber selesai memproses pesan sebelumnya dan siap mengambil pesan berikutnya.

Ini menunjukkan salah satu fungsi utama Message Broker sebagai **buffer**, yang menampung beban kerja saat terjadi lonjakan data (spike) agar sistem tidak *crash* meskipun pemrosesan di sisi consumer lebih lambat.

## d. Simulation: Multiple Subscribers

Pada tahap ini, kita menjalankan **tiga subscriber** sekaligus untuk memproses pesan dari satu publisher yang sama.

![Multiple Subscribers](assets/images/3subcriber1producer.png)

### Refleksi: Mengapa antrean berkurang lebih cepat?

Saat menjalankan tiga subscriber, lonjakan pesan (spike) pada chart RabbitMQ berkurang jauh lebih cepat dibandingkan saat hanya ada satu subscriber. Hal ini terjadi karena:

1.  **Horizontal Scaling (Parallelism)**: RabbitMQ mendistribusikan pesan secara merata (round-robin) ke semua subscriber yang aktif. Dengan 3 subscriber yang masing-masing memproses 1 pesan/detik, kapasitas total pemrosesan meningkat menjadi **3 pesan/detik**.
2.  **Efisiensi Waktu**: Jika satu subscriber membutuhkan 5 detik untuk menyelesaikan 5 pesan, maka dengan 3 subscriber, waktu yang dibutuhkan hanya sekitar 2 detik saja.

### Analisis Kode: Apa yang bisa ditingkatkan?

Setelah melihat kode pada `publisher` dan `subscriber`, berikut adalah beberapa hal yang bisa ditingkatkan:

1.  **Error Handling (Robustness)**:
    - Penggunaan `.unwrap()` pada pembuatan koneksi (`CrosstownBus::new_queue_...`) berisiko membuat program *crash* jika RabbitMQ belum siap. Sebaiknya menggunakan mekanisme *retry* atau penanganan error yang lebih elegan.
2.  **Main Loop Subscriber (CPU Efficiency)**:
    - Pada `subscriber/src/main.rs`, terdapat `loop {}` kosong di akhir fungsi `main`. Ini adalah *busy-wait* yang mengonsumsi CPU secara sia-sia.
    - **Saran**: Gunakan signal listener seperti `tokio::signal::ctrl_c().await` atau `std::thread::park()` untuk menjaga program tetap berjalan tanpa membebani CPU.
3.  **Logging**:
    - Saat ini program hanya menggunakan `println!`. Penggunaan *logging framework* seperti `tracing` atau `log` akan memudahkan pemantauan sistem di lingkungan produksi.
4.  **Konfigurasi**:
    - URL RabbitMQ dan nama queue masih di-*hardcode*. Sebaiknya dipindahkan ke variabel lingkungan (`environment variables`) atau file konfigurasi.

---

# Bonus: Running on Cloud

## Screenshot RabbitMQ Web UI di Cloud 
![Running 3 subscribers on Cloud (RabbitMQ Web UI)](assets/images/bonus3s1prabbitmq.png)

Pada tampilan RabbitMQ Management yang di-host pada Railway, broker berhasil berjalan dan dapat diakses melalui URL `https://rabbitmq-web-ui-production-b12e.up.railway.app`. Terlihat pada bagian global counts terdapat `Connections: 3`, `Exchanges: 10`, dan `Queues: 2`, yang menunjukkan bahwa ketiga subscriber telah berhasil terkoneksi secara bersamaan ke RabbitMQ yang berjalan di cloud. Selain itu, node aktif yang digunakan adalah `rabbit@rabbitmq`, menandakan service RabbitMQ berjalan dengan baik pada environment Railway.

## Screenshot Terminal Menjalankan 3 Subscriber dan 1 Publisher di Cloud

![Running 3 subscribers on Cloud (Terminal)](assets/images/bonus3s1pterminal.png)

Selain itu, distribusi message pada RabbitMQ cloud menunjukkan bahwa mekanisme round-robin berjalan dengan baik. Berdasarkan hasil pada masing-masing terminal subscriber, event dibagikan secara bergantian sebagai berikut:

Terminal 1:
- `UserCreatedEventMessage { user_id: "3", user_name: "2406495445-Cica" }`

Terminal 2:
- `UserCreatedEventMessage { user_id: "1", user_name: "2406495445-Amir" }`
- `UserCreatedEventMessage { user_id: "4", user_name: "2406495445-Dira" }`

Terminal 3:
- `UserCreatedEventMessage { user_id: "2", user_name: "2406495445-Budi" }`
- 'UserCreatedEventMessage { user_id: "5", user_name: "2406495445-Emir" }'

Hasil tersebut menunjukkan bahwa RabbitMQ berhasil mendistribusikan event ke beberapa subscriber secara bergiliran, sehingga beban pemrosesan queue dapat terbagi lebih merata antar subscriber yang aktif.

