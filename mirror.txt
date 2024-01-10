/*

Name: Muhammad Haseeb Ahmad
Course: Advance System's Programming
Course ID: COMP8527
Members: Individual
Section: 3
mirror.c


*/

#include <stdio.h>  //standard input output
#include <stdlib.h>    //standard Library
#include <string.h>   //handling strings
#include <unistd.h>   //POSIX
#include <arpa/inet.h>   //network when used on thwo systesm
#include <sys/types.h> //symbol and structure of typetdef
#include <sys/stat.h>   //data return strructures
#include <fcntl.h>   //posix
#include <dirent.h>   //directoruies 
#include <errno.h>  //error handling
#include <sys/wait.h> //wait


/*same as server.c minnor mirror fuctionalyies are handlers*/ 

#define PORT 4500
#define MAX_CLIENTS 10
#define MAX_COMMAND_SIZE 1024

typedef struct {
    int id;
    int socket;
    struct sockaddr_in addr;
} Client;

void create_client_folder() { //same as server.c
    char command[50];
    sprintf(command, "mkdir -p ~/f23project_mirror");
    system(command);
    printf("[*-->] Folder Name as created/checked: f23project_mirror\n");
}

void copy_file_to_mirror_folder(int client_id, char* filename) {  //same as server as c
    char command[100];
    sprintf(command, "cp ~/%s ~/f23project_mirror/f23project_mirror_client%d_%s", filename, client_id, filename);
    system(command);
    printf("[*] File sent to mirror %s to f23project_mirror/f23project_mirror_client%d_%s\n", filename, client_id, filename);
}

void send_archive(int client_id, char* archive_name) {
    int fd = open(archive_name, O_RDONLY);
    if (fd == -1) {
        perror("Error opening archive file");
        return;
    }

    struct stat file_stat;
    if (fstat(fd, &file_stat) == -1) {
        perror("Error getting file stats");
        close(fd);
        return;
    }

    send(client_id, "Archive", sizeof("Archive"), 0);

    send(client_id, &file_stat.st_size, sizeof(off_t), 0);

    char buffer[1024];
    ssize_t read_size, sent_size = 0;

    while ((read_size = read(fd, buffer, sizeof(buffer))) > 0) {
        ssize_t sent = send(client_id, buffer, read_size, 0);
        if (sent == -1) {
            perror("Error sending archive");
            break;
        }
        sent_size += sent;
    }

    printf("[*--->] Sent %zd bytes of archive to the client %d\n", sent_size, client_id);

    close(fd);
    remove(archive_name);
}

void create_archive_with_file_types(int client_id, const char* ext1, const char* ext2, const char* ext3) {
    char archive_name[50];
    sprintf(archive_name, "f23project_mirror_client%d_temp.tar.gz", client_id);

    char command[150];
    sprintf(command, "find ~/ -type f \\( -name '*.%s' -o -name '*.%s' -o -name '*.%s' \\) -exec tar czf %s {} + 2>/dev/null", ext1, ext2, ext3, archive_name);
    int status = system(command);

    if (WIFEXITED(status) && WEXITSTATUS(status) == 0) {
        printf("[*--->] Created archive %s with files of types %s, %s, %s\n", archive_name, ext1, ext2, ext3);
        send_archive(client_id, archive_name);
    } else {
        printf("[!] Error creating archive or no files found with specified extensions\n");
        send(client_id, "No file found", sizeof("No file found."), 0);
    }
}

void create_archive_with_size_range(int client_id, int size1, int size2) {
    char archive_name[50];
    sprintf(archive_name, "f23project_mirror_client%d_temp.tar.gz", client_id);

    char command[150];
    sprintf(command, "find ~/ -type f -size +%d -a -size -%d -exec tar czf %s {} + 2>/dev/null", size1, size2, archive_name);
    int status = system(command);

    if (WIFEXITED(status) && WEXITSTATUS(status) == 0) {
        printf("[*--->] Created archive %s with files in size range %d - %d\n", archive_name, size1, size2);
        send_archive(client_id, archive_name);
    } else {
        printf("[!] Error creating archive or no files found within the specified size range\n");
        send(client_id, "No file found", sizeof("No file found"), 0);
    }
}

void handle_client(Client client) {
    printf("[+] Mirror Client %d connected from %s:%d\n", client.id, inet_ntoa(client.addr.sin_addr), ntohs(client.addr.sin_port));

    create_client_folder();

    char welcome_message[] = "Welcome to the mirror! Type 'quitm' to exit. Use 'getfn filename' to copy a file to f23project_mirror folder. Use 'getfz size1 size2' to get files within the specified size range. Use 'getft ext1 ext2 ext3' to get files with specific extensions.\n";
    send(client.socket, welcome_message, sizeof(welcome_message), 0);

    while (1) {
        char buffer[MAX_COMMAND_SIZE];
        ssize_t recv_size = recv(client.socket, buffer, sizeof(buffer), 0);

        if (recv_size <= 0 || strncmp(buffer, "quitm", 5) == 0) {
            printf("[-] Mirror Client %d disconnected.\n", client.id);
            break;
        }

        printf("[*] Received from Mirror Client %d: %s", client.id, buffer);

        if (strncmp(buffer, "getfn", 5) == 0) {
            char filename[MAX_COMMAND_SIZE];
            sscanf(buffer, "getfn %s", filename);
            copy_file_to_mirror_folder(client.id, filename);
        } else if (strncmp(buffer, "getfz", 5) == 0) {
            int size1, size2;
            sscanf(buffer, "getfz %d %d", &size1, &size2);
            create_archive_with_size_range(client.id, size1, size2);
        } else if (strncmp(buffer, "getft", 5) == 0) {
            char ext1[MAX_COMMAND_SIZE], ext2[MAX_COMMAND_SIZE], ext3[MAX_COMMAND_SIZE];
            sscanf(buffer, "getft %s %s %s", ext1, ext2, ext3);
            create_archive_with_file_types(client.id, ext1, ext2, ext3);
        }
    }

    close(client.socket);
}

int main() {
    int server_socket, mirror_client_socket;
    struct sockaddr_in server_addr;
    socklen_t mirror_client_addr_len = sizeof(struct sockaddr_in);

    if ((server_socket = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("Error creating socket");
        exit(EXIT_FAILURE);
    }

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    if (bind(server_socket, (struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("Error binding socket");
        exit(EXIT_FAILURE);
    }

    if (listen(server_socket, MAX_CLIENTS) == -1) {
        perror("Error listening for connections");
        exit(EXIT_FAILURE);
    }

    printf("[*--->] Listening for Mirror Clients through port number:%d\n", PORT);

    int mirror_client_id = 1;

    while (1) {
        if ((mirror_client_socket = accept(server_socket, (struct sockaddr*)&server_addr, &mirror_client_addr_len)) == -1) {
            perror("Error accepting connection");
            continue;
        }

        Client new_mirror_client;
        new_mirror_client.id = mirror_client_id++;
        new_mirror_client.socket = mirror_client_socket;
        new_mirror_client.addr = server_addr;

        if (fork() == 0) {
            close(server_socket);
            handle_client(new_mirror_client);
            exit(EXIT_SUCCESS);
        } else {
            close(mirror_client_socket);
        }
    }

    return 0;
}
