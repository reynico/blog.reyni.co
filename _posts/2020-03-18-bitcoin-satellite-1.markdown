---
layout: post
title:  "Configurando una estación satelital para Bitcoin"
date:   2020-03-18 20:18:27 -0300
categories: [bitcoin, radio]
---
Hace un tiempo vi el repo [Blockstream satellite](https://github.com/Blockstream/satellite) en Github. Parece que [Blockstream](https://blockstream.com/satellite/) está broadcasteando la blockchain de Bitcoin a una serie de satélites geoestacionarios alrededor del planeta tierra, 24/7, gratis. La intención es tener una blockchain de Bitcoin completamente sincronizada sin necesidad de una conexión a internet. Suena complicado eh?

### Día 0: Idea
Tengo algo de background en comunicaciones de radio y satélites. Tengo funcionando hace un tiempo [una estación automatizada](https://twitter.com/argentinasat) que descargar imagenes de satélites del clima y las postea en Twitter.

### Día 1: Hardware y antenas
Estoy hace unos días en mi ciudad natal y debido a COVID-19, voy a estar aquí un buen rato asi puedo pasar algo de tiempo con mi familia. También para invertir algo de tiempo en jugar con antenas. Mi padre es [radioaficionado](https://www.qrz.com/db/LU3DJ) desde hace algunas décadas, asi que toda cuestión de radio está bastante bien cubierta.

El readme del proyecto [Blockstream satellite](https://github.com/Blockstream/satellite) está bastante bien detallado en como configurar una estación terrestre, pero les voy a dejar mi experiencia por escrito, probablemente sea mas cómodo (y en español).

# Antena
Para este proyecto estoy usando un plato de 90cm de DirecTV que estaba en desuso, junto con un LNB barato de eBay

![dish antenna](/assets/images/blockstream/antenna.jpg)

Lo primero que hay que hacer es apuntar al satélite. Dada mi ubicación, el que cubre el país es el [Eutelsat 113w](https://www.eutelsat.com/en/satellites/eutelsat-113-west.html). El satélite transmite en banda Ku y banda C, siendo banda Ku la elegida para comunicar ya que es mas fácil y no requiere antenas tan grandes como las de banda C.

![dish pointing data](/assets/images/blockstream/eutelsat113w-dishpointing.png)

Utilizando un poco de sentido común y una brújula se puede hacer el apuntamiento "grueso" de la antena, para luego ir por lo fino y encontrar el satélite correcto. Tengan en cuenta que allí arriba hay muchísimos satélites dando vueltas y emitiendo señales.

![satfinder](/assets/images/blockstream/satfinder.jpg)

Aprovechando que el Eutelsat 113w tiene [algunos canales de FTA (televisión)](http://www.eutelsat.com/deploy_tvLineUp/struts/advancedSearch.do?orbitalPositionId=113%B0%20WEST&Langue=EN) a mi padre se le ocurrió que sería buena idea buscar los canales de TV del satélite para encontrarlo mas rápido, junto con el buen [Satfinder](https://articulo.mercadolibre.com.ar/MLA-812217255-buscador-localizador-satfinder-satelital-4-leds-brujula-cable-de-conexion-_JM). Si bien el Satfinder no puede discriminar entre portadoras de satélites, si sabe cuando hay una señal satelital, para el no tan grueso pero no tan fino, es una gran herramienta. Luego de un par de horas logramos encontrar las portadoras del Eutelsat 113w. Nos dimos cuenta de esto basados en las protadoras de FTA que tenía, que concordaban con las que Eutelsat entrega en su sitio web.

![transponders](/assets/images/blockstream/eutelsat-transponder.jpg){:class="img-responsive"} 

Ya apuntado conecté el LNB al power inserter y de ahí al receptor SDR. y NADA. Investigando un poco, midiendo y midiendo me di cuenta que el power inserter no estaba haciendo su labor. Ya agotado cerca de las 3 AM se me ocurre que podría utilizar un decodificador de TDA para alimentar el LNB, ya que estos decodificadores tienen una conexión loop para conectar mas receptores en cascada haciendo que el primero se encargue de la alimentación del LNB

![power inserter](/assets/images/blockstream/power-inserter.jpg){:class="img-responsive"} 

Así que perdido por perdido, conecté el decodificador, sintonicé polarización horizontal y.. Bingo! Recepción y portadoras en el SDR

![portadoras](/assets/images/blockstream/portadoras-sdr.jpg){:class="img-responsive"} 

### Día 2: Software y señales de radio
Ya con la recepción en la palma de la mano sentí que había avanzado un montón así que el próximo paso era empezar con la parte de software. El proyecto [Blockstream satellite](https://github.com/Blockstream/satellite) ya incluye todo el software necesario y al momento de escribir esta guía la decodificación se hace con bloques de GNURadio. También tiene un par de herramientas como la de apuntamiento de antena (siguiente imagen) que nos permite encontrar la portadora correspondiente y ajustar la posición de la antena para ganar Signal Strength (mas es mejor).

![scan mode](/assets/images/blockstream/recepcion-scanmode-1.jpg){:class="img-responsive"} 

Una vez hecho eso decidí empezar a ejecutar el archivo de GNURadio que tiene toda la implementación dentro (muy buen trabajo!) y algunos objetos visuales (QT GUI) para poder ver la información de manera gráfica además de la consola de datos. En la ventana de GNURadio se puede observar el espectro con su respectivo roll-off, la constelación BPSK y un par de indicadores de señal y señal sobre ruido. Sobre GNURadio se encuentra la consola que nos informa que está bajando bloques a una velocidad de 13.5kB/sec.

![GNURadio](/assets/images/blockstream/recepcion-gnuradio.jpg){:class="img-responsive"} 


Con un poco de lluvia de por medio durante las pruebas empecé a sincronizar una Blockchain completa. La lluvia me jugó un poco en contra, ya que al ser un poco vago no apunté perfectamente la antena por lo tanto la señal alcanzaba pero estaba con lo justo. Un poco de lluvia y la sincronización se complicó bastante, pero aún así funcionó bastante bien

![Blockchain sync](/assets/images/blockstream/recepcion-blockchain-sync.jpg){:class="img-responsive"} 


### Día 3: Desarrollo final del hardware

Ya con todo probado bien atado con alambre decidí que sería una mejor idea montar todo en un gabinete. Opté por reciclar un viejo decodificador de DirecTV que ya no funcionaba y de paso reutilicé su fuente de alimentación. Me viene bien para alimentar el LNB.

![Raspberry PI](/assets/images/blockstream/raspberry-fijado.jpg){:class="img-responsive"} 

El resultado final me gustó bastante. Quedó todo ordenado y sin problemas. Tuve que dejar la fuente original del Raspberry PI 4 porque no encontré tensión con la corriente suficiente como para alimentarlo desde la fuente de alimentación del decodificador. No es lo mas lindo pero igual es funcional. El receptor SDR lo monté con un perfil de chapa apretado por el mismo conector SMA y de ahí me fui con un pigtail SMA al power inserter.

![Gabinete final](/assets/images/blockstream/gabinete-final.jpg){:class="img-responsive"} 


A la foto anterior terminé agregándole un [LM317](https://www.ti.com/lit/ds/symlink/lm317.pdf) como regulador de tensión para poder ajustar la salida a 13v y de esa manera lograr polarización horizontal en el LNB. Con esto me ahorro tener que poner el receptor de FTA en el medio para alimentar el LNB

![Power inserter](/assets/images/blockstream/power-inserter-gabinete.jpg){:class="img-responsive"} 



### Día 4: Conclusión final

Como conclusión final a mi criterio (100% opinión personal) esto está buenísimo para experimentar y jugar pero no tiene un uso que se pueda aplicar completamente a la vida real.
1. Desde el momento en que tenemos que presincronizar la Blockchain para poder mantenerla up to date con el satélite ya pierde bastante el sentido
2. Esto nos limita a que Blockstream siga teniendo toda esta infra (que es gigante) funcionando para poder bajar los bloques
3. Solo Blockstream está actualmente haciendo esto por lo tanto no tenemos mucho para elegir

Hay algunas cosas divertidas como enviar mensajes o archivos desde internet y descargarlos por satélite, es bastante reconfortante ver como funciona!
