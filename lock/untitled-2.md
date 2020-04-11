# Exercise: Implement atomic counter

Multiple threads running `counter = counter + 1;` could leads to unexpected results due to GCC/Compiler generates multiple assembly instructions for assignment. 

Multiprocess, thread switching, device interrupt could lead to invalid results.

Example program uses `counter = counter + 1`

```c
int max;
volatile int counter = 0; // shared global variable

void *mythread(void *arg) {
    char *letter = arg;
    int I; // stack (private per thread) 
    printf(“%s: begin [addr of I: %p]\n”, letter, &i);
    for (I = 0; I < max; I++) {
        counter = counter + 1; // shared: only one
    }
    printf("%s: done\n", letter);
    return NULL;
}

int main(int argc, char *argv[]) {                    
    if (argc != 2) {
    fprintf(stderr, "usage: main-first <loopcount>\n");
    exit(1);
    }
    max = atoi(argv[1]);

    pthread_t p1, p2;
    printf("main: begin [counter = %d] [%x]\n", counter, 
       (unsigned int) &counter);
    Pthread_create(&p1, NULL, mythread, "A"); 
    Pthread_create(&p2, NULL, mythread, "B");
    // join waits for the threads to finish
    Pthread_join(p1, NULL); 
    Pthread_join(p2, NULL); 
    printf("main: done\n [counter: %d]\n [should: %d]\n", 
       counter, max*2);
    return 0;
}
```

Result

```text
$ ./t1 1000000
main: begin [counter = 0] [9aad048]
A: begin [addr of I: 0x70000a9d8f9c]
B: begin [addr of I: 0x70000aa5bf9c]
B: done
A: done
main: done
 [counter: 1138660]
 [should: 2000000]
```

### Use atomic operation

Replace the `counter = counter + 1`, with `__sync_fetch_and_add(&counter, 1);`

**Purpose**

This function atomically adds the value of **v to the variable that** p points to. The result is stored in the address that is specified by \_\_p.

A full memory barrier is created when this function is invoked.

**Prototype**

`T __sync_fetch_and_add (T* __p, U __v, …);`

**Parameters**

`__p` The pointer of a variable to which **v is to be added. The value of this variable is to be changed to the result of the add operation. \`**v\` The variable whose value is to be added to the variable that **p points to. Return value The function returns the initial value of the variable that** p points to.

### Expected Result

```text
$ ./t1 1000000
main: begin [counter = 0] [fe3f048]
A: begin [addr of I: 0x700006be2f9c]
B: begin [addr of I: 0x700006c65f9c]
B: done
A: done
main: done
 [counter: 2000000]
 [should: 2000000]
```

