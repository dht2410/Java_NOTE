## JDK中的NIO

系统调用	socket——》bind——》listen	无论BIO/NIO都用

accept，会阻塞，返回一个文件描述符，BIO形式

**根本原因是accept和recv都阻塞，放在同一个线程里，一个会耽误另一个**

阻塞受制于内核



NIO在java里是New IO，操作系统语境下是Non-blocking IO



