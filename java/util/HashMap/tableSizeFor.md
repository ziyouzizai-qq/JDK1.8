 ```
    cap: 9
        n: 8     ===>    00000000 00000000 00000000 00001000
    |   n >>> 1  ===>    00000000 00000000 00000000 00000100
    --------------------------------------------------------
        n        ===>    00000000 00000000 00000000 00001100
    |   n >>> 2  ===>    00000000 00000000 00000000 00000011
    --------------------------------------------------------
        n        ===>    00000000 00000000 00000000 00001111
    |   n >>> 4  ===>    00000000 00000000 00000000 00000000
    --------------------------------------------------------
        n        ===>    00000000 00000000 00000000 00001111
    。。。
        n = 00000000 00000000 00000000 00001111 = 15


    假设cap不减1，并且指定16为容量
        cap: 16      ===>    00000000 00000000 00000000 00010000
    |   cap   >>> 1  ===>    00000000 00000000 00000000 00001000
    ------------------------------------------------------------
        cap          ===>    00000000 00000000 00000000 00011000
    |   cap   >>> 2  ===>    00000000 00000000 00000000 00000110
    ------------------------------------------------------------
        cap          ===>    00000000 00000000 00000000 00011110
    |   cap   >>> 4  ===>    00000000 00000000 00000000 00000001
    ------------------------------------------------------------
        cap          ===>    00000000 00000000 00000000 00011111
    。。。
        cap = 00000000 00000000 00000000 00011111 = 31

    假设cap减1，并且指定16为容量
    n: 15        ===>    00000000 00000000 00000000 00001111
    |   n >>> 1  ===>    00000000 00000000 00000000 00000111
    --------------------------------------------------------
    。。。
        n = 00000000 00000000 00000000 00001111 = 15
    ```