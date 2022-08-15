
# Senai Kalafat - RabbitMQ QoS Konfigürasyonu Üzerine Kısa Bir Vaka Analizi

**RabbitMQ QoS Konfigürasyonu Üzerine Kısa Bir Vaka Analizi**

Mikroservis mimarilerinin hızla yaygınlaştığı son yıllarda bu sürecin parlayan yıldızlarından biri elbette RabbitMQ oldu. RabbitMQ "Advanced Message Queuing Protocol" standardını temel alan bir mesaj dağıtım uygulamasıdır.

Uygulamanızda kullanacağınız mesajlar için özel alanlar belirlemenizi, bu alanlara mesaj yazma ve okuma yetkisi olan clientların erişmesini ve bu mesajların sıra ve iletim garantisi gibi konfigürasyon ve yönetimlerin yapılabilmesini sağlar.

RabbitMQ, Exchange mekanizması, pluginleri, yetenekleri, hızı ile kendini ispat etmiş kapsamlı bir message bus uygulama altyapısıdır.

Teknik detaylarını gerek kendi sitesinden gerek kullanım tecrübelerinin paylaşımında öğrenerek ilerleyebilirsiniz.

Bizim konumuz ise çok basit olmakla birlikte bir mikroservis mimarisinde doğru kurgulanmamış bir kuyruk mekanizmasının performansa olan etkilerini bir durum senaryosu üzerinde inecelemek olacak.

Mümkün olduğunca basit ve teknik detaylardan uzak bir şekilde, diğer mesajlaşma altyapılarında da karşılaşılabilecek bu senaryoyu konuşacağız.

**Mevcut Durum;**

RabbitMQ üzerinde bir kuyruğumuz var, bu kuyruğun adı HardTaskEventQueue. Adından da anlaşılacağı gibi zorlu görevlerin dağıtım yeri :D.

Bu kuyruğa 5 adet producer servis, işlenmesi gereken HardTask eventlerini koyuyorlar.

Yine bu kuyruktan bu HardTask eventlerini işlemekle sorumlu 5 adet Consumer servisimiz var.

Basitçe aşağıdaki şekilde bir yapımız olduğunu düşünelim.

![](RackMultipart20220815-1-c2gibz_html_4654e53d43b36f94.png)

Beklenen akış aşağıdaki gibi olmalıdır.

1. Producer rolündeki servisler ihtiyaçları olan ağır iş yükünü ilgili kuyruğa gönderirler.

2. Kuyrukta kayıt altına alınan ilgili taskler, bu kuyruğu dinleyen consumer servisler tarafından alınıp işleme sokulur.

3. Sonuçlar uygulamanın akışına göre ilgili yerlere gönderilir.

Birçok senaryo için bu durum temel ve varsayılan konfigürasyon ile iş görecektir.

Aşağıdaki gibi basit dengeli bir şekilde mesaj kuyruğunda biriken mesajların işlendiğini göreceksiniz. Dengeli kelimesi çok önemli tüm servis kopyalara aşağı yukarı aynı miktarda mesaj sorumluluğu alarak süreci yürütüyorlar.

![](RackMultipart20220815-1-c2gibz_html_cc8f3b1fedb099f5.png)

**Durum Senaryosu**

Ortamımızı biraz güncelleyelim ve birkaç spesifik detay ekleyelim.

![](RackMultipart20220815-1-c2gibz_html_28638d0b29b38c0a.png)

Yukarıda gördüğünüz gibi consumer servisler farklı farklı ağ ortamlarında olabilir(Farklı VLAN, Azure, AWS, DigitalOceon vs). Bu bize dağıtık mimarinin getirdiği ve mikroservislerle nimetlerinden sonuna kadar faydalandığımız çok önemli bir özellik.

Fakat ortamın farklı ağ gecikmeleri ağ performansları olacaktır. Yine her uygulamanın koştuğu ortamın donanımsal yeteneklerinin aynı olması beklenemez.

Peki bu sorun mu? Sorun demek ne kadar doğrudur bilmiyorum ama yönetilmesi gereken bir durum olduğu kesin.

Çünkü aşağıdaki gibi bir sonuç oluşabilir.

![](RackMultipart20220815-1-c2gibz_html_28d639310b084b4f.png)Aynı VLAN üzerindeki makina Aç Gözlü davranarak tüm mesajları alma eğilimindedir.

Görüldüğü üzere servisler kendi ortam ve donanım performanslarına göre kuyruktan "Aç Gözlüce" task almaya başlayabilirler. Fakat kendileri bile bunu işleyip işleyemeceğinin farkında olmayabilir. En kolay ifadeyle aç gözlüce işleyeceğini taahhüt ederek aldığı bu mesajları geciktirebilir, hatta zaman aşımına uğratabilir. Eğer ilgili servis böyle bir yoğunluğu yönetmek üzere iyi kurgulanmamışsa servisin durmasına ve yeniden başlamasına sebep olabilir.

İstenmeyecek bu durumları yönetmek zordur. Ve karşılaşıldığında ilk önce ne olduğu tam anlaşılamayabilir. Ve direk Consumer servis sayısının arttırılmasına gidilir. Fakat Aç Gözlü servislerin durumu çözülmediği sürece yeni eklenen servislerden istenen verim alamaz.

**Peki ne yapılabilir?**

Yapılacak şey aslında çok kolay, dokümantasyonu okumak. Evet sizin de tahmin edebileceğiniz gibi aslında bu durumlar sadece sizin başınıza gelmiyor. Uygulamalar geliştirilirken bu süreçler yaşanıyor ve ona ilişkin çözümler üretiliyor. Ama genellikle çok fazla dikkat çekici durmadığı için atlanıyor.

Yine Teknik olarak fazla detaya girmeden size bunun en basit tabirle nasıl yönetildiğini anlatmaya çalışacağım.

Örneğimizi bir mesajın sadece bir consumer servisi tarafından işlenebileceği konfigürasyon üzerinden verelim. Böyle bir senaryoda her bir consumer bir mesaj aldığında bu mesajı işlemeye başladığına, iletildiğine ve işlendiğine dair mesajlar gönderir. Mesajı aldığında kuyruk sistemi mesajı başka bir servise iletmez. Sadece ilgili mesajı alan servisten bu mesajı işlediğine ilişkin bir ACK mesajı ister. Consumer servis içerisinde süreci tamamlanmamış tüm mesajlar RabbitMq için beklemede olan ve süreci devam eden mesaj durumundadır. (unacknowledged messages)

Geliştiriciler bunu basitçe her bir consumer için üzerine max kaç adet durumu bildirilmeyen (unacknowledged messages) mesaj alabilir limiti belirleyerek çözmüşlerdir. Amaç aç gözlülüğün önüne geçerek aç gözlü servisin çalışma stabilitesini artırmak.

Örnek vermek gerekirse diyelim ki her bir kuyruğa max 10 adet durumunu bildirmediğin mesaj alabilirsin limiti konulursa, network ne kadar hızlı olursa olsun 10'dan fazla mesaj alamayacaktır. Yeni bir mesaj alabilmesi için üzerine aldığı mesajlardan en az birini işleyerek RabbitMQ'ya bildirmesi gerekecektir.

![](RackMultipart20220815-1-c2gibz_html_bdca09f60c820a34.png)QoS Parametreleri Set Edilmiş bir kuyruk sistemi (Aynı anda cevabı dönülmemiş Max 10 mesaj sorumluluğu alabilir)

Yukarıda gördüğünüz gibi sisteme kısmi denge gelmiştir. Peki bu yeterli midir? Elbette daha birçok optimizasyon ile sistemin yük dengelenmesi daha performanslı hale getirilebilir.

Yine sisteminizin ihtiyacına göre kuyruktan alınacak max mesaj sayısına da sınırlama getirebilirsiniz. (Consumer seviyesinde ve Kanal Seviyesinde)

Uygulamanız ayağa kalkerken kendi iç denetimleri ile performans değerlendirmesi yapıp uygulamanın bu limitleri dinamik olarak belirlemesini sağlayabilirsiniz.

**Sonuç**

Yazılım geliştirici arkadaşlar çoğu zaman hızlı çözümler üretmek zorunda kalabilirler ve bu çözümler zamanla sistem yükünü taşıyamayabilir. Burada gösterildiği gibi aslında süreci takip etmek, kriz anlarında anlık olarak durumu kavrayabilmek zor olabiliyor.

Özellikle sisteminizin Veri tabanları ve Altyapı bileşenleri gibi kritik parçalarında bu tarz optimizasyon ve QoS (Quality of Service) detaylarına dikkat etmenizde fayda vardır.

Aşağıda RabbitMQ için ilgili dökümantasyona erişebilirsiniz.

[ **Consumer Prefetch**
_Consumer prefetch is an extension to the channel prefetch mechanism. AMQP 0-9-1 specifies the basic.qos method to make…_www.rabbitmq.com](https://www.rabbitmq.com/consumer-prefetch.html)

Senai Kalafat
