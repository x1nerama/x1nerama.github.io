---
layout: post
title: Conditionals and Loops in Assembly
description: Process Injection Attacks
tags: x86-assembly nasm  
---

Merhabalar. bu blogumda Assembly'de döngülerden ve Koşullu ve Koşulsuz karşılaştırmaları ele alacağım. Bu iki konuyu neden tek bir çatı altında anlattığımı ilerideki dakikalarda bahsedeceğim.

## 0x0 – Conditionals (Karşılaştırmalar)
Teorik olarak karşılaştırmalardan bahsetmeyeceğim ancak ufak bir tanım yapmak gerekirse, belirtilen koşula bağlı olarak programın akışını değiştirilmesini sağlar.

Yüksek seviyeli birçok dilde karşılaştırmalar için if-else kullanılır bunu zaten biliyorsunuzdur. Assembly'de karşılaştırmalar diğer dillere göre farklıdır.

Assembly'de karşılaştırma durumu ikiye ayrılır: Conditional Jump (Koşullu Atlama) ve Unconditional Jump (Koşulsuz Atlama) olarak ikiye ayrılır. Bu konuda ikisini de detaylandırmaya çalışacağım.

Conditional Jump yani koşullu zıplamanın isminden de anlaşıldığı gibi koşulun sağlanması durumunda programın akışını değiştirmesini sağlar. Unconditional Jump yani koşulsuz zıplama ise direkt bir koşul olmadan programın akışını değiştirir. Özellikle Binary Exploitation tarafıyla ilgileniyorsanız bufferoverflow saldırılarında unconditional jump önemli bir konudur programın akışını değiştirmek açısından.

## 0x00 - CMP Instruction
Assembly'de karşılaştırma yapmak için cmp Instruction'u kullanıyoruz. Bu cmp Instruction’u iki işleneni karşılaştırmak için kullanılır. Program bu Instruction'u kullanırken, iki işlenenin eşit olup olmadığını anlamak için birbirinden çıkarır. ‘Çıkarmak’ derken ikisi arasında bir çıkarma işlemi yapar. Bu karşılaştırma sonucunu ZF (Zero Flag) ile anlayabiliriz. Eğer cmp sonucunda Zero Flag 0 olarak resetlenmişse karşılaştıranların eşit olmadığını gösterir ancak Zero flag 1 olarak setlenirse iki tarafın eşit olduğunu gösterir.

Burada Zero Flag’ın ne olduğunu anlamamız gerekiyor. Zero Flag, herhangi bir işlem sonucunun sıfır olması durumunda 1 değerini alır. Ancak herhangi bir işlem sonucu sıfır ile sonuçlanmazsa 0 ile resetlenir.

Konu biraz karışık olduğu için örnek kod ile detaylandıralım:​

Kod:

```asm
section .data
    msg: db "Assembly with bekoo",0
    len: equ $ - msg

section .text
    global _start

_start:
    mov ah,5    ; ah register'a 5 yükle
    mov al,5    ; al register'a 5 yükle
    cmp ah,al   ; ah ile al register'ı karşılaştır
    jz _printMessage  ; karşılaştırma sonucu 0 ise _printMessage fonksiyonuna git

_exit:
    ; programı bitir
    mov eax,1
    xor ebx,ebx
    int 0x80

_printMessage:
    ; msg'ı ekrana bastır
    mov eax,4
    mov ebx,1
    mov ecx,msg
    mov edx,len
    int 0x80

    ;  _exit fonksiyonuna git
    jmp _exit
```

Bu programda ah ile al register'a 5 değerini gönderilir. Ardından yukarıda bahsettiğim cmp aracılığıyla karşılaştırma yapılır. Eğer bu karşılaştırma sonucu 0 ile sonuçlanırsa program _printMessage adlı fonksiyona gönderiliyor değil ise _exit fonksiyonuna gidip program bitirilir.

Burada karşılaştırmanın 0 olması demek, karşılaştırılan iki değerin de eşit olduğunu gösterir. Mesela ah register'a 3 ve al register'a 5 gönderip tekrar karşılaştırmaya tabii tuttuğumuzda Zero Flag set edilmez. Çünkü cmp Instruction'ın yaptığı çıkartma işlemi sonucu 0 olmayacak.

Şimdi ise karşılaştırma sonucunda Zero Flag, gerçekten set edilmiş mi gdb ile ona bakalım:

Kod:
```shell
gdb-peda$ disas _start
Dump of assembler code for function _start:
   0x0000000000401000 <+0>:    mov    ah,0x5
   0x0000000000401002 <+2>:    mov    al,0x5
   0x0000000000401004 <+4>:    cmp    ah,al
   0x0000000000401006 <+6>:    je     0x401011 <_printMessage>
End of assembler dump.
```

gdb ile _start içeriğine baktığımızda bu kodlar bizi karşılıyor. Şimdi ise break *0x0000000000401006 yazıp karşılaştırmanın sonucuna bakalım. Programın breakpoint koyduğum kısma dikkat edin. Program tam _printMessage adlı fonksiyona gitmeden duracak:

Kod:
```shell
gdb-peda$ disas
Dump of assembler code for function _start:
   0x0000000000401000 <+0>:    mov    ah,0x5
   0x0000000000401002 <+2>:    mov    al,0x5
   0x0000000000401004 <+4>:    cmp    ah,al
=> 0x0000000000401006 <+6>:    je     0x401011 <_printMessage>
End of assembler dump.
```

Programın nerede olduğunu => işareti ile anlayabiliriz. Bu işaret, çalıştırılacak kodu gösterir. Şimdi ise info registers $eflags ile hangi flag'ların set edildiğine bakalım:

Kod:
```shell
gdb-peda$ i r $eflags
eflags         0x246               [ PF ZF IF ]
```

eflags register'ın içerisinde gördüğünüz flag'lar, set edilen flag'lardır. Dolayasıyla programın şuan ki kısmında PF, ZF ve IF flag'ları set edilmiş olduğunu görebiliyoruz. Yani programı devam ettirdiğimizde, jz karşılaşmayı sağladığı için _printMessage adlı fonksiyona gidecek ve mesajı ekrana bastıracak.

jz veya jmp gibi şeyler kafanızı karıştırmasın şimdi ise bunları detaylandıracağım. Burada sadece anlamız gereken şey cmp'ın mantığı ve Zero Flag'ın ne işe yaradığı.


<h2> 0x01 - Conditional Jump (Koşullu Atlama) </h2>

Koşullu zıplamanın zaten adıyla ne olduğunu anlamışsınızdır. Belirli koşulun sağlanması durumunda programın akışını değiştirmesini sağlar.

Koşullu işlemler için bir çok Instruction bulunmaktadır. Koşullar için kullanılan genellike arimetiksel Instruction'lardır. Bu konumda ise sadece bunlara yer vereceğim. 

Bu kadar şeyden sonra yukarıdaki örnek Assembly koduna tekrar dönelim:

Kod:
```asm
_start:
    mov ah,5    ; ah register'a 5 yükle
    mov al,5    ; al register'a 5 yükle
    cmp ah,al   ; ah ile al register'ı karşılaştır
    jz _printMessage  ; karşılaştırma sonucu 0 ise _printMessage fonksiyonuna git
```

Bu kodda JZ Instruction'u kullandım. Siz burada JZ değil de JE'de kullansanız program aynı sonucu gösterir.


<h2> 0x02 - Unconditional Jump (Koşulsuz Atlama) </h2>

Yep, şimdi ise asıl önemli konuya geldik.

Conditional için koşullu olarak programın akışını değiştirdiğini söylemiştim. Unconditional'da ise koşula bağlı olmadan direkt programın akışını değiştirir.

Unconditional için sadece jmp Instruction kullanılır. Örnek Assembly kodunu ele alayım:
​
Kod:

```asm
_printMessage:
    mov eax,4
    mov ebx,1
    mov ecx,msg
    mov edx,len
    int 0x80

    jmp _exit
```

_printMessage adlı fonksiyon içerisine baktığımızda write syscall ile bir ekrana bastırma işlemi yapılıyor ardından jmp aracılığıyla program, _exit fonksiyona gidiyor ve program sonlanıyor.

Yani kısaca jmp ile koşula bağlı olmadan programın akışını değiştirebilirsiniz.


<h2> 0x1 – Loops </h2>

Gel gelelim asıl konumuza, döngülerrr.

Bilirsiniz döngüler, programlamada dillerinde önemli bir konudur. Assembly'de ise bambaşka bir yeri var.

Eğer Assembly ile döngü oluşturmak isterseniz ya yukarıda bahsettiğim koşullu atlama yöntemi ile ya da loop Instruction'u ile yapabilirsiniz. Bu konu da ikisinden de bahsedeceğim.

<h2> 0x10 - Loop Instruction </h2>

loop Instruction'u, döngüler için kullandığımız bir komuttur. İşlevli bir Instruction olsa da kendi görüşüme göre pek yaygın bir kullanımı yok. Daha çok yukarıda bahsettiğim, koşullu atlamalar ile döngüler oluşturmak daha yaygındır.

Her şeyden önce loop Instruction'un çalışma mantığını anlamak gerek. Bu Instruction, ecx (Extended Control Register) register'ına verilen değere göre komutu çalıştırır. Peki, 'neden ecx register? Farklı register'a değer veremez miyiz?' diye bir soru gelebilir, Hayır. Eğer loop instruction ile çalışacaksanız sadece ecx'e değer vererek döngü sayısını belirleyebilirsiniz. ecx'in işlemcide döngünün sayacını tutmak gibi görevleri vardır.

Aynı zamanda loop instruction, her çalıştığında ecx'den bir değer azaltır. Mesela siz bir mesajı 10 defa ekrana bastırmak istiyorsunuz ve ecx'e 10 değerini verdiniz. loop komutu her çalıştığında ecx'i 1 azaltır. Kafanız karışmasın örnek ile detaylandıracağım.

Şimdi ise basit bir örnek yapalım. Örneğimiz, "bekoo" mesajını 10 kere bastırmak olsun:
​
Kod:

```asm
BITS 32

section .data
    msg: db "bekoo", 0
    len: equ $ - msg

section .text
    global _start

_start:
    mov ecx,10 ; döngü sayısı 10 olarak belirtildi
    jmp _loop   ; programı _loop fonksiyona yönelt

_exit:
    ; programı bitir
    mov eax,1
    xor ebx,ebx
    int 0x80

_loop:
    push ecx ; ecx'in değerini stack' gönder
    ; ecx'in değerini stack'e gönderme sebebimiz, kodun devamında ekrana bastırma işlemi için ecx'i kullanacağız dolayasıyla bu değeri kaybetmemek için stack'e gönderiyoruz.
    
    ; mesajı ekrana bastır
    mov eax,4
    mov ebx,1
    mov ecx,msg
    mov edx,len
    int 0x80

    pop ecx ; stack'e gönderilen değeri al
    loop _loop  ; _başa dön

    cmp ecx,0 ; ecx'in değeri 0 ile karşılaştırılıyor
    jmp _exit   ; eğer ecx'in değeri 0 olmuşsa jmp ile programı _exit fonksiyona yönelt
```

Bu program, 10 kez 'bekoo' mesajını ekrana bastırır.

Programda başta ecx'e 10 değeri gönderilir. Buradaki 10, döngünün kaç defa çalışacağını temsil eder. Siz buraya 10 değil de mesela 3 girseniz 3 defa çalışacaktır. Ardından jmp aracılığıyla program, _loop fonksiyonuna yönlendiriliyor. Bu fonksiyon içerisinde ise mesaj bastırılmadan hemen önce ecx'in değeri stack'e gönderiliyor. Bunun sebebi, ilerideki kodlarda write syscall ile ekrana mesaj bastırma işlemi gerçekleştirelecek. Eğer ecx'in değeri stack'e gönderilmeden direkt mesajı ekrana bastırırsak, döngü sayısını kaybedebiliriz.

Mesaj ekrana bastırıldıktan sonra pop aracılığıyla stack'e gönderilen değer geri alınıyor ve loop instruction ile bu işlemler tekrar ediyor. Taaa ki, ecx register içerisinde değer 0 olana kadar. Hatırlayın, loop instruction'u ecx'a bağlı çalışır ve her çalıştığında ecx'den 1 değer azaltır.

Özellikle bu kodda, conditionals ve unconditionals konuların daha anlaşılır olması için jmp ve cmp komutlarını da dahil ettim.

Arka plana bakmak için gdb kullanalım:

Kod:
```shell
gdb-peda$ disas _start
Dump of assembler code for function _start:
   0x0000000000401000 <+0>:    mov    ecx,0xa
   0x0000000000401005 <+5>:    jmp    0x401010 <_loop>
End of assembler dump.
```

_start fonksiyonu disassemble ettiğimiz zaman bu kodları görüyoruz. Yukarıda bahsettiğim gibi, ecx'e hexdecimal olarak 10 gönderiliyor ve program jmp ile _loop'a yönlendiriliyor. Şimdi ise _loop'u disassemble edelim:

Kod:
```shell
gdb-peda$ disas _loop
Dump of assembler code for function _loop:
   0x0000000000401010 <+0>:     push   rcx
   0x0000000000401011 <+1>:     mov    eax,0x4
   0x0000000000401016 <+6>:     mov    ebx,0x1
   0x000000000040101b <+11>:    mov    ecx,0x402000
   0x0000000000401020 <+16>:    mov    edx,0xf
   0x0000000000401025 <+21>:    int    0x80
   0x0000000000401027 <+23>:    pop    rcx
   0x0000000000401028 <+24>:    loop   0x401010 <_loop>
   0x000000000040102a <+26>:    cmp    ecx,0x0
   0x000000000040102d <+29>:    jmp    0x401007 <_exit>
End of assembler dump.
```

_loop içerisine başta baktığımızda başta rcx değeri - program arka planda rcx kullanmış. Burada ecx'in değeri bulunmaktadır - stack'e gönderildiği ardından ekrana bastırma işlemi yapıldığı görülmektedir. Daha sonra pop ile stack'e gönderilen değer geri alınıyor ve loop komutu çalıştırılıyor.

loop instruction'un her çalıştığında ecx'den bir değer azaltığını söylemiştim. Şimdi ise canlı canlı bir de buradan bakalım. Başta break *0x0000000000401028 ile loop komutunun olduğu yerde programı durduralım ve ardından çalıştıralım:

Kod:
```shell
gdb-peda$ disas
Dump of assembler code for function _loop:
   0x0000000000401010 <+0>:   push   rcx
   0x0000000000401011 <+1>:   mov    eax,0x4
   0x0000000000401016 <+6>:   mov    ebx,0x1
   0x000000000040101b <+11>:  mov    ecx,0x402000
   0x0000000000401020 <+16>:  mov    edx,0xf
   0x0000000000401025 <+21>:  int    0x80
   0x0000000000401027 <+23>:  pop    rcx
=> 0x0000000000401028 <+24>:  loop   0x401010 <_loop>
   0x000000000040102a <+26>:  cmp    ecx,0x0
   0x000000000040102d <+29>:  jmp    0x401007 <_exit>
End of assembler dump.
```

Program, tam loop komutu çalıştırılmadan hemen durdu. Tam burada info registers $ecx aracılığıyla ecx'in durumuna bir bakalım:

Kod:
```shell
gdb-peda$ info registers $ecx
ecx            0xa                 0xa
```

0xa'nın decimal karşılığı 10'dur. Dolayasıyla şuan ecx'in içerisinde 10 değeri var. Şimdi programı ni aracılığıyla devam ettirelim ve loop komutu çalıştırıldıktan sonra tekrar bakalım:

Kod:
```shell
gdb-peda$ disas
Dump of assembler code for function _loop:
=> 0x0000000000401010 <+0>:    push   rcx
   0x0000000000401011 <+1>:    mov    eax,0x4
   0x0000000000401016 <+6>:    mov    ebx,0x1
   0x000000000040101b <+11>:   mov    ecx,0x402000
   0x0000000000401020 <+16>:   mov    edx,0xf
   0x0000000000401025 <+21>:   int    0x80
   0x0000000000401027 <+23>:   pop    rcx
   0x0000000000401028 <+24>:   loop   0x401010 <_loop>
   0x000000000040102a <+26>:   cmp    ecx,0x0
   0x000000000040102d <+29>:   jmp    0x401007 <_exit>
End of assembler dump.
```

Göründüğü gibi, _loop fonksiyonun başına tekrar dönüldü. Aynı şekilde tekrar info registers $ecx ile ecx'in durumuna tekrar bakalım:

Kod:
```shell
gdb-peda$ info registers $ecx
ecx            0x9                 0x9
```

Programın şuan ki kısmında ise ecx'in içerisinde 9 değeri var. Yani ecx'in 1 değer azaltıldığını görebiliyoruz. Bu program böyle devam edecek ve en sonunda 0 olduğunda program bitecek.

Genel olarak loop instruction'un çalışma mantığı bu şekilde. Şimdi ise diğer konuya geçelim.


<h2> 0x11 - Loops with Conditionals </h2>

Şimdi ise en çok tercih ettiğimiz yönteme geldik.

Koşullarla döngü oluşturmak sıklıkla tercih edilen bir yöntemdir.

Konuyu aydınlatmak için basit bir örnek yapalım. Yukarıdaki uygulamanın tersini yapalım. Bu sefer ecx'i 0 değerinden başlatalım ve 10 olana kadar 'bekoo' mesajını ekrana bastırsın:

Kod:

```asm
section .data
        msg: db "bekoo", 0
        len: equ $ - msg

section .text
        global _start

_start:
        mov ecx,1
        jmp _loop

_exit:
        mov eax,1
        xor ebx,ebx
        int 0x80

_loop:
        cmp ecx, 10 ; ecx'in değeri 10 ile karşılaştırılıyor
        jz _exit  ; eğer ecx'in değeri 10 ile eşitse, program _exit fonksiyonuna yönlendirilerek program bitiriliyor
        
        ; bu kısımda msg ekrana bastırılır
        mov eax,4  
        mov ebx,1
        mov ecx,msg
        mov edx,len
         int 0x80
        jmp _loop ; _loop'un başına dön
```

Aslında bu program, loop instruction konusunda örnek olarak verdiğim assembly kodunun tam tersi. Loop instruction'u verilen değeri azaltırken burada arttırma yapılarak koşullar ile programı yönetiyoruz. 