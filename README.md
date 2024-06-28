# LAPORAN RESMI FUN PROJECT

## Anggota Kelompok

1. Aryasatya Alaauddin 5027231082
2. Diandra Naufal Abror 5027231004
3. Muhamad Rizq Taufan 5027231021

## discorit.c
### 1. _Header_ dan Deklarasi
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <errno.h>
#include <crypt.h>

#define PORT 8080
#define BUF_SIZE 1024

void handle_commands(int sock, const char *username);
```
Kumpulan header library yang kami gunakan untuk mengerjakan _final project_ ini. Dilanjut dengan pendefinisian `PORT` untuk koneksi _socket_ serta ukuran _buffer_, fungsi `handle_commands` akan menangani _input_ pengguna setelah _login_.

### 2. Fungsi `main` untuk Menginisialisasi _Socket_ dan Registrasi Pengguna
```
int main(int argc, char *argv[]) {
    if (argc < 4) {
        fprintf(stderr, "Usage: %s REGISTER|LOGIN username -p password\n", argv[0]);
        return -1;
    }

    struct sockaddr_in address;
    int sock = 0;
    struct sockaddr_in serv_addr;
    char buffer[BUF_SIZE] = {0};

    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("Socket creation error");
        return -1;
    }

    address.sin_family = AF_INET;
    address.sin_port = htons(PORT);

    if (inet_pton(AF_INET, "127.0.0.1", &address.sin_addr) <= 0) {
        perror("Invalid address/ Address not supported");
        return -1;
    }

    if (connect(sock, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("Connection Failed");
        return -1;
    }

    const char *username = argv[2];
    const char *password = argv[4];

    if (strcmp(argv[1], "REGISTER") == 0) {
        snprintf(buffer, sizeof(buffer), "REGISTER %s -p %s", username, password);
        send(sock, buffer, strlen(buffer), 0);
        read(sock, buffer, BUF_SIZE);
        printf("%s\n", buffer);
    } else if (strcmp(argv[1], "LOGIN") == 0) {
        snprintf(buffer, sizeof(buffer), "LOGIN %s -p %s", username, password);
        send(sock, buffer, strlen(buffer), 0);
        read(sock, buffer, BUF_SIZE);
        if (strstr(buffer, "berhasil login")) {
            printf("%s\n", buffer);
            handle_commands(sock, username);
        } else {
            printf("Login gagal\n");
        }
    } else {
        fprintf(stderr, "Invalid command. Use REGISTER or LOGIN.\n");
    }

    close(sock);
    return 0;
}
```
Fungsi tersebut bertujuan menangani _command line_ registrasi dan _login_ pengguna dengan menginisialisasi koneksi _socket_ ke **server.c**, mengirim perintah `REGISTER` juga `LOGIN`, dan menangani respons _server_. Jika proses login berhasil, maka akan memanggil `handle_command` untuk mengelola interaksi pengguna selanjutnya. Di sisi lain, fungsi tersebut juga menangani _error_ koneksi dan memastikan _socket_ ditutup sebelum program berakhir.

### 3. Fungsi untuk Menampilkan _Prompt_ sesuai Konteks Pengguna
```
void handle_channel_prompt(int sock, const char *username, const char *channel) {
    char buffer[BUF_SIZE];
    snprintf(buffer, sizeof(buffer), "%s/%s", username, channel);
    printf("[%s] ", buffer);
    fflush(stdout);
}

void handle_room_prompt(int sock, const char *username, const char *channel, const char *room) {
    char buffer[BUF_SIZE];
    snprintf(buffer, sizeof(buffer), "%s/%s/%s", username, channel, room);
    printf("[%s] ", buffer);
    fflush(stdout);
}
```
Fungsi `handle_channel_prompt` berguna untuk menampilkan _prompt_ ketika pengguna berada di dalam _"channel"_, sedangkan `handle_room_prompt` untuk menampilkan _prompt_ di dalam _"room"_. _Prompt_ akan berguna bagi pengguna untuk memahami konteks struktur DiscorIT dengan menampilkan kombinasi _username_, _channel_, dan _room_.

### 4. Fungsi `handle_commands`
```
void handle_commands(int sock, const char *username) {
    char buffer[BUF_SIZE];
    char channel[BUF_SIZE] = "";
    char key[BUF_SIZE];
    char room[BUF_SIZE] = "";
    int key_prompt = 0;
    int in_channel = 0;
    int in_room = 0;
    char new_username[BUF_SIZE];
```
Merupakan inti dari interaksi pengguna dalam DiscorIT dengan mengelola _loop_ utama yang terus menerima _input_ dari user, menghubungkannya ke **server.c**, dan menangani respons sesuai _state_ pengguna seperti `JOIN` atau `EXIT` serta memperbarui informasi lokal seperti _username_, nama _channel_, nama _room_, dan pesan.
```
    while (1) {
        if (in_channel && in_room) {
            handle_room_prompt(sock, username, channel, room);
        } else if (in_channel) {
            handle_channel_prompt(sock, username, channel);
        } else if (key_prompt) {
            printf("Key: ");
            fflush(stdout);
            key_prompt = 0;
        } else {
            printf("[%s] ", username);
        }
```
Logika untuk menampilkan _prompt_ kepada pengguna sesuai _state user_ (apakah sedang berada di _channel_ atau _room_ atau memerlukan _input key_) lalu akan memanggilnya. Vital untuk pengguna agar mereka selalu memiliki konteks visual tentang posisi mereka dalam struktur DisorIT.
```
fgets(buffer, BUF_SIZE, stdin);
        buffer[strcspn(buffer, "\n")] = 0; // Remove newline character

        if (strcmp(buffer, "EXIT") == 0) {
            if (in_room) {
                send(sock, buffer, strlen(buffer), 0);
                in_room = 0;
                memset(room, 0, sizeof(room));
            } else if (in_channel) {
                send(sock, buffer, strlen(buffer), 0);
                in_channel = 0;
                memset(channel, 0, sizeof(channel));
            } else {
                send(sock, buffer, strlen(buffer), 0);
                break;
            }

            memset(buffer, 0, sizeof(buffer));
            int bytes_read = read(sock, buffer, BUF_SIZE);
            buffer[bytes_read] = '\0';

            if (strstr(buffer, "Keluar Room")) {
                printf("%s\n", buffer);
            } else if (strstr(buffer, "Keluar Channel")) {
                printf("%s\n", buffer);
            } else {
                printf("%s\n", buffer);
            }
        } else {
            char command[BUF_SIZE];
            sscanf(buffer, "%s", command);

            if (strcmp(command, "JOIN") == 0) {
                char arg[BUF_SIZE];
                sscanf(buffer, "%*s %s", arg);

                if (in_channel) {
                    // Join room
                    snprintf(buffer, sizeof(buffer), "JOIN %s", arg);
                    in_room = 1;
                } else {
                    // Join channel
                    snprintf(buffer, sizeof(buffer), "JOIN %s", arg);
                }
            }

            send(sock, buffer, strlen(buffer), 0);

            memset(buffer, 0, sizeof(buffer));
            int bytes_read = read(sock, buffer, BUF_SIZE);
            buffer[bytes_read] = '\0';

            if (strstr(buffer, "bergabung dengan channel")) {
                char *channel_name = strstr(buffer, "bergabung dengan channel ") + strlen("bergabung dengan channel ");
                strcpy(channel, channel_name);
                in_channel = 1;
            } else if (strstr(buffer, "bergabung dengan room")) {
                char *room_name = strstr(buffer, "bergabung dengan room ") + strlen("bergabung dengan room ");
                strcpy(room, room_name);
                in_room = 1;
            } else if (strstr(buffer, "Key: ")) {
                key_prompt = 1;
            } else if (strstr(buffer, "Keluar Channel")) {
                in_channel = 0;
                memset(channel, 0, sizeof(channel));
            } else if (strstr(buffer, "Keluar Room")) {
                in_room = 0;
                memset(room, 0, sizeof(room));
            } else if (strstr(buffer, "Channel tidak ditemukan")) {
                printf("%s\n", buffer);
                memset(channel, 0, sizeof(channel));
                in_channel = 0;
            } else if (strstr(buffer, "Anda dibanned dari channel ini")) {
                printf("%s\n", buffer);
                memset(channel, 0, sizeof(channel));
                in_channel = 0;
            } else if (strstr(buffer, "Room tidak ditemukan")) {
                printf("%s\n", buffer);
                memset(room, 0, sizeof(room));
                in_room = 0;
            } else if (strstr(buffer, "berhasil diubah menjadi")){
                char *new_username = strstr(buffer, "berhasil diubah menjadi ") + strlen("berhasil diubah menjadi ");
                strcpy(username, new_username);
            } else if (strstr(buffer, "nama room berubah menjadi")){
                char *new_room = strstr(buffer, "nama room berubah menjadi ") + strlen("nama room berubah menjadi ");
                strcpy(room, new_room);
            } else if (strstr(buffer, "nama channel berubah menjadi")){
                char *new_channel = strstr(buffer, "nama channel berubah menjadi ") + strlen("nama channel berubah menjadi ");
                strcpy(channel, new_channel);
            } else if (strstr(buffer, "Chat Baru")) {
                printf("%s\n", buffer);
                memset(buffer, 0, sizeof(buffer));
            } else if (strstr(buffer, "Chat Diubah")) {
                printf("%s\n", buffer);
                memset(buffer, 0, sizeof(buffer));
            } else if (strstr(buffer, "Chat Dihapus")) {
                printf("%s\n", buffer);
                memset(buffer, 0, sizeof(buffer));
            } else {
                printf("%s\n", buffer);
            }
        }
    }
}
```
_Loop_ utama yang menangani _input_ pengguna serta interaksi dengan **server.c**. Prosesnya adalah membaca _input_ pengguna > mengirimkannya ke _server_ > memprosesnya. Terdapat beberapa respons seperti _username_, nama _channel_ atau _room_, pesan baru, pesan yang diubah atau yang telah dihapus.

## monitor.c
berfungsi sebagai klien chat yang dapat menghubungkan diri ke server, login dengan menggunakan username dan password, dan memantau percakapan dalam chat room tertentu
### Definisi Konstanta dan Struktur
````
#define PORT 8080
#define BUF_SIZE 1024
#define DISCORIT_DIR "/home/ax3lrod/sisop/fp/DiscorIT"

typedef struct {
    char channel[BUF_SIZE];
    char room[BUF_SIZE];
    int sock;
} ChatMonitorArgs;

````
PORT adalah port yang digunakan untuk koneksi ke server.
BUF_SIZE adalah ukuran buffer yang digunakan untuk berbagai operasi string.
DISCORIT_DIR adalah direktori tempat file chat disimpan.
ChatMonitorArgs adalah struktur yang digunakan untuk menyimpan argumen yang dilewatkan ke thread pemantauan chat.


### Fungsi 'display-chat'
````
void display_chat(const char *channel, const char *room) {
    // ...
}

````
Fungsi ini membaca file chat CSV untuk channel dan room tertentu, kemudian menampilkan isinya ke layar.

### Fungsi 'monitor_chat'
````
void *monitor_chat(void *arg) {
    // ...
}
````
Fungsi ini dijalankan dalam thread terpisah untuk memantau perubahan file chat. Jika file chat berubah (diperbarui), fungsi ini akan memanggil display_chat untuk menampilkan isi chat yang terbaru.

### Fungsi 'handle_commands"
````
void handle_commands(int sock, const char *username) {
    // ...
}
````
Fungsi ini menangani perintah yang diterima dari server. Termasuk memulai thread pemantauan chat jika channel dan room berubah, dan menampilkan pesan dari server.

### Fungsi 'main"
````
int main(int argc, char *argv[]) {
    // ...
}
````
Fungsi ini adalah titik masuk program.
Memeriksa argumen program untuk login (LOGIN username -p password).
Menginisialisasi socket dan menghubungkan ke server.
Mengirim perintah login ke server dan membaca responsnya.
Jika login berhasil, fungsi handle_commands dipanggil untuk menangani interaksi lebih lanjut dengan server.

### Alur Program Utama:

Program memeriksa argumen command line untuk login.
Membuat socket dan menghubungkan ke server.
Mengirimkan perintah login ke server dan menunggu respons.
Jika login berhasil, program menunggu perintah dari server, termasuk perintah untuk mengganti channel dan room, serta perintah untuk keluar.
Jika channel dan room baru diterima, program memulai thread untuk memantau perubahan pada file chat dan menampilkan isi chat.

### Pemantauan dan Tampilan Chat:

Thread pemantauan (dijalankan oleh monitor_chat) akan terus memeriksa perubahan pada file chat setiap detik.
Jika ada perubahan, isi chat akan dibaca ulang dan ditampilkan ke layar dengan fungsi display_chat.

### Penghentian Program:

Jika perintah EXIT diterima dari server, atau jika thread pemantauan diakhiri, program akan keluar dari loop dan menutup socket sebelum berhenti.

Secara keseluruhan, kode ini berfungsi sebagai client untuk memonitor dan menampilkan pesan chat dari server dalam format yang terstruktur menggunakan thread untuk pemantauan file chat yang berubah.

## server.c
### 1. Header dan Deklarasi
````
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <pthread.h>
#include <crypt.h>
#include <errno.h>
#include <time.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <syslog.h>
#include <time.h>
#include <stdbool.h>
#include <dirent.h>
#include <signal.h>

#define PORT 8080
#define BUF_SIZE 1024
#define MAX_CLIENTS 100
#define USER_FILE "/home/ax3lrod/sisop/fp/DiscorIT/users.csv"
#define DISCORIT_DIR "/home/ax3lrod/sisop/fp/DiscorIT"
#define CHANNEL_FILE "/home/ax3lrod/sisop/fp/DiscorIT/channels.csv"

#define MAX_USERS 100
//int logged_in_users[MAX_USERS] = {0};

typedef struct {
    int regular;
    int monitor;
} UserLoginStatus;

UserLoginStatus logged_in_users[MAX_USERS] = {0};

typedef struct {
    int logged_in;
    int user_id;
    char username[BUF_SIZE];
    char role[BUF_SIZE];
    int in_channel;
    char current_channel[BUF_SIZE];
    int in_room;
    char current_room[BUF_SIZE];
    int is_monitor;
} Session;

typedef struct {
    int logged_in;
} monitorSession;

typedef struct {
    int socket;
    bool is_monitor;
    bool logged_in;
    char username[BUF_SIZE];
} Client;

Client clients[MAX_CLIENTS] = {0};
int client_count = 0;

````
- #include mencakup berbagai header file yang diperlukan untuk operasi jaringan, threading, enkripsi, logging, dan operasi sistem file.
- #define digunakan untuk mendefinisikan konstanta seperti port server, ukuran buffer, jumlah maksimal klien, dan lokasi file yang menyimpan data pengguna dan saluran.
- UserLoginStatus: Menyimpan status login pengguna biasa dan monitor.
- Session: Menyimpan informasi sesi untuk setiap pengguna yang login, termasuk status login, ID pengguna, username, role, saluran dan ruangan saat ini, serta status monitor.
- monitorSession: Struktur untuk menyimpan status login monitor.
- Client: Menyimpan informasi untuk setiap klien yang terhubung, termasuk socket, status monitor, status login, dan username.
- logged_in_users: Array yang menyimpan status login pengguna berdasarkan ID.
- clients: Array yang menyimpan informasi klien yang terhubung.
- client_count: Menyimpan jumlah klien yang terhubung saat ini.
- Prototipe untuk berbagai fungsi yang digunakan dalam server untuk menangani klien, mengelola saluran dan ruangan, mengedit informasi pengguna, dan mengirim pesan.
- handle_client(void *arg): Fungsi untuk menangani komunikasi dengan klien.
- bcrypt(const char *password): Fungsi untuk mengenkripsi kata sandi menggunakan bcrypt.
- Fungsi lain untuk mengelola saluran, ruangan, dan pengguna, serta untuk mengirim dan mengedit pesan.

### 2. Deskripsi Fungsi
````
void *handle_client(void *arg);
char *bcrypt(const char *password);
void list_channels(int socket);
void create_channel(int socket, const char *username, int id, const char *channel_name, const char *key);
void edit_channel(int socket, const char *username, const char *channel_name, const char *new_channel_name);
void delete_channel(int socket, const char *channel_name);
void kick_user(int socket, const char *channel_name, const char *username);
void join_channel(int socket, const char *username, const char *channel, int id,const char *key);
void create_room(int socket, const char *username, const char *channel, const char *room);
void join_room(int socket, const char *username, const char *channel, const char *room);
void edit_room(int socket, const char *username, const char *channel, const char *room, const char *new_room);
void delete_room(int socket, const char *channel, const char *room, const char *username);
void delete_all_rooms(int socket, const char *channel, const char *username);
void list_rooms(int socket, const char *channel);
void list_users(int socket);
void list_channel_users(int socket, const char *channel);
void edit_user_name(int socket, const char *username, const char *new_username);
void edit_user_name_other(int socket, const char *username, const char *new_username);
void edit_user_password(int socket, const char *username, const char *new_password);
void remove_user(int socket, const char *username);
void ban_user(int socket, const char *channel, const char *username);
void unban_user(int socket, const char *channel, const char *username);
void chat(int socket, const char *channel, const char *room, const char *username, const char *message);
void edit_chat(int socket, const char *channel, const char *room, const char *username, int id_chat, const char *new_message);
void delete_chat(int socket, const char *channel, const char *room, int id_chat);
void see_chat(int socket, const char *channel, const char *room);
//void channel_room_info_for_monitor(int socket, const char *channel, const char *room);

/*void broadcast_message(const char *message) {
    for (int i = 0; i < client_count; i++) {
        if (clients[i].socket != 0) {
            send(clients[i].socket, message, strlen(message), 0);
        }
    }
}*/
````
Channel Management:
- list_channels: Menampilkan daftar saluran yang tersedia.
- create_channel: Membuat saluran baru.
- edit_channel: Mengubah nama saluran yang ada.
- delete_channel: Menghapus saluran.
- kick_user: Mengeluarkan pengguna dari saluran.
- join_channel: Bergabung dengan saluran yang ada.

Room Management:
- create_room: Membuat ruangan dalam saluran tertentu.
- join_room: Bergabung dengan ruangan dalam saluran tertentu.
- edit_room: Mengubah nama ruangan yang ada.
- delete_room: Menghapus ruangan.
- delete_all_rooms: Menghapus semua ruangan dalam saluran tertentu.
- list_rooms: Menampilkan daftar ruangan dalam saluran tertentu.

User Management:
- list_users: Menampilkan daftar pengguna.
- list_channel_users: Menampilkan daftar pengguna dalam saluran tertentu.
- edit_user_name: Mengubah nama pengguna sendiri.
- edit_user_name_other: Mengubah nama pengguna lain (oleh monitor).
- edit_user_password: Mengubah kata sandi pengguna.
- remove_user: Menghapus pengguna.
- ban_user: Melarang pengguna dari saluran tertentu.
- unban_user: Mengizinkan kembali pengguna yang dilarang dari saluran.

Chat Management:
- chat: Mengirim pesan dalam ruangan tertentu.
- edit_chat: Mengedit pesan yang sudah dikirim.
- delete_chat: Menghapus pesan yang sudah dikirim.
- see_chat: Melihat pesan dalam ruangan tertentu.

### 3. Fungsi `remove_directory`
````
int remove_directory(const char *path) {
    DIR *d = opendir(path);
    size_t path_len = strlen(path);
    int r = -1;

    if (d) {
        struct dirent *p;
        r = 0;
        while (!r && (p = readdir(d))) {
            int r2 = -1;
            char *buf;
            size_t len;

            // Skip the names "." and ".." as we don't want to recurse on them
            if (!strcmp(p->d_name, ".") || !strcmp(p->d_name, ".."))
                continue;

            len = path_len + strlen(p->d_name) + 2;
            buf = malloc(len);

            if (buf) {
                struct stat statbuf;
                snprintf(buf, len, "%s/%s", path, p->d_name);

                if (!stat(buf, &statbuf)) {
                    if (S_ISDIR(statbuf.st_mode))
                        r2 = remove_directory(buf);
                    else
                        r2 = unlink(buf);
                }
                free(buf);
            }
            r = r2;
        }
        closedir(d);
    }

    if (!r)
        r = rmdir(path);

    return r;
}
````
fungsi remove_directory yang bertujuan untuk menghapus direktori beserta isinya, termasuk subdirektori dan file yang ada di dalamnya. Fungsi ini secara rekursif menghapus semua file dan subdirektori sebelum akhirnya menghapus direktori itu sendiri. 

#### Deklarasi dan inisialisasi
````
int remove_directory(const char *path) {
    DIR *d = opendir(path);
    size_t path_len = strlen(path);
    int r = -1;
````
- DIR *d = opendir(path);: Membuka direktori yang path-nya diberikan sebagai argumen dan mengembalikan pointer ke struktur direktori.
- size_t path_len = strlen(path);: Menghitung panjang string dari path direktori.
- int r = -1;: Inisialisasi variabel r dengan nilai -1, yang menunjukkan kegagalan operasi jika tidak berubah.

#### Pemeriksaan Direktori
````
if (d) {
    struct dirent *p;
    r = 0;

````
Jika direktori berhasil dibuka (d tidak NULL), inisialisasi variabel p dari tipe struct dirent dan set r ke 0, yang menandakan keberhasilan.

#### Pembacaan Direktori
````
    while (!r && (p = readdir(d))) {
        int r2 = -1;
        char *buf;
        size_t len;

````
- while (!r && (p = readdir(d))): Iterasi melalui setiap entri dalam direktori selama r tetap 0 dan masih ada entri yang bisa dibaca.
- int r2 = -1;: Inisialisasi variabel r2 dengan -1 untuk memeriksa hasil penghapusan entri.
- char *buf;: Pointer untuk menyimpan path lengkap dari entri yang sedang diproses.
- size_t len;: Menyimpan panjang dari path lengkap.

#### Melewati Entri Khusus
````
        if (!strcmp(p->d_name, ".") || !strcmp(p->d_name, ".."))
            continue;

````
Memeriksa dan melewati entri khusus "." dan ".." yang merepresentasikan direktori saat ini dan direktori induk.

#### Penggabungan Path dan Penghapusan
````
len = path_len + strlen(p->d_name) + 2;
        buf = malloc(len);

        if (buf) {
            struct stat statbuf;
            snprintf(buf, len, "%s/%s", path, p->d_name);

            if (!stat(buf, &statbuf)) {
                if (S_ISDIR(statbuf.st_mode))
                    r2 = remove_directory(buf);
                else
                    r2 = unlink(buf);
            }
            free(buf);
        }
        r = r2;
    }
    closedir(d);
````
- len = path_len + strlen(p->d_name) + 2;: Menghitung panjang buffer yang diperlukan untuk path lengkap.
- buf = malloc(len);: Mengalokasikan memori untuk buffer.
- snprintf(buf, len, "%s/%s", path, p->d_name);: Menggabungkan path direktori dengan nama entri untuk membentuk path lengkap.
- stat(buf, &statbuf): Mendapatkan informasi status dari entri.
- if (S_ISDIR(statbuf.st_mode)): Jika entri adalah direktori, panggil remove_directory secara rekursif.
- else unlink(buf);: Jika entri adalah file, hapus file tersebut.
- free(buf);: Membebaskan memori yang dialokasikan untuk buffer.
- r = r2;: Set nilai r ke hasil dari operasi penghapusan entri.

#### Menutup Direktori dan Menghapus Direktori
````
    closedir(d);
}

if (!r)
    r = rmdir(path);

return r;
````
- closedir(d);: Menutup direktori setelah selesai diproses.
- if (!r) r = rmdir(path);: Jika semua entri berhasil dihapus (r adalah 0), hapus direktori itu sendiri.
- return r;: Mengembalikan nilai r yang menunjukkan keberhasilan (0) atau kegagalan (nilai negatif) operasi penghapusan direktori.

### 4. Fungsi `get_timestamp`
````
void get_timestamp(char *buffer, size_t buffer_size) {
    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    strftime(buffer, buffer_size, "[%d/%m/%Y %H:%M:%S]", t);
}
````
Fungsi ini menghasilkan timestamp dalam format [dd/mm/yyyy hh:mm:ss] dan menyimpannya di buffer yang diberikan.

time_t now = time(NULL);: Mendapatkan waktu saat ini.
struct tm *t = localtime(&now);: Mengonversi waktu ke format lokal.
strftime(buffer, buffer_size, "[%d/%m/%Y %H:%M:%S]", t);: Memformat waktu dan menyimpannya dalam buffer.

### 5. Fungsi `log_action`
````
void log_action(const char *channel_name, const char *event) {
    char log_file_path[BUF_SIZE];
    snprintf(log_file_path, sizeof(log_file_path), "%s/%s/admin/user.log", DISCORIT_DIR, channel_name);
    FILE *log_file = fopen(log_file_path, "a");
    if (!log_file) {
        perror("fopen");
        return;
    }

    char timestamp[BUF_SIZE];
    get_timestamp(timestamp, sizeof(timestamp));

    fprintf(log_file, "%s %s\n", timestamp, event);
    fclose(log_file);
}
````
Fungsi ini mencatat sebuah aksi ke file log (user.log) yang terletak di dalam folder admin pada direktori saluran tertentu.

- char log_file_path[BUF_SIZE];: Buffer untuk path file log.
- snprintf(log_file_path, sizeof(log_file_path), "%s/%s/admin/user.log", DISCORIT_DIR, channel_name);: Menggabungkan direktori dan nama saluran untuk membuat path lengkap ke file log.
- FILE *log_file = fopen(log_file_path, "a");: Membuka file log dalam mode append. Jika gagal, menampilkan pesan error dan keluar dari fungsi.
- char timestamp[BUF_SIZE];: Buffer untuk menyimpan timestamp.
- get_timestamp(timestamp, sizeof(timestamp));: Mendapatkan timestamp saat ini.
- fprintf(log_file, "%s %s\n", timestamp, event);: Menulis timestamp dan event ke file log.
- fclose(log_file);: Menutup file log.

### 6. Fungsi `daemonize`
````
void daemonize() {
    pid_t pid;

    pid = fork();

    if (pid < 0) {
        perror("Fork failed");
        exit(EXIT_FAILURE);
    }

    if (pid > 0) {
        exit(EXIT_SUCCESS);
    }

    if (setsid() < 0) {
        perror("setsid failed");
        exit(EXIT_FAILURE);
    }

    signal(SIGCHLD, SIG_IGN);

    pid = fork();

    if (pid < 0) {
        perror("Fork failed");
        exit(EXIT_FAILURE);
    }

    if (pid > 0) {
        exit(EXIT_SUCCESS);
    }

    umask(0);

    if (chdir("/") < 0) {
        perror("chdir failed");
        exit(EXIT_FAILURE);
    }

    open("/dev/null", O_RDWR);
    dup(0);
    dup(0);
}
````
Fungsi ini mengubah proses yang sedang berjalan menjadi daemon, yaitu sebuah proses yang berjalan di latar belakang tanpa kontrol terminal.

Fork the Parent Process:
- pid = fork();: Membuat proses anak.
- Jika fork gagal (pid < 0), menampilkan pesan error dan keluar.
- Jika pid > 0, ini adalah proses induk yang keluar dengan status sukses.

Set Session Leader:
- if (setsid() < 0): Proses anak menjadi pemimpin sesi baru.
- Jika setsid gagal, menampilkan pesan error dan keluar.

Ignore SIGCHLD:
- signal(SIGCHLD, SIG_IGN);: Mengabaikan sinyal dari proses anak yang berakhir.

Fork Again:
- pid = fork();: Membuat proses anak kedua.
- Jika fork gagal, menampilkan pesan error dan keluar.
- Jika pid > 0, proses induk kedua keluar dengan status sukses.

Set File Permissions:
- umask(0);: Mengatur mode file mask untuk memastikan file yang dibuat memiliki izin yang tepat.

Change Working Directory:
- if (chdir("/") < 0): Mengubah direktori kerja ke root untuk memastikan proses daemon tidak mengunci direktori yang sedang berjalan.
- Jika chdir gagal, menampilkan pesan error dan keluar.

Close File Descriptors:
- open("/dev/null", O_RDWR);: Membuka /dev/null untuk input/output standar.
- dup(0);: Menyalin file descriptor stdin ke stdout.
- dup(0);: Menyalin file descriptor stdin ke stderr.

Fungsi ini memastikan bahwa daemon tidak terkait dengan terminal dan dapat berjalan di latar belakang tanpa interaksi langsung dari pengguna.

### 7. Fungsi `main`
````
int main() {
    daemonize();

    int server_fd, new_socket;
    struct sockaddr_in address;
    int addrlen = sizeof(address);
    
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    if(setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &(int){1}, sizeof(int)) < 0) {
        perror("setsockopt(SO_REUSEADDR) failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    if(setsockopt(server_fd, SOL_SOCKET, SO_REUSEPORT, &(int){1}, sizeof(int)) < 0) {
        perror("setsockopt(SO_REUSEPORT) failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);
    
    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }
    if (listen(server_fd, 3) < 0) {
        perror("listen");
        close(server_fd);
        exit(EXIT_FAILURE);
    }
    
    while (1) {
        if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen)) < 0) {
            perror("accept");
            continue;
        }
        pthread_t thread_id;
        pthread_create(&thread_id, NULL, handle_client, (void *)&new_socket);
    }

    return 0;
}
````
#### Daemonisasi:
````
daemonize();
````
Memanggil fungsi daemonize untuk membuat proses menjadi daemon.

#### Membuat Socket:
````
if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
    perror("socket failed");
    exit(EXIT_FAILURE);
}
````
Membuat socket file descriptor untuk komunikasi TCP.

#### Mengatur Opsi Socket:
````
if(setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &(int){1}, sizeof(int)) < 0) {
    perror("setsockopt(SO_REUSEADDR) failed");
    close(server_fd);
    exit(EXIT_FAILURE);
}
if(setsockopt(server_fd, SOL_SOCKET, SO_REUSEPORT, &(int){1}, sizeof(int)) < 0) {
    perror("setsockopt(SO_REUSEPORT) failed");
    close(server_fd);
    exit(EXIT_FAILURE);
}
````
Mengatur opsi socket untuk memungkinkan penggunaan kembali alamat dan port.

#### Binding:
````
address.sin_family = AF_INET;
address.sin_addr.s_addr = INADDR_ANY;
address.sin_port = htons(PORT);

if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
    perror("bind failed");
    close(server_fd);
    exit(EXIT_FAILURE);
}
````
Mengaitkan socket ke alamat dan port tertentu.

#### Mendengarkan Koneksi:
````
if (listen(server_fd, 3) < 0) {
    perror("listen");
    close(server_fd);
    exit(EXIT_FAILURE);
}
````
Menempatkan socket dalam mode mendengarkan untuk menerima koneksi masuk.

#### Menerima dan Menangani Koneksi Klien:
````
while (1) {
    if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen)) < 0) {
        perror("accept");
        continue;
    }
    pthread_t thread_id;
    pthread_create(&thread_id, NULL, handle_client, (void *)&new_socket);
}
````
Menerima koneksi masuk dan membuat thread baru untuk menangani setiap klien menggunakan fungsi handle_client.

### 8. Fungsi `send_to_monitor`
````
void send_to_monitor(const char *message) {
    for (int i = 0; i < client_count; i++) {
        if (clients[i].socket != 0 && clients[i].is_monitor) {
            send(clients[i].socket, message, strlen(message), 0);
        }
    }
}
````
Penjelasan:
Fungsi ini mengirim pesan ke semua klien yang bertindak sebagai monitor.
Iterasi melalui semua klien dan mengirim pesan jika klien adalah monitor.

### 9. Fungsi `find_monitor_client`
````
int find_monitor_client(const char *username) {
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i].socket != 0 && clients[i].is_monitor && strcmp(clients[i].username, username) == 0) {
            return i;
        }
    }
    return -1;
}
````
Penjelasan:
Fungsi ini mencari klien monitor berdasarkan nama pengguna.
Mengembalikan indeks klien jika ditemukan, atau -1 jika tidak ditemukan.

### 10. Fungsi `send_message_to_monitor`
````
void send_message_to_monitor(Session *session, const char *message) {
    int monitor_index = find_monitor_client(session->username);
    if (monitor_index != -1) {
        send(clients[monitor_index].socket, message, strlen(message), 0);
    }
}
````
Penjelasan:
Fungsi ini mengirim pesan ke klien monitor yang sesuai dengan nama pengguna dalam sesi.
Menggunakan find_monitor_client untuk menemukan klien dan mengirim pesan jika ditemukan.

### 11. Fungsi `close_monitor_sessions`
````
void close_monitor_sessions() {
    for (int i = 0; i < client_count; i++) {
        if (clients[i].socket != 0 && clients[i].is_monitor) {
            close(clients[i].socket);
            clients[i].socket = 0;
            clients[i].is_monitor = false;
            client_count--;
        }
    }
}
````
Penjelasan:
Fungsi ini menutup semua sesi klien yang bertindak sebagai monitor.
Iterasi melalui semua klien, menutup socket klien yang adalah monitor, dan mengatur ulang status klien.

### 12. Fungsi `is_logged_in`
````
bool is_logged_in(const char *username) {
    for (int i = 0; i < client_count; i++) {
        if (clients[i].logged_in && strcmp(clients[i].username, username) == 0) {
            return 1;
        }
    }
    return 0;
}\
````
Penjelasan:
Fungsi ini memeriksa apakah pengguna dengan username tertentu sudah login.
Melakukan iterasi melalui daftar klien (clients), jika ditemukan klien yang sudah login dengan username yang cocok, mengembalikan true (1). Jika tidak ditemukan, mengembalikan false (0).

### 13. Fungsi `handle_client`
````
void *handle_client(void *arg) {
    int socket = *(int *)arg;
    char buffer[BUF_SIZE];
    int bytes_read;
    Session session = {0}; // Initialize session
    monitorSession monitor_session = {0}; // Initialize monitor session 
    int client_index = -1;

    // Add client to the list
    for (int i = 0; i < MAX_CLIENTS; i++) {
        if (clients[i].socket == 0) {
            clients[i].socket = socket;
            client_index = i;
            client_count++;
            break;
        }
    }

    //printf("Client connected\n"); // Debugging

    while ((bytes_read = read(socket, buffer, BUF_SIZE)) > 0) {
        buffer[bytes_read] = '\0';
        //printf("Received: %s\n", buffer);

        char command[BUF_SIZE], username[BUF_SIZE], password[BUF_SIZE];
        sscanf(buffer, "%s %s -p %s", command, username, password);

        if (strcmp(command, "REGISTER") == 0) {
            if (register_user(username, password)) {
                snprintf(buffer, sizeof(buffer), "%s berhasil register", username);
            } else {
                snprintf(buffer, sizeof(buffer), "%s sudah terdaftar", username);
            }
        } else if (strcmp(command, "LOGIN") == 0) {
            int login_result = login_user(username, password, &session, 0);
            if (login_result == -1) {
                snprintf(buffer, sizeof(buffer), "User sudah login sebagai regular user");
            } else if (login_result == 1) {
                snprintf(buffer, sizeof(buffer), "%s berhasil login", username);
                clients[client_index].logged_in = true;
                clients[client_index].is_monitor = false;
                strcpy(clients[client_index].username, username);
            } else {
                snprintf(buffer, sizeof(buffer), "Login gagal");
            }
        } else if (strcmp(command, "LOGIN_MONITOR") == 0) {
            int login_result = login_user(username, password, &session, 1);
            if (login_result == -1) {
                snprintf(buffer, sizeof(buffer), "User sudah login sebagai monitor");
            } else if (login_result == 1) {
                snprintf(buffer, sizeof(buffer), "%s berhasil login sebagai monitor", username);
                clients[client_index].is_monitor = true;
                clients[client_index].logged_in = true;  // Set this to true for monitors as well
                strcpy(clients[client_index].username, username);
                monitor_session.logged_in = 1;
            } else {
                snprintf(buffer, sizeof(buffer), "Login gagal");
            }
        } else if (session.logged_in) {
            // Process other commands only if logged in
            process_command(socket, &session, buffer, &monitor_session);
            memset(buffer, 0, sizeof(buffer));
        } else {
            snprintf(buffer, sizeof(buffer), "Anda harus login terlebih dahulu");
        }

        write(socket, buffer, strlen(buffer));
        memset(buffer, 0, sizeof(buffer));
    }

    //printf("Client disconnected\n"); // Debugging
    close(socket);

    // Remove client from the list
    if (clients[client_index].logged_in) {
        logout_user(&session);
    }
    clients[client_index].socket = 0;
    clients[client_index].is_monitor = false;
    clients[client_index].logged_in = false;
    strcpy(clients[client_index].username, "");
    client_count--;

    return NULL;
}
````
Penjelasan:

Inisialisasi:
- Socket klien diterima sebagai argumen.
- Buffer untuk membaca data dari klien dan variabel lainnya diinisialisasi.
- Session dan monitorSession diinisialisasi untuk menyimpan informasi sesi dan monitor.
- Mencari indeks klien yang kosong untuk menambahkan klien baru ke dalam daftar clients.

Menerima dan Memproses Perintah Klien:
- Loop membaca data dari socket klien.
- Data yang diterima diparsing untuk mendapatkan perintah, username, dan password.
- Memproses perintah REGISTER, LOGIN, dan LOGIN_MONITOR.
- Jika perintah adalah REGISTER, memanggil register_user dan mengirimkan pesan berhasil atau gagal.
- Jika perintah adalah LOGIN, memanggil login_user dan mengirimkan pesan berhasil atau gagal, serta mengatur status login klien dalam daftar clients.
- Jika perintah adalah LOGIN_MONITOR, memanggil login_user untuk login sebagai monitor dan mengatur status monitor dan login klien dalam daftar clients.
- Jika pengguna sudah login, memproses perintah lainnya dengan memanggil process_command.
- Jika pengguna belum login, mengirimkan pesan bahwa pengguna harus login terlebih dahulu.

Menutup Koneksi Klien:
- Jika koneksi klien terputus, socket ditutup dan status klien dalam daftar clients direset.
- Jika klien sudah login, memanggil logout_user untuk keluar dari sesi.
- Mengurangi jumlah klien yang terhubung (client_count).

### 14. Fungsi `register_user`
````
int register_user(const char *username, const char *password) {
    FILE *file = fopen(USER_FILE, "r");
    int max_id = 0;

    if (file) {
        char line[BUF_SIZE];
        while (fgets(line, sizeof(line), file)) {
            int id;
            char stored_username[BUF_SIZE];
            if (sscanf(line, "%d,%[^,]", &id, stored_username) == 2) {
                if (strcmp(stored_username, username) == 0) {
                    fclose(file);
                    return 0; // Username already exists
                }
                if (id > max_id) {
                    max_id = id;
                }
            }
        }
        fclose(file);
    }

    file = fopen(USER_FILE, "a");
    if (!file) {
        perror("fopen");
        return 0;
    }

    int new_id = max_id + 1;
    char *encrypted_password = bcrypt(password);
    fprintf(file, "%d,%s,%s,%s\n", new_id, username, encrypted_password, new_id == 1 ? "ROOT" : "USER");
    free(encrypted_password);
    fclose(file);
    return 1;
}
````
Penjelasan:
Membaca File Pengguna:

Membuka file USER_FILE untuk membaca.
Memeriksa apakah username sudah ada di file.
Menentukan max_id untuk ID pengguna baru.
Menambahkan Pengguna Baru:

Jika username belum ada, membuka file USER_FILE dalam mode append.
Mengenkripsi password menggunakan bcrypt.
Menulis informasi pengguna baru ke dalam file.
Menutup file dan mengembalikan nilai 1 yang menunjukkan registrasi berhasil.

### 15. Fungsi `login_user`
````
int login_user(const char *username, const char *password, Session *session, int is_monitor) {
    FILE *file = fopen(USER_FILE, "r");
    if (!file) {
        perror("fopen");
        return 0;
    }

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), file)) {
        int id;
        char stored_username[BUF_SIZE], stored_password[BUF_SIZE], role[BUF_SIZE];
        int num_fields = sscanf(line, "%d,%[^,],%[^,],%s", &id, stored_username, stored_password, role);

        if (num_fields < 4) {
            printf("Malformed line: %s\n", line);
            continue;
        }

        if (strcmp(stored_username, username) == 0) {
            if ((is_monitor && logged_in_users[id].monitor) || (!is_monitor && logged_in_users[id].regular)) {
                fclose(file);
                return -1; // User already logged in for this type of session
            }
            if (strcmp(crypt(password, stored_password), stored_password) == 0) {
                session->logged_in = 1;
                strncpy(session->username, username, BUF_SIZE);
                strncpy(session->role, role, BUF_SIZE);
                session->user_id = id;
                session->is_monitor = is_monitor;
                if (is_monitor) {
                    logged_in_users[id].monitor = 1;
                } else {
                    logged_in_users[id].regular = 1;
                }
                fclose(file);
                printf("User %s logged in %s\n", username, is_monitor ? "as monitor" : ""); // Debugging
                return 1;
            }
            break;
        }
    }

    fclose(file);
    return 0;
}
````
Penjelasan:
Membaca File Pengguna:

Membuka file USER_FILE untuk membaca.
Memeriksa apakah username ada di file.
Memeriksa Status Login dan Password:

Memeriksa apakah pengguna sudah login.
Membandingkan password yang dienkripsi.
Jika cocok, memperbarui sesi pengguna dengan informasi login.
Menutup file dan mengembalikan nilai 1 untuk menunjukkan login berhasil.

### 16. Fungsi `logout_user`
````
void logout_user(Session *session) {
    if (session->logged_in) {
        if (session->is_monitor) {
            logged_in_users[session->user_id].monitor = 0;
        } else {
            logged_in_users[session->user_id].regular = 0;
        }
        session->logged_in = 0;
        memset(session->username, 0, BUF_SIZE);
        memset(session->role, 0, BUF_SIZE);
        session->user_id = 0;
        session->is_monitor = 0;
    }
}
````
Penjelasan:
Logout Pengguna:
Memeriksa apakah sesi pengguna sedang login.
Mengubah status login dalam daftar logged_in_users berdasarkan tipe sesi (monitor atau regular).
Mengatur ulang informasi sesi pengguna menjadi nol atau kosong.

### 17. Fungsi `logout_monitor`
````
void logout_monitor(monitorSession *monitor_session) {
    if (monitor_session->logged_in) {
        monitor_session->logged_in = 0;
    }
}
````
Penjelasan:
Logout Monitor:
Memeriksa apakah sesi monitor sedang login.
Mengubah status login monitor menjadi tidak login (0).


### 18. Fungsi `login_monitor`
````
int login_monitor(const char *username, const char *password, monitorSession *session) {
    FILE *file = fopen(USER_FILE, "r");
    if (!file) {
        perror("fopen");
        return 0;
    }

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), file)) {
        int id;
        char stored_username[BUF_SIZE], stored_password[BUF_SIZE], role[BUF_SIZE];
        int num_fields = sscanf(line, "%d,%[^,],%[^,],%s", &id, stored_username, stored_password, role);

        if (num_fields < 4) {
            printf("Malformed line: %s\n", line);
            continue;
        }

        if (strcmp(stored_username, username) == 0) {
            if (strcmp(crypt(password, stored_password), stored_password) == 0) {
                session->logged_in = 1;
                fclose(file);
                printf("User %s logged in as monitor\n", username); // Debugging
                return 1;
            }
            break;
        }
    }

    fclose(file);
    return 0;
}
````
Fungsi ini bertujuan untuk memeriksa kredensial login pengguna dan mengatur sesi login jika pengguna berhasil masuk sebagai monitor.

Parameter:
- username: Nama pengguna yang mencoba login.
- password: Kata sandi yang diberikan oleh pengguna.
- session: Pointer ke struktur sesi yang digunakan untuk mencatat status login.

Proses:
- Membuka file pengguna (USER_FILE) dalam mode baca. Jika gagal, mencetak pesan kesalahan dan mengembalikan 0. -
- Membaca setiap baris dari file dan memparsingnya menjadi variabel yang sesuai (id, stored_username, stored_password, role).
- Jika baris tidak terformat dengan benar (kurang dari 4 bidang), mencetak pesan kesalahan dan melanjutkan ke baris berikutnya.
- Membandingkan stored_username dengan username yang diberikan.
- Jika cocok, membandingkan kata sandi yang di-hash menggunakan fungsi crypt dengan stored_password.
- Jika kata sandi cocok, mengatur status login dalam session dan menutup file, lalu mengembalikan 1.
- Jika tidak cocok, menutup file dan mengembalikan 0.

### 19. Fungsi `bool is_admin`
````
bool is_admin(int socket, const char *channel_name, const char *username) {
    char admin_dir_path[BUF_SIZE];
    snprintf(admin_dir_path, sizeof(admin_dir_path), "%s/%s/admin", DISCORIT_DIR, channel_name);

    char auth_dir_path[BUF_SIZE];
    snprintf(auth_dir_path, sizeof(auth_dir_path), "%s/auth.csv", admin_dir_path);

    FILE *auth_file = fopen(auth_dir_path, "r");
    if (!auth_file) {
        perror("fopen");
        return;
    }

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), auth_file)) {
        int id;
        char stored_username[BUF_SIZE], role[BUF_SIZE];
        sscanf(line, "%d,%[^,],%s", &id, stored_username, role);
        if (strcmp(stored_username, username) == 0 && strcmp(role, "ADMIN") == 0) {
            fclose(auth_file);
            return 1;
        }
    }

    fclose(auth_file);
    return 0;
}
````
Fungsi ini memeriksa apakah seorang pengguna adalah admin pada suatu channel tertentu.

Parameter:
socket: Soket yang digunakan untuk komunikasi (tidak digunakan dalam implementasi ini).
channel_name: Nama channel yang diperiksa.
username: Nama pengguna yang diperiksa.

Proses:
Mengatur jalur direktori admin berdasarkan DISCORIT_DIR dan channel_name.
Membuka file autentikasi (auth.csv) dalam direktori tersebut.
Membaca setiap baris dari file dan memparsingnya menjadi variabel yang sesuai (id, stored_username, role).
Jika stored_username cocok dengan username dan role adalah "ADMIN", menutup file dan mengembalikan true.
Jika tidak ditemukan, menutup file dan mengembalikan false.

### 20. Fungsi `bool is_root`    
````
bool is_root(int socket, const char *username) {
    FILE *file = fopen(USER_FILE, "r");
    if (!file) {
        perror("fopen");
        return false;
    }

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), file)) {
        int id;
        char stored_username[BUF_SIZE], role[BUF_SIZE], password[BUF_SIZE];
        if (sscanf(line, "%d,%[^,],%[^,],%s", &id, stored_username, password, role) == 4) {
            if (strcmp(stored_username, username) == 0 && strcmp(role, "ROOT") == 0) {
                fclose(file);
                return true;
            }
        }
    }

    fclose(file);
    return false;
}
````
Fungsi ini memeriksa apakah seorang pengguna adalah root (superuser) dalam sistem.

Parameter:
socket: Soket yang digunakan untuk komunikasi (tidak digunakan dalam implementasi ini).
username: Nama pengguna yang diperiksa.

Proses:
Membuka file pengguna (USER_FILE) dalam mode baca.
Membaca setiap baris dari file dan memparsingnya menjadi variabel yang sesuai (id, stored_username, password, role).
Jika stored_username cocok dengan username dan role adalah "ROOT", menutup file dan mengembalikan true.
Jika tidak ditemukan, menutup file dan mengembalikan false.

### 21. Fungsi `bool is_member`
````
bool is_member(int socket, const char *channel_name, const char *username) {
    char auth_dir_path[BUF_SIZE];
    snprintf(auth_dir_path, sizeof(auth_dir_path), "%s/%s/admin/auth.csv", DISCORIT_DIR, channel_name);

    FILE *auth_file = fopen(auth_dir_path, "r");
    if (!auth_file) {
        perror("fopen");
        return;
    }

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), auth_file)) {
        int id;
        char stored_username[BUF_SIZE], role[BUF_SIZE];
        sscanf(line, "%d,%[^,],%s", &id, stored_username, role);
        if (strcmp(stored_username, username) == 0) {
            fclose(auth_file);
            return 1;
        }
    }

    fclose(auth_file);
    return 0;
}
````
Fungsi ini memeriksa apakah seorang pengguna adalah anggota dari suatu channel tertentu.

Parameter:
socket: Soket yang digunakan untuk komunikasi (tidak digunakan dalam implementasi ini).
channel_name: Nama channel yang diperiksa.
username: Nama pengguna yang diperiksa.

Proses:
Mengatur jalur direktori autentikasi berdasarkan DISCORIT_DIR dan channel_name.
Membuka file autentikasi (auth.csv) dalam direktori tersebut.
Membaca setiap baris dari file dan memparsingnya menjadi variabel yang sesuai (id, stored_username, role).
Jika stored_username cocok dengan username, menutup file dan mengembalikan true.
Jika tidak ditemukan, menutup file dan mengembalikan false.

### 22. Fungsi `bool validate_key`
````
bool validate_key(int socket, const char *channel_name, const char *key) {
    FILE *file = fopen(CHANNEL_FILE, "r");
    if (!file) {
        perror("fopen");
        return false;  // Changed from 'return;' to 'return false;'
    }

    // Trim newline from key if present
    char trimmed_key[BUF_SIZE];
    strncpy(trimmed_key, key, BUF_SIZE - 1);
    trimmed_key[BUF_SIZE - 1] = '\0';  // Ensure null-termination
    char *newline = strchr(trimmed_key, '\n');
    if (newline) *newline = '\0';

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), file)) {
        char stored_channel[BUF_SIZE], stored_key[BUF_SIZE];
        sscanf(line, "%*d,%[^,],%s", stored_channel, stored_key);
        if (strcmp(stored_channel, channel_name) == 0 && strcmp(crypt(trimmed_key, stored_key), stored_key) == 0) {
            fclose(file);
            return true;
        }
    }

    fclose(file);
    return false;
}
````
Fungsi ini memeriksa apakah kunci yang diberikan valid untuk suatu channel.

Parameter:
socket: Soket yang digunakan untuk komunikasi (tidak digunakan dalam implementasi ini).
channel_name: Nama channel yang diperiksa.
key: Kunci yang diberikan untuk divalidasi.

Proses:
Membuka file channel (CHANNEL_FILE) dalam mode baca. Jika gagal, mencetak pesan kesalahan dan mengembalikan false.
Memangkas newline dari kunci jika ada.
Membaca setiap baris dari file dan memparsingnya menjadi variabel yang sesuai (stored_channel, stored_key).
Jika stored_channel cocok dengan channel_name dan kunci yang di-hash cocok dengan stored_key, menutup file dan mengembalikan true.
Jika tidak ditemukan, menutup file dan mengembalikan false.

### 23. Fungsi `bool channel_exists`
````
bool channel_exists(int socket, const char *channel_name) {
    FILE *file = fopen(CHANNEL_FILE, "r");
    if (!file) {
        perror("fopen");
        return;
    }

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), file)) {
        char stored_channel[BUF_SIZE];
        sscanf(line, "%*d,%[^,],%*s", stored_channel);
        if (strcmp(stored_channel, channel_name) == 0) {
            fclose(file);
            return 1;
        }
    }

    fclose(file);
    return 0;
}
````
Fungsi ini memeriksa apakah suatu channel ada.

Parameter:
socket: Soket yang digunakan untuk komunikasi (tidak digunakan dalam implementasi ini).
channel_name: Nama channel yang diperiksa.

Proses:
Membuka file channel (CHANNEL_FILE) dalam mode baca. Jika gagal, mencetak pesan kesalahan dan mengembalikan false.
Membaca setiap baris dari file dan memparsingnya menjadi variabel yang sesuai (stored_channel).
Jika stored_channel cocok dengan channel_name, menutup file dan mengembalikan true.
Jika tidak ditemukan, menutup file dan mengembalikan false.

### 24. Fungsi `bool room_exists`
````
bool room_exists(int socket, const char *channel_name, const char *room_name) {
    char room_dir_path[BUF_SIZE];
    snprintf(room_dir_path, sizeof(room_dir_path), "%s/%s/%s", DISCORIT_DIR, channel_name, room_name);

    struct stat st;
    if (stat(room_dir_path, &st) == 0) {
        return 1;
    }

    return 0;
}
````
Fungsi ini memeriksa apakah suatu room ada dalam channel tertentu.

Parameter:
socket: Soket yang digunakan untuk komunikasi (tidak digunakan dalam implementasi ini).
channel_name: Nama channel yang diperiksa.
room_name: Nama room yang diperiksa.

Proses:
Menyusun jalur direktori room berdasarkan DISCORIT_DIR, channel_name, dan room_name.
Memeriksa apakah jalur tersebut ada menggunakan fungsi stat.
Jika jalur ada, mengembalikan true, jika tidak, mengembalikan false.

### 25. Fungsi `bool is_banned`
````
bool is_banned(int socket, const char *channel_name, const char *username) {
    char auth_dir_path[BUF_SIZE];
    snprintf(auth_dir_path, sizeof(auth_dir_path), "%s/%s/admin/auth.csv", DISCORIT_DIR, channel_name);

    FILE *auth_file = fopen(auth_dir_path, "r");
    if (!auth_file) {
        perror("fopen");
        return;
    }

    char line[BUF_SIZE];
    while (fgets(line, sizeof(line), auth_file)) {
        int id;
        char stored_username[BUF_SIZE], role[BUF_SIZE];
        sscanf(line, "%d,%[^,],%s", &id, stored_username, role);
        if (strcmp(stored_username, username) == 0 && strcmp(role, "BANNED") == 0) {
            fclose(auth_file);
            return 1;
        }
    }

    fclose(auth_file);
    return 0;
}
````
Fungsi ini memeriksa apakah seorang pengguna dilarang (banned) dalam channel tertentu.

Parameter:
socket: Soket yang digunakan untuk komunikasi (tidak digunakan dalam implementasi ini).
channel_name: Nama channel yang diperiksa.
username: Nama pengguna yang diperiksa.

Proses:
Menyusun jalur direktori autentikasi berdasarkan DISCORIT_DIR dan channel_name.
Membuka file autentikasi (auth.csv) dalam direktori tersebut.
Membaca setiap baris dari file dan memparsingnya menjadi variabel yang sesuai (id, stored_username, role).
Jika stored_username cocok dengan username dan role adalah "BANNED", menutup file dan mengembalikan true.
Jika tidak ditemukan, menutup file dan mengembalikan false.
