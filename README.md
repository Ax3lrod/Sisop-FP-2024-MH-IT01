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
### Header dan Deklarasi
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

### Deskripsi Fungsi
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
