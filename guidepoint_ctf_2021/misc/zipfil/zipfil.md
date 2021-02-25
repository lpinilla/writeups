# Zipfil

En esta oportunidad les traigo un ejercicio de la categoría de "misc" de la GuidePoint CTF 2021.

## Enunciado

En este caso simplemente nos dan un archivo llamado "zipfil.zip" pero si realizamos el comando `file` podemos ver algo interesante:

```
$ file zipfil.zip
$ original.zip: data
```

Podemos ver entonces que si bien el nombre induce a que es un archivo .zip, no se reconoce como tal.

Si ejecutamos el comando `strings` para ver las cadentas de texto legibles dentro de larchivo, esto solamente nos devuelve la palabra "PEUG". Al principio pensé que se trataba de algún método de enpaquetamiento así que después de buscar un rato y no encontrar nada útil, decidí ignorarlo.

## Otras alternativas

La única otra alternativa que se me ocurrió es examinar los valores hexadecimales del archivo, a ver si hay algo que nos llama la atención y de repente..

```
00000000: 0000 0000 00e8 0000 00e4 0010 0010 0000  ................
00000010: 0000 6050 b405 0000 0000 4000 0000 0040  ..`P......@....@
00000020: 1000 b087 57d5 c89d 9130 0050 4555 4787  ....W....0.PEUG.
00000030: 47e2 7616 c666 0000 0000 184a 0000 0010  G.v..f.....J....
00000040: 0000 0000 0081 0080 0000 0003 0000 00c3  ................
00000050: 8686 e8c9 f4a3 b5d8 0000 0090 00a0 30e1  ..............0.
00000060: 2010 b405 0000 0003 0000 00c3 8686 e8c9   ...............
00000070: 8070 b405 d429 68c0 6fd0 5f87 9ed2 1e54  .p...)h.o._....T
00000080: 60e5 794f 80d2 8c4d e80a f217 cb9b 28a5  `.yO...M......(.
00000090: 1cf3 b655 1163 c31c ca92 8ecd 0e7e f6cd  ...U.c.......~..
000000a0: 9eef 55b8 9df4 382b 8525 59ea f1d7 87bd  ..U...8+.%Y.....
000000b0: 0000 0000 4000 0000 0040 1000 b087 57d5  ....@....@....W.
000000c0: c89d 91d5 c89d 9130 0090 4555 4787 47e2  .......0..EUG.G.
000000d0: 7616 c666 00c1 0080 0000 0003 0000 00c3  v..f............
000000e0: 8686 e8c9 f4a3 b5d8 0000 0090 00a0 4030  ..............@0
000000f0: b405                                     ..
```

Lo interesante es que los primeros 2 bytes vengan en 0x00. Normalmente estos son los que se utilizan para determinar el tipo de archivo.

Buscando luego en [filesignatures.net](https://www.filesignatures.net/index.php?search=zip&mode=EXT) podemos ver que los archivos de tipo ZIP deben tener los primeros 4 bytes `50 4B 03 04`, así que nos podríamos preguntar, qué pasa si agrego esto manualmente? Si piso los 4 bytes por estos de forma tal que ahora si se reconozca que es un zip y el archivo esté arreglado?

Para realizar esto simplemente podemos utilizar un heditor de hexadecimal y simplemente pisar el valor y guardar a un nuevo archivo. Lamentablemente, esta no es la solución al problema.

## El truco

Si volvemos a ver el archivo, volvemos a ver que los primeros 4 bytes están en 0 mientras que los últimos están ocupados. En este caso, esto llama la atención.

Normalmente hay algunos archivos que tienen al final del éste una "Firma de fin de archivo" pero en este caso, como no hay una firma inicial en los primeros bytes, no tendría mucho sentido que si haya una firma de fin de archivo.

Pero si miramos bien los últimos bytes podemos ver que llaman mucho la atención con los bytes que mencionamos antes

`4030 b405` -> `40 30 b4 05` -> `50 4b 03 04`. Acá está la firma del archivo ZIP!!!

Acá está el truco. El archivo está al revéz. Para leerlo habría que leer de abajo para arriba y de derecha a izquierda.

Así que ahora podemos construir un script en python para poder recuperar el archivo original.

## El script

```python
#abrir archivo y guardar los bytes en una variable
f = open('zipfil.zip', 'rb')
bytes_arr = [b for b in bytearray(f.read())]
f.close()

#invertir el archivo
bytes_arr.reverse()

def revert_byte(b):
    #agarrar los primeros 4 bits del byte
    upper_byte = b & 0xf0
    #agarrar los últimos 4 bits del byte
    lower_byte = b & 0x0f
    result = b

    #swap
    #limpiar y setear los primeros bytes
    result ^= upper_byte ^ (lower_byte << 4)

    #limpiar y setear los últimos bytes
    result ^= lower_byte ^ (upper_byte >> 4)
    return result

#guardar a un archivo
f = open('inverted.zip', 'wb+')
for i in range(len(bytes_arr)):
    f.write(bytes([revert(bytes_arr[i])]))
f.close()
```

De esta forma podemos recuperar el archivo original.

## Etapa final

Si bien pudimos recuperar el archivo original, todavía no terminó el desafío. Si intentamos abrir el archivo, este nos pide una contraseña.

Esto lo podemos solucionar utilizando JohnTheRipper, por lo que necesitamos primero obtener el hash del zip file para que JTR pueda utilizarlo.

```
$ zip2john inverted.zip > zipfile.hash
$ john zipfile.hash
```

De esta forma, podemos finalmente conseguir la contraseña (que era simplemente 123456) y así obtener el flag.
