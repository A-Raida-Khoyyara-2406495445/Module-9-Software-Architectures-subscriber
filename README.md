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