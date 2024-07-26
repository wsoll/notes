# Circular/Ring Buffer
```c

#include <stdlib.h>
#include <stdio.h>
#include <limits.h>


struct CB {
    int* array;
    int size, head, tail;
};

struct CB* createCB(int size) {
    struct CB* cB = (struct CB*) malloc(sizeof(struct CB));
    cB->size = size;
    cB->head = 0;
    cB->tail = 0;
    cB->array = (int*) malloc(sizeof(int) * size);
    return cB;
}

void enqueue(struct CB** cB, int item) {
    int currentSize =  (*cB)->tail - (*cB)->head;
    if (currentSize == (*cB)->size) {
        printf("Cannot enqueue the item. Buffer is full.\n");
        return;
    }
    (*cB)->array[(*cB)->tail++ % (*cB)->size] = item;
    printf("%d has been enqueued\n", item);
}

int dequeue(struct CB** cB) {
    if ((*cB)->head == (*cB)->tail) {
      printf("Cannot read the item. Buffer is empty.\n");
      return INT_MIN;
    }
    int data = (*cB)->array[(*cB)->head++ % (*cB)->size];
    printf("%d has been dequeued\n", data);
    return data;
}
```

```c

int main() {
    struct CB* cB = createCB(3);
    enqueue(&cB, 10);
    enqueue(&cB, 20);
    enqueue(&cB, 30);
    enqueue(&cB, 40);
    dequeue(&cB);
    enqueue(&cB, 40);
    dequeue(&cB);
    dequeue(&cB);
    dequeue(&cB);
    dequeue(&cB);
    enqueue(&cB, 50);
    enqueue(&cB, 60);
    dequeue(&cB);
    enqueue(&cB, 70);
}

```

```commandline
10 has been enqueued
20 has been enqueued
30 has been enqueued
Cannot enqueue the item. Buffer is full.
10 has been dequeued
40 has been enqueued
20 has been dequeued
30 has been dequeued
40 has been dequeued
Cannot read the item. Buffer is empty.
50 has been enqueued
60 has been enqueued
50 has been dequeued
70 has been enqueued
```