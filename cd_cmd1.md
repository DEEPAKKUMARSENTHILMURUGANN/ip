# Networking Lab — Exercise Guide

---

## Exercise 1: Sliding Window Protocol

### Step-by-step Procedure

1. Create a folder: `mkdir sliding_window && cd sliding_window`
2. Create the server file: `nano server.c` → paste the server code below → save (`Ctrl+O`, `Enter`, `Ctrl+X`)
3. Create the client file: `nano client.c` → paste the client code below → save
4. Compile both files:

```
gcc server.c -o server
gcc client.c -o client
```
5. Open a second terminal (same folder).
6. Terminal 1: `./server`
7. Terminal 2: `./client`
8. Enter case number (1–4), number of frames, window size, and frame numbers when prompted.
9. Observe the output in both terminals.
10. Repeat steps 7–9 for each case (1: success, 2: frame loss, 3: ACK loss, 4: out of order).

### server.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define MAX 100

int main() {
    int server_fd, client_fd;
    struct sockaddr_in server, client;
    socklen_t len = sizeof(client);

    char buffer[MAX];
    int frame, expected = 1;
    int case_no;

    server_fd = socket(AF_INET, SOCK_STREAM, 0);

    server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr *)&server, sizeof(server));
    listen(server_fd, 5);

    printf("Server waiting...\n");

    client_fd = accept(server_fd, (struct sockaddr *)&client, &len);

    recv(client_fd, &case_no, sizeof(case_no), 0);

    while (1)
    {
        memset(buffer, 0, MAX);

        if (recv(client_fd, buffer, MAX, 0) <= 0)
            break;

        frame = atoi(buffer);

        if (frame == -1)
            break;

        printf("Received Frame: %d\n", frame);

        if (case_no == 2 && frame == 2)
        {
            printf("Frame 2 LOST\n");
            case_no = 0;
            continue;
        }

        if (case_no == 4)
        {
            if (frame != expected)
            {
                printf("Out of Order Frame: %d\n", frame);
                printf("Expected Frame: %d\n", expected);

                send(client_fd, "NACK", 4, 0);
                continue;
            }
        }

        if (case_no == 3 && frame == 2)
        {
            printf("ACK 2 LOST\n");
            case_no = 0;
            expected++;
            continue;
        }

        char ack[20];
        sprintf(ack, "ACK %d", frame);

        send(client_fd, ack, strlen(ack) + 1, 0);

        printf("ACK %d sent\n", frame);

        expected++;
    }

    close(client_fd);
    close(server_fd);

    return 0;
}
```

### client.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080
#define MAX 100

int main() {
    int sock;
    struct sockaddr_in server;

    int n, window, case_no;
    int frames[MAX];

    char buffer[MAX];

    sock = socket(AF_INET, SOCK_STREAM, 0);

    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    server.sin_addr.s_addr = inet_addr("127.0.0.1");

    connect(sock, (struct sockaddr *)&server, sizeof(server));

    printf("Sliding Window Protocol\n");

    printf("Enter case number:\n");
    printf("1. Successful transmission\n");
    printf("2. Frame loss\n");
    printf("3. ACK loss\n");
    printf("4. Out of order frame\n");
    printf("Enter case: ");
    scanf("%d", &case_no);

    printf("Enter number of frames: ");
    scanf("%d", &n);

    printf("Enter window size: ");
    scanf("%d", &window);

    printf("Enter frame numbers:\n");

    for (int i = 0; i < n; i++)
    {
        printf("Enter frame %d: ", i + 1);
        scanf("%d", &frames[i]);
    }

    send(sock, &case_no, sizeof(case_no), 0);

    if (case_no == 4 && n >= 2)
    {
        int temp = frames[0];
        frames[0] = frames[1];
        frames[1] = temp;
    }

    int base = 0;

    while (base < n)
    {
        int end = base + window;

        if (end > n)
            end = n;

        printf("\nCurrent Window: ");

        for (int i = base; i < end; i++)
            printf("%d ", frames[i]);

        printf("\n");

        for (int i = base; i < end; i++)
        {
            printf("Sending Frame: %d\n", frames[i]);

            sprintf(buffer, "%d", frames[i]);

            send(sock, buffer, strlen(buffer) + 1, 0);
        }

        int ack_count = 0;

        while (ack_count < end - base)
        {
            memset(buffer, 0, MAX);

            recv(sock, buffer, MAX, 0);

            if (strcmp(buffer, "NACK") == 0)
            {
                printf("NACK received\n");
                printf("Retransmitting frames in correct order\n");

                for (int i = base; i < end; i++)
                {
                    sprintf(buffer, "%d", frames[i]);

                    printf("Retransmitting Frame: %d\n", frames[i]);

                    send(sock, buffer, strlen(buffer) + 1, 0);

                    memset(buffer, 0, MAX);

                    recv(sock, buffer, MAX, 0);

                    printf("%s received\n", buffer);
                }

                ack_count = end - base;
                break;
            }

            printf("%s received\n", buffer);

            ack_count++;
        }

        printf("Window moved forward\n");

        base = end;
    }

    printf("\nAll frames transmitted successfully.\n");

    strcpy(buffer, "-1");
    send(sock, buffer, strlen(buffer) + 1, 0);

    close(sock);

    return 0;
}
```

---

## Exercise 2: Stop-and-Wait Protocol

### Step-by-step Procedure

1. Create a folder: `mkdir stop_wait && cd stop_wait`
2. `nano server.c` → paste server code → save
3. `nano client.c` → paste client code → save
4. Compile:

```
gcc server.c -o server
gcc client.c -o client
```
5. Terminal 1: `./server`
6. Terminal 2: `./client`
7. Enter choice (1–4) and frame data when prompted.
8. Observe ACK/timeout/retransmission behavior in both terminals.
9. Repeat for all 4 cases (ACK received, ACK not received/timeout, frame not sent, duplicate frame).

### server.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080

struct Frame {
    int seq;
    int data;
    int choice;
};

int main() {
    int server_fd, client_fd;
    struct sockaddr_in server, client;
    socklen_t len = sizeof(client);

    struct Frame frame;
    int expected = 1;

    server_fd = socket(AF_INET, SOCK_STREAM, 0);

    server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr *)&server, sizeof(server));
    listen(server_fd, 5);

    printf("Server waiting...\n");

    client_fd = accept(server_fd, (struct sockaddr *)&client, &len);

    while (1)
    {
        int n = recv(client_fd, &frame, sizeof(frame), 0);

        if (n <= 0)
            break;

        printf("\nFrame received\n");
        printf("Sequence Number : %d\n", frame.seq);
        printf("Case            : %d\n", frame.choice);
        printf("Data            : %d\n", frame.data);

        if (frame.choice == 1)
        {
            printf("CASE 1: Frame received successfully.\n");

            send(client_fd, &frame.seq, sizeof(int), 0);

            printf("ACK %d sent.\n", frame.seq);
        }

        else if (frame.choice == 2)
        {
            if (frame.seq == 1)
            {
                printf("CASE 2: Frame received, ACK NOT sent.\n");
                printf("Client should timeout and retransmit.\n");

                expected = 2;
            }
            else
            {
                printf("CASE 2: Retransmitted frame received.\n");

                send(client_fd, &frame.seq, sizeof(int), 0);

                printf("ACK %d sent after retransmission.\n", frame.seq);
            }
        }

        else if (frame.choice == 3)
        {
            printf("CASE 3: No frame should be received.\n");
        }

        else if (frame.choice == 4)
        {
            if (frame.seq == expected)
            {
                printf("CASE 4: Frame received successfully.\n");

                send(client_fd, &frame.seq, sizeof(int), 0);

                printf("ACK %d sent.\n", frame.seq);

                expected++;
            }
            else if (frame.seq < expected)
            {
                printf("CASE 4: DUPLICATE FRAME detected!\n");
                printf("Expected sequence number: %d\n", expected);
                printf("Received sequence number: %d\n", frame.seq);

                send(client_fd, &frame.seq, sizeof(int), 0);

                printf("ACK %d sent again for duplicate frame.\n", frame.seq);
            }
        }
    }

    close(client_fd);
    close(server_fd);

    return 0;
}
```

### client.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8080

struct Frame {
    int seq;
    int data;
    int choice;
};

int main() {
    int sock;
    struct sockaddr_in server;

    struct Frame frame;

    int choice;
    int data;
    int ack;

    struct timeval timeout;

    sock = socket(AF_INET, SOCK_STREAM, 0);

    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);
    server.sin_addr.s_addr = inet_addr("127.0.0.1");

    connect(sock, (struct sockaddr *)&server, sizeof(server));

    printf("Stop and Wait Protocol\n\n");

    printf("Enter choice: ");
    scanf("%d", &choice);

    if (choice == 1)
    {
        printf("Enter frame data: ");
        scanf("%d", &data);

        frame.seq = 0;
        frame.data = data;
        frame.choice = 1;

        printf("\nSending Frame 0...\n");

        send(sock, &frame, sizeof(frame), 0);

        recv(sock, &ack, sizeof(int), 0);

        printf("ACK %d received.\n", ack);

        printf("CASE 1 successful.\n");
    }

    else if (choice == 2)
    {
        printf("Enter frame data: ");
        scanf("%d", &data);

        frame.seq = 1;
        frame.data = data;
        frame.choice = 2;

        printf("\nSending Frame 1...\n");

        send(sock, &frame, sizeof(frame), 0);

        timeout.tv_sec = 3;
        timeout.tv_usec = 0;

        setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &timeout, sizeof(timeout));

        int result = recv(sock, &ack, sizeof(int), 0);

        if (result <= 0)
        {
            printf("\nCASE 2: ACK NOT received.\n");
            printf("Timeout occurred after 3 seconds.\n");

            printf("Retransmitting frame 1...\n");

            send(sock, &frame, sizeof(frame), 0);

            timeout.tv_sec = 5;
            timeout.tv_usec = 0;

            setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &timeout, sizeof(timeout));

            recv(sock, &ack, sizeof(int), 0);

            printf("ACK %d received after retransmission.\n", ack);
        }
        else
        {
            printf("ACK %d received.\n", ack);
        }
    }

    else if (choice == 3)
    {
        printf("\nCASE 3: Frame is NOT sent.\n");
        printf("No frame was transmitted to the server.\n");
    }

    else if (choice == 4)
    {
        printf("Enter frame data: ");
        scanf("%d", &data);

        frame.seq = 1;
        frame.data = data;
        frame.choice = 4;

        printf("\nCASE 4: DUPLICATION\n");
        printf("Sending original frame...\n");

        send(sock, &frame, sizeof(frame), 0);

        recv(sock, &ack, sizeof(int), 0);

        printf("ACK %d received.\n", ack);

        printf("Sending DUPLICATE frame with sequence 1...\n");

        send(sock, &frame, sizeof(frame), 0);

        recv(sock, &ack, sizeof(int), 0);

        printf("ACK %d received for duplicate.\n", ack);
    }

    else
    {
        printf("Invalid choice.\n");
    }

    close(sock);

    return 0;
}
```

---

## Exercise 3: Basic Network Commands (No Coding)

### Step-by-step Procedure

1. Open terminal.
2. Run each command below, one at a time.
3. Note or screenshot the output for your lab record.

### Commands

```
ifconfig
ping 8.8.8.8
hostname
traceroute google.com     # Linux equivalent of tracert
nmap localhost
route -n                  # or: netstat -r
nslookup google.com
netstat -tulnp
wget https://example.com/file.txt
```

> Note: on newer Ubuntu, if `ifconfig`/`route`/`netstat` aren't found, install them with:
> `sudo apt install net-tools`

---

## Exercise 4: UDP Echo Chat Program

### Step-by-step Procedure

1. Create folder: `mkdir udp_echo && cd udp_echo`
2. `nano server3.c` → paste server code → save
3. `nano client.c` → paste client code → save
4. Compile:

```
gcc server3.c -o server3
gcc client.c -o udpclient
```
5. Terminal 1: `./server3` (starts and waits for messages)
6. Terminal 2: `./udpclient`
7. Enter server IP (use `127.0.0.1` if testing on the same machine).
8. Enter a message and press Enter.
9. See the echoed message printed back on the client.
10. (Optional) Open more terminals and run `./udpclient` again to simulate multiple clients.

### server3.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 5432
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    char buffer[BUFFER_SIZE];
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    int n;

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    if (sockfd < 0)
    {
        perror("Socket creation failed");
        exit(1);
    }

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    if (bind(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0)
    {
        perror("Bind failed");
        close(sockfd);
        exit(1);
    }

    printf("UDP Echo Server is running...\n");

    while (1)
    {
        n = recvfrom(sockfd, buffer, BUFFER_SIZE - 1, 0,
                     (struct sockaddr *)&client_addr, &client_len);

        if (n < 0)
        {
            perror("Receive failed");
            continue;
        }

        buffer[n] = '\0';

        printf("Client: %s\n", buffer);

        sendto(sockfd, buffer, n, 0,
               (struct sockaddr *)&client_addr, client_len);
    }

    close(sockfd);

    return 0;
}
```

### client.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 5432
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    char server_ip[50];
    char message[BUFFER_SIZE];
    char buffer[BUFFER_SIZE];
    struct sockaddr_in server_addr;
    socklen_t server_len = sizeof(server_addr);
    int n;

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    if (sockfd < 0)
    {
        perror("Socket creation failed");
        exit(1);
    }

    printf("Enter Server IP: ");
    scanf("%s", server_ip);
    getchar();

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);

    if (inet_pton(AF_INET, server_ip, &server_addr.sin_addr) <= 0)
    {
        printf("Invalid server IP address\n");
        close(sockfd);
        exit(1);
    }

    printf("Enter Message: ");
    fgets(message, BUFFER_SIZE, stdin);
    message[strcspn(message, "\n")] = '\0';

    sendto(sockfd, message, strlen(message), 0,
           (struct sockaddr *)&server_addr, server_len);

    n = recvfrom(sockfd, buffer, BUFFER_SIZE - 1, 0,
                 (struct sockaddr *)&server_addr, &server_len);

    if (n < 0)
    {
        perror("Receive failed");
        close(sockfd);
        exit(1);
    }

    buffer[n] = '\0';

    printf("Echo from Server: %s\n", buffer);

    close(sockfd);

    return 0;
}
```

---

## Exercise 5: DNS Implementation Using UDP Sockets

### Part A: Forward DNS Resolver (Domain → IP)

#### Step-by-step Procedure

1. Create folder: `mkdir dns_forward && cd dns_forward`
2. `nano server.c` → paste server code → save
3. `nano client.c` → paste client code → save
4. Compile:

```
gcc server.c -o dns_server
gcc client.c -o dns_client
```
5. Terminal 1: `./dns_server`
6. Terminal 2: `./dns_client`
7. Enter a domain name (e.g. `google.com`) at the client prompt.
8. Observe the resolved IP printed on the client, and the query log on the server.
9. Try a domain not in the list (e.g. `test.com`) to see "Domain not found".

#### server.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8080
#define BUFFER_SIZE 1024
#define NUM_DOMAINS 12

int main() {
    int server_sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t addr_len = sizeof(client_addr);
    char domain[BUFFER_SIZE];
    char response[BUFFER_SIZE];

    const char *domains[NUM_DOMAINS] = {
        "example.com", "google.com", "openai.com", "github.com",
        "yahoo.com", "amazon.com", "facebook.com", "apple.com",
        "microsoft.com", "netflix.com", "nasa.gov", "wikipedia.org"
    };
    const char *ips[NUM_DOMAINS] = {
        "93.184.216.34", "142.250.190.14", "104.18.12.123",
        "140.82.113.3", "98.137.11.163", "176.32.103.205",
        "157.240.20.35", "17.172.224.47", "40.113.200.201",
        "52.26.14.20", "129.164.179.22", "208.80.154.224"
    };

    server_sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (server_sock < 0) {
        perror("Socket creation failed");
        return 1;
    }

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    server_addr.sin_port = htons(PORT);

    if (bind(server_sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("Bind failed");
        close(server_sock);
        return 1;
    }

    printf("[DNS Server] Waiting for domain query...\n");

    recvfrom(server_sock, domain, BUFFER_SIZE, 0,
             (struct sockaddr *)&client_addr, &addr_len);

    printf("[DNS Server] Query received for domain: %s\n", domain);

    int found = 0;
    for (int i = 0; i < NUM_DOMAINS; i++) {
        if (strcmp(domain, domains[i]) == 0) {
            strcpy(response, ips[i]);
            found = 1;
            break;
        }
    }

    if (!found) {
        strcpy(response, "Domain not found");
    }

    sendto(server_sock, response, strlen(response) + 1, 0,
           (struct sockaddr *)&client_addr, addr_len);

    printf("[DNS Server] Sent IP: %s\n", response);

    close(server_sock);
    return 0;
}
```

#### client.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main() {
    int client_sock;
    struct sockaddr_in server_addr;
    char domain[BUFFER_SIZE];
    char response[BUFFER_SIZE];

    client_sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (client_sock < 0) {
        perror("Client socket creation failed");
        return 1;
    }

    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    server_addr.sin_port = htons(PORT);

    printf("Enter domain name: ");
    scanf("%s", domain);

    sendto(client_sock, domain, strlen(domain) + 1, 0,
           (struct sockaddr *)&server_addr, sizeof(server_addr));

    printf("[DNS Client] Query sent for: %s\n", domain);

    recvfrom(client_sock, response, BUFFER_SIZE, 0, NULL, NULL);

    printf("[DNS Client] IP received: %s\n", response);

    close(client_sock);
    return 0;
}
```

### Part B: Reverse DNS Resolver (IP → Domain)

#### Step-by-step Procedure

1. Create folder: `mkdir dns_reverse && cd dns_reverse`
2. `nano reverse_dns.c` → paste the code below → save
3. Compile: `gcc reverse_dns.c -o reverse_dns`
4. Run: `./reverse_dns` (this single program forks into both client and server — only one terminal needed)
5. Enter an IP address from the list (e.g. `142.250.190.14`) when prompted.
6. Observe the resolved domain name printed by the client side, and the query log from the server side.
7. Try an IP not in the list to see "IP not found".

#### reverse_dns.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <arpa/inet.h>

#define PORT 5354
#define BUFFER_SIZE 1024
#define NUM_RECORDS 12

struct DNSRecord {
    char ip[20];
    char domain[50];
};

int main() {
    char input_ip[BUFFER_SIZE];

    struct DNSRecord records[NUM_RECORDS] = {
        {"93.184.216.34", "example.com"},
        {"142.250.190.14", "google.com"},
        {"104.18.12.123", "openai.com"},
        {"140.82.113.3", "github.com"},
        {"98.137.11.163", "yahoo.com"},
        {"176.32.103.205", "amazon.com"},
        {"157.240.20.35", "facebook.com"},
        {"17.172.224.47", "apple.com"},
        {"40.113.200.201", "microsoft.com"},
        {"52.26.14.20", "netflix.com"},
        {"129.164.179.22", "nasa.gov"},
        {"208.80.154.224", "wikipedia.org"}
    };
    int n = NUM_RECORDS;

    printf("Enter IP address to resolve: ");
    scanf("%s", input_ip);

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return 1;
    }

    if (pid == 0) {
        /* Child = Client */
        int client_sock;
        struct sockaddr_in server_addr;
        char response[BUFFER_SIZE];

        sleep(1); /* give parent (server) time to bind */

        client_sock = socket(AF_INET, SOCK_DGRAM, 0);
        if (client_sock < 0)
            exit(1);

        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sin_family = AF_INET;
        server_addr.sin_port = htons(PORT);
        inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

        sendto(client_sock, input_ip, strlen(input_ip) + 1, 0,
               (struct sockaddr *)&server_addr, sizeof(server_addr));

        printf("[Reverse DNS Client] Query sent for IP: %s\n", input_ip);
        fflush(stdout);

        recvfrom(client_sock, response, BUFFER_SIZE, 0, NULL, NULL);

        printf("[Reverse DNS Client] Domain received: %s\n", response);
        fflush(stdout);

        close(client_sock);
        exit(0);
    }
    else {
        /* Parent = Server */
        int server_sock;
        struct sockaddr_in server_addr, client_addr;
        socklen_t client_len = sizeof(client_addr);
        char query_ip[BUFFER_SIZE];
        char response[BUFFER_SIZE];

        server_sock = socket(AF_INET, SOCK_DGRAM, 0);
        if (server_sock < 0)
            return 1;

        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sin_family = AF_INET;
        server_addr.sin_port = htons(PORT);
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

        if (bind(server_sock, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
            close(server_sock);
            return 1;
        }

        printf("[Reverse DNS Server] Waiting for IP query...\n");
        fflush(stdout);

        recvfrom(server_sock, query_ip, BUFFER_SIZE, 0,
                 (struct sockaddr *)&client_addr, &client_len);

        printf("[Reverse DNS Server] Query received for IP: %s\n", query_ip);
        fflush(stdout);

        int found = 0;
        for (int i = 0; i < n; i++) {
            if (strcmp(query_ip, records[i].ip) == 0) {
                strcpy(response, records[i].domain);
                found = 1;
                break;
            }
        }

        if (!found) {
            strcpy(response, "IP not found");
        }

        printf("[Reverse DNS Server] Sent domain: %s\n", response);
        fflush(stdout);

        sendto(server_sock, response, strlen(response) + 1, 0,
               (struct sockaddr *)&client_addr, client_len);

        close(server_sock);
        wait(NULL);
    }

    return 0;
}
```

---

## Exercise 6: TCP Socket Programming

Each part below is a **single self-contained program** that uses `fork()` to run the server as the child process and the client as the parent process (or vice versa), all inside one executable. You only need **one terminal** per part.

### Part 1: TCP Echo Chat System

#### Step-by-step Procedure

1. Create folder: `mkdir tcp_echo_chat && cd tcp_echo_chat`
2. `nano echo_chat.c` → paste the code below → save
3. Compile: `gcc echo_chat.c -o echo_chat`
4. Run: `./echo_chat`
5. Type messages one per line; type `bye` as the last line to end input.
6. Press `Ctrl+D` (EOF) if you want to stop entering lines early.
7. Observe each message echoed back by the server, ending with "bye".

#### echo_chat.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/wait.h>

#define PORT 9090
#define SIZE 1024

int send_all(int sock, const void *buf, int len) {
    int total = 0;
    while (total < len) {
        int n = send(sock, (char *)buf + total, len - total, 0);
        if (n <= 0)
            return -1;
        total += n;
    }
    return total;
}

int recv_all(int sock, void *buf, int len) {
    int total = 0;
    while (total < len) {
        int n = recv(sock, (char *)buf + total, len - total, MSG_WAITALL);
        if (n <= 0)
            return -1;
        total += n;
    }
    return total;
}

int main() {
    char messages[100][SIZE];
    int count = 0;

    printf("Enter messages (type 'bye' to stop):\n");
    while (count < 100 && fgets(messages[count], SIZE, stdin)) {
        messages[count][strcspn(messages[count], "\r\n")] = '\0';
        count++;
        if (strcmp(messages[count - 1], "bye") == 0)
            break;
    }

    int ready_pipe[2];
    pipe(ready_pipe);

    pid_t pid = fork();
    if (pid < 0)
        return 1;

    if (pid == 0) {
        /* Child = Client */
        int client_socket;
        struct sockaddr_in server_addr;
        char ch;

        close(ready_pipe[1]);

        client_socket = socket(AF_INET, SOCK_STREAM, 0);
        if (client_socket < 0)
            exit(1);

        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sin_family = AF_INET;
        server_addr.sin_port = htons(PORT);
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

        read(ready_pipe[0], &ch, 1);

        if (connect(client_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0)
            exit(1);

        printf("[EchoClient] Connected to server.\n");
        fflush(stdout);

        for (int i = 0; i < count; i++) {
            int len = strlen(messages[i]);

            if (send_all(client_socket, &len, sizeof(int)) < 0)
                break;
            if (send_all(client_socket, messages[i], len) < 0)
                break;

            int echo_len;
            if (recv_all(client_socket, &echo_len, sizeof(int)) < 0)
                break;

            char echo[SIZE];
            memset(echo, 0, SIZE);
            if (recv_all(client_socket, echo, echo_len) < 0)
                break;
            echo[echo_len] = '\0';

            if (strcmp(echo, "bye") != 0) {
                printf("[EchoClient] Echoed by server: %s\n\n", echo);
                fflush(stdout);
            } else {
                printf("[EchoClient] Echoed by server: bye\n\n");
                printf("[EchoClient] Chat ended.\n");
                fflush(stdout);
            }
        }

        close(client_socket);
        close(ready_pipe[0]);
        exit(0);
    }
    else {
        /* Parent = Server */
        int server_socket, client_socket;
        struct sockaddr_in server_addr, client_addr;
        socklen_t client_len = sizeof(client_addr);
        char received[100][SIZE];
        int received_count = 0;

        close(ready_pipe[0]);

        server_socket = socket(AF_INET, SOCK_STREAM, 0);
        if (server_socket < 0)
            return 1;

        int opt = 1;
        setsockopt(server_socket, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sin_family = AF_INET;
        server_addr.sin_port = htons(PORT);
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

        if (bind(server_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0)
            return 1;

        if (listen(server_socket, 1) < 0)
            return 1;

        printf("[EchoServer] Waiting for client...\n");
        fflush(stdout);

        write(ready_pipe[1], "R", 1);

        client_socket = accept(server_socket, (struct sockaddr *)&client_addr, &client_len);
        if (client_socket < 0)
            return 1;

        printf("[EchoServer] Client connected.\n");
        fflush(stdout);

        while (received_count < 100) {
            int len;
            if (recv_all(client_socket, &len, sizeof(int)) < 0)
                break;
            if (len <= 0 || len >= SIZE)
                break;
            if (recv_all(client_socket, received[received_count], len) < 0)
                break;
            received[received_count][len] = '\0';

            send_all(client_socket, &len, sizeof(int));
            send_all(client_socket, received[received_count], len);

            printf("[EchoServer] Message received: %s\n", received[received_count]);
            fflush(stdout);

            received_count++;

            if (strcmp(received[received_count - 1], "bye") == 0)
                break;
        }

        close(client_socket);
        close(server_socket);
        close(ready_pipe[1]);
        wait(NULL);
    }

    return 0;
}
```

### Part 2: Order Confirmation Chat

#### Step-by-step Procedure

1. Create folder: `mkdir tcp_order && cd tcp_order`
2. `nano order_chat.c` → paste the code below → save
3. Compile: `gcc order_chat.c -o order_chat`
4. Run: `./order_chat`
5. Enter your name, then order ID, then `yes` or `no` when asked to cancel.
6. Observe the greeting, order confirmation, and cancellation status printed.

#### order_chat.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/wait.h>

#define PORT 9090
#define SIZE 1024

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        /* Child = Server */
        int server_fd, client_fd;
        struct sockaddr_in server_addr, client_addr;
        socklen_t len = sizeof(client_addr);
        char buffer[SIZE], reply[SIZE], orderID[SIZE];

        server_fd = socket(AF_INET, SOCK_STREAM, 0);

        int opt = 1;
        setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
        server_addr.sin_port = htons(PORT);

        bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
        listen(server_fd, 1);

        printf("[Server] Waiting for client connection...\n");

        client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &len);
        printf("[Server] Client connected.\n");

        recv(client_fd, buffer, SIZE, 0);
        printf("[Server] Name received: %s\n", buffer);
        sprintf(reply, "Hello, %s", buffer);
        send(client_fd, reply, strlen(reply) + 1, 0);

        recv(client_fd, buffer, SIZE, 0);
        printf("[Server] Order ID received: %s\n", buffer);
        strcpy(orderID, buffer);
        sprintf(reply, "Your order ID is: %s", buffer);
        send(client_fd, reply, strlen(reply) + 1, 0);

        recv(client_fd, buffer, SIZE, 0);
        printf("[Server] Client response: %s\n", buffer);

        if (strcmp(buffer, "yes") == 0)
            sprintf(reply, "%s Your order has been canceled.", orderID);
        else
            sprintf(reply, "%s Your order remains active.", orderID);

        send(client_fd, reply, strlen(reply) + 1, 0);

        close(client_fd);
        close(server_fd);
        exit(0);
    }
    else {
        /* Parent = Client */
        int sockfd;
        struct sockaddr_in server_addr;
        char buffer[SIZE], name[SIZE], orderID[SIZE], choice[SIZE];

        sockfd = socket(AF_INET, SOCK_STREAM, 0);
        server_addr.sin_family = AF_INET;
        server_addr.sin_port = htons(PORT);
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

        while (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0);

        printf("[Client] Connected to server.\n");

        printf("Enter your name: ");
        fgets(name, SIZE, stdin);
        name[strcspn(name, "\n")] = '\0';

        printf("Enter order ID: ");
        fgets(orderID, SIZE, stdin);
        orderID[strcspn(orderID, "\n")] = '\0';

        printf("Cancel order? (yes/no): ");
        fgets(choice, SIZE, stdin);
        choice[strcspn(choice, "\n")] = '\0';

        send(sockfd, name, strlen(name) + 1, 0);
        recv(sockfd, buffer, SIZE, 0);
        printf("%s\n", buffer);

        send(sockfd, orderID, strlen(orderID) + 1, 0);
        recv(sockfd, buffer, SIZE, 0);
        printf("%s\n", buffer);

        send(sockfd, choice, strlen(choice) + 1, 0);
        recv(sockfd, buffer, SIZE, 0);
        printf("%s\n", buffer);

        close(sockfd);
        wait(NULL);
    }

    return 0;
}
```

### Part 3: Server Sends Headlines to Client

#### Step-by-step Procedure

1. Create folder: `mkdir tcp_headlines && cd tcp_headlines`
2. `nano headlines.c` → paste the code below → save
3. Compile: `gcc headlines.c -o headlines`
4. Run: `./headlines`
5. Enter 3 headlines, one per line, when prompted.
6. Observe the client printing each as `Headline 1: ...`, `Headline 2: ...`, `Headline 3: ...`.

#### headlines.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/wait.h>

#define PORT 9090
#define SIZE 1024

int main() {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        /* Child = Server */
        int server_fd, client_fd;
        struct sockaddr_in server_addr, client_addr;
        socklen_t len = sizeof(client_addr);
        char buffer[SIZE];

        server_fd = socket(AF_INET, SOCK_STREAM, 0);

        int opt = 1;
        setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
        server_addr.sin_port = htons(PORT);

        bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
        listen(server_fd, 1);

        client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &len);

        printf("Enter 3 headlines:\n");
        for (int i = 0; i < 3; i++) {
            memset(buffer, 0, SIZE);
            fgets(buffer, SIZE, stdin);
            buffer[strcspn(buffer, "\n")] = '\0';
            send(client_fd, buffer, SIZE, 0);
        }

        close(client_fd);
        close(server_fd);
        exit(0);
    }
    else {
        /* Parent = Client */
        int sockfd;
        struct sockaddr_in server_addr;
        char buffer[SIZE];

        sockfd = socket(AF_INET, SOCK_STREAM, 0);
        server_addr.sin_family = AF_INET;
        server_addr.sin_port = htons(PORT);
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

        while (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0);

        for (int i = 1; i <= 3; i++) {
            memset(buffer, 0, SIZE);
            int total = 0;
            while (total < SIZE) {
                int n = recv(sockfd, buffer + total, SIZE - total, 0);
                if (n <= 0)
                    break;
                total += n;
            }
            printf("Headline %d: %s\n", i, buffer);
        }

        close(sockfd);
        wait(NULL);
    }

    return 0;
}
```

### Part 4: Client Sends Sensor Data to Server

#### Step-by-step Procedure

1. Create folder: `mkdir tcp_sensor && cd tcp_sensor`
2. `nano sensor.c` → paste the code below → save
3. Compile: `gcc sensor.c -o sensor`
4. Run: `./sensor`
5. Enter sensor data at the prompt (e.g. `Temp:25.6,Humidity:60`).
6. Observe the server printing the received sensor data.

#### sensor.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/wait.h>

#define PORT 9090
#define SIZE 1024

int main() {
    pid_t pid = fork();
    if (pid < 0)
        return 1;

    if (pid == 0) {
        /* Child = Server */
        int server_fd, client_fd;
        struct sockaddr_in server_addr, client_addr;
        socklen_t len = sizeof(client_addr);
        char buffer[SIZE];

        server_fd = socket(AF_INET, SOCK_STREAM, 0);

        int opt = 1;
        setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
        server_addr.sin_port = htons(PORT);

        bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));
        listen(server_fd, 1);

        printf("[Server] Waiting for sensor data...\n");

        client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &len);

        recv(client_fd, buffer, SIZE, 0);
        printf("[Server] Sensor Data Received: %s\n", buffer);
        fflush(stdout);

        close(client_fd);
        close(server_fd);
        exit(0);
    }
    else {
        /* Parent = Client */
        int sockfd;
        struct sockaddr_in server_addr;
        char buffer[SIZE];

        sockfd = socket(AF_INET, SOCK_STREAM, 0);
        server_addr.sin_family = AF_INET;
        server_addr.sin_port = htons(PORT);
        server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

        while (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0);

        printf("Enter sensor data (e.g. Temp:25.6,Humidity:60): ");
        fgets(buffer, SIZE, stdin);
        buffer[strcspn(buffer, "\n")] = '\0';

        send(sockfd, buffer, strlen(buffer) + 1, 0);
        printf("[Client] Data sent to server.\n");
        fflush(stdout);

        close(sockfd);
        wait(NULL);
    }

    return 0;
}
```

---

## Summary — All 6 Exercises

| Exercise | Files | Run Style |
|---|---|---|
| 1 | Sliding Window Protocol (`server.c`, `client.c`) | 2 terminals |
| 2 | Stop-and-Wait Protocol (`server.c`, `client.c`) | 2 terminals |
| 3 | Basic Network Commands | 1 terminal |
| 4 | UDP Echo Chat Program (`server3.c`, `client.c`) | 2 terminals |
| 5 | DNS over UDP (Forward + Reverse) | Part A: 2 terminals, Part B: 1 terminal |
| 6 | TCP Socket Programming (4 parts) | 1 terminal each |

> Note: The original DNS and TCP source you pasted had several OCR/formatting errors (missing braces, jumbled lines, misplaced statements) that would not compile as-is. The code above has been cleaned up and corrected to compile and run correctly while preserving the same logic and algorithm described in each problem statement.
