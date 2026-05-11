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
