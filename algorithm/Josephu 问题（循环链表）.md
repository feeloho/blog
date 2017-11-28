# Josephu 问题（循环链表）

```c
#include <stdio.h>

typedef int Elem;
typedef struct {
    Elem elem;
    struct Node *next;
} Node;

Node *create(int num) {
    Node *n, *h, *t;
    h = n = (Node *) malloc(sizeof(Node));

    for (int i=1; i<=num; i++) {
        t = (Node *) malloc(sizeof(Node));
        t->elem = i;

        n->next = t;
        n = t;
    }

    n->next = h->next;
    return n->next;
}

int main() {
    Node *n = create(41);

    int i = 0;
    while (n->next != n) {
        if (++i % 2 == 0) {
            Node *t = n->next;
            printf("%d -> ", t->elem);
            n->next = t->next;
        }

        n = n->next;
    }

    printf("%d\n", n->elem);
}
```

