---
title: "Cuaderno de laboratorio"
author: "Edgar Caballero"
date: "2022-07-14"
output: 
  html_document:
            keep_md: yes
            toc: true
            toc_depth: 5
            toc_float:
              collapsed: false
              smooth_scroll: true
---



## Stacks en Hydra

### ¿ Qué es Stacks ?

Stacks es una herramienta bioinformática que sirve para analizar secuencias Rad que permite exportar los resultados para hacer análisis posteriores con otras herramientas bioinformáticas. En esta sección trabajaré con una base de datos de secuencias RAD, previamente limpia, para armar loci utilizando el programa `denovo_map.pl` , desde la super computadora [Hydra](https://confluence.si.edu/display/HPC/High+Performance+Computing).

### Análisis `Denovo_map.pl`

Lo primero que haremos será ir dentro del terminal de linux en Hydra para ubicarnos

    pwd

Esto sirve para saber donde estamos y luego nos ubicamos donde se encuentra la base de datos para armar el comando.

Luego cambiamos la dirección a nuestro espacio en Hydra con el comando `cd` de la siguiente manera:

    cd /scratch/genomics/caballeroeg /

Aquí con el comando `mkdir` crearemos un directorio que servirá para guardar los archivos generados por nuestra primera corrida del programa `denovo_map.pl` para encontrar la combinación de parámetros más óptima que nos genere la mayor cantidad de loci que estén presentes en la mayoría de individuos en nuestro sset de datos.

    mkdir -p optimization_denovo/m1 optimization_denovo/m2 optimization_denovo/m3 # and so it goes until we reach m9

Crearemos 9 subdirectorios dentro del directorio "padre" optimization_denovo, para que los resultados de las siguientes corridas no se mezclen.

### Optimización de los parámetros.

Stacks contiene tres parámetros principales que controlan la formación e identificación de posbiles loci en la población. Uno de ellos es `-m`, que especifíca el número mínimo de lecturas única (depth of coverage) para cada secuencia que debe cumplir para considerarse una lectura primaria exitosa (y formar un stacks), si no se llega al número mínimo, la secuencia es considera para una lectura secundaria que será procesada después.

El parámetro `-M` controla el número de diferencias entre nucleótidos permitidos entre lecturas primarias o stacks para considerarlos loci distintos.

Por último, el parámetro `-n` compara cada uno de los loci, de los individuos presentes, con el catálogo formado por todos los loci de todos los individuos del set de datos, para determinar las diferentes formas o alelo de cada locus, que podría ser monomórfico o polifórmico.

Para escoger la mejor combinación de `-m`, `-M` y `-n` para nuestros datos debemos crear un submuestreo de nuestros datos que denominaremos `popt.txt`. El popt.txt lo cree tomando cinco individuos de cada sitio de muestreo de manera aleatoria cuyo coverage es mayor a 60.00 y lo coloque usando el siguiente formato:

    1s02<tab>opt
    2s09F<tab>opt
    2s08<tab>opt
    2s25<tab>opt
    ...<tab>opt
    ...

Es importante usar la tecla `tab` para que el programa reconozca el espacio, de lo contrario no leerá el archivo. Para esta prueba no es necesario especificar el sitio de muestreo y por eso coloque opt.

Una vez definido el popmap (popt.txt), los archivos de stacks y el directorio donde se guardarán los resultados vamos a ejectuar el comando `denovo_map.pl`, con el siguiente formato:

``` /bin/sh
# ----------------Parameters---------------------- #
#$  -S /bin/sh
#$ -pe mthread 30
#$ -q mThM.q
#$ -l mres=240G,h_data=8G,h_vmem=8G,himem
#$ -cwd
#$ -j y
#$ -N o8
#$ -o o8.log
#$ -m bea
#$ -M caballeroeg@si.edu
#
# ----------------Modules------------------------- #
module load ~/modulefiles/miniconda
source activate ste
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
denovo_map.pl -m 3 -M 8 -N 8 --samples /scratch/genomics/caballeroeg/CPrads/clean_rads/ --popmap /scratch/genomics/caballeroeg/optimization_denovo/m1/popt.txt  -o /scratch/genomics/caballeroeg/optimization_denovo/m8   --paired -r 0.8
 
#
echo = `date` job $JOB_NAME done

```

En cada corrida de optimización los parámetros a varias son `-M` y `-n` que irán de de uno hasta nueve sucesivamente. Para conocer más detalles acerca del método de optimización puede hacer click [aquí](https://www.biorxiv.org/content/biorxiv/early/2021/11/04/2021.11.02.466953.full.pdf) y visitar el sitio web de [Stacks](https://catchenlab.life.illinois.edu/stacks/comp/denovo_map.php).

Cuando terminen las corridas podemos extraer la cantidad de loci identificados para cada set de parámetros con el siguiente comando:

    $ cat populations.sumstats.tsv | \
     grep -v '^#' | \ # Quita los comentarios
     cut -f 1 | \ # Corta la primera columna (Locus ID)
     sort -n -u | \ # ordena por orden númerico y valores únicos
     wc -l # Cuenta el número de lineas
     10065

Por ejemplo para `-M`=1 y `n`=1 la cantidad de loci polimórficos encontrados fue de 10065. Esto lo haremos para cada set de parámetros y los acdomodadres en la siguiente tabla.


```r
optimization <- read.csv("r_80.csv",sep = ";")
optimization
```

```
##   M R_80.populations.sumstat.tsv. change.in._r_80
## 1 1                         10065               -
## 2 2                         10526             461
## 3 3                         10569              43
## 4 4                         10472             -97
## 5 5                         10359            -113
## 6 6                         10173            -186
## 7 7                         10000            -173
## 8 8                          9869            -131
## 9 9                          9760            -109
```

Para conocer cual parámetro *denovo* es el más óptimo para nuestros datos, debemos calcular la diferencias de loci identificados para cada combinación de datos. Este valor se extrae (change.in.\_r_80) cálculando la diferencia entre `-M` (por ejemplo, `M2` -`M1`) para conocer cual parámetro nos genera mayor loci. No calculamos el valor de `-M`=1 porque no identifica loci polimórficos. De tal manera quepara calcular, el cambio de loci en `M=2` `M2` -`M1` da como resultado 461 en la tercera columna. Este valor es el que vamos a utilizar para saber cual es la combinación más precisa para nuestros datos.

El valor de la diferencia entre `-M` más cercano a 0 y que sea positivo es el valor que nos dirá cuál parámetro identifica la mayor cantidad de loci polimórficos con cierta flexibilidad, reduciendo el error de colocar loci polimórficos como diferentes loci o viceversa, identificar diferentes loci como un gran loc polifórmico. En nuestro caso el parámetro más óptimo es `M3`.


```r
library(ggplot2)
change <- as.numeric(optimization$change.in._r_80)
```

```
## Warning: NAs introducidos por coerción
```

```r
class(change)
```

```
## [1] "numeric"
```

```r
ggplot(data= optimization, aes(x = optimization$M, y=change, na.rm = T)) + geom_point(color ="steelblue", size = 2) +geom_line() + theme_grey(base_family="mono")+ ggtitle(" Change in loci identified by the M parameter")+ylab("Change in M") + xlab(" M parameter") + theme(plot.title = element_text(hjust = 0.5)) + xlim(0,9)
```

```
## Warning: Use of `optimization$M` is discouraged. Use `M` instead.
```

```
## Warning: Use of `optimization$M` is discouraged. Use `M` instead.
```

```
## Warning: Removed 1 rows containing missing values (geom_point).
```

```
## Warning: Removed 1 row(s) containing missing values (geom_path).
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

`M1` no está en la grfica porque no hay comparación, por lo que el cambio en la gráfica empieza desde `M2`.

### Ejecutando el comando `denovo_map.pl`

Luego de conocer el parámetro óptimo para nuestros datos procederemos a ejecutar el comnado `denovo_map.pl` con las siguientes especificaciones:

    # /bin/sh                                                                                                         
    # ----------------Parameters---------------------- #          
    #$  -S /bin/sh                                                                             
    #$ -pe mthread 30           
    #$ -q mThM.q                                                                                                      
    #$ -l mres=240G,h_data=8G,h_vmem=8G,himem       
    #$ -cwd                                                                                                           
    #$ -j y                                                                                                           
    #$ -N m3p                       
    #$ -o m3p.log                                                                                                     
    #$ -m bea                 
    #$ -M caballeroeg@si.edu
    #                                                                                                                 
    # ----------------Modules------------------------- #                                                              
    module load ~/modulefiles/miniconda                                                                               
    source activate ste
    #                                                                                                           
    # ----------------Your Commands------------------- #     
    #                                                                                                                          
    echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME                                     
    echo + NSLOTS = $NSLOTS 

    denovo_map.pl -m 3 -M 3 -N 3 
    --samples /scratch/genomics/caballeroeg/CPrads/clean_rads/   #Directorio donde están los archivos stacks
    --popmap /scratch/genomics/caballer oeg/CPrads/clean_rads/popmp_1.txt  ##population map indicando el individuo y la población
    -o /scratch/genomics/caballeroeg/CPrads/clean_rads/edgar_analysis/m3p/  #Directorio para guardar 
    --paired  #Para nuestras lecturas de pares

    #                                                                                                                          
    echo = `date` job $JOB_NAME done  

En este caso el archivo popmap contiene el siguiente formato:

    1s01F<tab>1
    1s01M<tab>1
    1s02<tab>1
    ...

Especificando la población donde pertenece el individuo.

Una vez finalizado el programa `denovo_map.pl` completamente podemos revisar la cantidad de loci identificados, haciendo uso del siguiente comando en la consola.

    cat gstacks.log | grep -B 2 -A 3 '^Genotyped'

El cual imprime la cantidad de loci identificados :

    Genotyped 93683 loci:
      effective per-sample coverage: mean=49.4x, stdev=28.4x, min=4.5x, max=188.2x                                             
      mean number of sites per locus: 465.0                                                                                    
       a consistent phasing was found for 756770 of out 999308 (75.7%) diploid loci needing phasing    

También podemos obervar a nivel de individuos cuales muestras, tienen un coverage bajo (10x\>) y que deberíamos descartar para los análisis siguientes ya que nos pueden generar ruido y desviaciones en nuestra data. Entonces bajo el siguiente comando

     stacks-dist-extract gstacks.log.distribs effective_coverages_per_sample \
     | grep -v '^#' \
     | cut -f 1-2,4-5,8

    # For mean_cov_ns, the coverage at each locus is weighted by the number of                                                 
    # samples present at that locus (i.e. coverage at shared loci counts more).                                                
    sample  n_loci  n_used_fw_reads mean_cov        mean_cov_ns                                                                
    1s01F   15217   600125  39.438  42.716                                                                                     
    1s01M   28001   3383027 120.818 167.263                                                                                    
    1s02    15521   568731  36.643  40.246                                                                                     
    1s03    17532   734543  41.897  48.854                                                                                     
    1s04    16607   1120518 67.473  75.308                                                                                     
    1s05    17848   796382  44.620  52.041                                                                                     
    1s06    17807   938246  52.690  60.962    

En en shell podemos convertir el arhivo en un txt de la siguiente manera:

    stacks-dist-extract gstacks.log.distribs effective_coverages_per_sample | grep -v '^#' | cut -f 1-2,4-5,8 > raw_cov.txt

Desde el servidor Hydra exportamos este archivo y lo podemos guardar a nuestro ordenador cargndo el siguiente módulo que nos llevará al siguiente [enlace](https://send.vis.ee) para descargar nuestro archivo :

    module load tools/ffsend
    ffupload raw_cov.txt
    https://send.vis.ee/download/d0099f8000924544/#hYKJOtgtz0d1sMyXrhOLgw   

Luego, podemos editar el archivo `raw_cov.txt` desde un editor de texto como [Visual Studio Code](https://code.visualstudio.com), [Notepad++](https://notepad-plus-plus.org/downloads/) o el de su preferencia dependiendo de sus gustos, o sistema operativo. No es recomendable usar el bloc de notas o Microsoft Word.

En el editor de texto, abrimos el archivo :

```
sample|depth of|max  |reads  | %reads
      |cov     |cov  |incorpo| incor
------|--------|-----|-------|-----
3Bx42 | 19.37  |278  |375366 |  77.0
3S36  | 14.43  |256  |265521 |  73.3
3S38H | 5.58   |118  |15367  |  38.5
3S39M | 6.32   |181  |77588  |  68.1
```

En la segunda columna están los depth of coverages que nos interesan. Aquellos individuos que tengan un depth of coverage menor a 10x (como la muestras 3s338H) serán descartados para el próximo análisis, el cual es el comando `populations` que nos servirá para otros programas bioinformáticos.

### Populations software

El programa [`populations`](https://catchenlab.life.illinois.edu/stacks/comp/populations.php) analizará los individuos dentro de las poblaciones y calculará varios estadísticos asociados como FST, FIS, también calculará la frecuencia asociada a la heterocigosidad esperada, heterocigosidad observada y en diferentes formatos como [STRUCTURE](https://web.stanford.edu/group/pritchardlab/structure.html), [plink](https://www.cog-genomics.org/plink/1.9/formats), [genepop](https://www.cog-genomics.org/plink/1.9/formats) por mencionar algunos.

    # /bin/sh                                                                                                                  
    # ----------------Parameters---------------------- #                                                                       
    #$  -S /bin/sh                                                                                                             
    #$ -pe mthread 30                                                                                                          
    #$ -q mThM.q                                                                                                               
    #$ -l mres=240G,h_data=8G,h_vmem=8G,himem                                                                                  
    #$ -cwd                                                                                                                    
    #$ -j y                                                                                                                    
    #$ -N paired_populations                                                                                                   
    #$ -o paired_populations.log                                                                                               
    #$ -m bea                                                                                                                  
    #$ -M caballeroeg@si.edu                                                                                                   
    #                                                                                                                          
    # ----------------Modules------------------------- #                                                                       
    module load ~/modulefiles/miniconda                                                                                        
    source activate ste                                                                                                        
    #                                                                                                                          
    # ----------------Your Commands------------------- #                                                                       
    #                                                                                                                          
    echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME                                              
    echo + NSLOTS = $NSLOTS                                                                                                    
    #                                                                                                                          
    populations -P /scratch/genomics/caballeroeg/CPrads/clean_rads/edgar_analysis/m3p 
    -M /scratch/genomics/caballeroeg/populations_ana/paired_populations/paired_popmap.txt 
    -O /scratch/genomics/caballeroeg/populations_ana/paired_populations 
    -p 3 
    -r 0.50 
    --fstats 
    --vcf 
    --genepop 
    --structure 
    --treemix 
    --hwe
    #                                                                                                                          
    echo = `date` job $JOB_NAME done

El comando que se presenta a continuación es un modelo de como funciona el programa y se puede adaptar para cada análisis posterior. hay dos parámetros que son pertinentes explicar `-p` indica el número mínimo que tiene que estar presente un locus para procesarlo en el análisis y el parámetro `-r` que indica el procentaje mínimo de individuos en una población para procesar un locus para esa población.

Para ver los resultados del populations ejecutamos el siguiente comando:

    Removed 78826 loci that did not pass sample/population constraints from 93683 loci.
    Kept 14857 loci, composed of 10149843 sites; 86404 of those sites were filtered, 412383 variant sites remained.            
    Number of loci with PE contig: 14857.00 (100.0%);                                                                          
      Mean length of loci: 673.17bp (stderr 0.60);                                                                             
    Number of loci with SE/PE overlap: 14856.00 (100.0%);

En este caso el número de loci presentes son 14857.

## STRUCTURE /Faststructure

[Structure](https://web.stanford.edu/group/pritchardlab/structure.html) es un programa de libre acceso para analizar datos genotípicos e investigar la estructura genética de una población.  Sirve para identificar "clusters" o grupos genotípicos en un set de datos asignados, a través de un algoritmo basado en el [equilibrio Hardy-Vweinberg](http://bioinformatica.uab.es/base/base3.asp?sitio=geneticapoblaciones&anar=concep&item=Hardy-Weinberg). [Faststructure](https://rajanil.github.io/fastStructure/) es un algoritmo similar al de STRUCTURE diseñado para identificar la estructura genotípica poblacional con datos basados en polimorfismos de un solo nucleótido (SNP, por sus siglas en inglés).

El programa asigna a cada individuo de la población (referido como genotipo y que se encuentra en la columna ) una probabilidad estadística asociada al grupo que pertenece el individuo de acuerdo a la cantidad de grupos genotípicos que sospechamos que podrían haber,

### Ejecutando populations software para análisis en STRUCTURE o Faststructure.

Primero, debemos ejecutar el comando llamar al programa

```
# /bin/sh                                                                                                        
# ----------------Parameters---------------------- #                                                              
#$  -S /bin/sh                                                                                                    
#$ -pe mthread 30                                                                                                 
#$ -q mThM.q                                                                                                      
#$ -l mres=240G,h_data=8G,h_vmem=8G,himem                                                                         
#$ -cwd                                                                                                       
#$ -j y                                                                                                        
#$ -N random_snps_population                                                                                   
#$ -o random_snps_population.log                                                                                  
#$ -m bea                                                                                                         
#$ -M caballeroeg@si.edu                                                                                        
#                                                                                                                 
# ----------------Modules------------------------- #                                                            
module load ~/modulefiles/miniconda                                                                               
source activate ste                                                                                               
#                                                                                                                 
# ----------------Your Commands------------------- #                                                            
#                                                                                                              
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME                                   
echo + NSLOTS = $NSLOTS                                                                                           
#                                                                                                                 
populations -P /scratch/genomics/caballeroeg/CPrads/clean_rads/edgar_analysis/m3p 
-M /scratch/genomics/caballeroeg/populations_ana/paired_populations/paired_popmap.txt 
-O /scratch/genomics/caballeroeg/populations_ana/random_snps_population 
-p 3 -
r 0.50 
--fstats 
--vcf 
--genepop 
--structure 
--treemix 
--hwe 
--write-random-snp
#                                                                                                                          
echo = `date` job $JOB_NAME done   

```
En nuestro  `jobfile` las especificaciones son similares al `populations` anterior, esta vez habilitamos la opción
`fstats` que calcula estadísticos F. la opción `hwe` evalúa los loci que están en equilibrio Hardy-Weinberg, la opción `write-random-snp` es para seleccionar al azar un snp por cada locus, de igual manera podemos seleccionar el primer snp de cada locus con la opción `write-single-snp`. Finalmente las opciones `vcf`, `genepop`,`structure` y `treemix` exportan los resultados en los formatos correspondientes. Para nuestros análisis usaremos el archivo en formato `structure`.

Antes de ejecutar Fasstructure a este archivo debemos modificarlo, para ello lo descargamos y podemos editarlo con [R](https://www.r-project.org), o [Microsft Excel](https://www.microsoft.com/es-es/microsoft-365/excel).


```r
raw_pop <- read.delim('Estructure_data.txt')
raw_pop[1:8,1:8]
```

```
##       X X.1 X1_115 X2_199 X4_491 X6_135 X9_107 X15_302
## 1 1s01F   1      2      2     -9      3      2       3
## 2 1s01F   1      3      2     -9      3      2       3
## 3 1s01M   1      3      2     -9      3      2      -9
## 4 1s01M   1      3      2     -9      3      2      -9
## 5  1s02   1      3      2      4      3      2      -9
## 6  1s02   1      3      2      4      3      2      -9
## 7  1s03   1      3      2     -9      3      2       3
## 8  1s03   1      3      2     -9      3      2       3
```

La estructura del archivo populations es la siguiente :

* Columna 1: Identificación de la muestra.

* Columna 2: Población de origen de la muestra.

* Columna 3-n: Datos de "SNPs"

Debemos  modificar ligeramente nuestros datos ya que las columnas 1-6 en faststructure, son metadatos y no incluyen la data de los SNPS. Con excel o R (donde te sientas más cómodo), agregaremos cuatro columnas del símbolo
'#', para que a partir de la columna 7 los datos de SNPS sean leídos corrrectamente. Para STRUCTURE pero para Faststructure no estoy seguro, es una buena práctica reemplazar los 0 con -9, para representar los datos en blanco o 'missing data' y eliminar los enbaezados de tal manera que la fila 1 sea del individuo 1. Como verán cada individuo ocupa dos filas y el programa solo acepta análisis diploides. Cabe destacar que el formato de este archivo debe ser **str** 

Nuestro archivo modificado debe quedar así:


```r
modi <- read.delim('structure_modified.txt')
modi[1:8,1:12]
```

```
##   X. X..1 X..2 X..3 X1s01F X1 X2 X2.1 X.9 X3 X2.2 X3.1
## 1  #    #    #    #  1s01F  1  3    2  -9  3    2    3
## 2  #    #    #    #  1s01M  1  3    2  -9  3    2   -9
## 3  #    #    #    #  1s01M  1  3    2  -9  3    2   -9
## 4  #    #    #    #   1s02  1  3    2   4  3    2   -9
## 5  #    #    #    #   1s02  1  3    2   4  3    2   -9
## 6  #    #    #    #   1s03  1  3    2  -9  3    2    3
## 7  #    #    #    #   1s03  1  3    2  -9  3    2    3
## 8  #    #    #    #   1s04  1  2    2   4  3    2   -9
```
Luego podemos aplicar el programa fasstructure, el cual tiene los siguientes argumentos :

```
Uso: python structure.py
     -K <int>    número de poblaciones asumidas
     --input=<file>   (/ruta/hacia/archivo /de/formato/'achivo.str') #sin comillas
     --output=<file>   (/ruta/hacia/resultados/del/programa)
     --tol=<float>   (criterio de convergencia; por defecto: 10e-6)
     --prior={simple,logistic}   (elección por defecto: simple)
     --cv=<int>   (números de validaciones cruzadas efectuadas en cada prueba, 0 implica sin validación cruzada; por defecto: 0)
     --format={bed,str} (formato del archivo de entrada del; por defecto es: bed. En nuestro caso 'str')
     --full   (Desplegar todos los parámetros variables; opcional)
     --seed=<int>   (especificar manualmente   el número de la semilla aleatorio, optional)

```

Esto fue traducido de la página de [vdreis](https://vdreis.com/faststructure/), el cual describe como transformar los datos par faststucture, a su vez del [sitio web oficial de fastructure](https://rajanil.github.io/fastStructure/). 


En nuestro caso me interesa saber cuál es la estructura genómica de mi población, con 5 sitios de muestreo distintos. En este caso haré 10 pruebas con la opción `K` de 1 a 10. Correré 10 pruebas seguidas y luego las compararé con el programa `ChooseK.py` que permite conocer cuál grupo es la mejor estructura genómica del grupo.


```
# /bin/sh                                                                                                         
# ----------------Parameters---------------------- #
#$ -S /bin/sh                                                                                                   
#$ -pe mthread 30                                                                                                 
#$ -q mThM.q                                                                                                      
#$ -l mres=240G,h_data=8G,h_vmem=8G,himem                   
#$ -cwd                                                                                                         
#$ -j y                                                                                                           
#$ -N k1                                                                                                          
#$ -o K_$TASK_ID.log                                                                                              
#$ -t 1-10                                                                                                        
#$ -m bea                                                                                                
#$ -M caballeroeg@si.edu  
# ----------------Modules------------------------- #                                                              
module load bio/faststructure/1.0                                                                                 
#                                                                                                               
# ----------------Your Commands------------------- #                                                              
#                                                                                                                 
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME                                              

echo + NSLOTS = $NSLOTS                                                                                                    
structure.py -K $SGE_TASK_ID
--input=/scratch/genomics/caballeroeg/FasSTRUCTURE/population
--output=/scratch/genomics/caballeroeg/FasSTRUCTURE/K 
--prior=simple 
--format=str 
--full 
--tol=10e-6 

```

De esta manera tenemos los archivos agrupados en un mismo directorio y ahora vamos a conocer el mejor modelo para nuestros datos:

```
# ----------------Parameters---------------------- #                                                              
#$ -S /bin/sh                                                                                                     
#$ -pe mthread 30                                                                             
#$ -q mThM.q                                                                                                      
#$ -l mres=240G,h_data=8G,h_vmem=8G,himem 
#$ -cwd                                                                                                           
#$ -j y                                                                                                           
#$ -N best                                                              
#$ -o best_k.log  
#$ -m bea                                                                                                         
#$ -M caballeroeg@si.edu
# 
# ----------------Modules------------------------- #          
module load bio/faststructure/1.0
#                                                                                                                 
# ----------------Your Commands------------------- #                                                              
#                                                                                                                          
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME                                              
echo + NSLOTS = $NSLOTS                                                                                           
chooseK.py --input=/scratch/genomics/caballeroeg/FasSTRUCTURE/K_                                                  
```

Y nuestro resultado es el siguiente.

```
Model complexity that maximizes marginal likelihood = 1                                                                    
Model components used to explain structure in data = 1 

```
 Para nuestra figura generada podemos usar otro algoritmo que nos permite visualizar los datos y entender la distribución genómica poblacional a través del siguiente comando :
 
```
# ----------------Parameters---------------------- #                                                              
#$ -S /bin/sh                                                                                                     
#$ -pe mthread 30                                                                             
#$ -q mThM.q                                                                                                      
#$ -l mres=240G,h_data=8G,h_vmem=8G,himem 
#$ -cwd                                                                                                           
#$ -j y                                                                                                           
#$ -N best                                                              
#$ -o best_k.log  
#$ -m bea                                                                                                         
#$ -M caballeroeg@si.edu
# 
# ----------------Modules------------------------- #          
module load bio/faststructure/1.0
#                                                                                                                 
# ----------------Your Commands------------------- #   

#                                                                                                                          
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME                                              
echo + NSLOTS = $NSLOTS  

distruct.py \          
-K $SGE_TASK_ID \                                                                                                 
--input=/scratch/genomics/caballeroeg/FasSTRUCTURE/k$SGE_TASK_ID/K_$SGE_TASK_ID \                                 
--output=/scratch/genomics/caballeroeg/FasSTRUCTURE/K$SGE_TASK_ID \                                               
--popfile=/scratch/genomics/caballeroeg/FasSTRUCTURE/popfile.txt                                                  
--title=K-means $SGE_TASK_ID  
```
 
 Estas figuras también se pueden gráficar con R 
 

```r
setwd("C:/Users/edgar/OneDrive/Documentos/Bioinformática/Structure/FASSTRUCTURE/meanQ_data")
tbl1<-read.table("K_1.1.meanQ")
attach(tbl1)
k1<-barplot(t(as.matrix(tbl1)), col=rainbow(3),
  xlab="Individual", ylab="Ancestry", border=NA,main="K=1")
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

```r
k1
```

```
##   [1]   0.7   1.9   3.1   4.3   5.5   6.7   7.9   9.1  10.3  11.5  12.7  13.9
##  [13]  15.1  16.3  17.5  18.7  19.9  21.1  22.3  23.5  24.7  25.9  27.1  28.3
##  [25]  29.5  30.7  31.9  33.1  34.3  35.5  36.7  37.9  39.1  40.3  41.5  42.7
##  [37]  43.9  45.1  46.3  47.5  48.7  49.9  51.1  52.3  53.5  54.7  55.9  57.1
##  [49]  58.3  59.5  60.7  61.9  63.1  64.3  65.5  66.7  67.9  69.1  70.3  71.5
##  [61]  72.7  73.9  75.1  76.3  77.5  78.7  79.9  81.1  82.3  83.5  84.7  85.9
##  [73]  87.1  88.3  89.5  90.7  91.9  93.1  94.3  95.5  96.7  97.9  99.1 100.3
##  [85] 101.5 102.7 103.9 105.1 106.3 107.5 108.7 109.9 111.1 112.3 113.5 114.7
##  [97] 115.9 117.1 118.3 119.5 120.7 121.9 123.1 124.3 125.5 126.7 127.9 129.1
## [109] 130.3 131.5 132.7 133.9 135.1 136.3 137.5 138.7 139.9 141.1 142.3 143.5
## [121] 144.7 145.9 147.1 148.3 149.5 150.7 151.9 153.1 154.3 155.5 156.7 157.9
## [133] 159.1 160.3 161.5 162.7 163.9 165.1 166.3 167.5 168.7 169.9 171.1 172.3
## [145] 173.5 174.7 175.9 177.1 178.3 179.5 180.7 181.9 183.1 184.3 185.5 186.7
```
 
Como podemos observar para K= 10 no existen 10 grupos diferenciables. Esto es porque el algoritmo detecta un gran grupo genotípico y no existen diferencias a nivel de genoma entre los 5 grupos de muestreos.


En el siguiente apartado veremos como hacer lo mismo con Admixture.

```r
setwd("C:/Users/edgar/OneDrive/Documentos/Bioinformática/Structure/FASSTRUCTURE/meanQ_data")
tbl2 = read.table("K_2.2.meanQ")
attach(tbl2)
```

```
## The following object is masked from tbl1:
## 
##     V1
```

```r
k2<-barplot(t(as.matrix(tbl2)), col=rainbow(3),
  xlab="Individual", ylab="Ancestry", border=NA,main="K=2")
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

```r
k2
```

```
##   [1]   0.7   1.9   3.1   4.3   5.5   6.7   7.9   9.1  10.3  11.5  12.7  13.9
##  [13]  15.1  16.3  17.5  18.7  19.9  21.1  22.3  23.5  24.7  25.9  27.1  28.3
##  [25]  29.5  30.7  31.9  33.1  34.3  35.5  36.7  37.9  39.1  40.3  41.5  42.7
##  [37]  43.9  45.1  46.3  47.5  48.7  49.9  51.1  52.3  53.5  54.7  55.9  57.1
##  [49]  58.3  59.5  60.7  61.9  63.1  64.3  65.5  66.7  67.9  69.1  70.3  71.5
##  [61]  72.7  73.9  75.1  76.3  77.5  78.7  79.9  81.1  82.3  83.5  84.7  85.9
##  [73]  87.1  88.3  89.5  90.7  91.9  93.1  94.3  95.5  96.7  97.9  99.1 100.3
##  [85] 101.5 102.7 103.9 105.1 106.3 107.5 108.7 109.9 111.1 112.3 113.5 114.7
##  [97] 115.9 117.1 118.3 119.5 120.7 121.9 123.1 124.3 125.5 126.7 127.9 129.1
## [109] 130.3 131.5 132.7 133.9 135.1 136.3 137.5 138.7 139.9 141.1 142.3 143.5
## [121] 144.7 145.9 147.1 148.3 149.5 150.7 151.9 153.1 154.3 155.5 156.7 157.9
## [133] 159.1 160.3 161.5 162.7 163.9 165.1 166.3 167.5 168.7 169.9 171.1 172.3
## [145] 173.5 174.7 175.9 177.1 178.3 179.5 180.7 181.9 183.1 184.3 185.5 186.7
```

```r
setwd("C:/Users/edgar/OneDrive/Documentos/Bioinformática/Structure/FASSTRUCTURE/meanQ_data")
tbl3 = read.table("K_3.3.meanQ")
attach(tbl3)
```

```
## The following objects are masked from tbl2:
## 
##     V1, V2
```

```
## The following object is masked from tbl1:
## 
##     V1
```

```r
k3<-barplot(t(as.matrix(tbl3)), col=rainbow(3),
  xlab="Individual", ylab="Ancestry", border=NA,main="K=3")
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

```r
k3
```

```
##   [1]   0.7   1.9   3.1   4.3   5.5   6.7   7.9   9.1  10.3  11.5  12.7  13.9
##  [13]  15.1  16.3  17.5  18.7  19.9  21.1  22.3  23.5  24.7  25.9  27.1  28.3
##  [25]  29.5  30.7  31.9  33.1  34.3  35.5  36.7  37.9  39.1  40.3  41.5  42.7
##  [37]  43.9  45.1  46.3  47.5  48.7  49.9  51.1  52.3  53.5  54.7  55.9  57.1
##  [49]  58.3  59.5  60.7  61.9  63.1  64.3  65.5  66.7  67.9  69.1  70.3  71.5
##  [61]  72.7  73.9  75.1  76.3  77.5  78.7  79.9  81.1  82.3  83.5  84.7  85.9
##  [73]  87.1  88.3  89.5  90.7  91.9  93.1  94.3  95.5  96.7  97.9  99.1 100.3
##  [85] 101.5 102.7 103.9 105.1 106.3 107.5 108.7 109.9 111.1 112.3 113.5 114.7
##  [97] 115.9 117.1 118.3 119.5 120.7 121.9 123.1 124.3 125.5 126.7 127.9 129.1
## [109] 130.3 131.5 132.7 133.9 135.1 136.3 137.5 138.7 139.9 141.1 142.3 143.5
## [121] 144.7 145.9 147.1 148.3 149.5 150.7 151.9 153.1 154.3 155.5 156.7 157.9
## [133] 159.1 160.3 161.5 162.7 163.9 165.1 166.3 167.5 168.7 169.9 171.1 172.3
## [145] 173.5 174.7 175.9 177.1 178.3 179.5 180.7 181.9 183.1 184.3 185.5 186.7
```


```r
setwd("C:/Users/edgar/OneDrive/Documentos/Bioinformática/Structure/FASSTRUCTURE/meanQ_data")
tbl4 = read.table("K_4.4.meanQ")
attach(tbl4)
```

```
## The following objects are masked from tbl3:
## 
##     V1, V2, V3
```

```
## The following objects are masked from tbl2:
## 
##     V1, V2
```

```
## The following object is masked from tbl1:
## 
##     V1
```

```r
k4<-barplot(t(as.matrix(tbl4)), col=rainbow(3),
  xlab="Individual", ylab="Ancestry", border=NA,main="K=4")
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-8-1.png)<!-- -->

```r
k4
```

```
##   [1]   0.7   1.9   3.1   4.3   5.5   6.7   7.9   9.1  10.3  11.5  12.7  13.9
##  [13]  15.1  16.3  17.5  18.7  19.9  21.1  22.3  23.5  24.7  25.9  27.1  28.3
##  [25]  29.5  30.7  31.9  33.1  34.3  35.5  36.7  37.9  39.1  40.3  41.5  42.7
##  [37]  43.9  45.1  46.3  47.5  48.7  49.9  51.1  52.3  53.5  54.7  55.9  57.1
##  [49]  58.3  59.5  60.7  61.9  63.1  64.3  65.5  66.7  67.9  69.1  70.3  71.5
##  [61]  72.7  73.9  75.1  76.3  77.5  78.7  79.9  81.1  82.3  83.5  84.7  85.9
##  [73]  87.1  88.3  89.5  90.7  91.9  93.1  94.3  95.5  96.7  97.9  99.1 100.3
##  [85] 101.5 102.7 103.9 105.1 106.3 107.5 108.7 109.9 111.1 112.3 113.5 114.7
##  [97] 115.9 117.1 118.3 119.5 120.7 121.9 123.1 124.3 125.5 126.7 127.9 129.1
## [109] 130.3 131.5 132.7 133.9 135.1 136.3 137.5 138.7 139.9 141.1 142.3 143.5
## [121] 144.7 145.9 147.1 148.3 149.5 150.7 151.9 153.1 154.3 155.5 156.7 157.9
## [133] 159.1 160.3 161.5 162.7 163.9 165.1 166.3 167.5 168.7 169.9 171.1 172.3
## [145] 173.5 174.7 175.9 177.1 178.3 179.5 180.7 181.9 183.1 184.3 185.5 186.7
```

```r
setwd("C:/Users/edgar/OneDrive/Documentos/Bioinformática/Structure/FASSTRUCTURE/meanQ_data")
tbl5 = read.table("K_5.5.meanQ")
attach(tbl5)
```

```
## The following objects are masked from tbl4:
## 
##     V1, V2, V3, V4
```

```
## The following objects are masked from tbl3:
## 
##     V1, V2, V3
```

```
## The following objects are masked from tbl2:
## 
##     V1, V2
```

```
## The following object is masked from tbl1:
## 
##     V1
```

```r
k5<-barplot(t(as.matrix(tbl5)), col=rainbow(3),
  xlab="Individual", ylab="Ancestry", border=NA,main="K=5")
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

```r
k5
```

```
##   [1]   0.7   1.9   3.1   4.3   5.5   6.7   7.9   9.1  10.3  11.5  12.7  13.9
##  [13]  15.1  16.3  17.5  18.7  19.9  21.1  22.3  23.5  24.7  25.9  27.1  28.3
##  [25]  29.5  30.7  31.9  33.1  34.3  35.5  36.7  37.9  39.1  40.3  41.5  42.7
##  [37]  43.9  45.1  46.3  47.5  48.7  49.9  51.1  52.3  53.5  54.7  55.9  57.1
##  [49]  58.3  59.5  60.7  61.9  63.1  64.3  65.5  66.7  67.9  69.1  70.3  71.5
##  [61]  72.7  73.9  75.1  76.3  77.5  78.7  79.9  81.1  82.3  83.5  84.7  85.9
##  [73]  87.1  88.3  89.5  90.7  91.9  93.1  94.3  95.5  96.7  97.9  99.1 100.3
##  [85] 101.5 102.7 103.9 105.1 106.3 107.5 108.7 109.9 111.1 112.3 113.5 114.7
##  [97] 115.9 117.1 118.3 119.5 120.7 121.9 123.1 124.3 125.5 126.7 127.9 129.1
## [109] 130.3 131.5 132.7 133.9 135.1 136.3 137.5 138.7 139.9 141.1 142.3 143.5
## [121] 144.7 145.9 147.1 148.3 149.5 150.7 151.9 153.1 154.3 155.5 156.7 157.9
## [133] 159.1 160.3 161.5 162.7 163.9 165.1 166.3 167.5 168.7 169.9 171.1 172.3
## [145] 173.5 174.7 175.9 177.1 178.3 179.5 180.7 181.9 183.1 184.3 185.5 186.7
```


```r
library(ggplot2)
library(ggpubr)
ggarrange(k1, k2, k3, k4, k5,ncol= 1, nrow = 2)
```

```
## Warning in as_grob.default(plot): Cannot convert object of class numeric into a
## grob.

## Warning in as_grob.default(plot): Cannot convert object of class numeric into a
## grob.

## Warning in as_grob.default(plot): Cannot convert object of class numeric into a
## grob.

## Warning in as_grob.default(plot): Cannot convert object of class numeric into a
## grob.

## Warning in as_grob.default(plot): Cannot convert object of class numeric into a
## grob.
```

```
## $`1`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-10-1.png)<!-- -->

```
## 
## $`2`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-10-2.png)<!-- -->

```
## 
## $`3`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-10-3.png)<!-- -->

```
## 
## attr(,"class")
## [1] "list"      "ggarrange"
```


## Admixture

[Admixture](https://dalexander.github.io/admixture/) es una alternativa similar a faststructure y una alrernativa para usar este software en caso de presentar dificultades con faststructure o STRUCTURE o para corroborar resultados. Para saber más sobre la documentación de [Admixture haga click aquí.](http://dalexander.github.io/admixture/admixture-manual.pdf)

```
# ----------------Parameters---------------------- #   

#$ -S /bin/sh                                                                                                     
#$ -pe mthread 30                                                                             
#$ -q mThM.q                                                                                                      
#$ -l mres=240G,h_data=8G,h_vmem=8G,himem 
#$ -cwd                                                                                                           
#$ -j y                                                                                                           
#$ -N best                                                              
#$ -o best_k.log  
#$ -m bea                                                                                                         
#$ -M caballeroeg@si.edu
# 
# ----------------Modules------------------------- #          
module load bio/faststructure/1.0
#                                                                                                                 
# ----------------Your Commands------------------- #   

#                                                                                                                    
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME                                              
echo + NSLOTS = $NSLOTS

admixture --cv /scratch/genomics/caballeroeg/post-STACKS/admixture/population_plink.ped $SGE_TASK_ID > log$SGE_TASK_ID.out 

```
Este archivo lo podemos interpretar de la siguiente manera, entre más bajo el valor de cv para cada K, es más probable que ese sea la cantidad de grupos genotípicos en nuestra población o set de datos.


```r
cv<- read.delim('cross_validation.txt')
cv
```

```
##     K CV.Error
## 1   1  0.25988
## 2   2  0.26884
## 3   3  0.28143
## 4   4  0.29361
## 5   5  0.30184
## 6   6  0.32059
## 7   7  0.33382
## 8   8  0.33866
## 9   9  0.35574
## 10 10  0.35849
```
Podemos observar que el valor más bajo es para `K`=1, el cual coincide con el valor obtenido por faststructure.


```r
library(ggrepel)
library(ggplot2)
ggplot(data = cv,aes(cv$K,cv$CV.Error)) + geom_point() + geom_line(color='aquamarine1') + ggtitle('Cross validation') + theme_classic() + theme(plot.title = element_text(hjust = 0.5)) + geom_text_repel(aes(label= cv$K))+ xlab("K") + ylab("Cross validation error")
```

```
## Warning: Use of `cv$K` is discouraged. Use `K` instead.
```

```
## Warning: Use of `cv$CV.Error` is discouraged. Use `CV.Error` instead.
```

```
## Warning: Use of `cv$K` is discouraged. Use `K` instead.
```

```
## Warning: Use of `cv$CV.Error` is discouraged. Use `CV.Error` instead.
```

```
## Warning: Use of `cv$K` is discouraged. Use `K` instead.
## Use of `cv$K` is discouraged. Use `K` instead.
```

```
## Warning: Use of `cv$CV.Error` is discouraged. Use `CV.Error` instead.
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-12-1.png)<!-- -->
Loading required libraries


```r
library(readr)
library(ggplot2)
```

Loading the libraries


```r
getwd()
```

```
## [1] "C:/Users/edgar/OneDrive/Documentos/R/cuaderno_de_laboratorio"
```

```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
```

Reading the `.tsv` files


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps1 <- read_tsv("populations.fst_1-2.tsv")
```

```
## Rows: 258708 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps1
```

```
## # A tibble: 258,708 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          2 un       85     84       0.461 
##  2            1          1          2 un      116    115       0.178 
##  3            1          1          2 un      118    117       1     
##  4            1          1          2 un      204    203       0.448 
##  5            1          1          2 un      207    206       0.0636
##  6            1          1          2 un      235    234       0.501 
##  7            1          1          2 un      247    246       0.842 
##  8            1          1          2 un      264    263       1     
##  9            1          1          2 un      299    298       0.754 
## 10            1          1          2 un      312    311       1     
## # … with 258,698 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

Adding a column with fakes bp to prevent locus to merge


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps1$bpfake <- 1:nrow(ps1)
ps1
```

```
## # A tibble: 258,708 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          2 un       85     84       0.461 
##  2            1          1          2 un      116    115       0.178 
##  3            1          1          2 un      118    117       1     
##  4            1          1          2 un      204    203       0.448 
##  5            1          1          2 un      207    206       0.0636
##  6            1          1          2 un      235    234       0.501 
##  7            1          1          2 un      247    246       0.842 
##  8            1          1          2 un      264    263       1     
##  9            1          1          2 un      299    298       0.754 
## 10            1          1          2 un      312    311       1     
## # … with 258,698 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

plotting the FST vs fake-bp for the comparison bettween population 1 and popuation 2


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
attach(ps1) #Attach the names of the columns without typing the dataframe using the "$"
names(ps1[12]) -> AMOVA_FST
ps1
```

```
## # A tibble: 258,708 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          2 un       85     84       0.461 
##  2            1          1          2 un      116    115       0.178 
##  3            1          1          2 un      118    117       1     
##  4            1          1          2 un      204    203       0.448 
##  5            1          1          2 un      207    206       0.0636
##  6            1          1          2 un      235    234       0.501 
##  7            1          1          2 un      247    246       0.842 
##  8            1          1          2 un      264    263       1     
##  9            1          1          2 un      299    298       0.754 
## 10            1          1          2 un      312    311       1     
## # … with 258,698 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
fst_2s_3b<-ggplot(data= ps1) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) +theme_classic() + ggtitle("2S27 vs 3B52") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_2s_3b
```

![](cuaderno_de_laboratorio_files/figure-html/fst_2s_3b_plot-1.png)<!-- -->

Checking outliers and max values to contrast the plots max `Fst`


```r
max(`AMOVA Fst`)
```

```
## [1] 0.70007
```

## Comparison 2 2S28 vs 3s92


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps2 <- read_tsv("populations.fst_1-3.tsv")
```

```
## Rows: 286268 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps2
```

```
## # A tibble: 286,268 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          3 un       85     84        0.469
##  2            1          1          3 un      116    115        0.766
##  3            1          1          3 un      118    117        1    
##  4            1          1          3 un      151    150        0.475
##  5            1          1          3 un      207    206        1    
##  6            1          1          3 un      235    234        0.496
##  7            1          1          3 un      247    246        1    
##  8            1          1          3 un      264    263        1    
##  9            1          1          3 un      299    298        0.739
## 10            1          1          3 un      312    311        0.421
## # … with 286,258 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

Adding the fake columns


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps2$bpfake <- 1:nrow(ps2)
ps2
```

```
## # A tibble: 286,268 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          3 un       85     84        0.469
##  2            1          1          3 un      116    115        0.766
##  3            1          1          3 un      118    117        1    
##  4            1          1          3 un      151    150        0.475
##  5            1          1          3 un      207    206        1    
##  6            1          1          3 un      235    234        0.496
##  7            1          1          3 un      247    246        1    
##  8            1          1          3 un      264    263        1    
##  9            1          1          3 un      299    298        0.739
## 10            1          1          3 un      312    311        0.421
## # … with 286,258 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

plotting the `FST` values


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
attach(ps2) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
attach(ps2) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps2 (pos = 3):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
names(ps2[12]) -> AMOVA_FST
ps2
```

```
## # A tibble: 286,268 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          3 un       85     84        0.469
##  2            1          1          3 un      116    115        0.766
##  3            1          1          3 un      118    117        1    
##  4            1          1          3 un      151    150        0.475
##  5            1          1          3 un      207    206        1    
##  6            1          1          3 un      235    234        0.496
##  7            1          1          3 un      247    246        1    
##  8            1          1          3 un      264    263        1    
##  9            1          1          3 un      299    298        0.739
## 10            1          1          3 un      312    311        0.421
## # … with 286,258 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
fst_2s_3s <- ggplot(data = ps2) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle("2S27 vs 3s92") + xlab("") + theme(plot.title = element_text(hjust = 0.5))

fst_2s_3s
```

![](cuaderno_de_laboratorio_files/figure-html/fst_2s_3s-1.png)<!-- -->

##Comparison 3 2s27 vs 42s28

Here is the code


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps3 <- read_tsv("populations.fst_1-4.tsv") #calling the file
```

```
## Rows: 308773 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps3
```

```
## # A tibble: 308,773 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          4 un       85     84        0.302
##  2            1          1          4 un      116    115        1    
##  3            1          1          4 un      118    117        1    
##  4            1          1          4 un      169    168        1    
##  5            1          1          4 un      207    206        0.773
##  6            1          1          4 un      235    234        0.622
##  7            1          1          4 un      247    246        0.689
##  8            1          1          4 un      264    263        0.5  
##  9            1          1          4 un      299    298        1    
## 10            1          1          4 un      312    311        0.680
## # … with 308,763 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps3$bpfake <- 1:nrow(ps3)  #adding the fake bp
ps3
```

```
## # A tibble: 308,773 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          4 un       85     84        0.302
##  2            1          1          4 un      116    115        1    
##  3            1          1          4 un      118    117        1    
##  4            1          1          4 un      169    168        1    
##  5            1          1          4 un      207    206        0.773
##  6            1          1          4 un      235    234        0.622
##  7            1          1          4 un      247    246        0.689
##  8            1          1          4 un      264    263        0.5  
##  9            1          1          4 un      299    298        1    
## 10            1          1          4 un      312    311        0.680
## # … with 308,763 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps3) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps2 (pos = 3):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 4):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_2s_4s <- ggplot(data = ps3) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle(" 2S27 vs  4s92") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_2s_4s
```

![](cuaderno_de_laboratorio_files/figure-html/fst_2s_4s-1.png)<!-- -->

## Comparison 4 2s27 vs 42b8


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps4 <- read_tsv("populations.fst_1-5.tsv") #calling the file
```

```
## Rows: 284285 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps4
```

```
## # A tibble: 284,285 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          5 un       85     84        0.799
##  2            1          1          5 un      116    115        1    
##  3            1          1          5 un      118    117        0.460
##  4            1          1          5 un      169    168        0.499
##  5            1          1          5 un      207    206        0.223
##  6            1          1          5 un      229    228        1    
##  7            1          1          5 un      235    234        0.220
##  8            1          1          5 un      247    246        0.675
##  9            1          1          5 un      264    263        0.471
## 10            1          1          5 un      299    298        1    
## # … with 284,275 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps4$bpfake <- 1:nrow(ps4)  #adding the fake bp
ps4
```

```
## # A tibble: 284,285 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          1          5 un       85     84        0.799
##  2            1          1          5 un      116    115        1    
##  3            1          1          5 un      118    117        0.460
##  4            1          1          5 un      169    168        0.499
##  5            1          1          5 un      207    206        0.223
##  6            1          1          5 un      229    228        1    
##  7            1          1          5 un      235    234        0.220
##  8            1          1          5 un      247    246        0.675
##  9            1          1          5 un      264    263        0.471
## 10            1          1          5 un      299    298        1    
## # … with 284,275 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps3) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps3 (pos = 3):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 4):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 5):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_2s_4b <- ggplot(data = ps4) + geom_point(aes(x= bpfake, y= `AMOVA Fst`), color= 'green') + theme_classic() + ggtitle("2S27 vs  42b8") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_2s_4b
```

![](cuaderno_de_laboratorio_files/figure-html/fst_2s_4b-1.png)<!-- -->

## comparison 5 3b52 vs 3s92


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps5 <- read_tsv("populations.fst_2-3.tsv") #calling the file
```

```
## Rows: 249676 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps5
```

```
## # A tibble: 249,676 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          2          3 un       85     84       1     
##  2            1          2          3 un      116    115       0.429 
##  3            1          2          3 un      118    117       1     
##  4            1          2          3 un      151    150       1     
##  5            1          2          3 un      204    203       0.464 
##  6            1          2          3 un      207    206       0.0358
##  7            1          2          3 un      247    246       1     
##  8            1          2          3 un      299    298       0.510 
##  9            1          2          3 un      312    311       0.370 
## 10            1          2          3 un      322    321       0.201 
## # … with 249,666 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps5$bpfake <- 1:nrow(ps5)  #adding the fake bp
ps5
```

```
## # A tibble: 249,676 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          2          3 un       85     84       1     
##  2            1          2          3 un      116    115       0.429 
##  3            1          2          3 un      118    117       1     
##  4            1          2          3 un      151    150       1     
##  5            1          2          3 un      204    203       0.464 
##  6            1          2          3 un      207    206       0.0358
##  7            1          2          3 un      247    246       1     
##  8            1          2          3 un      299    298       0.510 
##  9            1          2          3 un      312    311       0.370 
## 10            1          2          3 un      322    321       0.201 
## # … with 249,666 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps5) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps3 (pos = 3):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 4):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 5):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 6):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_3b_3s <- ggplot(data = ps5) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle("3b52 vs  3s92") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_3b_3s
```

![](cuaderno_de_laboratorio_files/figure-html/fst_3b_3s-1.png)<!-- -->

##Comparison 6 3b52 vs 42s28


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps6 <- read_tsv("populations.fst_2-4.tsv") #calling the file
```

```
## Rows: 268908 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps6
```

```
## # A tibble: 268,908 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          2          4 un       85     84        0.782
##  2            1          2          4 un      116    115        0.279
##  3            1          2          4 un      118    117        1    
##  4            1          2          4 un      169    168        1    
##  5            1          2          4 un      204    203        0.448
##  6            1          2          4 un      207    206        0.124
##  7            1          2          4 un      235    234        1    
##  8            1          2          4 un      247    246        0.552
##  9            1          2          4 un      299    298        0.751
## 10            1          2          4 un      312    311        0.384
## # … with 268,898 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps6$bpfake <- 1:nrow(ps6)  #adding the fake bp
ps6
```

```
## # A tibble: 268,908 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          2          4 un       85     84        0.782
##  2            1          2          4 un      116    115        0.279
##  3            1          2          4 un      118    117        1    
##  4            1          2          4 un      169    168        1    
##  5            1          2          4 un      204    203        0.448
##  6            1          2          4 un      207    206        0.124
##  7            1          2          4 un      235    234        1    
##  8            1          2          4 un      247    246        0.552
##  9            1          2          4 un      299    298        0.751
## 10            1          2          4 un      312    311        0.384
## # … with 268,898 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps6) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps5:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 4):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 5):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 6):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 7):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_3b_4s <- ggplot(data = ps6) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle("3b52 vs  42s28") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_3b_4s
```

![](cuaderno_de_laboratorio_files/figure-html/fst_3b_4s-1.png)<!-- -->

##Comparison 7 3b52 vs 42b8


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps7 <- read_tsv("populations.fst_2-5.tsv") #calling the file
```

```
## Rows: 240636 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps7
```

```
## # A tibble: 240,636 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          2          5 un       85     84        1    
##  2            1          2          5 un      116    115        0.159
##  3            1          2          5 un      118    117        0.433
##  4            1          2          5 un      169    168        0.509
##  5            1          2          5 un      204    203        0.419
##  6            1          2          5 un      207    206        0.392
##  7            1          2          5 un      229    228        1    
##  8            1          2          5 un      247    246        0.534
##  9            1          2          5 un      299    298        0.740
## 10            1          2          5 un      312    311        1    
## # … with 240,626 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps7$bpfake <- 1:nrow(ps7)  #adding the fake bp
ps7
```

```
## # A tibble: 240,636 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          2          5 un       85     84        1    
##  2            1          2          5 un      116    115        0.159
##  3            1          2          5 un      118    117        0.433
##  4            1          2          5 un      169    168        0.509
##  5            1          2          5 un      204    203        0.419
##  6            1          2          5 un      207    206        0.392
##  7            1          2          5 un      229    228        1    
##  8            1          2          5 un      247    246        0.534
##  9            1          2          5 un      299    298        0.740
## 10            1          2          5 un      312    311        1    
## # … with 240,626 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps7) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps6:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps5:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 5):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 6):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 7):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 8):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_3b_4b <- ggplot(data = ps7) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle("3b52 vs  42b8") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_3b_4b
```

![](cuaderno_de_laboratorio_files/figure-html/fst_3b_4b-1.png)<!-- -->

## Comparison 8 3s92 vs 42s28


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps8 <- read_tsv("populations.fst_3-4.tsv") #calling the file
```

```
## Rows: 299511 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps8
```

```
## # A tibble: 299,511 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          3          4 un       85     84        0.604
##  2            1          3          4 un      116    115        0.779
##  3            1          3          4 un      118    117        1    
##  4            1          3          4 un      151    150        0.475
##  5            1          3          4 un      169    168        1    
##  6            1          3          4 un      207    206        0.580
##  7            1          3          4 un      235    234        1    
##  8            1          3          4 un      247    246        0.569
##  9            1          3          4 un      299    298        1    
## 10            1          3          4 un      312    311        1    
## # … with 299,501 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps8$bpfake <- 1:nrow(ps8)  #adding the fake bp
ps8
```

```
## # A tibble: 299,511 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          3          4 un       85     84        0.604
##  2            1          3          4 un      116    115        0.779
##  3            1          3          4 un      118    117        1    
##  4            1          3          4 un      151    150        0.475
##  5            1          3          4 un      169    168        1    
##  6            1          3          4 un      207    206        0.580
##  7            1          3          4 un      235    234        1    
##  8            1          3          4 un      247    246        0.569
##  9            1          3          4 un      299    298        1    
## 10            1          3          4 un      312    311        1    
## # … with 299,501 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps8) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps7:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps6:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps5:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 6):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 7):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 8):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 9):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_3s_4s <- ggplot(data = ps8) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle("3S92 and  42s28") + xlab("") + theme(plot.title = element_text(hjust = 0.5))

fst_3s_4s
```

![](cuaderno_de_laboratorio_files/figure-html/fst_3s_4s-1.png)<!-- -->

##comparison 9 3s92 vs 42b8


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps9 <- read_tsv("populations.fst_3-5.tsv") #calling the file
```

```
## Rows: 273106 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps9
```

```
## # A tibble: 273,106 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          3          5 un       85     84        1    
##  2            1          3          5 un      116    115        0.751
##  3            1          3          5 un      118    117        0.469
##  4            1          3          5 un      151    150        0.446
##  5            1          3          5 un      169    168        0.500
##  6            1          3          5 un      207    206        0.223
##  7            1          3          5 un      229    228        1    
##  8            1          3          5 un      247    246        0.551
##  9            1          3          5 un      299    298        1    
## 10            1          3          5 un      312    311        0.178
## # … with 273,096 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps9$bpfake <- 1:nrow(ps9)  #adding the fake bp
ps9
```

```
## # A tibble: 273,106 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          3          5 un       85     84        1    
##  2            1          3          5 un      116    115        0.751
##  3            1          3          5 un      118    117        0.469
##  4            1          3          5 un      151    150        0.446
##  5            1          3          5 un      169    168        0.500
##  6            1          3          5 un      207    206        0.223
##  7            1          3          5 un      229    228        1    
##  8            1          3          5 un      247    246        0.551
##  9            1          3          5 un      299    298        1    
## 10            1          3          5 un      312    311        0.178
## # … with 273,096 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps9) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps8:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps7:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps6:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps5:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 7):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 8):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 9):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 10):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_3s_4b <- ggplot(data = ps3) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle("3s92 and  42b8") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_3s_4b
```

![](cuaderno_de_laboratorio_files/figure-html/fst_3s_4b-1.png)<!-- -->

## Comparison 10 42s28 vs 42b8


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
ps10 <- read_tsv("populations.fst_4-5.tsv") #calling the file
```

```
## Rows: 284742 Columns: 14
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: "\t"
## chr  (1): Chr
## dbl (13): # Locus ID, Pop 1 ID, Pop 2 ID, BP, Column, Fisher's P, Odds Ratio...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
ps10
```

```
## # A tibble: 284,742 × 14
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          4          5 un       85     84        0.576
##  2            1          4          5 un      116    115        1    
##  3            1          4          5 un      118    117        0.233
##  4            1          4          5 un      169    168        1    
##  5            1          4          5 un      207    206        0.515
##  6            1          4          5 un      229    228        1    
##  7            1          4          5 un      235    234        0.471
##  8            1          4          5 un      247    246        1    
##  9            1          4          5 un      299    298        1    
## 10            1          4          5 un      312    311        0.197
## # … with 284,732 more rows, and 7 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>
```

```r
ps10$bpfake <- 1:nrow(ps10)  #adding the fake bp
ps10
```

```
## # A tibble: 284,742 × 15
##    `# Locus ID` `Pop 1 ID` `Pop 2 ID` Chr      BP Column `Fisher's P`
##           <dbl>      <dbl>      <dbl> <chr> <dbl>  <dbl>        <dbl>
##  1            1          4          5 un       85     84        0.576
##  2            1          4          5 un      116    115        1    
##  3            1          4          5 un      118    117        0.233
##  4            1          4          5 un      169    168        1    
##  5            1          4          5 un      207    206        0.515
##  6            1          4          5 un      229    228        1    
##  7            1          4          5 un      235    234        0.471
##  8            1          4          5 un      247    246        1    
##  9            1          4          5 un      299    298        1    
## 10            1          4          5 un      312    311        0.197
## # … with 284,732 more rows, and 8 more variables: `Odds Ratio` <dbl>,
## #   `CI Low` <dbl>, `CI High` <dbl>, LOD <dbl>, `AMOVA Fst` <dbl>,
## #   `Smoothed AMOVA Fst` <dbl>, `Smoothed AMOVA Fst P-value` <dbl>,
## #   bpfake <int>
```

```r
attach(ps10) #Attach the names of the columns without typing the dataframe using the "$"
```

```
## The following objects are masked from ps9:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps8:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps7:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps6:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps5:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 8):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps3 (pos = 9):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 10):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps2 (pos = 11):
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
## 
## The following objects are masked from ps1:
## 
##     # Locus ID, AMOVA Fst, BP, bpfake, Chr, CI High, CI Low, Column,
##     Fisher's P, LOD, Odds Ratio, Pop 1 ID, Pop 2 ID, Smoothed AMOVA
##     Fst, Smoothed AMOVA Fst P-value
```

```r
fst_4s_4b <- ggplot(data = ps10) + geom_point(aes(x= bpfake, y= `AMOVA Fst`)) + theme_classic() + ggtitle("42s28 and  42b8") + xlab("") + theme(plot.title = element_text(hjust = 0.5))
fst_4s_4b
```

![](cuaderno_de_laboratorio_files/figure-html/fst_4s_4b-1.png)<!-- -->

## Figures arrange

### FST comparison based on phenotyoe (Host: Solanum ursinum)


```r
setwd("C:/Users/edgar/OneDrive/Documentos/R/DPCA_Alchisme_grossa/FST")
library(ggplot2)
library(ggpubr)
ggarrange(fst_3b_3s,fst_4s_4b,fst_2s_3s,fst_2s_4s,fst_3s_4s,fst_3s_4b,fst_3b_4s,fst_2s_3b,fst_2s_4s, ncol= 1, nrow = 2)
```

```
## $`1`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-20-1.png)<!-- -->

```
## 
## $`2`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-20-2.png)<!-- -->

```
## 
## $`3`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-20-3.png)<!-- -->

```
## 
## $`4`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-20-4.png)<!-- -->

```
## 
## $`5`
```

![](cuaderno_de_laboratorio_files/figure-html/unnamed-chunk-20-5.png)<!-- -->

```
## 
## attr(,"class")
## [1] "list"      "ggarrange"
```










