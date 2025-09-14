#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>

// Structure to represent a memory frame
typedef struct {
    int page; // Page number stored in the frame
    int access_time; // For LRU: tracks when the page was last accessed
} Frame;

// Structure to represent a queue node for FIFO
typedef struct QueueNode {
    int page;
    struct QueueNode* next;
} QueueNode;

typedef struct {
    QueueNode* front;
    QueueNode* rear;
} Queue;

// Function prototypes
void initQueue(Queue* q);
void enqueue(Queue* q, int page);
int dequeue(Queue* q);
bool isQueueFull(Queue* q, int capacity);
bool isPageInQueue(Queue* q, int page);
void printMemory(Frame* frames, int capacity, int page_faults, const char* algo);
int findLRU(Frame* frames, int capacity);
int predictOptimal(int page[], int pn, int index, Frame* frames, int capacity);

// FIFO Page Replacement
int fifoPageReplacement(int pages[], int n, int capacity) {
    Queue q;
    initQueue(&q);
    Frame* frames = (Frame*)malloc(capacity * sizeof(Frame));
    for (int i = 0; i < capacity; i++) {
        frames[i].page = -1; // Initialize frames as empty
    }
    int page_faults = 0;

    printf("\nFIFO Page Replacement:\n");
    for (int i = 0; i < n; i++) {
        if (!isPageInQueue(&q, pages[i])) {
            if (isQueueFull(&q, capacity)) {
                int victim = dequeue(&q);
                for (int j = 0; j < capacity; j++) {
                    if (frames[j].page == victim) {
                        frames[j].page = pages[i];
                        break;
                    }
                }
            } else {
                for (int j = 0; j < capacity; j++) {
                    if (frames[j].page == -1) {
                        frames[j].page = pages[i];
                        break;
                    }
                }
            }
            enqueue(&q, pages[i]);
            page_faults++;
            printMemory(frames, capacity, page_faults, "FIFO");
        } else {
            printMemory(frames, capacity, page_faults, "FIFO");
        }
    }
    free(frames);
    while (q.front != NULL) {
        QueueNode* temp = q.front;
        q.front = q.front->next;
        free(temp);
    }
    return page_faults;
}

// LRU Page Replacement
int lruPageReplacement(int pages[], int n, int capacity) {
    Frame* frames = (Frame*)malloc(capacity * sizeof(Frame));
    for (int i = 0; i < capacity; i++) {
        frames[i].page = -1;
        frames[i].access_time = 0;
    }
    int page_faults = 0;
    int time = 0;

    printf("\nLRU Page Replacement:\n");
    for (int i = 0; i < n; i++) {
        bool found = false;
        for (int j = 0; j < capacity; j++) {
            if (frames[j].page == pages[i]) {
                frames[j].access_time = ++time;
                found = true;
                break;
            }
        }
        if (!found) {
            int lru_index = findLRU(frames, capacity);
            frames[lru_index].page = pages[i];
            frames[lru_index].access_time = ++time;
            page_faults++;
        }
        printMemory(frames, capacity, page_faults, "LRU");
    }
    free(frames);
    return page_faults;
}

// Optimal Page Replacement
int optimalPageReplacement(int pages[], int n, int capacity) {
    Frame* frames = (Frame*)malloc(capacity * sizeof(Frame));
    for (int i = 0; i < capacity; i++) {
        frames[i].page = -1;
    }
    int page_faults = 0;

    printf("\nOptimal Page Replacement:\n");
    for (int i = 0; i < n; i++) {
        bool found = false;
        for (int j = 0; j < capacity; j++) {
            if (frames[j].page == pages[i]) {
                found = true;
                break;
            }
        }
        if (!found) {
            if (i == 0 || page_faults < capacity) {
                for (int j = 0; j < capacity; j++) {
                    if (frames[j].page == -1) {
                        frames[j].page = pages[i];
                        break;
                    }
                }
            } else {
                int replace_index = predictOptimal(pages, n, i, frames, capacity);
                frames[replace_index].page = pages[i];
            }
            page_faults++;
        }
        printMemory(frames, capacity, page_faults, "Optimal");
    }
    free(frames);
    return page_faults;
}

// Initialize queue
void initQueue(Queue* q) {
    q->front = q->rear = NULL;
}

// Enqueue a page
void enqueue(Queue* q, int page) {
    QueueNode* newNode = (QueueNode*)malloc(sizeof(QueueNode));
    newNode->page = page;
    newNode->next = NULL;
    if (q->rear == NULL) {
        q->front = q->rear = newNode;
    } else {
        q->rear->next = newNode;
        q->rear = newNode;
    }
}

// Dequeue a page
int dequeue(Queue* q) {
    if (q->front == NULL) return -1;
    QueueNode* temp = q->front;
    int page = temp->page;
    q->front = q->front->next;
    if (q->front == NULL) q->rear = NULL;
    free(temp);
    return page;
}

// Check if queue is full
bool isQueueFull(Queue* q, int capacity) {
    int count = 0;
    QueueNode* temp = q->front;
    while (temp != NULL) {
        count++;
        temp = temp->next;
    }
    return count >= capacity;
}

// Check if page is in queue
bool isPageInQueue(Queue* q, int page) {
    QueueNode* temp = q->front;
    while (temp != NULL) {
        if (temp->page == page) return true;
        temp = temp->next;
    }
    return false;
}

// Find the index of the least recently used page
int findLRU(Frame* frames, int capacity) {
    int min_time = frames[0].access_time;
    int lru_index = 0;
    for (int i = 1; i < capacity; i++) {
        if (frames[i].access_time < min_time) {
            min_time = frames[i].access_time;
            lru_index = i;
        }
    }
    return lru_index;
}

// Predict the page to replace for Optimal algorithm
int predictOptimal(int page[], int pn, int index, Frame* frames, int capacity) {
    int res = -1, farthest = index;
    for (int i = 0; i < capacity; i++) {
        int j;
        for (j = index; j < pn; j++) {
            if (frames[i].page == page[j]) {
                if (j > farthest) {
                    farthest = j;
                    res = i;
                }
                break;
            }
        }
        if (j == pn) return i; // Page not found in future, replace it
    }
    return (res == -1) ? 0 : res;
}

// Print memory state
void printMemory(Frame* frames, int capacity, int page_faults, const char* algo) {
    printf("%s | Frames: [", algo);
    for (int i = 0; i < capacity; i++) {
        if (frames[i].page == -1) printf(" -");
        else printf(" %d", frames[i].page);
    }
    printf(" ] | Page Faults: %d\n", page_faults);
}

int main() {
    int pages[] = {1, 3, 0, 3, 5, 6, 3}; // Example page reference string
    int n = sizeof(pages) / sizeof(pages[0]);
    int capacity = 3; // Number of memory frames

    printf("Page Reference String: ");
    for (int i = 0; i < n; i++) {
        printf("%d ", pages[i]);
    }
    printf("\nNumber of Frames: %d\n", capacity);

    int fifo_faults = fifoPageReplacement(pages, n, capacity);
    int lru_faults = lruPageReplacement(pages, n, capacity);
    int opt_faults = optimalPageReplacement(pages, n, capacity);

    printf("\nSummary:\n");
    printf("FIFO Page Faults: %d\n", fifo_faults);
    printf("LRU Page Faults: %d\n", lru_faults);
    printf("Optimal Page Faults: %d\n", opt_faults);

    return 0;
}
