# Jawaban UTS Keamanan Jaringan

- **NIM**: 24552011069
- **Jumlah digit NIM**: 35
- **Sisa bagi (mod 10)**: 5
- **Nomor kasus**: Kasus 5

## Tugas 1 (Pertemuan 1): Docker dan volume

- Volume `vol-k5` dibuat, diisi `24552011069` ke `/data/laporan-k5.txt` lewat container `w-k5`, lalu dibaca ulang dari container baru (`--rm`) setelah `w-k5` dihapus dengan hasil `24552011069`.
- Stack compose proyek `k5` (`k5-target-1` dan `k5-attacker-1` pada jaringan `k5_lab`) dijalankan dan `wget -qO- http://target` dari `attacker` berhasil menampilkan halaman HTML `Welcome to nginx!`.

## Tugas 2 (Pertemuan 2): Kriptografi

- Enkripsi `laporan-k5.txt` menjadi `laporan-k5.enc` (`aes-256-cbc -pbkdf2`, password `kunci-k5-2026`) dan dekripsi menjadi `hasil-k5.txt` berhasil dengan nilai `sha256sum` yang identik (`22e90e5baa6048f9871927be4fa8af9d141b10a1071cf2b975272db8343f79b5`).
- Kunci privat RSA 2048-bit (`key-k5.pem`) dan sertifikat X.509 (`cert-k5.pem`, masa berlaku 30 hari: `Nov 24 17:56:45 2026 GMT` s.d. `Dec 24 17:56:45 2026 GMT`) untuk `CN=k5.uts.local` berhasil dibuat dan diverifikasi (`subject`, `issuer`, `dates`).

## Tugas 3 (Pertemuan 3): ARP Spoofing

- Jaringan `lab-arp-k5` dibuat dengan container `korban-k5` (`172.19.0.2`), `gateway-k5` (`172.19.0.3`, MAC `b2:2f:08:bc:e5:a3`), dan `penyerang-k5` (`172.19.0.4`, MAC `1a:61:eb:c9:1a:23`).
- `arpspoof -i eth0 -t 172.19.0.2 172.19.0.3` dijalankan dari `penyerang-k5`, lalu mitigasi static ARP (`arp -s 172.19.0.3 b2:2f:08:bc:e5:a3`) diterapkan pada `korban-k5` sehingga entri gateway tampil `PERM` pada `arp -a`.

## Tugas 4 (Pertemuan 4): Analisis Handshake WPA2

- Perintah `aircrack-ng -w wordlist-k5.txt handshake-k5.cap` dijalankan menggunakan container `uts-kali-tools`.
- Hasil pengujian: `This file is not a regular 802.11 (wireless) capture.`, `Read 0 packets.`, `No networks found, exiting.` (tidak ada kata sandi WPA2 yang ditemukan karena file sampel bukan tangkapan paket 802.11 wireless nyata).
- Penjelasan 4-way handshake WPA2:
  1. **Message 1 (AP -> Client)**: AP mengirim `ANonce` (nonce acak) agar client dapat membentuk `PTK` (Pairwise Transient Key).
  2. **Message 2 (Client -> AP)**: Client membuat `SNonce`, menghitung `PTK` dari `PMK` + `ANonce` + `SNonce` + MAC AP + MAC Client, lalu mengirim `SNonce` dan `MIC` (Message Integrity Code) untuk membuktikan kepemilikan `PMK`/PSK.
  3. **Message 3 (AP -> Client)**: AP memverifikasi `MIC`, lalu mengirim `GTK` (Group Temporal Key, terenkripsi) dan `MIC` untuk mengonfirmasi pemasangan kunci.
  4. **Message 4 (Client -> AP)**: Client mengirim `ACK`/`MIC` terakhir sebagai konfirmasi bahwa kunci siap dipakai untuk mengenkripsi data.

## Tugas 5 (Pertemuan 5): ICMP dengan Scapy

- Skrip `kirim_icmp.py` dijalankan dari `172.19.0.5` ke target `172.19.0.2` pada jaringan `lab-arp-k5` melalui container `uts-kali-tools` (`IP / ICMP 172.19.0.5 > 172.19.0.2 echo-request 0`).
- Hasil balasan ICMP: **TTL balasan = `64`**, **Tipe ICMP balasan = `0`** (ICMP Echo Reply).
- **Penjelasan Smurf Attack**: Serangan *smurf attack* adalah serangan DoS/DDoS berbasis ICMP dan IP spoofing di mana penyerang mengirim ICMP Echo Request ke alamat *broadcast* suatu jaringan dengan memalsukan IP sumber (*source IP*) menjadi IP korban. Seluruh host aktif di jaringan tersebut merespons dengan ICMP Echo Reply ke IP korban sehingga membanjiri bandwidth dan sumber daya korban.

## Tugas 6 (Pertemuan 6): SYN Flood dan Three-Way Handshake

- Stack `b/b3-syn-flood/docker-compose.yml` dijalankan dengan project `-p k5` (`k5_lab`, `k5-target-1`, `k5-attacker-1`), lalu dibersihkan dengan `down -v`.
- Jumlah koneksi `SYN_RECV` pada container `target` meningkat dari **`0`** sebelum serangan menjadi **`1`** saat `hping3 -S -p 80 --flood target` dijalankan dari `attacker`.
- Penangkapan paket TCP port 80 dengan `tcpdump -i eth0 -nn tcp port 80` (`12 packets captured`) saat `curl -s http://target > /dev/null` menunjukkan three-way handshake:
  - **SYN**: `172.20.0.3.57194 > 172.20.0.2.80: Flags [S], seq 2732426339`
  - **SYN-ACK**: `172.20.0.2.80 > 172.20.0.3.57194: Flags [S.], seq 2270967199, ack 2732426340`
  - **ACK**: `172.20.0.3.57194 > 172.20.0.2.80: Flags [.], ack 1` (diikuti `HTTP: GET / HTTP/1.1` dan `HTTP: HTTP/1.1 200 OK`).
