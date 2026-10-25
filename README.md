


<div align="center">

## gRPC Polyglot

![Logo PNL](img/logo-pnl.png)

### Disusun Oleh:  
Nama               : Muhammad Hafizd Maulana  
NIM                : 2024903430056   
Kelas              : TRKJ-3C


### Program Studi Teknologi Rekayasa Komputer Jaringan
### Jurusan Teknologi Informasi dan Komputer
### Politeknik Negeri Lhokseumawe
### 2026


---


## Lembar Pengesahan


| No. Praktikum     | : | 02 |
|------------------:|:-:|:--------------|
| Judul Praktikum   | : | gRPC Polyglot |
| Tanggal Praktikum | : | 19 September 2026 |
| Tanggal Penyerahan| : | 29 Oktober 2026 |
| Nama Praktikan    | : | Muhammad Hafizd Maulana |
| NIM/Kelas Praktikan| : | 2024903430056 / TRKJ-3C |
| Nilai Praktikum   | : | .......................... |
| Dosen Pengampu    | : | Muhammad Reza Zulman, S.ST., M.Cs |

Mengetahui,  
Dosen Pengampu
<br><br><br><br><br>

Muhammad Reza Zulman, S.ST., M.Cs
</div>



## Daftar Isi

1. [Arsitektur dan Struktur Proyek](#1-arsitektur-dan-struktur-proyek)
2. [Prasyarat](#2-prasyarat)
3. [Langkah 1: Setup Workspace dan Git](#3-langkah-1-setup-workspace-dan-git)
4. [Langkah 2: Menulis Kontrak API (`user.proto`)](#4-langkah-2-menulis-kontrak-api-userproto)
5. [Langkah 3: gRPC Server (Go)](#5-langkah-3-grpc-server-go)
6. [Langkah 4: gRPC Client (Python)](#6-langkah-4-grpc-client-python)
7. [Langkah 5: gRPC Client (Node.js)](#7-langkah-5-grpc-client-nodejs)
8. [Langkah 6: Pengujian](#8-langkah-6-pengujian)
9. [Langkah 7: Push ke GitHub](#9-langkah-7-push-ke-github)
10. [Evaluasi Mandiri 1: Menambah Field `phone`](#10-evaluasi-mandiri-1-menambah-field-phone)
11. [Evaluasi Mandiri 2: Analisis Pendekatan Kompilasi](#11-evaluasi-mandiri-2-analisis-pendekatan-kompilasi)
12. [Troubleshooting](#12-troubleshooting)

---

# gRPC Polyglot Lab

Hands-on **Komunikasi Inter-service dengan gRPC (Polyglot)**: satu **server Go** dilayani oleh dua **client** berbeda bahasa, yaitu **Python** dan **Node.js**, yang seluruhnya berbagi satu kontrak `user.proto` (pendekatan *Schema-First Development* dengan Protocol Buffers).

## 1. Arsitektur dan Struktur Proyek

```
                 user.proto  (Single Source of Truth)
                      |
        +-------------+--------------+
        |             |              |
   protoc (static) protoc (static) proto-loader (dynamic)
        |             |              |
   Go Server  <--  Python Client  Node.js Client
   :50051       gRPC / HTTP/2    gRPC / HTTP/2
```

Struktur akhir folder proyek:

```
grpc-polyglot-lab/
├── .gitignore
├── README.md
├── user.proto
├── go-server/
│   ├── go.mod
│   ├── go.sum
│   ├── main.go
│   └── pb/
│       ├── user.pb.go
│       └── user_grpc.pb.go
├── python-client/
│   ├── client.py
│   ├── user_pb2.py
│   └── user_pb2_grpc.py
└── node-client/
    ├── client.js
    ├── package.json
    └── package-lock.json
```

Konsep singkat:

| Konsep | Penjelasan |
|---|---|
| Protocol Buffers | Format data biner. File `.proto` menjadi kontrak layanan yang mendefinisikan struktur data dan metode. |
| HTTP/2 | Protokol dasar gRPC: *multiplexing*, *streaming* dua arah, dan kompresi. |
| Code Generation | Dari satu `.proto`, `protoc` menghasilkan kode server/client untuk berbagai bahasa. |
| Pola komunikasi | *Unary*, *server streaming*, *client streaming*, dan *bidirectional streaming*. Proyek ini memakai *unary*. |

---

## 2. Prasyarat

Langkah di bawah ditulis untuk **Ubuntu/Debian**. Alat yang digunakan:

| Alat | Fungsi |
|---|---|
| Git | Version control dan push ke GitHub |
| Python 3 (+ `venv`, `pip`) | Client Python |
| Node.js dan npm | Client Node.js |
| Go | Server gRPC |
| `protoc` | Compiler file `.proto` |

Cek apakah sudah terpasang:

```bash
git --version
python3 --version
node -v
npm -v
go version
protoc --version
```

Jika ada yang belum terpasang:

```bash
sudo apt update && sudo apt install -y git python3 python3-pip python3-venv nodejs npm golang-go protobuf-compiler
```

> **Catatan:** ketik `--version` dengan dua tanda hubung biasa (`-` `-`). Pada Ubuntu gunakan `python3` dan `pip3`, bukan `python` dan `pip`, di luar virtual environment.

Daftarkan folder plugin Go ke `PATH` (cukup **sekali**). Plugin `protoc-gen-go` dan `protoc-gen-go-grpc` disimpan di `~/go/bin`, sehingga tanpa langkah ini `protoc` menampilkan galat `program not found`:

```bash
echo 'export PATH="$PATH:$(go env GOPATH)/bin"' >> ~/.bashrc
source ~/.bashrc
echo $PATH | tr ':' '\n' | grep go
```

Output yang diharapkan: satu baris yang berakhir di `/go/bin`.

---

## 3. Langkah 1: Setup Workspace dan Git

Buat folder utama proyek dan inisialisasi Git:

```bash
mkdir grpc-polyglot-lab
cd grpc-polyglot-lab
git init
```

Atur identitas Git jika belum pernah dilakukan:

```bash
git config --global user.name "Nama Kamu"
git config --global user.email "emailgithub@kamu.com"
```

Buat `.gitignore` agar pustaka kompilasi dan dependensi pihak ketiga tidak ikut terunggah. Perintah `printf` di bawah menulis empat baris sekaligus sehingga tidak terduplikasi jika dijalankan ulang:

```bash
printf "node_modules/\n__pycache__/\n.venv/\n*.exe\n" > .gitignore
```

Commit pertama:

```bash
git add .gitignore
git commit -m "Initial commit: Add gitignore rules"
```

> Git dapat menampilkan petunjuk bahwa nama cabang awal adalah `master`. Nama ini diubah menjadi `main` pada langkah push.

> Jika nama folder induk memiliki spasi, kutip path saat `cd`, misalnya `cd ~/Documents/"nama folder"/grpc-polyglot-lab`.

---

## 4. Langkah 2: Menulis Kontrak API (`user.proto`)

Buat `user.proto` di folder utama proyek. File ini menjadi *Single Source of Truth* bagi server dan semua client.

```bash
cat > user.proto <<'EOF'
syntax = "proto3";

package user;

// Opsi khusus untuk bahasa Go agar struktur output rapi
option go_package = "./pb";

// Kontrak Request: Klien wajib mengirim user_id berupa angka
message UserRequest {
  int32 user_id = 1;
}

// Kontrak Response: Server akan mengembalikan nama dan email
message UserResponse {
  string name = 1;
  string email = 2;
}

// Definisi Service gRPC
service UserService {
  rpc GetUserProfile (UserRequest) returns (UserResponse);
}
EOF
```

Penjelasan:

- `package user` adalah nama paket. Client Node.js mengakses layanan lewat `.user`.
- `option go_package = "./pb"` membuat kode Go hasil generate disimpan di folder `pb`.
- `UserRequest` berisi satu angka `user_id`; `UserResponse` berisi `name` dan `email`.
- Angka pada tiap field (`= 1`, `= 2`) adalah nomor identitas field pada format biner. Jangan diubah setelah dipakai.
- `service UserService` mendefinisikan satu metode jarak jauh, yaitu `GetUserProfile`.

---

## 5. Langkah 3: gRPC Server (Go)

**3.1 Inisialisasi modul Go**

```bash
mkdir go-server
cd go-server
go mod init go-server
```

Nama modul `go-server` dipakai pada `import "go-server/pb"`.

**3.2 Pasang dependensi dan plugin**

```bash
go get google.golang.org/grpc
go get google.golang.org/protobuf
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

Verifikasi plugin ditemukan:

```bash
which protoc-gen-go protoc-gen-go-grpc
```

Output yang diharapkan:

```
/home/<user>/go/bin/protoc-gen-go
/home/<user>/go/bin/protoc-gen-go-grpc
```

**3.3 Generate kode Go dari `user.proto`**

```bash
protoc --proto_path=../ --go_out=. --go-grpc_out=. ../user.proto
ls pb/
```

Output yang diharapkan: `user.pb.go` dan `user_grpc.pb.go`.

**3.4 Tulis `main.go`**

```bash
cat > main.go <<'EOF'
package main

import (
	"context"
	"fmt"
	"log"
	"net"

	"go-server/pb"
	"google.golang.org/grpc"
)

type server struct {
	pb.UnimplementedUserServiceServer
}

func (s *server) GetUserProfile(ctx context.Context, req *pb.UserRequest) (*pb.UserResponse, error) {
	log.Printf("Menerima request untuk User ID: %d\n", req.GetUserId())
	if req.GetUserId() == 1 {
		return &pb.UserResponse{
			Name:  "Budi Santoso",
			Email: "budi@universitas.ac.id",
		}, nil
	}
	return &pb.UserResponse{
		Name:  "User Tidak Ditemukan",
		Email: "-",
	}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Gagal listening port: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterUserServiceServer(s, &server{})
	fmt.Println("gRPC Server (Go) berjalan di port :50051...")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("Gagal menjalankan server: %v", err)
	}
}
EOF
go mod tidy
```

Penjelasan singkat:

- `GetUserProfile` mencatat log setiap request, lalu mengembalikan data "Budi Santoso" jika `user_id == 1`, selain itu "User Tidak Ditemukan". Data ini sengaja *hardcoded* (data tiruan).
- `net.Listen("tcp", ":50051")` membuka port 50051, lalu `grpc.NewServer()` dan `RegisterUserServiceServer` mendaftarkan layanan.
- `go mod tidy` merapikan dependensi di `go.mod` dan `go.sum`.

---

## 6. Langkah 4: gRPC Client (Python)

Pada Ubuntu, `pip install` global ditolak (galat `externally-managed-environment`), jadi gunakan *virtual environment*. Folder `.venv/` sudah masuk `.gitignore`.

```bash
cd ../
mkdir python-client
cd python-client
python3 -m venv .venv
source .venv/bin/activate
pip install grpcio grpcio-tools
python -m grpc_tools.protoc --proto_path=../ --python_out=. --grpc_python_out=. ../user.proto
ls
```

Output yang diharapkan: `user_pb2.py` dan `user_pb2_grpc.py`.

Tulis `client.py`:

```bash
cat > client.py <<'EOF'
import grpc
import user_pb2
import user_pb2_grpc

def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        print("Mengirim request ke Go Server menggunakan Python...")
        response = stub.GetUserProfile(user_pb2.UserRequest(user_id=1))
        print(f"Respon dari Server -> Nama: {response.name}, Email: {response.email}")

if __name__ == '__main__':
    run()
EOF
```

Penjelasan singkat: `insecure_channel` membuka koneksi tanpa enkripsi ke server, `UserServiceStub` adalah perwakilan server di sisi client, dan `GetUserProfile(UserRequest(user_id=1))` mengirim request lalu menunggu balasan.

---

## 7. Langkah 5: gRPC Client (Node.js)

```bash
cd ../
mkdir node-client
cd node-client
npm init -y
npm install @grpc/grpc-js @grpc/proto-loader
```

Tulis `client.js`:

```bash
cat > client.js <<'EOF'
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, '../user.proto');
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
});
const userProto = grpc.loadPackageDefinition(packageDefinition).user;

function main() {
    const client = new userProto.UserService('localhost:50051', grpc.credentials.createInsecure());
    console.log("Mengirim request ke Go Server menggunakan Node.js...");
    client.GetUserProfile({ user_id: 1 }, (error, response) => {
        if (!error) {
            console.log(`Respon dari Server -> Nama: ${response.name}, Email: ${response.email}`);
        } else {
            console.error("Terjadi error:", error);
        }
    });
}
main();
EOF
```

Penjelasan singkat: `protoLoader.loadSync` membaca `../user.proto` **saat program berjalan** (tanpa langkah kompilasi), dan `keepCase: true` menjaga nama field tetap `user_id`.

---

## 8. Langkah 6: Pengujian

Buka **3 terminal terpisah** dari folder `grpc-polyglot-lab`.

**Terminal 1: server Go**

```bash
cd go-server
go run main.go
```

**Terminal 2: client Python**

```bash
cd python-client
source .venv/bin/activate
python client.py
```

**Terminal 3: client Node.js**

```bash
cd node-client
node client.js
```

Output yang diharapkan:

```
# Terminal 1
gRPC Server (Go) berjalan di port :50051...
2026/10/22 21:01:52 Menerima request untuk User ID: 1
2026/10/22 21:02:17 Menerima request untuk User ID: 1

# Terminal 2
Mengirim request ke Go Server menggunakan Python...
Respon dari Server -> Nama: Budi Santoso, Email: budi@universitas.ac.id

# Terminal 3
Mengirim request ke Go Server menggunakan Node.js...
Respon dari Server -> Nama: Budi Santoso, Email: budi@universitas.ac.id
```

Kriteria keberhasilan:

| No | Kriteria | Status |
|---|---|---|
| 1 | Tidak ada galat (`Connection refused`, `Cannot find module`) di ketiga terminal | Terpenuhi |
| 2 | Data profil "Budi Santoso" tercetak di terminal Python dan Node.js, padahal hanya ditulis di server Go | Terpenuhi |
| 3 | Terminal server Go mencatat log baru setiap ada client yang mengeksekusi kode | Terpenuhi |

---

## 9. Langkah 7: Push ke GitHub

Buat dulu **repositori kosong** di GitHub (tanpa README, `.gitignore`, atau lisensi), lalu salin URL-nya.

```bash
cd ..    # kembali ke folder grpc-polyglot-lab
git add .
git commit -m "Feat: Add full polyglot implementation of Go server, Python client, and Node client"
git branch -M main
git remote add origin https://github.com/USERNAME_ANDA/NAMA_REPO.git
git push -u origin main
```

> Saat `git push` meminta password, gunakan **Personal Access Token** (GitHub: *Settings > Developer settings > Personal access tokens*, centang scope `repo`), bukan password akun.

---

## 10. Evaluasi Mandiri 1: Menambah Field `phone`

**10.1 Ubah `user.proto`.** Tambahkan field `phone` bernomor 3 pada `UserResponse`:

```proto
message UserResponse {
  string name = 1;
  string email = 2;
  string phone = 3;
}
```

**10.2 Generate ulang** kode Go dan Python. Node.js tidak perlu karena skema dibaca langsung saat dijalankan.

```bash
# Go (dari folder go-server)
cd go-server
protoc --proto_path=../ --go_out=. --go-grpc_out=. ../user.proto

# Python (dari folder python-client)
cd ../python-client
source .venv/bin/activate
python -m grpc_tools.protoc --proto_path=../ --python_out=. --grpc_python_out=. ../user.proto
```

**10.3 Sesuaikan implementasi.**

Server Go (`go-server/main.go`), tambahkan `Phone` pada data balasan:

```go
return &pb.UserResponse{
    Name:  "Budi Santoso",
    Email: "budi@universitas.ac.id",
    Phone: "081234567890",
}, nil
```

Client Python (`python-client/client.py`), ganti baris `print` terakhir:

```python
print(f"Respon dari Server -> Nama: {response.name}, Email: {response.email}, Phone: {response.phone}")
```

Client Node.js (`node-client/client.js`), ganti baris `console.log` di dalam `if (!error)`:

```js
console.log(`Respon dari Server -> Nama: ${response.name}, Email: ${response.email}, Phone: ${response.phone}`);
```

**10.4 Uji ulang.** Hentikan server lama (`Ctrl+C`), lalu jalankan ulang server dan kedua client seperti pada [Langkah 6](#8-langkah-6-pengujian). Output yang diharapkan:

```
Respon dari Server -> Nama: Budi Santoso, Email: budi@universitas.ac.id, Phone: 081234567890
```

---

## 11. Evaluasi Mandiri 2: Analisis Pendekatan Kompilasi

Pada proyek ini, berkas `user.proto` diproses dengan dua pendekatan berbeda. Go dan Python memakai *static code generation*: compiler `protoc` (dengan plugin `protoc-gen-go`, `protoc-gen-go-grpc`, atau `grpc_tools`) dijalankan lebih dahulu untuk menghasilkan berkas kode nyata (`user.pb.go`, `user_grpc.pb.go`, `user_pb2.py`, dan `user_pb2_grpc.py`) yang kemudian di-*import* oleh program. Keuntungannya, struktur data dan *stub* sudah menjadi bagian dari kode sumber sehingga tipe data dapat diperiksa saat kompilasi (khususnya pada Go), performa awal program lebih cepat, dan editor dapat memberikan *autocomplete*. Kekurangannya, setiap perubahan skema, misalnya penambahan field `phone`, mewajibkan proses generate ulang dan berkas hasil generate harus dikelola di repositori.

Sebaliknya, Node.js memakai *dynamic runtime generation*: pustaka `@grpc/proto-loader` membaca berkas `user.proto` langsung saat program dijalankan dan membangun definisi layanan di memori tanpa langkah kompilasi. Pendekatan ini lebih praktis dan fleksibel karena perubahan skema langsung terbaca setelah program dijalankan ulang. Namun, tidak ada pemeriksaan tipe saat kompilasi sehingga kesalahan nama field baru terdeteksi saat program berjalan, dan terdapat sedikit beban tambahan untuk mem-*parsing* berkas `.proto` setiap kali aplikasi dimulai.

| Aspek | Static (Go dan Python) | Dynamic (Node.js) |
|---|---|---|
| Waktu pemrosesan `.proto` | Sebelum program dijalankan (`protoc`) | Saat program dijalankan (`proto-loader`) |
| Berkas hasil | Ada (`user.pb.go`, `user_pb2.py`, dan sebagainya) | Tidak ada, definisi dibuat di memori |
| Pemeriksaan tipe | Saat kompilasi (kuat pada Go) | Saat runtime |
| Perubahan skema | Wajib generate ulang | Cukup jalankan ulang program |
| Kelebihan | Aman tipe, performa awal cepat, ada autocomplete | Praktis dan fleksibel |
| Kekurangan | Berkas hasil generate perlu dikelola | Galat terdeteksi terlambat, ada beban parsing awal |

Simpan perubahan `README.md` dan push:

```bash
git add README.md
git commit -m "Docs: Add README with steps and static vs dynamic code generation analysis"
git push
```

---

## 12. Troubleshooting

| Error | Penyebab dan solusi |
|---|---|
| `protoc: command not found` | `protoc` belum terpasang. Jalankan `sudo apt install -y protobuf-compiler`. |
| `protoc-gen-go: program not found` | PATH belum berisi `~/go/bin`. Ulangi langkah pada bagian [Prasyarat](#2-prasyarat). |
| `cannot find package "go-server/pb"` | Generate Go belum dijalankan, atau lupa `go mod tidy`. |
| `Connection refused` / `UNAVAILABLE` | Server Go belum berjalan, atau port bukan 50051. |
| `externally-managed-environment` | `pip` dijalankan di luar venv. Jalankan `source .venv/bin/activate`. |
| `ModuleNotFoundError: user_pb2` | `client.py` dijalankan dari folder lain, atau generate Python belum dilakukan. |
| `Cannot find module '@grpc/grpc-js'` | `npm install` belum dijalankan di folder `node-client`. |
| `cd: No such file or directory` | Nama folder induk mengandung spasi. Kutip path atau gunakan Tab untuk melengkapi nama. |
| `git push` ditolak / minta password | Gunakan Personal Access Token, bukan password akun. |