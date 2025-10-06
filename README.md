#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

// --------- Data Structures ---------

// Frame structure for memory
typedef struct {
    int page;
    int access_time;  // For LRU
} Frame;

// Node for FIFO queue
typedef struct QueueNode {
    int page;
    struct QueueNode* next;
} QueueNode;

// Queue structure
typedef struct {
    QueueNode* front;
    QueueNode* rear;
} Queue;

// --------- Queue Functions (for FIFO) ---------

void initQueue(Queue* q) {
    q->front = q->rear = NULL;
}

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

int dequeue(Queue* q) {
    if (q->front == NULL) return -1;
    QueueNode* temp = q->front;
    int page = temp->page;
    q->front = q->front->next;
    if (q->front == NULL)
        q->rear = NULL;
    free(temp);
    return page;
}

bool isQueueFull(Queue* q, int capacity) {
    int count = 0;
    QueueNode* temp = q->front;
    while (temp != NULL) {
        count++;
        temp = temp->next;
    }
    return count >= capacity;
}

bool isPageInQueue(Queue* q, int page) {
    QueueNode* temp = q->front;
    while (temp != NULL) {
        if (temp->page == page)
            return true;
        temp = temp->next;
    }
    return false;
}

// --------- Utility Functions ---------

void printMemory(Frame* frames, int capacity, int page_faults, const char* algo) {
    printf("%s | Frames: [", algo);
    for (int i = 0; i < capacity; i++) {
        if (frames[i].page == -1)
            printf(" -");
        else
            printf(" %d", frames[i].page);
    }
    printf(" ] | Page Faults: %d\n", page_faults);
}

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

int predictOptimal(int pages[], int total, int current, Frame* frames, int capacity) {
    int farthest = current;
    int replace_index = -1;

    for (int i = 0; i < capacity; i++) {
        int j;
        for (j = current; j < total; j++) {
            if (frames[i].page == pages[j]) {
                if (j > farthest) {
                    farthest = j;
                    replace_index = i;
                }
                break;
            }
        }
        if (j == total)
            return i;  // Page not used again
    }
    return (replace_index == -1) ? 0 : replace_index;
}

// --------- Page Replacement Algorithms ---------

int fifoPageReplacement(int pages[], int n, int capacity) {
    Queue q;
    initQueue(&q);
    Frame* frames = (Frame*)malloc(capacity * sizeof(Frame));
    for (int i = 0; i < capacity; i++)
        frames[i].page = -1;

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
        }
        printMemory(frames, capacity, page_faults, "FIFO");
    }

    // Free queue memory
    while (q.front != NULL) {
        QueueNode* temp = q.front;
        q.front = q.front->next;
        free(temp);
    }
    free(frames);
    return page_faults;
}

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

int optimalPageReplacement(int pages[], int n, int capacity) {
    Frame* frames = (Frame*)malloc(capacity * sizeof(Frame));
    for (int i = 0; i < capacity; i++)
        frames[i].page = -1;

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
            bool empty_found = false;
            for (int j = 0; j < capacity; j++) {
                if (frames[j].page == -1) {
                    frames[j].page = pages[i];
                    empty_found = true;
                    break;
                }
            }
            if (!empty_found) {
                int index_to_replace = predictOptimal(pages, n, i + 1, frames, capacity);
                frames[index_to_replace].page = pages[i];
            }
            page_faults++;
        }
        printMemory(frames, capacity, page_faults, "Optimal");
    }
    free(frames);
    return page_faults;
}

// --------- Main Function ---------

int main() {
    int pages[] = {1, 3, 0, 3, 5, 6, 3};  // Example reference string
    int n = sizeof(pages) / sizeof(pages[0]);
    int capacity = 3;  // Number of memory frames

    printf("=== PAGE REPLACEMENT ALGORITHM SIMULATOR ===\n");
    printf("Page Reference String: ");
    for (int i = 0; i < n; i++) {
        printf("%d ", pages[i]);
    }
    printf("\nNumber of Frames: %d\n", capacity);

    int fifo_faults = fifoPageReplacement(pages, n, capacity);
    int lru_faults = lruPageReplacement(pages, n, capacity);
    int opt_faults = optimalPageReplacement(pages, n, capacity);

    // Summary
    printf("\n=== Summary ===\n");
    printf("FIFO Page Faults   : %d\n", fifo_faults);
    printf("LRU Page Faults    : %d\n", lru_faults);
    printf("Optimal Page Faults: %d\n", opt_faults);

    return 0;
}
