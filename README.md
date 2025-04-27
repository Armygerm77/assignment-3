server/
├── aesdsocket.c
├── Makefile
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <signal.h>
#include <errno.h>

#define PORT 9000
#define BACKLOG 5
#define BUFFER_SIZE 1024

int server_fd = -1;

void handle_exit() {
    if (server_fd != -1) {
        close(server_fd);
        printf("Server socket closed.\n");
    }
}

void sigint_handler(int signo) {
    printf("SIGINT received, shutting down server...\n");
    handle_exit();
    exit(0);
}

int main() {
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    int client_fd;
    char buffer[BUFFER_SIZE];

    // Register SIGINT handler
    signal(SIGINT, sigint_handler);

    // Create socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // Bind address to socket
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind");
        handle_exit();
        exit(EXIT_FAILURE);
    }

    // Start listening
    if (listen(server_fd, BACKLOG) == -1) {
        perror("listen");
        handle_exit();
        exit(EXIT_FAILURE);
    }

    printf("Server listening on port %d...\n", PORT);

    while (1) {
        client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_addr_len);
        if (client_fd == -1) {
            perror("accept");
            continue;
        }

        printf("Connection accepted from %s:%d\n",
               inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

        // Read data from client
        ssize_t bytes_received;
        while ((bytes_received = recv(client_fd, buffer, BUFFER_SIZE, 0)) > 0) {
            buffer[bytes_received] = '\0'; // Null-terminate the received data
            printf("Received: %s", buffer);
            send(client_fd, buffer, bytes_received, 0); // Echo back the data
        }

        if (bytes_received == -1) {
            perror("recv");
        }

        printf("Closing connection...\n");
        close(client_fd);
    }

    handle_exit();
    return 0;
}
sudo ./aesdsocket
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <signal.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/stat.h>

#define PORT 9000
#define BACKLOG 5
#define BUFFER_SIZE 1024
#define DATA_FILE "/var/tmp/aesdsocketdata"

int server_fd = -1;
int data_fd = -1;

void handle_exit() {
    if (server_fd != -1) {
        close(server_fd);
        printf("Server socket closed.\n");
    }

    if (data_fd != -1) {
        close(data_fd);
        printf("Data file closed.\n");
    }

    // Remove the data file
    unlink(DATA_FILE);
    printf("Data file removed.\n");
}

void sigint_handler(int signo) {
    printf("SIGINT received, shutting down server...\n");
    handle_exit();
    exit(0);
}

int main() {
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    int client_fd;
    char buffer[BUFFER_SIZE];

    // Register SIGINT handler
    signal(SIGINT, sigint_handler);

    // Create socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // Bind address to socket
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind");
        handle_exit();
        exit(EXIT_FAILURE);
    }

    // Start listening
    if (listen(server_fd, BACKLOG) == -1) {
        perror("listen");
        handle_exit();
        exit(EXIT_FAILURE);
    }

    printf("Server listening on port %d...\n", PORT);

    // Open the data file for appending (create if it doesn't exist)
    data_fd = open(DATA_FILE, O_CREAT | O_WRONLY | O_APPEND, 0644);
    if (data_fd == -1) {
        perror("open data file");
        handle_exit();
        exit(EXIT_FAILURE);
    }

    while (1) {
        client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_addr_len);
        if (client_fd == -1) {
            perror("accept");
            continue;
        }

        printf("Connection accepted from %s:%d\n",
               inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

        // Read data from client and append to the file
        ssize_t bytes_received;
        while ((bytes_received = recv(client_fd, buffer, BUFFER_SIZE, 0)) > 0) {
            // Write the received data to the file
            if (write(data_fd, buffer, bytes_received) == -1) {
                perror("write to data file");
                break;
            }
        }

        if (bytes_received == -1) {
            perror("recv");
        }

        printf("Closing connection...\n");
        close(client_fd);
    }

    handle_exit();
    return 0;
}sudo apt-get install netcat
sudo apt-get install netcat
nc --version
sockettest.sh
-d
ls -l full-test.sh
chmod +x full-test.sh
./full-test.sh
/**
 * @file aesd-circular-buffer.c
 * @brief Functions and data related to a circular buffer imlementation
 *
 * @author Dan Walkes
 * @date 2020-03-01
 * @copyright Copyright (c) 2020
 *
 */

#ifdef __KERNEL__
#include <linux/string.h>
#else
#include <string.h>
#endif

#include "aesd-circular-buffer.h"

/**
 * @param buffer the buffer to search for corresponding offset.  Any necessary locking must be performed by caller.
 * @param char_offset the position to search for in the buffer list, describing the zero referenced
 *      character index if all buffer strings were concatenated end to end
 * @param entry_offset_byte_rtn is a pointer specifying a location to store the byte of the returned aesd_buffer_entry
 *      buffptr member corresponding to char_offset.  This value is only set when a matching char_offset is found
 *      in aesd_buffer.
 * @return the struct aesd_buffer_entry structure representing the position described by char_offset, or
 * NULL if this position is not available in the buffer (not enough data is written).
 */
struct aesd_buffer_entry *aesd_circular_buffer_find_entry_offset_for_fpos(struct aesd_circular_buffer *buffer,
            size_t char_offset, size_t *entry_offset_byte_rtn )
{
    /**
    * TODO: implement per description
    */
    return NULL;
}

/**
* Adds entry @param add_entry to @param buffer in the location specified in buffer->in_offs.
* If the buffer was already full, overwrites the oldest entry and advances buffer->out_offs to the
* new start location.
* Any necessary locking must be handled by the caller
* Any memory referenced in @param add_entry must be allocated by and/or must have a lifetime managed by the caller.
*/
void aesd_circular_buffer_add_entry(struct aesd_circular_buffer *buffer, const struct aesd_buffer_entry *add_entry)
{
    /**
    * TODO: implement per description
    */
}

/**
* Initializes the circular buffer described by @param buffer to an empty struct
*/
void aesd_circular_buffer_init(struct aesd_circular_buffer *buffer)
{
    memset(buffer,0,sizeof(struct aesd_circular_buffer));
git fetch assignmentsibase
git fetch assignments-base
git remote add assignments-base git@github.com:cu-ecen-aeld/aesd-assignments.git
git merge assignments-base/assignment7
git submodule update --init --recursive
./full-test.sh
