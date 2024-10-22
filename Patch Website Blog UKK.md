# Patch Website Blog UKK

## Introduction
Dalam kasus ini diberikan sebuah website blog2 dari Github [danial-smktelkom-mlg][dill]. Tugas ini adalah bertujuan untuk menganalisa dan mencoba menutup celah yang ada pada website sudah diberikan. Antara lain file yang perlu di perbaiki adalah :

- pearch.php
- admin_login.php
- post.php
- gb.php

## Search.php
### Analisa

Sebelum menutup celah sebaiknya analisa terlebih dahulu program yang digunakan pada halaman serch.php
```sh
session_start();
include 'koneksi.php';
$q = $_GET['q'];
$posts = mysqli_query($conn, "SELECT * FROM post WHERE judul LIKE '%{$q}%' OR konten LIKE '%{$q}%'");
```
Pada bagian "$q = $_GET['q'];" adalah superglobal array di PHP yang digunakan untuk mendapatkan nilai parameter q dari URL. Dengan contoh:
```sh
"http://localhost/index.php?q=search"
```
Maka "$_GET['q']"" akan mendapatkan nilai search. Tetapi pada ini tidak memiliki sebuah filter yang dapat mencegah serangan, script ini dapat di exploit dengan menggunakan Cross-Site Scripting (XSS).

Pada bagian mysql :
```sh
mysqli_query($conn, "SELECT * FROM post WHERE judul LIKE '%{$q}%' OR konten LIKE '%{$q}%'")
```
mysqli_query() digunakan untuk mengeksekusi query SQL pada database. Hal ini, query yang dieksekusi adalah pencarian semua data dari tabel post yang judul atau konten mengandung nilai dari variabel $q. Dengan ini celah dapat di exploit dengan SQL Injection, Karena nilai $_GET['q'] langsung dimasukkan ke dalam query SQL tanpa validasi atau sanitasi yang memadai

### Menutup Celah

 Pada bagian GET :
```sh
$q = $_GET['q'];
```
Pada bagian ini diberikan sebuah filter untuk menutup celah yang dapat di exploit dengan XSS. Pembenaran :
```sh
$q = htmlspecialchars($_GET['q'], ENT_QUOTES, 'UTF-8');
```

Rangkuman:
1.htmlspecialchars
- Fungsi htmlspecialchars() dalam PHP digunakan untuk mengkonversi karakter-karakter khusus menjadi entitas HTML.

2.ENT_QUOTES
- Konstanta ini memberitahu fungsi htmlspecialchars() untuk mengonversi baik tanda kutip tunggal ('') maupun tanda kutip ganda ("") menjadi entitas HTML.

3.UTF-8
- Adalah parameter yang menentukan encoding karakter. fungsi memastikan bahwa input diinterpretasikan sebagai karakter UTF-8, yang merupakan standar encoding untuk sebagian besar aplikasi web modern.

Pada bagian mysql:
```sh
mysqli_query($conn, "SELECT * FROM post WHERE judul LIKE '%{$q}%' OR konten LIKE '%{$q}%'")
```
Dengan mysql langsung meng-eksekusi mentod GET maka dari itu harus di patch. Dengan prepared statements, query dan data input dipisahkan sehingga data akan di-escape dengan aman. Pembenaran :
```sh
$stmt = $conn->prepare("SELECT * FROM post WHERE judul LIKE ? OR konten LIKE ?");
$search_term = "%{$q}%";
$stmt->bind_param("ss", $search_term, $search_term);
$stmt->execute();
$result = $stmt->get_result();
```
Rangkuman:
1.$stmt = $conn->prepare("SELECT * FROM post WHERE judul LIKE ? OR konten LIKE ?");
- Prepare() adalah metode dari objek koneksi database ($conn) yang digunakan untuk mempersiapkan query SQL yang akan dieksekusi.
- Query ini mengekstraksi semua kolom (SELECT *) dari tabel post, di mana kolom judul atau konten cocok dengan suatu pola pencarian.
- Tanda ? dalam query adalah placeholder yang akan diganti dengan nilai variabel input nantinya. Placeholder ini memastikan bahwa data input akan di-escape dengan aman oleh MySQL, menghindari SQL Injection.

2.$search_term = "%{$q}%";
- Ini adalah penyiapan variabel $search_term untuk digunakan sebagai parameter pencarian. Tanda % adalah wildcard dalam SQL, yang berarti "apa saja" atau "sekumpulan karakter."
- Variabel $q adalah input pencarian dari pengguna (misalnya dari $_GET['q']).
- Jika $q bernilai "example", maka $search_term akan menjadi "%example%", yang berarti MySQL akan mencari setiap judul atau konten yang mengandung kata "example" di dalamnya.

3.$stmt->bind_param("ss", $search_term, $search_term);
- bind_param() digunakan untuk mengikat nilai variabel ke placeholder (?) di query SQL yang sudah dipersiapkan sebelumnya dengan prepare().
- Parameter pertama "ss" menunjukkan tipe data dari variabel yang akan di-bind:
"s" berarti string. Karena ada dua tanda tanya (?) dalam query SQL, maka kita menggunakan dua "s" (satu untuk setiap placeholder).
- Variabel $search_term diikat ke kedua placeholder (?). Artinya, pencarian akan dilakukan di kolom judul dan konten menggunakan nilai dari $search_term.

4.$stmt->execute();
- Fungsi ini memastikan bahwa query yang dieksekusi aman dan tidak rentan terhadap SQL Injection karena variabel input sudah di-escape dengan benar oleh database.

5.$result = $stmt->get_result();
- Setelah query dieksekusi, kita dapat mengambil hasilnya dengan menggunakan get_result().
- Hasil ini akan disimpan dalam variabel $result, yang bisa kita gunakan untuk mengambil baris-baris data yang ditemukan sesuai dengan kriteria pencarian.
- Data ini kemudian bisa ditampilkan atau diproses lebih lanjut, misalnya menggunakan fetch_assoc() untuk mengubah hasil query menjadi array asosiatif.

## Admin_login.php
### Analisa
Sebelum menutup celah sebaiknya kita mengalisa terlebih dahulu agar mengetahui celah yang dimiliki.
```sh
$username = $_POST['username'];
$password = $_POST['password'];

$login = mysqli_query($conn, "SELECT * FROM user WHERE username = '{$username}' AND password = '{$password}'");

if (mysqli_num_rows($login) == 0) {
	die("Username atau password salah!");
} else {
	$_SESSION['admin'] = 1;
	header("Location: admin.php");
```
Terlihat bahwa tidak ada validasi pada bagian login user admin, seperti inputan tidak boleh kosong dan lainnya.
- Pada bagian $username dan $password berfungsi sebagai mengambil username dan password yang di inputkan oleh pengguna
- Pada bagian $login mengirimkan query SQL untuk memeriksa apakah ada pengguna dengan username dan password yang sesuai di tabel user.
- Pada bagian if dan else berfungsi, jika tidak ada hasil yang ditemukan (username atau password salah), skrip akan menghentikan eksekusi dan menampilkan pesan kesalahan. Jika berhasil, sesi admin akan disetel dan pengguna diarahkan ke halaman admin.

## Menutup Celah
Sript awal :
```sh
$username = $_POST['username'];
$password = $_POST['password'];
	
$login = mysqli_query($conn, "SELECT * FROM user WHERE username ='{$username}' AND password = '{$password}'");
```
Perbaikan script :
```sh
$username = mysqli_real_escape_string($conn, $_POST['username']);
$password = mysqli_real_escape_string($conn, $_POST['password']);
$login = mysqli_query($conn, "SELECT * FROM user WHERE username ='{$username}' AND password = '{$password}'");
```
Rangkuman :
- Pada bagian veriabel $username dan $password melalui proses mysqli_real_escape_string() yang berfungsi mengamankan input pengguna dengan menambahkan escape karakter pada karakter khusus di input (seperti tanda kutip), sehingga dapat mencegah serangan SQL Injection.

Di sini saya juga menambahkan script agar pengguna harus mengisi inputan dari username dan password.
```sh
if (empty($username) || empty($password)) {
	die("Username atau password tidak boleh kosong!");
}
```
Fungsi empty() digunakan untuk memeriksa apakah suatu variabel kosong. Variabel akan dianggap "kosong" jika:
- Tidak diinisialisasi (tidak memiliki nilai).
- Berisi nilai yang dievaluasi sebagai false, seperti string kosong "", angka 0, atau nilai null.

|| (OR operator):
- Operator || adalah operator logika OR. Kondisi ini akan menghasilkan true jika salah satu dari dua kondisi tersebut benar. Artinya, jika username atau password kosong, maka blok kode dalam if akan dieksekusi.

## Post.php
### Analisa
Analisa lagi. :b
```sh
$id = $_GET['id'];
$q  = mysqli_query($conn, "SELECT * FROM post WHERE id = {$id}") or die(mysqli_error($conn));
$post = mysqli_fetch_array($q);
```
Hampir sama dengan celah yang dimiliki oleh search.php, pada bagian GET digunakan untuk mendapatkan nilai parameter id. Di sinilah letak dari celah yang ada pada post.php. Contoh URL :
```sh
http://localhost/blog1/post.php?id=1
```
Dapat di lihat pada URL memiliki parameter id=1 ini bisa jadi merupakan celah yang dapat di exploit dengan SQL UNION dan sqlmap untuk mendapatkan database dari server, jika tidak di berikan validasi atau filter pada php.
- Parameter $id metode untuk mendapatkan data dari query string seperti contoh di atas
- Parameter $q menjalankan query SQL menggunakan mysqli_query() untuk mengambil semua data dari tabel post di database yang memiliki id sama dengan nilai $id. Jika terjadi kesalahan dalam query, fungsi or die(mysqli_error($conn)) akan menghentikan script dan menampilkan pesan error dari database.
- Parameter $post digunakan untuk mengeksekusi mysqli_fetch_array(), yang mengambil hasil query SQL yang disimpan di variabel $q dan mengembalikannya sebagai array asosiatif atau numerik (atau keduanya). Data ini kemudian disimpan dalam variabel $post, yang bisa digunakan untuk menampilkan informasi dari tabel post berdasarkan ID.

### Menutup Celah
Script awal :
```sh
$id = $_GET['id'];
```
Di parameter ini seharusnya di berikan validasi agar tidak terjadi celah yang bisa dimanfaatkan. Perbaikan :
```sh
if (!isset($_GET['id']) || !is_numeric($_GET['id'])) {
    echo "vulnerability sudah ditutup ya:)";
    exit;
}
```
1.!isset($_GET['id'])
- isset() digunakan untuk memeriksa apakah variabel $_GET['id'] telah didefinisikan atau tidak, artinya apakah parameter id ada di URL atau tidak. Jika id tidak ada, maka isset() akan mengembalikan false. Dengan adanya tanda ! (NOT), maka kondisi ini menjadi true jika $_GET['id'] tidak ada.

2.|| (OR operator):
- Operator || (logika OR) berarti jika salah satu dari dua kondisi bernilai true, maka blok kode dalam if akan dieksekusi.

3.!is_numeric($_GET['id'])
- is_numeric() digunakan untuk memeriksa apakah nilai dari $_GET['id'] adalah angka (numeric). Jika $_GET['id'] bukan angka (misalnya berupa string atau karakter lain), maka is_numeric() akan mengembalikan false. Tanda ! (NOT) membalik kondisi ini menjadi true jika nilai $_GET['id'] bukan angka.

4.echo
- Sebuah fungsi untuk memasukan atau menampilkan strings ketika pengguna melakukan exploit pada website

## Gb.php
### Analisa

Analisa pada file gb.php
```sh
while($row = mysqli_fetch_array($pesan)) {
	echo "<small>Oleh <b>{$row['nama']}</b> pada {$row['tanggal']}</small>";
	echo "<p>{$row['pesan']}</p>";
	echo "<hr>";
}
```
Terlihat bahwa pada php tidak ada validasi yang membatasi pengguna untuk memasukan inputan. Ini bisa menjadi celah dari Cross-Site Scripting (XSS).
1.while($row = mysqli_fetch_array($pesan))
- mysqli_fetch_array($pesan) digunakan untuk mengambil baris hasil query SQL dari variabel $pesan (yang diasumsikan berisi hasil query ke database) dalam bentuk array. Setiap iterasi dalam while akan mengisi variabel $row dengan satu baris data dari hasil query, dan perulangan akan terus berjalan selama masih ada baris data yang tersisa.

2.echo "<small>Oleh <b>{$row['nama']}</b> pada {$row['tanggal']}</small>";
- Menampilkan nama dan tanggal pengirim pesan. $row['nama'] dan $row['tanggal'] adalah kolom dari hasil query yang diambil dari database. Nama pengirim ditampilkan dengan tag <b> (bold), dan tanggal pesan ditampilkan dalam elemen <small>, biasanya untuk memberi tampilan teks yang lebih kecil.

3.echo "<p>{$row['pesan']}</p>";
- Menampilkan isi pesan yang tersimpan di kolom pesan dari database. Tag <p> digunakan untuk membungkus teks pesan dalam paragraf HTML agar terlihat rapi dan terstruktur saat ditampilkan di halaman web.

4.echo "<hr>";
- Tag HTML <hr> digunakan untuk menampilkan garis horizontal sebagai pemisah antara pesan yang satu dengan yang lain. Ini berguna untuk memisahkan setiap pesan secara visual pada halaman web, sehingga tampilan lebih jelas.

### Menutup Celah
Script awal :
```sh
while($row = mysqli_fetch_array($pesan)) {
    echo "<small>Oleh <b>{$row['nama']}</b> pada {$row['tanggal']}</small>";
    echo "<p>{$row['pesan']}</p>";
    echo "<hr>";
}
```
Pada script ini diberikan validasi agar command XSS tidak dapat dijalankan. Perbaikan :
```sh
while ($row = mysqli_fetch_array($pesan)) {
    echo "<small>Oleh <b>" . htmlspecialchars($row['nama']) . "</b> pada " . htmlspecialchars($row['tanggal']) . "</small>";
    echo "<p>" . htmlspecialchars($row['pesan']) . "</p>";
    echo "<hr>";
    }
```
Validasi htmlspecialchars digunakan pada penutupan celah ini karena dalam PHP digunakan untuk mengonversi karakter khusus (special characters) dalam teks menjadi entitas HTML (HTML entities). 

   [dill]: <https://github.com/danial-smktelkom-mlg>

 
