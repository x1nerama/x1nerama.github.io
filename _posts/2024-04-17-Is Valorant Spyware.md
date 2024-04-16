---
layout: post
title: Is Valorant Spyware?
description: Valorant oyunun spyware iddiaları hakkında bir analiz yazısı 
published: true 
tags: valorant spyware analysis malware-analysis
---
<style>
    img {
        margin: 3% 0;
    }

    p {
        margin: 2% 0;
    }

    .reddit {
        margin-left: 10%;
    }

    .sup-for-references {
        font-size: 11px;
    }

    .sup-for-references:active ~ .references {
        color: white;
    }  

.image-container {
  position: relative;
  max-width: 500px; /* İhtiyacınıza göre ayarlayabilirsiniz */
  margin: auto;
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: none;
}

.modal {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: white;
  padding: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
  display: none;
}

.modal img {
  max-width: 100%;
  max-height: 80vh;
}

.close {
  position: absolute;
  top: 10px;
  right: 10px;
  font-size: 20px;
  cursor: pointer;
}
</style>

<script>
document.addEventListener("DOMContentLoaded", function() {
  const overlay = document.querySelector(".overlay");
  const modal = document.querySelector(".modal");
  const closeButton = document.querySelector(".close");
  
  overlay.addEventListener("click", closeModal);
  closeButton.addEventListener("click", closeModal);
  
  document.querySelectorAll(".image-container img").forEach(img => {
    img.addEventListener("click", openModal);
  });
  
  function openModal() {
    overlay.style.display = "block";
    modal.style.display = "block";
  }
  
  function closeModal() {
    overlay.style.display = "none";
    modal.style.display = "none";
  }
});
</script>

## Giriş

Valorant oyunu, çıkışından beri hem oyuncuların ilgisini çeken hem de tartışmalara neden olan bir yapım olmuştur. Özellikle oyunun anti-hile sistemi olan Vanguard'ın, oyuncuların gizlilik haklarını ihlal ettiği ve spyware olarak adlandırılabilecek bir yazılım olduğu iddialarıyla sıkça gündeme gelmiştir. Bu iddialar, oyuncuların güvenlik endişelerini artırmış ve Valorant'ın popülaritesiyle birlikte bu konuda birçok tartışma başlatılmıştır.

Reddit ve diğer sosyal medya platformlarında yayılan bu iddialar, Valorant'ın geliştiricisi olan Riot Games'in, oyuncuların bilgisayarlarında istenmeyen izleme ve kontrol yeteneklerine sahip olduğu yönünde eleştirilmesine neden olmuştur. Ancak, bu iddiaların gerçekliği ve Vanguard'ın gerçekten bir spyware olup olmadığı konusu hala netlik kazanmamıştır. Bu yazıda, Valorant'ın spyware olduğu iddialarını ele alacak ve bu iddiaların ne kadar temellendirilmiş olduğunu araştıracağız.

## Uyarı

Bu yazıdaki analiz ve değerlendirmeler, yalnızca araştırma amaçlıdır ve herhangi bir suçlama içermemektedir. Valorant'ın güvenlik önlemleri ve Vanguard anti-hile sistemi hakkındaki iddiaların objektif bir bakış açısıyla incelenmesi amaçlanmıştır. Bu analiz, Riot Games ya da Valorant'ın kullanıcılarına yönelik herhangi bir kötü niyetli davranışı ima etmekten ziyade, topluluğun gündeme getirdiği endişeleri ve eleştirileri anlama ve değerlendirme amacını taşımaktadır. 

## Anti-Cheat Yazılımları nasıl Çalışır?

Araştırmamıza Anti-Cheat yazılımların nasıl çalıştığını anlayarak başlamamız gerekiyor.

Genel olarak Anti-Cheat yazılımları, oyun içerisinde hile yapılmasını engellemek ve hile yapanları tespit etmek amacıyla geliştirilmiş yazılımlardır. Bu yazılımlar, oyunun çalıştığı cihazın kernel seviyesinde çalışarak, oyun için hazırlanan hileleri tespit etmektir.

Anti-Cheat yazılımların çalışma mantığı ikiye ayrılabilir; server-side (sunucu taraflı) ve client-side (istemci taraflı) çalışma yöntemleri. Server-side sadece oyun sunucusu tarafında çalışır ve gelen data paketlerini denetler. Bu paketler, oyuncuların eylemleri ve durumuyla ilgili paketlerdir. Client-side tabanlı anti-cheat yazılımları ise oyuncunun kendi cihazında çalışan ve sunucuya veri gönderen yazılımlardır.<sup class="sup-for-references"><a href="#source-1">[1]</a></sup>

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2f/Priv_rings.svg/1024px-Priv_rings.svg.png">

İlk başlarda Anti-Cheat yazılımları Ring 3 seviyesinde çalışıyordu. Bilindiği gibi ring 3 alanı, **User Mode** olarak geçmektedir. Eğer bu konuya aşina değilseniz hızlı bir şekilde araştırmanızı tavsiye ederim.

Bilindiği üzere User Mode alanında ayrıcalıklar çok kısıtlıdır; işletim sistemi üzerinde her ayrıcalığa sahip olmadığınız ve çok kısıtlı olduğunuz bir alan. Anti-Cheat yazılımları ise bu alanda çalışıyordu. Bu alanda doğrudan donanıma erişim sağlayamadığınız için ve belleğe erişim sağlayamadığınız için anti-cheat yazılımları herhangi bir tarama gerçekleşmeden önce Application Programming Interface (API)'lardan izin alması gerekirdi. Bunlarla beraber bu alanda anti-cheat yazılımları, diğer uygulamara göre izole bir şekilde çalışırdı ve bu yüzden başka bir uygulamaya ait olan datalara müdahale edemez ya da datayı değiştiremezdi.

Fakat Cheater'lar, anti-cheat yazılımların Ring 3 seviyesinde çalıştığını bildikleri için daha sofistike yöntemler geliştirdiler. Yöntemleri ise hazırladıkları hileleri Ring 3 seviyesinde değil Ring 0 alanında çalıştırmaktı yani **Kernel Mode** alanında. Böylece hazırladıkları hileler daha fazla ayrıcalıklara sahip oluyordu ve tespit edilmesi ciddi anlamda zorlaşıyordu. Ring 0 yani Kernel Mode alanı, User Mode alanı gibi kısıtlamalara sahip değildir; bu alana kıyasla daha fazla ayrıcalıklara sahiptir, herhangi bir instruction yürütebilir ve herhangi bir bellek adresini rahatça okuyabilir. Hatta Cheater'lar o kadar ileri seviyeye gitmişler ki, Ring 3 alanında çalışan Anti-Cheat yazılımların data almak için kullandığı system call'ları yani sistem çağrılarını bile hooklayabiliyordı.

Oyun şirketleri ise bu yönteme karşı daha benzer bir sofistike bir yöntem geliştirdiler ve kernel seviyesinde yani Ring 0 alanında çalışan anti-cheat yazılımlar geliştirdiler. Ring 0'da çalışan bu yazılımlar, Ring 3 alanında olduğu gibi sistemi aynı şekilde tarayabiliyor ancak bunu direkt kernel seviyesinde yapıyor yani tamamen yüksek yetkiler ile. Bunun avantajı olduğu gibi de bu blogta baya tartışacağımız dezavantajı da vardır. Anti-Cheat yazılımların Ring 0 alanında çalışma avantajı, tespit edilemesi zor olan hileleri bile kolayca tespit etmesi ve oyunu hileye karşı daha güvenli tutmasıdır. Dezavantajı ise **gizlilik problemi**.

Düşünün ki, eğer bu yazılımlar sizi hilelerden korumak için yüksek yetkiler ile tarama yapıp ve her şeyi okuyabiliyorsa teorik olarak bilgisayarınızdaki her şeye de erişebilir anlamına da geliyor. Bu yazılımlar kernel seviyesinde çalışıyor. Yani yüksek ayrıcalıklarla sizin bilgisayarınızdaki anlık her şeyi okuyabilir ve erişebilirler. Sizlere şu soruyu yönlendirmek isterim, oynadığınız oyunda hilecilere katlanmak istemezsiniz ve bu hilecilere karşı bir önlem alınmasını istersiniz ama bununla beraber gizliliğinizi riske atmak ister miydiniz?

## Anti-Cheat yazılımların getirdiği Potansiyel Tehlikeler

Bende dahil oyunlarda neredeyse hepimiz hilecilere karşı bir önlemler alınmasını isteriz. Buna karşın oyun şirketleri de anti-cheat yazılımlarını bizlere sunar ancak bu yazılımlar isteğimizi karşılasa da yanında getirdikleri ciddi sorunlar var; bunlardan biri de yukarıda ele aldığım gizlilik sorunu.

Geçmiş yıllarda anti-cheat yazılımların gizlilik ilkelerini ihlal eden olaylar olmuştur. Bunlardan biri de 2yılında ESEA şirketinden bir geliştiricinin işletim sisteminin kernel seviyesini kullanarak oyuncuların bilgisiyarından gizli ve büyük ölçekte bir bitcoin madenciliği gerçekleştirmesi bu konuya güzel örnektir. Geliştirici, <b>14.000</b> oyuncunun bilgisiyarındaki GPU ile yerleştirdiği mining yazılımıyla yaklaşık 4.000$ elde etti<sup class="sup-for-references"><a href="#source-4">[4]</a></sup> ve bu olay ortaya çıktıktan sonra ESEA şirketi US Regulators (ABD Düzenleyicileri) tarafından <b>1.000.000</b> <a href="https://nj.gov/oag/newsreleases13/pr20131119a.html">para cezası verildi</a>. Bu olaylardan sonra ESEA şirketi bizzat kamuoyundan <a href="https://play.esea.net/news/12692">özür diledi</a> ve ESEA şirketi mining ile elde edilen paraları ödül potları aracılığıyla oyunculara dağıttı ve American Cancer Society (Amerikan Kanser Derneği)'ne <b>$7.427.10</b> bağışladı.<sup class="sup-for-references"><a href="#source-4">[4]</a></sup><sup class="sup-for-references"><a href="#source-5">[5]</a></sup>


Bu yazılımlar tamamen yüksek ve ayrıcalıklı yetkiler ise oyuncunun bilgisayarını tarayabilir ve her dataya erişebilir. Bu da sistemde genel bir istikrarsızlığa neden olabileciği gibi gizlilik problemine de neden olur. 

Bununla beraber bazı hackerların oyuncuların bilgisayarına girmesine neden olabilecek güvenlik açıklara ya da İşletim sistemi için problemlere neden olabilirler. Anti-Cheat'lar bir driver yani sürücü olarak çalıştığı için sürücü kodundaki küçük bir hata, işletim sistemin <b>Blue Screen of Death (BSOD)</b> benzeri çökmelere neden olurken, sürücü kodunda bulunan ciddi ihmaller veya eksiklikler ise bufferoverflow gibi açıklara neden olabilir.

<b>"Ulan beko alt tarafı oyunun güvenliğini sağlıyorlar ne abarttın be güvenlik açığı bilmem ne diye"</b> dediğinizi duyar gibiyim. Gelin reddit'ten bu sorunlara maruz kalmış insanların konularına bir göz atalım:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/r-com1.png?raw=true" class="reddit" style="width: 80%;">

Burada kullanıcı bilgisiyarı açtığı anda yukarıda bahsettiğim **SYSTEM THREAD EXCEPTION NOT HANDLED** hata koduyla mavi hata ekrana düştüğünü belirtiyor. Çözüm üretmek için Windows'a girmeye ya da windows'u güvenli modda çalıştırmayı denediğini ancak başaramadığını belirtiyor. Son olarak ise format atarak sorunu çözdüğünü belirtiyor.

**vgk.sys**, RIOT Games'a ait olan bir Vanguard anti-cheat yazılımın driver yani sürücü dosyasıdır.

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/r-com2.png?raw=true" class="reddit" style="width: 80%;">

Burada ise kullanıcı Counter Strike 2 oyununu oynadığı sürece yine aynı hata koduyla mavi ekrana düştüğünü ve çözüm aradığını belirtiyor. Burada tam olarak anti-cheat yüzünden olduğunu söylemek biraz güç gibi gözüküyor ancak yine de paylaşmak istedim. Ama bu hata nedeninin sürücü kodundan olduğu açıkça belli.

Peki neden sürücüler mavi ekran gibi hatalara neden olur? Anti-Cheat yazılımın kernel seviyesinde çalıştığını birçok defa tekrarladım. Kernel seviyesinde çalışan sürücülerde gerçekleşen **en ufacık** bir hata işletim sistemin çökmesine neden olur. Çünkü işletim sistemin en derinindesiniz. Ring 0 yani kernel mode alanı hatalara karşı çok hassastır. O yüzden kernel seviyesinde çalışıyorsanız çok dikkatli olmanız da gerekir. Oyun şirketleri ise bunları dikkate alarak anti-cheat yazılımları geliştirmeli. 

## Neden Valorant için Spyware iddiaları bulunmakta?

Valorant için bu iddiaların bulunmasındaki en büyük sebeplerden biri de Çin ülkesine ait olan Tencent şirketinin RIOT Games'ın %100'üne sahip olması, bu iddialara öncülük etmektedir. Tencent şirketi, Şubat 2011'de Riot Games'in yüzde 93 hissesi için 400 milyon yatırım yapmıştı ve 16 Aralık 2015'te kalan yüzde 7 hisse için de belirtilmeyen fiyat ile yatırım yaparak %100 hisseyi almıştır.<sup class="sup-for-references"><a href="#source-3">[3]</a></sup>

Tencent şirketi, Çin hükümeti ile yakın ilişkileri olan bir şirkettir ve Çin hükümeti, internet üzerindeki tüm faaliyetleri kontrol etmek istemektedir. Bu durum, Tencent'in RIOT Games'ın %100'üne sahip olması ve Valorant'ın anti-cheat yazılımının kernel seviyesinde çalışması, bu yazılımın Çin hükümeti tarafından kullanılabileceği iddialarına neden olmuştur. 

Bu tartışmaların başlangıcını 3-4 yıl öncesinde başladığını görebiliriz. Mesela <a href="https://www.reddit.com/r/privacy/comments/kz872x/is_valorant_malwarespyware/?rdt=41178">r/privacy</a> subreddit'inde denk geldiğim şu konuya bir göz atabiliriz:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/r-com3.png?raw=true" class="reddit" style="width: 80%;">

İçerikte, Valorant'ın anti-cheat yazılımı olan Vanguard'ın tehlikeli olup olmayacağına dair bir soru sorulmuş. Yine içerikte bu yazılımın arka tarafta sürekli çalıştığı söyleniyor. Peki gerçekten öyle mi?

Bu soruyu araştırmaya öncelikle RIOT'un kendi makalesi olan <a href="https://support-valorant.riotgames.com/hc/tr/articles/360046160933-Vanguard-nedir">'Vanguard Nedir'</a> ile başlamaya karar verdim. Makalenin başında da kendileri de Vanguad'ın kernel modu sürücüsü olarak çalıştıklarını söylüyorlar. Ancak makalenin şu kısmı çok dikkatimi çekti:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/vanguard-article.png?raw=true">

Açıkçası bu metinler beni tatmin etmedi. Evet, RIOT makalesinde Vanguard'ın zaten bilgisiyar açılışından itibaren çalıştığını açıkça söylüyor ancak bana soracak olursanız açıklama yetersiz. <b>Gizlilik</b> gibi böyle bir konuyu detaylandırarak oyunculara açıklamak ve güvenini kazanmak yerine böyle kısa açıklama ile konuyu kapatmak sizce de mantıklı mı? 

Bu makale bana yetersiz geldiği için tekrardan reddit ortamlarına döndüm ve bu konuda tartışan insanların yorumlarına göz atmaya başladım. Daha sonra r/VALORANT subreddit'inde şu konuya denk geldim:

<blockquote class="reddit-embed-bq" style="height:316px" data-embed-theme="dark" data-embed-height="612"><a href="https://www.reddit.com/r/VALORANT/comments/fzxdl7/anticheat_starts_upon_computer_boot/">Anticheat starts upon computer boot</a><br> by<a href="https://www.reddit.com/user/DolphinWhacker/">u/DolphinWhacker</a> in<a href="https://www.reddit.com/r/VALORANT/">VALORANT</a></blockquote><script async="" src="https://embed.reddit.com/widgets.js" charset="UTF-8"></script>

Gerçekten r/VALORANT subreddit'inde baya tartışılmış bir konu.

Tartışmayı başlatan kişi, vgk.sys'in bilgisiyar başlangıcından itibaren çalıştığını ve bunu umursamasa da bunun nedenini soruyor ve gerçekten güzel bir soru.

Daha sonra VANGUARD anti-cheat yazılımın lideri olan u/RiotArkem uzun bir cevabına denk geldim:

<blockquote class="reddit-embed-bq" data-embed-theme="dark" data-embed-height="876"><a href="https://www.reddit.com/r/VALORANT/comments/fzxdl7/comment/fn6yqbe/">Comment</a><br> by<a href="https://www.reddit.com/user/DolphinWhacker/">u/DolphinWhacker</a> from discussion<a href="https://www.reddit.com/r/VALORANT/comments/fzxdl7/anticheat_starts_upon_computer_boot/"><no value=""></no></a><br> in<a href="https://www.reddit.com/r/VALORANT/">VALORANT</a></blockquote><script async="" src="https://embed.reddit.com/widgets.js" charset="UTF-8"></script>

Bu açıklama, Vanguard üzerine okuduğum makaleye kıyasla daha açıklayıcıydı. Gelin bir detaylı göz atalım!

vgk.sys sürücüsünün gerçekten bilgisiyar başlangıcında çalıştığını söylüyor ancak oyun çalışmadığı sürece hiçbir şeyin taramadığını, sunucularla iletişim kurmadığını ve mümkün olduğunca az sistem kaynağı kullanarak çalıştığını söylüyor ve bu yazılımın istendiği zaman kaldırılabileceği söyleniyor.

Neden başlangıçta çalıştığını ise sistem başlangıcında yüklenmediği sürece bilgisiyarı güvenilir olarak kabul etmediği için ve em önemlisi de anti-cheat yazılımları bypass etmenin yollarından biri olan anti-cheat yazılımların yüklenmesinden önce hemen hilenin yüklenmesi ya da sistem bileşenlerini değiştirerek hileyi eklemek gibi bypass yöntemlerine çözüm olabileceği söylenmiş.

Ayrıca bu sürücünün güvenliğine ve ayrıcalıklarına da değinmiş. Güvenlik olarak güvenlik araştırma ekiplerine incelettiklerini, sürücünün mümkün olduğunda az şeyler yaptığını ve sürücüye en az ayrıcalıklar verdiklerini söylemiş

Son olarak bizim için önemli açıklama ise bu yazılımın hiçbir şekilde oyuncunun bilgisiyar hakkında bir bilgi toplamadığı veya sunuculara göndermediği belirtlmiş ve sadece oyun çalıştığında etkin olabileceği söylenmiş. <br/> <br/>

Açıklamaya göz attığımızda orijinal makaleye kıyasla daha açıklayıcı ve güzel duruyor. Şimdi ise bu topladığımız bilgileri analize dökerek bir kontrol edelim. 

## Analizin gerçekleşmesi

Şimdi temel bilgilerden sonra artık basitçe neler yapabileceğimizi çözdük ve artık yavaştan analize başlayabiliriz.

Öncelikle analize wireshark ile başlamak istedim. RIOT Client uygulaması başlatıldığında nerelere bağlantı kurduğunu görmek istedim ve sonuç korkutucuydu:

<video controls>
    <source src="https://github.com/x1nerama/x1nerama.github.io/raw/847f438aba42f3b2508c2803be731d5709f0ef3b/video/videos-for-valorant-topic/wireshark-video.mp4" type="video/mp4">
</video>

Göründüğü gibi Client uygulaması çalıştırıldığında düşünüldüğünden daha fazla birçok yere veri gönderiliyor.

Burada gönderilen verilerin içeriğini görmek pek mümkün olmayacaktır çünkü videoda görülebileceği gibi veriler şifrelenmiş halde. Dolayasıyla bunun izinden gitmemiz pek mümkün olmayacaktır. Bu yüzden bende bireysel bağlantıların adreslerinden bir kaçına göz atmaya karar verdim:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/tcpview-for-riotclient.png?raw=true" style="margin-bottom: 0">
<div class="image-container"> 
    <div class="overlay"></div>
    <div class="modal">
        <span class="close">&times;</span>
        <img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/tcpview-for-riotclient.png?raw=true">
    </div>
    <div align="center">
        <h4 style="text-align:center;"> Fotoğrafı yakından görmek için üzerine tıklayın. </h4>
    </div>  
</div>  

Verilerin nereye gönderildiğine dair bir iz sürmeye çalıştığımda verilerin nereye gönderildiğine dair bir sonuca varamadım ancak <a href="https://www.youtube.com/@pcsecuritychannel">The PC Security Channel</a> adlı kanalın <a href="https://www.youtube.com/watch?v=UqLI1xKc-L4">'Is Valorant Spyware?'</a> videosunda analizini izlediğimde kendisinin birçok IP adresinin Amazon sunucularına ait olduğunu belirtiyor. Ayrıca bu konuyu hazırlarken ilham aldığım bahsi geçen videoya da göz atabilirsiniz. Gerçekten güzel ve açıklayıcı bir analiz gerçekleştiriyor.



Bağlantı sayılarına buradan da göz attığımızda çok kadar fazla bağlantı olduğunu görebiliyoruz ve maalesef bu iç açıcı bir şey değil. Kendi kendime bunun abarttığımı düşündüm ve bilgisiyarımda yüklü olan Epic Games uygulaması için de kontrol etmek istedim ve sonuç:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/tcpview-for-epicgames.png?raw=true" style="margin-bottom: 0">
<div class="image-container"> 
    <div class="overlay"></div>
    <div class="modal">
        <span class="close">&times;</span>
        <img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/tcpview-for-epicgames.png?raw=true">
    </div>
    <div align="center">
        <h4 style="text-align:center;"> Fotoğrafı yakından görmek için üzerine tıklayın. </h4>
    </div>  
</div>

Göründüğü gibi RIOT Client uygulamasına kıyasla daha az bir bağlantı var. Yani RIOT Client uygulamasının cidden fazla veri gönderimi yaptığını anlayabiliyoruz. <br/> <br/>

Daha sonra yönümü .sys dosyasına çevirdim ve <b>Process Explorer</b> aracılığıyla vgk.sys'e kısaca bakmak istedim:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/sys-in-pe.png?raw=true" style="margin-bottom: 0">
<div class="image-container"> 
    <div class="overlay"></div>
    <div class="modal">
        <span class="close">&times;</span>
        <img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/sys-in-pe.png?raw=true">
    </div>
    <div align="center">
        <h4 style="text-align:center;"> Fotoğrafı yakından görmek için üzerine tıklayın. </h4>
    </div>  
</div>

Aynı zamanda vgk.sys'in durumunu **driverquery** aracı ile daha hızlı kontrol edebiliriz:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/driverquery-result.png?raw=true" style="margin-bottom: 0">
<div class="image-container"> 
    <div class="overlay"></div>
    <div class="modal">
        <span class="close">&times;</span>
        <img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/driverquery-result.png?raw=true">
    </div>
    <div align="center">
        <h4 style="text-align:center;"> Fotoğrafı yakından görmek için üzerine tıklayın. </h4>
    </div>  
</div>

Fakat bir sorun var. Şuan işletim sisteminde RIOT'un herhangi bir uygulaması çalışmıyor - arka planda bile -. Bu, Riot Vanguard için de geçerli:

<video controls>
    <source src="https://github.com/x1nerama/x1nerama.github.io/raw/main/video/videos-for-valorant-topic/process-explorer.mp4" type="video/mp4">
</video>

İşletim sistemimde RIOT'un tüm uygulamaları başlangıçta başlaması kapalı. Ancak yine de vgk.sys'in çalıştığını tespit ettim. Şimdi ise akıllara şu soru takılıyor, **"Sadece oyunlarda hileleri engelleyen bir yazılım, ilgili oyunun kapalı olmasına rağmen neden arka planda çalışıyor?"**.

RIOT'un makalesinde zaten arka planda çalıştığını belirtse de RIOT ile ilgili tüm uygulamaların kapalı olmasına rağmen bu kernel sürücüsünün yine arka planda çalışması çok olanaksız. Oyun tamamen kapalı ise oyunlarda hileden koruyan bir yazılım beni neyden koruyabilir? <br/> <br/>

Araştırmalara, yukarıda Vanguard'ın eski lideri Arkhem'in bahsettiği şu kısmı ele alara devam etmek istiyorum:

> Yes we run a driver at system startup, it doesn't scan anything (unless the game is running), it's designed to take up as few system resources as possible and it doesn't communicate to our servers. You can remove it at anytime.

Yazılımın istendiği zaman kaldırabileceği belirtilmiş. Bunu tekrar okuduktan sonra Vanguard'ı kaldırdım ardından işletim sistemini yeniden başlattım ve vgk.sys'in yine sistemde olup olmadığını kontrol ettim ve sonuç:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/driverquery-result-2.png?raw=true" style="margin-bottom: 0">
<div class="image-container"> 
    <div class="overlay"></div>
    <div class="modal">
        <span class="close">&times;</span>
        <img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/driverquery-result-2.png?raw=true">
    </div>
</div>

Gerçekten de Vanguard yazılımı kaldırıldığında vgk.sys sistemden kaldırılıyor. Bunu doğrulamış olduk. <br/> <br/>

## vgk.sys Sürücüsünü Analiz Etme

vgk.sys sürücüsünü yakından analiz etmeye başlayacaktım ancak maalesef bunun izinden süremedim. Çünkü RIOT hiçbir şekilde vgk.sys'i analiz etmemize fırsat vermediğini öğrendim.

İlk başta VALORANT oyununu sanal makineye kurdum ve sistemi yeniden başlattıktan sonra VANGUARD yazılımın başlatılmadığını fark ettim. Hatalardan olabileceğini düşünerek çeşitli yollar denedim ancak olmadı:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/valorant-in-vm.png?raw=true" style="margin-bottom: 0">
<div class="image-container"> 
    <div class="overlay"></div>
    <div class="modal">
        <span class="close">&times;</span>
        <img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/valorant-in-vm.png?raw=true">
    </div>
</div>

Sanal makinede valorant çalıştırmak istediğimde karşıma çıkan ekran buydu. Bunun sebebi ise vgk.sys sürücüsünün başlatılmaması. 

Sanal makinelerde hiçbir şekilde vgk.sys sürücüsünü başlatılmıyor ve analizde fark ettiğim bir diğer şey ise sanal makine olmasa bile eğer işletim sisteminizde debugging aktifse vgk.sys çalıştırılmıyor. Yani ana makinenizde çalışan vgk.sys dosyasını analiz etmek için debugging aktifleştirseniz bile kendini devre dışı bırakacaktır. <br/> <br/>

Dinamik analizine gidemediğim için statik analize yönelmeye karar verdim ve .sys dosyasının kullandığı fonksiyonlara göz atmak istedim:

<img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/used-function.png?raw=true" style="margin-bottom: 0">
<div class="image-container"> 
    <div class="overlay"></div>
    <div class="modal">
        <span class="close">&times;</span>
        <img src="https://github.com/x1nerama/x1nerama.github.io/blob/main/images/photos-for-is-valorant-malware/used-function.png?raw=true">
    </div>
</div>

API'lara göz attığımızda vgk.sys sürücüsü oyuncunun bilgisiyarın çalışma ortamını kontrol edebilecek ve sistem saat ve sistem dizinini elde edebilecek API'lar kullandığı göze çarpmaktadır. Tabi ki dinamik analiz gerçekleştiremediğimiz bunların ne amaçla kullandığını da bilemeyiz. 

Eğer fonksiyonların tamamına siz de göz atmak isterseniz aşağıya listenin tamamını ekliyorum:

```
ZwClose	
KeInitializeSpinLock	
KeAcquireSpinLockAtDpcLevel	
KeAcquireSpinLockRaiseToDpc	
KeReleaseSpinLock	
KeReleaseSpinLockFromDpcLevel	
ExAllocatePoolWithTag	
KeLowerIrql	
KfRaiseIrql	
KeInitializeDpc	
KeInitializeTimer	
KeSetTimer	
MmMapLockedPagesSpecifyCache	
MmUnmapLockedPages	
MmAllocatePagesForMdl	
MmFreePagesFromMdl	
IoFreeMdl	
IoAllocateWorkItem	
IoQueueWorkItem	
IoInitializeWorkItem	
RtlDuplicateUnicodeString	
ObfDereferenceObject	
KeBugCheckEx	
_stricmp	
__C_specific_handler	
KeIpiGenericCall	
ExFreePoolWithTag	
ProbeForRead	
IoGetCurrentProcess	
wcscpy_s	
RtlInitUnicodeString	
RtlTimeToTimeFields	
KeAreAllApcsDisabled	
ExSystemTimeToLocalTime	
ZwWriteFile	
IoCreateFileEx	
ZwFlushBuffersFile	
swprintf_s	
vswprintf_s	
_vsnwprintf	
KeInitializeApc	
KeInsertQueueApc	
wcscat_s	
ZwReadFile	
ZwQuerySystemInformation	
IoGetStackLimits	
strchr	
RtlPrefixUnicodeString	
RtlMultiByteToUnicodeN	
MmHighestUserAddress	
ObReferenceObjectByHandle	
IoFileObjectType	
strnlen	
```
Dediğim gibi dinamik olarak analiz yapamadığımız için burada kullanılan fonksiyonların ne işe yaradığını söylemek pek mümkün olmayacaktır.

# Sonuç

Analiz sonucunda, Vanguard'ın sürücüsü olan vgk.sys'in davranışlarını değerlendirdiğinde, Valorant oyununun spyware olabileceği iddialarının daha yakın olduğu görülmektedir. Ancak bu iddiaların doğru olup olmadığını kesin olarak söylemek için daha fazla analiz yapılması gerekmektedir.


 <video preload="auto" poster="https://pbs.twimg.com/tweet_video_thumb/D5aj3tfW0AIiSxo.jpg" src="https://video.twimg.com/tweet_video/D5aj3tfW0AIiSxo.mp4" type="video/mp4" autoplay="" controls=""></video>

# References

- <a href="https://helda.helsinki.fi/server/api/core/bitstreams/89d7c14b-58e0-441f-a0de-862254f95551/content" id="source-1"><b>[1] - University of HELSINKI: Comparative Study of Anti-cheat Methods in Video Games </b></a>
- <a href="https://www.schellman.com/blog/cybersecurity/what-is-anti-cheat" id="source-2"><b>[2] - Schellman: Understanding Anti-Cheat</b></a>
- <a href="https://en.wikipedia.org/wiki/Riot_Games#History" id="source-3"><b>[3] - EN Wikipedia: Riot Games </b></a>
- <a href="https://www.theregister.com/2013/11/20/esea_gaming_bitcoin_fine/" id="source-4"><b>[4] - TheRegister: Gaming co ESEA hit by $1m fine for hidden Bitcoin mining enslaver</b></a>
- <a href="https://www.engadget.com/valorant-vanguard-riot-games-security-interview-video-170025435.html" id="source-5"><b>[5] - Engadget: A closer look at Valorant's always-on anti-cheat system</b></a>