Mediante el uso del sensor MQ-135 se detectarán gases como monóxido de carbono (CO), dióxido 
de carbono (CO₂), tolueno, amoníaco (NH₃) y acetona. De acuerdo con la información proporcionada 
en la hoja de datos (datasheet), este sensor no es capaz de diferenciar entre las concentraciones 
específicas de cada gas, por lo que se recomienda su uso en entornos donde se prevea la presencia de 
alguno de los compuestos mencionados. 

![Ejerciocio 1](/Imagenes/mq135.png)

La gráfica **RS/R0** vs ppm indica cómo varía la resistencia del sensor MQ-135 al detectar diferentes gases en función de su concentración.   
**RS**: resistencia del sensor en el momento de la medición.   
**R0**: resistencia del sensor en aire limpio (se calibra al inicio).   
**RS/R0**: relación que se usa para identificar gases específicos.   
Como se observa en la gráfica, el sensor presenta mayor sensibilidad al monóxido de carbono (CO), 
por lo que este será el primero en superar el umbral de concentración en caso de presencia de gases, 
siendo reflejado en la pantalla como una alerta de exceso (Como se puede ver en la imagen).   

![Ejerciocio 1](/Imagenes/montaje.png)

Como se mencionó anteriormente, el sensor no puede diferenciar con exactitud entre distintos 
gases. Sin embargo su resistencia varía de manera distinta según el tipo de gas, es posible 
estimar cuál está presente calculando la relación **RS/R0** y comparándola con las pendientes 
de las curvas mostradas en la gráfica del datasheet. De esta manera, se puede hacer una 
estimación aproximada del gas predominante, permitiendo afirmar: “*hay una alta 
probabilidad de que sea CO en lugar de NH₄*”.   

Existen ciertas limitaciones importantes en el uso del sensor MQ-135, ya que responde 
simultáneamente a todos los gases presentes en el ambiente. Por ejemplo, si hay CO y NH₄ 
al mismo tiempo, la lectura obtenida será una combinación de ambos, sin posibilidad de 
diferenciarlos de forma precisa. Además, las curvas de sensibilidad se cruzan en varias zonas 
del gráfico. A concentraciones como 10 ppm, los valores de **RS/R0** son muy similares para 
distintos gases, lo que puede generar confusión e incertidumbre en la estimación del gas 
predominante.

Al momento de detectar un gas, el microcontrolador evaluará si la concentración supera el 
umbral establecido en el código. En caso afirmativo, se activará una alarma sonora y 
comenzará el envío de datos a la plataforma Ubidots a razón de una lectura por segundo, 
únicamente mientras persista la condición de alta concentración. Se determinó que no es 
necesario transmitir valores normales de concentración, ya que esto podría saturar el servidor 
con datos irrelevantes. Por lo tanto, una vez que los niveles de gas desciendan por debajo del 
umbral, se detendrá automáticamente el envío de datos. 

![Ejerciocio 1](/Imagenes/grafica_ubidots1.png)

En este caso se eligió monitorear metano, ya que el sensor se ubicará en la cocina de un hogar.  

![Ejerciocio 1](/Imagenes/ubidots2.png)

En el código se modificó cada variable con unos valores predeterminados, son los 
coeficientes de la fórmula logarítmica que se usa para estimar la concentración del gas (en 
ppm) a partir del valor que da el sensor. Estos valores ya han sido calculados por la 
comunidad o por el fabricante para usarlos directamente.

El codigo se presenta en el otro archivo de la carpeta.
