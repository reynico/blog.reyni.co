---
layout: post
title:  "Enventanado de señales con GNURadio y receptor SDR"
date:   2021-03-14 17:44:27 -0300
categories: [gnuradio, dsp]
---

El enventanado de señales utilizando filtros FIR es de vital importancia a la hora de analizar señales. Nos podemos imaginar un filtro donde tenemos un recorte de frecuencias, entonces de un espectro amplísimo como tiene cualquier receptor SDR podemos recortar pequeñas porciones de señal para poder analizarlas.

Una buena manera de demostrar la utilización de los filtros FIR en GNURadio es con la creación de un receptor de radio FM donde su ancho de banda nominal es de 140 KHz pero un receptor SDR puede escuchar simultaneamente 2 MHz o mas. Entonces, como recortamos lo que nos interesa?

# Construyendo nuestro receptor FM
Lo primero que vamos a hacer es elegir la frecuencia de alguna estación FM cercana, en mi caso elegí 101.5 MHz. También una tasa de muestreo fácil de dividir en base 2: 2048 Msps.

1. Buscamos un bloque RTL-SDR source u Osmocom SDR source
2. Buscamos un bloque QT GUI Sink
3. Buscamos un bloque Variable y lo colocamos dos veces
4. Conectamos la salida del bloque SDR Source a la entrada del bloque QT GUI Sink
5. A una de las variables la llamaremos `samp_rate`. Ésta será nuestra tasa de muestreo. Es buena idea siempre usar variables para valores que vamos a utilizar en varios bloques. En mi caso `samp_rate` me interesa para el bloque del receptor SDR y en el bloque de muestreo donde voy a poner la cascada. 
6. La otra variable se llamará `freq` y será la frecuencia de recepción central de nuestro SDR. Para 101.5 MHz deberán introducir `101.5e6` cuya notación convierte 101.5 MHz a Hz.

Ésta es la configuración de mi bloque Osmocom SDR source:

[![configuracion bloque sdr](/assets/images/gnuradio-enventanado/config-receptor.jpg)](/assets/images/gnuradio-enventanado/config-receptor.jpg)

Y este es el flujo completo por ahora:

[![flujo completo paso 1](/assets/images/gnuradio-enventanado/flujo-gnuradio-paso-1.jpg)](/assets/images/gnuradio-enventanado/flujo-gnuradio-paso-1.jpg)


Ahora ya podemos darle `Play` a nuestro flujo de trabajo y ver que ocurre. Podemos jugar un poco con los parámetros de ganancia e intensidad o simplemente presionar el botón `Auto Scale` para que los acomode automáticamente. Veremos entonces varias portadoras con la forma típica de una transmisión FM. En total estaremos escuchando 2 MHz de ancho de banda (101.5 MHz como central, 100.5 MHz a la izquierda y 102.5 MHz a la derecha), en mi caso esto indica que estoy escuchando 6 estaciones FM en simultaneo. Por supuesto que si quisieramos escuchar esto tal cual está ahí no entenderíamos absolutamente nada.

[![flujo paso 1 ejecutado](/assets/images/gnuradio-enventanado/receptor-paso-1.jpg)](/assets/images/gnuradio-enventanado/receptor-paso-1.jpg)

[Pueden descargar el archivo .grc desde este enlace](/assets/files/gnuradio-enventanado/receptor-fm-paso-1.grc)

## Enventanando la señal que nos interesa

El siguiente paso entonces será enventanar la señal, recortar solo lo que nos interesa y depreciar el resto. Para eso vamos a usar un bloque específico: `Frequency Xlating FIR Filter` que nos permitirá no solo pasarle un filtro a la señal, sino también hacer correcciones en su frecuencia (como para sintonizar otra frecuencia y no solo la central)

[![configuracion fir filter](/assets/images/gnuradio-enventanado/config-fir-filter-1.jpg)](/assets/images/gnuradio-enventanado/config-fir-filter-1.jpg)

Como verán, hay una sección `Taps` que se encuentra vacía y es justo ahí donde vamos a definir nuestro filtro por software. La sintaxis va a ser la siguiente

```python
firdes.low_pass(ganancia, tasa_muestreo, frecuencia_recorte, transición, firdes.tipo_ventana)
```

Donde:
* `ganancia` tendrá un valor `1` por ahora.
* `tasa_muestreo` va a tomar el valor `samp_rate` que entonces será ajustado a nuestra variable `samp_rate` en 2048 Msps.
* `frecuencia_recorte` será entonces la mitad del ancho de banda total de una transmisión FM: `70 KHz` pues toma los valores absolutos +/- 70 KHz. Ingresaremos `70000`.
* `transición` es la banda de transición del filtro en sus bordes. No existe un filtro capaz de dejar pasar exactamente 70 KHz de ancho de banda y atenuar todo el resto a cero, entonces creamos una banda de transición para que interpole la ganancia, por ahora podemos probar con 20 KHz e ir ajustándolo a medida que escuchemos. El valor será entonces `20000`.
* `tipo_ventana` es que enventanado le queremos dar a nuestra señal. [Hay diferentes tipos de enventanamientos posibles](https://es.wikipedia.org/wiki/Ventana_(funci%C3%B3n)) pero no nos vmaos a centrar en la explicación teórica pues es algo pesada. Probaremos con `WIN_KAISER` para utilizar una [ventana de Kaiser](https://es.qaz.wiki/wiki/Kaiser_window)

Entonces nuestro código resultante será

```python
firdes.low_pass(1, samp_rate, 70000, 20000, firdes.WIN_KAISER)
```

Lo bueno de todo esto es que podemos simular el enventanamiento en una consola de Python. Si están usando GNURadio 3.8 ejecuten `python3` en una terminal y luego ejecuten el siguiente código:

```python
from gnuradio.filter import firdes
from scipy import signal
import matplotlib.pyplot as plt
import numpy as np

samp_rate = 2048000
func_ventana = firdes.low_pass(1, samp_rate, 70000, 20000, firdes.WIN_KAISER)
w, h = signal.freqz(func_ventana)

plt.plot(w, 20 * np.log10(abs(h)), 'b')
plt.ylabel('Amplitud [dB]', color='b')
plt.xlabel('Frecuencia')
plt.show()
```

Si todo salió bien van a obtener algo así:

[![simulación filtro fir](/assets/images/gnuradio-enventanado/simulacion_filtro.jpg)](/assets/images/gnuradio-enventanado/simulacion_filtro.jpg)

Ahora vamos a plasmar este filtro en nuestro bloque FIR. En el campo `Taps` ingresen:

```python
firdes.low_pass(1, samp_rate, 70000, 20000, firdes.WIN_KAISER)
```

Y ejecuten el flujo. Verán entonces una señal ganadora: nuestra frecuencia central, todo lo demás entonces queda atenuado gracias a la configuración de nuestro filtro por software.

[![ejecución filtro fir](/assets/images/gnuradio-enventanado/filtro-fir-kaiser.jpg)](/assets/images/gnuradio-enventanado/filtro-fir-kaiser.jpg)

Para darle un poco mas de dinamismo a nuestro flujo de trabajo, agreguemos dos bloques `QT GUI Range`. Los utilizaremos para variar en tiempo real el ancho de nuestra ventana y su banda de transición. Editaremos nuevamente el bloque `Frequency Xlating FIR Filter` y en Taps vamos a colocar:

```python
firdes.low_pass(1, samp_rate, filter_cutoff_freq, filter_transition_band, firdes.WIN_KAISER)
```

Donde `filter_cutoff_freq` y `filter_transition_band` serán nuestros nuevos elementos `QT GUI Range`. Nuestro flujo debería verse así:

![filtros variables](/assets/images/gnuradio-enventanado/filtros-variables.jpg)

Y si ejecutamos nuestro flujo de trabajo ahora podremos editar en tiempo real el ancho de recorte de frecuencias y su banda de transición

[![filtros variables](/assets/images/gnuradio-enventanado/ejecucion-filtros-variables.jpg)](/assets/images/gnuradio-enventanado/ejecucion-filtros-variables.jpg)

[Pueden descargar el archivo .grc que contiene el filtro variable desde este enlace](/assets/files/gnuradio-enventanado/receptor-fm-paso-2.grc)

## Diezmado de la señal

Ahora que entendimos como seleccionar una frecuencia, debemos continuar con un paso importante: Decimation o Diezmado de la señal. El diezmado es un proceso donde descartamos algunas muestras de nuestra señal original de esa manera logrando reducir la frecuencia de muestreo. Por ejemplo: haciendo un diezmado de factor 8 reduciremos nuestra tasa de muestreo de 2048 KHz en `2048/8 = 256 KHz`. El bloque `Frequency Xlating FIR Filter` ya posee esta característica en el parametro `Decimation`. Tengan en cuenta que esto mismo hay que hacerlo en el bloque `QT GUI Sink`. De otra manera el waterfall funcionará 8 veces mas lento pues seguirá usando la tasa de muestreo de `2048`.

Para trabajar mas cómodos agregaremos una variable mas a nuestro flujo: `decimation_rate` y su valor será `8`. Ahora editamos el bloque `QT GUI Sink` y en el campo Bandwidth colocaremos `samp_rate/decimation_rate`, GNURadio se encargará de hacer la división correspondiente. También tenemos que editar `Frequency Xlating FIR Filter` y colocar `decimation_rate` en el campo Decimation.

[![configuracion de diezmado](/assets/images/gnuradio-enventanado/configuracion-diezmado.jpg)](/assets/images/gnuradio-enventanado/configuracion-diezmado.jpg)

Ejecutando el flujo de GNURadio ahora veremos que hicimos _zoom_ sobre la señal elegida, con un factor de _aumento_  de 8 unidades:

![señal diezmada](/assets/images/gnuradio-enventanado/waterfall-diezmado.jpg)

[Pueden descargar el archivo .grc sobre diezmado desde este enlace](/assets/files/gnuradio-enventanado/receptor-fm-paso-3.grc)

## Demodulación FM

Con la señal enventanada y diezmada ya estamos listos para demodular y escuchar nuestra estación elegida. Para ello buscaremos el bloque `WBFM Receive` en GNURadio y lo conectaremos a la salida de `Frequency Xlating FIR Filter`, en paralelo con el `QT GUI Sink`. 

[![Configuración de WBFM receive](/assets/images/gnuradio-enventanado/wbfm-receive-config.jpg)](/assets/images/gnuradio-enventanado/wbfm-receive-config.jpg)

Hay algunas cosas que configurar aquí. Para el Quadrature rate vamos a usar la misma tasa de muestreo del diezmado: `samp_rate/decimation_rate`, o sea 256 KHz. Para el diezmado del audio vamos a volver a usar `decimation_rate`, de esta manera volveremos a diezmar sobre 8 consiguiendo una frecuencia resultante de `256 KHz / 8 = 32 KHz`, donde 32 KHz es una frecuencia conveniente para prácticamente cualquier placa de sonido.

Por último agregaremos un bloque `Audio Sink` que es quien nos dará sonido desde el demodulador FM. En `Sample Rate` vamos a elegir `32 KHz` y en `Device Name` yo personalmente tuve que usar `sysdefault` para que me funcione correctamente (Corriendo GNURadio en una máquina virtual Linux desde un host Mac OS).

[![Flujo completo demodulador FM](/assets/images/gnuradio-enventanado/flujo-demodulador-fm.jpg)](/assets/images/gnuradio-enventanado/flujo-demodulador-fm.jpg)

Si ejecutan el flujo de GNURadio, estarán escuchando la emisora FM en la frecuencia que eligieron (En mi caso, 101.5 MHz). Y jugando con la frecuencia de corte y la banda de transición podrán notar como eso afecta a la señal resultante (el audio). Por debajo de 70 KHz de ancho de banda empezaremos a notar el recorte de frecuencias.


[Pueden descargar el archivo .grc con el demodulador FM completo desde este enlace](/assets/files/gnuradio-enventanado/receptor-fm-paso-4.grc)


## Escuchando varias señales en simultaneo

Una de las ventajas de los receptores SDR es la posibilidad de recibir y guardar varias señales en paralelo. Por ejemplo para AIS (Automatic Identification System) nos interesan dos portadoras de 25 KHz que se encuentran en 161.975 MHz y 162.025 MHz. Al estar tan cercanas entre si, es fácil consumir los datos de cada portadora con un mismo receptor SDR, utilizando dos filtros FIR (`Frequency Xlating FIR Filter`) y derivando cada salida filtrada a dos `File Sink` diferentes, que operan como los `Audio Sink` pero en lugar de escribir hacia la placa de sonido, lo hacen a un archivo en formato RAW.


[![Flujo completo receptor doble](/assets/images/gnuradio-enventanado/receptor-doble.jpg)](/assets/images/gnuradio-enventanado/receptor-doble.jpg)

En este flujo podemos observar una misma fuente de información (Un SDR) que alimenta a dos filtros FIR, uno está centrado a 0 Khz y el otro a - 400 Khz, de esa manera uno está sintonizado entonces a 101.5 Mhz y el otro a 101.1 Mhz, siendo entonces dos emisoras FM diferentes. Cada demodulador FM está conectado a un multiplicador que toma valores de tipo float entre 0 y 1. Ésta es la ganancia de cada salida que luego termina en un sumador y de ahí al `Audio Sink`. Entonces variando los volúmenes (con dos `QT GUI Range`) podemos escuchar una emisora u otra.
 
[Pueden descargar el archivo .grc con el doble demodulador de FM desde este enlace](/assets/files/gnuradio-enventanado/receptor-fm-paso-5.grc)
