---
layout: post
title: "Endianess Issues"
date: 2016-07-21 09:40:00 -0700
categories: Programming
tags: programming software c
---

Many of you have likely run into issues with transferring data between big-endian and little-endian systems. One of my projects during my internship required me to send data out of the RAM of a microcontroller to another over SPI. The memory dumps on the transmitter and receiver looked perfect but when I was trying to read the "recreated" structure elements from the memory block, I was reading values that did not seem right.

I checked with the manufacturer's datasheets and the microcontrollers were of different endianness. The master was big-endian and the slave little-endian. At this point, I had to decide how to achieve efficient data transfer between the two.

There are many approaches to tackling this problem. One of them is to flip the byte order in any type that is longer than one byte. This requires knowing the type beforehand which stopped me from proceeding because I was looking for something more generic. I came out with a procedure that does not care about the type and results in the correct endian format.

Let's assume we have received an array of bytes over SPI representing a certain structure of a big-endian machine. Now, we reverse the entire array and apply the same structure but backward (meaning that the element order in the structure is flipped). Now apply the structure to the array you just flipped and that's it.

You can put a `#ifdef`, `#else`, and `#endif` in your structure declaration to make it cross-endian and then simply put a `#define IS_BIG_ENDIAN` on your big-endian machine. If you do not define this, the structure will be declared in reverse for the low-endian machine.

This works for any type, except for arrays. Any sequences of bytes (arrays) are ordered "naturally", in other words, the lowest index of the array resides on the lowest address and the highest in the greatest. To eliminate the inevitable reversing of your array, you have to reverse the array element in your structure separately after you do the complete reverse. If the array in your struct is of type `char` or `uint8_t`, you can simply apply the same method used previously. If your array is of the type that is larger than one byte (`short`, `int`, `long`, etc.), you will have to reverse the order of multiple-byte blocks. Note that with the complete reverse, the bytes of an individual multi-byte entity are in the correct order but the order of the multi-byte entities is wrong. An example of how to accomplish this would be (let's assume we have a `uint16_t`) to take the first two bytes and swap them with the last two bytes, next we take the third and fourth bytes and swap them with the third and fourth last bytes. The procedure continues until you reach the middle of the array. This is crucial since the code below does not show that.

{% highlight c %}
#include <stdio.h>
#include <stdint.h>
#include <string.h>

// #define IS_BIG_ENDIAN

#pragma pack(push, 1)
typedef struct {
    #ifdef IS_BIG_ENDIAN
    uint8_t byteTest;
    uint16_t shortTest;
    uint32_t longTest;
    uint8_t byteArrayTest[8];
    #else
    uint8_t byteArrayTest[8];
    uint32_t longTest;
    uint16_t shortTest;
    uint8_t byteTest;
    #endif
}
Test_t;
#pragma pack(pop)

void init_struct(Test_t * s) {
    s -> byteTest = 0x11;
    s -> shortTest = 0x2233;
    s -> longTest = 0x44556677;

    s -> byteArrayTest[0] = 0x1F;
    s -> byteArrayTest[1] = 0x2F;
    s -> byteArrayTest[2] = 0x3F;
    s -> byteArrayTest[3] = 0x4F;
    s -> byteArrayTest[4] = 0x5F;
    s -> byteArrayTest[5] = 0x6F;
    s -> byteArrayTest[6] = 0x7F;
    s -> byteArrayTest[7] = 0x8F;
}

void reverse_array(uint8_t * array_ptr, uint16_t byte_length) {
    uint16_t i, j;
    uint8_t temp;

    j = byte_length - 1;
    for (i = 0; i < (byte_length / 2); i++) {
        temp = array_ptr[i];
        array_ptr[i] = array_ptr[j];
        array_ptr[j] = temp;
        j--;
    }
}

void print_array(uint8_t * array, int length) {
    int i;
    for (i = 0; i < length; i++) {
        printf("0x%02X ", array[i]);
    }
}

void print_struct(Test_t * s) {
    printf("structure: \n");
    printf(" byteTest = 0x%02X\n", s -> byteTest);
    printf(" shortTest = 0x%04X\n", s -> shortTest);
    printf(" longTest = 0x%08X\n", s -> longTest);
    printf(" byteArrayTest = ");
    print_array(s -> byteArrayTest, sizeof(s -> byteArrayTest));
    printf("\n\n");
}

void hex_dump(const char * desc,
    const void * addr, uint16_t len) {

    uint16_t i;
    uint8_t buff[17];
    uint8_t * pc = (uint8_t * ) addr;

    if (desc != NULL) {
        printf("%s:\r\n", desc);
    }

    for (i = 0; i < len; i++) {
        if ((i % 16) == 0) {
            if (i != 0)
                printf(" %s\r\n", buff);
            printf(" %04X ", i);
        }

        printf(" %02X", pc[i]);

        if ((pc[i] < 0x20) || (pc[i] > 0x7E)) {
            buff[i % 16] = '.';
        } else {
            buff[i % 16] = pc[i];
        }
        buff[(i % 16) + 1] = '\0';
    }

    while (i % 16 != 0) {
        printf(" ");
        i++;
    }
    printf(" %s\r\n\n", buff);
}

int main() {
    uint8_t buffer[15] = {
        0x11,
        0x22,
        0x33,
        0x44,
        0x55,
        0x66,
        0x77,
        0x1F,
        0x2F,
        0x3F,
        0x4F,
        0x5F,
        0x6F,
        0x7F,
        0x8F
    };
    Test_t test;
    hex_dump("buffer array before", & buffer, sizeof(buffer));
    reverse_array((void * ) & buffer, sizeof(buffer));
    hex_dump("buffer array after total reverse", & buffer, sizeof(buffer));
    memcpy( & test, & buffer, sizeof(buffer));
    print_struct( & test);
    reverse_array((void * ) & (test.byteArrayTest), sizeof(test.byteArrayTest));
    hex_dump("struct after array back-reverse", (void * ) & test, sizeof(test));
    print_struct( & test);
    return 0;
}
{% endhighlight %}
