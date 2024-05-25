# Circular Buffer
```c
#include <stdlib.h>
#include <stdio.h>
#include <limits.h>

// Ring Buffer
// Circular Buffer

struct CB
{
  int* array;
  int size, head, tail;
};

struct CB* createCB(int size)
{
  struct CB* cB = (struct CB*) malloc(sizeof(struct CB));
  cB->size = size;
  cB->head = 0;
  cB->tail = 0;
  cB->array = (int*) malloc(sizeof(int) * size);
}

void save(struct CB** cB, int data)
{
  int currentSize =  (*cB)->tail - (*cB)->head;
  if (currentSize == (*cB)->size)
    return;
  (*cB)->array[(*cB)->tail++ % (*cB)->size] = data;
  printf("%d has been saved\n", data);
}

int read(struct CB** cB)
{
  if ((*cB)->head == (*cB)->tail)
    return INT_MIN;
  return (*cB)->array[(*cB)->head++ % (*cB)->size];
}
```